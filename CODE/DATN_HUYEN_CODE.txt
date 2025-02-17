#include <WiFi.h>
#include <ESPAsyncWebServer.h>
#include <Wire.h>
#include <EEPROM.h>
#define EEPROM_SIZE 32 // Kích thước vùng EEPROM giả lập (tối đa 4096 bytes trên ESP32)
#include <SHT3x.h>
SHT3x Sensor;
#include <LiquidCrystal_I2C.h>
LiquidCrystal_I2C lcd(0x27, 16, 2);
#define fan 4
#define humidifier 17
#define temps 16
#define out1 5
#define out2 18
#define out3 19
#define in1 32
#define in2 33
#define in3 27
void get_data_Sensor();
void print_lcd();
void controlDevice();
void read_button();
void saveByteToEEPROM(int address, byte value);
byte readByteFromEEPROM(int address);
// Thông tin WiFi
const char *ssid = "ESP32_WebServer";
const char *password = "12345678";
// Tạo web server
AsyncWebServer server(80);

// Biến nhiệt độ và độ ẩm
float temperature = 0;
float humidity = 0;

// Biến trạng thái
bool autoMode = true;
int device[6] = {out1, out2, out3, fan, temps, humidifier};
int input_btn[3] = {in1, in2, in3};
bool deviceStates[6] = {false, false, false, false, false, false}; // Trạng thái của 6 thiết bị
int stt_input[3] = {0, 0, 0};
int tempHW, tempLW, humiW;
// HTML giao diện người dùng
const char index_html[] PROGMEM = R"rawliteral(
<!DOCTYPE html>
<html>

<head>
    <meta charset="UTF-8">
    <title>ESP32 WEBSERVER</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            text-align: center;
            margin: 0;
            padding: 0;
        }

        .container {
            display: flex;
            flex-direction: column;
            align-items: center;
            padding: 20px;
        }

        .row {
            display: flex;
            justify-content: center;
            gap: 20px;
            margin: 10px 0;
        }

        .card {
            background-color: #aaffaa;
            padding: 20px;
            border-radius: 10px;
            width: 150px;
            text-align: center;
        }

        .divider {
            width: 80%;
            /* Chiều rộng đường kẻ ngang */
            border: none;
            height: 2px;
            /* Độ dày đường kẻ */
            background-color: #ccc;
            /* Màu đường kẻ */
            margin: 20px auto;
            /* Khoảng cách giữa các hàng */
        }

        .button {
            background-color: #4CAF50;
            border: none;
            color: white;
            padding: 5px 10px;
            font-size: 14px;
            border-radius: 5px;
            cursor: pointer;
            transition: background-color 0.3s;
        }

        .button.off {
            background-color: #f44336;
        }

        .switch {
            display: flex;
            align-items: center;
            gap: 10px;
        }

        .thresholds {
            display: flex;
            flex-direction: column;
            gap: 10px;
            align-items: center;
            margin: 20px 0;
        }

        .threshold-row {
            display: flex;
            align-items: center;
            gap: 10px;
            width: 100%;
            max-width: 400px;
            /* Giới hạn độ rộng tối đa */
        }

        .threshold-row label {
            flex: 1;
            text-align: left;
            font-size: 16px;
            font-weight: 500;
        }

        .input-box {
            width: 60px;
            padding: 5px;
            text-align: center;
            border: 1px solid #ccc;
            border-radius: 5px;
            font-size: 16px;
        }

        .padding-8 {
            padding-top: 8px;
            padding-bottom: 8px;
        }

        .mt-0 {
            margin-top: 0px;
            margin-bottom: 0px;
        }

        .w-100 {
            width: 118px;
        }
    </style>
</head>

