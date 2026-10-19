/*
 * ============================================================================
 * PROJECT: Dx Solution HS3PIK - Dual Screen POS System
 * NODE 1 : POS MASTER NODE (Optimized Boot & Deferred QR Check)
 * ============================================================================
 */

#include <Arduino.h>
#include <SPI.h>
#include <Wire.h>
#include <WiFi.h>
#include <WiFiClientSecure.h>
#include <esp_now.h>
#include <esp_wifi.h>
#include <FS.h>
#include <LittleFS.h>
#include <Preferences.h>
#include <WiFiManager.h>
#include <RTClib.h>
#include <TFT_eSPI.h>
#include <XPT2046_Touchscreen.h>
#include <lvgl.h>

// ================= ใช้ Broadcast Address สำหรับค้นหาจอ QR =================
uint8_t broadcastAddress[] = {0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF};
uint8_t targetMac[6] = {0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF};
uint32_t last_pong_time = 0; 

// ================= ฮาร์ดแวร์และการตั้งค่าพิน =================
#define TFT_BL 21
#define XPT2046_IRQ 36  
#define XPT2046_MOSI 32
#define XPT2046_MISO 39
#define XPT2046_CLK 25
#define XPT2046_CS 33
#define BUZZER_PIN 22   

#define TOUCH_MIN_X 280
#define TOUCH_MAX_X 3800
#define TOUCH_MIN_Y 310
#define TOUCH_MAX_Y 3700

XPT2046_Touchscreen ts(XPT2046_CS, XPT2046_IRQ); 
SPIClass touchSPI(HSPI);
TFT_eSPI tft = TFT_eSPI();

// ================= ธีมสี (Forecourt POS Style) =================
#define COLOR_BG        0x0A0F1D  
#define COLOR_PANEL     0x111827  
#define COLOR_CARD      0x1F2937  
#define COLOR_CYAN      0x00F5D4  
#define COLOR_GREEN     0x2ECC71  
#define COLOR_BLUE      0x3B82F6  
#define COLOR_BORDER    0x3B82F6  
#define COLOR_ORANGE    0xE67E22  
#define COLOR_YELLOW    0xF1C40F  
#define COLOR_RED       0xEF4444  
#define COLOR_WHITE     0xFFFFFF  
#define COLOR_GRAY      0x6B7280  

// ================= โครงสร้างข้อมูล ESP-NOW =================
enum PacketCommand : uint8_t { CMD_PING=1, CMD_SHOW_QR=2, CMD_CANCEL_QR=3, CMD_PAID_OK=4 };
typedef struct __attribute__((packed)) {
  uint8_t command;
  float amount;
  char promptpay_id[20];
  char shop_name[40];
} esp_now_payload_t;
esp_now_payload_t outgoingMsg, incomingMsg;

// ================= ตัวแปรระบบ & State Machine =================
enum AppState {
  STATE_BOOT_START,  
  STATE_BOOT_HW_OK,  
  STATE_WAIT_START,  
  STATE_POS_READY
};
AppState current_state = STATE_BOOT_START;
uint32_t state_timer = 0;
int active_screen = 1; 

RTC_DS3231 rtc;
Preferences preferences;
bool rtc_ok = false;
volatile bool is_qr_node_online = false;
bool last_online_state = false;
volatile bool flag_qr_paid_received = false;

float current_amount = 0.0;
float total_cash = 0.0;
float total_qr = 0.0;
int tx_count = 0;

char cfg_bot_token[60] = "";
char cfg_chat_id[30]   = "";
char cfg_promptpay[20] = "0812345678";
char cfg_shop_name[40] = "Dx Solution POS";

uint32_t last_tick = 0;
uint32_t last_clock_update = 0;

// ================= ประกาศตัวแปร LVGL Widgets ล่วงหน้า =================
lv_obj_t *scr_logo, *scr_config, *scr_ip_display;
lv_obj_t *page_boot, *page_pos, *page_history, *page_sum;
lv_obj_t *lbl_boot_rtc, *lbl_boot_wifi, *lbl_config_countdown;
lv_obj_t *lbl_pos_time, *btn_qr_status_interactive, *lbl_qr_status_text, *lbl_amount;
lv_obj_t *btn_start_pos, *btn_pay_qr;
lv_obj_t *lbl_history_text;
lv_obj_t *lbl_sum_cash, *lbl_sum_qr, *lbl_sum_total, *lbl_tg_status;
lv_timer_t *heartbeat_timer = NULL;

lv_obj_t *modal_cash = NULL;
lv_obj_t *modal_qr = NULL;
lv_obj_t *modal_shift = NULL; 

bool config_interrupted = false;

// ================= ฟังก์ชันจัดการข้อมูลและเสียง =================
void load_settings() {
  preferences.begin("pos_cfg", true);
  strlcpy(cfg_shop_name, preferences.getString("shop", "Dx Solution").c_str(), sizeof(cfg_shop_name));
  strlcpy(cfg_promptpay, preferences.getString("pp", "0812345678").c_str(), sizeof(cfg_promptpay));
  strlcpy(cfg_bot_token, preferences.getString("token", "").c_str(), sizeof(cfg_bot_token));
  strlcpy(cfg_chat_id,   preferences.getString("chatid", "").c_str(), sizeof(cfg_chat_id));
  preferences.end();
}

void beep(int duration_ms) {
  pinMode(BUZZER_PIN, OUTPUT);
  digitalWrite(BUZZER_PIN, HIGH);
  delay(duration_ms);
  digitalWrite(BUZZER_PIN, LOW);
}

void beepAlert(int type) {
  pinMode(BUZZER_PIN, OUTPUT);
  if (type == 1) { // หลุดการเชื่อมต่อ: ดังยาว 1 ครั้ง (800ms)
    digitalWrite(BUZZER_PIN, HIGH); delay(800); digitalWrite(BUZZER_PIN, LOW);
  } else if (type == 2) { // เชื่อมต่อสำเร็จ: ดังยาว 2 ครั้ง
    for (int i = 0; i < 2; i++) {
      digitalWrite(BUZZER_PIN, HIGH); delay(400); digitalWrite(BUZZER_PIN, LOW);
      if (i == 0) delay(200);
    }
  }
}

String getCurrentTimestamp() {
  if (!rtc_ok) return "00:00:00 , 00:00:00";
  DateTime now = rtc.now();
  char buf[32];
  snprintf(buf, sizeof(buf), "%02d:%02d:%02d , %02d:%02d:%02d", 
           now.day(), now.month(), now.year() % 100, 
           now.hour(), now.minute(), now.second());
  return String(buf);
}

