# systemd: примеры юнитов

| Файл | Что показывает |
|---|---|
| [myapp.service](myapp.service) | Изоляция сетевого сервиса: `DynamicUser`, `ProtectSystem=strict`, фильтр системных вызовов, лимиты ресурсов |
| [backup.service](backup.service) | `Type=oneshot` с пониженным приоритетом CPU и I/O и уведомлением об ошибке |
| [backup.timer](backup.timer) | Замена cron: `OnCalendar`, `Persistent`, случайная задержка |
| [notify-failure@.service](notify-failure@.service) | Шаблонный юнит для `OnFailure=` |

## Как ужесточать существующий сервис

1. Посмотреть текущую оценку (0 — лучше, 10 — хуже):

   ```bash
   systemd-analyze security myapp.service
   ```

2. Добавлять директивы группами через drop-in и после каждой проверять работу:

   ```bash
   systemctl edit myapp.service
   systemctl restart myapp.service && journalctl -u myapp.service -e
   ```

3. Разумный порядок — от безопасных к рискованным:

   | Шаг | Директивы | Что может сломаться |
   |---|---|---|
   | 1 | `NoNewPrivileges`, `PrivateTmp`, `ProtectHome`, `ProtectKernel*`, `ProtectControlGroups` | почти ничего |
   | 2 | `ProtectSystem=strict` + `StateDirectory`/`ReadWritePaths` | запись в неучтённые каталоги |
   | 3 | `User=`/`DynamicUser=`, `CapabilityBoundingSet=` | порты <1024, права на файлы |
   | 4 | `RestrictAddressFamilies`, `PrivateDevices` | нестандартные сокеты, доступ к устройствам |
   | 5 | `SystemCallFilter`, `MemoryDenyWriteExecute` | JIT, редкие системные вызовы |

Оценка `systemd-analyze security` — ориентир, а не цель: она показывает, какие
механизмы включены, но не знает, что нужно конкретному приложению.

## Таймер вместо cron

Что даёт таймер по сравнению со строкой в crontab:

- вывод задачи попадает в журнал: `journalctl -u backup.service`;
- видно время прошлого и следующего запуска: `systemctl list-timers`;
- `Persistent=true` выполняет пропущенный запуск после включения сервера;
- два экземпляра одновременно не запустятся;
- к задаче применимы лимиты ресурсов и изоляция, как к любому сервису.

Проверить выражение расписания:

```bash
systemd-analyze calendar "Mon..Fri *-*-* 02:00:00"
```
