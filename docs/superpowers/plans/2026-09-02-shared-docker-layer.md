# Общий слой Docker — план реализации

> **Для исполнителя:** задачи идут строго по порядку. Шаги отмечаются чекбоксами.
> Каждая задача заканчивается проверкой и коммитом. Откат описан в задаче 5.

**Цель:** вынести точку входа сервера и общую инфраструктуру из compose-проекта
`game_world_tycoon_idle` в отдельный репозиторий ClaudeDocker; в репозиториях
проектов оставить compose, которым проект поднимается на отдельной машине.

**Архитектура:** ClaudeDocker владеет `nginx`, `portainer`, общими `postgres` и
`redis` и по одному оверлею на проект. Проекты объявляют свои standalone-сервисы
под профилем `standalone`, который на VPS не включается.

**Стек:** Docker Compose v5, `nginx:alpine` с штатным envsubst, `postgis/postgis:16-3.4`,
`redis:7-alpine`, `portainer/portainer-ce`.

**Спека:** `docs/superpowers/specs/2026-09-02-shared-docker-layer-design.md`

## Общие ограничения

- Репозиторий ClaudeDocker **публичный**: домены (содержат IP сервера), бинды
  портов и пароли — только в `.env`, в git уходит `.env.example` с пустыми значениями.
- Пути `root` в nginx менять нельзя: `SCRIPT_FILENAME` разворачивает php-fpm,
  который видит код по `/var/www/cm` и `/var/www/uq`.
- Сайт `uq` доступен только из VPN-подсети `10.8.0.0/24` — правила `allow`/`deny`
  переносятся дословно.
- Сертификаты берутся из `/etc/letsencrypt` хоста, ACME-челлендж — из `~/infra/acme`.
- Имена контейнеров `cm-php`, `uq-php` фиксированы: на них ссылаются конфиги.
- Тома `game_world_tycoon_idle_postgres_data` и `game_world_tycoon_idle_portainer_data`
  подключаются `external` под этими же именами. Данные не переносятся.

---

### Задача 1: Сеть общих данных и каркас репозитория

**Файлы:**
- Создать: `.env.example`, `.env`, `README.md`

- [ ] **Шаг 1: Создать сеть для общих данных**

```bash
docker network create shared-data
docker network ls | grep shared-data
```

Ожидаемо: сеть в списке. Именно на ней `postgres` и `redis` получат алиасы
`postgres` и `redis`, поэтому `DB_HOST=postgres` и `REDIS_HOST=redis` в `.env`
проектов останутся рабочими без правок.

- [ ] **Шаг 2: Написать `.env.example`**

```bash
# Домены. Содержат адрес сервера — реальные значения только в .env.
DOMAIN=
CM_DOMAIN=
UQ_DOMAIN=
MP_DOMAIN=

# Бинды портов (host:container). Публичные — nginx, остальное на loopback.
HTTP_BIND=80:80
HTTPS_BIND=443:443
PORTAINER_BIND=127.0.0.1:9000:9000

# Общий postgres: значения должны совпадать с .env проекта game.
POSTGRES_DB=
POSTGRES_USER=
POSTGRES_PASSWORD=
```

- [ ] **Шаг 3: Собрать `.env` из существующих значений, не печатая их**

```bash
cd /home/boris/projects/ClaudeDocker
G=/home/boris/projects/game_world_tycoon_idle/.env
{
  for k in DOMAIN CM_DOMAIN HTTP_BIND HTTPS_BIND PORTAINER_BIND; do grep -E "^$k=" "$G"; done
  echo "UQ_DOMAIN=uq.$(grep -E '^DOMAIN=' "$G" | cut -d= -f2-)"
  echo "MP_DOMAIN=myproject.$(grep -E '^DOMAIN=' "$G" | cut -d= -f2-)"
  grep -E '^DB_NAME=' "$G" | sed 's/^DB_NAME=/POSTGRES_DB=/'
  grep -E '^DB_USER=' "$G" | sed 's/^DB_USER=/POSTGRES_USER=/'
  grep -E '^DB_PASS=' "$G" | sed 's/^DB_PASS=/POSTGRES_PASSWORD=/'
} > .env
chmod 600 .env
cut -d= -f1 .env
```

