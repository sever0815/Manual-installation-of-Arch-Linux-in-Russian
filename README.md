# Ручная установка Arch Linux

## 1. Подключение к сети
### Используем iwctl
```
iwctl
station wlan0 scan
station wlan0 get-networks
station wlan0 connect имя_сети
Пароль
```
#### Проверка соединения
```
ping google.com
```
#### Если пинг приходит, то ctrl + C и идем дальше


## 2. Разметка диска
#### Введите **lsblk** чтобы было проще ориентироваться в разделах
##### Например:
```
NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
nvme0n1     259:0    0 476,9G  0 disk 
├─nvme0n1p1 259:1    0     1G  0 part /boot/efi
├─nvme0n1p2 259:2    0 462,2G  0 part /
└─nvme0n1p3 259:3    0  13,7G  0 part [SWAP]
```
#### **nvme0n1** сам диск, а **nvme0n1p1** и ниже разделы

### Используем cfdisk
```
cfdisk /dev/nvme0n1
```
### Интерфейс cfdisk

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

### 3) Пользователь, пароли и права суперпользователя

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

### 4) Загрузчик (GRUB)
```
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id="Arch Linux"

grub-mkconfig -o /boot/grub/grub.cfg
```


## 5. Установка драйверов
 