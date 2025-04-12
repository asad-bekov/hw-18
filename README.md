# Домашнее задание к занятию «Защита сети»
*Асадбеков Асадбек*

## Подготовка к выполнению

1. Подготовка защищаемой системы:

- установите **Suricata**,
- установите **Fail2Ban**.

2. Подготовка системы злоумышленника: установите **nmap** и **thc-hydra** либо скачайте и установите **Kali linux**.

Обе системы должны находится в одной подсети.

## Задание 1
Проведите разведку системы и определите, какие сетевые службы запущены на защищаемой системе:

`sudo nmap -sA < ip-адрес >`

`sudo nmap -sT < ip-адрес >`

`sudo nmap -sS < ip-адрес >`

`sudo nmap -sV < ip-адрес >`

По желанию можете поэкспериментировать с опциями: https://nmap.org/man/ru/man-briefoptions.html.

*В качестве ответа пришлите события, которые попали в логи Suricata и Fail2Ban, прокомментируйте результат*.

### Защищаемая система (IP: `192.168.10.1`):
- Установлена **Suricata** для обнаружения атак по сигнатурам.
- Установлен и настроен **Fail2Ban** для блокировки при подборе паролей по SSH.

```bash
sudo apt update && sudo apt install suricata fail2ban -y
```

### Система злоумышленника (IP: `192.168.10.2`):
- Установлены утилиты **nmap** и **hydra** для сканирования и брутфорса.

```bash
sudo apt update && sudo apt install nmap hydra -y
```

Обе виртуальные машины находятся в одной подсети `192.168.10.0/24` через интерфейс `enp0s8` (Host-Only).

---

## Задание 1: Разведка системы с помощью nmap

С атакующей машины были выполнены следующие команды:

```bash
nmap -sA 192.168.10.1
nmap -sT 192.168.10.1
nmap -sS 192.168.10.1
nmap -sV 192.168.10.1
```

Дополнительно был выполнен трафик на порт 8080:

```bash
curl http://192.168.10.1:8080
nc 192.168.10.1 8080
```

### Конфигурация правила в Suricata

Файл `/var/lib/suricata/rules/test.rules`:

```suricata
alert tcp any any -> any 8080 (msg:"TEST RULE - Access to port 8080"; sid:1000001; rev:1;)
```

Файл `/etc/suricata/suricata.yaml`, секция `rule-files`:

```yaml
rule-files:
  - suricata.rules
  - test.rules
```

### Перезапуск Suricata:

```bash
sudo systemctl restart suricata
```

### Лог fast.log:

```
[**] [1:1000001:1] TEST RULE - Access to port 8080 [**] {TCP} 192.168.10.2:xxxxx -> 192.168.10.1:8080
```

![alt text](https://github.com/asad-bekov/hw-18/blob/main/img/1.png)

### Fail2Ban:

На данном этапе — без срабатываний, т.к. не было попыток входа по SSH.

---

## Задание 2

Проведите атаку на подбор пароля для службы SSH:

`hydra -L users.txt -P pass.txt < ip-адрес > ssh`

1. Настройка **hydra**:

- создайте два файла: **users.txt** и **pass.txt**;
- в каждой строчке первого файла должны быть имена пользователей, второго — пароли. В нашем случае это могут быть случайные строки, но ради эксперимента можете добавить имя и пароль существующего пользователя.

Дополнительная информация по **hydra**: https://kali.tools/?p=1847.

2. Включение защиты SSH для **Fail2Ban**:

- открыть файл `/etc/fail2ban/jail.conf`,
- найти секцию ssh,
- установить `enabled` в `true`.

Дополнительная информация по Fail2Ban:https://putty.org.ru/articles/fail2ban-ssh.html.

*В качестве ответа пришлите события, которые попали в логи Suricata и Fail2Ban, прокомментируйте результат*.

## Брутфорс SSH с помощью Hydra

Создание файлов на атакующей машине:

```bash
echo -e "root
fakeuser" > users.txt
echo -e "1234
qwerty
netology" > pass.txt
```

Атака Hydra:

```bash
hydra -L users.txt -P pass.txt ssh://192.168.10.1
```

### Конфигурация Fail2Ban `/etc/fail2ban/jail.local`:

```ini
[sshd]
enabled = true
port = ssh
filter = sshd
backend = systemd
maxretry = 5
findtime = 600
bantime = 600
```

### Проверка статуса Fail2Ban:

```bash
sudo fail2ban-client status sshd
```

Вывод:
```
|- Currently failed: 1
|- Total failed: 9
|- Currently banned: 1
`- Banned IP list: 192.168.10.2
```

![alt text](https://github.com/asad-bekov/hw-18/blob/main/img/2.png)

**Лог fail2ban.log:**
```
Found 192.168.10.2 - 2025-04-12 12:17:13
Ban 192.168.10.2
```
![alt text](https://github.com/asad-bekov/hw-18/blob/main/img/3.png)
---

## Выводы:

- Suricata успешно обнаруживает сетевые подключения по заданным правилам.

- Fail2Ban эффективно блокирует злоумышленников при попытках брутфорса SSH.

Обе системы защиты работают корректно и усиливают безопасность сервера в локальной сети.

---
