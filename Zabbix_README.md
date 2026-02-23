# Содержание

1.  [Описание проекта](#описание-проекта)
2.  [Подготовка окружения (вынесен в отдельныйй файл)](Zabbix_README_p2.md)
3.  [Структура каталогов проекта](#структура-каталогов-проекта)
4.  [Создание эталонной конфигурации (base)](#создание-эталонной-конфигурации-base)
5.  [Таблица портов для учебных стендов](#таблица-портов-для-учебных-стендов)
6.  [Стенд 1: Создание кастомного шаблона (lab1-template)](#стенд-1-создание-кастомного-шаблона-lab1-template)
7.  [Стенд 2: Добавление хостов с Zabbix Agent (lab2-add-hosts)](#стенд-2-добавление-хостов-с-zabbix-agent-lab2-add-hosts)
8.  [Стенд 3: Применение кастомного шаблона (lab3-apply)](#стенд-3-применение-кастомного-шаблона-lab3-apply)
9.  [Стенд 4: Создание дашборда (lab4-dashboard)](#стенд-4-создание-дашборда-lab4-dashboard)
10. [Инструкция по добавлению нового стенда](#инструкция-по-добавлению-нового-стенда)
11. [Полезные команды и скрипты](#полезные-команды-и-скрипты)

---

# Описание проекта

Данный репозиторий предназначен для развертывания учебных стендов Zabbix 7.0 LTS в среде Windows 11 с использованием подсистемы WSL2 и Docker. Цель проекта — последовательное освоение навыков администрирования Zabbix: от создания шаблонов до разработки пользовательских дашбордов.

[К оглавлению](#содержание)

# Структура каталогов проекта

Все файлы проекта будут храниться в домашней директории пользователя WSL2. Создайте следующую структуру каталогов:

```bash
mkdir -p ~/zabbix-labs/{base,hosts,lab1-template,lab2-add-hosts,lab3-apply,lab4-dashboard,scripts}
cd ~/zabbix-labs
tree
```

Ожидаемый результат:
```
zabbix-labs/
├── base/                   # Эталонный compose-файл и конфиги
├── hosts/                  # Dockerfile и конфиги для сборки кастомного образа агента
├── lab1-template/           # Инструкции и compose-файл для стенда 1
├── lab2-add-hosts/          # Инструкции и compose-файл для стенда 2
├── lab3-apply/              # Инструкции для стенда 3
├── lab4-dashboard/          # Инструкции для стенда 4
└── scripts/                 # Вспомогательные скрипты
```

[К оглавлению](#содержание)

# Создание эталонной конфигурации (base)

В каталоге `~/zabbix-labs/base/` создайте файл `docker-compose.base.yml`. Это шаблон для всех будущих стендов.

**Файл: `base/docker-compose.base.yml`**

```yaml
# Эталонная конфигурация Zabbix 7.0 LTS (PostgreSQL)
# Используется как основа для всех учебных стендов.

services:
  # База данных PostgreSQL
  postgres-server:
    image: postgres:15-alpine
    container_name: zabbix-postgres
    # Переменные окружения для инициализации БД
    environment:
      POSTGRES_USER: zabbix
      POSTGRES_PASSWORD: zabbix
      POSTGRES_DB: zabbix
    volumes:
      # Том для хранения данных БД, чтобы они не терялись при перезапуске
      - postgres-data:/var/lib/postgresql/data
    networks:
      - zabbix-shared-net
    restart: unless-stopped
    # Здоровье контейнера для зависимостей
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U zabbix"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Zabbix Server (ядро системы)
  zabbix-server:
    image: zabbix/zabbix-server-pgsql:ubuntu-7.0-latest
    container_name: zabbix-server
    environment:
      DB_SERVER_HOST: postgres-server
      POSTGRES_USER: zabbix
      POSTGRES_PASSWORD: zabbix
      POSTGRES_DB: zabbix
      # Отключаем проверку подлинности для агентов (учебный режим)
      ZBX_ENABLE_SNMP_TRAPS: false
    ports:
      # Основной порт для связи с агентами (будет изменяться в стендах)
      - "10051:10051"
    networks:
      - zabbix-shared-net
    depends_on:
      postgres-server:
        condition: service_healthy
    restart: unless-stopped

  # Zabbix Web UI (интерфейс на PHP + Nginx)
  zabbix-web:
    image: zabbix/zabbix-web-nginx-pgsql:ubuntu-7.0-latest
    container_name: zabbix-web
    environment:
      ZBX_SERVER_HOST: zabbix-server
      DB_SERVER_HOST: postgres-server
      POSTGRES_USER: zabbix
      POSTGRES_PASSWORD: zabbix
      POSTGRES_DB: zabbix
      # Учетные данные по умолчанию: Admin / zabbix
    ports:
      # Порт для доступа к веб-интерфейсу из Windows (будет изменяться в стендах)
      - "8080:8080"
    networks:
      - zabbix-shared-net
    depends_on:
      postgres-server:
        condition: service_healthy
      zabbix-server:
        condition: service_started
    restart: unless-stopped

# Единая пользовательская сеть для всех контейнеров стенда
networks:
  zabbix-shared-net:
    driver: bridge
    name: zabbix-shared-net

# Именованный том для данных PostgreSQL
volumes:
  postgres-data:
```

[К оглавлению](#содержание)

# Таблица портов для учебных стендов

При создании нового независимого стенда (с собственным сервером) используйте следующее правило смещения портов, чтобы избежать конфликтов.

| Компонент | Базовый порт (Стенд 1) | Порт для Стенда N |
| :--- | :--- | :--- |
| Web UI (HTTP) | `8080` | `8080 + (N-1)` |
| Zabbix Server (трафик от агентов) | `10051` | `10051 + (N-1)` |

[К оглавлению](#содержание)

# Стенд 1: Создание кастомного шаблона (lab1-template)

**Цель:** Развернуть базовый стенд и создать шаблон с метриками CPU и RAM.

**1. Подготовка конфигурации стенда**
Перейдите в каталог стенда и скопируйте эталонный файл:
```bash
cd ~/zabbix-labs/lab1-template
cp ../base/docker-compose.base.yml ./docker-compose.yml
```
Для первого стенда изменения портов не требуются (используются базовые `8080` и `10051`).

**2. Запуск стенда**
```bash
docker compose up -d
```
Дождитесь полной инициализации всех контейнеров (около 1-2 минут). Проверить статус можно командой `docker compose ps`.

**3. Доступ к веб-интерфейсу**
Откройте браузер на вашей Windows-машине и перейдите по адресу:
```
http://localhost:8080
```
Логин: `Admin`, пароль: `zabbix`.

**4. Создание кастомного шаблона**
В веб-интерфейсе Zabbix выполните следующие шаги:
1.  Перейдите в раздел **Configuration (Настройка) -> Templates (Шаблоны)**.
2.  Нажмите кнопку **Create template (Создать шаблон)** в правом верхнем углу.
3.  Заполните поля:
    *   **Template name:** `Training Template CPU RAM`
    *   **Groups:** Выберите "Templates/Applications" (или создайте новую группу, например "Training").
4.  Перейдите на вкладку **Items (Элементы данных)** и нажмите **Create item (Создать элемент данных)**.
    *   **Создайте первый элемент (CPU):**
        *   **Name:** `CPU utilization`
        *   **Key:** `system.cpu.util[,idle]` (мы возьмем значение idle, а в Zabbix настроим расчет загрузки как 100 - idle). *Примечание: можно использовать и `system.cpu.util[user]`, но для простоты используем расчет.*
        *   **Type of information:** Numeric (float)
        *   **Units:** %
        *   **Store value:** Delta (speed per second) - для процессора обычно так и делается, но для idle подойдет As is.
        *   Нажмите **Add**.
    *   **Создайте второй элемент (RAM):**
        *   **Name:** `Memory utilization`
        *   **Key:** `vm.memory.size[pused]` (этот ключ уже возвращает процент использования).
        *   **Type of information:** Numeric (float)
        *   **Units:** %
        *   Нажмите **Add**.
5.  Нажмите кнопку **Add** внизу формы для сохранения шаблона.

**Ожидаемый результат:** В списке шаблонов появился "Training Template CPU RAM".

[К оглавлению](#содержание)

# Стенд 2: Добавление хостов с Zabbix Agent (lab2-add-hosts)

**Цель:** Развернуть два контейнера-агента и подключить их к серверу.

**Важно:** Этот стенд является независимым и требует изменения портов (так как сервер Zabbix будет свой). Для простоты в рамках одной учебной среды можно запускать стенды последовательно, останавливая предыдущие (`docker compose down` в каталоге lab1), либо использовать разные порты.

**1. Подготовка конфигурации стенда**
Перейдите в каталог стенда, скопируйте эталонный файл и отредактируйте порты.
```bash
cd ~/zabbix-labs/lab2-add-hosts
cp ../base/docker-compose.base.yml ./docker-compose.yml
```
Отредактируйте файл `docker-compose.yml`, изменив проброс портов для сервисов `zabbix-server` и `zabbix-web` (согласно таблице: N=2 -> +1):
*   Сервер: `"10052:10051"`
*   Веб: `"8082:8080"`

**2. Добавление сервисов агентов в `docker-compose.yml`**
Откройте файл `docker-compose.yml` и добавьте в конец раздела `services` (после `zabbix-web`) следующие два сервиса. Обратите внимание на отступы (YAML-формат).

```yaml
  # Первый агент (имя хоста: motorinag-1)
  zabbix-agent-1:
    image: zabbix/zabbix-agent2:ubuntu-7.0-latest
    container_name: motorinag-1
    environment:
      # Имя хоста, которое будет отображаться в Zabbix
      ZBX_HOSTNAME: motorinag-1
      # Адрес сервера Zabbix (имя сервиса в сети Docker)
      ZBX_SERVER_HOST: zabbix-server
      # Разрешаем подключения от сервера (пассивные проверки)
      ZBX_SERVER_PORT: 10051
      # Отключаем активные проверки для простоты (опционально)
      ZBX_ACTIVE_ALLOW: false
    # Для пассивных проверок порт должен быть открыт (стандартный 10050)
    ports:
      - "10050" # Пробрасываем случайный порт хоста, чтобы избежать конфликтов между стендами
    networks:
      - zabbix-shared-net
    depends_on:
      - zabbix-server
    restart: unless-stopped

  # Второй агент (имя хоста: motorinag-2)
  zabbix-agent-2:
    image: zabbix/zabbix-agent2:ubuntu-7.0-latest
    container_name: motorinag-2
    environment:
      ZBX_HOSTNAME: motorinag-2
      ZBX_SERVER_HOST: zabbix-server
      ZBX_SERVER_PORT: 10051
      ZBX_ACTIVE_ALLOW: false
    ports:
      - "10050" # Случайный порт
    networks:
      - zabbix-shared-net
    depends_on:
      - zabbix-server
    restart: unless-stopped
```

**3. Запуск стенда и добавление хостов в веб-интерфейсе**
```bash
docker compose up -d
```
Откройте веб-интерфейс по адресу `http://localhost:8082`.

В веб-интерфейсе:
1.  Перейдите в **Configuration (Настройка) -> Hosts (Хосты)**.
2.  Нажмите **Create host (Создать хост)**.
3.  Заполните для **motorinag-1**:
    *   **Host name:** `motorinag-1`
    *   **Groups:** выберите "Linux Servers" (или создайте новую группу, например "Training Hosts").
    *   **Interfaces:** нажмите "Add" для агента. Укажите **DNS name** = `motorinag-1` (так как контейнеры в одной сети Docker, они могут обращаться друг к другу по именам контейнеров). Порт оставьте `10050`.
    *   Перейдите на вкладку **Templates (Шаблоны)**. Нажмите "Select" и выберите шаблон "Linux by Zabbix agent". Нажмите "Add".
    *   Нажмите кнопку **Add** внизу формы.
4.  Повторите шаги для хоста `motorinag-2`.

**Ожидаемый результат:** Через несколько минут в разделе **Monitoring (Мониторинг) -> Latest data (Последние данные)** появятся данные от обоих хостов.

[К оглавлению](#содержание)

# Стенд 3: Применение кастомного шаблона (lab3-apply)

**Цель:** Применить созданный ранее кастомный шаблон к хостам.

**Важно:** Этот стенд использует инфраструктуру Стенда 2 (lab2-add-hosts). Убедитесь, что контейнеры из lab2 запущены. Если вы останавливали lab2, перейдите в его каталог и выполните `docker compose up -d`.

**1. Подготовка (если стенд 2 не запущен)**
```bash
cd ~/zabbix-labs/lab2-add-hosts
docker compose up -d
```

**2. Применение шаблона**
1.  Откройте веб-интерфейс Стенда 2 (`http://localhost:8082`).
2.  Перейдите в **Configuration (Настройка) -> Hosts (Хосты)**.
3.  Нажмите на имя хоста `motorinag-1`.
4.  Перейдите на вкладку **Templates (Шаблоны)**.
5.  В поле "Link new templates" начните вводить "Training" и выберите шаблон "Training Template CPU RAM".
6.  Нажмите **Add** (синяя ссылка), а затем кнопку **Update (Обновить)** внизу страницы.
7.  Повторите шаги для хоста `motorinag-2`.

**Ожидаемый результат:** Через некоторое время (до 1 минуты) в разделе **Monitoring (Мониторинг) -> Latest data (Последние данные)** для каждого хоста появятся новые элементы данных: "CPU utilization" и "Memory utilization".

[К оглавлению](#содержание)

# Стенд 4: Создание дашборда (lab4-dashboard)

**Цель:** Разработать пользовательский дашборд для мониторинга системы.

**1. Создание дашборда**
В веб-интерфейсе Стенда 2 (`http://localhost:8082`):
1.  Перейдите в раздел **Dashboards (Дашборды)**.
2.  Нажмите на название текущего дашборда (обычно "Dashboard") и выберите **Create dashboard (Создать дашборд)**.
3.  Введите имя: **System Monitoring**.
4.  Нажмите **Create**.

**2. Добавление графиков**
Теперь нужно добавить виджеты на пустой дашборд.
1.  Нажмите кнопку **Add widget (Добавить виджет)** в правом верхнем углу.
2.  В открывшемся окне выберите тип виджета **Graph (Classic)** (График (классический)).
3.  **Для графика CPU хоста motorinag-1:**
    *   **Name:** `motorinag-1: CPU`
    *   В поле **Source (Источник)** перейдите на вкладку **Items (Элементы)**.
    *   Нажмите **Add (Добавить)**. В поиске введите `motorinag-1` и выберите элемент данных `CPU utilization`. Нажмите **Select (Выбрать)**.
    *   Нажмите **Add** для добавления виджета на дашборд.
4.  Повторите шаги 1-3, создав виджеты для:
    *   CPU хоста `motorinag-2` (элемент `CPU utilization`).
    *   RAM хоста `motorinag-1` (элемент `Memory utilization`).
    *   RAM хоста `motorinag-2` (элемент `Memory utilization`).
5.  **Для графика сетевой активности:**
    *   Добавьте новый виджет типа **Graph (Classic)**.
    *   **Name:** `motorinag-1: Network traffic`
    *   В поле **Source** перейдите на вкладку **Items**.
    *   Нажмите **Add**. Найдите и выберите два элемента данных, относящихся к хосту `motorinag-1`: `net.if.in[eth0]` и `net.if.out[eth0]` (если интерфейс называется не eth0, найдите соответствующие элементы в шаблоне "Linux by Zabbix agent"). Нажмите **Select**.
    *   Нажмите **Add**.
6.  Расположите виджеты на дашборде с помощью перетаскивания.
7.  Нажмите кнопку **Save changes (Сохранить изменения)** в правом верхнем углу.

**Ожидаемый результат:** Дашборд "System Monitoring" отображает графики загрузки CPU, использования RAM и сетевой активности для обоих хостов.

[К оглавлению](#содержание)

# Инструкция по добавлению нового стенда

Для создания следующего независимого стенда (например, lab5) выполните следующие шаги:

1.  **Создайте каталог** для нового стенда:
    ```bash
    cd ~/zabbix-labs
    mkdir lab5-new-feature
    ```
2.  **Скопируйте эталонный файл:**
    ```bash
    cd lab5-new-feature
    cp ../base/docker-compose.base.yml ./docker-compose.yml
    ```
3.  **Рассчитайте смещение портов:** для стенда N=5, смещение = N-1 = 4.
    *   Web UI: `8080 + 4 = 8084`
    *   Server port: `10051 + 4 = 10055`
4.  **Отредактируйте** файл `docker-compose.yml`:
    *   Найдите сервис `zabbix-server` и измените секцию `ports` на `"10055:10051"`.
    *   Найдите сервис `zabbix-web` и измените секцию `ports` на `"8084:8080"`.
5.  **Добавьте необходимые сервисы** (например, агенты) в файл `docker-compose.yml` по аналогии со стендом 2.
6.  **Запустите новый стенд:**
    ```bash
    docker compose up -d
    ```
7.  Убедитесь, что порты не конфликтуют с другими запущенными стендами (используйте `docker ps` или `netstat -tulpn | grep 8084`).

[К оглавлению](#содержание)

# Полезные команды и скрипты

В каталоге `~/zabbix-labs/scripts/` можно создать вспомогательные скрипты.

**Пример скрипта для остановки всех контейнеров стендов:**
Создайте файл `scripts/stop-all-labs.sh`:
```bash
#!/bin/bash
echo "Остановка всех контейнеров Zabbix Labs..."
cd ~/zabbix-labs/lab1-template && docker compose down
cd ~/zabbix-labs/lab2-add-hosts && docker compose down
cd ~/zabbix-labs/lab3-apply && docker compose down # Если он у вас отдельный
cd ~/zabbix-labs/lab4-dashboard && docker compose down
echo "Готово."
```
Не забудьте дать права на выполнение: `chmod +x scripts/stop-all-labs.sh`.

**Другие полезные команды:**
*   Просмотр логов конкретного контейнера: `docker logs -f motorinag-1`
*   Зайти внутрь контейнера агента: `docker exec -it motorinag-1 bash`
*   Перезапустить сервисы стенда: `docker compose restart`
*   Остановить и удалить тома (стереть БД): `docker compose down -v`

[К оглавлению](#содержание)