<body>
    <h1>ESP32 WEBSERVER</h1>
    <h3>Đồ án tốt nghiệp 2024</h3>
    <div class="container">
        <div class="row">
            <div class="card">
                <h2>NHIỆT ĐỘ</h2>
                <p id="temp">28 &deg;C</p>
            </div>
            <div class="card">
                <h2>ĐỘ ẨM</h2>
                <p id="humidity">68%</p>
            </div>
        </div>
        <div class="row">
            <button class="button padding-8 w-100" id="device1" onclick="controlDevice(1)">
                <p class="mt-0">Thiết bị 1</p>
                <p class="mt-0">OFF</p>
            </button>
            <button class="button padding-8 w-100" id="device2" onclick="controlDevice(2)">
                <p class="mt-0">Thiết bị 2</p>
                <p class="mt-0">OFF</p>
            </button>
            <button class="button padding-8 w-100" id="device3" onclick="controlDevice(3)">
                <p class="mt-0">Thiết bị 3</p>
                <p class="mt-0">OFF</p>
            </button>
        </div>
        <hr class="divider">
        <div class="row">
            <button class="button padding-8 w-100" id="device4" onclick="controlDevice(4)">
                <p class="mt-0">Quạt</p>
                <p class="mt-0">OFF</p>
            </button>
            <button class="button padding-8 w-100" id="device5" onclick="controlDevice(5)">
                <p class="mt-0">Đèn sưởi</p>
                <p class="mt-0">OFF</p>
            </button>
            <button class="button padding-8 w-100" id="device6" onclick="controlDevice(6)">
                <p class="mt-0">Máy tạo ẩm</p>
                <p class="mt-0">OFF</p>
            </button>
        </div>
        <div class="row switch">
            <label for="auto">Auto Mode:</label>
            <input type="checkbox" id="auto" onchange="toggleAutoMode()">
        </div>
        <div class="thresholds">
            <div class="threshold-row">
                <label id="tempHighLabel" for="tempHigh">Ngưỡng nhiệt độ trên: 0 |</label>
                <input type="number" id="tempHigh" value="0" class="input-box">
                <button class="button" onclick="setThreshold('tempHigh')">SET</button>
            </div>
            <div class="threshold-row">
                <label id="tempLowLabel" for="tempLow">Ngưỡng nhiệt độ dưới: 0 |</label>
                <input type="number" id="tempLow" value="0" class="input-box">
                <button class="button" onclick="setThreshold('tempLow')">SET</button>
            </div>
            <div class="threshold-row">
                <label id="humiLabel" for="humidityThreshold">Ngưỡng độ ẩm: 0 |</label>
                <input type="number" id="humidityThreshold" value="0" class="input-box">
                <button class="button" onclick="setThreshold('humidityThreshold')">SET</button>
            </div>
        </div>
    </div>
    <script>
       let autoMode = false;

function controlDevice(device) {
    if (autoMode && (device === 4 || device === 5 || device === 6)) return;

    const button = document.getElementById(`device${device}`);
    const state = button.innerText.includes('OFF') ? 'ON' : 'OFF';

    // Cập nhật giao diện ngay lập tức
    const label = device === 4 ? 'Quạt' :
                  device === 5 ? 'Đèn Sưởi' :
                  device === 6 ? 'Máy Tạo Ẩm' : `Thiết bị ${device}`;
    button.innerHTML = `
        <p class="mt-0">${label}</p>
        <p class="mt-0">${state}</p>
    `;
    button.classList.toggle('off', state === 'OFF');

    // Gửi yêu cầu đến server
    fetch(`/control?device=${device}&state=${state}`)
        .then(response => response.text())
        .then(() => {
            console.log(`Đã cập nhật trạng thái: Thiết bị ${device} - ${state}`);
        })
        .catch(error => {
            console.error('Lỗi khi gửi yêu cầu:', error);

            // Khôi phục trạng thái cũ nếu có lỗi xảy ra
            const prevState = state === 'ON' ? 'OFF' : 'ON';
            button.innerHTML = `
                <p class="mt-0">${label}</p>
                <p class="mt-0">${prevState}</p>
            `;
            button.classList.toggle('off', prevState === 'OFF');
        });
}

function updateDeviceStates() {
    fetch('/states')
        .then(response => response.json())
        .then(data => {
            for (let i = 1; i <= 6; i++) {
                const button = document.getElementById(`device${i}`);
                const state = data[`device${i}`];
                const label = i === 4 ? 'Quạt' :
                              i === 5 ? 'Đèn Sưởi' :
                              i === 6 ? 'Máy Tạo Ẩm' : `Thiết bị ${i}`;
                button.innerHTML = `
                    <p class="mt-0">${label}</p>
                    <p class="mt-0">${state}</p>
                `;
                button.classList.toggle('off', state === 'OFF');
            }
        })
        .catch(error => console.error('Lỗi khi cập nhật trạng thái thiết bị:', error));
}