void recordSale(const char* method, float amount) {
  bool isNew = !LittleFS.exists("/sales.csv");
  File f = LittleFS.open("/sales.csv", FILE_APPEND);
  if (!f) return;
  if (isNew) f.println("No,DateTime,Method,Amount");
  tx_count++;
  f.printf("%d,%s,%s,%.0f\n", tx_count, getCurrentTimestamp().c_str(), method, amount);
  f.close();
}

void updateHistoryUI() {
  if (!LittleFS.exists("/sales.csv")) {
    lv_label_set_text(lbl_history_text, "No transaction history.\n");
    return;
  }
  File f = LittleFS.open("/sales.csv", "r");
  if (!f) return;
  
  String txt = "";
  if (f.available()) f.readStringUntil('\n'); 
  while (f.available()) {
    String line = f.readStringUntil('\n');
    line.trim();
    if (line.length() > 0) txt += line + "\n";
  }
  f.close();
  lv_label_set_text(lbl_history_text, txt.c_str());
}

bool sendTelegramSummaryAndCSV() {
  if (strlen(cfg_bot_token) == 0 || strlen(cfg_chat_id) == 0) return false;
  WiFiClientSecure client;
  client.setInsecure();
  if (!client.connect("api.telegram.org", 443)) return false;

  String summaryText = "📊 *Sales Summary: " + String(cfg_shop_name) + "*\n";
  summaryText += "🕒 Time: " + getCurrentTimestamp() + "\n";
  summaryText += "💵 Cash: " + String(total_cash, 0) + " THB\n";
  summaryText += "📱 QR Scan: " + String(total_qr, 0) + " THB\n";
  summaryText += "💰 *Grand Total: " + String(total_cash + total_qr, 0) + " THB*\n";
  summaryText += "🧾 Total Bills: " + String(tx_count);

  String jsonPayload = "{\"chat_id\":\"" + String(cfg_chat_id) + "\",\"text\":\"" + summaryText + "\",\"parse_mode\":\"Markdown\"}";
  jsonPayload.replace("\n", "\\n");

  client.println("POST /bot" + String(cfg_bot_token) + "/sendMessage HTTP/1.1");
  client.println("Host: api.telegram.org");
  client.println("Content-Type: application/json");
  client.print("Content-Length: ");
  client.println(jsonPayload.length());
  client.println();
  client.print(jsonPayload);
  client.stop();
  return true;
}