Ожидаемо: список ключей без значений. `UQ_DOMAIN` и `MP_DOMAIN` собираются из
`DOMAIN` — проверить глазами, что они совпадают с `server_name` в
`uzbek_queue/deploy/nginx/uq.conf` и `myproject.conf`.

- [ ] **Шаг 4: Убедиться, что `.env` не попадёт в git**

```bash
git check-ignore -v .env
```

Ожидаемо: строка с `.gitignore:3:.env`.

- [ ] **Шаг 5: Коммит**

```bash
git add .env.example && git commit -m "feat: каркас общего слоя — переменные окружения"
```

---

### Задача 2: Шаблоны nginx

**Файлы:**
- Создать: `nginx/templates/nginx.conf.template`
- Создать: `nginx/templates/sites/{game,cm,uq,myproject}.conf.template`

**Интерфейс:** entrypoint `nginx:alpine` рендерит `templates/**` в `/etc/nginx/**`
с сохранением относительных путей, подставляя переменные из `NGINX_ENVSUBST_FILTER`.

- [ ] **Шаг 1: Каркас `nginx/templates/nginx.conf.template`**

Из нынешнего шаблона в `game` берётся всё, кроме server-блоков: они уезжают в `sites/`.

```nginx
events { worker_connections 1024; }

http {
  include mime.types;
  default_type application/octet-stream;

  # По одному файлу на проект. Рендерятся из templates/sites/*.conf.template.
  include /etc/nginx/sites/*.conf;
}
```

- [ ] **Шаг 2: `sites/game.conf.template`**

Переносится дословно из `game/nginx/templates/nginx.conf.template`: HTTP-редирект
с ACME (он обслуживает и `${DOMAIN}`, и `${CM_DOMAIN}` — оставить оба имени),
затем 443-блок игры целиком, включая `/devbot/webhook`, `/api`, `/socket.io` и
правила кеширования. `root /var/www/game` не меняется.

- [ ] **Шаг 3: `sites/cm.conf.template`**

Переносится 443-блок `${CM_DOMAIN}` дословно: `root /var/www/cm/public`,
`fastcgi_pass cm-php:9000`, `fastcgi_param HTTPS on`, `client_max_body_size 32m`,
запрет точечных файлов.

- [ ] **Шаг 4: `sites/uq.conf.template`**

Из `uzbek_queue/deploy/nginx/uq.conf`, домен заменяется на `${UQ_DOMAIN}`.
**Обязательно сохранить** `allow 10.8.0.0/24; allow 127.0.0.1; deny all;` — сайт VPN-only.

- [ ] **Шаг 5: `sites/myproject.conf.template`**

Из `uzbek_queue/deploy/nginx/myproject.conf`, домен на `${MP_DOMAIN}`,
апстрим `proxy_pass http://myproject:3003` без изменений.

- [ ] **Шаг 6: Проверить, что шаблоны рендерятся и nginx их принимает**

```bash
cd /home/boris/projects/ClaudeDocker
set -a; . ./.env; set +a
docker run --rm --entrypoint sh \
  -e DOMAIN -e CM_DOMAIN -e UQ_DOMAIN -e MP_DOMAIN \
  -e NGINX_ENVSUBST_OUTPUT_DIR=/etc/nginx \
  -e 'NGINX_ENVSUBST_FILTER=^(DOMAIN|CM_DOMAIN|UQ_DOMAIN|MP_DOMAIN)$' \
  -v "$PWD/nginx/templates:/etc/nginx/templates:ro" \
  nginx:alpine -c '/docker-entrypoint.sh true >/dev/null 2>&1; grep -h server_name /etc/nginx/sites/*.conf'
```

