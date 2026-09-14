### Задача 3

**Условия**
1. Разворачиваем веб-приложение 
2. Все компоненты запускаются как systemd-сервисы
3. Nginx проксирует запросы на backend
4. Приложение доступно по ` http://localhost `
5. Все сервисы запускаются автоматически при запуске
6. Настроены логи и мониторинг

> Использую OS ubuntu server 24.04
---
**Практика**

- Обновляем пакеты: ` sudo apt update && sudo apt upgrade -y `
- Устанавливаем PostgreSQL: ` sudo apt install postgresql postgresql-contrib -y `
- Устанавливаем Nginx: ` sudo apt install nginx -y `
- Устанавливаем Python и pip: ` sudo apt install python3 python3-pip python3-venv -y `
- Проверяем установки: ` python3 --version ` / ` psql --version ` / ` nginx -v `

- Запускаем postgresql: ` sudo systemctl start postgresql ` / ` sudo systemctl enable postgresql `
- Проверяем статус: ` systemctl status postgresql `
- Переключаем на пользователя postgresql: ` sudo -u postgres psql `

- Внутри psql:
```
-- Создаем базу данных
CREATE DATABASE myapp_db;

-- Создаем пользователя с паролем
CREATE USER myapp_user WITH ENCRYPTED PASSWORD 'StrongPass12345!';

-- Выдаем права пользователю на базу
GRANT ALL PRIVILEGES ON DATABASE myapp_db TO myapp_user;

-- Подключаемся к базе для дальнейших настроек
\c myapp_db

-- Выдаем права на схему public
GRANT ALL ON SCHEMA public TO myapp_user;

-- Выйти из psql
\q
```

- Пробуем подключится новым пользователем: ` psql -U myapp_user -d myapp_db -h localhost `
- Создаем директорию для приложения: ` sudo mkdir -p /opt/myapp ` / ` sudo chown $USER:$USER /opt/myapp `
- Создаем структуру папок: ` mkdir -p /opt/myapp/{backend,frontend,logs} `
- Переходим в backend: ` cd /opt/myapp/backend `

- Создаем виртуальное окружение: ` python3 -m venv venv ` / ` source venv/bin/activate `
- Устанавливаем зависимости: ` pip install fastapi uvicorn psycipg2-binary python-dotenv `
- Создаем файл конфигурации: ` nano .env `
```
DATABASE_URL=postgresql://myapp_user:StrongPass12345!@localhost:5432/myapp_db
APP_HOST=127.0.0.1
APP_PORT=8000
LOG_LEVEL=info
```

- Создаем приложение: ` nano main.py `
```
#!/usr/bin/env python3
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
import psycopg2
from psycopg2.extras import RealDictCursor
from dotenv import load_dotenv
import os
import logging
from datetime import datetime

# Загрузить переменные окружения
load_dotenv()

# Настроить логирование
logging.basicConfig(
    level=getattr(logging, os.getenv('LOG_LEVEL', 'info').upper()),
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)
logger = logging.getLogger(__name__)

# Инициализировать приложение
app = FastAPI(title="MyApp API", version="1.0.0")

# Настроить CORS (разрешить запросы с frontend)
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Модель данных
class Item(BaseModel):
    name: str
    description: str = None

# Функция подключения к БД
def get_db_connection():
    return psycopg2.connect(
        host="localhost",
        database=os.getenv('DATABASE_URL').split('@')[1].split(':')[0],
        user=os.getenv('DATABASE_URL').split('//')[1].split(':')[0],
        password=os.getenv('DATABASE_URL').split('@')[0].split(':')[2],
        cursor_factory=RealDictCursor
    )

# Создать таблицу при запуске
@app.on_event("startup")
async def startup_event():
    logger.info("Запуск приложения...")
    conn = get_db_connection()
    cur = conn.cursor()
    cur.execute("""
        CREATE TABLE IF NOT EXISTS items (
            id SERIAL PRIMARY KEY,
            name VARCHAR(100) NOT NULL,
            description TEXT,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )
    """)
    conn.commit()
    cur.close()
    conn.close()
    logger.info("Таблица items проверена/создана")

# API эндпоинты
@app.get("/")
async def root():
    return {"message": "MyApp API работает!", "timestamp": datetime.now().isoformat()}

@app.get("/api/items")
async def get_items():
    try:
        conn = get_db_connection()
        cur = conn.cursor()
        cur.execute("SELECT * FROM items ORDER BY created_at DESC")
        items = cur.fetchall()
        cur.close()
        conn.close()
        return {"items": items, "count": len(items)}
    except Exception as e:
        logger.error(f"Ошибка получения items: {e}")
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/api/items")
async def create_item(item: Item):
    try:
        conn = get_db_connection()
        cur = conn.cursor()
        cur.execute(
            "INSERT INTO items (name, description) VALUES (%s, %s) RETURNING *",
            (item.name, item.description)
        )
        new_item = cur.fetchone()
        conn.commit()
        cur.close()
        conn.close()
        logger.info(f"Создан item: {new_item['name']}")
        return {"item": new_item}
    except Exception as e:
        logger.error(f"Ошибка создания item: {e}")
        raise HTTPException(status_code=500, detail=str(e))

@app.delete("/api/items/{item_id}")
async def delete_item(item_id: int):
    try:
        conn = get_db_connection()
        cur = conn.cursor()
        cur.execute("DELETE FROM items WHERE id = %s RETURNING *", (item_id,))
        deleted_item = cur.fetchone()
        conn.commit()
        cur.close()
        conn.close()
        if deleted_item:
            logger.info(f"Удалён item: {deleted_item['name']}")
            return {"message": "Item удалён", "item": deleted_item}
        else:
            raise HTTPException(status_code=404, detail="Item не найден")
    except HTTPException:
        raise
    except Exception as e:
        logger.error(f"Ошибка удаления item: {e}")
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/api/health")
async def health_check():
    try:
        conn = get_db_connection()
        cur = conn.cursor()
        cur.execute("SELECT 1")
        cur.close()
        conn.close()
        return {"status": "healthy", "database": "connected"}
    except Exception as e:
        return {"status": "unhealthy", "database": "disconnected", "error": str(e)}
```

