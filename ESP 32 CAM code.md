code:

```
// ============================================================
// ESP32-CAM (AI-Thinker board) - Live video for the robot car
//
// This board does ONE job: join the car's WiFi network and
// serve an MJPEG stream at http://<this-ip>/stream
//
// Flashing: you need an FTDI USB-serial adapter.
//   FTDI TX  -> CAM U0R
//   FTDI RX  -> CAM U0T
//   FTDI 5V  -> CAM 5V
//   FTDI GND -> CAM GND
//   CAM GPIO0 -> GND  (ONLY while uploading, remove after)
// Board setting in Arduino IDE: "AI Thinker ESP32-CAM"
// ============================================================

#include "esp_camera.h"
#include <WiFi.h>
#include <WebServer.h>

// ===== AI-Thinker ESP32-CAM pin map (fixed by the board itself) =====
#define PWDN_GPIO_NUM     32
#define RESET_GPIO_NUM    -1
#define XCLK_GPIO_NUM      0
#define SIOD_GPIO_NUM     26
#define SIOC_GPIO_NUM     27
#define Y9_GPIO_NUM       35
#define Y8_GPIO_NUM       34
#define Y7_GPIO_NUM       39
#define Y6_GPIO_NUM       36
#define Y5_GPIO_NUM       21
#define Y4_GPIO_NUM       19
#define Y3_GPIO_NUM       18
#define Y2_GPIO_NUM        5
#define VSYNC_GPIO_NUM    25
#define HREF_GPIO_NUM     23
#define PCLK_GPIO_NUM     22

// ===== Join the car's WiFi AP =====
const char* ssid     = "ESP32_Robot_Car";
const char* password = "12345678";

// Static IP on the car's AP subnet (car's softAP gateway is 192.168.4.1).
// Picked outside the default DHCP pool so the phone can't be handed this
// same address. Must match CAM_BASE_URL in the car board's sketch.
IPAddress local_IP(192, 168, 4, 50);
IPAddress gateway(192, 168, 4, 1);
IPAddress subnet(255, 255, 255, 0);

WebServer server(80);

void handleStream() {
  WiFiClient client = server.client();
  String response = "HTTP/1.1 200 OK\r\n";
  response += "Content-Type: multipart/x-mixed-replace; boundary=frame\r\n\r\n";
  client.print(response);

  while (client.connected()) {
    camera_fb_t * fb = esp_camera_fb_get();
    if (!fb) { continue; }

    client.print("\r\n--frame\r\n");
    char header[64];
    size_t hlen = snprintf(header, sizeof(header),
                            "Content-Type: image/jpeg\r\nContent-Length: %u\r\n\r\n", fb->len);
    client.write(header, hlen);
    client.write(fb->buf, fb->len);

    esp_camera_fb_return(fb);
    if (!client.connected()) break;
  }
}

void handleCapture() {
  camera_fb_t * fb = esp_camera_fb_get();
  if (!fb) {
    server.send(500, "text/plain", "capture failed");
    return;
  }
  server.sendHeader("Cache-Control", "no-cache, no-store, must-revalidate");
  server.sendHeader("Content-Type", "image/jpeg");
  server.send_P(200, "image/jpeg", (const char*)fb->buf, fb->len);
  esp_camera_fb_return(fb);
}

void setup() {
  Serial.begin(115200);

  camera_config_t config;
  config.ledc_channel = LEDC_CHANNEL_0;
  config.ledc_timer   = LEDC_TIMER_0;
  config.pin_d0 = Y2_GPIO_NUM;  config.pin_d1 = Y3_GPIO_NUM;
  config.pin_d2 = Y4_GPIO_NUM;  config.pin_d3 = Y5_GPIO_NUM;
  config.pin_d4 = Y6_GPIO_NUM;  config.pin_d5 = Y7_GPIO_NUM;
  config.pin_d6 = Y8_GPIO_NUM;  config.pin_d7 = Y9_GPIO_NUM;
  config.pin_xclk = XCLK_GPIO_NUM;
  config.pin_pclk = PCLK_GPIO_NUM;
  config.pin_vsync = VSYNC_GPIO_NUM;
  config.pin_href = HREF_GPIO_NUM;
  config.pin_sscb_sda = SIOD_GPIO_NUM;
  config.pin_sscb_scl = SIOC_GPIO_NUM;
  config.pin_pwdn = PWDN_GPIO_NUM;
  config.pin_reset = RESET_GPIO_NUM;
  config.xclk_freq_hz = 20000000;
  config.pixel_format = PIXFORMAT_JPEG;

  if (psramFound()) {
    config.frame_size = FRAMESIZE_QVGA; // 320x240 - lighter load, smoother on marginal power
    config.jpeg_quality = 14;
    config.fb_count = 2;
  } else {
    config.frame_size = FRAMESIZE_QQVGA; // 160x120 fallback, no PSRAM
    config.jpeg_quality = 16;
    config.fb_count = 1;
  }

  esp_err_t err = esp_camera_init(&config);
  if (err != ESP_OK) {
    Serial.printf("Camera init failed with error 0x%x\n", err);
    return;
  }

  WiFi.mode(WIFI_STA);
  WiFi.config(local_IP, gateway, subnet);
  WiFi.begin(ssid, password);
  Serial.print("Connecting to car WiFi");
  while (WiFi.status() != WL_CONNECTED) {
    delay(300);
    Serial.print(".");
  }
  Serial.println();
  Serial.print("Camera ready at http://");
  Serial.println(WiFi.localIP());
  Serial.println("Stream:  /stream");
  Serial.println("Snapshot: /capture");

  server.on("/stream", HTTP_GET, handleStream);
  server.on("/capture", HTTP_GET, handleCapture);
  server.begin();
}

void loop() {
  server.handleClient();
}
```