Ожидаемо: четыре пары `server_name` с подставленными доменами, без `${`.
`nginx -t` на этом шаге пройти не может — нет сертификатов и апстримов;
он выполняется в задаче 4 уже в боевом контейнере.

- [ ] **Шаг 7: Коммит**

```bash
git add nginx && git commit -m "feat: шаблоны nginx — по файлу на проект"
```

---

### Задача 3: docker-compose общего слоя

**Файлы:**
- Создать: `docker-compose.yml`

- [ ] **Шаг 1: Написать compose**

```yaml
# Общий слой VPS: точка входа, общие данные, просмотр контейнеров.
# Проекты сюда не входят — у них свои compose плюс оверлей в projects/.
name: shared

services:
  nginx:
    image: nginx:alpine
    ports:
      - "${HTTP_BIND}"
      - "${HTTPS_BIND}"
    environment:
      - DOMAIN=${DOMAIN}
      - CM_DOMAIN=${CM_DOMAIN}
      - UQ_DOMAIN=${UQ_DOMAIN}
      - MP_DOMAIN=${MP_DOMAIN}
      - NGINX_ENVSUBST_OUTPUT_DIR=/etc/nginx
      - NGINX_ENVSUBST_FILTER=^(DOMAIN|CM_DOMAIN|UQ_DOMAIN|MP_DOMAIN)$$
    volumes:
      - ./nginx/templates:/etc/nginx/templates:ro
      # Только public каждого проекта и строго по тому пути, который видит
      # его php-fpm: SCRIPT_FILENAME разворачивает php-fpm, а не nginx.
      - /home/boris/projects/game_world_tycoon_idle/public:/var/www/game:ro
      - /home/boris/projects/cross_messenger/public:/var/www/cm/public:ro
      - /home/boris/projects/uzbek_queue/public:/var/www/uq/public:ro
      - /etc/letsencrypt:/etc/letsencrypt:ro
      - /home/boris/infra/acme:/var/www/acme:ro
    networks:
      # web — cm-php и uq-php; claude-net — devbot и myproject;
      # game — алиас `server` для /api и /socket.io.
      - web
      - claude-net
      - game
    restart: unless-stopped

  postgres:
    image: postgis/postgis:16-3.4
    environment:
      - POSTGRES_DB=${POSTGRES_DB}
      - POSTGRES_USER=${POSTGRES_USER}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks: [data]
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    volumes:
      - redis_data:/data
    networks: [data]
    restart: unless-stopped

  portainer:
    image: portainer/portainer-ce:latest
    ports:
      - "${PORTAINER_BIND}"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data
    restart: unless-stopped

volumes:
  # Тома уже существуют и принадлежали compose-проекту game — подключаем их
  # под нынешними именами, чтобы данные остались на месте.
  postgres_data:
    external: true
    name: game_world_tycoon_idle_postgres_data
  portainer_data:
    external: true
    name: game_world_tycoon_idle_portainer_data
  redis_data:

networks:
  web:
    external: true
  claude-net:
    external: true
  game:
    external: true
    name: game_world_tycoon_idle_default
  data:
    external: true
    name: shared-data
```

- [ ] **Шаг 2: Проверить сборку конфигурации без запуска**

```bash
cd /home/boris/projects/ClaudeDocker && docker compose config >/dev/null && echo OK
```

Ожидаемо: `OK`. Ошибка `network ... declared as external, but could not be found`
означает, что пропущен шаг 1 задачи 1.

- [ ] **Шаг 3: Коммит**

```bash
git add docker-compose.yml && git commit -m "feat: compose общего слоя"
```

---

### Задача 4: Переключение точки входа

Здесь начинается простой. Файлы проектов до задачи 6 не меняются — это и есть путь отката.

- [ ] **Шаг 1: Записать эталон до переключения**

