/*
 * ============================================================================
 * PROJECT: Dx Solution HS3PIK - Dual Screen POS System
 * NODE 2 : QR CUSTOMER DISPLAY NODE (Auto-Discovery, No Decimals, Buzzer)
 * ============================================================================
 */

#include <Arduino.h>
#include <SPI.h>
#include <WiFi.h>
#include <esp_now.h>
#include <esp_wifi.h>
#include <TFT_eSPI.h>
#include <XPT2046_Touchscreen.h>
#include <lvgl.h>

uint8_t posMasterMac[6];
bool pos_paired = false;

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

#define COLOR_BG        0x0A0F1D  
#define COLOR_PANEL     0x111827  
#define COLOR_CARD      0x1F2937  
#define COLOR_CYAN      0x00F5D4  
#define COLOR_GREEN     0x2ECC71  
#define COLOR_BLUE      0x3B82F6  
#define COLOR_ORANGE    0xE67E22  
#define COLOR_YELLOW    0xF1C40F  
#define COLOR_RED       0xEF4444  
#define COLOR_WHITE     0xFFFFFF  
#define COLOR_GRAY      0x6B7280  

enum PacketCommand : uint8_t { CMD_PING=1, CMD_SHOW_QR=2, CMD_CANCEL_QR=3, CMD_PAID_OK=4 };
typedef struct __attribute__((packed)) {
  uint8_t command;
  float amount;
  char promptpay_id[20];
  char shop_name[40];
} esp_now_payload_t;

esp_now_payload_t inMsg, outMsg;
volatile bool flag_show_qr = false;
volatile bool flag_cancel_qr = false;

lv_obj_t *scr_standby, *scr_qr, *scr_thankyou;
lv_obj_t *qr_widget = NULL;
lv_obj_t *lbl_qr_amount, *btn_confirm_paid;
uint32_t last_tick = 0;

uint16_t calcCRC16(const char *data, size_t length) {
  uint16_t crc = 0xFFFF;
  for (size_t i = 0; i < length; i++) {
    crc ^= ((uint16_t)data[i] << 8);
    for (uint8_t j = 0; j < 8; j++) {
      if (crc & 0x8000) crc = (crc << 1) ^ 0x1021;
      else crc <<= 1;
    }
  }
  return crc;
}

String generatePromptPayPayload(String targetId, float amount) {
  targetId.trim();
  String merchantAccount = "";
  if (targetId.length() == 10 && targetId.startsWith("0")) {
    String formattedPhone = "0066" + targetId.substring(1);
    merchantAccount = "0016A0000006770101110113" + formattedPhone;
  } else if (targetId.length() == 13) {
    merchantAccount = "0016A0000006770101110213" + targetId;
  } else {
    char lenBuf[4]; snprintf(lenBuf, sizeof(lenBuf), "%02d", (int)targetId.length());
    merchantAccount = "0016A00000067701011101" + String(lenBuf) + targetId;
  }
  char tag29Len[4]; snprintf(tag29Len, sizeof(tag29Len), "%02d", (int)merchantAccount.length());
  String payload = "000201010212";
  payload += "29" + String(tag29Len) + merchantAccount;
  payload += "5802TH5303764";
  
  if (amount > 0.0) {
    String amtStr = String(amount, 2); // มาตรฐาน PromptPay ใช้ทศนิยม 2 ตำแหน่ง
    char amtLen[4]; snprintf(amtLen, sizeof(amtLen), "%02d", (int)amtStr.length());
    payload += "54" + String(amtLen) + amtStr;
  }
  
  payload += "6304";
  uint16_t crc = calcCRC16(payload.c_str(), payload.length());
  char crcHex[6]; snprintf(crcHex, sizeof(crcHex), "%04X", crc);
  payload += String(crcHex);
  return payload;
}

