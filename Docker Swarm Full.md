# Docker Swarm: полное и исчерпывающее руководство для Senior-разработчика (PHP 8.2 + Symfony 6.4)

> Это руководство — практический справочник по **Docker Swarm** для backend-разработчика, который пишет на PHP 8.2 / Symfony 6.4 и хочет уверенно эксплуатировать свои сервисы в кластере. Внутри: архитектура Swarm и алгоритм Raft, модель сервисов и задач, scheduling, overlay-сети и routing mesh, secrets/configs, stack-файлы, rolling и zero-downtime деплой, интеграция с Symfony (PHP-FPM + Nginx, Messenger workers, миграции, cron), CI/CD через GitLab/GitHub Actions, мониторинг (Prometheus + Loki), безопасность, тестирование, troubleshooting, anti-patterns и честное сравнение с Kubernetes/Nomad/обычным `docker compose`. Все примеры рабочие и приближены к production-задачам.

---

## Оглавление

1. [[#1. Что такое Docker Swarm и зачем он нужен|Что такое Docker Swarm и зачем он нужен]]
2. [[#2. Архитектура кластера и Raft|Архитектура кластера и Raft]]
3. [[#3. Модель сервисов, задач и контейнеров|Модель сервисов, задач и контейнеров]]
4. [[#4. Инициализация кластера, joining, lifecycle нод|Инициализация кластера, joining, lifecycle нод]]
5. [[#5. Scheduling, constraints, placement preferences|Scheduling, constraints, placement preferences]]
6. [[#6. Сети Swarm: overlay, ingress, routing mesh|Сети Swarm: overlay, ingress, routing mesh]]
7. [[#7. Тома, bind-mounts, плагины хранения|Тома, bind-mounts, плагины хранения]]
8. [[#8. Secrets и configs|Secrets и configs]]
9. [[#9. Stack-файлы и docker compose v3|Stack-файлы и docker compose v3]]
10. [[#10. Rolling updates, rollback, zero-downtime деплой|Rolling updates, rollback, zero-downtime деплой]]
11. [[#11. Healthchecks, restart policy, update policy|Healthchecks, restart policy, update policy]]
12. [[#12. Масштабирование и распределение нагрузки|Масштабирование и распределение нагрузки]]
13. [[#13. Production-стенд для Symfony 6.4 (Nginx + PHP-FPM)|Production-стенд для Symfony 6.4 (Nginx + PHP-FPM)]]
14. [[#14. Symfony Messenger workers в Swarm|Symfony Messenger workers в Swarm]]
15. [[#15. Миграции, cron и одноразовые задачи|Миграции, cron и одноразовые задачи]]
16. [[#16. Reverse proxy и TLS: Traefik в Swarm|Reverse proxy и TLS: Traefik в Swarm]]
17. [[#17. CI/CD: сборка образов и `docker stack deploy`|CI/CD: сборка образов и `docker stack deploy`]]
18. [[#18. Логирование и мониторинг (Prometheus, Loki, Grafana)|Логирование и мониторинг (Prometheus, Loki, Grafana)]]
19. [[#19. Безопасность кластера и образов|Безопасность кластера и образов]]
20. [[#20. Резервное копирование и disaster recovery|Резервное копирование и disaster recovery]]
21. [[#21. Тестирование инфраструктуры и приложения|Тестирование инфраструктуры и приложения]]
22. [[#22. Типичные ошибки и анти-паттерны|Типичные ошибки и анти-паттерны]]
23. [[#23. Troubleshooting: диагностика на живом кластере|Troubleshooting: диагностика на живом кластере]]
24. [[#24. Сравнение с Kubernetes, Nomad и docker compose|Сравнение с Kubernetes, Nomad и docker compose]]
25. [[#25. Когда не стоит брать Swarm|Когда не стоит брать Swarm]]
26. [[#26. Проверочные вопросы с ответами|Проверочные вопросы с ответами]]
27. [[#27. Источники|Источники]]

---

## 1. Что такое Docker Swarm и зачем он нужен

**Docker Swarm** (точнее — **Swarm mode**, встроенный в Docker Engine начиная с 1.12, 2016 г.) — это нативный оркестратор контейнеров от Docker Inc. Он превращает группу хостов с установленным Docker в единый логический кластер, в котором можно декларативно описывать сервисы и обновлять их без простоя.

**Что Swarm делает за вас:**
- Поддерживает заданное число реплик (`replicas: 5` — всегда 5 запущенных контейнеров, если хватает ресурсов).
- Распределяет контейнеры по нодам (scheduling) с учётом меток, ресурсов и предпочтений.
- Балансирует трафик через **routing mesh** (любая нода принимает трафик и проксирует на нужную реплику).
- Шифрует управляющий и (опционально) data-plane между нодами (mTLS из коробки).
- Хранит конфиги и секреты в зашифрованном Raft-хранилище.
- Делает rolling update и автоматический rollback при сбоях healthcheck.
- Поднимает упавшие контейнеры (self-healing).

**Бизнес-ценность:**
- **Низкий порог входа:** если ваш проект уже работает в `docker compose`, переезд в Swarm — это правка нескольких полей в YAML и команда `docker stack deploy`. Не нужны Helm, Operators, etcd, CNI-плагины, отдельная команда DevOps.
- **Минимальные операционные расходы:** Swarm — часть Docker Engine. Никаких отдельных control-plane компонентов как у k8s.
- **Достаточен для 80% web-проектов:** 3–10 нод, десятки сервисов, тысячи RPS — Swarm справляется.

**Где он реально полезен:**
- Небольшие и средние SaaS / e-commerce / B2B-приложения на 3–20 нод.
- On-premise установки клиента, где «привезли пару серверов — поднимите всё сами».
- Edge-инсталляции (медицина, ритейл): один локальный Swarm на филиал.
- Dev/staging-окружения, идентичные prod, но дешевле Kubernetes.

**Где Swarm не лучший выбор:** см. [[#25. Когда не стоит брать Swarm]].

> **Статус проекта.** В 2019 г. Mirantis купила Docker Enterprise и публично подтвердила долгосрочную поддержку Swarm («Swarm is here to stay»). Активная разработка новых features почти остановилась, но баги и CVE в Docker Engine и SwarmKit продолжают исправлять. Для большинства команд это плюс — стабильное API без ломающих изменений каждые полгода.

---

## 2. Архитектура кластера и Raft

Swarm — кластер с **разделением ролей**:

- **Manager** — нода, участвующая в управлении кластером. Хранит состояние, принимает команды (`docker service ...`), выполняет scheduling. Manager одновременно может быть и worker (по умолчанию запускает задачи).
- **Worker** — нода, которая только запускает задачи. Не участвует в Raft, не имеет доступа к секретам, кроме тех, что ей выданы для её задач.
- **Leader** — один из manager-ов, выбранный алгоритмом Raft. Только лидер пишет в Raft-лог; остальные managers — followers, реплицирующие лог.

```mermaid
flowchart LR
    subgraph Managers["Managers (Raft quorum)"]
        L[Leader]
        F1[Follower]
        F2[Follower]
    end
    subgraph Workers
        W1[Worker 1]
        W2[Worker 2]
        W3[Worker 3]
    end
    Client[CLI / docker stack deploy] --> L
    L <-->|Raft log| F1
    L <-->|Raft log| F2
    L -->|task assignment| W1
    L -->|task assignment| W2
    F1 -->|task assignment| W3
```

### Raft и кворум

SwarmKit использует **Raft** (тот же алгоритм, что в etcd, Consul). Чтобы кластер принимал решения, нужен **кворум** — большинство managers. Формула отказоустойчивости:

| Managers | Кворум | Допустимо потерь |
|---|---|---|
| 1 | 1 | 0 |
| 3 | 2 | 1 |
| 5 | 3 | 2 |
| 7 | 4 | 3 |

**Чётное число managers — анти-паттерн** (4 manager-а допускают такую же 1 потерю, как и 3, но повышают вероятность split-brain).

> **Как это понять новичку.** Raft работает по принципу «большинство решает». Если из 3 managers осталось 2 — кластер живой, можно деплоить. Если осталась 1 — нет кворума, кластер становится **read-only**: контейнеры продолжают работать, но `docker service update` зависает. Поэтому 3 manager-а — минимум для production. 5 — для крупных кластеров (≥ 50 нод). Больше 7 не делают: каждый дополнительный manager увеличивает latency Raft (репликация по сети).

### Размещение managers

- **Manager-ы держим на отдельных хостах** в разных зонах доступности (AZ): потеря одной AZ не должна выводить кворум.
- На manager-нодах **не запускаем тяжёлые prod-сервисы** (БД, FPM с 200 worker-ами): `docker node update --availability drain <node>` или `--label-add role=manager` + constraint в сервисах.
- **Не делайте все ноды managers** — это плохо масштабируется (Raft-репликация на каждый чих) и опасно (любая нода может писать в кластер).

### Управляющий и data-plane трафик

- **Control plane** (между managers и workers): порт TCP `2377` — взаимный TLS обязателен.
- **Cluster management** (gossip между всеми нодами): TCP/UDP `7946`.
- **Overlay data plane** (трафик контейнеров через VXLAN): UDP `4789`.

Всё это нужно открыть в firewall между нодами и **закрыть наружу**. Утечка `2377` в интернет = доступ к API кластера.

---

## 3. Модель сервисов, задач и контейнеров

В Swarm есть **3 уровня абстракции**:

```text
service  →  task  →  container
   1     :   N    :    1
```

- **Service** (`docker service create ...`) — декларативное описание: какой образ, сколько реплик, какие порты, секреты, ограничения. Это **желаемое состояние** в Raft.
- **Task** — единица планирования. Manager выбирает ноду, создаёт task в состоянии `pending → assigned → preparing → starting → running → ...`. Task жёстко привязана к ноде и контейнеру.
- **Container** — обычный Docker-контейнер, запущенный workerом по описанию task. Если контейнер упал, task переходит в `failed`/`shutdown`, а manager создаёт **новую** task (а не перезапускает старую).

**Важно:** task **никогда не мигрирует** между нодами. Если нода упала, manager создаёт новые tasks на других нодах. Поэтому нельзя полагаться на «контейнер с тем же ID после рестарта».

### Режимы сервисов

- **Replicated** (по умолчанию): `--replicas N` — кластер поддерживает ровно N task-ов, manager сам решает, где их разместить.
- **Global**: `--mode global` — ровно одна task на каждой ноде, удовлетворяющей constraint-ам. Используется для агентов: `cadvisor`, `node-exporter`, `filebeat`, fluent-bit, traefik (если по одному на ноду).
- **Replicated jobs / Global jobs** (Docker 20.10.0+): one-shot задачи (миграции, бэкапы, batch-обработка). Завершённые tasks не пересоздаются.

```bash
# Долгоживущий сервис
docker service create --name api --replicas 5 myorg/api:1.4.2

# Агент на каждой ноде
docker service create --name node-exporter --mode global \
  --mount type=bind,src=/proc,dst=/host/proc,ro \
  prom/node-exporter

# Одноразовая job — миграция БД
docker service create --name db-migrate --mode replicated-job \
  --replicas 1 --restart-condition none \
  myorg/api:1.4.2 bin/console doctrine:migrations:migrate --no-interaction
```

> **Почему job, а не `docker run`?** Job наследует placement, secrets, configs, network сервиса. `docker run` на одной из нод может не иметь доступа к overlay-сети `app_internal` или к секрету `db_password`, и вы будете долго гадать, почему миграция «локально работает, в проде нет».

---

## 4. Инициализация кластера, joining, lifecycle нод

### Инициализация

```bash
# На будущей первой manager-ноде
docker swarm init --advertise-addr 10.0.1.10

# Вывод даёт join-token для workers
docker swarm join-token worker
docker swarm join-token manager
```

`--advertise-addr` обязателен на хостах с несколькими сетевыми интерфейсами: указывает, какой адрес другие ноды должны видеть. Иначе Swarm может выбрать `eth0` с приватным IP, недоступным извне.

### Join

```bash
# Worker
docker swarm join --token SWMTKN-1-xxx... 10.0.1.10:2377

# Promotion / demotion (изменение роли уже подключённой ноды)
docker node promote node-2
docker node demote node-1
```

### Lifecycle нод

```bash
docker node ls
# ID           HOSTNAME   STATUS    AVAILABILITY   MANAGER STATUS
# abc...       mgr1       Ready     Active         Leader
# def... *     mgr2       Ready     Active         Reachable
# ghi...       wkr1       Ready     Active         -
```

Поля:
- **STATUS**: `Ready` / `Down` — реальная доступность.
- **AVAILABILITY**: `Active` / `Pause` / `Drain`.
- **MANAGER STATUS**: `Leader` / `Reachable` / `Unreachable` (последнее — managers, потерявшие связь с кворумом).

```bash
# Вывести ноду из ротации без удаления (миграция реплик на другие)
docker node update --availability drain wkr1

# Снова в работу
docker node update --availability active wkr1
```

`drain` корректно эвакуирует все task-и: manager создаёт реплики на других нодах **до того как** убьёт старые (если позволяет `update-config`). Это штатный способ обслуживания: апгрейд ядра, замена диска, плановый ребут.

### Метки нод

```bash
docker node update --label-add zone=eu-west-1a wkr1
docker node update --label-add disk=ssd --label-add role=db wkr3
```

Метки — основа scheduling-а. Используйте предметные имена: `zone`, `disk`, `gpu`, `tier=critical`, а не `node1`/`node2`.

> **Ключевая ошибка новичков.** `docker node rm wkr1` без `drain` — мгновенно убивает все task-и на ноде и нарушает SLA. Правильный порядок: `drain` → дождаться, что `docker service ps <service>` показывает все task-и в `Running` на других нодах → `docker swarm leave` на самой ноде → `docker node rm`.

### Резервная копия Raft

State кластера хранится в `/var/lib/docker/swarm/`. Snapshot Raft пишется автоматически. Для бэкапа:

```bash
sudo systemctl stop docker
sudo tar czf swarm-$(date +%F).tgz /var/lib/docker/swarm
sudo systemctl start docker
```

Бэкап нужен в редких сценариях (потеря всех managers, см. [[#20. Резервное копирование и disaster recovery]]). Восстановление: `docker swarm init --force-new-cluster` из распакованного состояния.

---

## 5. Scheduling, constraints, placement preferences

Manager на каждый task вычисляет «подходящую ноду». Алгоритм:

1. Отбрасываем ноды, которые не удовлетворяют **constraints**.
2. Из оставшихся выбираем по **placement preferences** (например, равномерно по `zone`).
3. Среди равных по preferences — выбираем нод с наименьшим числом уже размещённых task-ов этого сервиса (spread по умолчанию).

> **Важно:** Swarm **не** учитывает текущую загрузку CPU/RAM ноды — только зарезервированные ресурсы (`--reserve-cpu`, `--reserve-memory`). Поэтому **всегда** ставьте reservations на тяжёлых сервисах, иначе scheduler может сложить два FPM по 2 ГБ на ноду с 2 ГБ RAM.

### Constraints

```yaml
deploy:
  placement:
    constraints:
      - node.role == worker          # не на managers
      - node.labels.zone == eu-west-1a
      - node.labels.disk == ssd
      - node.platform.arch == x86_64
```

`==` и `!=` — единственные операторы. Регексп / `in` — нет.

Встроенные ключи: `node.id`, `node.hostname`, `node.role` (`manager`/`worker`), `node.platform.os`, `node.platform.arch`, `node.labels.*`, `engine.labels.*`.

> **Разница `node.labels` и `engine.labels`.** `node.labels` ставит admin кластера через `docker node update`, они хранятся в Raft. `engine.labels` задаются в `daemon.json` локально на ноде; их нельзя переписать снаружи. Для обычного scheduling используйте `node.labels`.

### Placement preferences

```yaml
deploy:
  replicas: 6
  placement:
    preferences:
      - spread: node.labels.zone   # по 2 реплики в каждую из 3 зон
```

`spread` — единственный поддерживаемый алгоритм. Если в одной зоне нод нет, реплики уйдут в другие — это **soft constraint**, не жёсткий.

### Resources

```yaml
deploy:
  resources:
    limits:                # жёсткий потолок (cgroups)
      cpus: "1.0"
      memory: 512M
    reservations:          # для scheduler-а
      cpus: "0.25"
      memory: 256M
```

- **`reservations`** — учитывается при scheduling: scheduler не разместит на ноде сервисов больше, чем у неё есть свободных ресурсов в сумме reservations.
- **`limits`** — максимум; превышение по памяти = OOM kill контейнера.

Под PHP-FPM полезно поставить `limits.memory` чуть выше, чем `pm.max_children * memory_limit`, плюс запас на opcache.

---

## 6. Сети Swarm: overlay, ingress, routing mesh

### Драйверы сетей

- **`bridge`** — классическая локальная сеть Docker. В Swarm не для inter-service связи.
- **`overlay`** — VXLAN-сеть поверх UDP/4789, прозрачно объединяющая контейнеры на разных хостах. Это «сетевой бэкбон» Swarm.
- **`ingress`** — специальная overlay-сеть для **routing mesh** (см. ниже). Создаётся автоматически при `swarm init`.
- **`macvlan`** / **`host`** — для специфичных сценариев, обходят routing mesh.

```bash
# Прикладная overlay-сеть с шифрованием data-plane (IPSec)
docker network create -d overlay --opt encrypted --attachable app_internal
```

- `--attachable` нужен, если вы хотите запускать обычные контейнеры (`docker run --network app_internal ...`) на ноде кластера, например в CI или для отладки. Без неё в overlay могут попадать только service-task-и.
- `--opt encrypted` — IPSec поверх VXLAN. Стоит **~10–15% overhead на CPU**, нужен только если нет приватного VPC. Control plane (внутренние команды Swarm) шифруется всегда.

### Service discovery

Каждый сервис получает **VIP** (виртуальный IP) и DNS-имя `<service>` внутри overlay. Имя резолвится **только** в этой сети.

```yaml
services:
  api:
    image: myorg/api:1.4.2
    networks: [app_internal]
  postgres:
    image: postgres:16
    networks: [app_internal]
```

Внутри `api` доступ к БД по `postgres:5432`, а не по IP. Это и есть discovery.

Альтернатива: `endpoint_mode: dnsrr` (DNS round-robin). Тогда `nslookup api` вернёт IP всех task-ов, и клиент сам выбирает. Полезно для **stateful**-протоколов (gRPC long-lived connections, RabbitMQ, Redis Cluster), где VIP-балансировка ломается, потому что L4-балансер прикрепляет соединение к одной task раз и навсегда.

```yaml
deploy:
  endpoint_mode: dnsrr   # vs vip (default)
```

### Routing mesh

```mermaid
flowchart LR
    Client((Internet)) --> LB[External LB / Cloud LB]
    LB --> N1[Node 1\n:80]
    LB --> N2[Node 2\n:80]
    LB --> N3[Node 3\n:80]
    N1 -- ingress --> T1[(api task A)]
    N2 -- ingress --> T2[(api task B)]
    N3 -- ingress --> T1
    N3 -- ingress --> T2
```

При `published: 80, mode: ingress` (default) **каждая нода** слушает порт 80 и форвардит на любую реплику сервиса через ingress overlay. Это удобно: внешний LB может направить трафик на любую ноду, даже на ту, где реплики `api` нет.

Цена:
- Лишний прыжок по сети (могут быть микросекунды или миллисекунды).
- Ingress использует L4 IPVS — **видит src IP не клиента, а ноды**. Чтобы получить реальный IP в логах Nginx, нужен `mode: host` или прокси с PROXY-protocol перед Swarm (Cloudflare → Traefik с `proxyProtocol`).

**`mode: host`** — порт публикуется только на той ноде, где живёт task; внешний LB должен знать, какие именно ноды держат task. Подходит для Traefik как edge-прокси (по одному на ноду + global mode).

```yaml
ports:
  - target: 80
    published: 80
    mode: host    # без routing mesh, source IP сохраняется
```

> **Подводный камень.** Ingress в старых версиях Docker (≤ 19.03) ломался при определённых сценариях rolling update, оставляя «зависшие» NAT-правила в IPVS. На современных движках (24.x+) это исправлено, но если столкнулись — `docker swarm update --ingress-rotate` пересоздаёт ingress overlay.

### Сети сервисов: best practice

- Делите на **публичные** (`traefik_public`) и **приватные** (`app_internal`, `db_internal`) сети.
- БД должна сидеть **только в `db_internal`** + быть доступной только сервису API. Не публикуйте `5432` через ingress.
- Один сервис — несколько сетей (`networks: [traefik_public, app_internal, db_internal]`).

---

## 7. Тома, bind-mounts, плагины хранения

Контейнеры **эфемерны**: после рестарта/перепланировки данные внутри пропадают. Для stateful-сервисов нужны mounts.

### Типы mounts

```yaml
services:
  postgres:
    image: postgres:16
    volumes:
      - pgdata:/var/lib/postgresql/data           # named volume
      - type: bind                                 # bind-mount
        source: /etc/localtime
        target: /etc/localtime
        read_only: true
      - type: tmpfs                                # tmpfs (только в RAM)
        target: /tmp
        tmpfs:
          size: 128M

volumes:
  pgdata:
    driver: local
```

| Тип | Где живёт | Когда брать |
|---|---|---|
| **named volume** (driver `local`) | `/var/lib/docker/volumes/<name>` на ноде | Локальные кеши, тесты, временные данные |
| **bind mount** | произвольный путь хоста | TLS-сертификаты, `/etc/localtime`, монтирование сокетов |
| **tmpfs** | RAM | Секреты в памяти, временные файлы |
| **NFS / CIFS / Cloud** | сетевое хранилище | Stateful-сервисы (см. ниже) |

### Главная ловушка stateful в Swarm

Локальный том существует **только на той ноде**, где он создан. Если `postgres` task переедет на другую ноду — у БД будут пустые данные.

**Решения:**

1. **Pin сервис к ноде** (constraint): `node.labels.role == db`. БД всегда на одной и той же ноде; данные на её локальном SSD/NVMe.
2. **Сетевой volume:** NFS, GlusterFS, Ceph RBD, AWS EFS, Azure Files. Прозрачно доступен с любой ноды. Цена — latency (сеть вместо локального SSD), особенно болезненно для БД.
3. **Не запускать stateful в Swarm.** БД, очереди — managed-сервис (RDS, ElastiCache, CloudAMQP). Это часто самое дешёвое решение по TCO.

Пример NFS volume:

```yaml
volumes:
  shared_uploads:
    driver: local
    driver_opts:
      type: nfs
      o: addr=10.0.0.5,nfsvers=4.1,rw
      device: ":/exports/uploads"
```

> **Боль из практики.** Видел кластер, где `postgres` со `--replicas 2` и default placement превратил данные в фарш: один task поднял БД, при rolling update второй task встал на другой ноде, увидел пустой volume, инициализировал новый кластер PG — и у второй реплики записи начали разъезжаться. Защита: `replicas: 1` + строгий placement constraint + `update_config.order: stop-first` (см. [[#10. Rolling updates, rollback, zero-downtime деплой]]).

### Volumes у сервиса со многими репликами

В Swarm **нет** концепции `PersistentVolumeClaim` как в k8s: каждая реплика получает **свой** named volume на своей ноде. Если 5 реплик `cache` смонтировали том `cache_data` — это 5 независимых томов на 5 нодах с одинаковым именем. Это редко то, что нужно.

Используйте volumes для stateful только при `replicas: 1` или замените на сетевое хранилище.

---

## 8. Secrets и configs

### Secrets

Swarm secrets — зашифрованные значения в Raft, доступные task-ам как файлы в `/run/secrets/<name>` (tmpfs, не на диске!).

```bash
echo -n 's3cret-pass' | docker secret create db_password -
docker secret create tls_cert ./certs/server.crt
docker secret ls
```

```yaml
services:
  api:
    image: myorg/api:1.4.2
    secrets:
      - source: db_password
        target: db_password
        mode: 0400
      - tls_cert
    environment:
      DB_PASSWORD_FILE: /run/secrets/db_password

secrets:
  db_password:
    external: true   # уже создан вне stack
  tls_cert:
    external: true
```

**Почему через файл, а не env?**
- Env переменные видны в `docker inspect` любому, у кого есть доступ к Docker daemon на ноде, в `/proc/<pid>/environ`, в крэш-дампах.
- Файл монтируется в tmpfs и виден **только** контейнеру с этим секретом.
- Symfony умеет читать `_FILE`-суффикс: достаточно прокси-переменной, см. [[#13. Production-стенд для Symfony 6.4 (Nginx + PHP-FPM)]].

### Configs

Configs — то же самое, но для **не-секретных** данных (Nginx-конфиги, php.ini, fail2ban-правила). Тоже в Raft, тоже монтируются файлом, но **не шифруются** на диске managers и видны в `docker config inspect`.

```bash
docker config create nginx_app_conf ./nginx/app.conf
```

```yaml
services:
  nginx:
    image: nginx:1.27-alpine
    configs:
      - source: nginx_app_conf
        target: /etc/nginx/conf.d/app.conf
configs:
  nginx_app_conf:
    file: ./nginx/app.conf
```

### Иммутабельность и ротация

Secret/config **нельзя обновить**: попытка `docker secret rm db_password` выдаст ошибку, если он используется. Правильный паттерн — версионирование:

```yaml
secrets:
  db_password_v3:
    external: true
services:
  api:
    secrets:
      - source: db_password_v3
        target: db_password
```

При смене секрета:
1. `docker secret create db_password_v3 -` со значением.
2. Обновить stack-файл (`v2 → v3`).
3. `docker stack deploy ...` — Swarm пересоздаст task-и с новым секретом.
4. После успеха `docker secret rm db_password_v2`.

Этот же приём — для TLS-сертификатов, ключей API, JWT-secret-ов: rolling-ротация без даунтайма.

> **Анти-паттерн.** Класть секреты в env через `.env`-файл, коммитить в git, передавать `-e DB_PASS=...` на CLI. В лучшем случае они утекут в логи CI, в худшем — в crash dump или systemd journal.

---

## 9. Stack-файлы и docker compose v3

Stack — набор сервисов, описанный в YAML и развёрнутый одной командой. Swarm понимает **Compose Specification** (унифицированный преемник «v3»), но игнорирует/отрицает часть полей, специфичных для одиночного `docker compose` (например, `build`, `depends_on` с `condition: service_healthy`, `container_name`).

```yaml
# docker-stack.yml
version: "3.9"

x-php-deploy: &php-deploy           # YAML-якоря для DRY
  update_config:
    parallelism: 1
    order: start-first
    failure_action: rollback
    monitor: 30s
  rollback_config:
    parallelism: 1
    order: stop-first
  restart_policy:
    condition: any
    delay: 5s

services:
  api:
    image: registry.example.com/myorg/api:${API_VERSION:-1.4.2}
    networks: [traefik_public, app_internal]
    secrets: [db_password, app_secret]
    configs:
      - source: php_ini_v4
        target: /usr/local/etc/php/conf.d/zz-app.ini
    environment:
      APP_ENV: prod
      DATABASE_URL: postgresql://app:${POSTGRES_PASSWORD}@postgres:5432/app?serverVersion=16
    deploy:
      <<: *php-deploy
      replicas: 4
      resources:
        limits: { cpus: "1.0", memory: 512M }
        reservations: { cpus: "0.25", memory: 256M }
      placement:
        constraints: [node.role == worker]
        preferences: [spread: node.labels.zone]
      labels:
        - traefik.enable=true
        - traefik.http.routers.api.rule=Host(`api.example.com`)
        - traefik.http.routers.api.entrypoints=websecure
        - traefik.http.routers.api.tls.certresolver=le
        - traefik.http.services.api.loadbalancer.server.port=8080

networks:
  traefik_public: { external: true }
  app_internal:   { driver: overlay, attachable: false }

secrets:
  db_password: { external: true }
  app_secret:  { external: true }

configs:
  php_ini_v4:
    file: ./config/php.ini
```

```bash
docker stack deploy -c docker-stack.yml app --with-registry-auth
```

`--with-registry-auth` передаёт workers credentials для приватного registry — иначе нода без `docker login` не сможет вытянуть образ.

### `deploy` vs `compose`-поля

В Swarm всё, что относится к runtime-кластеру (replicas, restart, placement), **должно** быть в `deploy:`. Поля верхнего уровня (`restart`, `cpus`, `mem_limit`) — не работают в swarm-режиме и тихо игнорируются. Это частая ошибка при копи-паст из обычного compose.

### Что не работает в Swarm

- `build:` — Swarm не собирает образы, только запускает. Сборка — в CI.
- `depends_on:` — стартует service A до B, **но** не ждёт `healthy`. Полагайтесь на retry-логику в приложении (см. [[#11. Healthchecks, restart policy, update policy]]).
- `container_name:` — имена контейнеров генерируются Swarm (`<stack>_<service>.<task-slot>.<task-id>`).
- `restart:` (verbose) — заменён `deploy.restart_policy`.

---

## 10. Rolling updates, rollback, zero-downtime деплой

### Алгоритм rolling update

При `docker service update` или `docker stack deploy` с изменённой конфигурацией Swarm:

1. Берёт `parallelism` task-ов (по умолчанию 1).
2. Применяет `order` (`stop-first` или `start-first`).
3. Ждёт `monitor` (по умолчанию 5 сек) — следит за состоянием новой task.
4. Если новая task не падает в этот период — берёт следующую партию.
5. Если падает или возвращает `unhealthy` — `failure_action`: `pause` (default), `continue`, `rollback`.

```yaml
deploy:
  replicas: 6
  update_config:
    parallelism: 2          # обновляем по 2 task-а
    delay: 10s              # пауза между партиями
    order: start-first      # сначала поднять новый, потом убить старый
    failure_action: rollback
    monitor: 30s            # период наблюдения healthcheck
    max_failure_ratio: 0.2  # ≤ 20% задач могут упасть, прежде чем считаем deploy провальным
  rollback_config:
    parallelism: 1
    order: stop-first
    failure_action: pause
```

### `start-first` vs `stop-first`

- **`start-first`** — на короткое время живут и старая, и новая реплики. Нужен для **stateless** сервисов (HTTP API за балансером): zero-downtime, нет потери трафика.
- **`stop-first`** (default) — сначала убили, потом подняли. Деплой кратковременно теряет 1/N мощности. **Обязателен для stateful** (БД c одной репликой, экземпляр Redis с фиксированным volume) — иначе две реплики одновременно держат один port или один volume.

> **Тонкость.** `start-first` требует, чтобы новая реплика встала **в healthy** до того, как Swarm убьёт старую. Без healthcheck Swarm считает task healthy через ~10 секунд после `running`, и вы получаете «5xx во время деплоя из-за непрогретого FPM/opcache». Поэтому **healthcheck обязателен** для zero-downtime.

### Rollback

```bash
docker service update --rollback api
docker stack deploy -c docker-stack.yml app   # повторный deploy с предыдущей версией тоже работает
```

`--rollback` возвращает service в **предыдущую** версию (Swarm хранит одну предыдущую spec). После rollback кнопки «два шага назад» нет — это не git.

`failure_action: rollback` — автоматический rollback при сбое деплоя. Стоит включать для production: лучше 2 минуты на старой версии, чем 5xx во время разбирательства.

### Канареечный деплой

В Swarm нет встроенного traffic splitting (10% на новую версию). Реализуется через два сервиса + Traefik weight:

```yaml
services:
  api-stable:
    image: myorg/api:1.4.2
    deploy:
      replicas: 9
      labels:
        - traefik.http.services.api.loadbalancer.server.port=8080
        - traefik.http.services.api.loadbalancer.weighted.services.0.name=api-stable
        - traefik.http.services.api.loadbalancer.weighted.services.0.weight=9
  api-canary:
    image: myorg/api:1.4.3-rc1
    deploy:
      replicas: 1
```

(Точный синтаксис зависит от версии Traefik; начиная с v2 поддерживаются weighted services.)

---

## 11. Healthchecks, restart policy, update policy

### Healthcheck

Без healthcheck Swarm видит task как «здоровую» сразу после `docker run`, что ломает rolling update и self-healing.

```yaml
services:
  api:
    image: myorg/api:1.4.2
    healthcheck:
      test: ["CMD", "curl", "-fsS", "http://127.0.0.1:8080/health"]
      interval: 10s
      timeout: 3s
      retries: 3
      start_period: 30s     # дать FPM/opcache прогреться
```

`start_period` — окно, в течение которого падающие проверки **не считаются** провалом. Критично для PHP: без него 30 секунд после старта вы будете получать `unhealthy` пока FPM собирает воркеры и opcache.

В контейнере должен быть `curl` или `wget`. Если не хочется тащить — используйте `php -r 'exit(file_get_contents("http://127.0.0.1:8080/health") ? 0 : 1);'` или собственный бинарник `/usr/local/bin/healthcheck`.

### Endpoint `/health`

В Symfony делают так:

```php
// src/Controller/HealthController.php
declare(strict_types=1);

namespace App\Controller;

use Doctrine\DBAL\Connection;
use Symfony\Component\HttpFoundation\JsonResponse;
use Symfony\Component\Routing\Attribute\Route;

final class HealthController
{
    public function __construct(
        private readonly Connection $db,
    ) {
    }

    /**
     * Liveness — отвечаем 200 всегда, пока процесс жив.
     * Используется Swarm-ом как "container alive".
     */
    #[Route('/health', name: 'health_live', methods: ['GET'])]
    public function live(): JsonResponse
    {
        return new JsonResponse(['status' => 'ok'], 200);
    }

    /**
     * Readiness — проверяем зависимости (БД, кэш).
     * Если 503 — балансер уберёт ноду из ротации.
     * Используется Traefik / внешним LB, НЕ Swarm-ом для рестартов.
     */
    #[Route('/ready', name: 'health_ready', methods: ['GET'])]
    public function ready(): JsonResponse
    {
        try {
            $this->db->executeQuery('SELECT 1');
        } catch (\Throwable) {
            return new JsonResponse(['status' => 'db_unreachable'], 503);
        }

        return new JsonResponse(['status' => 'ok'], 200);
    }
}
```

> **Почему два endpoint-а.** Если БД мигнула на 5 секунд, а у вас один healthcheck с проверкой БД, Swarm убьёт **все** task-и API одновременно (самовызванный crash). Liveness-чек должен говорить «процесс жив», а readiness — «готов принимать трафик». Liveness привязан к Swarm `healthcheck`, readiness — к балансеру (Traefik умеет проверять отдельный URL).

### Restart policy

```yaml
deploy:
  restart_policy:
    condition: any         # on-failure | none | any (default any)
    delay: 5s              # задержка перед рестартом
    max_attempts: 0        # 0 = без лимита
    window: 120s           # окно, в котором max_attempts учитывается
```

Без `delay` упавший контейнер с багом стартапа порождает CPU-storm: тысяча рестартов в секунду. `delay: 5s` + `max_attempts: 5` спасают от этого.

---

## 12. Масштабирование и распределение нагрузки

### Горизонтальное масштабирование сервиса

```bash
docker service scale app_api=10
# или через stack с обновлённым deploy.replicas: 10 + docker stack deploy
```

### Авто-масштабирование

В отличие от k8s (HPA), в Swarm **нет встроенного auto-scaling**. Варианты:

1. **Внешние решения:** `orbiter`, `swarm-autoscaler`, кастомный скрипт на Prometheus + `docker service scale`. Все сторонние, поддержка вяло.
2. **Ручной scaling по расписанию:** cron, который перед пиком вызывает `docker service scale`.
3. **Заложенный запас:** держим 2x replicas от пиковой нагрузки. Дёшево и предсказуемо.

Для большинства SaaS-проектов п.3 — лучший вариант: latency предсказуема, никаких холодных стартов FPM в момент пика.

### Балансировка между репликами

- **VIP (default)**: Linux IPVS на каждой ноде — round-robin / least-conn по task-ам сервиса.
- **DNS round-robin** (`endpoint_mode: dnsrr`): клиент сам резолвит и выбирает.
- **Внешний L7 (Traefik / Nginx / HAProxy)**: видит каждый task отдельно, умеет sticky session, weighted routing, rate limiting. **Production-выбор для HTTP.**

> **Sticky sessions в Swarm.** Routing mesh их не делает. Если приложение использует session в файлах (`session.save_handler = files`), а реплик ≥ 2, без sticky-balancing-а пользователь теряет сессию между запросами. Решение: Redis/Memcached для session (`framework.session.handler_id: 'redis://session:6379'`) — stateless approach, сразу работает в любом scheduling-е.

---

## 13. Production-стенд для Symfony 6.4 (Nginx + PHP-FPM)

Минимальный production-стенд: Nginx (статика, gzip, TLS), PHP-FPM (Symfony), PostgreSQL (managed или отдельная нода), Redis (кэш + session + Messenger transport). Образы собираем многоступенчатым `Dockerfile`.

### Dockerfile (multi-stage)

```dockerfile
# syntax=docker/dockerfile:1.7
# ---------- Stage 1: composer ----------
FROM composer:2.7 AS composer

# ---------- Stage 2: build ----------
FROM php:8.2-fpm-alpine AS build

RUN apk add --no-cache git icu-dev oniguruma-dev libzip-dev postgresql-dev linux-headers \
 && docker-php-ext-install -j"$(nproc)" intl opcache pdo_pgsql zip bcmath \
 && pecl install redis-6.0.2 apcu-5.1.23 \
 && docker-php-ext-enable redis apcu \
 && rm -rf /tmp/* /var/cache/apk/*

COPY --from=composer /usr/bin/composer /usr/bin/composer
WORKDIR /app

# Сначала только composer.* — чтобы кешировался слой с vendor
COPY composer.json composer.lock symfony.lock ./
RUN --mount=type=cache,target=/root/.composer/cache \
    composer install --no-dev --no-scripts --no-autoloader --prefer-dist --no-interaction

COPY . .
RUN composer dump-autoload --classmap-authoritative --no-dev \
 && composer run-script --no-dev post-install-cmd

# ---------- Stage 3: production ----------
FROM php:8.2-fpm-alpine AS prod

RUN apk add --no-cache icu-libs oniguruma libzip libpq tzdata \
 && rm -rf /var/cache/apk/*

# Копируем расширения и конфиги PHP из build-стадии
COPY --from=build /usr/local/lib/php/extensions/ /usr/local/lib/php/extensions/
COPY --from=build /usr/local/etc/php/conf.d/ /usr/local/etc/php/conf.d/
COPY docker/php/zz-prod.ini /usr/local/etc/php/conf.d/zz-prod.ini
COPY docker/php-fpm/www.conf /usr/local/etc/php-fpm.d/www.conf

WORKDIR /app
COPY --from=build --chown=www-data:www-data /app /app

# Прогрев кешей в составе образа — ускоряет первый запрос после деплоя
RUN php bin/console cache:warmup --env=prod --no-debug

USER www-data
EXPOSE 9000
HEALTHCHECK --interval=10s --timeout=3s --start-period=30s --retries=3 \
  CMD php-fpm-healthcheck || exit 1

CMD ["php-fpm", "--nodaemonize"]
```

### `docker/php/zz-prod.ini`

```ini
memory_limit = 256M
opcache.enable = 1
opcache.memory_consumption = 256
opcache.max_accelerated_files = 20000
opcache.validate_timestamps = 0          ; в проде код не меняется без redeploy
opcache.preload = /app/config/preload.php
opcache.preload_user = www-data
realpath_cache_size = 4096K
realpath_cache_ttl = 600
expose_php = Off
date.timezone = UTC
```

`opcache.validate_timestamps = 0` — обязательное условие для производительного prod, но требует **rebuild + redeploy на любое изменение PHP-кода**. В контейнерной модели это норма.

### Чтение секретов через `_FILE`-переменные

Symfony 6.4 умеет автоматически: `DATABASE_URL_FILE=/run/secrets/database_url`. В коде используется обычная `$_ENV['DATABASE_URL']`. Но если приложение собственное, добавьте обёртку:

```php
// public/index.php или config/bootstrap.php
declare(strict_types=1);

(function (): void {
    foreach ($_ENV as $key => $value) {
        if (str_ends_with($key, '_FILE') && is_string($value) && is_readable($value)) {
            $envName = substr($key, 0, -5);
            $_ENV[$envName] = $_SERVER[$envName] = trim((string) file_get_contents($value));
            unset($_ENV[$key], $_SERVER[$key]);
        }
    }
})();
```

### Stack-файл

```yaml
version: "3.9"

services:
  nginx:
    image: registry.example.com/myorg/nginx:1.4.2
    networks: [traefik_public, app_internal]
    deploy:
      replicas: 4
      update_config:
        order: start-first
        parallelism: 1
        failure_action: rollback
      placement:
        preferences: [spread: node.labels.zone]
      labels:
        - traefik.enable=true
        - traefik.http.routers.app.rule=Host(`app.example.com`)
        - traefik.http.routers.app.entrypoints=websecure
        - traefik.http.routers.app.tls.certresolver=le
        - traefik.http.services.app.loadbalancer.server.port=80
        - traefik.http.services.app.loadbalancer.healthcheck.path=/health
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://127.0.0.1/health"]
      interval: 10s
      timeout: 3s
      retries: 3
      start_period: 10s

  php:
    image: registry.example.com/myorg/php:1.4.2
    networks: [app_internal]
    secrets:
      - source: db_password_v3
        target: db_password
      - source: app_secret_v2
        target: app_secret
    environment:
      APP_ENV: prod
      APP_DEBUG: "0"
      DATABASE_URL_FILE: /run/secrets/db_password
      REDIS_DSN: redis://redis:6379/0
      MESSENGER_TRANSPORT_DSN: redis://redis:6379/messages
    deploy:
      replicas: 4
      update_config:
        order: start-first
        parallelism: 1
        failure_action: rollback
        monitor: 30s
      resources:
        limits: { cpus: "1.0", memory: 512M }
        reservations: { cpus: "0.25", memory: 256M }
      placement:
        constraints: [node.role == worker]
        preferences: [spread: node.labels.zone]

  redis:
    image: redis:7.2-alpine
    command: ["redis-server", "--appendonly", "yes", "--maxmemory", "256mb", "--maxmemory-policy", "allkeys-lru"]
    networks: [app_internal]
    volumes:
      - redis_data:/data
    deploy:
      replicas: 1
      placement:
        constraints: [node.labels.role == cache]
      update_config:
        order: stop-first

networks:
  traefik_public: { external: true }
  app_internal:   { driver: overlay }

volumes:
  redis_data:

secrets:
  db_password_v3: { external: true }
  app_secret_v2:  { external: true }
```

> **Тонкости.** Nginx и PHP — два разных сервиса, разные образы, разные replicas. Связь между ними — через overlay по DNS (`fastcgi_pass php:9000`). Каждая Nginx-task балансирует к VIP `php` — а Swarm раскладывает на конкретный FPM. Это даёт независимое масштабирование (например, 4 Nginx + 8 PHP).

### Конфиг Nginx → FPM

```nginx
# nginx/app.conf
upstream php_upstream {
    server php:9000;        # DNS Swarm разрезолвит в VIP сервиса
    keepalive 16;
}

server {
    listen 80;
    server_name _;
    root /app/public;
    index index.php;

    client_max_body_size 20m;

    location /health {
        access_log off;
        return 200 'ok';
        add_header Content-Type text/plain;
    }

    location / {
        try_files $uri /index.php$is_args$args;
    }

    location ~ ^/index\.php(/|$) {
        fastcgi_pass php_upstream;
        fastcgi_split_path_info ^(.+\.php)(/.*)$;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        fastcgi_param DOCUMENT_ROOT $realpath_root;
        fastcgi_param HTTPS on;             # за TLS-терминатором
        fastcgi_keep_conn on;
        internal;
    }

    location ~ \.php$ {
        return 404;     # никаких скриптов кроме index.php
    }
}
```

> **Анти-паттерн.** Класть Nginx и PHP-FPM в один контейнер (через supervisord или s6-overlay). Это единый failure domain, нельзя независимо масштабировать, плохо профилируется. В Swarm/k8s-эпохе это табу.

---

## 14. Symfony Messenger workers в Swarm

Messenger workers — это **долгоживущие CLI-процессы**, которые читают очередь и выполняют handlers. В Swarm они оформляются как обычные сервисы с `replicas: N`.

### Сервис worker-а

```yaml
services:
  messenger-async:
    image: registry.example.com/myorg/php:1.4.2
    command: >
      bin/console messenger:consume async failed
        --time-limit=3600
        --memory-limit=192M
        --limit=1000
        --no-interaction
    networks: [app_internal]
    secrets: [db_password_v3, app_secret_v2]
    environment:
      APP_ENV: prod
      MESSENGER_TRANSPORT_DSN: redis://redis:6379/messages
    deploy:
      replicas: 6
      update_config:
        order: start-first
        parallelism: 2
        failure_action: rollback
        monitor: 30s
      restart_policy:
        condition: any
        delay: 5s
      resources:
        limits: { memory: 256M }
        reservations: { memory: 128M }
    healthcheck:
      # worker — это процесс, проверяем по pid-файлу или lock
      test: ["CMD", "pgrep", "-f", "messenger:consume"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 20s
    stop_grace_period: 60s    # дать worker-у завершить текущее сообщение
    stop_signal: SIGTERM
```

### Почему `--time-limit` и `--memory-limit`

Symfony Messenger по дизайну предполагает, что worker умирает «как только что-то выглядит подозрительно»: накопил мусор, исчерпал лимит времени, обработал N сообщений. После этого Swarm рестартует контейнер. Это идеальный союз:

- Меньше утечек памяти — частые рестарты.
- Можно деплоить новую версию: старые worker-ы доедают свои `--limit` сообщений и плавно сменяются (см. `update_config.order: start-first`).
- Не нужен `supervisord` внутри контейнера — Swarm и есть супервайзер.

### Graceful shutdown

При деплое Swarm посылает `SIGTERM`, ждёт `stop_grace_period` (default 10 сек, маловато для Messenger), потом `SIGKILL`. Symfony Messenger корректно обрабатывает `SIGTERM`: завершает текущее сообщение, сохраняет ack, выходит. Если ваш handler делает 30-секундный API-вызов — увеличивайте `stop_grace_period` до 60–120 секунд.

```php
// Кастомизация: явно подписаться на сигналы (Symfony 6.4 уже делает это сам)
declare(strict_types=1);

namespace App\Messenger;

use Symfony\Component\Console\SignalRegistry\SignalRegistry;
// Symfony 6.4 регистрирует SIGINT/SIGTERM в Application::run автоматически
// и пробрасывает в Worker через SignalableCommandInterface
```

### Несколько типов worker-ов

```yaml
services:
  messenger-async:
    command: bin/console messenger:consume async --time-limit=3600 --memory-limit=192M
    deploy: { replicas: 6 }

  messenger-priority:
    command: bin/console messenger:consume priority --time-limit=3600 --memory-limit=192M
    deploy: { replicas: 2 }

  messenger-failed:
    # Авто-рестарт failed-сообщений раз в 5 минут (cron-подход — см. ниже)
    command: bin/console messenger:failed:retry --force --time-limit=60
    deploy:
      replicas: 1
      restart_policy: { condition: any, delay: 300s }
```

> **Почему не `supervisord`.** Swarm уже выполняет роль супервайзера: рестарт по политике, логи в stdout, ресурсы через cgroups. Двойной супервайзер скрывает реальные крашы (контейнер «жив», а внутри supervisord перезапускает worker в цикле). Один процесс на контейнер — каноничный путь.

---

## 15. Миграции, cron и одноразовые задачи

### Миграции БД

Запускать `doctrine:migrations:migrate` в `entrypoint`/`CMD` основного контейнера — анти-паттерн: **N реплик одновременно стартанут N конкурентных миграций**. Doctrine ставит advisory lock в PG, но сам факт N процессов, ожидающих lock, замедлит деплой.

**Правильный путь — отдельный one-shot job ПЕРЕД деплоем приложения:**

```yaml
services:
  api:
    image: registry.example.com/myorg/php:1.4.2
    # ...

  migrate:
    image: registry.example.com/myorg/php:1.4.2
    command: bin/console doctrine:migrations:migrate --no-interaction --allow-no-migration
    networks: [app_internal]
    secrets: [db_password_v3]
    environment:
      APP_ENV: prod
    deploy:
      mode: replicated-job
      replicas: 1
      restart_policy:
        condition: on-failure
        max_attempts: 3
      placement:
        constraints: [node.role == manager]
```

Деплой такой:

```bash
# В CI:
docker stack deploy -c docker-stack-migrate.yml app-migrate --with-registry-auth
docker service ls --filter name=app-migrate_migrate
# дождаться, что job завершилась успешно (replicas 0/1 после завершения)
docker stack rm app-migrate

# Потом основной stack:
docker stack deploy -c docker-stack.yml app --with-registry-auth
```

Или используем helper-скрипт, который ждёт завершения job. Пример проверки в bash:

```bash
#!/usr/bin/env bash
set -euo pipefail
docker service create --name app-migrate \
    --network app_internal \
    --secret db_password_v3 \
    --restart-condition none \
    --constraint node.role==manager \
    --mode replicated-job --replicas 1 \
    "registry.example.com/myorg/php:${API_VERSION}" \
    bin/console doctrine:migrations:migrate --no-interaction

# Ждём завершения
while :; do
    state="$(docker service ps app-migrate --format '{{.CurrentState}}' --no-trunc | head -n1)"
    case "$state" in
        Complete*) echo "Migration done"; break ;;
        Failed*|Rejected*) echo "Migration failed: $state" >&2; docker service logs app-migrate; exit 1 ;;
        *) sleep 2 ;;
    esac
done
docker service rm app-migrate
```

### Backward-compatible миграции

В rolling-update-кластере **2 версии приложения работают одновременно ~1–5 минут**. Поэтому миграции пишем по правилу expand–contract:

1. **Expand**: добавляем колонку/таблицу/индекс. Старый код не знает о ней, новый умеет писать.
2. **Deploy** новой версии — оба кода умеют читать старую структуру.
3. **Contract**: после полного выкатывания удаляем старое поле/таблицу.

Никогда не делайте `DROP COLUMN` в той же миграции, в которой деплоится новая версия кода.

### Cron-задачи

В Swarm нет встроенного `CronJob` (как в k8s). Варианты:

1. **Сервис-«шедулер»** с `replicas: 1` и `command` запускающим `cron` или `supercronic` (cron, который пишет в stdout):

```yaml
services:
  scheduler:
    image: registry.example.com/myorg/php:1.4.2
    command: ["supercronic", "/app/docker/crontab"]
    networks: [app_internal]
    secrets: [db_password_v3]
    deploy:
      replicas: 1
      placement:
        constraints: [node.labels.role == scheduler]
      restart_policy: { condition: any, delay: 10s }
```

`crontab`:
```
*/5 * * * * php /app/bin/console app:reports:rebuild --env=prod
0 3 * * *  php /app/bin/console app:cleanup:expired --env=prod
```

`replicas: 1` критично: иначе каждый task выполнит cron, и вы 5 раз пересчитаете отчёт.

2. **Symfony Scheduler** (Symfony 6.4+ из коробки) — стартует как обычный Messenger-worker:

```bash
bin/console messenger:consume scheduler_default
```

Это нативный Symfony-подход, рекомендую начиная с 6.4.

### Резервное копирование

Bash-скрипт раз в сутки на manager-ноде через системный cron + `docker exec` или service-job, который запускает `pg_dump`/`borg`/`restic` в storage (S3, NFS).

---

## 16. Reverse proxy и TLS: Traefik в Swarm

Traefik — стандарт де-факто edge-прокси для Swarm. Он подписывается на Docker socket, видит сервисы кластера через labels и автоматически перестраивает свою маршрутизацию. Плюс — нативная поддержка Let's Encrypt.

### Stack `traefik`

```yaml
version: "3.9"

services:
  traefik:
    image: traefik:v3.1
    command:
      - --providers.swarm=true
      - --providers.swarm.exposedbydefault=false
      - --providers.swarm.network=traefik_public
      - --entrypoints.web.address=:80
      - --entrypoints.web.http.redirections.entrypoint.to=websecure
      - --entrypoints.web.http.redirections.entrypoint.scheme=https
      - --entrypoints.websecure.address=:443
      - --certificatesresolvers.le.acme.email=ops@example.com
      - --certificatesresolvers.le.acme.storage=/letsencrypt/acme.json
      - --certificatesresolvers.le.acme.tlschallenge=true
      - --metrics.prometheus=true
      - --metrics.prometheus.entrypoint=metrics
      - --entrypoints.metrics.address=:8082
      - --accesslog=true
      - --log.level=INFO
    ports:
      - target: 80
        published: 80
        mode: host        # сохраняем real IP клиента
      - target: 443
        published: 443
        mode: host
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro    # обязательно на manager
      - traefik_certs:/letsencrypt
    networks: [traefik_public]
    deploy:
      mode: global                 # по одному на каждой ноде с labels.edge=true
      placement:
        constraints: [node.role == manager, node.labels.edge == true]
      update_config:
        parallelism: 1
        order: start-first

networks:
  traefik_public:
    external: true

volumes:
  traefik_certs:
    driver: local
```

> **Почему `mode: host`.** Через ingress Traefik видит src-IP как IP ноды, и Let's Encrypt TLS-ALPN challenge ломается с балансером перед Swarm. `mode: host` отключает routing mesh для этого порта: на каждой edge-ноде Traefik слушает 80/443 «голым» сокетом. Внешний LB (или DNS A-records) указывает на эти ноды.
>
> **Почему на managers.** Только manager имеет доступ к Docker API всего кластера через socket. Альтернатива — `tcp://docker-socket-proxy:2375` с ограниченным `containers: 1, services: 1, networks: 1`. Это безопаснее: socket — root-эквивалент.

### TLS challenge: HTTP-01 vs DNS-01

- **TLS-ALPN-01** (рекомендация) — Traefik сам отвечает на 443 challenge, не нужно открывать 80.
- **HTTP-01** — нужен 80 порт.
- **DNS-01** — для wildcard-сертификатов (`*.example.com`). Traefik пишет DNS-запись через провайдера (Cloudflare, Route53). Подробно — в доке Traefik по `dnsChallenge`.

Хранилище `acme.json` обязательно с правами 600 и `replicas: 1` для Traefik (иначе несколько нод одновременно делают challenge — Let's Encrypt ratelimit). С `mode: global` нужно либо shared volume (NFS), либо использовать Traefik в **HA-режиме** через Redis/etcd backend для сертификатов:

```yaml
- --certificatesresolvers.le.acme.storage=traefik-redis
# ... плюс провайдер kv (Redis/Consul)
```

---

## 17. CI/CD: сборка образов и `docker stack deploy`

### Конвейер

```mermaid
flowchart LR
    Push[git push] --> Build[Build image]
    Build --> Test[Run unit/integration tests]
    Test --> Scan[Trivy/Grype scan]
    Scan --> Push2[Push to registry]
    Push2 --> Migrate[Run migration job]
    Migrate --> Deploy[docker stack deploy]
    Deploy --> Smoke[Smoke tests]
    Smoke -->|fail| Rollback
    Smoke -->|ok| Done[Notify Slack]
```

### GitHub Actions

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v4

      - uses: docker/setup-buildx-action@v3

      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - uses: docker/build-push-action@v5
        with:
          context: .
          file: docker/php/Dockerfile
          push: true
          target: prod
          tags: |
            ghcr.io/myorg/php:${{ github.sha }}
            ghcr.io/myorg/php:latest
          cache-from: type=registry,ref=ghcr.io/myorg/php:buildcache
          cache-to:   type=registry,ref=ghcr.io/myorg/php:buildcache,mode=max

      - name: Trivy scan
        uses: aquasecurity/trivy-action@0.20.0
        with:
          image-ref: ghcr.io/myorg/php:${{ github.sha }}
          severity: HIGH,CRITICAL
          exit-code: 1

  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4

      - name: Setup SSH
        run: |
          install -m 700 -d ~/.ssh
          echo "${{ secrets.SWARM_DEPLOY_KEY }}" > ~/.ssh/id_ed25519
          chmod 600 ~/.ssh/id_ed25519
          ssh-keyscan -H ${{ secrets.SWARM_HOST }} >> ~/.ssh/known_hosts

      - name: Deploy stack
        env:
          DOCKER_HOST: ssh://deploy@${{ secrets.SWARM_HOST }}
          API_VERSION: ${{ github.sha }}
        run: |
          # 1) Миграции через one-shot job
          ./scripts/run-migration.sh "$API_VERSION"

          # 2) Основной деплой
          docker stack deploy \
            -c docker-stack.yml \
            --with-registry-auth \
            --prune \
            app

          # 3) Ждём, что rolling update прошёл
          ./scripts/wait-for-deploy.sh app_php
          ./scripts/wait-for-deploy.sh app_nginx

      - name: Smoke test
        run: curl -fsS https://app.example.com/health
```

### `wait-for-deploy.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail
service="$1"
deadline=$(( $(date +%s) + 300 ))    # 5 минут

while :; do
    json="$(docker service inspect --format '{{json .UpdateStatus}}' "$service")"
    state="$(echo "$json" | jq -r '.State // "completed"')"
    case "$state" in
        completed|"") echo "[$service] update completed"; exit 0 ;;
        rollback_completed) echo "[$service] rolled back" >&2; exit 1 ;;
        paused|rollback_paused) echo "[$service] update paused: $json" >&2; exit 1 ;;
    esac
    [[ $(date +%s) -lt $deadline ]] || { echo "[$service] timeout" >&2; exit 1; }
    sleep 5
done
```

### `DOCKER_HOST=ssh://...`

Это ключ к простому и безопасному удалённому деплою: локальный CLI запускается на runner, но команды выполняются через SSH на manager. Нет открытого Docker API наружу, нет TLS-сертификатов на каждом runner — только SSH-ключ.

### Анти-паттерн: `docker save | ssh ... docker load`

Тащить tar-архивы образов в CI — медленно, забивает диски, ломает digest verification. Используйте registry. Если нет внешнего — поднимите `registry:2` внутри Swarm.

---

## 18. Логирование и мониторинг (Prometheus, Loki, Grafana)

### Логи

Docker по умолчанию пишет stdout/stderr контейнеров в `json-file` driver на ноде. В кластере это плохо: логи разбросаны, ротация пилит файлы прямо на хосте, `docker service logs <svc>` стримит из всех нод, но не индексируется.

**Каноничный стек: Loki + Promtail + Grafana**.

```yaml
services:
  promtail:
    image: grafana/promtail:3.0.0
    command: -config.file=/etc/promtail/config.yaml
    networks: [observability]
    volumes:
      - /var/log:/var/log:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
    configs:
      - source: promtail_cfg_v3
        target: /etc/promtail/config.yaml
    deploy:
      mode: global         # один агент на каждой ноде
      resources:
        limits: { memory: 256M }

  loki:
    image: grafana/loki:3.0.0
    networks: [observability]
    volumes: [loki_data:/loki]
    deploy:
      replicas: 1
      placement:
        constraints: [node.labels.role == observability]

  grafana:
    image: grafana/grafana:11.1.0
    networks: [observability, traefik_public]
    secrets:
      - source: grafana_admin_password_v1
        target: grafana_admin_password
    environment:
      GF_SECURITY_ADMIN_PASSWORD__FILE: /run/secrets/grafana_admin_password
    volumes: [grafana_data:/var/lib/grafana]
    deploy:
      replicas: 1
      labels:
        - traefik.enable=true
        - traefik.http.routers.grafana.rule=Host(`grafana.example.com`)
        - traefik.http.routers.grafana.entrypoints=websecure
        - traefik.http.services.grafana.loadbalancer.server.port=3000
```

`promtail` глобально, читает контейнерные логи через docker socket, добавляет labels (`service`, `stack`, `node`) и шлёт в Loki. Поиск в Grafana: `{service="app_php"} |= "Exception"`.

### Метрики

Prometheus + node-exporter + cadvisor + (опционально) `dockerd metrics endpoint`:

```json
// /etc/docker/daemon.json на каждой ноде
{
  "metrics-addr": "0.0.0.0:9323",
  "experimental": true
}
```

Это даёт метрики самого Docker daemon (engine_daemon_*, swarm_*).

`cadvisor` (mode global) собирает метрики каждого контейнера: cpu, memory, network, fs.

### Symfony-метрики

Подключите `promphp/prometheus_client_php` или `artprima/prometheus-metrics-bundle`:

```php
// src/EventSubscriber/HttpMetricsSubscriber.php
declare(strict_types=1);

namespace App\EventSubscriber;

use Prometheus\CollectorRegistry;
use Symfony\Component\EventDispatcher\Attribute\AsEventListener;
use Symfony\Component\HttpKernel\Event\RequestEvent;
use Symfony\Component\HttpKernel\Event\TerminateEvent;
use Symfony\Component\HttpKernel\KernelEvents;

final class HttpMetricsSubscriber
{
    public function __construct(
        private readonly CollectorRegistry $registry,
    ) {
    }

    #[AsEventListener(event: KernelEvents::REQUEST, priority: 4096)]
    public function onRequest(RequestEvent $event): void
    {
        $event->getRequest()->attributes->set('_metric_started_at', hrtime(true));
    }

    #[AsEventListener(event: KernelEvents::TERMINATE)]
    public function onTerminate(TerminateEvent $event): void
    {
        $req = $event->getRequest();
        $startedAt = $req->attributes->get('_metric_started_at');
        if (!\is_int($startedAt)) {
            return;
        }
        $durationSec = (hrtime(true) - $startedAt) / 1e9;

        $histogram = $this->registry->getOrRegisterHistogram(
            'app',
            'http_request_duration_seconds',
            'HTTP request duration',
            ['method', 'route', 'status'],
            [0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0],
        );
        $histogram->observe($durationSec, [
            $req->getMethod(),
            (string) $req->attributes->get('_route', 'unknown'),
            (string) $event->getResponse()->getStatusCode(),
        ]);
    }
}
```

> **Почему histogram, а не gauge.** Для latency нужен квантиль (p50, p95, p99). Histogram-buckets позволяют агрегировать по нескольким репликам через `histogram_quantile()`, gauge — нет (только последнее значение).

В Swarm scrape-config Prometheus указывает на `dnsrr` сервис: каждая task светит свои метрики, Prometheus их обнаруживает через DNS SD:

```yaml
scrape_configs:
  - job_name: 'symfony-app'
    dns_sd_configs:
      - names: ['tasks.app_php']     # tasks.<service> резолвится во все task-IP
        type: A
        port: 8080
```

### Алертинг

Базовый набор алертов на Swarm:
- `swarm_node_state != "ready"` дольше 2 минут → нода упала.
- `desired_replicas - running_replicas > 0` → сервис недоукомплектован.
- p95 latency `/health` > 1 сек → проблема с сетью или scheduler-ом.
- Контейнер OOMKilled > 3 раз / час → утечка памяти, либо неправильный `limits.memory`.

---

## 19. Безопасность кластера и образов

### Кластер

- **Tokens.** Не публикуйте join-token в git. Это ключ от кластера. Регулярно ротируйте: `docker swarm join-token --rotate worker`.
- **Firewall.** Порты `2377`, `7946`, `4789` открыты **только** между нодами (и не наружу). Используйте security groups / iptables.
- **mTLS.** Включён по умолчанию. CA рутируется автоматически каждые 90 дней. `--cert-expiry` можно настроить, но обычно не нужно.
- **Docker socket.** Не монтируйте `/var/run/docker.sock` в произвольные сервисы. Это полный root на хост (escape тривиален). Если контейнеру нужен ограниченный доступ (Traefik, Portainer, watchtower) — проксируйте через `tecnativa/docker-socket-proxy` с минимальными правами.
- **Не запускайте контейнеры от root.** В `Dockerfile` — `USER www-data`. В compose — `user: "1000:1000"`.
- **`read_only: true`** для контейнеров без необходимости писать на диск (большинство web-сервисов):
  ```yaml
  services:
    php:
      read_only: true
      tmpfs:
        - /tmp
        - /var/cache/symfony
  ```
- **`cap_drop: [ALL]`** + добавлять только нужное (обычно `NET_BIND_SERVICE` для < 1024 портов, но FPM слушает 9000 — не нужно).
- **`security_opt: [no-new-privileges:true]`** — запрет setuid-эскалации внутри контейнера.

### Образы

- **Минимальный базовый образ:** `php:8.2-fpm-alpine` или distroless для статичных бинарей. Меньше CVE.
- **Pin tag-ов:** не `php:8.2-fpm-alpine`, а `php:8.2.21-fpm-alpine3.20`. Ещё лучше — pin по digest: `php@sha256:abc...`.
- **Сканируйте:** Trivy / Grype в CI, gating на CRITICAL.
- **Подписи:** Cosign (sigstore) на образы; верификация на pull через Notation/cosign-policy.
- **Не клади секреты в `ENV`/`ARG`.** `ARG` остаются в истории слоёв, видимы через `docker history`.

### Network policies

Swarm не имеет network policies уровня k8s NetworkPolicy. Защита — на уровне сетей: разные overlay для разных trust-zone. БД должна быть в `db_internal`, в которой только `api`. Не добавляйте Redis/postgres в `traefik_public`.

---

## 20. Резервное копирование и disaster recovery

### Что бэкапить

1. **Состояние Swarm:** `/var/lib/docker/swarm/` на одной из manager-нод. Это Raft-лог + ключи CA. Без него можно потерять кластер при одновременной смерти всех managers.
2. **Volumes сервисов:** БД, uploads, Redis (если нужна durability). Метод зависит от приложения: `pg_dump` для PG, snapshot LVM/ZFS, `restic`/`borg` для filesystem.
3. **Stack-файлы и `.env`:** в git. Это ваш «source of truth» инфраструктуры. Без них нечего восстанавливать.
4. **Образы:** в registry. Если registry упадёт, восстановите образы из cache CI или собрав заново из тегов git.

### Сценарии восстановления

**Все managers потеряны, workers живы:**
1. На одной из выживших нод (или новой): `docker swarm leave --force` → восстановить `/var/lib/docker/swarm/` из бэкапа → `systemctl start docker` → `docker swarm init --force-new-cluster --advertise-addr <ip>`.
2. `--force-new-cluster` сохраняет state, но удаляет всех старых managers (становится 1 manager). Workers переподключатся.
3. Поднять остальных managers: `docker node promote ...`, `docker swarm join-token manager`.

**Полный disaster (все ноды потеряны):**
1. Поднять новые ноды.
2. `docker swarm init` на первой.
3. `docker stack deploy -c docker-stack.yml app`.
4. Восстановить данные volumes из бэкапа.
5. Проверить миграции (могут быть несовместимости версий).

> **Drill.** DR-план без репетиции — фантазия. Раз в квартал поднимайте «копию production» из бэкапов в изолированной сети, прогоняйте smoke-тесты. Реальные инциденты вскрывают проблемы с правами на `acme.json`, забытыми внешними секретами, переименованными overlay-сетями.

---

## 21. Тестирование инфраструктуры и приложения

### Тесты приложения (Symfony)

Сами по себе Swarm-специфичные нюансы тестируются редко. Уровни тестов:

- **Unit** — без Swarm, обычный PHPUnit.
- **Functional** (`WebTestCase`) — стартует Symfony Kernel, делает HTTP-вызовы in-process. Тоже не требует Swarm.
- **Integration с реальной БД/Redis** — поднимаем зависимости через `docker compose` (не Swarm) в CI:

```yaml
# docker-compose.test.yml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_PASSWORD: test
      POSTGRES_DB: app_test
    tmpfs: [/var/lib/postgresql/data]   # быстрее на CI
  redis:
    image: redis:7-alpine
```

```bash
docker compose -f docker-compose.test.yml up -d --wait
APP_ENV=test bin/phpunit
```

### End-to-end на реальном Swarm

Stage-окружение — отдельный Swarm-кластер (1 manager + 2 workers). Деплой того же stack-файла, что и prod, но с переменной `APP_ENV=staging`. После деплоя — Cypress/Playwright/k6 тесты.

### Тестирование healthcheck-ов

Локально:
```bash
docker run --rm -d --name api-test myorg/api:1.4.2
sleep 30
docker inspect --format '{{.State.Health.Status}}' api-test
# должен быть "healthy"
```

### Chaos-тесты

Проверяем устойчивость кластера:
```bash
# Убить случайный task сервиса
task=$(docker service ps --format '{{.ID}}' app_api | shuf -n1)
docker service ps app_api | grep "$task"
# Свалить ноду — drain + restart
docker node update --availability drain wkr2
sleep 30
docker node update --availability active wkr2
# Проверить, что все task-и снова Running и сервис отвечает
curl -fsS https://app.example.com/health
```

Из инструментов: `pumba` (Chaos Engineering для Docker), `chaos-mesh` (для k8s, не Swarm).

---

## 22. Типичные ошибки и анти-паттерны

| Ошибка | Почему плохо | Как правильно |
|---|---|---|
| Чётное число managers (2, 4) | Не повышает отказоустойчивость, риск split-brain | 1 / 3 / 5 / 7 |
| Все ноды — managers | Raft latency, лишний attack surface | 3 managers + N workers |
| `docker run` на ноде «как костыль» | Контейнер не виден кластеру, не учитывается scheduler | `docker service create` или service-job |
| `docker node rm` без `drain` | Резкое убийство всех task-ов на ноде | `drain` → дождаться → `rm` |
| Stateful (БД) с `replicas > 1` без сетевого storage | Данные расходятся, corruption | `replicas: 1` + pinned constraint, или managed DB |
| Секреты через env | Видны в `inspect`, в `/proc/<pid>/environ`, в логах | `secrets:` + `_FILE` |
| `update_config.order: stop-first` для stateless API | Кратковременный downtime | `start-first` + healthcheck |
| Healthcheck без `start_period` | Падающий деплой пока FPM/opcache прогревается | `start_period: 30s` |
| Один контейнер с Nginx+FPM+supervisord | Нельзя независимо масштабировать, плохо отлаживается | Раздельные сервисы |
| Миграции в `entrypoint` | N реплик гонятся за advisory lock, замедляя деплой | Отдельный `replicated-job` |
| `SELECT *` логов, `docker service logs --tail=all` | Грузит ноду, забивает stdout | Loki/Promtail + индекс |
| Ingress для БД портов | Утечка БД наружу, IPVS меняет src IP | Внутренний overlay, без `published` |
| `--no-cache` всегда в CI | Медленные сборки (5–15 мин) | BuildKit `--mount=type=cache` |
| `restart: always` в `deploy` | Игнорируется в Swarm-режиме | `deploy.restart_policy.condition: any` |
| Pin образов к `latest` | Откат непредсказуем, сборка не воспроизводится | Pin к git-sha / semver / digest |
| Игнор `stop_grace_period` для воркеров | Messenger обрывает обработку → дубли/потери | 30–120 с + `--limit` в воркере |
| Локальные volumes на разных нодах с одним именем | Данные «прыгают» при rescheduling | Pinned placement или сетевое хранилище |
| Открытые `2377/tcp` наружу | Публичный API кластера | Firewall, VPN, private network |

---

## 23. Troubleshooting: диагностика на живом кластере

### Сервис не запускается

```bash
# Куда планировал, какое сейчас состояние
docker service ps app_api --no-trunc

# Колонки CURRENT STATE / ERROR — главное
# Running 5 minutes      — норм
# Failed 2 minutes ago   — упал, в ERROR причина
# Pending 10 seconds     — нет ноды под constraints
# Rejected 1 minute ago  — нет подходящей ноды (ресурсы/labels)
```

`Pending` чаще всего: нет ноды, удовлетворяющей `constraints` или нет свободных reservations.

### Образ не пуллится

```bash
docker service ps app_api --no-trunc | grep -i 'no such image\|denied\|manifest unknown'
# Решение: 
#   - неправильный тег → правка stack
#   - private registry → docker stack deploy --with-registry-auth
#   - на ноде нет логина → docker login на каждой или auth.json в /etc/docker/auth/
```

### Не видно сервис изнутри другого

```bash
# Внутри контейнера:
nslookup api
getent hosts api
nc -zv api 8080

# Проверить, что оба сервиса в одной overlay
docker network inspect app_internal --format '{{range .Containers}}{{.Name}} {{.IPv4Address}}{{"\n"}}{{end}}'
```

Если DNS не резолвится — обычно сервис не в указанной сети, либо забыт `attachable: true` для overlay.

### Memory leak в FPM

```bash
docker stats --no-stream
# Видим php-task с растущей RSS

# Внутри контейнера:
docker exec -it <container> sh -c 'cat /proc/1/status | grep -E "VmRSS|VmPeak"'
```

В Swarm обычно решают радикально: `pm.max_requests = 200` в FPM (worker умирает после N запросов и пересоздаётся), плюс `limits.memory` ловит самые жирные утечки через OOM + restart.

### Долгий rolling update

```bash
docker service inspect --format '{{json .UpdateStatus}}' app_api | jq
# state: paused/rollback_paused — failure_action сработал, читать .Message
```

Частая причина: новый образ `unhealthy` → Swarm не идёт дальше → задеплоилась 1 task, остальное на старом. `failure_action: rollback` автоматически вернёт.

### Раскоп Raft проблем

```bash
docker node ls
# Если managers видны как Unreachable — потеря кворума
journalctl -u docker -n 200 --no-pager | grep -i raft
```

При loss-quorum: `docker swarm init --force-new-cluster` на оставшемся manager (см. [[#20. Резервное копирование и disaster recovery]]).

### Что копать первым делом

1. `docker node ls` — все ноды Ready?
2. `docker service ls` — replicas совпадают с desired?
3. `docker service ps <svc> --no-trunc` — таски на месте, без Failed?
4. `docker service logs <svc> --since 5m --tail 200` — приложенческие ошибки.
5. `journalctl -u docker -n 200` на проблемной ноде — daemon-уровень.

---

## 24. Сравнение с Kubernetes, Nomad и docker compose

| Критерий | Docker Swarm | Kubernetes | HashiCorp Nomad | docker compose |
|---|---|---|---|---|
| Установка | `swarm init` (5 минут) | kubeadm/k3s/managed (часы–дни) | `nomad agent` | `docker compose up` |
| Кривая обучения | низкая (тот же compose) | высокая (десятки CRD) | средняя (HCL, отдельная философия) | минимальная |
| API объём | ~30 команд | сотни типов ресурсов | ~20 команд | ~10 команд |
| Multi-host | да | да | да | нет (один хост) |
| Rolling update | встроен | встроен (Deployment) | встроен (job update) | вручную |
| Auto-scaling | нет (внешние) | HPA/VPA из коробки | autoscaler-плагины | нет |
| Network policies | нет | NetworkPolicy + CNI | Consul Connect | нет |
| Storage classes | нет (volumes per-node) | PVC/CSI | CSI | volumes |
| Service mesh | нет | Istio/Linkerd | Consul Connect | нет |
| Стейтфул | сложно (ручной pinning) | StatefulSet, operators | да | да |
| Workloads ≠ Docker | нет | OCI/CRI любой | Java/exec/raw_exec/qemu/docker | только Docker |
| Multi-cluster | плохо | federation/argo-cd | да | нет |
| Ecosystem | малый | огромный | средний | большой |
| Стоимость владения | низкая | высокая | средняя | минимальная |
| Когда брать | 3–20 нод, web-stack, малая команда | 50+ нод, микросервисы, специальная команда | смешанные workload-ы (Java + контейнеры) | разработка, single-host prod |

**Kubernetes** — индустриальный стандарт, но за счёт сложности. На команде в 2–3 backend без выделенного DevOps Kubernetes съедает больше времени, чем приносит пользы.

**Nomad** — недооценённая альтернатива: быстрый, простой, поддерживает не только Docker. Но экосистема меньше, и в России меньше специалистов.

**docker compose** на одной prod-ноде — рабочий вариант для маленьких проектов (≤ 1000 RPS). Минусы: нет HA, нет rolling update без скриптов, нет native secrets. Когда вырастаете — миграция в Swarm требует только правки compose-файла (`deploy:` секции добавить).

---

## 25. Когда не стоит брать Swarm

- Нужен **auto-scaling** из коробки (резкие пики, edge cases). В Swarm только костыли.
- Нужны **сложные сетевые политики** (zero-trust, mesh, mTLS между сервисами). k8s + Istio/Linkerd.
- **Гетерогенные workload-ы**: контейнеры + VM + JVM + standalone. Nomad.
- **Большой кластер** (> 50 нод, > 1000 task-ов). Swarm работает, но Raft latency и tooling страдают.
- **Команда уже умеет** k8s. Не делайте «шаг назад» ради экономии — операционный риск перевесит.
- **Нужны** Operators / CRD / Helm / GitOps (ArgoCD/Flux). Это всё ecosystem k8s.
- **Сертификация требует k8s** (CIS, PCI DSS audit бывает заточен под k8s tooling).
- **SaaS-провайдер хостит вам кластер** — почти всегда это managed k8s (EKS/GKE/AKS), Swarm-сервиса нет.

Если нет ни одного из этих критериев — Swarm даст вам 90% функционала за 10% сложности.

---

## 26. Проверочные вопросы с ответами

> [!question]- Что такое Docker Swarm и чем он отличается от обычного `docker compose`?
> Swarm mode — встроенный в Docker Engine оркестратор для группы хостов. Превращает кластер в единый виртуальный «движок»: декларативные сервисы, scheduling, mTLS, secrets, rolling updates. `docker compose` работает с одним хостом и не предоставляет HA, балансировку трафика по нодам, secrets как объекты, rolling. Stack-файл Swarm — почти то же самое, что compose, плюс блок `deploy:`.
>
> 🔗 [[#1. Что такое Docker Swarm и зачем он нужен]]

> [!question]- Сколько manager-ов нужно в production-кластере и почему?
> Минимум 3, чтобы пережить падение одного manager-а; при особо критичных нагрузках — 5 (переживёт 2 одновременных). Чётные числа (2, 4) бессмысленны: «допустимое число потерь» такое же, как у меньшего нечётного, но риск split-brain выше. Все managers должны жить в разных AZ/стойках. Больше 7 — рост Raft latency.
>
> 🔗 [[#2. Архитектура кластера и Raft]]

> [!question]- Что такое кворум Raft и что произойдёт при его потере?
> Кворум — большинство managers (например, 2 из 3). Только при кворуме лидер может писать в Raft-лог: создавать сервисы, менять конфигурацию, выдавать секреты. Без кворума кластер становится **read-only**: запущенные task-и продолжают работать, но любые `docker service ...` зависают. Восстановление — `docker swarm init --force-new-cluster` на одном из выживших managers.
>
> 🔗 [[#2. Архитектура кластера и Raft]] · [[#20. Резервное копирование и disaster recovery]]

> [!question]- Объясни связь service → task → container.
> Сервис — декларация желаемого состояния («хочу 5 реплик образа X»). Task — единица планирования: manager закрепляет каждую за конкретной нодой. Container — фактически запущенный Docker-контейнер по описанию task. Task **не мигрирует**: упала нода — manager создаёт **новые** task-и на других нодах. Идентификатор контейнера после рестарта другой.
>
> 🔗 [[#3. Модель сервисов, задач и контейнеров]]

> [!question]- В чём разница между `replicated`, `global` и `replicated-job` сервисами?
> `replicated` — N task-ов на кластер, manager сам выбирает ноды. `global` — ровно одна task на каждой подходящей ноде (агенты: node-exporter, promtail). `replicated-job` / `global-job` — одноразовые задачи (миграции, бэкапы): после успешного завершения task не пересоздаётся.
>
> 🔗 [[#3. Модель сервисов, задач и контейнеров]] · [[#15. Миграции, cron и одноразовые задачи]]

> [!question]- Чем отличаются constraints и placement preferences?
> Constraints — **жёсткие** условия. Если ни одна нода не подходит, task встаёт в `Pending`. Preferences — **мягкие**: алгоритм `spread` старается равномерно разложить реплики по значениям метки (например, по зонам), но при нехватке нод в зоне отдаст реплику в другую.
>
> 🔗 [[#5. Scheduling, constraints, placement preferences]]

> [!question]- Почему важно задавать `resources.reservations`, а не только `limits`?
> Scheduler учитывает только reservations при расчёте свободных ресурсов ноды. Без них он не знает «реальной» загрузки и может разместить два прожорливых сервиса на одной ноде, после чего OOM или CPU-троттлинг убьют обоих. `limits` — это уже cgroups-потолок исполнения.
>
> 🔗 [[#5. Scheduling, constraints, placement preferences]]

> [!question]- Что такое routing mesh и его подводные камни?
> Каждая нода кластера слушает опубликованный порт сервиса и проксирует запрос (через ingress overlay + IPVS) к любой реплике. Это упрощает внешний LB — можно бить в любую ноду. Подводные камни: src IP «теряется» (видно IP ноды, не клиента), L4-балансировка не учитывает application-метрики, лишний прыжок по сети. Для edge-прокси с Let's Encrypt и сохранения IP — `mode: host`.
>
> 🔗 [[#6. Сети Swarm: overlay, ingress, routing mesh]]

> [!question]- Когда выбрать `endpoint_mode: dnsrr` вместо VIP?
> Когда клиент держит **долгие соединения** (gRPC, WebSocket, AMQP-каналы, Redis-pub/sub), и L4 VIP-балансировка прикрепит его к одной реплике на всё время сессии — нагрузка не равномерна, и при росте реплик новые останутся без трафика. DNS-RR отдаёт клиенту все IP, и клиент-балансер сам решает.
>
> 🔗 [[#6. Сети Swarm: overlay, ingress, routing mesh]]

> [!question]- Как правильно работать со stateful-сервисами в Swarm?
> Три стратегии: 1) `replicas: 1` + строгий placement constraint к конкретной ноде с локальным SSD/NVMe; 2) сетевые volumes (NFS, EFS, Ceph) — терпимая latency, но прозрачное перепланирование; 3) вынести БД из Swarm в managed-сервис. Множество реплик стейтфул-сервиса с разными локальными volumes даёт corruption.
>
> 🔗 [[#7. Тома, bind-mounts, плагины хранения]]

> [!question]- Чем отличаются Swarm `secrets` от `configs` и почему не использовать env?
> Оба — иммутабельные объекты в Raft, монтируются файлами в tmpfs контейнера. Secrets зашифрованы on-disk на managers, configs — нет. Env-переменные видны в `docker inspect`, в `/proc/<pid>/environ`, могут утечь в crash-дамп и логи. Symfony умеет читать `_FILE`-суффикс, поэтому DB_PASSWORD_FILE=/run/secrets/db_password — стандартный путь.
>
> 🔗 [[#8. Secrets и configs]] · [[#13. Production-стенд для Symfony 6.4 (Nginx + PHP-FPM)]]

> [!question]- Как ротировать секрет без даунтайма?
> Секреты иммутабельны. Паттерн: создать `secret_v2`, обновить stack-файл (target имя контейнера прежнее, source — новый), `docker stack deploy` — Swarm пересоздаст task-и rolling update-ом со `start-first`. После полного выкатывания удалить `secret_v1`.
>
> 🔗 [[#8. Secrets и configs]]

> [!question]- Что делать, если в Swarm-режиме `depends_on` с `condition: service_healthy` не работает?
> Swarm запускает сервисы независимо и не ждёт healthy. Решение — отказоустойчивость в самом приложении: retry-цикл при подключении к БД (`Connection refused` ловится → задержка → повтор). Doctrine + Symfony Messenger такой паттерн поддерживают.
>
> 🔗 [[#9. Stack-файлы и docker compose v3]] · [[#11. Healthchecks, restart policy, update policy]]

> [!question]- В чём разница между `update_config.order: start-first` и `stop-first`?
> `start-first`: новая реплика поднимается, дожидается healthy, потом убивается старая. Нужен для stateless web-API — даёт zero-downtime. `stop-first`: сначала убили, потом подняли. Обязателен для stateful с `replicas: 1` и для сервисов, где две одновременные реплики ломают invariant (один порт, один volume, один внешний lock).
>
> 🔗 [[#10. Rolling updates, rollback, zero-downtime деплой]]

> [!question]- Зачем нужен `start_period` в healthcheck и каким его делать для PHP-FPM?
> Это окно, в течение которого падающие проверки **не считаются** провалом. Без него PHP-FPM с прогревом opcache (`cache:warmup`, preload) и подключением к БД считается `unhealthy`, и Swarm убивает task до её реального старта. Стандарт — 20–60 секунд, измерять реально на cold-start.
>
> 🔗 [[#11. Healthchecks, restart policy, update policy]]

> [!question]- Зачем разделять liveness и readiness endpoint-ы?
> Liveness (`/health`) отвечает «процесс жив», без проверки внешних зависимостей. Привязан к Swarm `healthcheck` — его падение приведёт к рестарту контейнера. Readiness (`/ready`) проверяет БД/кеш — если они недоступны, балансер выводит реплику из ротации, **не убивая** контейнер. Иначе кратковременный сбой БД вызовет каскадный рестарт всех реплик и полный outage.
>
> 🔗 [[#11. Healthchecks, restart policy, update policy]]

> [!question]- Как правильно деплоить миграции БД в кластере?
> Отдельный `replicated-job` сервис, запускаемый **до** обновления приложения: `docker service create --mode replicated-job ... bin/console doctrine:migrations:migrate`. CI ждёт завершения job, затем `docker stack deploy` основного приложения. Сами миграции пишем backward-compatible (expand-contract): добавляем поля без `DROP` в той же миграции.
>
> 🔗 [[#15. Миграции, cron и одноразовые задачи]]

> [!question]- Как запускать Symfony Messenger workers в Swarm?
> Обычный сервис с `command: bin/console messenger:consume ...`, `replicas: N`, `--time-limit` и `--memory-limit` для самоубийства, `restart_policy.condition: any`, `stop_grace_period: 60s` для корректной обработки `SIGTERM`. Без `supervisord` — Swarm сам супервайзер.
>
> 🔗 [[#14. Symfony Messenger workers в Swarm]]

> [!question]- Что такое cron в Swarm и как его сделать?
> Встроенного `CronJob` нет. Варианты: 1) сервис с `replicas: 1` и supercronic (cron, пишущий в stdout); 2) Symfony Scheduler 6.4+ как обычный messenger:consume worker. `replicas: 1` критично — иначе таски выполнятся N раз.
>
> 🔗 [[#15. Миграции, cron и одноразовые задачи]]

> [!question]- Почему Traefik в Swarm обычно ставят с `mode: global` и `mode: host` для портов?
> `global` — по одному инстансу на каждой edge-ноде. `mode: host` для 80/443 — обходит ingress routing mesh, чтобы Let's Encrypt TLS-ALPN-01 challenge работал, а в логах был реальный IP клиента. Хранилище `acme.json` либо на shared-volume, либо в KV-backend (Redis/Consul) для HA.
>
> 🔗 [[#16. Reverse proxy и TLS: Traefik в Swarm]]

> [!question]- Безопасно ли монтировать `/var/run/docker.sock` в Traefik / Portainer?
> С точки зрения безопасности это эквивалент root-а на хосте: можно создать привилегированный контейнер, записать на хост-FS и т.д. В малом кластере — оправданный риск (Traefik только на manager-нодах, ограниченных по доступу). В среднем/крупном — `tecnativa/docker-socket-proxy` с разрешением только нужных эндпоинтов (services R, containers R).
>
> 🔗 [[#16. Reverse proxy и TLS: Traefik в Swarm]] · [[#19. Безопасность кластера и образов]]

> [!question]- Как реализовать CI/CD в Swarm безопасно (без открытого Docker API)?
> `DOCKER_HOST=ssh://deploy@manager` — CLI ходит к manager-ноде по SSH, выполняет `docker stack deploy` локально на ней. Никаких открытых TLS-сокетов, только SSH-ключ. Образы — в registry, аутентификация — `--with-registry-auth`.
>
> 🔗 [[#17. CI/CD: сборка образов и `docker stack deploy`]]

> [!question]- Как собирать метрики Symfony из всех реплик в Swarm?
> Prometheus DNS SD: `dns_sd_configs.names: ['tasks.app_php']` резолвится в IP **всех** task-ов (а не VIP), Prometheus скрейпит каждого. На стороне Symfony — histogram-метрики (`http_request_duration_seconds`), агрегация в Grafana через `histogram_quantile()`.
>
> 🔗 [[#18. Логирование и мониторинг (Prometheus, Loki, Grafana)]]

> [!question]- Почему histogram, а не gauge для latency?
> Gauge показывает «последнее значение» — нельзя усреднить квантили нескольких реплик. Histogram содержит counts по bucket-ам, и `histogram_quantile()` корректно вычисляет p50/p95/p99 по сумме всех реплик.
>
> 🔗 [[#18. Логирование и мониторинг (Prometheus, Loki, Grafana)]]

> [!question]- Как правильно ротировать join-token Swarm?
> `docker swarm join-token --rotate worker` (или `manager`). Старые ноды остаются в кластере (они уже подключены), но новый join возможен только с новым токеном. Делают при увольнении админа, утечке секрета, либо планово раз в полгода-год.
>
> 🔗 [[#19. Безопасность кластера и образов]]

> [!question]- Как восстановить кластер после потери всех managers?
> На одной из выживших нод (или новой) восстановить `/var/lib/docker/swarm/` из бэкапа, запустить `docker swarm init --force-new-cluster --advertise-addr <ip>`. Кластер вернётся с одним manager, остальных promote-ить заново. Поэтому `/var/lib/docker/swarm` должен бэкапиться независимо от volumes приложения.
>
> 🔗 [[#20. Резервное копирование и disaster recovery]]

> [!question]- Главные анти-паттерны Swarm?
> Чётное число managers, все ноды managers, секреты в env, миграции в `entrypoint`, Nginx+FPM в одном контейнере, `replicas > 1` на stateful с локальным volume, healthcheck без `start_period`, `update_config.order: stop-first` для stateless API, открытый `2377/tcp` наружу, pin образов на `latest`.
>
> 🔗 [[#22. Типичные ошибки и анти-паттерны]]

> [!question]- Когда Swarm — правильный выбор, а когда нет?
> **Брать**: 3–20 нод, web-stack (PHP/Node/Go), малая команда без выделенного DevOps, on-premise, edge-кейсы, dev/staging-окружение. **Не брать**: нужен auto-scaling, network policies / service mesh, гетерогенные workload-ы, > 50 нод, GitOps/Operators экосистема, команда уже умеет k8s.
>
> 🔗 [[#24. Сравнение с Kubernetes, Nomad и docker compose]] · [[#25. Когда не стоит брать Swarm]]

> [!question]- Как отлаживать сервис, застрявший в `Pending`?
> `docker service ps <svc> --no-trunc` покажет причину в `ERROR`. Чаще всего: нет ноды под `placement.constraints` (опечатка в label, нет нод нужной зоны), или нет свободных `reservations` (сумма по существующим task-ам уже занимает всю ноду). Решения: расширить constraints, добавить ноды, уменьшить reservations.
>
> 🔗 [[#23. Troubleshooting: диагностика на живом кластере]]

> [!question]- Почему `restart: always` в `deploy:` не работает?
> В swarm-режиме верхнеуровневые `restart`, `cpus`, `mem_limit`, `container_name`, `build` тихо игнорируются. Их аналоги — внутри `deploy:`: `restart_policy.condition: any`, `resources.limits.cpus`, и т.д. Это частая ошибка при копировании из обычного `docker compose`.
>
> 🔗 [[#9. Stack-файлы и docker compose v3]] · [[#22. Типичные ошибки и анти-паттерны]]

> [!question]- Как сделать канареечный деплой в Swarm?
> Встроенного traffic splitting нет. Реализуется через два сервиса (`api-stable` 9 реплик, `api-canary` 1 реплика) и Traefik weighted services. После успешного наблюдения за canary — увеличиваем replicas canary, уменьшаем stable, деплоим обновление stable.
>
> �� [[#10. Rolling updates, rollback, zero-downtime деплой]]

> [!question]- Что такое expand-contract миграция и почему она важна именно в кластере?
> При rolling update **2 версии приложения работают одновременно** несколько минут. Если миграция в той же выкатке делает `DROP COLUMN` или `ALTER TABLE` несовместимо со старым кодом — старые реплики начнут падать с SQL-ошибками. Expand: добавили колонку (oба кода живут). Contract: удалили старое поле в **следующем** релизе.
>
> 🔗 [[#15. Миграции, cron и одноразовые задачи]]

---

## 27. Источники

### Официальная документация Docker / SwarmKit
- Swarm mode overview: https://docs.docker.com/engine/swarm/
- Glossary (service, task, node): https://docs.docker.com/engine/swarm/key-concepts/
- Раздел Raft в Swarm: https://docs.docker.com/engine/swarm/raft/
- `docker service create` reference: https://docs.docker.com/reference/cli/docker/service/create/
- `docker stack deploy`: https://docs.docker.com/reference/cli/docker/stack/deploy/
- Compose Specification: https://github.com/compose-spec/compose-spec/blob/master/spec.md
- `deploy` ключевое слово (Compose Specification): https://docs.docker.com/reference/compose-file/deploy/
- Swarm secrets: https://docs.docker.com/engine/swarm/secrets/
- Swarm configs: https://docs.docker.com/engine/swarm/configs/
- Networking: https://docs.docker.com/engine/swarm/networking/
- Routing mesh: https://docs.docker.com/engine/swarm/ingress/
- Healthcheck (Dockerfile): https://docs.docker.com/reference/dockerfile/#healthcheck
- Update / rollback policies: https://docs.docker.com/engine/swarm/swarm-tutorial/rolling-update/
- Manage nodes: https://docs.docker.com/engine/swarm/manage-nodes/
- Restore Swarm cluster: https://docs.docker.com/engine/swarm/admin_guide/#back-up-the-swarm
- SwarmKit (исходник, design docs): https://github.com/moby/swarmkit
- Raft (Diego Ongaro, Stanford, оригинальная статья): https://raft.github.io/raft.pdf

### Mirantis / поддержка Swarm
- Mirantis: «Swarm is here to stay»: https://www.mirantis.com/blog/mirantis-will-continue-to-support-and-develop-docker-swarm/

### Symfony / PHP
- Symfony 6.4 Doctrine: https://symfony.com/doc/6.4/doctrine.html
- Symfony 6.4 Messenger: https://symfony.com/doc/6.4/messenger.html
- Symfony 6.4 Scheduler: https://symfony.com/doc/6.4/scheduler.html
- Symfony 6.4 Configuring Environments / Secrets vault: https://symfony.com/doc/6.4/configuration/secrets.html
- Doctrine Migrations: https://www.doctrine-project.org/projects/doctrine-migrations/en/3.8/
- PHP 8.2 release notes: https://www.php.net/releases/8.2/en.php
- PHP-FPM tuning: https://www.php.net/manual/en/install.fpm.configuration.php
- OPcache preload: https://www.php.net/manual/en/opcache.preloading.php

### Edge / Reverse proxy
- Traefik v3 docs: https://doc.traefik.io/traefik/
- Traefik Swarm provider: https://doc.traefik.io/traefik/providers/swarm/
- Let's Encrypt с Traefik: https://doc.traefik.io/traefik/https/acme/

### Мониторинг / логирование
- Prometheus: https://prometheus.io/docs/introduction/overview/
- Prometheus DNS SD: https://prometheus.io/docs/prometheus/latest/configuration/configuration/#dns_sd_config
- Loki: https://grafana.com/docs/loki/latest/
- Promtail (Docker discovery): https://grafana.com/docs/loki/latest/send-data/promtail/configuration/#docker_sd_configs
- Grafana: https://grafana.com/docs/grafana/latest/
- cAdvisor: https://github.com/google/cadvisor
- node_exporter: https://github.com/prometheus/node_exporter
- Docker daemon Prometheus metrics: https://docs.docker.com/config/daemon/prometheus/
- prometheus_client_php: https://github.com/PromPHP/prometheus_client_php

### Безопасность
- Docker socket proxy: https://github.com/Tecnativa/docker-socket-proxy
- Trivy: https://aquasecurity.github.io/trivy/
- Cosign / sigstore: https://docs.sigstore.dev/
- CIS Docker Benchmark: https://www.cisecurity.org/benchmark/docker

### Сравнение
- Kubernetes overview: https://kubernetes.io/docs/concepts/overview/
- HashiCorp Nomad: https://developer.hashicorp.com/nomad/docs

---

> Этот файл — живой справочник. Дополняйте его реальными инцидентами: какие label-схемы прижились в команде, какие лимиты `pm.max_children` оптимальны для вашего профиля нагрузки, что вылетало в `docker service ps` и как чинили. Любая практика, проверенная на проде, ценнее любого «best practice» из интернета.