```bash
set -a; . /home/boris/projects/ClaudeDocker/.env; set +a
for d in "$DOMAIN" "$CM_DOMAIN" "$MP_DOMAIN"; do
  echo "$d -> $(curl -sk -o /dev/null -w '%{http_code}' "https://$d/")"
done
echo "uq -> $(curl -sk -o /dev/null -w '%{http_code}' "https://$UQ_DOMAIN/")  (403 снаружи VPN — норма)"
```

Записать вывод: с ним сравнивается результат после переключения.

- [ ] **Шаг 2: Погасить старые общие контейнеры**

```bash
docker stop game_world_tycoon_idle-nginx-1 game_world_tycoon_idle-adminer-1 game_world_tycoon_idle-portainer-1
```

- [ ] **Шаг 3: Поднять общий слой**

```bash
cd /home/boris/projects/ClaudeDocker && docker compose up -d
docker compose ps
```

Ожидаемо: `nginx`, `postgres`, `redis`, `portainer` в состоянии `running`.
`postgres` поднимается на существующем томе — в логах должен быть обычный старт,
а не инициализация новой базы.

- [ ] **Шаг 4: Проверить конфиг nginx внутри контейнера**

```bash
docker compose exec nginx nginx -t
```

Ожидаемо: `syntax is ok` и `test is successful`.

- [ ] **Шаг 5: Прогнать домены и сравнить с эталоном**

```bash
set -a; . ./.env; set +a
for d in "$DOMAIN" "$CM_DOMAIN" "$MP_DOMAIN" "$UQ_DOMAIN"; do
  echo "$d -> $(curl -sk -o /dev/null -w '%{http_code}' "https://$d/")"
done
```

Ожидаемо: те же коды, что в шаге 1.

- [ ] **Шаг 6: Проверить, что в интернет-контейнере нет секретов проектов**

```bash
docker compose exec nginx sh -c 'find /var/www -name ".env" | head; echo "---"; ls /var/www'
```

Ожидаемо: пустой список до `---`. Это то, что было сломано в прежней схеме.

- [ ] **Шаг 7: Проверить вебхук бота**

Отправить любое сообщение боту в Telegram и убедиться, что оно дошло до сессии.

- [ ] **Шаг 8: Откат, если что-то из проверок не сошлось**

```bash
cd /home/boris/projects/ClaudeDocker && docker compose down
docker start game_world_tycoon_idle-nginx-1 game_world_tycoon_idle-adminer-1 game_world_tycoon_idle-portainer-1
```

- [ ] **Шаг 9: Удалить контейнер adminer — он больше не нужен**

```bash
docker rm game_world_tycoon_idle-adminer-1
```

---

### Задача 5: Оверлеи проектов под VPS

**Файлы:**
- Создать: `projects/{game,cross_messenger,uzbek_queue,myproject}/docker-compose.yml`

**Интерфейс:** каждый оверлей применяется вместе с базовым файлом проекта:
`COMPOSE_FILE=docker-compose.yml:/home/boris/projects/ClaudeDocker/projects/<имя>/docker-compose.yml`
в `.env` проекта.

- [ ] **Шаг 1: `projects/game/docker-compose.yml`**

```yaml
# game на этом VPS: база и redis — из общего слоя, свой веб-сервер не нужен.
services:
  server:
    depends_on: []          # свой postgres выключен профилем
    networks: [default, data]

networks:
  data:
    external: true
    name: shared-data
```

- [ ] **Шаг 2: `projects/uzbek_queue/docker-compose.yml`**

```yaml
# uzbek_queue на этом VPS: postgres и redis — общие, nginx — общий.
services:
  php:
    depends_on: []
    networks: [default, web, data]
  worker:
    depends_on: []
    networks: [default, data]

networks:
  web:
    external: true
  data:
    external: true
    name: shared-data
```

- [ ] **Шаг 3: `projects/cross_messenger/docker-compose.yml`**

Содержимое переносится из `cross_messenger/docker-compose.vps.yml` (сервис `php`
получает сеть `web`). Своя `mysql` у проекта остаётся: она не общая.

