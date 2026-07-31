#ESP 32 CODE

```
#include <WiFi.h>
#include <Wire.h>
#include <HardwareSerial.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

// --- OLED Settings ---
#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_RESET    -1
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

// --- Wi-Fi Settings ---
const char* ssid = "ESP32_Robot_Car";
const char* password = "12345678";
WiFiServer server(80);

// --- ESP32-CAM location on this same network ---
const char* CAM_BASE_URL = "http://192.168.4.50";

// --- Motor Pins & PWM ---
const int IN1 = 25;
const int IN2 = 26;
const int IN3 = 27;
const int IN4 = 14;
const int freq = 30000;
const int resolution = 8;
const bool REVERSE_DRIVE = true;

int currentSpeed = 200;
String currentText = "";

// --- MP3-TF-16P audio module (UART2: GPIO16=RX2, GPIO17=TX2) ---
HardwareSerial mp3Serial(2);

// --- Expression / face states ---
#define FACE_HAPPY     0
#define FACE_ANGRY     1
#define FACE_SLEEPY    2
#define FACE_TEXT      3
#define FACE_LOVE      4
#define FACE_FIRE      5
#define FACE_WINK      6
#define FACE_COOL      7
#define FACE_SURPRISED 8
int faceState = FACE_HAPPY;

// --- Non-blocking turn state ---
bool turning = false;
unsigned long turnStartTime = 0;
unsigned long turnDuration = 0;

// --- Function prototypes (HTML raw string below breaks IDE auto-prototyping) ---
void drawHappyFace();
void drawAngryFace();
void drawSleepyFace();
void drawCustomTextScreen(String text);
void drawHeart(int cx, int cy, int size);
void drawLoveFace();
void drawFireFace();
void drawWinkFace();
void drawCoolFace();
void drawSurprisedFace();
void setMotors(int left, int right);
void moveForward(int spd);
void moveBackward(int spd);
void turnLeft(int spd);
void turnRight(int spd);
void spinLeft(int spd);
void spinRight(int spd);
void stopRobot();
void startTurn(int spd, unsigned long duration);
void mp3SendCmd(uint8_t cmd, uint16_t param);
void mp3Play(uint16_t track);
void mp3SetVolume(uint8_t vol);
void mp3Stop();
String urlDecode(String input);

// --- Web Interface: split-screen layout for tablets ---
// Top 50% = camera (always visible). Bottom 50% = every control at once,
// laid out in columns (joystick | drive buttons | faces | sound/sliders).
const char index_html_part1[] PROGMEM = R"rawliteral(
<!DOCTYPE html><html><head><meta name="viewport" content="width=device-width, initial-scale=1, user-scalable=no">
<style>
:root { --bg:#0a0a0d; --panel:#131318; --line:#232329; --neon:#00ffcc; --gold:#ffaa00; --red:#ff3366; --muted:#7a7a85; }
* { box-sizing:border-box; }
html,body { height:100%; margin:0; overflow:hidden; }
body { font-family:'Segoe UI',Tahoma,sans-serif; background:var(--bg); color:#fff; display:flex; flex-direction:column; }

/* ---- TOP HALF: camera ---- */
#camPane { position:relative; height:50vh; background:#000; flex-shrink:0; }
#camStream { width:100%; height:100%; object-fit:cover; display:block; }
#camStatus { position:absolute; top:8px; right:10px; font-size:11px; padding:4px 10px; border-radius:20px; background:rgba(0,0,0,0.6); color:var(--muted); letter-spacing:1px; }
#camStatus.live { color:var(--neon); }
#camStatus.down { color:var(--red); }
#bigStop { position:absolute; bottom:10px; right:10px; background:var(--red); color:#000; border:none; font-weight:bold; font-size:14px; letter-spacing:1px; padding:12px 20px; border-radius:10px; box-shadow:0 0 14px rgba(255,51,102,0.6); }
#bigStop:active { transform:scale(0.95); }

/* ---- BOTTOM HALF: all controls, side by side ---- */
#controls { height:50vh; display:flex; gap:6px; padding:8px; overflow-x:auto; overflow-y:hidden; }
.col { background:var(--panel); border:1px solid var(--line); border-radius:12px; padding:8px; flex:1 1 0; min-width:150px; display:flex; flex-direction:column; overflow-y:auto; }
.colTitle { font-size:10px; color:var(--muted); text-transform:uppercase; letter-spacing:1.5px; margin-bottom:6px; text-align:center; }

button { background:#181820; color:var(--neon); border:1px solid var(--neon); border-radius:8px; font-weight:bold; cursor:pointer; }
button:active { background:var(--neon); color:var(--bg); transform:scale(0.96); }
.gold { color:var(--gold); border-color:var(--gold); }
.gold:active { background:var(--gold); color:var(--bg); }
.stopBtn { color:var(--red); border-color:var(--red); }
.stopBtn:active { background:var(--red); color:var(--bg); }

/* Joystick column */
#joyPad { position:relative; width:150px; height:150px; margin:auto; border-radius:50%; background:#000; border:2px solid var(--neon); touch-action:none; flex-shrink:0; }
#joyStick { position:absolute; width:50px; height:50px; border-radius:50%; background:var(--neon); left:50px; top:50px; }

/* Drive buttons column */
.driveGrid { display:grid; grid-template-columns:1fr 1fr 1fr; grid-template-rows:1fr 1fr 1fr; gap:6px; flex:1; }
.driveGrid button { font-size:12px; padding:0; }
.driveGrid .full { grid-column:1/-1; }

/* Face column */
.faceGrid { display:grid; grid-template-columns:1fr 1fr; gap:6px; }
.faceGrid button { font-size:11px; padding:10px 0; }
#txt { width:100%; padding:8px; background:#000; border:1px solid #333; color:var(--neon); border-radius:6px; margin:6px 0; font-size:12px; }

/* Sound/sliders column */
.soundGrid { display:grid; grid-template-columns:1fr 1fr; gap:6px; margin-bottom:6px; }
.soundGrid button { font-size:11px; padding:10px 0; }
.sliderRow { display:flex; justify-content:space-between; font-size:10px; color:var(--muted); text-transform:uppercase; letter-spacing:1px; margin:6px 0 2px; }
input[type=range] { width:100%; accent-color:var(--neon); }
</style>
</head><body>

<div id="camPane">
  <img id="camStream">
  <span id="camStatus">connecting</span>
  <button id="bigStop" onclick="sendCmd('STOP')">STOP</button>
</div>

<div id="controls">

  <div class="col" style="align-items:center; justify-content:center;">
    <div class="colTitle">Joystick</div>
    <div id="joyPad"><div id="joyStick"></div></div>
  </div>

  <div class="col">
    <div class="colTitle">Drive</div>
    <div class="driveGrid">
      <button class="gold" onclick="sendCmd('TURN_90')">90&deg;</button>
      <button onclick="sendCmd('FORWARD')">FWD</button>
      <button class="gold" onclick="sendCmd('TURN_180')">180&deg;</button>
      <button onclick="sendCmd('SPIN_LEFT')">&#8634;</button>
      <button class="stopBtn" onclick="sendCmd('STOP')">STOP</button>
      <button onclick="sendCmd('SPIN_RIGHT')">&#8635;</button>
      <button onclick="sendCmd('LEFT')">LEFT</button>
      <button onclick="sendCmd('BACKWARD')">REV</button>
      <button onclick="sendCmd('RIGHT')">RIGHT</button>
    </div>
    <div class="sliderRow"><span>Speed</span><span id="spdVal">200</span></div>
    <input type="range" min="100" max="255" value="200" oninput="updateSpd(this.value)">
  </div>

  <div class="col">
    <div class="colTitle">Face</div>
    <div class="faceGrid">
      <button class="gold" onclick="sendCmd('FACE=0')">Happy</button>
      <button class="gold" onclick="sendCmd('FACE=1')">Angry</button>
      <button class="gold" onclick="sendCmd('FACE=2')">Sleepy</button>
      <button class="gold" onclick="sendCmd('FACE=6')">Wink</button>
      <button class="gold" onclick="sendCmd('FACE=4')">Love</button>
      <button class="gold" onclick="sendCmd('FACE=5')">Fire</button>
      <button class="gold" onclick="sendCmd('FACE=7')">Cool</button>
      <button class="gold" onclick="sendCmd('FACE=8')">Wow</button>
    </div>
    <input type="text" id="txt" placeholder="Message...">
    <button style="width:100%; padding:8px 0;" onclick="sendText()">Push Text</button>
  </div>

  <div class="col">
    <div class="colTitle">Sound</div>
    <div class="soundGrid">
      <button class="gold" onclick="sendCmd('SOUND=1')">Horn</button>
      <button class="gold" onclick="sendCmd('SOUND=2')">Siren</button>
      <button class="gold" onclick="sendCmd('SOUND=3')">Laugh</button>
      <button class="stopBtn" onclick="sendCmd('SOUND_STOP')">Mute</button>
    </div>
    <div class="sliderRow"><span>Volume</span><span id="volVal">20</span></div>
    <input type="range" min="0" max="30" value="20" oninput="updateVol(this.value)">
  </div>

</div>

<script>
  const CAM_BASE = ")rawliteral";

const char index_html_part2[] PROGMEM = R"rawliteral(";

  function sendCmd(c) { fetch('/' + c); }
  function updateSpd(v) { document.getElementById('spdVal').innerText = v; fetch('/SPEED=' + v); }
  function updateVol(v) { document.getElementById('volVal').innerText = v; fetch('/VOL=' + v); }
  function sendText() { let t = document.getElementById('txt').value; fetch('/TEXT=' + encodeURIComponent(t)); }

  // --- Camera: snapshot polling (works on every browser, incl. iOS Safari) ---
  const camImg = document.getElementById('camStream');
  const camStatus = document.getElementById('camStatus');
  let camPending = false;

  function refreshCam() {
    if (camPending) return;
    camPending = true;
    const testImg = new Image();
    testImg.onload = function () {
      camImg.src = testImg.src;
      camStatus.textContent = 'live';
      camStatus.className = 'live';
      camPending = false;
    };
    testImg.onerror = function () {
      camStatus.textContent = 'no signal';
      camStatus.className = 'down';
      camPending = false;
    };
    testImg.src = CAM_BASE + '/capture?t=' + Date.now();
  }
  setInterval(refreshCam, 200);
  refreshCam();

  // --- Joystick ---
  (function() {
    const pad = document.getElementById('joyPad');
    const stick = document.getElementById('joyStick');
    const padRadius = 75;
    const stickRadius = 25;
    const maxDist = padRadius - stickRadius;
    let dragging = false;
    let lastSend = 0;

    function setStick(dx, dy) {
      stick.style.left = (padRadius - stickRadius + dx) + 'px';
      stick.style.top  = (padRadius - stickRadius - dy) + 'px';
    }

    function sendJoy(x, y) {
      const now = Date.now();
      if (now - lastSend < 110) return;
      lastSend = now;
      fetch('/JOY?x=' + Math.round(x) + '&y=' + Math.round(y));
    }

    function handleMove(clientX, clientY) {
      const rect = pad.getBoundingClientRect();
      let dx = clientX - (rect.left + padRadius);
      let dy = (rect.top + padRadius) - clientY;
      let dist = Math.sqrt(dx * dx + dy * dy);
      if (dist > maxDist) {
        const ratio = maxDist / dist;
        dx *= ratio; dy *= ratio;
      }
      setStick(dx, dy);
      sendJoy((dx / maxDist) * 100, (dy / maxDist) * 100);
    }

    function resetStick() {
      setStick(0, 0);
      fetch('/JOY?x=0&y=0');
    }

    pad.addEventListener('touchstart', e => { dragging = true; e.preventDefault(); });
    pad.addEventListener('touchmove', e => {
      if (!dragging) return;
      const t = e.touches[0];
      handleMove(t.clientX, t.clientY);
      e.preventDefault();
    });
    pad.addEventListener('touchend', () => { dragging = false; resetStick(); });

    pad.addEventListener('mousedown', () => { dragging = true; });
    window.addEventListener('mousemove', e => { if (dragging) handleMove(e.clientX, e.clientY); });
    window.addEventListener('mouseup', () => { if (dragging) { dragging = false; resetStick(); } });
  })();
</script>
</body></html>
)rawliteral";

void setup() {
  Serial.begin(115200);

  Wire.begin(21, 22);
  if(!display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {
    Serial.println(F("OLED failed"));
    for(;;);
  }

  ledcAttach(IN1, freq, resolution);
  ledcAttach(IN2, freq, resolution);
  ledcAttach(IN3, freq, resolution);
  ledcAttach(IN4, freq, resolution);
  stopRobot();

  mp3Serial.begin(9600, SERIAL_8N1, 16, 17);
  delay(500);
  mp3SetVolume(20);

  for(int i = 0; i <= 100; i += 8) {
    display.clearDisplay();
    display.drawRect(14, 25, 100, 12, SSD1306_WHITE);
    display.fillRect(16, 27, (96 * i) / 100, 8, SSD1306_WHITE);
    display.setTextSize(1);
    display.setTextColor(SSD1306_WHITE);
    display.setCursor(20, 45);
    display.print("SYSTEM BOOT: ");
    display.print(i);
    display.print("%");
    display.display();
    delay(100);
  }
  delay(300);

  WiFi.softAP(ssid, password);
  server.begin();

  Serial.println(F("Wi-Fi AP started."));
  Serial.print(F("Connect to \""));
  Serial.print(ssid);
  Serial.println(F("\" then browse to:"));
  Serial.print(F("http://"));
  Serial.println(WiFi.softAPIP());
}

void loop() {
  if (turning && (millis() - turnStartTime >= turnDuration)) {
    stopRobot();
    turning = false;
  }

  switch (faceState) {
    case FACE_HAPPY:     drawHappyFace(); break;
    case FACE_ANGRY:     drawAngryFace(); break;
    case FACE_SLEEPY:    drawSleepyFace(); break;
    case FACE_TEXT:      drawCustomTextScreen(currentText); break;
    case FACE_LOVE:      drawLoveFace(); break;
    case FACE_FIRE:      drawFireFace(); break;
    case FACE_WINK:      drawWinkFace(); break;
    case FACE_COOL:      drawCoolFace(); break;
    case FACE_SURPRISED: drawSurprisedFace(); break;
  }

  WiFiClient client = server.available();
  if (!client) return;

  unsigned long waitStart = millis();
  while (!client.available() && (millis() - waitStart) < 250) {
    delay(1);
  }
  if (!client.available()) {
    client.stop();
    return;
  }

  String request = client.readStringUntil('\r');
  while (client.available()) { client.read(); }

  bool isCommand = true;

  if (request.indexOf("/FORWARD") != -1) { turning = false; moveForward(currentSpeed); }
  else if (request.indexOf("/BACKWARD") != -1) { turning = false; moveBackward(currentSpeed); }
  else if (request.indexOf("/SPIN_LEFT") != -1) { turning = false; spinLeft(currentSpeed); }
  else if (request.indexOf("/SPIN_RIGHT") != -1) { turning = false; spinRight(currentSpeed); }
  else if (request.indexOf("/LEFT") != -1) { turning = false; turnLeft(currentSpeed); }
  else if (request.indexOf("/RIGHT") != -1) { turning = false; turnRight(currentSpeed); }
  else if (request.indexOf("/TURN_90") != -1) startTurn(currentSpeed, 350);
  else if (request.indexOf("/TURN_180") != -1) startTurn(currentSpeed, 700);
  else if (request.indexOf("/STOP") != -1) { turning = false; stopRobot(); }
  else if (request.indexOf("/JOY?") != -1) {
    turning = false;
    int xIdx = request.indexOf("x=");
    int yIdx = request.indexOf("y=");
    int jx = 0, jy = 0;
    if (xIdx != -1) {
      int end = request.indexOf('&', xIdx);
      if (end == -1) end = request.length();
      jx = request.substring(xIdx + 2, end).toInt();
    }
    if (yIdx != -1) {
      int end = request.indexOf(' ', yIdx);
      if (end == -1) end = request.length();
      jy = request.substring(yIdx + 2, end).toInt();
    }
    jx = constrain(jx, -100, 100);
    jy = constrain(jy, -100, 100);

    int left  = constrain(jy + jx, -100, 100);
    int right = constrain(jy - jx, -100, 100);
    left  = map(left,  -100, 100, -currentSpeed, currentSpeed);
    right = map(right, -100, 100, -currentSpeed, currentSpeed);
    setMotors(left, right);
  }
  else if (request.indexOf("/FACE=0") != -1) faceState = FACE_HAPPY;
  else if (request.indexOf("/FACE=1") != -1) faceState = FACE_ANGRY;
  else if (request.indexOf("/FACE=2") != -1) faceState = FACE_SLEEPY;
  else if (request.indexOf("/FACE=4") != -1) faceState = FACE_LOVE;
  else if (request.indexOf("/FACE=5") != -1) faceState = FACE_FIRE;
  else if (request.indexOf("/FACE=6") != -1) faceState = FACE_WINK;
  else if (request.indexOf("/FACE=7") != -1) faceState = FACE_COOL;
  else if (request.indexOf("/FACE=8") != -1) faceState = FACE_SURPRISED;
  else if (request.indexOf("/SPEED=") != -1) {
    int idx = request.indexOf("/SPEED=") + 7;
    int endIdx = idx;
    while (endIdx < (int)request.length() && isDigit(request.charAt(endIdx))) endIdx++;
    int val = request.substring(idx, endIdx).toInt();
    currentSpeed = constrain(val, 100, 255);
  } else if (request.indexOf("/TEXT=") != -1) {
    int idx = request.indexOf("/TEXT=") + 6;
    int spaceIdx = request.indexOf(' ', idx);
    if (spaceIdx == -1) spaceIdx = request.length();
    String rawText = request.substring(idx, spaceIdx);
    currentText = urlDecode(rawText);
    faceState = FACE_TEXT;
  } else if (request.indexOf("/SOUND_STOP") != -1) {
    mp3Stop();
  } else if (request.indexOf("/SOUND=") != -1) {
    int idx = request.indexOf("/SOUND=") + 7;
    int endIdx = idx;
    while (endIdx < (int)request.length() && isDigit(request.charAt(endIdx))) endIdx++;
    int track = request.substring(idx, endIdx).toInt();
    mp3Play(track);
  } else if (request.indexOf("/VOL=") != -1) {
    int idx = request.indexOf("/VOL=") + 5;
    int endIdx = idx;
    while (endIdx < (int)request.length() && isDigit(request.charAt(endIdx))) endIdx++;
    int vol = constrain(request.substring(idx, endIdx).toInt(), 0, 30);
    mp3SetVolume(vol);
  } else {
    isCommand = false;
  }

  client.println("HTTP/1.1 200 OK");
  if (isCommand) {
    client.println("Content-Type: text/plain");
    client.println("Connection: close");
    client.println();
    client.println("OK");
  } else {
    client.println("Content-Type: text/html");
    client.println("Connection: close");
    client.println();
    client.print(index_html_part1);
    client.print(CAM_BASE_URL);
    client.print(index_html_part2);
  }

  delay(1);
  client.stop();
}

// ==========================================
// OLED EXPRESSION ENGINE
// ==========================================
void drawHappyFace() {
  display.clearDisplay();
  display.fillRoundRect(24, 18, 26, 32, 6, SSD1306_WHITE);
  display.fillRoundRect(78, 18, 26, 32, 6, SSD1306_WHITE);
  if ((millis() / 2500) % 3 == 0) {
    display.fillRect(24, 18, 26, 12, SSD1306_BLACK);
    display.fillRect(78, 18, 26, 12, SSD1306_BLACK);
  }
  display.drawFastHLine(52, 54, 24, SSD1306_WHITE);
  display.drawPixel(51, 53, SSD1306_WHITE);
  display.drawPixel(76, 53, SSD1306_WHITE);
  display.display();
}

void drawAngryFace() {
  display.clearDisplay();
  display.fillRect(24, 25, 26, 15, SSD1306_WHITE);
  display.fillRect(78, 25, 26, 15, SSD1306_WHITE);
  display.fillTriangle(18, 15, 55, 32, 18, 30, SSD1306_BLACK);
  display.fillTriangle(110, 15, 73, 32, 110, 30, SSD1306_BLACK);
  display.drawFastHLine(48, 52, 32, SSD1306_WHITE);
  display.display();
}

void drawSleepyFace() {
  display.clearDisplay();
  display.fillRect(24, 28, 26, 8, SSD1306_WHITE);
  display.fillRect(78, 28, 26, 8, SSD1306_WHITE);
  display.drawCircle(64, 50, 4, SSD1306_WHITE);

  int snoreState = (millis() / 600) % 4;
  if(snoreState >= 1) display.drawCircle(88, 20, 2, SSD1306_WHITE);
  if(snoreState >= 2) display.drawCircle(100, 12, 3, SSD1306_WHITE);
  if(snoreState >= 3) display.drawCircle(114, 4, 4, SSD1306_WHITE);

  display.display();
}

void drawCustomTextScreen(String text) {
  display.clearDisplay();
  display.drawRect(0, 0, 128, 64, SSD1306_WHITE);
  display.setTextSize(2);
  display.setTextColor(SSD1306_WHITE);

  if (text.length() > 10) {
    display.setTextSize(1);
    display.setCursor(8, 28);
  } else {
    display.setCursor(10, 24);
  }
  display.println(text);
  display.display();
}

void drawHeart(int cx, int cy, int size) {
  if (size < 4) size = 4;
  int r = size / 2;
  display.fillCircle(cx - r / 2, cy - r / 2, r / 2, SSD1306_WHITE);
  display.fillCircle(cx + r / 2, cy - r / 2, r / 2, SSD1306_WHITE);
  display.fillTriangle(cx - r, cy - r / 4, cx + r, cy - r / 4, cx, cy + r, SSD1306_WHITE);
}

void drawLoveFace() {
  display.clearDisplay();
  float phase = (millis() % 1200) / 1200.0;
  float scale = 1.0 + 0.18 * sin(phase * 2 * PI);
  drawHeart(64, 32, (int)(32 * scale));

  int drift1 = (millis() / 30) % 46;
  int drift2 = (millis() / 24 + 22) % 46;
  drawHeart(20, 58 - drift1, 8);
  drawHeart(106, 58 - drift2, 7);

  display.display();
}

void drawFireFace() {
  display.clearDisplay();

  unsigned long t = millis();
  int flicker1 = (int)(4 * sin(t / 90.0));
  int flicker2 = (int)(3 * sin(t / 130.0 + 1.0));
  int flicker3 = (int)(2 * sin(t / 170.0 + 2.0));

  display.fillTriangle(24, 60, 104, 60, 64, 8 + flicker1, SSD1306_WHITE);
  display.fillTriangle(34, 55, 94, 55, 64, 20 + flicker2, SSD1306_BLACK);
  display.fillTriangle(46, 50, 82, 50, 64, 30 + flicker3, SSD1306_WHITE);

  for (int i = 0; i < 3; i++) {
    int phase = (t / 15 + i * 60) % 60;
    int sx = 30 + i * 30;
    int sy = 55 - phase;
    if (sy > 2) display.drawPixel(sx, sy, SSD1306_WHITE);
  }

  display.display();
}

void drawWinkFace() {
  display.clearDisplay();
  display.fillRoundRect(24, 18, 26, 32, 6, SSD1306_WHITE);

  bool eyeOpen = ((millis() / 3000) % 5 == 0);
  if (eyeOpen) {
    display.fillRoundRect(78, 18, 26, 32, 6, SSD1306_WHITE);
  } else {
    display.fillRoundRect(78, 30, 26, 6, 3, SSD1306_WHITE);
  }

  display.drawLine(50, 52, 64, 57, SSD1306_WHITE);
  display.drawLine(64, 57, 80, 50, SSD1306_WHITE);
  display.display();
}

void drawCoolFace() {
  display.clearDisplay();
  display.fillRoundRect(18, 22, 36, 20, 4, SSD1306_WHITE);
  display.fillRoundRect(74, 22, 36, 20, 4, SSD1306_WHITE);
  display.drawFastHLine(54, 30, 20, SSD1306_WHITE);
  display.drawLine(10, 24, 18, 28, SSD1306_WHITE);
  display.drawLine(110, 24, 118, 28, SSD1306_WHITE);
  display.drawFastHLine(48, 50, 32, SSD1306_WHITE);
  display.display();
}

void drawSurprisedFace() {
  display.clearDisplay();
  display.fillCircle(37, 30, 14, SSD1306_WHITE);
  display.fillCircle(91, 30, 14, SSD1306_WHITE);
  display.fillCircle(37, 30, 6, SSD1306_BLACK);
  display.fillCircle(91, 30, 6, SSD1306_BLACK);

  int r = 6 + (int)(2 * sin(millis() / 200.0));
  display.fillCircle(64, 52, r, SSD1306_WHITE);
  display.fillCircle(64, 52, r > 2 ? r - 3 : 1, SSD1306_BLACK);
  display.display();
}

// ==========================================
// MOTOR KINETICS
// ==========================================
void setMotors(int left, int right) {
  if (REVERSE_DRIVE) { left = -left; right = -right; }
  left = constrain(left, -255, 255);
  right = constrain(right, -255, 255);

  if (left >= 0) { ledcWrite(IN1, left);  ledcWrite(IN2, 0); }
  else           { ledcWrite(IN1, 0);     ledcWrite(IN2, -left); }

  if (right >= 0) { ledcWrite(IN3, right); ledcWrite(IN4, 0); }
  else            { ledcWrite(IN3, 0);     ledcWrite(IN4, -right); }
}

void moveForward(int spd)  { setMotors(spd, spd); }
void moveBackward(int spd) { setMotors(-spd, -spd); }
void turnLeft(int spd)     { setMotors(0, spd); }
void turnRight(int spd)    { setMotors(spd, 0); }
void spinLeft(int spd)     { setMotors(-spd, spd); }
void spinRight(int spd)    { setMotors(spd, -spd); }
void stopRobot()           { setMotors(0, 0); }

void startTurn(int spd, unsigned long duration) {
  spinRight(spd);
  turning = true;
  turnStartTime = millis();
  turnDuration = duration;
}

// ==========================================
// MP3-TF-16P AUDIO MODULE
// ==========================================
void mp3SendCmd(uint8_t cmd, uint16_t param) {
  uint8_t frame[10];
  frame[0] = 0x7E;
  frame[1] = 0xFF;
  frame[2] = 0x06;
  frame[3] = cmd;
  frame[4] = 0x00;
  frame[5] = (uint8_t)(param >> 8);
  frame[6] = (uint8_t)(param & 0xFF);
  uint16_t checksum = 0 - (frame[1] + frame[2] + frame[3] + frame[4] + frame[5] + frame[6]);
  frame[7] = (uint8_t)(checksum >> 8);
  frame[8] = (uint8_t)(checksum & 0xFF);
  frame[9] = 0xEF;
  mp3Serial.write(frame, 10);
}

void mp3Play(uint16_t track)     { mp3SendCmd(0x03, track); }
void mp3SetVolume(uint8_t vol)   { mp3SendCmd(0x06, vol); }
void mp3Stop()                   { mp3SendCmd(0x16, 0); }

// ==========================================
// HELPERS
// ==========================================
String urlDecode(String input) {
  String decoded = "";
  char hexBuf[3] = {0, 0, 0};
  unsigned int len = input.length();
  for (unsigned int i = 0; i < len; i++) {
    char c = input.charAt(i);
    if (c == '+') {
      decoded += ' ';
    } else if (c == '%' && i + 2 < len) {
      hexBuf[0] = input.charAt(i + 1);
      hexBuf[1] = input.charAt(i + 2);
      decoded += (char) strtol(hexBuf, NULL, 16);
      i += 2;
    } else {
      decoded += c;
    }
  }
  return decoded;
}
```


