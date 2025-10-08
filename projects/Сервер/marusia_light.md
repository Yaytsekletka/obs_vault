## 1) Маруся (DIY Hooks)

- Навык типа **DIY Hooks** с тремя командами:
    
    - «включи свет» → HTTP POST на твой HA: `/api/services/mqtt/publish` с payload `{"topic":"home/room/light/set","payload":"ON","qos":1,"retain":false}`
        
    - «выключи свет» → то же, но payload `"OFF"`
        
    - (опц.) «переключи свет» → payload `"TOGGLE"`
        
- Заголовки:  
    `Authorization: Bearer <HA_LONG_LIVED_TOKEN>`  
    `Content-Type: application/json`
    
- Webhook URL указываешь на **твой домен** (HTTPS, за Nginx).
    

## 2) Выделенный сервер (публичный IP)

**Компоненты:**

- **Nginx** — принимает HTTPS с Маруси и проксирует в HA.
    
- **Home Assistant** — принимает HTTP API вызовы и публикует в MQTT.
    
- **Mosquitto (MQTT брокер)** — брокер, к которому подключается ESP32.
    

**Порты (наружу):**

- 443/tcp (HTTPS, Nginx)
    
- 1883/tcp (MQTT без TLS) — можно **не выставлять наружу**, если ESP32 всегда в интернете; но ему нужен доступ. Лучше ограничить FW по странам/IP или сразу настроить TLS (8883).
    
- 8883/tcp (MQTT TLS) — если делаешь TLS.
    

**Связи внутри сервера:**

- Nginx → HA (localhost:8123)
    
- HA → Mosquitto (localhost:1883)
    

**Быстрый `docker-compose.yml` (пример):**

`services:   homeassistant:     image: ghcr.io/home-assistant/home-assistant:stable     container_name: homeassistant     network_mode: host     volumes: [ "./ha/config:/config" ]     restart: unless-stopped    mosquitto:     image: eclipse-mosquitto:2     container_name: mosquitto     ports:       - "1883:1883"     # можно убрать наружу и оставить только внутри, если ESP32 доберется по VPN       # - "8883:8883"   # включишь при TLS     volumes:       - "./mosquitto/conf:/mosquitto/config"       - "./mosquitto/data:/mosquitto/data"       - "./mosquitto/log:/mosquitto/log"     restart: unless-stopped    nginx:     image: nginx:alpine     container_name: proxy     ports: [ "80:80", "443:443" ]     volumes:       - "./nginx/conf.d:/etc/nginx/conf.d"       - "./nginx/certs:/etc/nginx/certs"   # сюда положишь cert+key (или настроишь certbot отдельно)     depends_on: [ homeassistant ]     restart: unless-stopped`

**Пример Nginx-конфига (проксирование HA API):**

`server {   listen 443 ssl http2;   server_name myhome.example.com;    ssl_certificate     /etc/nginx/certs/fullchain.pem;   ssl_certificate_key /etc/nginx/certs/privkey.pem;    client_max_body_size 5m;    location / {     proxy_pass http://127.0.0.1:8123;     proxy_set_header Host $host;     proxy_set_header X-Real-IP $remote_addr;     proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;     proxy_set_header X-Forwarded-Proto https;   } }`

**HA ↔ Mosquitto (в `configuration.yaml`):**

`mqtt:   broker: 127.0.0.1   port: 1883   username: "mqtt_user"   password: "mqtt_pass"`

**Mosquitto конфиг (`mosquitto.conf`):**

`listener 1883 allow_anonymous false password_file /mosquitto/config/passwd persistence true persistence_location /mosquitto/data/`

(пароли создашь `mosquitto_passwd -c /.../passwd mqtt_user`)

## 3) ESP32 (твоя прошивка, Arduino core / ESP-IDF)

- Подключается **по MQTT к твоему серверу** (`myhome.example.com:1883` или `8883` при TLS).
    
