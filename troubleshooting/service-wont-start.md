# Сервис systemd не запускается

## Порядок действий

```bash
systemctl status myapp.service          # состояние и последние строки лога
journalctl -u myapp.service -b -e       # журнал за текущую загрузку, с конца
journalctl -u myapp.service -f          # следить в реальном времени
systemctl cat myapp.service             # итоговый юнит вместе с drop-in файлами
systemd-analyze verify /etc/systemd/system/myapp.service
```

После правки юнита — `systemctl daemon-reload`. Если упёрлись в лимит
перезапусков («Start request repeated too quickly») —
`systemctl reset-failed myapp.service`.

## Коды из `status=`

В строке `Main PID: ... (code=exited, status=203/EXEC)` число до косой черты —
код, после — этап, на котором systemd не смог запустить процесс.

| Код | Причина | Что проверить |
|---|---|---|
| `200/CHDIR` | не удалось перейти в `WorkingDirectory=` | каталог существует, права |
| `203/EXEC` | не удалось выполнить `ExecStart=` | путь, бит `x`, shebang, флаг `noexec` у раздела, SELinux/AppArmor |
| `209/STDOUT` | не удалось открыть файл для вывода | путь в `StandardOutput=` |
| `217/USER` | пользователь из `User=` не существует | `getent passwd <user>` |
| `226/NAMESPACE` | не удалось собрать изоляцию | пути в `ReadWritePaths=`, `BindPaths=` существуют |
| `1`, `2`, ... | приложение запустилось и само завершилось с ошибкой | лог приложения |
| `status=9/KILL` | убит сигналом | OOM-killer: `journalctl -k \| grep -i oom`; `MemoryMax=` |
| `status=31/SYS` | запрещённый системный вызов | слишком строгий `SystemCallFilter=` |

## Частые ситуации

**Запускается вручную, но не из systemd.** Окружение разное: у systemd нет
ваших `PATH`, `HOME`, переменных из `.bashrc`, а рабочий каталог — `/`.
Указывайте абсолютные пути, задавайте `Environment=` / `EnvironmentFile=` и
`WorkingDirectory=`. Воспроизвести окружение сервиса:

```bash
systemd-run --pty --uid=myapp --working-directory=/opt/myapp /opt/myapp/bin/myapp
```

**Стартует и сразу «inactive (dead)».** Процесс уходит в фон, а в юните
`Type=simple`. Либо запускать приложение без демонизации (предпочтительно),
либо `Type=forking` и `PIDFile=`.

**Зависает в «activating» и падает по таймауту.** `Type=notify`, а
приложение не отправляет `sd_notify(READY=1)`, или `Type=forking`, а процесс
не форкается.

**«Permission denied» после включения изоляции.** `ProtectSystem=strict`
делает файловую систему доступной только для чтения. Каталоги для записи
задаются через `StateDirectory=`, `LogsDirectory=` или `ReadWritePaths=`.

**«Address already in use».** Порт занят: `ss -tlnp | grep :8080`.

**Не дожидается сети или базы.** `After=` задаёт только порядок и не
гарантирует готовность зависимости. Для сети —
`After=network-online.target` плюс `Wants=network-online.target`; для базы —
повторные попытки подключения в самом приложении или `Restart=on-failure`.

## Проверка изоляции

Если сервис перестал работать после ужесточения юнита, удобно временно снять
ограничения через drop-in и возвращать их по одному:

```bash
systemctl edit myapp.service      # создаёт override.conf
```

```ini
[Service]
SystemCallFilter=
MemoryDenyWriteExecute=no
```

Запрещённые системные вызовы видны в журнале аудита:
`journalctl -k | grep -i seccomp` или `ausearch -m SECCOMP`.
