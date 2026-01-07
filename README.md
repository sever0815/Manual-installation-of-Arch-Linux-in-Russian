# Как установить Arch Linux ручным способом
## Гайд был написан для тех, кто уже что-то знает в использовании linux дистрибутивов, поэтому гайд не более подробный, но обобщенный
## Стоит учесть что все описанное ниже предназначено для железа с поддержкой UEFI


## 1. Подключение к сети
### Используем iwctl
#### Введите `iwctl`
```
station wlan0 scan
station wlan0 get-networks
station wlan0 connect имя_сети
Пароль
```
#### Чтобы выйти введите `exit`
#### Проверка соединения
```
ping google.com
```
## 2. Разметка диска
#### Введите `lsblk` чтобы ориентироваться в разделах
##### Например:
```
NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
nvme0n1     259:0    0 476,9G  0 disk 
├─nvme0n1p1 259:1    0     1G  0 part /boot/efi
├─nvme0n1p2 259:2    0 462,2G  0 part /
└─nvme0n1p3 259:3    0  13,7G  0 part [SWAP]
```
#### `nvme0n1` сам диск, а `nvme0n1p1` и ниже разделы

### Используем `cfdisk`
```
cfdisk /dev/nvme0n1
```
### Интерфейс `cfdisk`

| Кнопка     | Что делает                                                                     |
| ---------- | ------------------------------------------------------------------------------ |
| **New**    | Создает новый раздел в выбранной области свободного места.                     |
| **Type**   | Меняет "паспорт" раздела (важно для EFI — поставить тип `EFI System`).         |
| **Delete** | Удаляет выбранный раздел.                                                      |
| **Write**  | **Самый важный шаг.** Пока вы не нажали Write, никакие изменения не применены. |
| **Quit**   | Выход из программы.                                                            |
### Создание разделов

Используйте стрелки **Влево/Вправо** для выбора кнопок внизу и **Вверх/Вниз** для выбора свободного места (Free space).

#### Шаг 1: EFI-раздел (загрузчик)

1. Выберите **[ New ]**, нажмите Enter.
	
2. Введите размер: `512MiB - 1GiB` .
	
3. Теперь выберите кнопку **[ Type ]** и в появившемся списке выберите **`EFI System`** (обычно самый первый в списке).

#### Шаг 2: Root-раздел (система и файлы)

1. Выберите оставшееся **Free space**.
	
2. Снова **[ New ]**, нажмите Enter (по умолчанию предложит все оставшееся место — соглашайтесь).
    
3. По умолчанию тип будет `Linux filesystem`, что нам и нужно.

### Сохранение изменений

Когда вы увидите в списке два раздела (например, `nvme0n1p1` и `nvme0n1p2`), сделайте следующее:

1. Выберите **[ Write ]**.
    
2. Введите слово **`yes`** (полностью) для подтверждения.
    
3. Выберите **[ Quit ]**.

### Форматирование

```
# Форматируем EFI в FAT32
mkfs.fat -F 32 /dev/nvme0n1p1

# Форматируем систему в EXT4
mkfs.ext4 /dev/nvme0n1p2
```

### Монтирование
#### 1. Корневой раздел
```
mount /dev/nvme0n1p2 /mnt
```
#### 2. Загрузочный раздел
```
mount --mkdir /dev/nvme0n1p1 /mnt/boot/efi
```

## 3. Установка пакетов
### Смена зеркал
#### 1. Откроем файл
```
nano /etc/pacman.d/mirroslist
```
#### 2. Для удобства воспользуемся заменой:
#### `ctrl + \`
#### Хотим заменить `Server` на `#Server`
#### 3. Ищем ближайший или в котором вы находитесь регион и стираем в начале каждой строки решетки (**#**)
#### 4. Сохраняем файл `ctrl + O`, `Enter`, `ctrl + X`


### Основные компоненты и microcode
#### Для **Amd** обязательно ставим `amd-ucode`,
#### а для **Intel** - `intel-ucode`
##### Пример для Amd:
```
pacstrap -K /mnt base amd-ucode base-devel linux linux-firmware linux-headers nano networkmanager pipewire wireplumber pipewire-alsa pipewire-pulse pipewire-jack xdg-desktop-portal grub efibootmgr sudo
```

