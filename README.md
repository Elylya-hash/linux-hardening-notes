# linux-hardening-notes

Рабочие заметки по защите и диагностике Linux-серверов: чек-листы, готовые
конфигурации и шпаргалки по разбору типовых проблем. Команды рассчитаны на
Debian/Ubuntu с systemd.

## Содержание

### Чек-листы

- [Защита нового сервера](checklists/server-hardening.md) — обновления, учётные
  записи, SSH, фаервол, ядро, сервисы, журналы, резервные копии
- [Подозрение на взлом](checklists/incident-first-response.md) — что смотреть в
  первые минуты и чего не делать

### Конфигурации

- [sysctl/60-hardening.conf](sysctl/60-hardening.conf) — параметры ядра с
  пояснением к каждому
- [systemd/](systemd/) — юнит с изоляцией сервиса, таймер вместо cron,
  уведомление о сбое

### Диагностика

- [Высокая нагрузка](troubleshooting/high-load.md) — CPU, диск или память:
  как отличить и что делать дальше
- [Закончилось место](troubleshooting/disk-full.md) — место, inodes, удалённые
  открытые файлы
- [Сервис не запускается](troubleshooting/service-wont-start.md) — коды
  `status=` и частые причины
- [Сеть](troubleshooting/network.md) — проверка по уровням от интерфейса до TLS

## Связанные проекты

- [linux-admin-scripts](https://github.com/Elylya-hash/linux-admin-scripts) —
  Bash-скрипты для бэкапов, мониторинга и аудита
- [ansible-server-setup](https://github.com/Elylya-hash/ansible-server-setup) —
  те же меры защиты в виде Ansible-ролей

## Лицензия

[MIT](LICENSE)
