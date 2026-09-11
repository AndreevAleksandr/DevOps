### Задание 2

**Условаия**
1. Создать простое Python HTTP-приложение
2. Написать systemd-юнит для этого приложения
3. Настроить автозапуск при загрузки системы
4. Настроить автоматический перезапуск при падении
5. Установить лимиты на ресурсы (RAM, CPU)
6. Проверить работу сервиса

>Использую ubuntu24.04
---
**Практика**

- Создаем директорию для приложения: ` mkdir -p /opt/PythonHTTP_app `
- Меняем права собственности на файл: ` sudo chown $USER:$USER /opt/PythonHTTP_app `
- Переходим в директорию: ` cd /opt/PythonHTTP_app `
- Создаем файл server.py: ` nano server.py `

```
#Код который содержится в файле
#!/usr/bin/env python3
from http.server import HTTPServer, SimpleHTTPRequestHandler
import os
import datetime

class MyHandler(SimpleHTTPRequestHandler):
    def do_GET(self):
        self.send_response(200)
        self.send_header('Content-type', 'text/html; charset=utf-8')
        self.end_headers()
        message = f"""
        <html>
            <body>
                <h1>Тестовый сервис работает!</h1>
                <p>PID процесса: {os.getpid()}</p>
                <p>Время: {datetime.datetime.now()}</p>
                <p>Хост: {os.uname().nodename}</p>
            </body>
        </html>
        """

        self.wfile.write(message.encode())

    def log_message(self, format, *args):
        print(f"[{datetime.datetime.now()}] {args[0]}")

if __name__ == '__main__':
        PORT = 8081
        server = HTTPServer(('0.0.0.0', PORT), MyHandler)
        print(f" Сервер запущен на порту {PORT}")
        server.serve_forever()
```

- Делаем файл исполняймым: ` chmod +x /opt/PythonHTTP_app/server.py `
- Запускаем для проверки работоспособности: ` python3 /opt/PythonHTTP_app/server.py `
- Проверка: ` curl http://localhost:8081

**Вывод команды:**
```
<html>
    <body>
        <h1>Тестовый сервис работает!</h1>
        <p>PID процесса: 204744</p>
        <p>Время: 2026-09-11 12:19:05.817899</p>
        <p>Хост: test-vm</p>
	</body>
</html>
```
- Создаем файл юнита: ` nano /etc/systemd/system/PythonHTTP_app.service `

```
#Содержимое файла юнита
[Unit]
Description=Тестовый Python HTTP сервер
Documentation=http://github.com/AndreevAleksandr/DevOps/tree/main/Linux/Practice
After=network.target
Wants=network.target

[Service]
Type=simple
User=mitest
Group=mitest
WorkingDirectory=/opt/PythonHTTP_app
ExecStart=/usr/bin/python3 /opt/PythonHTTP_app/server.py
Restart=always
RestartSec=5
StandardOutput=journal
StandardError=journal
SyslogIdentifier=PythonHTTP_app

#Лимиты ресурсов
MemoryMax=256M
CPUQuota=50%

#Безопасность
NoNewPrivileges=true
ProtectSystem=strict
ReadWritePaths=/opt/PythonHTTP_app

[Install]
WantedBy=multi-user.target
```

- Перезагружаем systemd: ` systemctl daemon-reload `
- Включаем автозагрузку: ` systemctl enable PythonHTTP_app `
- Запуска сервис: ` systemctl start PythonHTTP_app `
- Проверяем статус: ` systemctl status PythonHTTP_app `

```
#Вывод статуса
PythonHTTP_app.service - Тестовый Python HTTP сервер
     Loaded: loaded (/etc/systemd/system/PythonHTTP_app.service; enabled; preset: enabled)
     Active: active (running) since Fri 2026-09-11 13:08:36 UTC; 27s ago
       Docs: http://github.com/AndreevAleksandr/DevOps/tree/main/Linux/Practice
   Main PID: 204901 (python3)
      Tasks: 1 (limit: 4606)
     Memory: 8.9M (max: 256.0M available: 247.0M peak: 9.0M)
        CPU: 96ms
     CGroup: /system.slice/PythonHTTP_app.service
             └─204901 /usr/bin/python3 /opt/PythonHTTP_app/server.py
```
- Проверяем работу: ` curl http://localhost:8081 `

```
<html>
    <body>
        <h1>Тестовый сервис работает!</h1>
        <p>PID процесса: 204901</p>
        <p>Время: 2026-09-11 13:10:15.647413</p>
        <p>Хост: test-vm</p>
    </body>
</html>
```

- Проверям логи: ` sudo journal -u PythonHTTP_app `
```
#Вывод
systemd[1]: Started PythonHTTP_app.service - Тестовый Python HTTP сервер.
```

- Находим PID процесса: ` ps aux | grep server.py `
- Убиваем процесс: ` kill -9 204901 `
- Ждем перезапуск процесса: ` sleep 5 `
- Проверяем статус: ` systemctl status PythonHTTP_app `
```
#Вывод
PythonHTTP_app.service - Тестовый Python HTTP сервер
     Loaded: loaded (/etc/systemd/system/PythonHTTP_app.service; enabled; preset: enabled)
     Active: active (running) since Fri 2026-09-11 13:31:45 UTC; 18s ago
       Docs: http://github.com/AndreevAleksandr/DevOps/tree/main/Linux/Practice
   Main PID: 204971 (python3)
      Tasks: 1 (limit: 4606)
     Memory: 8.9M (max: 256.0M available: 247.0M peak: 9.1M)
        CPU: 102ms
     CGroup: /system.slice/PythonHTTP_app.service
             └─204971 /usr/bin/python3 /opt/PythonHTTP_app/server.py

#Процесс поднялся с новым PID
```

- Просматриваем логи: ` journal -u PythonHTTP_app `
```
#Вывод
systemd[1]: Started PythonHTTP_app.service - Тестовый Python HTTP сервер.
systemd[1]: PythonHTTP_app.service: Main process exited, code=killed, status=9/KILL
systemd[1]: PythonHTTP_app.service: Failed with result 'signal'.
systemd[1]: PythonHTTP_app.service: Scheduled restart job, restart counter is at 1.
systemd[1]: Started PythonHTTP_app.service - Тестовый Python HTTP сервер.
```

- Просматриваем назначенные лимиты: ` systemctl show PythonHTTP_app | grep -E "MemoryMax|CPUQuota" `

### Разбор команд

- Создаем директорию: ` mkdir `
- Флаг, создает промежуточные директории если их нет: ` -p `
- Меняем владельца: ` chown `
- Переменная окружения с именем текущего пользователя: ` $USER `
- Владелец:группа: ` $USER:$USER `

- Интерпретатор Python3: ` python3 `

- Client URL (Утилита для HTTP запросов): ` curl `

- Утилита управления systemd: ` systemctl `
- Перезапуск конфигурации systemd: ` daemon-reload `
- Включаем автозагрузку сервиса: ` enable `
- Отключаем автозагрузку сервиса: ` disable `
- Активен ли сервис сейчас: ` is-active `
- Перезагружаем конфигурация сервиса без остановки: ` reload `
- Смотрим все свойства юнита: ` systemctl show `
- Утилита для анализа systemd: ` systemd-analyze `
- Проверяем синтаксис юнит-файла: ` verify `
