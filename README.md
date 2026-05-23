``## Test 23:

### Directory Structure
> Продемонструвати розуміння Event-Driven архітектури. А саме налаштувати базової конфігурацію сервісу MQTT-брокер EMQX, розгорнувши два веб-сервіси на HTML/JS для обміну динамічними повідомленнями в реальному часі. Продемонстровано успішну крос-протокольну взаємодію між браузерними клієнтами та Postman. Відповідно до структури наведеної нище.

```
├── exam
│   ├── src
│   │   ├── web1 
│   │   │   ├── index.html
│   │   ├── web2 
│   │   │   ├── index.html  
│   ├── <message>.conf  
│   ├── docker-compose.yml
│   ├── .editorconfig
│   ├── .gitignore
│   ├── package.json
│   ├── README.md

```version: '3.8'

services:
  emqx:
    image: emqx/emqx:5.3.0
    container_name: emqx_broker
    ports:
      - "1883:1883"   # MQTT TCP
      - "8083:8083"   # MQTT over WebSockets
      - "18083:18083" # EMQX Dashboard / HTTP API
    volumes:
      - ./emqx.conf:/opt/emqx/etc/emqx.conf

  web_services:
    image: halverneus/static-file-server:latest
    container_name: web_clents_server
    ports:
      - "8080:8080"
    volumes:
      - ./src:/web
    environment:
      - PORT=8080
# Базові налаштування брокера
node.name = "emqx@127.0.0.1"
node.cookie = "emqx_secret_cookie"

# Дозволити анонімний доступ (тільки для розробки/тестування!)
allow_anonymous = true

# Конфігурація лістенерів
listeners.tcp.default {
  bind = "0.0.0.0:1883"
  max_connections = 102400
}

listeners.ws.default {
  bind = "0.0.0.0:8083"
  max_connections = 102400
}
<!DOCTYPE html>
<html lang="uk">
<head>
    <meta charset="UTF-8">
    <title>Веб-сервіс 1 (Генератор даних)</title>
    <script src="https://unpkg.com/mqtt/dist/mqtt.min.js"></script>
    <style>
        body { font-family: sans-serif; margin: 30px; background: #f4f6f9; }
        .container { background: white; padding: 20px; border-radius: 8px; box-shadow: 0 4px 6px rgba(0,0,0,0.1); }
        .log { background: #222; color: #0f0; padding: 15px; height: 200px; overflow-y: scroll; font-family: monospace; }
        button { padding: 10px 20px; background: #007bff; color: white; border: none; border-radius: 4px; cursor: pointer; }
    </style>
</head>
<body>
    <div class="container">
        <h2>Сервіс 1: Клієнт WebSockets</h2>
        <p>Статус: <strong id="status" style="color: red;">Відключено</strong></p>
        <button onclick="sendEvent()">Надіслати подію (sensor/data)</button>
        
        <h3>Вхідні сповіщення (alerts/web1):</h3>
        <div id="log" class="log"></div>
    </div>

    <script>
        // Підключення до EMQX через WebSocket (порт 8083)
        const client = mqtt.connect('ws://localhost:8083/mqtt');
        const logDiv = document.getElementById('log');

        client.on('connect', () => {
            document.getElementById('status').innerText = 'Підключено до EMQX';
            document.getElementById('status').style.color = 'green';
            client.subscribe('alerts/web1');
        });

        client.on('message', (topic, message) => {
            logDiv.innerHTML += `[${new Date().toLocaleTimeString()}] Топік: ${topic} -> ${message.toString()}<br>`;
            logDiv.scrollTop = logDiv.scrollHeight;
        });

        function sendEvent() {
            const payload = JSON.stringify({ temperature: (Math.random() * 10 + 20).toFixed(1), timestamp: Date.now() });
            client.publish('sensor/data', payload);
            logDiv.innerHTML += `<span style="color: #aaa;">[Вихідне] Опубліковано в sensor/data: ${payload}</span><br>`;
        }
    </script>
</body>
</html>
<!DOCTYPE html>
<html lang="uk">
<head>
    <meta charset="UTF-8">
    <title>Веб-сервіс 2 (Обробник / Аліртер)</title>
    <script src="https://unpkg.com/mqtt/dist/mqtt.min.js"></script>
    <style>
        body { font-family: sans-serif; margin: 30px; background: #f4f6f9; }
        .container { background: white; padding: 20px; border-radius: 8px; box-shadow: 0 4px 6px rgba(0,0,0,0.1); }
        .log { background: #222; color: #00d2ff; padding: 15px; height: 200px; overflow-y: scroll; font-family: monospace; }
        button { padding: 10px 20px; background: #dc3545; color: white; border: none; border-radius: 4px; cursor: pointer; }
    </style>
</head>
<body>
    <div class="container">
        <h2>Сервіс 2: Клієнт WebSockets</h2>
        <p>Статус: <strong id="status" style="color: red;">Відключено</strong></p>
        <button onclick="sendAlert()">Надіслати тривогу (alerts/web1)</button>
        
        <h3>Потік даних в реальному часі (sensor/data):</h3>
        <div id="log" class="log"></div>
    </div>

    <script>
        const client = mqtt.connect('ws://localhost:8083/mqtt');
        const logDiv = document.getElementById('log');

        client.on('connect', () => {
            document.getElementById('status').innerText = 'Підключено до EMQX';
            document.getElementById('status').style.color = 'green';
            client.subscribe('sensor/data');
        });

        client.on('message', (topic, message) => {
            logDiv.innerHTML += `[${new Date().toLocaleTimeString()}] Отримано: ${message.toString()}<br>`;
            logDiv.scrollTop = logDiv.scrollHeight;
        });

        function sendAlert() {
            const payload = JSON.stringify({ msg: "Увага! Критичний рівень!", code: 500 });
            client.publish('alerts/web1', payload);
        }
    </script>
</body>
</html>
{
  "name": "mqtt-eda-exam",
  "version": "1.0.0",
  "description": "Event-Driven Architecture demonstration via EMQX",
  "scripts": {
    "infra:up": "docker-compose up -d",
    "infra:down": "docker-compose down",
    "infra:logs": "docker-compose logs -f"
  }
}
root = true

[*]
indent_style = space
indent_size = 2
end_of_line = lf
charset = utf-8
trim_trailing_whitespace = true
insert_final_newline = true
node_modules/
.DS_Store
# Демонстрація Event-Driven архітектури на базі EMQX

## Запуск проекту
1. Виконайте команду для підняття контейнерів:
```bash
   npm run infra:up
    depends_on:
      - emqx