function toggleAutoMode() {
    autoMode = document.getElementById('auto').checked;
    fetch(`/setAutoMode?value=${autoMode}`)
    // Vô hiệu hóa các nút điều khiển thiết bị 4, 5, 6 khi Auto Mode bật
    ennableBtn();
}
function ennableBtn(){
    [4, 5, 6].forEach(device => {
        const button = document.getElementById(`device${device}`);
        button.disabled = autoMode;
    });
}
function setThreshold(param) {
    const value = document.getElementById(param).value;
    fetch(`/set?param=${param}&value=${value}`)
        .then(response => response.text())
        .then(() => alert(`Ngưỡng ${param} đã được đặt thành ${value}`))
        .catch(error => console.error('Lỗi khi đặt ngưỡng:', error));
}

// Cập nhật nhiệt độ, độ ẩm và trạng thái thiết bị mỗi giây
setInterval(() => {
    // Gửi yêu cầu đến server để lấy dữ liệu
    Promise.all([
        fetch('/data').then(response => response.json()),
        fetch('/dataW').then(response => response.json())
    ])
    .then(([data, dataW]) => {
        // Cập nhật dữ liệu từ '/data'
        document.getElementById('temp').innerText = data.temperature + ' °C';
        document.getElementById('humidity').innerText = data.humidity + ' %';
        document.getElementById("auto").checked = data.autoMode === "ON";
        autoMode = data.autoMode === "ON";
        ennableBtn();
        // Cập nhật dữ liệu từ '/dataW'
        document.getElementById('tempHighLabel').textContent  = "Ngưỡng nhiệt độ trên: " + dataW.tempHigh + " |";
        document.getElementById('tempLowLabel').textContent  = "Ngưỡng nhiệt độ dưới: " + dataW.tempLow + " |";
        document.getElementById("humiLabel").textContent  = "Ngưỡng độ ẩm: " + dataW.humiW +" |";
    })
    .catch(error => {
        console.error('Lỗi khi cập nhật dữ liệu:', error);
    });

    // Cập nhật trạng thái thiết bị
    updateDeviceStates();
    
}, 1000);
    </script>
</body>

</html>
)rawliteral";