- Запускаем приложение: ` uvicorn main:app --host 127.0.0.1 --port 8000 `
**Ошибка при запуске**
```
INFO:     Started server process [236635]
INFO:     Waiting for application startup.
2026-09-14 09:09:29,383 - main - INFO - Запуск приложения...
ERROR:    Traceback (most recent call last):
  File "/opt/myapp/backend/venv/lib/python3.12/site-packages/starlette/routing.py", line 648, in lifespan
    async with self.lifespan_context(app) as maybe_state:
  File "/opt/myapp/backend/venv/lib/python3.12/site-packages/fastapi/routing.py", line 265, in __aenter__
    await self._router._startup()
  File "/opt/myapp/backend/venv/lib/python3.12/site-packages/fastapi/routing.py", line 6375, in _startup
    await handler()
  File "/opt/myapp/backend/main.py", line 53, in startup_event
    conn = get_db_connection()
           ^^^^^^^^^^^^^^^^^^^
  File "/opt/myapp/backend/main.py", line 41, in get_db_connection
    return psycopg2.connect(
           ^^^^^^^^^^^^^^^^^
  File "/opt/myapp/backend/venv/lib/python3.12/site-packages/psycopg2/__init__.py", line 122, in connect
    conn = _connect(dsn, connection_factory=connection_factory, **kwasync)
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
psycopg2.OperationalError: connection to server at "localhost" (127.0.0.1), port 5432 failed: FATAL:  database "localhost" does not exist


ERROR:    Application startup failed. Exiting.
```
> Функция пытается подключится к базе данных с именем localhost вместо myapp_db
**Решение изменить функцию get_db_connection**
```
def get_db_connection():
    return psycopg2.connect(
        host="localhost",
        database="myapp_db",
        user="myapp_user",
        password="StrongPass12345!",
        cursor_factory=RealDictCursor
    )
```
- Проверяем: ` curl http://localhost:8000/ ` / ` curl http://localhost:8000/api/health `
- ` {"message":"MyApp API работает!","timestamp":"2026-09-14T09:18:17.459610"} `
- ` {"status":"healthy","database":"connected"} `

- Создаем systemd-юнит: ` sudo nano /etc/systemd/system/myapp-backend.service `
```
[Unit]
Description=MyApp Backend (FastAPI)
After=network.target postgresql.service
Wants=postgresql.service

[Service]
Type=simple
User=mitest
Group=mitest
WorkingDirectory=/opt/myapp/backend
Environment="PATH=/opt/myapp/backend/venv/bin"
ExecStart=/opt/myapp/backend/venv/bin/uvicorn main:app --host 127.0.0.1 --port 8000
Restart=always
RestartSec=5
StandardOutput=journal
StandardError=journal
SyslogIdentifier=myapp-backend

# Лимиты ресурсов
MemoryMax=512M
CPUQuota=70%

# Безопасность
NoNewPrivileges=true
ProtectSystem=strict
ReadWritePaths=/opt/myapp

[Install]
WantedBy=multi-user.target
```

