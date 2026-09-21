### Задача 5

**Условия**
1. Скрипты должны быть устойчивы к ошибкам
2. Генерировать читаемые отчеты в текстовом или JSON формате
3. Создать и управлять LVM структурой (PV, VG, LV)
4. Настроить RAID массивы и отслеживать их состояние
5. Настроить мониторинг

> Использую Ubuntu server 24.04
---
**Практика**

- Анализ дискового пространства
	- Вывод информации о всех смонтированных файловых системах: ` df -hT `
	- Ищем раздел с наибольшим использованием места: ` df -hT | awk 'NR>1 {print $6, $7}' | sort -rn | head -1 ` / ` df -hT | awk 'NR>1 {gsub(/%/,"",$6); print $6, $7}' | sort -rn | head -5 `
	
- Поиск больших файлов
	- Находим все файлы размером больше 100MB в /var/log: ` find /var/log -type f -size +100M -exec ls -lh {} \; ` / ` find /var/log -type f -size +100M -printf "%s %p\n" | sort -rn `
	- Считаем общий размер файлов в /home: ` du -sh /home ` / ` du -sh /home/* `
	- Ищем 10 больших директорий в /var: ` du -h /var 2>/dev/null | sort -rh | head -10 `

- Работа с inode
	- Проверка использование inode на всех разделах: ` df -i ` / ` df -i | awk 'NR>1 {printf "%s: %d used (%.2f%%)\n", $6, $3, ($3/$2)*100}' `
	- Ищем директорию с наибольшим кол-во файлов: ` find / -xdev -type d -exec sh -c 'echo "$(find "$1" -maxdepth 1 | wc -l) $1"' _ {} \; 2>/dev/null | sort -rn | head -10 `
	- Скрипт для поиска директорий с 10000-ми файлов:
	```
	cat > /tmp/find_many_files.sh << 'EOF'
	#!/bin/bash
	SEARCH_DIR="${1:-/home}"
	THRESHOLD="${2:-10000}"

	echo "Searching for directories with more than $THRESHOLD files in $SEARCH_DIR"
	echo "========================================================================"

	find "$SEARCH_DIR" -type d 2>/dev/null | while read dir; do
		count=$(find "$dir" -maxdepth 1 -type f 2>/dev/null | wc -l)
		if [ "$count" -gt "$THRESHOLD" ]; then
			echo "$count files in $dir"
		fi
	done | sort -rn
	EOF
	
	# chmod +x /tmp/find_many_files.sh
	# /tmp/find_many_files.sh /home 10000
	```
	
- Проблема с inode
	- Создаем директорию: ` mkdir -p /tmp/test_inode `
	- Создаем в ней 1000 пустых файлов: ` seq 1 1000 | xargs -I {} touch "file_{}.txt" `
	- Смотрим сколько inode использовано: ` df -i /tmp/test_inode `
	- Считаем сколько файлов в директори: ` find /tmp/test_inode -type f | wc -l `
	- Чистим: ` rm -rf /tmp/test_inode `
	
- Монтирование
	- Выводим список всех смонтрованных ФС: ` mount ` / ` mount | column -t `
	- Находим все NFS/CIFS: ` mount | grep -E 'nfs|cifs|smbfs' `

- Настройка авто-монтирования
	- Создаем резервную копию: ` sudo cp /etc/fstab /etc/fstab.backup_$(date +%Y%m%d) `
	- Создаем точку монтирования: ` sudo mkdir -p /mnt/temp `
	- Смотрим содержимое fstab: ` cat /etc/fstab
	- Добавляем запись: ` echo "tmpfs /mnt/temp tmpfs defaults,size=512M 0 0" | sudo tee -a /etc/fstab `
	- Проверка синтаксиса: ` sudo findmnt --verify `
	- Монтируем для теста: ` sudo mount /mnt/temp `
	- Проверяем: ` df -h /mnt/temp

- Анализ файловой системы
	- Определяем тип файловой системы для всех разделов: ` lsblk -l ` / ` blkid `
	- Смотрим зарезервированное место: ` sudo tune2fs -l /dev/sda2 | grep -i reserved `
	- Смотрим информацию о superblock: ` sudo dumpe2fs -h /dev/sda1 `
