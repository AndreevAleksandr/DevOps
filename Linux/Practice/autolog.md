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
	- Поиск по шаблону: ` grep `
	- Ищим все ERROR в логе: ` grep ERROR logs/app.log `
	- Игнорируем регистр: ` grep -i error logs/app.log `
	- Смотрим номер строки: ` grep -n ERROR logs/app.log `
	- Смотрим две строки до и после совпадения: ` grep -C 2 ERROR logs/app.log `
	- Обратный список: ` grep -v ERROR logs/app.log `
	- Подсчитать кол-во совпадений: ` grep -c ERROR logs/app.log `
	- Регулярыне выражения: ` grep -E "ERROR|WARN" logs/app.log ` / ` grep -E "2026-09-14 10:1[56]" logs/app.log `
	
	- Обработка колонок: ` awk `
	- Выводим первую колонку: ` awk '{print $1}' logs/access.log `
	- Выводим IP и метод запроса: ` awk '{print $1 $6}' logs/access.log | tr -d '"' `
	- Считаем сумму байт: ` awk '{sum += $9} END {print sum}' logs/access.log `
	- Фильтруем только 200ые ответы: ` awk '$8 == 200 {print $0}' logs/access.log `
	- Считаем кол-во запросов по IP: ` awk '{print $1}' logs/access.log | sort | uniq -c | sort -rn `
	- Средняя длинна ответа: ` awk '{sum += $10; count++} END {print "Average:", sum/count}' logs/access.log `
	
	- Замена текста: ` sed `
	- Меняем первое слово в строке: ` echo "hello world" | sed 's/hello/goodbye/' `
	- Меняем все вхождения: ` echo "hello hello world" | sed 's/hello/goodbye/g' `
	- Меняем файл: ` sed -i 's/debug/info/g' config.txt `
	- Удаляем строки с ERROR: ` sed '/ERROR/d' logs/app.log `
	- Показать строки с ERROR: ` sed -n '/ERROR/p' logs/app.log `
	- Меняем дату: ` sed 's/2026-09-14/2026-09-15/g' logs/app.log `
	- Удаляем пустые строки: ` sed '/^$/d' logs/app.log `
	
	- Выделение колонок: ` cut `
	- Вырезаем первую колонку: ` cut -d' ' -f1 logs/access.log `
	- Вырезаем IP и статус: ` cut -d' ' -f1,9 logs/access.log `
	- Вырезаем диапозон символов: ` echo "Hello World" | cut -c1-5 `
	- Разделитель таблицы: ` echo -e "name\tage\tcity" | cut -f2 `
	
	- ` sort, uniq, wc `
	- Сортировка: ` sort logs/app.log `
	- Обратная сортировка: ` sort -r logs/app.log `
	- Числовая сортировка: ` echo -e "5\n2\n10\n1" | sort -n `
	- Удаляем дубликаты: ` echo -e "a\na\nb\nb\na" | sort | uniq `
	- Считаем кол-во повторений: ` awk '{print $1}' logs/access.log | sort | uniq -c | sort -rn `
	- ` wc ` - word count
	- Только строки: ` wc -l logs/app.log `
	- Только слова: ` wc -w logs/app.log `
	- Только байты: ` wc -c logs/app.log `
	
