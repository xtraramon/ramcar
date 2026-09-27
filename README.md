<html lang="es">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
  <title>RamCar Control</title>
  <style>
    * {
      box-sizing: border-box;
      -webkit-touch-callout: none;
      -webkit-user-select: none;
      user-select: none;
      touch-action: manipulation;
    }
    body {
      background: #121212;
      color: #fff;
      font-family: sans-serif;
      display: flex;
      flex-direction: column;
      align-items: center;
      justify-content: center;
      height: 100vh;
      margin: 0;
    }
    #status {
      margin-bottom: 20px;
      padding: 6px 14px;
      border-radius: 20px;
      font-size: 14px;
      background: #222;
    }
    .connected { color: #00e676; }
    .disconnected { color: #ff5252; }

    .pad-grid {
      display: grid;
      grid-template-columns: repeat(3, 85px);
      grid-template-rows: repeat(3, 85px);
      gap: 12px;
    }
    .btn {
      background: #1f1f1f;
      border: 2px solid #333;
      border-radius: 16px;
      color: #fff;
      font-size: 28px;
      display: flex;
      align-items: center;
      justify-content: center;
      outline: none;
      cursor: pointer;
    }
    .btn:active { background: #007bff; border-color: #007bff; }
    
    #btnUp    { grid-column: 2; grid-row: 1; }
    #btnLeft  { grid-column: 1; grid-row: 2; }
    #btnStop  { 
      grid-column: 2; 
      grid-row: 2; 
      background: #dc3545; 
      border-color: #b02a37; 
      font-size: 22px; 
      font-weight: bold; 
    }
    #btnStop:active { background: #a71d2a; }
    #btnRight { grid-column: 3; grid-row: 2; }
    #btnDown  { grid-column: 2; grid-row: 3; }
  </style>
</head>
<body>

  <div id="status" class="disconnected">● Desconectado</div>

  <div class="pad-grid">
    <button class="btn" id="btnUp" onclick="enviarComando('F')">▲</button>
    <button class="btn" id="btnLeft" onclick="enviarComando('L')">◀</button>
    <button class="btn" id="btnStop" onclick="enviarComando('S')">■</button>
    <button class="btn" id="btnRight" onclick="enviarComando('R')">▶</button>
    <button class="btn" id="btnDown" onclick="enviarComando('B')">▼</button>
  </div>

  <script>
    let ws = null;
    const statusElem = document.getElementById("status");

    function initWebSocket() {
      const host = location.host || "192.168.4.1";
      ws = new WebSocket(`ws://${host}/ws`);

      ws.onopen = () => {
        statusElem.textContent = "● Conectado";
        statusElem.className = "connected";
      };

      ws.onclose = () => {
        statusElem.textContent = "● Reconectando...";
        statusElem.className = "disconnected";
        setTimeout(initWebSocket, 1500);
      };

      ws.onerror = () => ws.close();
    }

    // Ráfaga redundante de 3 disparos para asegurar entrega inmediata
    function enviarComando(cmd) {
      if (!ws || ws.readyState !== WebSocket.OPEN) return;
      
      ws.send(cmd);
      setTimeout(() => { if (ws.readyState === WebSocket.OPEN) ws.send(cmd); }, 30);
      setTimeout(() => { if (ws.readyState === WebSocket.OPEN) ws.send(cmd); }, 60);
    }

    window.addEventListener("load", initWebSocket);
  </script>
</body>
</html>
