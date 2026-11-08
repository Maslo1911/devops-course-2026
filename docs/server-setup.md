# Конфигурация виртуальной машины devops-vm

Гипервизор: VMware Workstation (в методичке — VirtualBox). ОС: Ubuntu Server 26.04.1 LTS, ядро 7.0.0-43-generic.
На хосте Windows 10 гипервизор Hyper-V отключён (`bcdedit /set hypervisorlaunchtype off`):
при работе VMware поверх Hyper-V гостевая система зависала (soft lockup CPU).

## 1. Параметры машины

| Параметр | Значение |
|---|---|
| Имя узла | `devops-vm` (FQDN `devops-vm.devops.local`) |
| Оперативная память | 2048 МБ |
| Процессор | 2 ядра |
| Диск | 25 ГБ, динамически расширяемый |

## 2. Сетевые интерфейсы

| Интерфейс | Тип адаптера (сеть VMware) | Адрес | Назначение |
|---|---|---|---|
| `ens33` | NAT (VMnet8, 192.168.91.0/24) | 192.168.91.129/24, DHCP | доступ в Интернет, репозитории пакетов; через него идёт проброс порта |
| `ens34` | Host-only (VMnet1, 192.168.153.0/24) | 192.168.153.129/24, DHCP | прямой доступ с хоста (devops.local), веб-сервер в ПР6 |

На хосте службы `VMware NAT Service` и `VMware DHCP Service` должны быть запущены
(тип запуска Automatic), иначе интерфейсы не получают адреса.

## 3. Правило проброса портов

Задаётся в VMware: Edit → Virtual Network Editor → VMnet8 → NAT Settings → Add
(файл `C:\ProgramData\VMware\vmnetnat.conf`, секция `[incomingtcp]`, затем перезапуск `VMware NAT Service`).

| Протокол | Порт хоста | Адрес гостя | Порт гостя |
|---|---|---|---|
| TCP | 2222 | 192.168.91.129 | 2222 (до задания 2 — 22) |

```
[incomingtcp]
2222 = 192.168.91.129:2222
```

Подключение с хоста: `ssh devops` (= `ssh -p 2222 -i ~/.ssh/devops_vm devops@127.0.0.1`).

## 4. Учётные записи

| Имя | Группы | Аутентификация по SSH |
|---|---|---|
| `student` (uid 1000) | student, adm, cdrom, sudo, dip, plugdev, users, lxd | запрещена (`AllowUsers devops`); вход только с консоли VM |
| `devops` (uid 1001) | devops, sudo, users | только по ключу ed25519 `~/.ssh/devops_vm` (комментарий `devops-vm-key`) |

Восстановление:

```bash
sudo adduser devops
sudo usermod -aG sudo devops
# на хосте:
ssh-keygen -t ed25519 -C "devops-vm-key" -f ~/.ssh/devops_vm
type $env:USERPROFILE\.ssh\devops_vm.pub | ssh devops@127.0.0.1 -p 2222 "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

Права на сервере: `~/.ssh` — 700, `~/.ssh/authorized_keys` — 600.

`~/.ssh/config` на хосте:

```
Host devops
    HostName 127.0.0.1
    Port 2222
    User devops
    IdentityFile ~/.ssh/devops_vm
    IdentitiesOnly yes
```

## 5. Служба SSH

- Порт: **2222**.
- Основной файл `/etc/ssh/sshd_config` не изменялся; копия — `/etc/ssh/sshd_config.backup`.
- Изменения — в файле `/etc/ssh/sshd_config.d/99-hardening.conf`:

```
Port 2222
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
PermitEmptyPasswords no
MaxAuthTries 3
LoginGraceTime 30
AllowUsers devops
X11Forwarding no
ClientAliveInterval 300
ClientAliveCountMax 2
```

- В `/etc/ssh/sshd_config.d/50-cloud-init.conf` закомментирована строка `PasswordAuthentication yes`
  (файл читается раньше `99-hardening.conf`, и действует первое встреченное значение).
- Активация по сокету отключена, иначе порт берётся из `ssh.socket`:

```bash
sudo sshd -t
sudo systemctl disable --now ssh.socket
sudo systemctl enable --now ssh.service
sudo systemctl restart ssh
sudo ss -tlnp | grep sshd    # 0.0.0.0:2222
```

После этого изменить порт гостя в правиле проброса (раздел 3) на 2222.

## 6. Правила межсетевого экрана

Политики по умолчанию: `deny (incoming)`, `allow (outgoing)`, `disabled (routed)`. Журналирование: `medium`.

| Порт | Действие | Комментарий |
|---|---|---|
| 2222/tcp | LIMIT IN | SSH rate-limited |
| 80/tcp | ALLOW IN | HTTP (ПР6) |
| 443/tcp | ALLOW IN | HTTPS |

Правила действуют для IPv4 и IPv6. Порядок восстановления (разрешающее правило — до `enable`):

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw limit 2222/tcp comment 'SSH rate-limited'
sudo ufw allow 80/tcp comment 'HTTP'
sudo ufw allow 443/tcp comment 'HTTPS'
sudo ufw enable
sudo ufw logging medium
```