## 4. Конфигурация системы
### 1) Генерируем таблицу разделов и заходим внутрь системы:
```
genfstab -U /mnt >> /mnt/etc/fstab
arch-chroot /mnt
```
### 2) Локализация и время
```
ln -sf /usr/share/zoneinfo/Europe/Moscow /etc/localtime
hwclock --systohc

# Раскомментируйте en_US.UTF-8 и ru_RU.UTF-8 в /etc/locale.gen
nano /etc/locale.gen
locale-gen

# Пример для русской локализации
echo "LANG=ru_RU.UTF-8" > /etc/locale.conf

# Имя хоста (ПК)
echo "archlinux" > /etc/hostname
```
### 3) Запуск службы NetworkManager
```
systemctl enable NetworkManager
```
### 4) Пользователь, пароли и права суперпользователя
#### Отредактируйте файл sudoers (`/etc/sudoers`)
```
nano /etc/sudoers
```
#### В конце файла нужно раскомментировать (удалить символ решетки (**#**) в начале строки) строки:
```
# %wheel ALL=(ALL:ALL) ALL
```
#### Должно получиться так:
```
## Uncomment to allow members of group wheel to execute any command  
%wheel ALL=(ALL:ALL) ALL
```
#### Установка паролей и создание пользователя
```
passwd # Пароль root
useradd -m -G wheel имя_пользователя # Создание пользователя, создание домашнего раздела для него и добавление в группу для возможности использовать права суперпользователя
passwd имя_пользователя # Пароль пользователя
```
### 4) Репозитории
#### Изначально доступны только `core` и `extra`, нужно добавить `multilib` для 32 битных программ и библиотек, например **Steam**
```
nano /etc/pacman.conf
# Найдите строку [multilib], раскомментируйте ее и строку ниже Include=
# Должно получиться так:

#[core-testing]  
#Include = /etc/pacman.d/mirrorlist  
  
[core]  
Include = /etc/pacman.d/mirrorlist  
  
#[extra-testing]  
#Include = /etc/pacman.d/mirrorlist  
  
[extra]  
Include = /etc/pacman.d/mirrorlist  
  
# If you want to run 32 bit applications on your x86_64 system,  
# enable the multilib repositories as required here.  
  
#[multilib-testing]  
#Include = /etc/pacman.d/mirrorlist  
  
[multilib]  
Include = /etc/pacman.d/mirrorlist
```
### 5) Загрузчик (GRUB)
```
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id="Arch Linux"

grub-mkconfig -o /boot/grub/grub.cfg
```

## 5. Размонтирование разделов
### Выйдите из окружения chroot, набрав `exit` или нажав `Ctrl+d`.
### Вы можете размонтировать все разделы с помощью команды `umount -R /mnt`, чтобы убедиться в том, что ни один из разделов не остался занят какой-либо программой. Если нужно, для поиска таких программ используйте [fuser(1)](https://man.archlinux.org/man/fuser.1).

### Теперь перезагрузите компьютер, набрав `reboot`: если какие-нибудь разделы остались смонтированными, [systemd](https://wiki.archlinux.org/title/Systemd_\(%D0%A0%D1%83%D1%81%D1%81%D0%BA%D0%B8%D0%B9\) "Systemd (Русский)") их размонтирует. Не забудьте извлечь установочный носитель. После загрузки войдите в систему в качестве суперпользователя (root).


# После установки 

## Установка соединения с Wi-Fi
### Введите `nmtui`, выберите опцию для активации подключения, выберите сеть, введите пароль

## Драйвера
### Если у вас встроенная в процессор графика или дискретная карта от amd, то скорей всего нам ничего дополнительно ставить не нужно, но я бы посоветовал на всякий случай установить некоторые пакеты
### Пользователям дискретных карт NVIDIA установка драйверов обязательна

https://wiki.archlinux.org/title/Graphics_processing_unit#Installation

### Общая таблица
![](screenshots/n1.png)

### Более подробные таблицы
#### AMD
![](screenshots/n2.png)
#### Intel
![](screenshots/n3.png)
#### NVIDIA
![](screenshots/n4.png)
### Рекомендую сразу установить lib32 пакеты нужного вам драйвера


## Остальные пакеты
### `power-profiles-daemon` - профили питания, обычно используются для ноутбуков
### Для **bluetooth** нужно установить пакеты  `bluez`,  `bluez-utils` и запустить службу
```
sudo systemctl enable bluetooth.service
```
