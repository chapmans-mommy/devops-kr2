# Контрольная работа 2

Вариант 3 для номера 7 в списке группы. Студент: Ivanova.

Приложение Flask из [шаблона преподавателя](https://gitverse.ru/dgimatdinov/kr2-template) опубликовано через Nginx по адресу https://stats.devops.local. Порт приложения — 8003.

## Конфигурация

- Dockerfile использует python:3.12-slim, как в ПР 7, устанавливает зависимости до копирования app.py и запускает Gunicorn от app с UID 999.
- Compose запускает app и Redis 7 Alpine в сети backend. Приложение ждёт healthy Redis.
- Redis требует пароль, сохраняет AOF в именованном томе redis_data. Порт 6379 не опубликован.
- Публикация app: 127.0.0.1:8003:8003. Перезапуск обоих сервисов: unless-stopped.
- Конфигурация Nginx: config/kr2. Старый ресурс devops-site не изменён.
- Единственный секрет REDIS_PASSWORD хранится только в /home/devops/kr2/.env на ВМ, права 600, владелец devops. Его значение в репозитории отсутствует. .env.example содержит только заглушки; несекретные параметры заданы в Compose.

## Доставка

Из корня репозитория:

```sh
rsync -avz --delete --exclude=.env --exclude=.git --dry-run ./ devops:kr2/
rsync -avz --delete --exclude=.env --exclude=.git ./ devops:kr2/
```

На сервере первоначально создаётся .env с новым случайным паролем (openssl rand -hex 32), umask 077. Не экспортируйте REDIS_PASSWORD в рабочей оболочке. Существующий .env при доставке сохраняется.

```sh
cd ~/kr2
docker compose config -q
docker compose up -d --build
docker compose ps
docker compose top
docker compose port app 8003
docker compose port redis 6379
```

Конфигурация Compose содержит ссылки на переменную; вывод docker compose config и полный docker inspect контейнера могут раскрыть пароль. Не включайте эти выводы в отчёт.

## HTTPS

В hosts Windows добавлена запись 192.168.56.101 stats.devops.local. Отдельный самоподписанный сертификат с SAN DNS:stats.devops.local находится в /etc/ssl/certs/kr2.crt. Закрытый ключ — /etc/ssl/private/kr2.key, root:root, 600. Nginx применён через nginx -t и systemctl reload nginx.

```sh
curl -I http://stats.devops.local/
curl --cacert ~/kr2.crt https://stats.devops.local/
curl --cacert ~/kr2.crt https://stats.devops.local/whoami
curl -m 3 http://stats.devops.local:8003/health
```

Ожидаются 301, строка KR2 со счётчиком, scheme: https и недоступность прямого порта. Без --cacert самоподписанный сертификат вызывает ошибку проверки.

## Контроль сохранности

Счётчик перед down: 4; после up: 5. Флаг -v не использовался. Правила UFW не менялись. Аудит по фактическому пути /home/devops/scripts/audit.sh: 14 проверок, 0 ошибок, код 0. Для неинтерактивного SSH сценарий запускается целиком через sudo.

История main сохранена из шаблона и содержит учебный .env. Пароль шаблона не используется: действующий пароль сгенерирован заново на ВМ. В последнем коммите рабочей ветки .env отсутствует. Логика app.py и requirements.txt сохранены; в app.py добавлен только комментарий первой строкой.

## Диагностика

При 502 проверьте docker compose ps, curl http://127.0.0.1:8003/health и журнал /var/log/nginx/kr2.error.log. При 500 с работающим /health проверьте docker compose logs app redis, разрешение имени redis и healthcheck. Секреты и закрытый ключ в журнал не выводите.
