# 임베디드 시스템 종합 과제
##### 20201113 주하진
### Arduino 코드
```
#include <math.h>    // log() 함수 사용

const int tempPin   = A1;   // 써미스터 입력 핀
const int lightPin  = A0;   // 조도 센서 입력 핀
const int ledPin    = 9;    // LED 핀
const int buzzerPin = 10;   // 피에조 스피커 핀

// 온도 / 조도 기준값 (실험하면서 조절)
const double TEMP_THRESHOLD  = 25.0;  // 25도 이상이면 덥다고 판단 (섭씨)
const int    LIGHT_THRESHOLD = 750;   // 이 값보다 작으면 어둡다고 판단 (ADC 값)

// LED 깜빡임용
const unsigned long BLINK_INTERVAL = 400; // 0.4초마다 깜빡
unsigned long lastBlinkTime = 0;
bool ledState = false;

// 집중 모드 (앱에서 '1' / '0'으로 제어)
int focusMode = 0;   // 0: OFF, 1: ON

// 써미스터 온도 계산 함수 (섭씨 반환)
// v: analogRead(tempPin) 값 (0~1023)
double th(int v) {
  double t; // http://en.wikipedia.org/wiki/Thermistor
  t = log(((10240000/v) - 10000));
  t = 1 /(0.001129148 + (0.000234125*t) + (0.0000000876741*t*t*t));
  t = t - 273.15; // 화씨를 섭씨로 바꾸어줌
  return t;
}

void setup() {
  Serial.begin(9600);

  pinMode(ledPin, OUTPUT);
  pinMode(buzzerPin, OUTPUT);

  digitalWrite(ledPin, LOW);
  noTone(buzzerPin);
}

void loop() {
  unsigned long now = millis();

  // 1. 센서 값 읽기
  int tempVal  = analogRead(tempPin);    // 0~1023
  int lightVal = analogRead(lightPin);   // 0~1023

  double tempC = th(tempVal);

  // 2. 앱에서 오는 시리얼 명령 처리 ('1' / '0')
  handleSerialCommand();

  // 3. 온도 기준 넘으면 버저 ON / 아니면 OFF
  handleBuzzer(tempC);

  // 4. LED 제어 (집중 모드 + 조도 기준 + millis 기반 깜빡임)
  handleLED(lightVal, now);

  // 5. 상태를 시리얼로 출력 (서버/앱에서 읽기 쉽도록)
  // 형식 예: 26.53,512,1
  Serial.print(tempC);
  Serial.print(",");
  Serial.print(lightVal);
  Serial.print(",");
  Serial.println(focusMode);

  // delay() 없음 → millis 기반으로 계속 순환
}

// -----------------------------
// 시리얼 명령 처리
// '1' → 집중 모드 ON
// '0' → 집중 모드 OFF
// -----------------------------
void handleSerialCommand() {
  while (Serial.available() > 0) {
    char cmd = Serial.read();
    if (cmd == '1') {
      focusMode = 1;
    } else if (cmd == '0') {
      focusMode = 0;
    }
    // 줄바꿈 문자('\n') 등은 무시
  }
}

// -----------------------------
// 온도 기준 넘으면 버저 울리기
// -----------------------------
void handleBuzzer(double tempC) {
  if (tempC >= TEMP_THRESHOLD) {
    // 꽤 덥다고 판단되면 계속 경고음 (2kHz)
    tone(buzzerPin, 2000);
  } else {
    noTone(buzzerPin);
  }
}

// -----------------------------
// LED 제어 (millis 기반)
// - focusMode == 1 이면서 어두울 때: 깜빡임
// - focusMode == 1 이면서 밝을 때: 계속 켜짐
// - focusMode == 0: 꺼짐
// -----------------------------
void handleLED(int lightVal, unsigned long now) {
  // 집중 모드가 꺼져 있으면 LED 항상 OFF
  if (focusMode == 0) {
    digitalWrite(ledPin, LOW);
    ledState = false;
    return;
  }

  // 여기부터는 focusMode == 1 인 경우
  bool isDark = (lightVal < LIGHT_THRESHOLD);

  if (isDark) {
    // 어두우면 깜빡임 (BLINK_INTERVAL마다 상태 토글)
    if (now - lastBlinkTime >= BLINK_INTERVAL) {
      ledState = !ledState;
      digitalWrite(ledPin, ledState ? HIGH : LOW);
      lastBlinkTime = now;
    }
  } else {
    // 충분히 밝으면 항상 켜두기
    digitalWrite(ledPin, HIGH);
    ledState = true;
  }
}

```

### Processing 코드
```
import processing.serial.*;
import processing.net.*;

Serial p;      // 아두이노와 시리얼 통신
Server s;      // 스마트폰과 통신하는 서버
Client c;

String msg = "0,0,0";  // tempC,light,focusMode 기본값

void setup() {
  size(400, 200);
  

  p = new Serial(this, "COM5", 9600);
  p.bufferUntil('\n'); // 줄 단위로 받기


  s = new Server(this, 12345);
  println("Server started on port 12345");
}

void draw() {
  background(200);
  fill(0);
  text("Last data from Arduino:", 10, 30);
  text(msg, 10, 50);

  // 스마트폰에서 온 요청 처리
  handleClient();
}

// -------------------------
// 아두이노 → 시리얼 데이터 읽기
// 예: "26.53,512,1"
// -------------------------
void serialEvent(Serial p) {
  String line = p.readStringUntil('\n');
  if (line != null) {
    line = line.trim();
    if (line.length() > 0) {
      msg = line;  // tempC,light,focusMode
      println("from Arduino: " + msg);
    }
  }
}

// -------------------------
// 스마트폰 → HTTP 비슷한 요청 처리
// -------------------------
void handleClient() {
  c = s.available();
  if (c == null) return;

  String g = c.readString(); // HTTP 요청 전체
  if (g != null) {
    println("from phone:\n" + g);

    // URL에 mode=1 / mode=0 이 들어있으면 그걸로 집중 모드 제어
    if (g.indexOf("mode=1") != -1) {
      println(">> focusMode ON (send '1' to Arduino)");
      p.write('1');       // 아두이노에 '1' 전송
    } else if (g.indexOf("mode=0") != -1) {
      println(">> focusMode OFF (send '0' to Arduino)");
      p.write('0');       // 아두이노에 '0' 전송
    }
  }

  // HTTP 응답 보내기 (아주 최소한의 헤더 + 바디)
  if (msg == null || msg.length() == 0) {
    msg = "0,0,0";
  }

  c.write("HTTP/1.1 200 OK\r\n");
  c.write("Content-Type: text/plain\r\n");
  c.write("Access-Control-Allow-Origin: *\r\n");
  c.write("\r\n");
  c.write(msg);   // 예: "26.53,512,1"
  c.stop();
}

```

### App Inventor 블록
<img width="250" height="141" alt="App1" src="https://github.com/user-attachments/assets/ad665541-0cf4-4495-b9d0-7ec455bbad00" />

<img width="298" height="354" alt="App2" src="https://github.com/user-attachments/assets/fd449646-7559-4475-9843-c70bd24c442e" />

<img width="298" height="182" alt="App3" src="https://github.com/user-attachments/assets/0edc12f0-1e76-4e72-883e-355c5d2fb267" />
