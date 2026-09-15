### Задача 4

**Условаия**
1. Скрипты должны быть устойчевы к ошибкам
2. Использовать пайпы и переменные
3. Работать с переменными окружения
4. Генерировать читаемые отчеты
5. Использовать find/grep/awk/sed/xargs

>Использую OS Ubuntu 24.04
---
**Практика**

- Создаем структуру проекта:
	- Создаем корнивую директорию: ` mkdir -p ~/log-analyz/{script,logs,reports,config} `
	- Переходим в директорию: ` cd ~/log-analyz/
	- Создаем тестовые логи: ` cat > logs/app.log << 'EOF' ... EOF`
	```
	2026-09-14 10:15:23 INFO [main] Application started
	2026-09-14 10:15:24 ERROR [db] Connection failed: timeout
	2026-09-14 10:15:25 WARN [api] Slow response: 2500ms
	2026-09-14 10:15:26 INFO [auth] User login: admin
	2026-09-14 10:15:27 ERROR [api] 500 Internal Server Error
	2026-09-14 10:15:28 INFO [db] Query executed: SELECT * FROM users
	2026-09-14 10:15:29 ERROR [db] Connection failed: timeout
	2026-09-14 10:15:30 DEBUG [cache] Cache miss for key: user_123
	2026-09-14 10:15:31 INFO [api] Request processed: 200 OK
	2026-09-14 10:15:32 ERROR [auth] Invalid token: expired
	2026-09-14 10:16:01 INFO [main] Health check passed
	2026-09-14 10:16:02 WARN [memory] High memory usage: 85%
	2026-09-14 10:16:03 ERROR [api] 503 Service Unavailable
	2026-09-14 10:16:04 INFO [db] Connection pool: 10/100
	2026-09-14 10:16:05 ERROR [db] Connection failed: timeout
	```
	
	- Второй лог: ` cat > logs/access.log << 'EOF' ... EOF `
	```
	192.168.1.10 - - [14/Sep/2026:10:15:23] "GET /api/users HTTP/1.1" 200 1234
	192.168.1.15 - - [14/Sep/2026:10:15:24] "POST /api/login HTTP/1.1" 401 89
	192.168.1.10 - - [14/Sep/2026:10:15:25] "GET /api/products HTTP/1.1" 200 5678
	192.168.1.20 - - [14/Sep/2026:10:15:26] "GET /api/users HTTP/1.1" 200 1234
	192.168.1.15 - - [14/Sep/2026:10:15:27] "DELETE /api/users/5 HTTP/1.1" 403 45
	192.168.1.10 - - [14/Sep/2026:10:15:28] "GET /api/orders HTTP/1.1" 500 120
	192.168.1.25 - - [14/Sep/2026:10:15:29] "POST /api/orders HTTP/1.1" 201 890
	192.168.1.10 - - [14/Sep/2026:10:15:30] "GET /api/users HTTP/1.1" 200 1234
	192.168.1.30 - - [14/Sep/2026:10:16:01] "GET /health HTTP/1.1" 200 15
	192.168.1.15 - - [14/Sep/2026:10:16:02] "POST /api/login HTTP/1.1" 200 456
	```

- find / stat / file
	- Проверяем созданные файлы: ` ls -la logs/ `
	
	- Ищем все лог файлы рекурсивно: ` find ~/log-analyz/ -name "*.log" -type f `
	- Ищем файлы больше 100байт: ` find ~/log-analyz/ -size +100c -type f `
	- Находим измененные за последние сутки: ` find ~/log-analyz/ -mtime -1 -type f `
	- Смотрим информацию о файле: ` stat logs/app.log `
	- Определяем тип файла: ` file logs/app.log ` / ` file /bin/ls ` / ` file /dev/null `
	```
	1. logs/app.log: ASCII text
	2. /bin/ls: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=44900829b39271d878615772bb2a6ffa422f0891, for GNU/Linux 3.2.0, stripped
	3. /dev/null: character special (1/3)
	```
	
	- Создаем символическую ссылку: ` ln -s ~/log-analyz/logs/app.log ~/log-analyz/latest.log ` / ` ls -la latest.log `
	```
	#Вывод проверки
	lrwxrwxrwx 1 mitest mitest 35 сен 15 09:06 latest.log -> /home/yukon/log-analyz/logs/app.log
	```
	- Жесткая ссылка: ` ln logs/app.log logs/app_hardlink.log ` / ` ls -li logs/*.log `
	```
	#Вывод проверки
	38200 -rw-rw-r-- 1 mitest mitest 751 сен 15 07:21 logs/access.log
	2277 -rw-rw-r-- 2 mitest mitest 848 сен 15 07:21 logs/app_hardlink.log
	2277 -rw-rw-r-- 2 mitest mitest 848 сен 15 07:21 logs/app.log
	```
	
- Потоки и перенаправления
	- Перенаправление stdout в файл: ` echo "Hello world" > output.txt ` / ` cat output.txt `
	- Дописываем в файл: ` echo "Second line" >> output.txt ` / ` cat output.txt `
	- Перенаправление stderr в файл: ` ls /nonexistent_directory 2> error.log ` / ` cat error.log `
	- Перенаправление srdout и stderr в один файл: ` ls /nonexistent 1> all.log 2>&1 ` / ls /nonexistent &> all.log ` / ` cat all.log `
	- Перенаправление stderr в stdout: ` ls /nonexistent 2>&1 | grep "No such" `
	- Черная дыра: ` /dev/null `/ ` ls /nonexistent 2>/dev/null ` / ` echo "Test" > /dev/null `
	- Ввод из строки: ` cat << EOF > config.txt server_name: myapp port:8080 debug: true EOF ` / ` cat config.txt ` 
	
- Пайпы
	- Простой пайп: ` cat logs/app.log | grep ERROR `
	- Цепочка пайпов: ` cat logs/app.log | grep ERROR | wc -l `
	- Считаем кол-во уникальных IP: ` cat logs/access.log | awk '{print $1}' | sort | uniq | wc -l `
	- Передача аргументов: ` find logs -name "*.log" | xargs wc -l `
	- Передача агрументов, если в имени есть пробелы: ` find logs -name "*.log" -print0 | xargs -0 wc -l `
	
- Переменные окружения
	- Смотрим все параметры окружения: ` env ` / ` printenv `
	- Смотрим конкретную переменную: ` echo $HOME ` / ` echo $PATH ` / ` echo $USER ` 
	- Создаем локальную переменную: ` MY_VAR="hello" ` / ` echo $MY_VAR `
	- Создаем глобальную переменную: ` export MY_VAR="hello" ` / ` export LOG_LEVEL=debug ` / ` export APP_PORT=8080 `
	- Смотрим что переменная экспортирована: ` printenv | grep MY_VAR `
	- Временная переменная: ` LOG_LEVEL=info python3 --version `
	- Удаляем переменную: ` unset MY_VAR `
	- Смотрим содержиме файлов: ` cat ~/.bashrc | head -20 ` / ` cat ~/.profile | head -20 `
	- ` .bashrc ` - выполняется при каждом запуске интерактивного bash
	- Добавляем своб переменную в .bashrc: ` echo 'export CUSTOM_VAR="product"' >> ~/.bashrc ` / ` echo 'alias ll="ls -la"' >> ~/.bashrc `
	- Применяем изменения без перезагрузки: ` source ~/.bashrc `
	- Проверяем: ` echo $CUSTOM_VAR `
	- Смотрим PATH: ` echo $PATH `
	- Список директорий разделенные запятыми, где система ищет испольняемые файлы: ` PATH `
	- Добавляем свою директорию в PATH: ` echo 'export PATH="$HOME/bin:$PATH"' >> ~/.bashrc ` / ` source ~/.bashrc `
	- Создаем скрипт: ` mkdir -p ~/bin ` / ` chmod +x ~/bin/hello
	```
	cat > ~/bin/hello-test << 'EOF'
	#!/bin/bash
	echo "Hello from custom script"
	EOF
	```
	Запускаем: ` ./hello-test `
	
- Текст (grep, awk, sed, cut, sort, uniq, wc)