- Включаем сервис: ` sudo systemctl daemon-reload ` / ` systemctl enable myapp-backend ` / ` systemctl start myapp-backend ` / ` systemctl status myapp-backend `
```
# Вывод статуса
● myapp-backend.service - MyApp Backend (FastAPI)
     Loaded: loaded (/etc/systemd/system/myapp-backend.service; enabled; preset: enabled)
     Active: active (running) since Mon 2026-09-14 09:24:05 UTC; 5s ago
   Main PID: 236820 (uvicorn)
      Tasks: 1 (limit: 4606)
     Memory: 32.2M (max: 512.0M available: 479.7M peak: 32.5M)
        CPU: 735ms
     CGroup: /system.slice/myapp-backend.service
             └─236820 /opt/myapp/backend/venv/bin/python3 /opt/myapp/backend/venv/bin/uvicorn main:app --host 127.0.0.1 --port 8000

systemd[1]: Started myapp-backend.service - MyApp Backend (FastAPI).
myapp-backend[236820]: INFO:     Started server process [236820]
myapp-backend[236820]: INFO:     Waiting for application startup.
myapp-backend[236820]: 2026-09-14 09:24:06,301 - main - INFO - Запуск приложения...
myapp-backend[236820]: 2026-09-14 09:24:06,336 - main - INFO - Таблица items проверена/создана
myapp-backend[236820]: INFO:     Application startup complete.
myapp-backend[236820]: INFO:     Uvicorn running on http://127.0.0.1:8000 (Press CTRL+C to quit)
```

- Создаем frontend: ` cd /opt/myapp/frontend ` / ` nano index.html `
```
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>MyApp</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { font-family: Arial, sans-serif; background: #f5f5f5; padding: 20px; }
        .container { max-width: 800px; margin: 0 auto; }
        h1 { color: #333; margin-bottom: 20px; }
        .form-group { margin-bottom: 15px; }
        input, textarea { width: 100%; padding: 10px; border: 1px solid #ddd; border-radius: 4px; }
        button { background: #4CAF50; color: white; padding: 10px 20px; border: none; border-radius: 4px; cursor: pointer; }
        button:hover { background: #45a049; }
        .item { background: white; padding: 15px; margin: 10px 0; border-radius: 4px; box-shadow: 0 2px 4px rgba(0,0,0,0.1); }
        .item-header { display: flex; justify-content: space-between; align-items: center; }
        .delete-btn { background: #f44336; padding: 5px 10px; font-size: 12px; }
        .status { padding: 10px; margin: 10px 0; border-radius: 4px; }
        .status.ok { background: #d4edda; color: #155724; }
        .status.error { background: #f8d7da; color: #721c24; }
    </style>
</head>
<body>
    <div class="container">
        <h1>MyApp</h1>
        
        <div id="status" class="status"></div>
        
        <h2>Добавить item</h2>
        <form id="itemForm">
            <div class="form-group">
                <input type="text" id="itemName" placeholder="Название" required>
            </div>
            <div class="form-group">
                <textarea id="itemDesc" placeholder="Описание"></textarea>
            </div>
            <button type="submit">Добавить</button>
        </form>
        
        <h2>Список items</h2>
        <div id="itemsList"></div>
    </div>

    <script>
        const API_URL = '/api';
        
        // Проверка здоровья
        async function checkHealth() {
            try {
                const response = await fetch(`${API_URL}/health`);
                const data = await response.json();
                const statusDiv = document.getElementById('status');
                if (data.status === 'healthy') {
                    statusDiv.className = 'status ok';
                    statusDiv.textContent = 'API работает, база данных подключена';
                } else {
                    statusDiv.className = 'status error';
                    statusDiv.textContent = 'API недоступен';
                }
            } catch (error) {
                const statusDiv = document.getElementById('status');
                statusDiv.className = 'status error';
                statusDiv.textContent = 'Не удалось подключиться к API';
            }
        }
        
        // Загрузить items
        async function loadItems() {
            try {
                const response = await fetch(`${API_URL}/items`);
                const data = await response.json();
                const itemsList = document.getElementById('itemsList');
                itemsList.innerHTML = '';
                
                if (data.items.length === 0) {
                    itemsList.innerHTML = '<p>Нет items. Добавьте первый!</p>';
                    return;
                }
                
                data.items.forEach(item => {
                    const itemDiv = document.createElement('div');
                    itemDiv.className = 'item';
                    itemDiv.innerHTML = `
                        <div class="item-header">
                            <h3>${item.name}</h3>
                            <button class="delete-btn" onclick="deleteItem(${item.id})">Удалить</button>
                        </div>
                        <p>${item.description || 'Нет описания'}</p>
                        <small>Создано: ${new Date(item.created_at).toLocaleString('ru-RU')}</small>
                    `;
                    itemsList.appendChild(itemDiv);
                });
            } catch (error) {
                console.error('Ошибка загрузки items:', error);
            }
        }
        
        // Добавить item
        document.getElementById('itemForm').addEventListener('submit', async (e) => {
            e.preventDefault();
            const name = document.getElementById('itemName').value;
            const description = document.getElementById('itemDesc').value;
            
            try {
                await fetch(`${API_URL}/items`, {
                    method: 'POST',
                    headers: { 'Content-Type': 'application/json' },
                    body: JSON.stringify({ name, description })
                });
                document.getElementById('itemName').value = '';
                document.getElementById('itemDesc').value = '';
                await loadItems();
            } catch (error) {
                console.error('Ошибка создания item:', error);
            }
        });
        
        // Удалить item
        async function deleteItem(id) {
            if (!confirm('Удалить этот item?')) return;
            
            try {
                await fetch(`${API_URL}/items/${id}`, { method: 'DELETE' });
                await loadItems();
            } catch (error) {
                console.error('Ошибка удаления item:', error);
            }
        }
        
        // Инициализация
        checkHealth();
        loadItems();
        setInterval(checkHealth, 30000); // Проверка каждые 30 секунд
    </script>
</body>
</html>
```