- [ ] **Шаг 4: `projects/myproject/docker-compose.yml`**

```yaml
# myproject на этом VPS: порт наружу не публикуется, вход через общий nginx.
services:
  myproject:
    ports: !reset []
```

- [ ] **Шаг 5: Проверить, что каждый оверлей собирается**

```bash
for p in game_world_tycoon_idle:game uzbek_queue:uzbek_queue cross_messenger:cross_messenger myproject:myproject; do
  d=${p%%:*}; o=${p##*:}
  echo -n "$d: "
  docker compose -f /home/boris/projects/$d/docker-compose.yml \
    -f /home/boris/projects/ClaudeDocker/projects/$o/docker-compose.yml config >/dev/null && echo OK
done
```

Ожидаемо: четыре `OK`.

- [ ] **Шаг 6: Коммит**

```bash
git add projects && git commit -m "feat: оверлеи проектов под VPS"
```

---

### Задача 6: Вычистить game_world_tycoon_idle

**Файлы:**
- Изменить: `game_world_tycoon_idle/docker-compose.yml`, `.env`
- Удалить: `game_world_tycoon_idle/nginx/templates/nginx.conf.template` (блоки чужих проектов)

- [ ] **Шаг 1: Убрать из compose `nginx`, `adminer`, `portainer`; `postgres` оставить под профилем**

`postgres` получает `profiles: ["standalone"]` и остаётся в репозитории — им
проект поднимается на отдельной машине. Свой `nginx` для standalone добавляется
тем же профилем и обслуживает только game: без монтирований чужих каталогов,
без `claude-net`, порт из `${APP_PORT:-8080}`.

- [ ] **Шаг 2: Перенести переменные**

Из `game/.env` удаляются `HTTP_BIND`, `HTTPS_BIND`, `ADMINER_BIND`,
`PORTAINER_BIND`, `CM_DOMAIN` (они теперь в ClaudeDocker). `DOMAIN` остаётся:
на него ссылается сам сервер. Добавляется
`COMPOSE_FILE=docker-compose.yml:/home/boris/projects/ClaudeDocker/projects/game/docker-compose.yml`.

- [ ] **Шаг 3: Применить и проверить**

```bash
cd /home/boris/projects/game_world_tycoon_idle
docker compose config >/dev/null && docker compose up -d --remove-orphans
docker compose ps
```

Ожидаемо: остался один `server`. Затем повторить проверку доменов из задачи 4, шаг 5.

- [ ] **Шаг 4: Коммит в репозитории game**

```bash
git add -A && git commit -m "refactor: общий слой вынесен в ClaudeDocker"
```

---

### Задача 7: Вычистить uzbek_queue и починить worker

**Файлы:**
- Изменить: `uzbek_queue/docker-compose.yml`, `.env`
- Удалить: `uzbek_queue/deploy/nginx/`, `deploy/render-nginx.sh`, `deploy/set-domain.sh`

- [ ] **Шаг 1: Убрать сеть `shared`, добавить свои postgres и redis под профилем**

Сеть `shared` с явным `name: game_world_tycoon_idle_default` удаляется целиком —
это и была межпроектная связь. Появляются сервисы `postgres` и `redis` с
`profiles: ["standalone"]`, чтобы проект поднимался на чистой машине.

- [ ] **Шаг 2: Удалить каталог с конфигами nginx и скрипты рендера**

```bash
cd /home/boris/projects/uzbek_queue
git rm -r deploy/nginx deploy/render-nginx.sh deploy/set-domain.sh
```

Домен теперь живёт в `ClaudeDocker/.env` (`UQ_DOMAIN`), рендер делает штатный
entrypoint nginx. В `README` проекта добавить строку об этом, иначе при переезде
на другой VPS никто не найдёт, где менять домен.

- [ ] **Шаг 3: Прописать оверлей в `.env`**