// ================= ESP-NOW Callbacks =================
#if ESP_ARDUINO_VERSION >= ESP_ARDUINO_VERSION_VAL(3, 0, 0)
void OnDataSent(const uint8_t *mac_addr, esp_now_send_status_t status) {}
void OnDataRecv(const esp_now_recv_info_t *info, const uint8_t *incomingData, int len) {
  uint8_t senderMac[6]; memcpy(senderMac, info->src_addr, 6);
#else
void OnDataSent(const uint8_t *mac_addr, esp_now_send_status_t status) {}
void OnDataRecv(const uint8_t *mac, const uint8_t *incomingData, int len) {
  uint8_t senderMac[6]; memcpy(senderMac, mac, 6);
#endif
  if (len == sizeof(esp_now_payload_t)) {
    memcpy(&incomingMsg, incomingData, sizeof(esp_now_payload_t));
    if (incomingMsg.command == CMD_PING) {
      last_pong_time = millis();
      is_qr_node_online = true;
      if (!esp_now_is_peer_exist(senderMac)) {
        esp_now_peer_info_t peerInfo = {};
        memcpy(peerInfo.peer_addr, senderMac, 6);
        peerInfo.channel = 0; peerInfo.encrypt = false;
        esp_now_add_peer(&peerInfo);
      }
      memcpy(targetMac, senderMac, 6);
    }
    else if (incomingMsg.command == CMD_PAID_OK) { flag_qr_paid_received = true; }
  }
}

void sendEspNowPacket(uint8_t cmd, float amount) {
  outgoingMsg.command = cmd;
  outgoingMsg.amount  = amount;
  strlcpy(outgoingMsg.promptpay_id, cfg_promptpay, sizeof(outgoingMsg.promptpay_id));
  strlcpy(outgoingMsg.shop_name, cfg_shop_name, sizeof(outgoingMsg.shop_name));
  esp_now_send(targetMac, (uint8_t *)&outgoingMsg, sizeof(outgoingMsg));
}

// ================= LVGL Touch & Flush =================
void my_disp_flush(lv_disp_drv_t *disp, const lv_area_t *area, lv_color_t *color_p) {
  uint32_t w = (area->x2 - area->x1 + 1); uint32_t h = (area->y2 - area->y1 + 1);
  tft.startWrite(); tft.setAddrWindow(area->x1, area->y1, w, h);
  tft.pushColors((uint16_t *)&color_p->full, w * h, true); tft.endWrite();
  lv_disp_flush_ready(disp);
}
void my_touchpad_read(lv_indev_drv_t *indev_driver, lv_indev_data_t *data) {
  if (ts.touched()) {
    TS_Point p = ts.getPoint();
    if (p.z > 300) {
      data->state = LV_INDEV_STATE_PR;
      data->point.x = map(p.x, TOUCH_MIN_X, TOUCH_MAX_X, 0, 320);
      data->point.y = map(p.y, TOUCH_MIN_Y, TOUCH_MAX_Y, 0, 240);
    } else { data->state = LV_INDEV_STATE_REL; }
  } else { data->state = LV_INDEV_STATE_REL; }
}

// ================= UI Helpers =================
static void updateAmountDisplay() {
  char buf[32]; snprintf(buf, sizeof(buf), "THB %.0f", current_amount);
  lv_label_set_text(lbl_amount, buf);
}

static void updateSummaryUI() {
  char b1[40], b2[40], b3[40];
  snprintf(b1, sizeof(b1), "Cash Total: %.0f THB", total_cash);
  snprintf(b2, sizeof(b2), "QR Total: %.0f THB", total_qr);
  snprintf(b3, sizeof(b3), "GRAND TOTAL: %.0f THB", total_cash + total_qr);
  lv_label_set_text(lbl_sum_cash, b1);
  lv_label_set_text(lbl_sum_qr, b2);
  lv_label_set_text(lbl_sum_total, b3);
}

static void heartbeat_timer_cb(lv_timer_t *timer) { sendEspNowPacket(CMD_PING, 0); }

static void screen_gesture_event_cb(lv_event_t * e) {
  lv_dir_t dir = lv_indev_get_gesture_dir(lv_indev_get_act());
  if (!modal_cash && !modal_qr && !modal_shift) {
    if (dir == LV_DIR_LEFT) {
      if (active_screen == 1) {
        updateHistoryUI(); lv_scr_load_anim(page_history, LV_SCR_LOAD_ANIM_MOVE_LEFT, 200, 0, false); active_screen = 2;
      } else if (active_screen == 2) {
        updateSummaryUI(); lv_scr_load_anim(page_sum, LV_SCR_LOAD_ANIM_MOVE_LEFT, 200, 0, false); active_screen = 3;
      }
    } else if (dir == LV_DIR_RIGHT) {
      if (active_screen == 3) {
        updateHistoryUI(); lv_scr_load_anim(page_history, LV_SCR_LOAD_ANIM_MOVE_RIGHT, 200, 0, false); active_screen = 2;
      } else if (active_screen == 2) {
        lv_scr_load_anim(page_pos, LV_SCR_LOAD_ANIM_MOVE_RIGHT, 200, 0, false); active_screen = 1;
      }
    }
  }
}

// ================= Callbacks ของระบบ Popup =================
static void btn_cash_ok_cb(lv_event_t *e) {
  if (current_amount > 0) { total_cash += current_amount; recordSale("CASH", current_amount); current_amount = 0; updateAmountDisplay(); }
  if (modal_cash) { lv_obj_del(modal_cash); modal_cash = NULL; }
}
static void btn_cash_cancel_cb(lv_event_t *e) { if (modal_cash) { lv_obj_del(modal_cash); modal_cash = NULL; } }

static void btn_qr_manual_ok_cb(lv_event_t *e) {
  if (current_amount > 0) { total_qr += current_amount; recordSale("QR", current_amount); current_amount = 0; updateAmountDisplay(); }
  sendEspNowPacket(CMD_CANCEL_QR, 0);
  if (modal_qr) { lv_obj_del(modal_qr); modal_qr = NULL; }
}
static void btn_qr_cancel_cb(lv_event_t *e) {
  sendEspNowPacket(CMD_CANCEL_QR, 0);
  if (modal_qr) { lv_obj_del(modal_qr); modal_qr = NULL; }
}

static void shift_only_clear_cb(lv_event_t *e) {
  if (LittleFS.exists("/sales.csv")) {
    String ts = getCurrentTimestamp(); ts.replace(" ", "_"); ts.replace(":", ""); ts.replace("-", "");
    LittleFS.rename("/sales.csv", "/sales_" + ts + ".csv");
  }
  updateHistoryUI(); 
  if(modal_shift) { lv_obj_del(modal_shift); modal_shift = NULL; }
  lv_label_set_text(lbl_tg_status, "Shift closed. Ready for new orders.");
  total_cash = 0; total_qr = 0; tx_count = 0; updateSummaryUI();
}

static void shift_send_tg_cb(lv_event_t *e) {
  if(modal_shift) { lv_obj_del(modal_shift); modal_shift = NULL; }
  lv_label_set_text(lbl_tg_status, "Connecting WiFi & Telegram..."); lv_timer_handler();

  WiFi.begin(); uint32_t t = millis();
  while (WiFi.status() != WL_CONNECTED && millis() - t < 8000) { delay(200); lv_timer_handler(); }

  if (WiFi.status() == WL_CONNECTED && sendTelegramSummaryAndCSV()) {
    lv_label_set_text(lbl_tg_status, "Telegram Sent Successfully!");
    if (LittleFS.exists("/sales.csv")) {
      String ts = getCurrentTimestamp(); ts.replace(" ", "_"); ts.replace(":", ""); ts.replace("-", "");
      LittleFS.rename("/sales.csv", "/sales_" + ts + ".csv");
    }
    updateHistoryUI(); total_cash = 0; total_qr = 0; tx_count = 0; updateSummaryUI();
  } else {
    lv_label_set_text(lbl_tg_status, "Send Failed! Check WiFi.");
  }
  WiFi.disconnect(); WiFi.mode(WIFI_STA); 
  esp_wifi_set_promiscuous(true); esp_wifi_set_channel(1, WIFI_SECOND_CHAN_NONE); esp_wifi_set_promiscuous(false);
}

static void btn_shift_close_event_cb(lv_event_t *e) {
  if(modal_shift || modal_cash || modal_qr) return;
  modal_shift = lv_obj_create(lv_scr_act()); lv_obj_set_size(modal_shift, 280, 180); lv_obj_center(modal_shift);
  lv_obj_set_style_bg_color(modal_shift, lv_color_hex(COLOR_CARD), 0); lv_obj_set_style_bg_opa(modal_shift, LV_OPA_COVER, 0);
  lv_obj_set_style_border_color(modal_shift, lv_color_hex(COLOR_CYAN), 0); lv_obj_set_style_border_width(modal_shift, 2, 0);

  lv_obj_t *lbl = lv_label_create(modal_shift);
  char m_buf[128]; snprintf(m_buf, sizeof(m_buf), "Shift Closed at:\n%s\n\nSend data to Telegram?", getCurrentTimestamp().c_str());
  lv_label_set_text(lbl, m_buf); lv_obj_set_style_text_color(lbl, lv_color_hex(COLOR_YELLOW), 0);
  lv_obj_set_style_text_font(lbl, &lv_font_montserrat_14, 0); lv_obj_set_style_text_align(lbl, LV_TEXT_ALIGN_CENTER, 0); lv_obj_align(lbl, LV_ALIGN_TOP_MID, 0, 15);

  lv_obj_t *btn_yes = lv_btn_create(modal_shift); lv_obj_set_size(btn_yes, 110, 40); lv_obj_align(btn_yes, LV_ALIGN_BOTTOM_LEFT, 10, -10);
  lv_obj_set_style_bg_color(btn_yes, lv_color_hex(COLOR_GREEN), 0); lv_obj_add_event_cb(btn_yes, shift_send_tg_cb, LV_EVENT_CLICKED, NULL);
  lv_obj_t *lbl_yes = lv_label_create(btn_yes); lv_label_set_text(lbl_yes, "Yes (Send)"); lv_obj_center(lbl_yes);

  lv_obj_t *btn_no = lv_btn_create(modal_shift); lv_obj_set_size(btn_no, 110, 40); lv_obj_align(btn_no, LV_ALIGN_BOTTOM_RIGHT, -10, -10);
  lv_obj_set_style_bg_color(btn_no, lv_color_hex(COLOR_ORANGE), 0); lv_obj_add_event_cb(btn_no, shift_only_clear_cb, LV_EVENT_CLICKED, NULL);
  lv_obj_t *lbl_no = lv_label_create(btn_no); lv_label_set_text(lbl_no, "No (Clear)"); lv_obj_center(lbl_no);
  
  lv_obj_t *btn_close = lv_btn_create(modal_shift); lv_obj_set_size(btn_close, 30, 30); lv_obj_align(btn_close, LV_ALIGN_TOP_RIGHT, 0, 0);
  lv_obj_set_style_bg_color(btn_close, lv_color_hex(COLOR_RED), 0);
  lv_obj_add_event_cb(btn_close, [](lv_event_t *ev){ if(modal_shift) { lv_obj_del(modal_shift); modal_shift = NULL; } }, LV_EVENT_CLICKED, NULL);
  lv_obj_t *lbl_close = lv_label_create(btn_close); lv_label_set_text(lbl_close, "X"); lv_obj_center(lbl_close);
}

// ================= สร้างหน้าจอทั้งหมด =================
void createScreens() {
  scr_logo = lv_obj_create(NULL); lv_obj_set_style_bg_color(scr_logo, lv_color_hex(COLOR_BG), 0);
  lv_obj_set_style_bg_opa(scr_logo, LV_OPA_COVER, 0); lv_obj_clear_flag(scr_logo, LV_OBJ_FLAG_SCROLLABLE);
  lv_obj_t * lbl_logo_title = lv_label_create(scr_logo); lv_label_set_text(lbl_logo_title, "POS SYSTEM");
  lv_obj_set_style_text_font(lbl_logo_title, &lv_font_montserrat_20, 0); lv_obj_set_style_text_color(lbl_logo_title, lv_color_hex(COLOR_CYAN), 0);
  lv_obj_align(lbl_logo_title, LV_ALIGN_CENTER, 0, -10);

  scr_config = lv_obj_create(NULL); lv_obj_set_style_bg_color(scr_config, lv_color_hex(COLOR_BG), 0);
  lv_obj_set_style_bg_opa(scr_config, LV_OPA_COVER, 0); lv_obj_clear_flag(scr_config, LV_OBJ_FLAG_SCROLLABLE);
  lv_obj_t * lbl_cfg_title = lv_label_create(scr_config); lv_label_set_text(lbl_cfg_title, "SYSTEM CONFIG MODE");
  lv_obj_set_style_text_font(lbl_cfg_title, &lv_font_montserrat_14, 0); lv_obj_set_style_text_color(lbl_cfg_title, lv_color_hex(COLOR_YELLOW), 0); lv_obj_align(lbl_cfg_title, LV_ALIGN_TOP_MID, 0, 15);
  lbl_config_countdown = lv_label_create(scr_config); lv_label_set_text(lbl_config_countdown, "Starting POS in 3s...");
  lv_obj_set_style_text_font(lbl_config_countdown, &lv_font_montserrat_12, 0); lv_obj_set_style_text_color(lbl_config_countdown, lv_color_hex(COLOR_WHITE), 0); lv_obj_align(lbl_config_countdown, LV_ALIGN_TOP_MID, 0, 45);
  lv_obj_t * btn_cfg = lv_btn_create(scr_config); lv_obj_set_size(btn_cfg, 240, 50); lv_obj_align(btn_cfg, LV_ALIGN_CENTER, 0, 10);
  lv_obj_set_style_bg_color(btn_cfg, lv_color_hex(COLOR_BLUE), 0); lv_obj_add_event_cb(btn_cfg, [](lv_event_t * e){ config_interrupted = true; }, LV_EVENT_CLICKED, NULL);
  lv_obj_t * lbl_btn_cfg = lv_label_create(btn_cfg); lv_label_set_text(lbl_btn_cfg, "CONFIG / WI-FI SETUP"); lv_obj_set_style_text_font(lbl_btn_cfg, &lv_font_montserrat_12, 0); lv_obj_set_style_text_color(lbl_btn_cfg, lv_color_hex(COLOR_WHITE), 0); lv_obj_center(lbl_btn_cfg);

  scr_ip_display = lv_obj_create(NULL); lv_obj_set_style_bg_color(scr_ip_display, lv_color_hex(COLOR_BG), 0);
  lv_obj_set_style_bg_opa(scr_ip_display, LV_OPA_COVER, 0); lv_obj_clear_flag(scr_ip_display, LV_OBJ_FLAG_SCROLLABLE);
  lv_obj_t * lbl_ip_title = lv_label_create(scr_ip_display); lv_label_set_text(lbl_ip_title, "WI-FI SETUP MODE");
  lv_obj_set_style_text_font(lbl_ip_title, &lv_font_montserrat_14, 0); lv_obj_set_style_text_color(lbl_ip_title, lv_color_hex(COLOR_GREEN), 0); lv_obj_align(lbl_ip_title, LV_ALIGN_TOP_MID, 0, 20);
  lv_obj_t * lbl_ip_address = lv_label_create(scr_ip_display); lv_label_set_text(lbl_ip_address, "SSID: POS_Config_AP\nIP: 192.168.4.1\n\nConnect phone to config");
  lv_obj_set_style_text_color(lbl_ip_address, lv_color_hex(COLOR_WHITE), 0); lv_obj_align(lbl_ip_address, LV_ALIGN_CENTER, 0, 0);

  // หน้าจอ HW Check (เอา QR Check ออก เพื่อไม่ให้บล็อกตอนบูต)
  page_boot = lv_obj_create(NULL); lv_obj_set_style_bg_color(page_boot, lv_color_hex(COLOR_BG), 0);
  lv_obj_set_style_bg_opa(page_boot, LV_OPA_COVER, 0); lv_obj_clear_flag(page_boot, LV_OBJ_FLAG_SCROLLABLE);
  lv_obj_t * lbl_boot_title = lv_label_create(page_boot); lv_label_set_text(lbl_boot_title, "SYSTEM DIAGNOSTICS & BOOT");
  lv_obj_set_style_text_font(lbl_boot_title, &lv_font_montserrat_14, 0); lv_obj_set_style_text_color(lbl_boot_title, lv_color_hex(COLOR_CYAN), 0); lv_obj_align(lbl_boot_title, LV_ALIGN_TOP_MID, 0, 15);
  
  lbl_boot_rtc = lv_label_create(page_boot); lv_label_set_text(lbl_boot_rtc, "1. RTC Check ... [WAIT]");
  lv_obj_set_style_text_color(lbl_boot_rtc, lv_color_hex(COLOR_GRAY), 0); lv_obj_align(lbl_boot_rtc, LV_ALIGN_TOP_LEFT, 20, 50);
  
  lbl_boot_wifi = lv_label_create(page_boot); lv_label_set_text(lbl_boot_wifi, "2. Wi-Fi & Time ... [WAIT]");
  lv_obj_set_style_text_color(lbl_boot_wifi, lv_color_hex(COLOR_GRAY), 0); lv_obj_align(lbl_boot_wifi, LV_ALIGN_TOP_LEFT, 20, 90);

  // ปุ่ม START POS ที่เมื่อกดแล้วจะทำการทดสอบเชื่อมต่อ QR Node 2 วินาทีก่อนเข้าหน้าจอขายจริง
  btn_start_pos = lv_btn_create(page_boot); lv_obj_set_size(btn_start_pos, 220, 48); lv_obj_align(btn_start_pos, LV_ALIGN_BOTTOM_MID, 0, -20);
  lv_obj_set_style_bg_color(btn_start_pos, lv_color_hex(COLOR_BLUE), 0); lv_obj_add_state(btn_start_pos, LV_STATE_DISABLED);
  
  lv_obj_add_event_cb(btn_start_pos, [](lv_event_t *e){
    // เมื่อกด START POS ให้แสดงข้อความทดสอบ QR Node 2 วินาที
    lv_label_set_text(lbl_boot_wifi, "Connecting QR Node (2s)...");
    lv_timer_handler();
    
    // ส่ง Ping ทดสอบหา QR Node
    sendEspNowPacket(CMD_PING, 0);
    uint32_t check_start = millis();
    while (millis() - check_start < 2000) {
      lv_timer_handler();
      delay(50);
    }
    
    current_state = STATE_POS_READY; 
    active_screen = 1; 
    lv_scr_load(page_pos);
  }, LV_EVENT_CLICKED, NULL);

  lv_obj_t *lbl_s = lv_label_create(btn_start_pos); lv_label_set_text(lbl_s, "START POS"); lv_obj_set_style_text_color(lbl_s, lv_color_hex(COLOR_WHITE), 0); lv_obj_center(lbl_s);

  // หน้า POS
  page_pos = lv_obj_create(NULL); lv_obj_set_style_bg_color(page_pos, lv_color_hex(COLOR_BG), 0);
  lv_obj_set_style_bg_opa(page_pos, LV_OPA_COVER, 0); lv_obj_add_event_cb(page_pos, screen_gesture_event_cb, LV_EVENT_GESTURE, NULL);
  lbl_pos_time = lv_label_create(page_pos); lv_label_set_text(lbl_pos_time, "00:00");
  lv_obj_set_style_text_color(lbl_pos_time, lv_color_hex(COLOR_YELLOW), 0); lv_obj_align(lbl_pos_time, LV_ALIGN_TOP_LEFT, 10, 10);
  
  btn_qr_status_interactive = lv_btn_create(page_pos); lv_obj_set_size(btn_qr_status_interactive, 95, 28);
  lv_obj_align(btn_qr_status_interactive, LV_ALIGN_TOP_RIGHT, -5, 6); lv_obj_set_style_bg_color(btn_qr_status_interactive, lv_color_hex(COLOR_CARD), 0);
  lv_obj_set_style_border_color(btn_qr_status_interactive, lv_color_hex(COLOR_BORDER), 0); lv_obj_set_style_border_width(btn_qr_status_interactive, 1, 0);
  lv_obj_add_event_cb(btn_qr_status_interactive, [](lv_event_t *e){ if (heartbeat_timer) lv_timer_reset(heartbeat_timer); sendEspNowPacket(CMD_PING, 0); }, LV_EVENT_CLICKED, NULL);
  lbl_qr_status_text = lv_label_create(btn_qr_status_interactive); lv_label_set_text(lbl_qr_status_text, "QR Offline");
  lv_obj_set_style_text_color(lbl_qr_status_text, lv_color_hex(COLOR_RED), 0); lv_obj_center(lbl_qr_status_text);

  lv_obj_t *panel_amt = lv_obj_create(page_pos); lv_obj_set_size(panel_amt, 300, 45);
  lv_obj_align(panel_amt, LV_ALIGN_TOP_MID, 0, 45); lv_obj_set_style_bg_color(panel_amt, lv_color_hex(COLOR_PANEL), 0);
  lv_obj_set_style_border_color(panel_amt, lv_color_hex(COLOR_BORDER), 0); lv_obj_set_style_border_width(panel_amt, 2, 0);
  lbl_amount = lv_label_create(panel_amt); lv_label_set_text(lbl_amount, "THB 0");
  lv_obj_set_style_text_color(lbl_amount, lv_color_hex(COLOR_CYAN), 0); lv_obj_set_style_text_font(lbl_amount, &lv_font_montserrat_20, 0); lv_obj_center(lbl_amount);

  int hk_vals[5] = {5, 10, 20, 40, 0}; const char *hk_Texts[5] = {"+5", "+10", "+20", "+40", "CLR"};
  for (int i = 0; i < 5; i++) {
    lv_obj_t *b = lv_btn_create(page_pos); lv_obj_set_size(b, 56, 42); lv_obj_align(b, LV_ALIGN_TOP_LEFT, 10 + (i * 61), 97);
    lv_obj_set_style_bg_color(b, i==4 ? lv_color_hex(COLOR_RED) : lv_color_hex(COLOR_CARD), 0);
    lv_obj_add_event_cb(b, [](lv_event_t *e){
      if(modal_cash || modal_qr || modal_shift) return; 
      int v = (int)(intptr_t)lv_event_get_user_data(e); if(v==0) current_amount = 0; else current_amount+=v; updateAmountDisplay();
    }, LV_EVENT_CLICKED, (void *)(intptr_t)hk_vals[i]);
    lv_obj_t *lb = lv_label_create(b); lv_label_set_text(lb, hk_Texts[i]); lv_obj_set_style_text_color(lb, lv_color_hex(COLOR_WHITE), 0); lv_obj_center(lb);
  }

  lv_obj_t *btn_cash = lv_btn_create(page_pos); lv_obj_set_size(btn_cash, 145, 48); lv_obj_align(btn_cash, LV_ALIGN_BOTTOM_LEFT, 10, -10);
  lv_obj_set_style_bg_color(btn_cash, lv_color_hex(COLOR_GREEN), 0);
  lv_obj_add_event_cb(btn_cash, [](lv_event_t *e){
    if(current_amount <= 0 || modal_cash || modal_qr || modal_shift) return;
    modal_cash = lv_obj_create(lv_scr_act()); lv_obj_set_size(modal_cash, 260, 160); lv_obj_center(modal_cash);
    lv_obj_set_style_bg_color(modal_cash, lv_color_hex(COLOR_CARD), 0); lv_obj_set_style_bg_opa(modal_cash, LV_OPA_COVER, 0); lv_obj_set_style_border_color(modal_cash, lv_color_hex(COLOR_GREEN), 0); lv_obj_set_style_border_width(modal_cash, 2, 0);
    lv_obj_t *lbl_title = lv_label_create(modal_cash); lv_label_set_text(lbl_title, "Confirm CASH Payment?"); lv_obj_set_style_text_color(lbl_title, lv_color_hex(COLOR_WHITE), 0); lv_obj_align(lbl_title, LV_ALIGN_TOP_MID, 0, 10);
    lv_obj_t *lbl_amt = lv_label_create(modal_cash); char m_buf[32]; snprintf(m_buf, sizeof(m_buf), "%.0f THB", current_amount);
    lv_label_set_text(lbl_amt, m_buf); lv_obj_set_style_text_color(lbl_amt, lv_color_hex(COLOR_YELLOW), 0); lv_obj_set_style_text_font(lbl_amt, &lv_font_montserrat_20, 0); lv_obj_align(lbl_amt, LV_ALIGN_CENTER, 0, -15);
    lv_obj_t *btn_ok = lv_btn_create(modal_cash); lv_obj_set_size(btn_ok, 100, 40); lv_obj_align(btn_ok, LV_ALIGN_BOTTOM_LEFT, 10, -10);
    lv_obj_set_style_bg_color(btn_ok, lv_color_hex(COLOR_GREEN), 0); lv_obj_add_event_cb(btn_ok, btn_cash_ok_cb, LV_EVENT_CLICKED, NULL);
    lv_obj_t *lok = lv_label_create(btn_ok); lv_label_set_text(lok, "OK"); lv_obj_center(lok);
    lv_obj_t *btn_cancel = lv_btn_create(modal_cash); lv_obj_set_size(btn_cancel, 100, 40); lv_obj_align(btn_cancel, LV_ALIGN_BOTTOM_RIGHT, -10, -10);
    lv_obj_set_style_bg_color(btn_cancel, lv_color_hex(COLOR_RED), 0); lv_obj_add_event_cb(btn_cancel, btn_cash_cancel_cb, LV_EVENT_CLICKED, NULL);
    lv_obj_t *lcan = lv_label_create(btn_cancel); lv_label_set_text(lcan, "Cancel"); lv_obj_center(lcan);
  }, LV_EVENT_CLICKED, NULL);
  lv_obj_t *lc = lv_label_create(btn_cash); lv_label_set_text(lc, "CASH"); lv_obj_center(lc);

  btn_pay_qr = lv_btn_create(page_pos); lv_obj_set_size(btn_pay_qr, 145, 48); lv_obj_align(btn_pay_qr, LV_ALIGN_BOTTOM_RIGHT, -10, -10);
  lv_obj_set_style_bg_color(btn_pay_qr, lv_color_hex(COLOR_ORANGE), 0);
  lv_obj_add_event_cb(btn_pay_qr, [](lv_event_t *e){
    if(current_amount <= 0 || !is_qr_node_online || modal_cash || modal_qr || modal_shift) return;
    sendEspNowPacket(CMD_SHOW_QR, current_amount);
    modal_qr = lv_obj_create(lv_scr_act()); lv_obj_set_size(modal_qr, 260, 180); lv_obj_center(modal_qr);
    lv_obj_set_style_bg_color(modal_qr, lv_color_hex(COLOR_CARD), 0); lv_obj_set_style_bg_opa(modal_qr, LV_OPA_COVER, 0); lv_obj_set_style_border_color(modal_qr, lv_color_hex(COLOR_ORANGE), 0); lv_obj_set_style_border_width(modal_qr, 2, 0);
    lv_obj_t *lbl_title = lv_label_create(modal_qr); lv_label_set_text(lbl_title, "Waiting QR Payment..."); lv_obj_set_style_text_color(lbl_title, lv_color_hex(COLOR_WHITE), 0); lv_obj_align(lbl_title, LV_ALIGN_TOP_MID, 0, 10);
    lv_obj_t *lbl_amt = lv_label_create(modal_qr); char m_buf[32]; snprintf(m_buf, sizeof(m_buf), "%.0f THB", current_amount);
    lv_label_set_text(lbl_amt, m_buf); lv_obj_set_style_text_color(lbl_amt, lv_color_hex(COLOR_YELLOW), 0); lv_obj_set_style_text_font(lbl_amt, &lv_font_montserrat_20, 0); lv_obj_align(lbl_amt, LV_ALIGN_CENTER, 0, -15);
    lv_obj_t *btn_force_ok = lv_btn_create(modal_qr); lv_obj_set_size(btn_force_ok, 100, 45); lv_obj_align(btn_force_ok, LV_ALIGN_BOTTOM_LEFT, 10, -10);
    lv_obj_set_style_bg_color(btn_force_ok, lv_color_hex(COLOR_GREEN), 0); lv_obj_add_event_cb(btn_force_ok, btn_qr_manual_ok_cb, LV_EVENT_CLICKED, NULL);
    lv_obj_t *lok = lv_label_create(btn_force_ok); lv_label_set_text(lok, "Manual OK"); lv_obj_center(lok);
    lv_obj_t *btn_cancel = lv_btn_create(modal_qr); lv_obj_set_size(btn_cancel, 100, 45); lv_obj_align(btn_cancel, LV_ALIGN_BOTTOM_RIGHT, -10, -10);
    lv_obj_set_style_bg_color(btn_cancel, lv_color_hex(COLOR_RED), 0); lv_obj_add_event_cb(btn_cancel, btn_qr_cancel_cb, LV_EVENT_CLICKED, NULL);
    lv_obj_t *lcan = lv_label_create(btn_cancel); lv_label_set_text(lcan, "Cancel"); lv_obj_center(lcan);
  }, LV_EVENT_CLICKED, NULL);
  lv_obj_t *lq = lv_label_create(btn_pay_qr); lv_label_set_text(lq, "SCAN QR"); lv_obj_center(lq);

  // หน้า 2: History
  page_history = lv_obj_create(NULL); lv_obj_set_style_bg_color(page_history, lv_color_hex(COLOR_BG), 0);
  lv_obj_set_style_bg_opa(page_history, LV_OPA_COVER, 0); lv_obj_add_event_cb(page_history, screen_gesture_event_cb, LV_EVENT_GESTURE, NULL);
  lv_obj_t *h_hist = lv_label_create(page_history); lv_label_set_text(h_hist, "2: TRANSACTION HISTORY"); lv_obj_set_style_text_color(h_hist, lv_color_hex(COLOR_CYAN), 0); lv_obj_align(h_hist, LV_ALIGN_TOP_LEFT, 10, 8);
  lv_obj_t *panel_hist = lv_obj_create(page_history); lv_obj_set_size(panel_hist, 304, 185); lv_obj_align(panel_hist, LV_ALIGN_BOTTOM_MID, 0, -5);
  lv_obj_set_style_bg_color(panel_hist, lv_color_hex(COLOR_PANEL), 0); lv_obj_set_style_border_color(panel_hist, lv_color_hex(COLOR_BORDER), 0);
  lbl_history_text = lv_label_create(panel_hist); lv_label_set_text(lbl_history_text, "No transaction history.\n"); lv_obj_set_style_text_color(lbl_history_text, lv_color_hex(COLOR_WHITE), 0); lv_obj_set_style_text_font(lbl_history_text, &lv_font_montserrat_12, 0);

  // หน้า 3: Summary
  page_sum = lv_obj_create(NULL); lv_obj_set_style_bg_color(page_sum, lv_color_hex(COLOR_BG), 0);
  lv_obj_set_style_bg_opa(page_sum, LV_OPA_COVER, 0); lv_obj_add_event_cb(page_sum, screen_gesture_event_cb, LV_EVENT_GESTURE, NULL);
  lv_obj_t *h_sum = lv_label_create(page_sum); lv_label_set_text(h_sum, "3: SALES SUMMARY & TELEGRAM"); lv_obj_set_style_text_color(h_sum, lv_color_hex(COLOR_YELLOW), 0); lv_obj_align(h_sum, LV_ALIGN_TOP_LEFT, 10, 8);
  lv_obj_t *panel_sum = lv_obj_create(page_sum); lv_obj_set_size(panel_sum, 304, 95); lv_obj_align(panel_sum, LV_ALIGN_TOP_MID, 0, 35); lv_obj_set_style_bg_color(panel_sum, lv_color_hex(COLOR_CARD), 0); lv_obj_set_style_border_color(panel_sum, lv_color_hex(COLOR_BORDER), 0);
  lbl_sum_cash = lv_label_create(panel_sum); lv_obj_set_style_text_color(lbl_sum_cash, lv_color_hex(COLOR_WHITE), 0); lv_obj_align(lbl_sum_cash, LV_ALIGN_TOP_LEFT, 8, 6);
  lbl_sum_qr = lv_label_create(panel_sum); lv_obj_set_style_text_color(lbl_sum_qr, lv_color_hex(COLOR_WHITE), 0); lv_obj_align(lbl_sum_qr, LV_ALIGN_TOP_LEFT, 8, 30);
  lbl_sum_total = lv_label_create(panel_sum); lv_obj_set_style_text_color(lbl_sum_total, lv_color_hex(COLOR_GREEN), 0); lv_obj_align(lbl_sum_total, LV_ALIGN_TOP_LEFT, 8, 58);
  lbl_tg_status = lv_label_create(page_sum); lv_label_set_text(lbl_tg_status, ""); lv_obj_set_style_text_color(lbl_tg_status, lv_color_hex(COLOR_CYAN), 0); lv_obj_align(lbl_tg_status, LV_ALIGN_BOTTOM_MID, 0, -52);
  lv_obj_t *btn_tg = lv_btn_create(page_sum); lv_obj_set_size(btn_tg, 280, 42); lv_obj_align(btn_tg, LV_ALIGN_BOTTOM_MID, 0, -8); lv_obj_set_style_bg_color(btn_tg, lv_color_hex(COLOR_BLUE), 0);
  lv_obj_add_event_cb(btn_tg, btn_shift_close_event_cb, LV_EVENT_CLICKED, NULL);
  lv_obj_t *ltg = lv_label_create(btn_tg); lv_label_set_text(ltg, "CLOSE SHIFT / SUMMARY"); lv_obj_set_style_text_color(ltg, lv_color_hex(COLOR_WHITE), 0); lv_obj_center(ltg);
}

void setup() {
  Serial.begin(115200);
  LittleFS.begin(true);
  load_settings();
  rtc_ok = rtc.begin();
  
  pinMode(TFT_BL, OUTPUT); analogWrite(TFT_BL, 230); 
  tft.begin(); tft.invertDisplay(false); tft.setRotation(1); tft.fillScreen(TFT_BLACK);

  touchSPI.begin(XPT2046_CLK, XPT2046_MISO, XPT2046_MOSI, XPT2046_CS);
  ts.begin(touchSPI); ts.setRotation(1);

  lv_init();
  static lv_disp_draw_buf_t draw_buf; static lv_color_t buf[320 * 10]; 
  lv_disp_draw_buf_init(&draw_buf, buf, NULL, 320 * 10);
  static lv_disp_drv_t disp_drv; lv_disp_drv_init(&disp_drv);
  disp_drv.hor_res = 320; disp_drv.ver_res = 240; disp_drv.flush_cb = my_disp_flush; disp_drv.draw_buf = &draw_buf;
  lv_disp_drv_register(&disp_drv);
  static lv_indev_drv_t indev_drv; lv_indev_drv_init(&indev_drv);
  indev_drv.type = LV_INDEV_TYPE_POINTER; indev_drv.read_cb = my_touchpad_read; lv_indev_drv_register(&indev_drv);

  // ตั้งค่า Wi-Fi และ ESP-NOW
  WiFi.mode(WIFI_STA);
  esp_wifi_set_promiscuous(true); esp_wifi_set_channel(1, WIFI_SECOND_CHAN_NONE); esp_wifi_set_promiscuous(false);
  esp_now_init();
  esp_now_register_send_cb((esp_now_send_cb_t)OnDataSent);
  esp_now_register_recv_cb((esp_now_recv_cb_t)OnDataRecv);
  esp_now_peer_info_t peerInfo = {};
  memcpy(peerInfo.peer_addr, broadcastAddress, 6);
  peerInfo.channel = 0; peerInfo.encrypt = false;
  esp_now_add_peer(&peerInfo);

  createScreens();

  // 1. แสดงโลโก้ POS 2 วินาที (แก้ปัญหาหน้าจอโลโก้หาย)
  lv_scr_load(scr_logo);
  lv_timer_handler();
  delay(2000);

  // 2. โหลดเข้าหน้าจอ Config นับถอยหลัง 3 วินาที
  lv_scr_load(scr_config);
  config_interrupted = false;

  for (int i = 3; i > 0; i--) {
    char buf_cd[32]; snprintf(buf_cd, sizeof(buf_cd), "Starting POS in %ds...", i);
    lv_label_set_text(lbl_config_countdown, buf_cd);
    uint32_t t_start = millis();
    while (millis() - t_start < 1000) { lv_timer_handler(); lv_tick_inc(10); if (config_interrupted) break; delay(10); }
    if (config_interrupted) break;
  }

  // 3. ถ้าผู้ใช้กดปุ่ม Config
  if (config_interrupted) {
    beep(100); lv_scr_load(scr_ip_display); lv_timer_handler();
    WiFiManager wm; wm.setConfigPortalTimeout(180);
    wm.setAPStaticIPConfig(IPAddress(192,168,4,1), IPAddress(192,168,4,1), IPAddress(255,255,255,0));
    
    WiFiManagerParameter custom_shop("shop", "Shop Name", cfg_shop_name, 40);
    WiFiManagerParameter custom_pp("pp", "PromptPay ID", cfg_promptpay, 20);
    WiFiManagerParameter custom_token("token", "Telegram Token", cfg_bot_token, 60);
    WiFiManagerParameter custom_chat("chat", "Chat ID", cfg_chat_id, 30);
    wm.addParameter(&custom_shop); wm.addParameter(&custom_pp); wm.addParameter(&custom_token); wm.addParameter(&custom_chat);
    
    wm.setSaveConfigCallback([&](){
      strlcpy(cfg_shop_name, custom_shop.getValue(), sizeof(cfg_shop_name));
      strlcpy(cfg_promptpay, custom_pp.getValue(), sizeof(cfg_promptpay));
      strlcpy(cfg_bot_token, custom_token.getValue(), sizeof(cfg_bot_token));
      strlcpy(cfg_chat_id, custom_chat.getValue(), sizeof(cfg_chat_id));
      preferences.begin("pos_cfg", false);
      preferences.putString("shop", cfg_shop_name); preferences.putString("pp", cfg_promptpay);
      preferences.putString("token", cfg_bot_token); preferences.putString("chatid", cfg_chat_id);
      preferences.end();
    });

    if (!wm.startConfigPortal("POS_Config_AP")) { delay(2000); ESP.restart(); }
    ESP.restart();
  }

  // 4. เข้าสู่หน้าจอ SYSTEM DIAGNOSTICS (เช็คเฉพาะ RTC และ Wi-Fi)
  lv_scr_load(page_boot);
  current_state = STATE_BOOT_START;
  state_timer = millis();

  WiFiManager wm; wm.setConnectTimeout(10); wm.autoConnect("POS_Config_AP");

  WiFi.disconnect(); WiFi.mode(WIFI_STA);
  esp_wifi_set_promiscuous(true); esp_wifi_set_channel(1, WIFI_SECOND_CHAN_NONE); esp_wifi_set_promiscuous(false);

  heartbeat_timer = lv_timer_create(heartbeat_timer_cb, 3000, NULL);
  last_tick = millis();
}

void loop() {
  uint32_t current_tick = millis();
  lv_tick_inc(current_tick - last_tick);
  last_tick = current_tick;
  lv_timer_handler(); 

  unsigned long current_millis = millis();

  // ตรวจจับการเปลี่ยนแปลงสถานะการเชื่อมต่อ QR เพื่อส่งเสียง Buzzer ตามเงื่อนไข (ยาว 1 / ยาว 2)
  if (is_qr_node_online != last_online_state) {
    last_online_state = is_qr_node_online;
    if (is_qr_node_online) {
      beepAlert(2); // เชื่อมต่อได้: ดังยาว 2 ครั้ง
    } else {
      beepAlert(1); // หลุดการเชื่อมต่อ: ดังยาว 1 ครั้ง
    }
  }

  switch (current_state) {
    case STATE_BOOT_START:
      if (current_millis - state_timer >= 500) {
        lv_label_set_text(lbl_boot_rtc, rtc_ok ? "1. RTC Check ... [OK]" : "1. RTC Check ... [FAIL]");
        lv_obj_set_style_text_color(lbl_boot_rtc, rtc_ok ? lv_color_hex(COLOR_GREEN) : lv_color_hex(COLOR_RED), 0);
        state_timer = current_millis;
        current_state = STATE_BOOT_HW_OK;
      }
      break;
      
    case STATE_BOOT_HW_OK:
      if (current_millis - state_timer >= 1000) { 
        if (WiFi.status() == WL_CONNECTED) {
          lv_label_set_text(lbl_boot_wifi, "2. Wi-Fi & Time ... [OK]");
          lv_obj_set_style_text_color(lbl_boot_wifi, lv_color_hex(COLOR_GREEN), 0);
        } else {
          lv_label_set_text(lbl_boot_wifi, "2. Wi-Fi & Time ... [OFFLINE]");
          lv_obj_set_style_text_color(lbl_boot_wifi, lv_color_hex(COLOR_ORANGE), 0);
        }

        lv_obj_clear_state(btn_start_pos, LV_STATE_DISABLED); 
        current_state = STATE_WAIT_START;
      }
      break;

    case STATE_POS_READY:
      if (is_qr_node_online && (current_millis - last_pong_time > 6000)) {
        is_qr_node_online = false;
        memcpy(targetMac, broadcastAddress, 6); 
      }

      if (current_millis - last_clock_update > 1000) {
        last_clock_update = current_millis;
        if (active_screen == 1 && rtc_ok) {
          DateTime now = rtc.now();
          char tbuf[16]; snprintf(tbuf, sizeof(tbuf), "%02d:%02d", now.hour(), now.minute());
          lv_label_set_text(lbl_pos_time, tbuf);
        }
      }

      if(is_qr_node_online) {
        lv_label_set_text(lbl_qr_status_text, "QR Ready");
        lv_obj_set_style_text_color(lbl_qr_status_text, lv_color_hex(COLOR_GREEN), 0);
        lv_obj_clear_state(btn_pay_qr, LV_STATE_DISABLED);
      } else {
        lv_label_set_text(lbl_qr_status_text, "QR Offline");
        lv_obj_set_style_text_color(lbl_qr_status_text, lv_color_hex(COLOR_RED), 0);
        lv_obj_add_state(btn_pay_qr, LV_STATE_DISABLED);
      }

      if (flag_qr_paid_received) {
        flag_qr_paid_received = false;
        if (current_amount > 0) {
          total_qr += current_amount; 
          recordSale("QR", current_amount); 
          current_amount = 0; 
          updateAmountDisplay();
        }
        if (modal_qr) { lv_obj_del(modal_qr); modal_qr = NULL; }
      }
      break;
  }

  delay(5);
}