- Создаем nginx конфиг: ` sudo nano /etc/nginx/sites-available/myapp `
```
server {
    listen 80;
    server_name localhost;

    # Frontend (статические файлы)
    location / {
        root /opt/myapp/frontend;
        index index.html;
        try_files $uri $uri/ /index.html;
    }

    # Backend API (проксирование на FastAPI)
    location /api/ {
        proxy_pass http://127.0.0.1:8000/api/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Health check endpoint
    location /api/health {
        proxy_pass http://127.0.0.1:8000/api/health;
        proxy_set_header Host $host;
    }

    # Логи
    access_log /var/log/nginx/myapp_access.log;
    error_log /var/log/nginx/myapp_error.log;
}
```

- Создаем символическую ссылку: ` sudo ln -s /etc/nginx/sites-available/myapp /etc/nginx/sites-enabled/ `
- Удаляем дефолтный сайт: ` sudo rm /etc/nginx/sites-enabled/default `
- Проверяем конфиг nginx: ` sudo nginx -t `
- Перезапускаем nginx: ` sudo systemctl restart nginx ` / ` sudo systemctl enable nginx `

> При проверки в браузере, страница не открывалась, а через curl я ее видел
> Проверил iptables ` sudo iptables -L -n -v `
> Увидел что нет разрешаюшего правила на 80ый порт
> Добавил правило: ` sudo iptables -I INPUT -p tcp --dport 80 -j ACCEPT `
> Сохранил правило: ` sudo netfilter-persistent save ` 
> После этого страница стала открываться в браузере

