# Небо над Украиной: онлайн-версия

Игра (один файл `public/index.html`) и маленький сервер `server.js`. Сервер:
- отдаёт саму игру;
- ведёт **таблицу лидеров** (лучший результат каждого игрока: ночей пережито, потом сбито);
- считает **онлайн** (игра шлёт сигнал раз в 20 с, онлайн = кто отметился за последние 60 с).

Зависимостей нет, нужен только **Node.js 18+**. Результаты хранятся в `data/scores.json`.

## Инструкция для того, кто разворачивает (человек или ИИ)

**Важно:** это не статический сайт. Netlify, GitHub Pages и подобные хостинги **не подойдут**, потому что нужен работающий Node.js-процесс. Подойдёт любой VPS (Ubuntu), Render, Railway, Fly.io или Docker.

### Вариант A. VPS (Ubuntu) + nginx + HTTPS

1. Установить Node.js 20:
   `curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash - && sudo apt install -y nodejs nginx`
2. Скопировать папку проекта, например в `/opt/nebo`, и проверить: `cd /opt/nebo && PORT=3000 node server.js`. Затем открыть `http://IP:3000`.
3. Запустить как службу через pm2:
   `sudo npm i -g pm2 && pm2 start server.js --name nebo && pm2 save && pm2 startup`
4. nginx: файл `/etc/nginx/sites-available/nebo`
   ```nginx
   server {
     server_name ВАШ-ДОМЕН;
     location / {
       proxy_pass http://127.0.0.1:3000;
       proxy_set_header Host $host;
       proxy_set_header X-Forwarded-For $remote_addr;
     }
   }
   ```
   Затем `sudo ln -s /etc/nginx/sites-available/nebo /etc/nginx/sites-enabled/ && sudo nginx -t && sudo systemctl reload nginx`.
5. DNS: у регистратора домена создать A-запись `@` (и `www`) на IP сервера.
6. HTTPS: `sudo apt install -y certbot python3-certbot-nginx && sudo certbot --nginx -d ВАШ-ДОМЕН`.

### Вариант B. Docker
```
docker build -t nebo .
docker run -d --name nebo -p 3000:3000 -v nebo-data:/app/data --restart unless-stopped nebo
```
Дальше nginx и HTTPS делаются так же, как в варианте A (шаги 4–6).

### Вариант C. Render / Railway
Создать Web Service из этой папки: Start command `node server.js`, переменная `PORT` подставится сама. Для сохранения таблицы подключить постоянный диск (Disk / Volume) и задать `DATA_DIR` = путь к нему, иначе таблица обнулится при перезапуске. Свой домен привязывается в настройках сервиса.

## Переменные окружения
| Имя | По умолчанию | Что делает |
|---|---|---|
| `PORT` | `3000` | порт сервера |
| `DATA_DIR` | `./data` | где хранить `scores.json` |

## API (для проверки)
- `GET /healthz`: отвечает `ok`
- `GET /api/online`: `{online}`
- `GET /api/top?id=...`: топ-50 и место игрока
- `POST /api/hb {id}`: сигнал «я в игре»
- `POST /api/score {id,run,name,nights,shot,launched}`: результат после каждой ночи
- `POST /api/name {id,name}`: сменить позывной (уникальный, 2–18 символов)

Есть базовая защита от накруток: лимиты запросов по IP, проверка правдоподобия (сбито ≤ запущено, не больше одной ночи за раз и не чаще раза в 15 с).

Если игра открыта просто файлом (без сервера), она работает как раньше, а таблица пишет «заработает на сайте».
Если игру держат на одном домене, а сервер на другом, перед игрой задайте `window.NEBO_API = 'https://api.домен'` (и включите CORS в nginx).
