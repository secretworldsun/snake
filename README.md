# snake
Игра из репозитория <https://github.com/Sidhu1504/snake-game-docker-compose> поднята
контейнером на сервере и доступна по адресу **https://snake.iakh.ru/** с сертификатом Let's Encrypt.
Сертификат выпущен и продлевается автоматически, без ручного запуска certbot.

* сервер: `176.53.255.143`, Ubuntu 26.04, Docker 29.1.3
* домен: `iakh.ru` (DNS — Selectel), новая запись `snake`
* e-mail для Let's Encrypt: `secret_world_sun@mail.ru`
* рабочий каталог игры на сервере: `/srv/snake`
* каталог стека прокси: `/srv/proxy`

## 1. Что представляет собой игра

| Файл | Роль |
|---|---|
| `app.py` | Flask-приложение, один маршрут `/` → `render_template("index.html")`, слушает **:5000** |
| `templates/index.html` | страница с `<canvas>` и кнопкой Restart |
| `static/script.js` | вся логика змейки (canvas, еда, счёт, setInterval 150 мс) |
| `static/styles.css` | оформление |
| `Dockerfile` | `python:3.10-slim` + `pip install -r requirements.txt` (Flask 2.2.5), `CMD python app.py` |
| `docker-compose.yml` | исходный: `build .`, `ports: 5000:5000`, `volumes: .:/app` |

## 2. Что изменено в конфигурации репозитория

Исходный `docker-compose.yml` для прокси не подходит: он публикует порт 5000 наружу
(в обход HTTPS) и не сообщает прокси, на каком домене и порту живёт контейнер.

```yaml
name: snake

services:
  snake_game:
    build:
      context: .
      dockerfile: Dockerfile
    image: snake-game:1.0
    container_name: snake-game
    environment:
      VIRTUAL_HOST: snake.iakh.ru
      VIRTUAL_PORT: 5000
      LETSENCRYPT_HOST: snake.iakh.ru
      LETSENCRYPT_EMAIL: secret_world_sun@mail.ru
    expose:
      - 5000
    networks:
      - proxy
    restart: unless-stopped

networks:
  proxy:
    external: true
    name: proxy_default
```

Ключевые отличия:

* **`ports` убраны, вместо них `expose: 5000`** — наружу порт не публикуется, трафик
  идёт только через `nginx-proxy`, который терминирует TLS.
* **`VIRTUAL_PORT: 5000`** — обязателен, потому что Flask слушает не 80-й порт.
* **`LETSENCRYPT_HOST` + `LETSENCRYPT_EMAIL`** — по этим переменным `acme-companion`
  понимает, на какое имя и на какой e-mail выпускать сертификат.

## 3. Развёртывание

```bash
ssh root@176.53.255.143
apt-get install -y git
git clone --depth 1 https://github.com/Sidhu1504/snake-game-docker-compose.git /srv/snake
cd /srv/snake
mv docker-compose.yml docker-compose.orig.yml
nano docker-compose.yaml     # замена конфигурации из раздела 2
docker compose up -d --build
docker compose ps
docker logs -f acme-companion
```

Проверка:

```bash
curl -s -o /dev/null -w '%{http_code}\n' https://snake.iakh.ru/
```

## 5. Результат

```
NAME         IMAGE            COMMAND           SERVICE      STATUS       PORTS
snake-game   snake-game:1.0   "python app.py"   snake_game   Up           5000/tcp
```

Сертификат выпустился сам, сразу после появления контейнера в сети прокси:

```
Creating/renewal snake.iakh.ru certificates... (snake.iakh.ru)
[22:00:30] Getting webroot for domain='snake.iakh.ru'
[22:00:31] Verifying: snake.iakh.ru
[22:00:37] Success
[22:00:40] Next renewal time picked from ARI window: 2027-01-15T14:17:38Z
[22:00:40] Installing full chain to: /etc/nginx/certs/snake.iakh.ru/fullchain.pem
```

| Проверка | Ответ |
|---|---|
| `http://snake.iakh.ru/` | 301 |
| `https://snake.iakh.ru/` | 200 |

Сертификат: `subject=CN=snake.iakh.ru`, `issuer=C=US, O=Let's Encrypt, CN=YR1`,
`notAfter=Feb 14 21:02:07 2027 GMT`, SAN `DNS:snake.iakh.ru`.
Продление — автоматически, по ARI-окну 15.01.2027.
