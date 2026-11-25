# ЛР 2. Docker Compose

Compose-проект из трёх сервисов на основе приложения и Dockerfile из ЛР 1: веб-сервис со счётчиком посещений, одноразовый сервис подготовки данных и nginx перед приложением.

```
init  --(завершился с кодом 0)-->  app  --(стал healthy)-->  proxy  <-- localhost:8080
  \                                 /
   `---- общий том lab2-data ------'          все три в сети lab2-net
```

| Файл | Что это |
|------|---------|
| `docker-compose.yml` | описание сервисов, сети и тома |
| `.env` | все переменные окружения и настройки |
| `Dockerfile` | образ приложения (из ЛР 1), собирается compose автоматически |
| `.dockerignore` | что не отправлять в сборку |
| `app/app.py`, `app/requirements.txt` | код приложения и библиотеки |
| `nginx/default.conf.template` | настройки nginx, адрес приложения подставляется из `.env` |

Каждый файл подробно прокомментирован внутри. Этот README: краткая сводка.

## Запуск

```bash
docker compose up -d --build   # собрать образ и запустить
docker compose ps -a           # init: Exited (0), app и proxy: Up (healthy)
curl localhost:8080/           # Visits: 101
curl localhost:8080/           # Visits: 102
docker compose down            # удалить контейнеры и сеть, том остаётся
docker compose up -d
curl localhost:8080/           # Visits: 103, счётчик сохранился в томе
docker compose down -v         # удалить всё вместе с томом
```

## Сервисы

| Сервис | Контейнер | Образ | Что делает |
|--------|-----------|-------|------------|
| `init` | `lab2-init` | `lab2-app:1.0`, собирается из `Dockerfile` | Одноразовый. Если в томе нет файла счётчика, создаёт его со значением `COUNTER_START` из `.env`. Если есть, ничего не меняет. Завершается с кодом 0. |
| `app` | `lab2-app` | `lab2-app:1.0` | Flask-приложение под gunicorn на порту 5000. Наружу не открыт, доступен только из сети `lab2-net`. |
| `proxy` | `lab2-proxy` | `nginx:1.27-alpine` | Обратный прокси. Принимает запросы на порт 8080 компьютера и пересылает их в `lab2-app:5000`. |

## Как выполнены требования

| Требование | Где в `docker-compose.yml` |
|------------|----------------------------|
| 1 init + 2 app сервиса, работают в связке | `init` готовит файл в томе, `app` его читает и увеличивает, `proxy` пересылает запросы в `app` |
| Сборка образа из лежащего рядом Dockerfile с именем | `build: .` и `image: ${APP_IMAGE}` (`lab2-app:1.0`) у `init` и `app` |
| Жёсткое именование контейнеров | `container_name: lab2-init`, `lab2-app`, `lab2-proxy` |
| `depends_on` | `app` ждёт `service_completed_successfully` от `init`, `proxy` ждёт `service_healthy` от `app` |
| `volume` | именованный том `lab2-data` в `/data` у `init` и `app`; bind mount файла настроек nginx у `proxy` |
| Проброс порта наружу | `proxy`: `"${PROXY_PORT}:80"` (8080:80) |
| `command` / `entrypoint` | `init`: `entrypoint: ["/bin/sh", "-c"]` + `command` со скриптом; `app`: `command` с запуском gunicorn |
| `healthcheck` | `app`: запрос к `/health` из python; `proxy`: запрос через nginx к `/health` через `wget` |
| Все env в `.env` | у каждого сервиса `env_file: .env`, в самом compose только ссылки `${...}`; блоков `environment:` нет |
| Явная общая сеть | `networks: lab2-net` (bridge), к ней подключены все три сервиса |

Дополнительно: у `app` ограничены ресурсы (`deploy.resources.limits`: 0.5 CPU, 256 МБ), у `app` и `proxy` `restart: unless-stopped`, у `init` `restart: "no"`. Строки `version:` нет: современный Docker Compose её не использует и предупреждает, что она устарела.

Проверить итоговую конфигурацию с подставленными значениями из `.env`: `docker compose config`.

## Ответы на вопросы

### Можно ли ограничивать ресурсы (память, CPU) для сервисов в docker-compose.yml?

Да. Compose передаёт ограничения в Docker, а Docker применяет их через механизм ядра Linux cgroups, тот же, что у флагов `docker run --memory` и `--cpus`.

Основной способ, ключ `deploy.resources`:

```yaml
services:
  app:
    deploy:
      resources:
        limits:            # верхняя граница
          cpus: "0.50"     # половина одного ядра
          memory: 256M     # при превышении процесс завершается (OOM)
          pids: 100        # максимум процессов
        reservations:      # гарантированный минимум
          memory: 64M
```

Раньше `deploy` работал только в Docker Swarm, а для обычного `docker compose` писали отдельные ключи. Современный Docker Compose применяет `deploy.resources` и без Swarm. Старые ключи тоже поддерживаются: `mem_limit: 256m`, `cpus: 0.5`, `mem_reservation`, `cpu_shares`, `pids_limit`.

В этом проекте ограничение задано у `app`. Проверка: `docker stats lab2-app` показывает `MEM USAGE / LIMIT ... / 256MiB`, а `docker inspect lab2-app --format '{{.HostConfig.Memory}} {{.HostConfig.NanoCpus}}'` выводит `268435456 500000000`.

### Как запустить только определённый сервис, не запуская остальные?

- `docker compose up -d app` запускает только `app`, но вместе с теми, от кого он зависит по `depends_on` (здесь `init`). `proxy` не запустится.
- `docker compose up -d --no-deps proxy` запускает только `proxy`, без зависимостей. Ему нужно, чтобы `app` уже работал, иначе nginx будет отвечать 502.
- `docker compose run --rm init` разово запускает один сервис как отдельную команду и удаляет контейнер после завершения. Удобно для одноразовых задач вроде `init`. Порты из `ports` при `run` не пробрасываются, если не добавить `--service-ports`.
- `docker compose start app` запускает уже созданный, но остановленный контейнер сервиса.
- Профили: если у сервиса указать `profiles: [debug]`, он не запускается обычным `docker compose up` и стартует только с `docker compose --profile debug up`.