void setup()
{
    Serial.begin(115200);
    // Cấu hình chân vào ra
    pinMode(fan, OUTPUT);
    pinMode(humidifier, OUTPUT);
    pinMode(temps, OUTPUT);
    pinMode(out1, OUTPUT);
    pinMode(out2, OUTPUT);
    pinMode(out3, OUTPUT);
    pinMode(in1, INPUT);
    pinMode(in2, INPUT);
    pinMode(in2, INPUT);
    // đưa các chân đầu ra khi khởi động ở mức thấp
    digitalWrite(fan, 1);
    digitalWrite(humidifier, 1);
    digitalWrite(temps, 1);
    digitalWrite(out1, 1);
    digitalWrite(out2, 1);
    digitalWrite(out3, 1);
    lcd.init();
    lcd.backlight();
    lcd.clear();
    lcd.print("ESP 32 ");
    if (!EEPROM.begin(EEPROM_SIZE))
    {
        Serial.println("Failed to initialize EEPROM");
        return;
    }
    tempHW = readByteFromEEPROM(0);
    tempLW = readByteFromEEPROM(1);
    humiW = readByteFromEEPROM(2);
    Serial.println(tempHW);
    Serial.println(tempLW);
    Serial.println(humiW);
    WiFi.softAP(ssid, password);
    Serial.println("WiFi Access Point started");

    server.on("/", HTTP_GET, [](AsyncWebServerRequest *request)
              { request->send_P(200, "text/html", index_html); });
    server.on("/states", HTTP_GET, [](AsyncWebServerRequest *request)
              {
    String json = "{";
    for (int i = 0; i < 6; i++) {
        json += "\"device" + String(i + 1) + "\":\"" + (deviceStates[i] ? "ON" : "OFF") + "\"";
        if (i < 5) json += ","; // Thêm dấu phẩy giữa các phần tử
    }
    json += "}";
    request->send(200, "application/json", json); });

    server.on("/data", HTTP_GET, [](AsyncWebServerRequest *request)
              {
                  String json = "{";
                  json += "\"temperature\":" + String(temperature) + ","; // Giá trị số không cần dấu ngoặc kép
                  json += "\"humidity\":" + String(humidity) + ",";
                  json += "\"autoMode\":\"" + String(autoMode ? "ON" : "OFF") + "\""; // Giá trị chuỗi cần dấu ngoặc kép
                  json += "}";
                  request->send(200, "application/json", json); // Gửi JSON
              });

    server.on("/dataW", HTTP_GET, [](AsyncWebServerRequest *request)
              {
                  String json = "{";
                  json += "\"tempHigh\":" + String(tempHW) + ","; // Giá trị số
                  json += "\"tempLow\":" + String(tempLW) + ",";
                  json += "\"humiW\":" + String(humiW); // Giá trị số
                  json += "}";
                  request->send(200, "application/json", json); // Gửi JSON
              });

    server.on("/control", HTTP_GET, [](AsyncWebServerRequest *request)
              {
        int device = request->getParam("device")->value().toInt();
        String state = request->getParam("state")->value();
        deviceStates[device - 1] = (state == "ON");
        Serial.printf("Device %d set to %s\n", device, state.c_str());
        request->send(200, "text/plain", "OK"); });
    server.on("/setAutoMode", HTTP_GET, [](AsyncWebServerRequest *request)
              {
    String autoModeValue = request->getParam("value")->value();
    autoMode = (autoModeValue == "true");
    Serial.printf("Auto Mode set to %s\n", autoMode ? "ON" : "OFF");
    request->send(200, "text/plain", "OK"); });

    server.on("/set", HTTP_GET, [](AsyncWebServerRequest *request)
              {
        String param = request->getParam("param")->value();
        String value = request->getParam("value")->value();
        if (param == "tempHigh")
        {
            if (tempHW != value.toInt())
            {
                tempHW = value.toInt();
                saveByteToEEPROM(0, tempHW);
                Serial.println("Save temph");
            }
        }
        else if (param == "tempLow")
        {
            if (tempLW != value.toInt())
            {
                tempLW = value.toInt();
                saveByteToEEPROM(1, tempLW);
                Serial.println("Save templ");
            }
        }
        else if (param == "humidityThreshold")
        {
            if (humiW != value.toInt())
            {
                humiW = value.toInt();
                saveByteToEEPROM(2, humiW);
                Serial.println("Save humiw");
            }
        }
        else
        {
            // Xử lý trường hợp không có param hợp lệ
            Serial.println("Invalid parameter");
        }
            Serial.printf("Set %s to %s\n", param.c_str(), value.c_str());
            request->send(200, "text/plain", "OK"); });

    server.begin();
    Serial.println("Server started");
}
int startTime = 0;
void loop()
{
    if (millis() - startTime > 1000)
    {
        startTime = millis();
        get_data_Sensor();
        print_lcd();
    }
    read_button();
    controlDevice();
}
void print_lcd()
{
    // Hiển thị lên LCD
    lcd.setCursor(0, 0);
    lcd.print("Temp  : ");
    lcd.print(abs(temperature));
    lcd.print((char)223);
    lcd.print("C     ");
    lcd.setCursor(0, 1);
    lcd.print("Humid : ");
    lcd.print(humidity);
    lcd.print("%H       ");
}
void get_data_Sensor()
{
    Sensor.UpdateData();
    if (Sensor.GetTemperature() != 0 || Sensor.GetRelHumidity() != 0)
    {
        temperature = Sensor.GetTemperature();
        temperature = round(temperature * 10) / 10;
        humidity = Sensor.GetRelHumidity();
        humidity = round(humidity * 10) / 10;
    }
}
void controlDevice()
{
    if (autoMode)
    {

        if (tempHW != 0 & temperature > tempHW)
        {
            deviceStates[3] = true;
            deviceStates[4] = false;
        }
        else if (tempHW != 0 & temperature < tempLW)
        {
            deviceStates[4] = true;
            deviceStates[3] = false;
        }
        else
        {
            deviceStates[3] = false;
            deviceStates[4] = false;
        }
        if (humiW != 0 & humidity < humiW)
        {
            deviceStates[5] = true;
        }
        else
        {
            deviceStates[5] = false;
        }
    }
    for (int i = 0; i < 6; i++)
    {
        digitalWrite(device[i], (deviceStates[i]) ? 0 : 1);
    }
}
void read_button()
{
    for (int i = 0; i < 3; i++)
    {
        if (digitalRead(input_btn[i]) == LOW)
        {
            stt_input[i] = 1;
            Serial.println("readbnt");
        }
        else
        {
            delay(100);
            if (stt_input[i] == 1 && digitalRead(input_btn[i]) == HIGH)
            {
                deviceStates[i] = (!deviceStates[i]);
                stt_input[i] = 0;
            }
        }
    }
}
void saveByteToEEPROM(int address, byte value)
{
    EEPROM.write(address, value); // Ghi byte
    EEPROM.commit();              // Áp dụng thay đổi
}

// Hàm đọc 1 byte từ EEPROM
byte readByteFromEEPROM(int address)
{
    return EEPROM.read(address); // Đọc byte
}