- Подписывается на командный топик: `home/room/light/set`
    
- Публикует:
    
    - состояние: `home/room/light/state` (`ON`/`OFF`, retain=true)
        
    - доступность (LWT): `home/room/light/avail` (`online`/`offline`, retain=true)
        
- Управляет GPIO, включающим твердотельное реле.
    

Мини-каркас (как мы выше уже набросали) у тебя есть; важно добавить:

- LWT при `mqtt.connect(..., willTopic, willQos, willRetain, "offline")`
    
- `publish(".../avail","online",true)` после подключения
    
- QoS=1 на подписке и публикациях команд/состояния
    
- (опц.) публикацию **MQTT Discovery** для HA (тогда устройство появится как «переключатель» автоматически)
    

# Как всё взаимодействует

## Команда «включи свет»

1. Ты говоришь: «Маруся, включи свет».
    
2. Облако VK (DIY Hooks) отправляет HTTPS POST → `https://myhome.example.com/api/services/mqtt/publish`
    
    - Headers: `Authorization: Bearer <token>`, `Content-Type: application/json`
        
    - Body: `{"topic":"home/room/light/set","payload":"ON","qos":1,"retain":false}`
        
3. Nginx проксирует в Home Assistant.
    
4. HA выполняет сервис `mqtt.publish` → публикует в Mosquitto.
    
5. Mosquitto доставляет ESP32 сообщение (`ON`) по топику `home/room/light/set`.
    
6. ESP32 включает реле, публикует `home/room/light/state = ON` (retain=true).
    
7. (опц.) В HA обновляется entity «свет включён»; можно озвучить ответ из Маруси.
    

## Команда «выключи свет»

— то же самое, но payload `OFF`.

## Смена состояния вручную (кнопка/выключатель)

1. Ты нажимаешь радио-кнопку/локальную кнопку → ESP32 ловит вход и делает `TOGGLE` локально.
    
2. ESP32 **сам** публикует новое состояние в `home/room/light/state` (retain=true).
    
3. HA видит изменение состояния (даже если речь не шла о Марусе).
    
4. При следующем запросе Маруси о состоянии — ответ будет корректным (если настроишь ответное озвучивание).
    

# Топики и правила

- Команда: `home/room/light/set` — `ON`/`OFF`/`TOGGLE` (без retain)
    
- Состояние: `home/room/light/state` — `ON`/`OFF` (retain=true)
    
- Доступность (LWT): `home/room/light/avail` — `online`/`offline` (retain=true)
    
- QoS: 1 везде, где есть смысл.
    
- Имена/иерархию можно менять на свою (главное — консистентно).
    

# Безопасность (минимум)

- **HTTPS** для доступа к HA (Nginx + валидный сертификат).
    
- **HA Long-Lived Token** только в настройке DIY Hooks (не светить в репозиториях).
    
- **Mosquitto**: запрет анонимных подключений; сложные пароли; при необходимости — **TLS (8883)** и/или ограничение входа по IP/стране на уровне firewall.
    
- По возможности **не публиковать 1883 наружу**; если ESP32 всегда в интернете и не может в VPN/TLS — публикуй 1883, но строго firewall.
    

# Тесты и отладка

- Проверка MQTT «в обход» Маруси:
    
    - `mosquitto_pub -h 127.0.0.1 -u mqtt_user -P mqtt_pass -t home/room/light/set -m ON`
        
    - `mosquitto_sub -t home/room/light/#` — видишь `state`/`avail`
        
- Проверка HA API:
    
    - `curl -H "Authorization: Bearer <token>" -H "Content-Type: application/json" \ -d '{"topic":"home/room/light/set","payload":"OFF","qos":1,"retain":false}' \ https://myhome.example.com/api/services/mqtt/publish`
        
- Логи:
    
    - `docker logs mosquitto`
        
    - `docker logs homeassistant`
        
    - `docker logs proxy`