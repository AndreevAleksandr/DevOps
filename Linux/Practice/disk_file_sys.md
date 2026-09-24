### Задача 5

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

- Работа с файловой системой дисков
	- ext4: ` mkfs.ext4 /dev/sdb1 `
	- xfs: ` mkfs.xfs /dev/sdb2 `
	- Создаем точки монтирования: ` mkdir -p /mnt/test_ext4 ` / ` mkdir -p /mnt/test_xfs `
	- Монтируем: ` mount /dev/sdb1 /mnt/test_ext4 ` / ` mount /dev/sdb2 /mnt/test_xfs `
	- Тестируем скорость создания файлов: ` time for i in {1..10000}; do touch /mnt/test_ext4/file_$i; done ` / ` time for i in {1..10000}; do touch /mnt/test_xfs/file_$i; done `
	- Анализ метаданных: ` dumpe2fs -h /dev/sdb1 | grep -E 'Filesystem features|Inode count|Block count ` / ` xfs_info /dev/sdb2 `
	- Очистка: ` umount /mnt/test_ext4 /mnt/test_xfs ` / ` rm -rf /mnt/test_ext4/* /mnt/test_xfs/* `
	
- LVM (Logical Volume Manager)
	- Создаем раздел: ` parted /dev/sdb mklabel msdos ` / ` parted /dev/sdb mkpart primary 1MiB 100% ` / ` parted /dev/sdb set 1 lvm on `
	- Создаем PV: ` pvcreate /dev/sdb1 `
	- Проверяем: ` pvdisplay `
	- Создаем Volume Group "vg_data": ` vgcreate -s 4M vg_data /dev/sdb1 `
	- Проверяем: ` vgdisplay `
	- Создаем Logical Volume "lv_home": ` lvcreate -L 10G -n lv_home vg_data `
	- Указываем PE: ` lvcreate -l 2560 -n lv_home vg_data `
	- Проверяем: ` lvdisplay /dev/vg_data/lv_home `
	- Форматируем в ext4: ` mkfs.ext4 /dev/vg_data/lv_home `
	- Создаем точку и монтируем: ` mkdir -p /data ` / ` mount /dev/vg_data/lv_home /data `
	- Проверяем: ` df -h /data `
	- Добавляем в /etc/fstab для авто-монтирования: ` echo "/dev/vg_data/lv_home /data ext4 defaults 0 0" | tee -a /etc/fstab `
	
- Управление LVM
	- Смотрим свободное место в VG: ` vgs vg_data `
	- Расширяем LV: ` lvextend -L +5G /dev/vg_data/lv_home `
	- Расширяем файловую систему ext4 / xfs: ` resize2fs /dev/vg_data/lv_home ` / ` xfs_growfs /data `
	- Проверяем: ` df -h /data ` / ` lvdisplay /dev/vg_data/lv_home `
	- Создаем snapshot lv_home: ` lvcreate -L 1G -s -n lv_home_snap /dev/vg_data/lv_home `
	- Проверяем: ` lvs -a vg_data ` / ` lvdisplay /dev/vg_data/lv_home_snap `
	- Монтируем snapshot: ` mkdir -p /mnt/snapshot ` / ` mount -o ro /dev/vg_data/lv_home_snap /mnt/snapshot `
	- Смотрим информацию о всех LV, VG, PV: ` pvdisplay ` - ` pvs ` / ` vgdisplay ` - ` vgs ` / ` lvdisplay ` - ` lvs `
	- Размонтируем: ` umount /data ` / ` umount /mnt/snapshot `
	- Удаляем snap: ` lvremove /dev/vg_data/lv_home_snap `
	- Проверяем файловую систему: ` e2fsck -f /dev/vg_data/lv_home `
	- Уменьшаем ФС: ` resize2fs /dev/vg_data/lv_home 12G `
	- Уменьшаем LV: ` lvreduce -L 12G /dev/vg_data/lv_home `
	- Монтируем обратно:  ` mount /dev/vg_data/lv_home /data `
	- Проверяем: ` df -h /data `
	
- RAID
	- Создаем разделы на /dev/sdb и /dev/sdc: ` parted /dev/sdc mklabel msdos ` / ` parted /dev/sdc mkpart primary 1MiB 100% ` / ` parted /dev/sdc set 1 raid on ` **Тоже самое для второго диска**
	- Создаем RAID 1: ` mdadm --create --verbose /dev/md0 --level=1 --raid-devices=2 /dev/sdc1 /dev/sdd1 `
	- Отслеживаем процесс синхронизации: ` watch cat /proc/mdstat `
	- Смотрим информацию о массиве: ` mdadm --detail /dev/md0 `
	- Сохраняем в файл: ` mdadm --detail /dev/md0 > /tmp/raid_info.txt `
	- Создаем ФС на RAID: ` mkfs.ext4 /dev/md0 `
	- Монтируем: ` mkdir -p /mnt/raid1 ` / ` mount /dev/md0 /mnt/raid1 `
	- Проверяем: ` df -h /mnt/raid1 `

- Управление RAID
	- **Симуляция отказа диска**
	- Помечаем неисправный диск: ` mdadm --manage /dev/md0 --fail /dev/sdd1 `
	- Удаляем диск из массива: ` mdadm --manage /dev/md0 --remove /dev/sdd1 `
	- Смотрим состояние: ` mdadm --detail /dev/md0 `
	- Добавляем новый диск в массив: ` mdadm --manage /dev/md0 --add /dev/sdd1 `
	- Отслеживаем rebuild: ` watch cat /proc/mdstat `
	- Подробная информация: ` mdadm --detail /dev/md0 | grep -E 'State|Rebuild|Reshape' `
	
- LVM поверх RAID 5
	- Создаем разделы /dev/sde, /dev/sdf, /dev/sdg: ` parted /dev/sde mklabel msdos ` / ` parted /dev/sde mkpart primary 1MiB 100% ` / ` parted /dev/sde set 1 raid on ` **Тоже самое для второго диска**
	- Создаем RAID 5: ` mdadm --create --verbose /dev/md1 --level=5 --raid-devices=3 /dev/sde1 /dev/sdf1 /dev/sdg1 `
	- Проверяем: ` watch cat /proc/mdstat `
	- Создаем PV: ` pvcreate /dev/md1 `
	- Создаем VG: ` vgcreate vg_raid /dev/md1 `
	- Создаем 2 LV: ` lvcreate -L 10G -n lv_data1 vg_raid ` / ` lvcreate -L 10G -n lv_data2 vg_raid `
	- Проверяем: ` lvs ` / ` vgs ` / ` pvs `
	- Форматируем в ext4: ` mkfs.ext4 /dev/vg_raid/lv_data1 ` / ` mkfs.ext4 /dev/vg_raid/lv_data2 `
	- Создаем точки монтирования: ` mkdir -p /data1 ` / ` mkdir -p /data2 `
	- Монтируем: ` mount /dev/vg_raid/lv_data1 /data1 ` / ` mount /dev/vg_raid/lv_data2 /data2 `
	- Проверяем: ` df -h | grep -E 'data1|data2' `
	- **Авто-монтирование**
	- Получаем UUID: ` blkid /dev/vg_raid/lv_data1 ` / ` blkid /dev/vg_raid/lv_data2 `
	- Добавляем в fstab:
	```
	bash -c 'cat >> /etc/fstab << EOF
	/dev/vg_raid/lv_data1  /data1  ext4  defaults  0  2
	/dev/vg_raid/lv_data2  /data2  ext4  defaults  0  2
	EOF'
	```
	- Тестируем fstab: ` mount -a `
	- Проверяем: ` df -h | grep data `