- Проверяем сервисы: ` sudo systemctl status postgresql ` / ` systemctl status myapp-backend ` /  ` systemctl status nginx `
```
● postgresql.service - PostgreSQL RDBMS
     Loaded: loaded (/usr/lib/systemd/system/postgresql.service; enabled; preset: enabled)
     Active: active (exited) since Mon 2026-09-14 08:04:32 UTC; 2h 10min ago
   Main PID: 234061 (code=exited, status=0/SUCCESS)
        CPU: 7ms

systemd[1]: Starting postgresql.service - PostgreSQL RDBMS...
systemd[1]: Finished postgresql.service - PostgreSQL RDBMS.

# ---

● myapp-backend.service - MyApp Backend (FastAPI)
     Loaded: loaded (/etc/systemd/system/myapp-backend.service; enabled; preset: enabled)
     Active: active (running) since Mon 2026-09-14 09:24:05 UTC; 51min ago
   Main PID: 236820 (uvicorn)
      Tasks: 1 (limit: 4606)
     Memory: 32.4M (max: 512.0M available: 479.5M peak: 32.6M)
        CPU: 7.762s
     CGroup: /system.slice/myapp-backend.service
             └─236820 /opt/myapp/backend/venv/bin/python3 /opt/myapp/backend/venv/bin/uvicorn main:app --host 127.0.0.1 --port 8000

myapp-backend[236820]: INFO:     127.0.0.1:49376 - "GET /api/health HTTP/1.0" 200 OK
myapp-backend[236820]: INFO:     127.0.0.1:36048 - "GET /api/health HTTP/1.0" 200 OK
myapp-backend[236820]: INFO:     127.0.0.1:32772 - "GET /api/health HTTP/1.0" 200 OK
myapp-backend[236820]: INFO:     127.0.0.1:36922 - "GET /api/health HTTP/1.0" 200 OK
myapp-backend[236820]: INFO:     127.0.0.1:49504 - "GET /api/health HTTP/1.0" 200 OK
myapp-backend[236820]: INFO:     127.0.0.1:59718 - "GET /api/health HTTP/1.0" 200 OK
myapp-backend[236820]: INFO:     127.0.0.1:55724 - "GET /api/health HTTP/1.0" 200 OK
myapp-backend[236820]: INFO:     127.0.0.1:35788 - "GET /api/health HTTP/1.0" 200 OK
myapp-backend[236820]: INFO:     127.0.0.1:59046 - "GET /api/health HTTP/1.0" 200 OK
myapp-backend[236820]: INFO:     127.0.0.1:48808 - "GET /api/health HTTP/1.0" 200 OK

# ---

● nginx.service - A high performance web server and a reverse proxy server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; preset: enabled)
     Active: active (running) since Mon 2026-09-14 09:36:38 UTC; 39min ago
       Docs: man:nginx(8)
   Main PID: 236904 (nginx)
      Tasks: 2 (limit: 4606)
     Memory: 1.8M (peak: 2.1M)
        CPU: 59ms
     CGroup: /system.slice/nginx.service
             ├─236904 "nginx: master process /usr/sbin/nginx -g daemon on; master_process on;"
             └─236905 "nginx: worker process"

systemd[1]: Starting nginx.service - A high performance web server and a reverse proxy server...
systemd[1]: Started nginx.service - A high performance web server and a reverse proxy server.
```

- Проверяем API: ` curl http://localhost/api/health `
```
{"status":"healthy","database":"connected"}
```

- Проверяем frontend: ` curl http://localhost/ `
```
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>MyApp</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { font-family: Arial, sans-serif; background: #f5f5f5; padding: 20px; }
        .container { max-width: 800px; margin: 0 auto; }
        h1 { color: #333; margin-bottom: 20px; }
        .form-group { margin-bottom: 15px; }
        input, textarea { width: 100%; padding: 10px; border: 1px solid #ddd; border-radius: 4px; }
        button { background: #4CAF50; color: white; padding: 10px 20px; border: none; border-radius: 4px; cursor: pointer; }
        button:hover { background: #45a049; }
        .item { background: white; padding: 15px; margin: 10px 0; border-radius: 4px; box-shadow: 0 2px 4px rgba(0,0,0,0.1); }
        .item-header { display: flex; justify-content: space-between; align-items: center; }
        .delete-btn { background: #f44336; padding: 5px 10px; font-size: 12px; }
        .status { padding: 10px; margin: 10px 0; border-radius: 4px; }
        .status.ok { background: #d4edda; color: #155724; }
        .status.error { background: #f8d7da; color: #721c24; }
    </style>
</head>
<body>
    <div class="container">
        <h1>MyApp</h1>

        <div id="status" class="status"></div>

        <h2>Добавить item</h2>
        <form id="itemForm">
            <div class="form-group">
                <input type="text" id="itemName" placeholder="Название" required>
            </div>
            <div class="form-group">
                <textarea id="itemDesc" placeholder="Описание"></textarea>
            </div>
            <button type="submit">Добавить</button>
        </form>

        <h2>Список items</h2>
        <div id="itemsList"></div>
    </div>

    <script>
        const API_URL = '/api';

        // Проверка здоровья
        async function checkHealth() {
            try {
                const response = await fetch(`${API_URL}/health`);
                const data = await response.json();
                const statusDiv = document.getElementById('status');
                if (data.status === 'healthy') {
                    statusDiv.className = 'status ok';
                    statusDiv.textContent = 'API работает, база данных подключена';
                } else {
                    statusDiv.className = 'status error';
                    statusDiv.textContent = 'API недоступен';
                }
            } catch (error) {
                const statusDiv = document.getElementById('status');
                statusDiv.className = 'status error';
                statusDiv.textContent = 'Не удалось подключиться к API';
            }
        }

        // Загрузить items
        async function loadItems() {
            try {
                const response = await fetch(`${API_URL}/items`);
                const data = await response.json();
                const itemsList = document.getElementById('itemsList');
                itemsList.innerHTML = '';

                if (data.items.length === 0) {
                    itemsList.innerHTML = '<p>Нет items. Добавьте первый!</p>';
                    return;
                }

                data.items.forEach(item => {
                    const itemDiv = document.createElement('div');
                    itemDiv.className = 'item';
                    itemDiv.innerHTML = `
                        <div class="item-header">
                            <h3>${item.name}</h3>
                            <button class="delete-btn" onclick="deleteItem(${item.id})">Удалить</button>
                        </div>
                        <p>${item.description || 'Нет описания'}</p>
                        <small>Создано: ${new Date(item.created_at).toLocaleString('ru-RU')}</small>
                    `;
                    itemsList.appendChild(itemDiv);
                });
            } catch (error) {
                console.error('Ошибка загрузки items:', error);
            }
        }

        // Добавить item
        document.getElementById('itemForm').addEventListener('submit', async (e) => {
            e.preventDefault();
            const name = document.getElementById('itemName').value;
            const description = document.getElementById('itemDesc').value;

            try {
                await fetch(`${API_URL}/items`, {
                    method: 'POST',
                    headers: { 'Content-Type': 'application/json' },
                    body: JSON.stringify({ name, description })
                });
                document.getElementById('itemName').value = '';
                document.getElementById('itemDesc').value = '';
                await loadItems();
            } catch (error) {
                console.error('Ошибка создания item:', error);
            }
        });

        // Удалить item
        async function deleteItem(id) {
            if (!confirm('Удалить этот item?')) return;

            try {
                await fetch(`${API_URL}/items/${id}`, { method: 'DELETE' });
                await loadItems();
            } catch (error) {
                console.error('Ошибка удаления item:', error);
            }
        }

        // Инициализация
        checkHealth();
        loadItems();
        setInterval(checkHealth, 30000); // Проверка каждые 30 секунд
    </script>
</body>
</html>
```

