# Infrastructure Setup Guide
> Руководство по развёртыванию доменной инфраструктуры AU-TEAM.IRPO

---

## Содержание

- [HQ-SRV — Файловое хранилище (RAID + NFS)](#hq-srv--файловое-хранилище-raid--nfs)
- [ISP — Служба сетевого времени (Chrony)](#isp--служба-сетевого-времени-chrony)
- [BR-SRV — Ansible](#br-srv--ansible)
- [BR-SRV — Samba Domain Controller](#br-srv--samba-domain-controller)
- [HQ-CLI — Ввод в домен](#hq-cli--ввод-в-домен)
---

## HQ-SRV — Файловое хранилище (RAID + NFS)

### 1. Создание RAID-0

```bash
lsblk
mdadm --create --level=0 --raid-devices=2 /dev/md/md0 /dev/sdb /dev/sdc
mdadm --detail /dev/md/md0
mdadm --detail --scan >> /etc/mdadm.conf
reboot
lsblk   # проверка нового устройства
```

### 2. Форматирование и монтирование

```bash
mkfs.ext4 /dev/md/md0
mkdir /raid

RAID_UUID=$(blkid -s UUID -o value /dev/md/md0)
echo $RAID_UUID

echo "UUID=\"$RAID_UUID\" /raid ext4 defaults 0 0" >> /etc/fstab
mount -a
```

### 3. Настройка NFS-сервера

```bash
apt-get install -y nfs-server
systemctl enable --now nfs
mkdir /raid/nfs

# Экспорт директории (сеть HQ-CLI: 192.168.200.0/29)
echo "/raid/nfs 192.168.200.0/29(rw,no_subtree_check)" >> /etc/exports
```

### 4. Настройка NFS-клиента (HQ-CLI)

```bash
apt-get install nfs-utils

# Проверить доступность HQ-SRV
mkdir /mnt/nfs
echo "192.168.100.2:/raid/nfs /mnt/nfs nfs defaults 0 0" >> /etc/fstab
```

---

## ISP — Служба сетевого времени (Chrony)

### Настройка сервера (ISP)

```bash
apt-get install chrony
systemctl enable --now chronyd
```

Файл `/etc/chrony.conf`:

```
pool ru.pool.ntp.org iburst
allow all
local stratum 5
```

```bash
systemctl restart chronyd
```

### Настройка клиентов (все остальные устройства)

Файл `/etc/chrony.conf`:

```
server 172.16.1.1 iburst
```

```bash
systemctl restart chronyd
chronyc sources -v
```

---

## BR-SRV — Ansible

### 1. Подготовка управляемых узлов

На каждом устройстве из инвентаря:

```bash
apt-get install -y openssh-server python3
systemctl enable --now  sshd
systemctl status sshd
```

### 2. Установка Ansible на BR-SRV

```bash
apt-get install -y ansible-core sshpass
```

### 3. Генерация и распространение SSH-ключа

```bash
ssh-keygen -t rsa

# Передать ключ на каждое устройство
ssh-copy-id -p 2026 sshuser@192.168.3.2
```

### 4. Файл инвентаря `/etc/ansible/hosts`

Формат (с явными IP):

```ini
HQ-SRV  ansible_host=192.168.100.2  ansible_user=sshuser    ansible_password=P@ssw0rd  ansible_port=2026
HQ-CLI  ansible_host=192.168.200.2  ansible_user=user       ansible_password=user
HQ-RTR  ansible_host=172.16.1.2   ansible_user=user  ansible_password=user 
BR-RTR  ansible_host=172.16.2.2   ansible_user=user  ansible_password=user  
```

### 5. Конфигурация `/etc/ansible/ansible.cfg`

```ini
[defaults]
inventory = /etc/ansible/hosts
host_key_checking = False
```

### 6. Проверка связности

```bash
ansible -m ping all
```

---
## BR-SRV — Samba Domain Controller

### 1. Очистка предыдущей конфигурации

```bash
rm -rf /run/samba/* /var/lib/samba/* /var/cache/samba/*
mkdir -p /var/lib/samba/sysvol
```

### 2. Установка пакетов

```bash
apt-get update
apt-get install -y bind-utils task-samba-dc
```

### 3. Проверка конфликтующих служб (должны быть **неактивны**)

```bash
systemctl status krb5kdc
systemctl status slapd
systemctl status bind
```

### 4. Резервирование старого конфига

```bash
mv /etc/samba/smb.conf{,.orig}
```

### 5. Настройка `/etc/resolv.conf`

```
search au-team.irpo
nameserver 127.0.0.1
```

### 6. Провизионирование домена

```bash
samba-tool domain provision \
  --server-role=dc \
  --use-rfc2307 \
  --dns-backend=SAMBA_INTERNAL \
  --realm=AU-TEAM.IRPO \
  --domain=AU-TEAM \
  --adminpass=P@ssw0rd \
  --option="interfaces=ens18"
```

### 7. Конфигурация Kerberos

```bash
cp /var/lib/samba/private/krb5.conf /etc/krb5.conf
```

### 8. Запуск Samba

```bash
systemctl enable --now samba
systemctl restart samba
```

### 9. Проверка DNS

```bash
host -t SRV _kerberos._udp.au-team.irpo
```

### 10. Создание пользователей и группы

```bash
samba-tool group add hq
samba-tool group list

for i in {1..5}; do
  samba-tool user add hquser$i P@ssw0rd
  samba-tool group addmembers "hq" hquser$i
done
```

---

## HQ-CLI — Ввод в домен

### 1. Установка пакета

```bash
qapt-get update
apt-get install -y task-auth-ad-sssd libnss-role krb5-kinit
```

### 2. DNS — указать BR-SRV в `/etc/resolv.conf`

```bash
host au-team.irpo   # ожидаемый ответ: 192.168.3.2
```

### 3. Ввод в домен

Через графические настройки: указать домен `au-team.irpo`, рабочую группу `au-team`, имя ПК, пароль администратора домена.

### 4. Проверка на BR-SRV

```bash
samba-tool computer list
```

### 5. Получение Kerberos-тикета

```bash
kinit hquser1@AU-TEAM.IRPO
```

### 6. Настройка sudo

```bash

# Связать доменную группу hq с локальной группой wheel
roleadd hq wheel

# Добавить разрешения в sudoers
echo "%wheel ALL=(ALL:ALL) /bin/cat, /bin/grep, /usr/bin/id" \
  >> /etc/sudoers.d/demo
```

### 7. Тестирование

```bash
exit && logout
su - hquser1@AU-TEAM.IRPO
sudo id                  # должен выполниться
sudo apt-get update      # должен быть запрещён
```
