<!DOCTYPE html>
<html lang="es">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
  <title>RamCar Web Bluetooth</title>
  <style>
    * { box-sizing: border-box; user-select: none; -webkit-user-select: none; }
    body {
      background: #111; color: #eee; font-family: sans-serif;
      display: flex; flex-direction: column; align-items: center; justify-content: center;
      height: 100vh; margin: 0;
    }
    #btnConnect {
      padding: 12px 24px; font-size: 16px; border-radius: 8px;
      border: none; background: #28a745; color: white; margin-bottom: 25px;
    }
    .grid { display: grid; grid-template-columns: repeat(3, 80px); gap: 12px; }
    .btn {
      height: 80px; font-size: 26px; border-radius: 12px; border: 2px solid #333;
      background: #222; color: #fff; display: flex; align-items: center; justify-content: center;
    }
    .btn:active, .btn.active { background: #007bff; border-color: #007bff; }
    #up    { grid-column: 2; grid-row: 1; }
    #left  { grid-column: 1; grid-row: 2; }
    #right { grid-column: 3; grid-row: 2; }
    #down  { grid-column: 2; grid-row: 3; }
  </style>
</head>
<body>

  <button id="btnConnect">Conectar RamCar (BLE)</button>

  <div class="grid">
    <button class="btn" id="up" data-cmd="F">▲</button>
    <button class="btn" id="left" data-cmd="L">◀</button>
    <button class="btn" id="right" data-cmd="R">▶</button>
    <button class="btn" id="down" data-cmd="B">▼</button>
  </div>

  <script>
    const SERVICE_UUID        = "4fafc201-1fb5-459e-8fcc-c5c9c331914b";
    const CHARACTERISTIC_UUID = "beb5483e-36e1-4688-b7f5-ea07361b26a8";

    let bleCharacteristic = null;
    let heartbeatTimer = null;
    let activeCmd = null;

    const btnConnect = document.getElementById("btnConnect");

    // 1. Emparejamiento por Web Bluetooth
    btnConnect.addEventListener("click", async () => {
      try {
        const device = await navigator.bluetooth.requestDevice({
          filters: [{ name: "RamCar_BLE" }],
          optionalServices: [SERVICE_UUID]
        });

        device.addEventListener("gattserverdisconnected", onDisconnected);

        const server = await device.gatt.connect();
        const service = await server.getPrimaryService(SERVICE_UUID);
        bleCharacteristic = await service.getCharacteristic(CHARACTERISTIC_UUID);

        btnConnect.textContent = "Conectado";
        btnConnect.style.background = "#007bff";
      } catch (error) {
        console.error("Fallo al conectar BLE:", error);
      }
    });

    function onDisconnected() {
      btnConnect.textContent = "Reconectar";
      btnConnect.style.background = "#dc3545";
      bleCharacteristic = null;
      detener();
    }

    // 2. Envío de datos binarios directos
    async function enviarComando(texto) {
      if (!bleCharacteristic) return;
      try {
        const encoder = new TextEncoder();
        // writeValueWithoutResponse minimiza la latencia (no espera ACK)
        await bleCharacteristic.writeValueWithoutResponse(encoder.encode(texto));
      } catch (e) {
        console.error("Error al escribir BLE:", e);
      }
    }

    function iniciar(cmd, btn) {
      if (activeCmd === cmd) return;
      activeCmd = cmd;
      btn.classList.add("active");

      enviarComando(cmd);

      clearInterval(heartbeatTimer);
      heartbeatTimer = setInterval(() => enviarComando("H"), 250);
    }

    function detener() {
      if (!activeCmd) return;
      activeCmd = null;

      clearInterval(heartbeatTimer);
      heartbeatTimer = null;

      document.querySelectorAll(".btn").forEach(b => b.classList.remove("active"));
      enviarComando("S");
    }

    // 3. Listeners táctiles y mouse
    document.querySelectorAll(".btn").forEach(btn => {
      const cmd = btn.getAttribute("data-cmd");

      btn.addEventListener("touchstart", (e) => { e.preventDefault(); iniciar(cmd, btn); }, { passive: false });
      btn.addEventListener("touchend", (e) => { e.preventDefault(); detener(); }, { passive: false });
      btn.addEventListener("touchcancel", (e) => { e.preventDefault(); detener(); }, { passive: false });

      btn.addEventListener("mousedown", () => iniciar(cmd, btn));
      btn.addEventListener("mouseup", detener);
      btn.addEventListener("mouseleave", detener);
    });
  </script>
</body>
</html>

void loop() {
  // Watchdog de seguridad
  if (deviceConnected && (millis() - lastCommandTime > TIMEOUT_MOTORES)) {
    pararMotores();
  }
}
