<!DOCTYPE html>
<html lang="es">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
  <title>RamCar Web Bluetooth</title>
  <style>
    * { box-sizing: border-box; -webkit-user-select: none; user-select: none; }
    body {
      background: #111;
      color: #fff;
      font-family: sans-serif;
      display: flex;
      flex-direction: column;
      align-items: center;
      justify-content: center;
      height: 100vh;
      margin: 0;
      touch-action: manipulation;
    }
    #btnConnect {
      padding: 12px 24px;
      font-size: 16px;
      font-weight: bold;
      border: none;
      border-radius: 8px;
      background: #00e676;
      color: #000;
      cursor: pointer;
      margin-bottom: 25px;
    }
    #btnConnect.connected { background: #ff5252; color: #fff; }
    .pad {
      display: grid;
      grid-template-columns: repeat(3, 85px);
      grid-template-rows: repeat(3, 85px);
      gap: 12px;
    }
    .btn {
      background: #222;
      border: 2px solid #444;
      border-radius: 16px;
      color: #fff;
      font-size: 28px;
      display: flex;
      align-items: center;
      justify-content: center;
    }
    .btn:active, .btn.active { background: #007bff; border-color: #007bff; }
    #up    { grid-column: 2; grid-row: 1; }
    #left  { grid-column: 1; grid-row: 2; }
    #right { grid-column: 3; grid-row: 2; }
    #down  { grid-column: 2; grid-row: 3; }
  </style>
</head>
<body>

  <button id="btnConnect">Conectar BLE</button>

  <div class="pad">
    <button class="btn" id="up" data-cmd="F">▲</button>
    <button class="btn" id="left" data-cmd="L">◀</button>
    <button class="btn" id="right" data-cmd="R">▶</button>
    <button class="btn" id="down" data-cmd="B">▼</button>
  </div>

  <script>
    const SERVICE_UUID = "4fafc201-1fb5-459e-8fcc-c5c9c331914b";
    const CHARACTERISTIC_UUID = "beb5483e-36e1-4688-b7f5-ea07361b26a8";

    let bleDevice = null;
    let bleCharacteristic = null;
    const btnConnect = document.getElementById("btnConnect");

    // Función para conectar el navegador al ESP32 por BLE
    async function toggleConnection() {
      if (bleDevice && bleDevice.gatt.connected) {
        bleDevice.gatt.disconnect();
        return;
      }

      try {
        // 1. Abre el diálogo nativo del sistema para elegir el dispositivo
        bleDevice = await navigator.bluetooth.requestDevice({
          filters: [{ name: "RamCar_BLE" }],
          optionalServices: [SERVICE_UUID]
        });

        bleDevice.addEventListener('gattserverdisconnected', onDisconnected);

        // 2. Conectar al servidor GATT
        const server = await bleDevice.gatt.connect();

        // 3. Obtener el servicio y la característica de control
        const service = await server.getPrimaryService(SERVICE_UUID);
        bleCharacteristic = await service.getCharacteristic(CHARACTERISTIC_UUID);

        btnConnect.textContent = "Desconectar";
        btnConnect.classList.add("connected");
      } catch (error) {
        console.error("Error al conectar BLE:", error);
      }
    }

    function onDisconnected() {
      btnConnect.textContent = "Conectar BLE";
      btnConnect.classList.remove("connected");
      bleCharacteristic = null;
    }

    // Envío del comando por Bluetooth
    function sendCommand(cmd) {
      if (bleCharacteristic) {
        const encoder = new TextEncoder();
        // writeValueWithoutResponse ofrece la menor latencia posible
        bleCharacteristic.writeValueWithoutResponse(encoder.encode(cmd))
          .catch(err => console.error("Error al enviar:", err));
      }
    }

    btnConnect.addEventListener("click", toggleConnection);

    // Asignación de eventos Press / Release
    document.querySelectorAll(".btn").forEach(btn => {
      const cmd = btn.getAttribute("data-cmd");

      const start = (e) => {
        e.preventDefault();
        btn.classList.add("active");
        sendCommand(cmd);
      };

      const stop = (e) => {
        e.preventDefault();
        btn.classList.remove("active");
        sendCommand("S");
      };

      btn.addEventListener("touchstart", start, { passive: false });
      btn.addEventListener("touchend", stop, { passive: false });
      btn.addEventListener("touchcancel", stop, { passive: false });

      btn.addEventListener("mousedown", start);
      btn.addEventListener("mouseup", stop);
      btn<Image alt="Arquitectura BLE y pila GATT para microcontroladores ESP32" caption="Pila Bluetooth Low Energy en ESP32" src="image_agent_tag_12711553090804823311"/>

---

Para comunicar una interfaz web con el ESP32-C3 mediante Bluetooth Low Energy (BLE), se define un **Servicio GATT** y una **Característica** identificados por UUIDs coincidentes en ambos extremos. El navegador escribe bytes en esa característica y el ESP32 reacciona mediante un callback de interrupción.

---

### 1. Código ESP32-C3 (Servidor BLE con Watchdog)

Este sketch no levanta Wi-Fi. Crea un periférico BLE llamado `"RamCar_BLE"` y escucha escrituras directas sobre los pines de tu puente H:

```cpp
#include <Arduino.h>
#include <BLEDevice.h>
#include <BLEUtils.h>
#include <BLEServer.h>

// UUIDs personalizados (deben coincidir con el frontend JS)
#define SERVICE_UUID        "4fafc201-1fb5-459e-8fcc-c5c9c331914b"
#define CHARACTERISTIC_UUID "beb5483e-36e1-4688-b7f5-ea07361b26a8"

// Pines puente H en ESP32-C3 Super Mini
const int PIN_IN1 = 2;
const int PIN_IN2 = 3;
const int PIN_IN3 = 4;
const int PIN_IN4 = 5;

unsigned long lastCommandTime = 0;
const unsigned long TIMEOUT_MOTORES = 800;
bool deviceConnected = false;

void pararMotores() {
  digitalWrite(PIN_IN1, LOW);
  digitalWrite(PIN_IN2, LOW);
  digitalWrite(PIN_IN3, LOW);
  digitalWrite(PIN_IN4, LOW);
}

void moverAdelante() {
  digitalWrite(PIN_IN1, HIGH);
  digitalWrite(PIN_IN2, LOW);
  digitalWrite(PIN_IN3, HIGH);
  digitalWrite(PIN_IN4, LOW);
}

void moverAtras() {
  digitalWrite(PIN_IN1, LOW);
  digitalWrite(PIN_IN2, HIGH);
  digitalWrite(PIN_IN3, LOW);
  digitalWrite(PIN_IN4, HIGH);
}

void girarIzquierda() {
  digitalWrite(PIN_IN1, LOW);
  digitalWrite(PIN_IN2, HIGH);
  digitalWrite(PIN_IN3, HIGH);
  digitalWrite(PIN_IN4, LOW);
}

void girarDerecha() {
  digitalWrite(PIN_IN1, HIGH);
  digitalWrite(PIN_IN2, LOW);
  digitalWrite(PIN_IN3, LOW);
  digitalWrite(PIN_IN4, HIGH);
}

// Callback de conexión/desconexión
class ServerCallbacks: public BLEServerCallbacks {
    void onConnect(BLEServer* pServer) {
      deviceConnected = true;
    };

    void onDisconnect(BLEServer* pServer) {
      deviceConnected = false;
      pararMotores();
      // Reiniciar publicidad para permitir reconexiones
      pServer->getAdvertising()->start();
    }
};

// Callback al recibir datos desde el navegador
class ControlCallbacks: public BLECharacteristicCallbacks {
    void onWrite(BLECharacteristic *pCharacteristic) {
      String value = pCharacteristic->getValue();

      if (value.length() > 0) {
        char cmd = value[0];
        lastCommandTime = millis();

        switch (cmd) {
          case 'F': moverAdelante();   break;
          case 'B': moverAtras();      break;
          case 'L': girarIzquierda();  break;
          case 'R': girarDerecha();    break;
          case 'S': pararMotores();    break;
          case 'H': /* Heartbeat */     break;
        }
      }
    }
};

void setup() {
  Serial.begin(115200);

  pinMode(PIN_IN1, OUTPUT);
  pinMode(PIN_IN2, OUTPUT);
  pinMode(PIN_IN3, OUTPUT);
  pinMode(PIN_IN4, OUTPUT);
  pararMotores();

  // Inicializar BLE
  BLEDevice::init("RamCar_BLE");
  BLEServer *pServer = BLEDevice::createServer();
  pServer->setCallbacks(new ServerCallbacks());

  BLEService *pService = pServer->createService(SERVICE_UUID);

  BLECharacteristic *pCharacteristic = pService->createCharacteristic(
                                         CHARACTERISTIC_UUID,
                                         BLECharacteristic::PROPERTY_WRITE |
                                         BLECharacteristic::PROPERTY_WRITE_NR
                                       );

  pCharacteristic->setCallbacks(new ControlCallbacks());
  pService->start();

  // Configurar y encender anuncio (Advertising)
  BLEAdvertising *pAdvertising = BLEDevice::getAdvertising();
  pAdvertising->addServiceUUID(SERVICE_UUID);
  pAdvertising->setScanResponse(true);
  pAdvertising->setMinPreferred(0x06);
  pAdvertising->setMinPreferred(0x12);
  BLEDevice::startAdvertising();

  Serial.println("BLE RamCar iniciado y visible.");
}

void loop() {
  // Watchdog de seguridad
  if (deviceConnected && (millis() - lastCommandTime > TIMEOUT_MOTORES)) {
    pararMotores();
  }
}
