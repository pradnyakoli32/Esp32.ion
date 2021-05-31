# Esp32.ion


#include <ArduinoJson.h> //ArduinoJson V6
#include <EEPROM.h>
#include <BLEDevice.h>
#include <BLEServer.h>
#include <BLEUtils.h>
#include <BLE2902.h>   m3.
String password = "hello";
String receivedPassword = "";



// Name of this BLE device (can be changed)
std::string deviceName = "SMART LOCK";

BLEServer* pServer = NULL;
BLECharacteristic* pCharacteristic = NULL;
BLECharacteristic* pCharacteristicR = NULL;

bool deviceConnected = false;
bool oldDeviceConnected = false;

#define SERVICE_UUID        "afa2f203-348f-4718-9cf3-f7ff0de38472"
#define CHARACTERISTIC_UUID "450fcc81-dad5-46a6-959f-c5f41e37b669"



class MyServerCallbacks: public BLEServerCallbacks {
    void onConnect(BLEServer* pServer) {
      deviceConnected = true;
      BLEDevice::startAdvertising();
    };

    void onDisconnect(BLEServer* pServer) {
      deviceConnected = false;
    }
};


// Execute this when some data is receviced by esp32
#define CHARACTERISTIC_UUID_RX "10fd2313-62c2-4053-b3af-7e6ca407bf21"
class MyCallbacks: public BLECharacteristicCallbacks {

    void onWrite(BLECharacteristic *pCharacteristic) {
      std::string rxValue = pCharacteristic->getValue();

      if (rxValue.length() > 0) {
        Serial.println("*");
        Serial.print("Received Value: ");
        
        for (int i = 0; i < rxValue.length(); i++) {
          Serial.print(rxValue[i]);
          receivedPassword = receivedPassword+rxValue[i];
        }       
      }
        Serial.print('\n');
        if(receivedPassword == password) {
        Serial.println("right password");
        digitalWrite(LED_BUILTIN, HIGH);
        receivedPassword ="";
        }
        else {
           Serial.println("wrong password");
           receivedPassword ="";
        }
        Serial.println("*");

    }
};



void setup() {
  
  // initialize serial
  Serial.begin(115200); 
  setupBlueTooth();    
  pinMode(LED_BUILTIN, OUTPUT); 
  digitalWrite(LED_BUILTIN, LOW);             
  while (!Serial);
  Serial.println(" ESP32 SMART LOCK initialized !");
}

void loop() {

  //Executes code in loop

}



// Creates a json packet of data, serializes it and send out of esp32 over BLE
void sendBLE() {
  // JSON modeling the raw data 
  const int bufferSize = 4 *  JSON_OBJECT_SIZE(4);
  StaticJsonDocument<bufferSize> doc;
  
  doc["MessageID"] = incomingMsgId;
  doc["SenderID"]  = senderId;
  doc["Payload"] = payload;

  String jsonstat;
  Serial.println(jsonstat);
  serializeJson(doc, jsonstat);

  // Pretty print on Serial
  serializeJsonPretty(doc, Serial);

  size_t dataLen = jsonstat.length();
  pCharacteristic->setValue((uint8_t*)&jsonstat[0], dataLen);
  pCharacteristic->notify();
  
}

/**
   setupBlueTooth
*/
void setupBlueTooth()
{
  // Create the BLE Device
  BLEDevice::init(deviceName);

  // Create the BLE Server
  pServer = BLEDevice::createServer();
  pServer->setCallbacks(new MyServerCallbacks());

  // Create the BLE Service
  BLEService *pService = pServer->createService(SERVICE_UUID);

  // Create a BLE Characteristic
  pCharacteristic = pService->createCharacteristic(
                      CHARACTERISTIC_UUID,
                      BLECharacteristic::PROPERTY_READ   |
                      BLECharacteristic::PROPERTY_WRITE  |
                      BLECharacteristic::PROPERTY_NOTIFY |
                      BLECharacteristic::PROPERTY_INDICATE
                    );

  // Create a BLE Descriptor
  pCharacteristic->addDescriptor(new BLE2902());

  pCharacteristic->setCallbacks(new MyCallbacks());

  // Start the service
  pService->start();

  // Start advertising
  BLEAdvertising *pAdvertising = BLEDevice::getAdvertising();
  pAdvertising->addServiceUUID(SERVICE_UUID);
  pAdvertising->setScanResponse(false);
  pAdvertising->setMinPreferred(0x0);  // set value to 0x00 to not advertise this parameter
  BLEDevice::startAdvertising();
  Serial.println("Waiting a client connection to notify...");
}



// writes data to eeprom
void writeString(char add,String data)
{
  int _size = data.length();
  int i;
  for(i=0;i<_size;i++)
  {
    EEPROM.write(add+i,data[i]);
  }
  EEPROM.write(add+_size,'\0');   //Add termination null character for String Data
  EEPROM.commit();
}



// reads data from eeprom
String read_String(char add)
{
  int i;
  char data[100]; //Max 100 Bytes
  int len=0;
  unsigned char k;
  k=EEPROM.read(add);
  while(k != '\0' && len<500)   //Read until null character
  {    
    k=EEPROM.read(add+len);
    data[len]=k;
    len++;
  }
  data[len]='\0';
  return String(data);
}