### Локальное доменное имя

- Хост (`C:\Windows\System32\drivers\etc\hosts`): `192.168.153.129   devops.local`, затем `ipconfig /flushdns`.
- Сервер: `sudo hostnamectl set-hostname devops-vm`, в `/etc/hosts` строка
  `127.0.1.1 devops-vm.devops.local devops-vm`; `hostname -f` → `devops-vm.devops.local`.

## 7. Снимки состояния

| Наименование | Момент создания | Состояние |
|---|---|---|
| `01-clean-install` | 2026-11-08 17:38 | чистая установка после `apt upgrade`, VM выключена |
| `02-keys-configured` | 2026-11-08 17:44 | создан пользователь devops, вход по ключу |
| `03-ssh-hardened` | 2026-11-08 17:52 | SSH усилен, порт 2222 (UFW ещё не включён) |
| `04-nginx-https` | 2026-11-08 18:33 | Nginx, ресурс devops.local по HTTPS |

## 8. Веб-сервер

- Пакет: `nginx` из репозитория Ubuntu (1.28.3), `sudo apt install -y nginx`. Главный процесс — root, рабочие — `www-data`.
- Порты 80 и 443 открыты правилами UFW из раздела 6; профили `Nginx Full` и т. п. не используются.
- Конфигурация ресурса: `/etc/nginx/sites-available/devops-site`, ссылка
  `/etc/nginx/sites-enabled/devops-site`; стандартный ресурс отключён удалением ссылки
  `/etc/nginx/sites-enabled/default` (файл в `sites-available` сохранён).
- Каталог ресурса: `/var/www/devops-site`, владелец `devops:devops`, каталоги 755, файлы 644.
  Содержимое доставляется из каталога `site/` репозитория сценарием `scripts/deploy.sh`
  (`rsync -avz --delete --chmod=D755,F644`), на хосте — из WSL1 (в Git Bash нет rsync).
- Журналы: `/var/log/nginx/devops-site.access.log`, `/var/log/nginx/devops-site.error.log`.
- Сертификат: `/etc/ssl/certs/devops.crt`, `root:root`, 644; закрытый ключ: `/etc/ssl/private/devops.key`,
  `root:root`, 600. Самоподписанный, RSA 2048, срок действия 365 дней (до 2027-11-08), SAN `DNS:devops.local`:

```bash
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/ssl/private/devops.key \
  -out /etc/ssl/certs/devops.crt \
  -subj "/CN=devops.local" \
  -addext "subjectAltName=DNS:devops.local"
```

- Клиенту сертификат передаётся по SSH: `scp devops:/etc/ssl/certs/devops.crt ~/devops.crt`,
  проверка — `curl --cacert ~/devops.crt -I https://devops.local`.
- `/etc/nginx/sites-available/devops-site`:

```nginx
# Блок 1: перенаправление HTTP -> HTTPS
server {
    listen 80;
    listen [::]:80;
    server_name devops.local;
    return 301 https://$host$request_uri;
}

# Блок 2: ресурс по HTTPS
server {
    listen 443 ssl;
    listen [::]:443 ssl;
    server_name devops.local;

    ssl_certificate     /etc/ssl/certs/devops.crt;
    ssl_certificate_key /etc/ssl/private/devops.key;
    ssl_protocols       TLSv1.2 TLSv1.3;

    root /var/www/devops-site;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }

    error_page 404 /404.html;

    access_log /var/log/nginx/devops-site.access.log;
    error_log  /var/log/nginx/devops-site.error.log;
}
```

Применение: `sudo nginx -t && sudo systemctl reload nginx` (не `restart`). Заголовок HSTS не добавляется.

## Проверка

`scripts/audit.sh` на сервере: все проверки пройдены, код возврата 0.