- Bash-скрипт
	- Создаем файл: ` nano script/analyz_logs.sh `
	```
	#!/bin/bash

	# ============================================
	# Скрипт анализа логов
	# ============================================

	# Переменные
	LOG_DIR="/home/$USER/log-analyzer/logs"
	REPORT_DIR="/home/$USER/log-analyzer/reports"
	DATE=$(date +%Y-%m-%d_%H-%M-%S)

	# Цвета для вывода
	RED='\033[0;31m'
	GREEN='\033[0;32m'
	YELLOW='\033[1;33m'
	NC='\033[0m' # No Color

	# Функция для вывода сообщений
	log_message() {
		echo -e "${GREEN}[$(date '+%Y-%m-%d %H:%M:%S')]${NC} $1"
	}

	# Функция для вывода ошибок
	log_error() {
		echo -e "${RED}[ERROR]${NC} $1" >&2
	}

	# Проверка существования директории
	if [ ! -d "$LOG_DIR" ]; then
		log_error "Директория с логами не найдена: $LOG_DIR"
		exit 1
	fi

	# Создать директорию для отчетов
	mkdir -p "$REPORT_DIR"

	log_message "Начало анализа логов..."

	# ============================================
	# Анализ app.log
	# ============================================
	APP_LOG="$LOG_DIR/app.log"

	if [ -f "$APP_LOG" ]; then
		log_message "Анализ $APP_LOG"
    
		# Подсчитать количество ошибок
		ERROR_COUNT=$(grep -c "ERROR" "$APP_LOG")
		WARN_COUNT=$(grep -c "WARN" "$APP_LOG")
		INFO_COUNT=$(grep -c "INFO" "$APP_LOG")
    
		echo "================================"
		echo "Статистика по app.log:"
		echo "  ERROR: $ERROR_COUNT"
		echo "  WARN:  $WARN_COUNT"
		echo "  INFO:  $INFO_COUNT"
		echo "================================"
    
		# Найти уникальные ошибки
		echo -e "\nУникальные ошибки:"
		grep "ERROR" "$APP_LOG" | awk -F'[][]' '{print $2}' | sort | uniq -c | sort -rn
    
	else
		log_error "Файл app.log не найден"
	fi

	# ============================================
	# Анализ access.log
	# ============================================
	ACCESS_LOG="$LOG_DIR/access.log"

	if [ -f "$ACCESS_LOG" ]; then
		log_message "Анализ $ACCESS_LOG"
    
		echo -e "\nТоп-5 IP адресов:"
		awk '{print $1}' "$ACCESS_LOG" | sort | uniq -c | sort -rn | head -5
    
		echo -e "\nРаспределение HTTP статусов:"
		awk '{print $9}' "$ACCESS_LOG" | sort | uniq -c | sort -rn
    
		echo -e "\nКоличество запросов по методам:"
		awk '{print $6}' "$ACCESS_LOG" | tr -d '"' | sort | uniq -c | sort -rn
    
	else
		log_error "Файл access.log не найден"
	fi

	# ============================================
	# Генерация отчета
	# ============================================
	REPORT_FILE="$REPORT_DIR/report_$DATE.txt"

	log_message "Генерация отчета: $REPORT_FILE"

	cat > "$REPORT_FILE" << EOF
	========================================
	ОТЧЕТ ПО АНАЛИЗУ ЛОГОВ
	Дата: $(date)
	========================================

	1. СТАТИСТИКА ПО app.log:
		ERROR: $ERROR_COUNT
		WARN:  $WARN_COUNT
		INFO:  $INFO_COUNT

	2. ТОП-5 IP АДРЕСОВ:
	$(awk '{print $1}' "$ACCESS_LOG" | sort | uniq -c | sort -rn | head -5)

	3. HTTP СТАТУСЫ:
	$(awk '{print $9}' "$ACCESS_LOG" | sort | uniq -c | sort -rn)

	4. МЕТОДЫ ЗАПРОСОВ:
	$(awk '{print $6}' "$ACCESS_LOG" | tr -d '"' | sort | uniq -c | sort -rn)

	========================================
	Конец отчета
	========================================
	EOF

	log_message "Отчет сохранен: $REPORT_FILE"
	log_message "Анализ завершен!"

	exit 0
	```
	- Делаем скрипт испольняемые: ` chmod +x script/analyz_logs.sh `
	- Запускаем: ` ./script/analyz_logs.sh
	- Смотрим что записалось в файл репорт: ` cat reports/reports_*.txt `
	
	- Условыне операторы: ` if / else / case `
	- Создаем скрипт: ` nano script/test_case.sh `
	```
	#!/bin/bash

	STATUS=200

	case $STATUS in
		200)
			echo "OK"
			;;
		404)
			echo "Not Found"
			;;
		500)
			echo "Server Error"
			;;
		*)
			echo "Unknown status"
			;;
	esac
	```
	- Запускаем: ` ./script/test_case.sh `
	
	- Цикл for
	- Создаем скрипт: ` nano script/test_loops.sh `
	```
	#!/bin/bash

	# Простой for
	for i in 1 2 3 4 5; do
		echo "Iteration $i"
	done

	# for с диапазоном
	for i in {1..5}; do
		echo "Number $i"
	done

	# for по файлам
	for file in logs/*.log; do
		echo "Processing: $file"
		wc -l "$file"
	done

	# while
	COUNTER=0
	while [ $COUNTER -lt 5 ]; do
		echo "Counter: $COUNTER"
		((COUNTER++))
	done

	# Чтение файла построчно
	while IFS= read -r line; do
		echo "Line: $line"
	done < logs/app.log | head -5
	```
	Запускаем: ` ./script/test_loops.sh `
	
	- Функции и exit
	- Создаем файл: ` nano script/test_function.sh `
	```
	#!/bin/bash

	# Простая функция
	greet() {
		echo "Hello, $1!"
	}

	greet "World"
	greet "DevOps"

	# Функция с возвратом значения
	check_file() {
		if [ -f "$1" ]; then
			return 0  # Успех
		else
			return 1  # Ошибка
		fi
	}

	# Использование
	if check_file "logs/app.log"; then
		echo "Файл найден"
	else
		echo "Файл не найден"
	fi

	# Exit-коды
	exit 0  # Успех
	exit 1  # Общая ошибка
	exit 2  # Неправильное использование
	exit 127 # Команда не найдена
	```
	
- SSH и передача данных
	- Редактируем: ` nano ~/.ssh/config `
	```
	# Локальный сервер (VM)
	Host myserver
		HostName 192.168.0.38
		User mitest
		Port 22
		IdentityFile ~/.ssh/id_ed25519
		ForwardAgent yes

	# Удаленный сервер (пример)
	Host production
		HostName example.com
		User deploy
		Port 22
		IdentityFile ~/.ssh/id_ed25519
		Compression yes
		ServerAliveInterval 60
	```
	> Теперь можно пдключаться по имени хоста: ` ssh myserver `
	
	- Копируем файл на сервер: ` scp logs/app.log myserver:/home/mitest/ `
	- Копируем директорию рекурсивно: ` scp -r scripts/ myserver:/home/mitest/scripts/ `
	- Копируем с сервера: ` scp myserver:/home/mitest/reports/report_*.txt ./reports/ `
	- С указанием порта: ` scp -P 2222 file.txt user@host:/path/ `
	
	- Синхронизация ` rsync `
	- Синхронизируем директорию: ` rsync -avz scripts/ myserver:/home/mitest/scripts/ `
	- Сохраняет права, даты, ссылки: ` -a `
	- Подробный вывод: ` -v `
	- Сжатие при передаче: ` -z `
	- Удаляет файлы на сервере, которых нет локально: ` rsync -avz --delete scripts/ myserver:/home/mitest/scripts/ `
	
	- Проброс портов

	- Локальный проброс: ` ssh -L 8000:localhost:8000 myserver `
	- Удаленный проброс: ` ssh -R 8080:localhost:80 myserver `
	- Динамический проброс: ` ssh -D 1080 myserver `
	
### Разбор команд

- ` find ~/log-analyz/ -name "*.log" -type f `
	- Рекурсивный поиск. ` -name ` фильтрует по имени (маска *), ` -type f ` гарантирует, что найдены только файлы
- ` find ~/log-analyz/ -size +100c -type f `
	- Поиск файлов больше 100 байт (` c = bytes `)
- ` find ~/log-analyz/ -mtime -1 -type f `
	- Поиск файлов, модифицированных за последние 24 часа (-1)
- ` stat logs/app.log `
	- Выводит детальную метаинформацию: размер, блоки, права доступа, время последнего доступа/изменения (` mtime, atime, ctime `) и ` inode `
- ` file logs/app.log `
	- Определяет тип файла по его содержимому, а не по расширению (например, ` ASCII text, ELF executable `)
- ` ln -s source target `
	- ` Символическая ссылка ` (ярлык). Указывает на путь. Если исходный файл удалить, ссылка станет "битой"
- ` ln source target `
	- ` Жесткая ссылка `. Создает еще одно имя для того же ` inode ` на диске. Файл будет удален физически только когда удалятся все жесткие ссылки на него
	
- Перенаправляет stdout (1) в файл, перезаписывая его содержимое: ` > output.txt `
- Перенаправляет stdout в файл, дописывая в конец (append): ` >> output.txt `
- Перенаправляет только поток ошибок (stderr) в файл: ` 2> error.log `
- Перенаправляет и stdout, и stderr в один файл. 2>&1 означает "направь поток 2 туда же, куда направлен поток 1": ` &> all.log ` или ` > all.log 2>&1 `
- "Черная дыра". Поток ошибок игнорируется и уничтожается: ` 2>/dev/null `
- Запись многострочного текста из скрипта прямо в файл: ` cat << EOF > config.txt `

- Соединяет stdout первой команды с stdin второй. Данные передаются потоком без создания временных файлов: ` cmd1 | cmd2 `
- Классический пайп: берем лог, фильтруем строки с ERROR, считаем количество строк: ` cat logs/app.log | grep ERROR | wc -l `
- xargs берет вывод find и подставляет его как аргументы в wc -l: `  find logs -name "*.log" | xargs wc -l `
- ` -print0 ` разделяет имена файлов нулевым байтом (\0), а -0 заставляет xargs читать их так же: ` find ... -print0 | xargs -0 wc -l `

- Делает переменную доступной для всех дочерних процессов (скриптов): ` export MY_VAR="value" `
- Временная переменная. Действует только для одной выполняемой команды: ` LOG_LEVEL=info python3 ... `
- Перечитывает файл конфигурации в текущей сессии, применяя изменения без перезапуска терминала: ` source ~/.bashrc `
- Добавляет пользовательскую директорию в начало пути поиска исполняемых файлов. Система будет искать команды в ~/bin раньше, чем в системных /usr/bin: ` PATH="$HOME/bin:$PATH" `

- -i игнорирует регистр: ` grep -i error `
- -n добавляет номер строки к выводу: ` grep -n ERROR `
- Показывает 2 строки контекста (до и после) совпадения: ` grep -C 2 ERROR `
- Инвертирует поиск (выводит все строки, кроме тех, где есть ERROR): ` grep -v ERROR `
- -E включает расширенные регулярные выражения (логическое ИЛИ): ` grep -E "ERROR\|WARN"

- -d задает разделитель (пробел), -f выбирает поля (1-е и 9-е): ` cut -d' ' -f1,9 `

- Печатает первую колонку: ` awk '{print $1}'
- Накапливает сумму 9-й колонки в переменной sum. Блок END выполняется один раз после обработки всего файла: ` awk '{sum += $9} END {print sum}' `
- Условный вывод. Печатает всю строку ($0), только если 9-я колонка равна 200: ` awk '$9 == 200 {print $0}' `

- s = substitute (замена), g = global (все вхождения в строке, а не только первое): ` sed 's/old/new/g' `
- d = delete. Удаляет все строки, содержащие "ERROR": ` sed '/ERROR/d' `
- -i (in-place) сохраняет изменения прямо в исходный файл: ` sed -i 's/.../...' file `

- -r (reverse, по убыванию), -n (numeric, числовая сортировка, а не алфавитная): ` sort -rn `
- Считает количество подряд идущих одинаковых строк: ` uniq -c `
- Считает только строки (lines): ` wc -l `

- ` ~/.ssh/config `
	- Позволяет создавать алиасы для серверов, избавляя от необходимости каждый раз вводить ssh -i key -p 2222 user@host
- ` rsync -avz --delete `
	- ` -a ` (archive): сохраняет права, владельца, время модификации и рекурсивно копирует директории
	- ` -v ` (verbose): подробный вывод
	- ` -z `(compress): сжимает данные при передаче по сети
	- ` --delete `: делает зеркальную копию (удаляет на приемнике файлы, которые были удалены на источнике)
=======
	- Локальный проброс: ` ssh -L 8000:localhost:8000 myserver `
	- Удаленный проброс: ` ssh -R 8080:localhost:80 myserver `
	- Динамический проброс: ` ssh -D 1080 myserver `