- Проверяем логи backend: ` sudo journalctl -u myapp-backend -n 20 `
```
myapp-backend[236820]: INFO:     127.0.0.1:39616 - "GET /api/health HTTP/1.0" 200 OK
myapp-backend[236820]: INFO:     127.0.0.1:58636 - "GET /api/health HTTP/1.0" 200 OK
myapp-backend[236820]: INFO:     127.0.0.1:40706 - "GET /api/health HTTP/1.0" 200 OK
myapp-backend[236820]: INFO:     127.0.0.1:49376 - "GET /api/health HTTP/1.0" 200 OK
myapp-backend[236820]: INFO:     127.0.0.1:36048 - "GET /api/health HTTP/1.0" 200 OK
```

- Проверяем логи nginx: ` tail -f /var/log/nginx/myapp_access.log `
```
192.168.0.10 - - [14/Sep/2026:10:19:26 +0000] "GET /api/health HTTP/1.1" 200 43 "http://192.168.0.38/" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/114.0.0.0 YaBrowser/23.7.1.1266 (corp) Yowser/2.5 Safari/537.36"
127.0.0.1 - - [14/Sep/2026:10:20:01 +0000] "GET / HTTP/1.1" 200 5841 "-" "curl/8.5.0"
192.168.0.10 - - [14/Sep/2026:10:20:26 +0000] "GET /api/health HTTP/1.1" 200 43 "http://192.168.0.38/" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/114.0.0.0 YaBrowser/23.7.1.1266 (corp) Yowser/2.5 Safari/537.36"
192.168.0.10 - - [14/Sep/2026:10:21:26 +0000] "GET /api/health HTTP/1.1" 200 43 "http://192.168.0.38/" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/114.0.0.0 YaBrowser/23.7.1.1266 (corp) Yowser/2.5 Safari/537.36"
192.168.0.10 - - [14/Sep/2026:10:22:26 +0000] "GET /api/health HTTP/1.1" 200 43 "http://192.168.0.38/" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/114.0.0.0 YaBrowser/23.7.1.1266 (corp) Yowser/2.5 Safari/537.36"
192.168.0.10 - - [14/Sep/2026:10:23:26 +0000] "GET /api/health HTTP/1.1" 200 43 "http://192.168.0.38/" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/114.0.0.0 YaBrowser/23.7.1.1266 (corp) Yowser/2.5 Safari/537.36"
192.168.0.10 - - [14/Sep/2026:10:24:26 +0000] "GET /api/health HTTP/1.1" 200 43 "http://192.168.0.38/" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/114.0.0.0 YaBrowser/23.7.1.1266 (corp) Yowser/2.5 Safari/537.36"
192.168.0.10 - - [14/Sep/2026:10:25:26 +0000] "GET /api/health HTTP/1.1" 200 43 "http://192.168.0.38/" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/114.0.0.0 YaBrowser/23.7.1.1266 (corp) Yowser/2.5 Safari/537.36"
192.168.0.10 - - [14/Sep/2026:10:26:26 +0000] "GET /api/health HTTP/1.1" 200 43 "http://192.168.0.38/" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/114.0.0.0 YaBrowser/23.7.1.1266 (corp) Yowser/2.5 Safari/537.36"
192.168.0.10 - - [14/Sep/2026:10:27:26 +0000] "GET /api/health HTTP/1.1" 200 43 "http://192.168.0.38/" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/114.0.0.0 YaBrowser/23.7.1.1266 (corp) Yowser/2.5 Safari/537.36"
```

