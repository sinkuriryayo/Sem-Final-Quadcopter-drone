# Sem-Final-Quadcopter-drone
Quadcopter control interface using web on smartphone/PC 


#include <WiFi.h>
#include <ESPAsyncWebServer.h>

// WiFi credentials
const char* ssid = "ESP32-DRONE";
const char* password = "quadcopter";

// BLDC motor PWM pins and channels
const int motorPins[4] = {23, 22, 21, 19};    // M1, M2, M3, M4
const int motorChannels[4] = {0, 1, 2, 3};
const int pwmFreq = 50;
const int pwmResolution = 16;

bool motorActive = false;

int throttle = 0;
int pitch = 0; // Forward/Backward
int roll = 0;  // Left/Right

AsyncWebServer server(80);

void writeMicroseconds(uint8_t channel, int microseconds) {
  int duty = (microseconds * 65536) / 20000;
  ledcWrite(channel, duty);
}

void applyMixer() {
  if (!motorActive) {
    for (int i = 0; i < 4; i++) {
      writeMicroseconds(motorChannels[i], 1000); // Stop signal
    }
    return;
  }

  // Mixer math: M1=MFL, M2=MFR, M3=MRR, M4=MRL
  int m1 = throttle + pitch + roll;
  int m2 = throttle + pitch - roll;
  int m3 = throttle - pitch - roll;
  int m4 = throttle - pitch + roll;

  // Constrain values and map to microseconds
  m1 = constrain(m1, 0, 255);
  m2 = constrain(m2, 0, 255);
  m3 = constrain(m3, 0, 255);
  m4 = constrain(m4, 0, 255);

  int us1 = map(m1, 0, 255, 1000, 2000);
  int us2 = map(m2, 0, 255, 1000, 2000);
  int us3 = map(m3, 0, 255, 1000, 2000);
  int us4 = map(m4, 0, 255, 1000, 2000);

  writeMicroseconds(motorChannels[0], us1);
  writeMicroseconds(motorChannels[1], us2);
  writeMicroseconds(motorChannels[2], us3);
  writeMicroseconds(motorChannels[3], us4);
}

void setup() {
  Serial.begin(115200);

  WiFi.softAP(ssid, password);
  delay(1000);
  Serial.println("ESP32 AP running at:");
  Serial.println(WiFi.softAPIP());

  for (int i = 0; i < 4; i++) {
    ledcSetup(motorChannels[i], pwmFreq, pwmResolution);
    ledcAttachPin(motorPins[i], motorChannels[i]);
    writeMicroseconds(motorChannels[i], 1000); // Arm ESCs
  }

  delay(2000);

  server.on("/", HTTP_GET, [](AsyncWebServerRequest *request){
    String html = R"rawliteral(
      <!DOCTYPE html>
      <html>
      <head>
        <title>Quadcopter Mixer Control</title>
        <style>
          body {
            font-family: Arial;
            background-color: #111;
            color: #fff;
            text-align: center;
          }
          h2 {
            color: #03dac6;
            margin-bottom: 20px;
          }
          .slider-container {
            display: flex;
            justify-content: center;
            gap: 60px;
            margin-bottom: 20px;
          }
          .vertical-slider {
            writing-mode: bt-lr;
            -webkit-appearance: slider-vertical;
            height: 200px;
            width: 40px;
          }
          .horizontal-slider {
            width: 300px;
            margin: 20px auto;
          }
          .master-button {
            background: green;
            border: none;
            color: black;
            font-size: 18px;
            padding: 15px 40px;
            border-radius: 10px;
            cursor: pointer;
            margin-top: 30px;
          }
          .footer {
            color: #fff;
            font-size: 16px;
            margin-top: 50px;
          }
        </style>
      </head>
      <body>
        <h2>Quadcopter BLDC Motor Mixer</h2>

        <div class="slider-container">
          <div>
            <p>Pitch (F/B)</p>
            <input type="range" min="-100" max="100" value="0" class="vertical-slider" id="pitch" oninput="sendData()">
            <p id="valPitch">0</p>
          </div>
          <div>
            <p>Roll (L/R)</p>
            <input type="range" min="-100" max="100" value="0" class="vertical-slider" id="roll" oninput="sendData()">
            <p id="valRoll">0</p>
          </div>
        </div>

        <p>Throttle</p>
        <input type="range" min="0" max="255" value="0" class="horizontal-slider" id="throttle" oninput="sendData()">
        <p id="valThrottle">0</p>

        <button class="master-button" id="startStopBtn" onclick="toggleMotor()">Start / Stop Motors</button>

        <div class="footer">
          <p>Created by: SiAli & Jules</p>
        </div>

        <script>
          let motorState = false;

          function sendData() {
            let pitch = document.getElementById("pitch").value;
            let roll = document.getElementById("roll").value;
            let throttle = document.getElementById("throttle").value;

            document.getElementById("valPitch").innerText = pitch;
            document.getElementById("valRoll").innerText = roll;
            document.getElementById("valThrottle").innerText = throttle;

            fetch(`/control?pitch=${pitch}&roll=${roll}&throttle=${throttle}`);
          }

          function toggleMotor() {
            motorState = !motorState;
            fetch(`/motor?state=${motorState ? 'on' : 'off'}`);

            // Change button color based on motor state
            const button = document.getElementById("startStopBtn");
            if (motorState) {
              button.style.backgroundColor = "red";  // Motors on
            } else {
              button.style.backgroundColor = "green"; // Motors off
            }
          }

          // Reset sliders to center position when mouse is released
          document.getElementById("pitch").addEventListener("mouseup", function() {
            this.value = 0;
            sendData();
          });

          document.getElementById("roll").addEventListener("mouseup", function() {
            this.value = 0;
            sendData();
          });
        </script>
      </body>
      </html>
    )rawliteral";
    request->send(200, "text/html", html);
  });

  server.on("/motor", HTTP_GET, [](AsyncWebServerRequest *request){
    String state = request->getParam("state")->value();
    motorActive = (state == "on");
    Serial.println(motorActive ? "Motors Activated" : "Motors Deactivated");

    applyMixer(); // Will stop or set values accordingly
    request->send(200, "text/plain", "OK");
  });

  server.on("/control", HTTP_GET, [](AsyncWebServerRequest *request){
    if (!motorActive) {
      request->send(200, "text/plain", "Motors are OFF");
      return;
    }

    pitch = request->getParam("pitch")->value().toInt();
    roll = request->getParam("roll")->value().toInt();
    throttle = request->getParam("throttle")->value().toInt();

    Serial.printf("T:%d, P:%d, R:%d\n", throttle, pitch, roll);
    applyMixer();

    request->send(200, "text/plain", "Updated");
  });

  server.begin();
}

void loop() {
  // Nothing here
}