```
COMPOSE_FILE=docker-compose.yml:/home/boris/projects/ClaudeDocker/projects/uzbek_queue/docker-compose.yml
COMPOSE_PROFILES=
```

- [ ] **Шаг 4: Применить**

```bash
cd /home/boris/projects/uzbek_queue && docker compose up -d
```

- [ ] **Шаг 5: Проверить, что redis наконец резолвится и worker жив**

```bash
docker exec uq-php sh -c 'getent hosts redis && getent hosts postgres'
sleep 60 && docker ps --filter name=uq-worker --format '{{.Status}}'
docker logs uq-worker --tail 20 | grep -i "redis\|error" || echo "ошибок redis нет"
```

Ожидаемо: оба имени резолвятся; статус `Up` больше минуты без рестартов;
в логах нет ошибок подключения. До этой задачи `uq-worker` падал в цикле.

- [ ] **Шаг 6: Коммит**

```bash
git add -A && git commit -m "refactor: свои postgres и redis, конфиг nginx уехал в ClaudeDocker"
```

---

### Задача 8: cross_messenger и myproject

- [ ] **Шаг 1: cross_messenger — убрать vps-оверлей из репозитория**

```bash
cd /home/boris/projects/cross_messenger && git rm docker-compose.vps.yml
```

В `.env` заменить `COMPOSE_FILE` на путь к оверлею в ClaudeDocker. В шапке
`docker-compose.yml` поправить комментарий: порты 80/443 держит не «nginx
соседнего проекта», а общий слой.

- [ ] **Шаг 2: Сервис `nginx` из local-оверлея перевести в базовый файл под профилем**

Так базовый compose становится самодостаточным: `docker compose --profile standalone up`
на чистой машине поднимает php, mysql и nginx.

- [ ] **Шаг 3: myproject — публикация порта под профилем**

В `myproject/docker-compose.yml` добавить `ports` с `profiles: ["standalone"]`
и `.env.example` с `COMPOSE_PROFILES=standalone`; на VPS оверлей их снимает.

- [ ] **Шаг 4: Применить и проверить оба домена**

```bash
cd /home/boris/projects/cross_messenger && docker compose up -d
cd /home/boris/projects/myproject && docker compose up -d
set -a; . /home/boris/projects/ClaudeDocker/.env; set +a
for d in "$CM_DOMAIN" "$MP_DOMAIN"; do echo "$d -> $(curl -sk -o /dev/null -w '%{http_code}' "https://$d/")"; done
```

- [ ] **Шаг 5: Коммиты в обоих репозиториях**

---

### Задача 9: Документация и финальная проверка

- [ ] **Шаг 1: README ClaudeDocker**

Описать: что здесь лежит, правило владения («ресурс нужен более чем одному
проекту — он здесь»), как подключить новый проект (файл в `nginx/templates/sites/`,
переменная домена в `.env`, оверлей в `projects/`), как запускать.

- [ ] **Шаг 2: Обновить `ClaudeService/docs/vps-rebuild-prompts.md`**

Сборка на чистом VPS теперь начинается с ClaudeDocker: сети `web`, `claude-net`,
`shared-data`, затем общий слой, затем проекты.

- [ ] **Шаг 3: Полная проверка**

```bash
set -a; . /home/boris/projects/ClaudeDocker/.env; set +a
for d in "$DOMAIN" "$CM_DOMAIN" "$MP_DOMAIN" "$UQ_DOMAIN"; do
  echo "$d -> $(curl -sk -o /dev/null -w '%{http_code}' "https://$d/")"
done
docker ps --format '{{.Names}}\t{{.Status}}'
docker exec uq-php getent hosts redis
```

Ожидаемо: коды как в эталоне задачи 4; в `game` только `server`;
`uq-worker` без рестартов; `redis` резолвится.

- [ ] **Шаг 4: Коммит и push ClaudeDocker**

---

## Вне объёма

`migr-mysql9-test` (поднят голым `docker run`), перевод `~/infra/vpn` под git,
веб-сервер для `my_portal`.