- Проверяем подключение к БД: ` psql -U myapp_user -d myapp_db -c "SELECT * FROM items;" `
> Ошибка: ` psql: error: connection to server on socket "/var/run/postgresql/.s.PGSQL.5432" failed: FATAL:  Peer authentication failed for user "myapp_user" `
> Ошибка возникла из за того, что мы написали ` psql -U myapp_user ... ` без имени хоста
> PSQL по умолчанию пытается подключится через локальный сокет, а не через сеть
> Локальный сокет PSQL по умолчанию использует метод аутентификации peer
> То есть ошибка возникла из за того, что я ввел команду не из под пользователя ` myapp_user `
**Решение:** ` psql -h localhost -U myapp_user -d myapp_db -c "SELECT * FROM items;" `
```
 id |    name     |     description      |         created_at
----+-------------+----------------------+----------------------------
  1 | Тестим item | Это тестовая запись! | 2026-09-14 09:51:01.919592
(1 row)
```

- Перезагружаем сервер и проверяем автозагрузку: ` sudo reboot ` / ` sudo systemctl status postgresql ` / ` myapp-backend ` / ` nginx ` / ` curl http://localhost/api/health ` 
```
● postgresql.service - PostgreSQL RDBMS
     Loaded: loaded (/usr/lib/systemd/system/postgresql.service; enabled; preset: enabled)
     Active: active (exited) since Mon 2026-09-14 10:42:19 UTC; 1min 35s ago
    Process: 1027 ExecStart=/bin/true (code=exited, status=0/SUCCESS)
   Main PID: 1027 (code=exited, status=0/SUCCESS)
        CPU: 8ms

systemd[1]: Starting postgresql.service - PostgreSQL RDBMS...
systemd[1]: Finished postgresql.service - PostgreSQL RDBMS.

# ---

● myapp-backend.service - MyApp Backend (FastAPI)
     Loaded: loaded (/etc/systemd/system/myapp-backend.service; enabled; preset: enabled)
     Active: active (running) since Mon 2026-09-14 10:42:19 UTC; 1min 35s ago
   Main PID: 1034 (uvicorn)
      Tasks: 1 (limit: 4601)
     Memory: 53.4M (max: 512.0M available: 458.5M peak: 53.7M)
        CPU: 1.119s
     CGroup: /system.slice/myapp-backend.service
             └─1034 /opt/myapp/backend/venv/bin/python3 /opt/myapp/backend/venv/bin/uvicorn main:app --host 127.0.0.1 --port 8000

systemd[1]: Started myapp-backend.service - MyApp Backend (FastAPI).
myapp-backend[1034]: INFO:     Started server process [1034]
myapp-backend[1034]: INFO:     Waiting for application startup.
myapp-backend[1034]: 2026-09-14 10:42:20,526 - main - INFO - Запуск приложения...
myapp-backend[1034]: 2026-09-14 10:42:20,617 - main - INFO - Таблица items проверена/создана
myapp-backend[1034]: INFO:     Application startup complete.
myapp-backend[1034]: INFO:     Uvicorn running on http://127.0.0.1:8000 (Press CTRL+C to quit)
myapp-backend[1034]: INFO:     127.0.0.1:40516 - "GET /api/health HTTP/1.0" 200 OK
myapp-backend[1034]: INFO:     127.0.0.1:49902 - "GET /api/health HTTP/1.0" 200 OK

# ---

● nginx.service - A high performance web server and a reverse proxy server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; preset: enabled)
     Active: active (running) since Mon 2026-09-14 10:42:15 UTC; 1min 39s ago
       Docs: man:nginx(8)
    Process: 842 ExecStartPre=/usr/sbin/nginx -t -q -g daemon on; master_process on; (code=exited, status=0/SUCCESS)
    Process: 851 ExecStart=/usr/sbin/nginx -g daemon on; master_process on; (code=exited, status=0/SUCCESS)
   Main PID: 855 (nginx)
      Tasks: 2 (limit: 4601)
     Memory: 3.2M (peak: 3.5M)
        CPU: 43ms
     CGroup: /system.slice/nginx.service
             ├─855 "nginx: master process /usr/sbin/nginx -g daemon on; master_process on;"
             └─856 "nginx: worker process"

systemd[1]: Starting nginx.service - A high performance web server and a reverse proxy server...
systemd[1]: Started nginx.service - A high performance web server and a reverse proxy server.

# ---
{"status":"healthy","database":"connected"}
```