// ฟังก์ชันเสียง Buzzer เครื่องที่ 2 (สั้น 1 / สั้น 3)
void beepAlert(int type) {
  pinMode(BUZZER_PIN, OUTPUT);
  if (type == 1) { // รับยอด: ดัง 1 ครั้งสั้นๆ (80ms)
    digitalWrite(BUZZER_PIN, HIGH); delay(80); digitalWrite(BUZZER_PIN, LOW);
  } else if (type == 3) { // กด Paid Confirm: ดัง 3 ครั้งสั้นๆ
    for (int i = 0; i < 3; i++) {
      digitalWrite(BUZZER_PIN, HIGH); delay(80); digitalWrite(BUZZER_PIN, LOW);
      if (i < 2) delay(80);
    }
  }
}

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

#if ESP_ARDUINO_VERSION >= ESP_ARDUINO_VERSION_VAL(3, 0, 0)
void OnDataRecv(const esp_now_recv_info_t *info, const uint8_t *data, int len) {
  memcpy(posMasterMac, info->src_addr, 6);
#else
void OnDataRecv(const uint8_t *mac, const uint8_t *data, int len) {
  memcpy(posMasterMac, mac, 6);
#endif
  pos_paired = true;
  if (!esp_now_is_peer_exist(posMasterMac)) {
    esp_now_peer_info_t peerInfo = {};
    memcpy(peerInfo.peer_addr, posMasterMac, 6);
    peerInfo.channel = 0; peerInfo.encrypt = false;
    esp_now_add_peer(&peerInfo);
  }

  if (len == sizeof(esp_now_payload_t)) {
    memcpy(&inMsg, data, sizeof(esp_now_payload_t));
    if (inMsg.command == CMD_PING) {
      esp_now_payload_t pongMsg; pongMsg.command = CMD_PING;
      esp_now_send(posMasterMac, (uint8_t *)&pongMsg, sizeof(pongMsg));
    }
    else if (inMsg.command == CMD_SHOW_QR) flag_show_qr = true;
    else if (inMsg.command == CMD_CANCEL_QR) flag_cancel_qr = true;
  }
}

static void return_standby_timer_cb(lv_timer_t *timer) {
  lv_scr_load(scr_standby); lv_timer_del(timer);
}

static void btn_paid_event_cb(lv_event_t *e) {
  if (!pos_paired) return;
  
  beepAlert(3); // ดัง 3 ครั้งสั้นๆ
  lv_obj_add_state(btn_confirm_paid, LV_STATE_DISABLED);
  
  outMsg.command = CMD_PAID_OK; outMsg.amount  = inMsg.amount;
  esp_now_send(posMasterMac, (uint8_t *)&outMsg, sizeof(outMsg));
  
  lv_scr_load(scr_thankyou);
  lv_timer_create(return_standby_timer_cb, 3000, NULL);
}

void buildUI() {
  scr_standby = lv_obj_create(NULL); lv_obj_set_style_bg_color(scr_standby, lv_color_hex(COLOR_BG), 0);
  lv_obj_set_style_bg_opa(scr_standby, LV_OPA_COVER, 0); lv_obj_clear_flag(scr_standby, LV_OBJ_FLAG_SCROLLABLE);
  lv_obj_t *lbl_st = lv_label_create(scr_standby); lv_label_set_text(lbl_st, "WELCOME\nDx Solution POS\n\nWaiting for order...");
  lv_obj_set_style_text_align(lbl_st, LV_TEXT_ALIGN_CENTER, 0); lv_obj_set_style_text_color(lbl_st, lv_color_hex(COLOR_WHITE), 0); lv_obj_center(lbl_st);

  scr_qr = lv_obj_create(NULL); lv_obj_set_style_bg_color(scr_qr, lv_color_hex(COLOR_BG), 0);
  lv_obj_set_style_bg_opa(scr_qr, LV_OPA_COVER, 0); lv_obj_clear_flag(scr_qr, LV_OBJ_FLAG_SCROLLABLE);
  
  lbl_qr_amount = lv_label_create(scr_qr); lv_label_set_text(lbl_qr_amount, "SCAN TO PAY\n\n0 THB");
  lv_obj_set_style_text_align(lbl_qr_amount, LV_TEXT_ALIGN_CENTER, 0); lv_obj_set_style_text_color(lbl_qr_amount, lv_color_hex(COLOR_CYAN), 0); 
  lv_obj_set_style_text_font(lbl_qr_amount, &lv_font_montserrat_20, 0);
  lv_obj_align(lbl_qr_amount, LV_ALIGN_TOP_RIGHT, -15, 30);
  
  btn_confirm_paid = lv_btn_create(scr_qr); lv_obj_set_size(btn_confirm_paid, 110, 65); lv_obj_align(btn_confirm_paid, LV_ALIGN_BOTTOM_RIGHT, -12, -20);
  lv_obj_set_style_bg_color(btn_confirm_paid, lv_color_hex(COLOR_GREEN), 0); lv_obj_add_event_cb(btn_confirm_paid, btn_paid_event_cb, LV_EVENT_CLICKED, NULL);
  lv_obj_t *lbtn = lv_label_create(btn_confirm_paid); lv_label_set_text(lbtn, "PAID\nCONFIRM"); lv_obj_set_style_text_align(lbtn, LV_TEXT_ALIGN_CENTER, 0); lv_obj_center(lbtn);

  scr_thankyou = lv_obj_create(NULL); lv_obj_set_style_bg_color(scr_thankyou, lv_color_hex(COLOR_BG), 0);
  lv_obj_set_style_bg_opa(scr_thankyou, LV_OPA_COVER, 0); lv_obj_clear_flag(scr_thankyou, LV_OBJ_FLAG_SCROLLABLE);
  
  lv_obj_t *lbl_ty = lv_label_create(scr_thankyou); lv_label_set_text(lbl_ty, "THANK YOU!\nPayment Successful");
  lv_obj_set_style_text_align(lbl_ty, LV_TEXT_ALIGN_CENTER, 0); lv_obj_set_style_text_color(lbl_ty, lv_color_hex(COLOR_GREEN), 0); lv_obj_center(lbl_ty);

  lv_scr_load(scr_standby);
}

void setup() {
  Serial.begin(115200);
  pinMode(TFT_BL, OUTPUT); analogWrite(TFT_BL, 230); 
  pinMode(BUZZER_PIN, OUTPUT); digitalWrite(BUZZER_PIN, LOW);

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

  WiFi.mode(WIFI_STA);
  WiFi.disconnect();
  esp_wifi_set_promiscuous(true);
  esp_wifi_set_channel(1, WIFI_SECOND_CHAN_NONE);
  esp_wifi_set_promiscuous(false);

  if (esp_now_init() == ESP_OK) { esp_now_register_recv_cb((esp_now_recv_cb_t)OnDataRecv); }

  buildUI();
  last_tick = millis();
}

void loop() {
  uint32_t current_tick = millis();
  lv_tick_inc(current_tick - last_tick);
  last_tick = current_tick;
  lv_timer_handler();

  if (flag_show_qr) {
    flag_show_qr = false;
    
    beepAlert(1); // ดัง 1 ครั้งสั้นๆ
    
    if (qr_widget != NULL) { lv_obj_del(qr_widget); qr_widget = NULL; }
    String payload = generatePromptPayPayload(String(inMsg.promptpay_id), inMsg.amount);
    qr_widget = lv_qrcode_create(scr_qr, 175, lv_color_hex(0x000000), lv_color_hex(0xFFFFFF));
    lv_obj_align(qr_widget, LV_ALIGN_LEFT_MID, 12, 0); lv_obj_set_style_border_color(qr_widget, lv_color_hex(COLOR_WHITE), 0); lv_obj_set_style_border_width(qr_widget, 4, 0);
    lv_qrcode_update(qr_widget, payload.c_str(), payload.length());
    
    char amtBuf[64]; 
    // แสดงผลเฉพาะจำนวนเต็ม (%.0f) ไม่ให้ตัวเลขล้นจอ
    snprintf(amtBuf, sizeof(amtBuf), "%s\n\n%.0f THB", inMsg.shop_name, inMsg.amount);
    lv_label_set_text(lbl_qr_amount, amtBuf);
    
    lv_obj_clear_state(btn_confirm_paid, LV_STATE_DISABLED);
    lv_scr_load(scr_qr);
  }

  if (flag_cancel_qr) { flag_cancel_qr = false; lv_scr_load(scr_standby); }
  delay(5);
}
