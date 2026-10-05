# Чек-лист: защита нового сервера

Порядок пунктов важен: сначала доступ, потом ограничения. Команды даны для
Debian/Ubuntu. Автоматизированный вариант первых разделов —
[ansible-server-setup](https://github.com/Elylya-hash/ansible-server-setup).

## 1. Обновления

- [ ] Установить все обновления

  ```bash
  apt update && apt full-upgrade
  ```

- [ ] Включить автоматическую установку обновлений безопасности

  ```bash
  apt install unattended-upgrades
  dpkg-reconfigure -plow unattended-upgrades
  # проверить, что будет установлено:
  unattended-upgrade --dry-run --debug
  ```

- [ ] Проверить, не требуется ли перезагрузка: `cat /var/run/reboot-required`

## 2. Учётные записи

- [ ] Создать личную учётную запись с sudo, не работать под root

  ```bash
  adduser alice && usermod -aG sudo alice
  ```

- [ ] Проверить, что UID 0 только у root

  ```bash
  awk -F: '$3 == 0 {print $1}' /etc/passwd
  ```

- [ ] Проверить, что нет учётных записей с пустым паролем

  ```bash
  awk -F: '$2 == "" {print $1}' /etc/shadow
  ```

- [ ] Просмотреть, у кого есть sudo: `getent group sudo`, `ls /etc/sudoers.d/`
- [ ] Заблокировать ненужные учётные записи: `usermod -L -e 1 <user>`

## 3. SSH

> Перед изменениями откройте **вторую** SSH-сессию и не закрывайте её, пока не
> убедитесь, что новый вход работает.

- [ ] Положить свой публичный ключ: `ssh-copy-id alice@server`
- [ ] Создать `/etc/ssh/sshd_config.d/00-hardening.conf`

  ```
  PermitRootLogin no
  PasswordAuthentication no
  KbdInteractiveAuthentication no
  PermitEmptyPasswords no
  MaxAuthTries 3
  X11Forwarding no
  ```

  Префикс `00-` важен: sshd берёт первое встреченное значение параметра, и файл
  `50-cloud-init.conf` с `PasswordAuthentication yes` иначе окажется главнее.

- [ ] Проверить синтаксис и итоговые значения, затем перезапустить

  ```bash
  sshd -t
  sshd -T | grep -Ei 'permitrootlogin|passwordauthentication'
  systemctl restart ssh
  ```

- [ ] Проверить вход из новой сессии

## 4. Фаервол

- [ ] Разрешить SSH **до** включения фаервола

  ```bash
  ufw limit 22/tcp comment ssh
  ufw default deny incoming
  ufw default allow outgoing
  ufw enable
  ufw status verbose
  ```

- [ ] Сравнить открытые порты с ожидаемыми

  ```bash
  ss -tulpn
  ```

  Всё, что слушает `0.0.0.0` или `[::]` и не должно быть доступно снаружи,
  перевести на `127.0.0.1` или закрыть.

- [ ] Помнить: Docker публикует порты в обход ufw. Для контейнеров указывать
      адрес явно: `-p 127.0.0.1:8080:80`.

## 5. Защита от перебора

- [ ] Установить fail2ban и включить jail для sshd

  ```bash
  apt install fail2ban
  printf '[sshd]\nenabled = true\nbackend = systemd\n' > /etc/fail2ban/jail.d/sshd.local
  systemctl restart fail2ban
  fail2ban-client status sshd
  ```

## 6. Ядро и файловая система

- [ ] Применить [sysctl/60-hardening.conf](../sysctl/60-hardening.conf)

  ```bash
  cp 60-hardening.conf /etc/sysctl.d/ && sysctl --system
  ```

- [ ] Смонтировать `/tmp` и `/dev/shm` с `nodev,nosuid,noexec`, если это не
      ломает приложения

  ```bash
  findmnt -o TARGET,OPTIONS /tmp /dev/shm
  ```

- [ ] Найти неожиданные setuid-файлы

  ```bash
  find / -xdev -perm -4000 -type f 2>/dev/null
  ```

- [ ] Найти файлы, доступные на запись всем

  ```bash
  find / -xdev -type f -perm -0002 2>/dev/null
  ```

## 7. Сервисы

- [ ] Отключить ненужное

  ```bash
  systemctl list-unit-files --state=enabled --type=service
  systemctl disable --now <service>
  ```

- [ ] Оценить изоляцию собственных сервисов и усилить её
      (пример — [systemd/myapp.service](../systemd/myapp.service))

  ```bash
  systemd-analyze security
  systemd-analyze security myapp.service
  ```

## 8. Журналы и время

- [ ] Проверить синхронизацию времени: `timedatectl status`
      (без точного времени логи бесполезны при разборе инцидентов)
- [ ] Ограничить размер журнала: `SystemMaxUse=1G` в `/etc/systemd/journald.conf`
- [ ] Настроить отправку логов на внешний сервер — локальные логи
      злоумышленник с root изменит первым делом

## 9. Резервные копии

- [ ] Настроить резервное копирование
      ([backup.service](../systemd/backup.service) + [backup.timer](../systemd/backup.timer))
- [ ] Хранить копии вне сервера
- [ ] **Проверить восстановление.** Бэкап, из которого ни разу не
      восстанавливались, бэкапом не считается.

## Проверка результата

```bash
ss -tulpn                              # только ожидаемые порты
sshd -T | grep -i passwordauth         # passwordauthentication no
ufw status                             # Status: active
fail2ban-client status sshd
systemctl --failed                     # пусто
```

Для сравнения с отраслевыми рекомендациями — аудит утилитой `lynis audit system`.