---
### Разбор команд:

- Установка и обновление пакетов:
	- Выполнение команд с правами root: ` sudo `
	- Обновление списков доступных пакетов из репозитория: ` apt update `
	- Логическое "И": ` && `
	- Скачивание и установка новых версий пакетов: ` apt upgrade ` 
	- Автоматические ответ "Да": ` -y `

- PostgreSQL:
	- Ключ sudo, который говорит использовать команду от пользователя postgres: ` -u postgres `
	- Интерактивный терминал PostgreSQL: ` psql `
	- Создает сущности в БД: ` CREATE DATABASE ... ` / ` CREATE USER `
	- Выдача полных прав на конкретную БД конкретного пользователя: ` GRANT ALL PRIVILEGES ON DATABASE ... TO ... `	
	- Внутренная команда psql для подключения к указанной БД: ` \c myapp_db `
	- Команда которая позваляет новым пользователям создавать таблицы в схеме public: ` GRANT ALL ON SCHEMA public TO ... `
	- Выход из терминала psql: ` \q `
	- Заставляет psql подключаться через TCP/IP, а не через сокет: ` -h localhost `
	- Имя пользователя БД: ` -U `
	- Имя БД: ` -d `
	- Выполнить команду и выйти, не заходя в интерактивный режим: ` -c `
	
- Файловая система:
	- Создает родительские директории, если их нет и не выдает ошибку если они есть: ` -p `
	- Brace expansion в Bash: ` {backend,frontend,logs} `
	- Переменная окружения Bash, которая подставляет имя пользователя: ` $USER `
	- Меняем владельца и группу на указанного пользователя: ` chown `
	- Запускаем встроенный модуль Python venv как скрипт: ` -m venv `
	- Имя папки которую нужно создать для виртуального окружения: ` venv `
	- Выполняет скрипт в текущей оболочке а не в дочерней: ` source `
	- Команды python и pip указывают на файлы внутри папки venv: ` $PATH `
	
- Python и FastAPI:
	- ` load_dotenv() `: Читает файл .env и загружает переменные в окружение процесса чтобы ` os_getenv() ` мог их прочитать
	- ` RealDictCursor `: Специальный курсор psycopg2 который возвращает строки из БД как словари Python( ` {'id': 1, 'name': 'test'} `), а не как кортежи ( `(1, 'test')` ) - это делает код чище: ` item['name'] ` всместо ` item[1] `
	- ` @app.on_event("startup") `:Хук, который выполняется один раз при запуске приложения, до того как оно начнет принимать HTTP-запросы

- Systemd:
	- Гарантия того, что systemd начнет запускать наш бэкенд только после того, как запуститься сеть и PSQL: ` After=network.target postgresql.service `
	- Systemd запускает процессы с минимальным окружением и не знает о виртуальном окружении: ` Environment="PATH=/opt/myapp/backend/venv/bin" `
	- Мы делаем всю файловую систему доступной только для чтения, но делаем исключение для папки приложения: ` ProtectSystem=strict ` + ` ReadWritePaths=/opt/myapp `
	
- Nginx:
	- Создание символической ссылки: ` ln -s `
	- Nginx снчала ищет точный файл (` $uri `) потом директорию (` $uri/ `) Если ничего не находит, он отдает /index.html
	- Когда Nginx проксирует запрос, backend видит что запрос пришел от 127.0.0.1. Этот заголовок добавляет реальный IP-клиента, чтобы backend мог его логировать
	
- Сеть и фаервол:
	- Самый важный ключ. Вставляет правило в самое начало цепочки INPUT: ` -I INPUT `
	- Протокол TCP: ` -p tcp `
	- Порт назначения: ` --dport 80 `
	- Принимает пакет: ` -J ACCEPT `
	- Правила ` iptables ` хранятся только в оперативной памяти ядра
	
- Мониторинг и логи:
	- Фильтр лога только для указанного systemd-юнита: ` -u `
	- Показывает только последнии 20 строк: ` -n 20 `
	- Режим слежения: ` -f `
