# IoT25-HW05

5-1은 서로 ble로 연결이 되었을 때 esp board 에 내장된 led가 2초마다 켜지고 꺼지고를 반복하게 만들었습니다.

## 5-1 서버 코드
#include <BLEDevice.h>
#include <BLEServer.h>

class MyServerCallbacks : public BLEServerCallbacks {
  void onConnect(BLEServer* pServer) override {
    Serial.println("✅ 클라이언트 연결됨!");
    for (int i = 0; i < 5; i++) {
      digitalWrite(LED_BUILTIN, HIGH);
      delay(2000);
      digitalWrite(LED_BUILTIN, LOW);
      delay(2000);
    }
    Serial.println("LED 깜빡임 완료");
  }

  void onDisconnect(BLEServer* pServer) override {
    Serial.println("🔌 클라이언트 연결 해제됨.");
  }
};

void setup() {
  Serial.begin(115200);
  while (!Serial) {
    delay(10);
  }

  Serial.println("BLE 서버 시작 중...");

  pinMode(LED_BUILTIN, OUTPUT);
  digitalWrite(LED_BUILTIN, LOW);

  BLEDevice::init("ESP32_BLE_SERVER");

  BLEServer* pServer = BLEDevice::createServer();
  pServer->setCallbacks(new MyServerCallbacks());

  BLEService* pService = pServer->createService("12345678-1234-1234-1234-1234567890ab");
  pService->start();

  BLEAdvertising* pAdvertising = BLEDevice::getAdvertising();
  pAdvertising->addServiceUUID("12345678-1234-1234-1234-1234567890ab");
  pAdvertising->start();

  Serial.println("BLE 광고 시작 완료");
}

void loop() {
  // nothing here; BLE event driven
}

## 5-1 클라이언트 코드
#include <BLEDevice.h>
#include <BLEUtils.h>
#include <BLEScan.h>
#include <BLEClient.h>

#define SERVICE_UUID "12345678-1234-1234-1234-1234567890ab"

BLEAdvertisedDevice* myDevice = nullptr;
BLEClient* pClient;
bool doConnect = false;
bool connected = false;

class MyAdvertisedDeviceCallbacks : public BLEAdvertisedDeviceCallbacks {
  void onResult(BLEAdvertisedDevice advertisedDevice) override {
    Serial.print("스캔 중: ");
    Serial.println(advertisedDevice.toString().c_str());

    if (advertisedDevice.haveServiceUUID() &&
        advertisedDevice.isAdvertisingService(BLEUUID(SERVICE_UUID))) {
      Serial.println("✅ 서버 발견!");
      myDevice = new BLEAdvertisedDevice(advertisedDevice);
      doConnect = true;
      BLEDevice::getScan()->stop();
    }
  }
};

void setup() {
  Serial.begin(115200);
  while (!Serial) {
    delay(10);
  }

  BLEDevice::init("");

  BLEScan* pBLEScan = BLEDevice::getScan();
  pBLEScan->setAdvertisedDeviceCallbacks(new MyAdvertisedDeviceCallbacks());
  pBLEScan->setActiveScan(true);
  Serial.println("🔎 서버 스캔 시작...");
  pBLEScan->start(10, false);  // 10초 스캔
}

void loop() {
  if (doConnect && !connected) {
    Serial.println("🔗 서버에 연결 시도...");
    pClient = BLEDevice::createClient();
    if (pClient->connect(myDevice)) {
      Serial.println("✅ 서버에 연결 성공!");
      connected = true;
    } else {
      Serial.println("❌ 서버 연결 실패!");
    }
    doConnect = false;
  }

  delay(1000);
}

## 5-1 동영상
https://github.com/user-attachments/assets/9a83543f-cbfa-4365-b110-558ff07a387e

## 5-2 코드
//라이브러리를 가져옵니다.
#include <BLEDevice.h>
#include <BLEServer.h>
#include <BLEUtils.h>
#include <BLE2902.h>
//서비스와 특성에 대한 UUID를 정의합니다.
#define SERVICE_UUID        "4fafc201-1fb5-459e-8fcc-c5c9c331914b"
#define RGB_CHARACTERISTIC_UUID "beb5483e-36e1-4688-b7f5-ea07361b26a8"
#define NOTYFY_CHARACTERISTIC_UUID "d501d731-f0f9-4121-a7f6-2c0f959f1583"
//RGB LED에 사용될 핀 번호를 정의합니다
#define RED_PIN   4
#define GREEN_PIN 15
#define BLUE_PIN  2
//BLE 관련 객체를 선언합니다.
BLEServer *pServer; //BLE 서버 객체
BLEService *pService; // BLE 서비스 객체
BLECharacteristic *pRGBCharacteristic = NULL; //BLE 특성 객체
BLECharacteristic* pCharacteristic = NULL; //BLE 특성 객체
//RGB 값을 저장할 변수들을 초기화합니
int redValue = 0;
int greenValue = 0;
int blueValue = 0;
//BLE 특성에 대한 콜백 클래스를 정의합니다.
class MyCallbacks : public BLECharacteristicCallbacks {
  //특성의 값을 읽어와 RGB 값을 파싱하고 LED를 제어합니다.
  void onWrite(BLECharacteristic *pCharacteristic) {
    String value = pCharacteristic->getValue().c_str(); //특성의 값을 문자열로 읽어옵니다.

    // 값이 유효한 경우에만 처리
    if (value.length() > 0) {
      // RGB 값을 파싱하여 각각의 변수에 저장
      int delimiterPos1 = value.indexOf(',');
      int delimiterPos2 = value.lastIndexOf(',');

      if (delimiterPos1 != -1 && delimiterPos2 != -1 \
             && delimiterPos2 < value.length() - 1) {
        redValue = value.substring(0, delimiterPos1).toInt();
        greenValue = value.substring(delimiterPos1 + 1, delimiterPos2).toInt();
        blueValue = value.substring(delimiterPos2 + 1).toInt();

        // RGB 값을 LED의 밝기로 변환하여 제어합니다.
        analogWrite(RED_PIN, 255 - redValue);
        analogWrite(GREEN_PIN, 255 - greenValue);
        analogWrite(BLUE_PIN, 255 - blueValue);
      }
    }
  }
};

void setup() {
  // BLE 초기화하고 이름을 "RGB LED Control"로 설정합니다
  BLEDevice::init("RGB LED Control");
  pServer = BLEDevice::createServer();
  pService = pServer->createService(SERVICE_UUID);
  
  // RGB 특성 생성 및 설정
  pRGBCharacteristic = pService->createCharacteristic(
    RGB_CHARACTERISTIC_UUID,
    BLECharacteristic::PROPERTY_WRITE
  );  
  pRGBCharacteristic->setCallbacks(new MyCallbacks());

  // Notify/Indicate 특성 생성 및 설정
  pCharacteristi정
![Image](https://github.com/user-attachments/assets/12ba2dc1-770b-465f-9802-57a9cf7d2d94)

![Image](https://github.com/user-attachments/assets/14d949df-71e9-4e50-b56a-4455112089a9)


