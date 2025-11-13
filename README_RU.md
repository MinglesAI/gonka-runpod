# Подробная инструкция по развертыванию Inference нод на RunPod

Эта инструкция описывает процесс развертывания inference (ML) нод на платформе RunPod с использованием готового шаблона. Инструкция дополняет и расширяет официальную документацию: https://gonka.ai/host/multiple-nodes/

---

## Содержание

1. [Обзор архитектуры](#обзор-архитектуры)
2. [Предварительные требования](#предварительные-требования)
3. [Развертывание Network Node](#развертывание-network-node)
4. [Развертывание Inference Node на RunPod](#развертывание-inference-node-на-runpod)
5. [Регистрация Inference Node в Network Node](#регистрация-inference-node-в-network-node)
6. [Проверка работоспособности](#проверка-работоспособности)
7. [Управление нодами](#управление-нодами)
8. [Устранение неполадок](#устранение-неполадок)

---

## Обзор архитектуры

В распределенной конфигурации Gonka используется следующая архитектура:

- **Network Node** — основной сервис, состоящий из:
  - **Chain Node** — подключается к блокчейну
  - **API Node** — обрабатывает пользовательские запросы и управляет inference нодами
  - Должен быть доступен из интернета (статический IP или домен)
  - Требования: 16 CPU cores, 64+ GB RAM, 1TB NVMe SSD, 100Mbps+ сеть

- **Inference (ML) Node** — сервис для выполнения инференса LLM на GPU:
  - Может быть развернут на отдельном сервере (например, RunPod)
  - Использует vLLM для инференса моделей
  - Подключается к Network Node через прокси (nginx)
  - Требования: GPU с достаточным объемом памяти для выбранной модели

### Особенности RunPod

RunPod предоставляет GPU-инстансы с предустановленным Docker и автоматическим проксированием через домены вида:
- `{RUNPOD_ID}-8080.proxy.runpod.net` — для POC порта (8080)
- `{RUNPOD_ID}-5000.proxy.runpod.net` — для inference порта (5000)

Это позволяет подключать inference ноды без необходимости настройки публичных IP и портов.

---

## Предварительные требования

### На Network Node сервере

1. **Завершите Quickstart guide до шага 3.4**, включая:
   - Установку Docker и Docker Compose
   - Настройку ключей (Account Key и ML Operational Key)
   - Регистрацию Host в сети
   - Настройку `config.env`

2. **Важно: Настройте `DAPI_API__POC_CALLBACK_URL`**

   В файле `config.env` на Network Node сервере убедитесь, что переменная `DAPI_API__POC_CALLBACK_URL` настроена правильно:

   ```bash
   # НЕ используйте внутренний Docker адрес для multi-node setup!
   # DAPI_API__POC_CALLBACK_URL=http://api:9100  # ❌ Неправильно
   
   # Используйте приватный IP или DNS имя вашего Network Node сервера
   DAPI_API__POC_CALLBACK_URL=http://<NETWORK_NODE_PRIVATE_IP>:9100  # ✅ Правильно
   # Или с доменом:
   DAPI_API__POC_CALLBACK_URL=http://your-network-node-domain.com:9100
   ```

   **Почему это важно:** Все inference ноды должны иметь возможность отправлять Proof-of-Compute (PoC) nonces на этот URL. Если используется внутренний Docker адрес, внешние ноды не смогут подключиться.

3. **Откройте порт 9100**

   Убедитесь, что порт `9100` открыт и доступен из интернета (или из приватной сети, где находятся inference ноды).

### На RunPod инстансе

1. **Создайте GPU инстанс на RunPod**
   - Перейдите по ссылке на шаблон RunPod: [https://console.runpod.io/deploy?template=vuf848tt4c&ref=1pxn4obn](https://console.runpod.io/deploy?template=vuf848tt4c&ref=1pxn4obn)
   - Выберите подходящий GPU (H100, A100, 4090 и т.д.)

2. **Подготовьте файлы конфигурации**

   Или используйте готовые файлы из директории `gonka-runpod/` (здесь нужно прописать ваш RUNPOD_ID только в docker-compose.mlnode.yml и nginx-runpod.conf.template):
   - `docker-compose.mlnode.yml` — конфигурация Docker Compose (обязательно укажите переменную окружения RUNPOD_ID: `environment: - RUNPOD_ID=...`)
   - `nginx-runpod.conf.template` — шаблон конфигурации nginx (RUNPOD_ID автоматически подставится при запуске)
   - `node-config.json` — пример конфигурации для регистрации: здесь не нужно указывать RUNPOD_ID, а в поле `host` укажите публичный адрес вашей Network Node, на которой будет работать контейнер с nginx

---

## Развертывание Network Node

### Вариант 1: Network Node + Inference Node на одной машине (с GPU)

Если ваш Network Node сервер имеет GPU и вы хотите запустить обе ноды на одной машине:

```bash
cd gonka/deploy/join
source config.env && \
docker compose -f docker-compose.yml -f up -d && \
docker compose -f docker-compose.yml -f logs -f
```

### Вариант 2: Только Network Node (без GPU)

Если Network Node сервер не имеет GPU:

```bash
cd gonka/deploy/join
source config.env && \
docker compose -f docker-compose.yml up -d && \
docker compose -f docker-compose.yml logs -f
```

### Проверка статуса Network Node

После запуска Network Node начнет участвовать в Proof of Computation (PoC) после того, как к нему подключится хотя бы одна inference нода. Проверить список активных Hosts можно через:

```
http://node2.gonka.ai:8000/v1/epochs/current/participants
```

**Примечание:** Изменения могут занять 1-3 часа для отображения в списке.

---

## Развертывание Inference Node на RunPod

### Шаг 1: Использование готового шаблона RunPod

**Быстрый способ:** Используйте готовый шаблон RunPod:

```
https://console.runpod.io/deploy?template=vuf848tt4c&ref=1pxn4obn
```

Этот шаблон автоматически настроит inference ноду с необходимыми конфигурациями.

### Шаг 2: Ручная настройка (если шаблон не используется)

#### 2.1. Подготовка файлов на RunPod инстансе

1. **Создайте рабочую директорию:**

   ```bash
   mkdir -p ~/gonka-runpod
   cd ~/gonka-runpod
   ```

2. **Создайте файл `docker-compose.mlnode.yml`:**

   ```yaml
   services:
     inference-node1:
       image: nginx:1.28.0-alpine
       hostname: inference-node1
       ports:
         - "8080:8080"  # POC порт (маппится на 8080 внутри контейнера)
         - "5000:5000"  # Inference порт (маппится на 5000 внутри контейнера)
       environment:
         - RUNPOD_ID=your_runpod_id_here  # ⚠️ ЗАМЕНИТЕ на ваш RUNPOD_ID
       volumes:
         - ./nginx-runpod.conf.template:/etc/nginx/nginx.conf.template:ro
       command: >
         /bin/sh -c "
           apk add --no-cache gettext &&
           envsubst '$$RUNPOD_ID' < /etc/nginx/nginx.conf.template > /etc/nginx/nginx.conf &&
           nginx -g 'daemon off;'
         "
       restart: always
   ```

3. **Скопируйте файл `nginx-runpod.conf.template`** из репозитория или создайте его:

   Этот файл уже должен быть в директории `gonka-runpod/`. Он содержит конфигурацию nginx, которая проксирует запросы на RunPod прокси-домены.

#### 2.2. Получение RUNPOD_ID

**Важно:** Вам нужно получить `RUNPOD_ID` вашего инстанса. Это можно сделать несколькими способами:

**Способ 1: Из URL RunPod консоли**
- Откройте консоль вашего инстанса в браузере
- В URL будет что-то вроде: `https://www.runpod.io/console/pods?podId=abc123xyz`
- `abc123xyz` — это ваш RUNPOD_ID

**Способ 2: Из переменных окружения RunPod**
- RunPod автоматически устанавливает переменную `RUNPOD_POD_ID`
- Проверьте: `echo $RUNPOD_POD_ID`

**Способ 3: Из метаданных инстанса**
- В терминале RunPod выполните: `curl http://localhost:9000/pod` (если доступно)

#### 2.3. Обновление конфигурации

Откройте `docker-compose.mlnode.yml` и замените `your_runpod_id_here` на ваш реальный RUNPOD_ID:

```yaml
environment:
  - RUNPOD_ID=e6rbg7z3qyr7a4  # Пример реального ID
```

#### 2.4. Запуск Inference Node

```bash
cd ~/gonka-runpod
docker compose -f docker-compose.mlnode.yml up -d
```

Проверьте логи:

```bash
docker compose -f docker-compose.mlnode.yml logs -f
```

#### 2.5. Проверка работы nginx прокси

После запуска nginx должен проксировать запросы на:
- `https://{RUNPOD_ID}-8080.proxy.runpod.net` — для POC (порт 8080)
- `https://{RUNPOD_ID}-5000.proxy.runpod.net` — для inference (порт 5000)

Проверить можно:

```bash
# Проверка POC порта
curl http://localhost:8080/health

# Проверка inference порта
curl http://localhost:5000/health
```

### Шаг 3: Настройка RunPod Template (альтернативный способ)

Если вы используете RunPod Template, убедитесь, что:

1. **Порты открыты в RunPod:**
   - Порт `8080` — для POC API
   - Порт `5000` — для Inference API

2. **Переменная окружения RUNPOD_ID установлена:**
   - В настройках Template должна быть переменная `RUNPOD_ID`
   - Или она должна автоматически определяться из метаданных инстанса

3. **Файлы конфигурации загружены:**
   - `docker-compose.mlnode.yml`
   - `nginx-runpod.conf.template`

---

## Регистрация Inference Node в Network Node

После того, как inference нода запущена на RunPod, её необходимо зарегистрировать в Network Node через Admin API.

### Шаг 1: Определение параметров для регистрации

Перед регистрацией соберите следующую информацию:

1. **ID ноды** — уникальный идентификатор (например, `runpod-node1`)
2. **Host** — публичный IP или домен RunPod инстанса
   - Для RunPod это будет: `{RUNPOD_ID}-8080.proxy.runpod.net` (для POC)
   - Или статический IP, если он назначен
3. **Inference Port** — порт для inference запросов: `5000`
4. **POC Port** — порт для управления: `8080`
5. **Max Concurrent** — максимальное количество одновременных запросов (например, `500`)
6. **Models** — список поддерживаемых моделей и их параметры

### Шаг 2: Подготовка конфигурации модели

Сейчас сеть поддерживает две модели:

1. **Qwen/Qwen3-235B-A22B-Instruct-2507-FP8**
2. **Qwen/Qwen3-32B-FP8**

#### Параметры vLLM для разных конфигураций:

| Модель и GPU | vLLM аргументы |
|--------------|----------------|
| Qwen/Qwen3-235B-A22B-Instruct-2507-FP8 на 8xH100 или 8xH200 | `"--tensor-parallel-size","4"` |
| Qwen/Qwen3-32B-FP8 на 1xH100 | (без дополнительных аргументов) |
| Qwen/Qwen3-32B-FP8 на 8x4090 | `"--tensor-parallel-size","4"` |
| Qwen/Qwen3-32B-FP8 на 8x3080 | `"--tensor-parallel-size","4","--pipeline-parallel-size","2"` |

**Совет:** Для выбора оптимальной конфигурации см. руководство: [Benchmark to Choose Optimal Deployment Config for LLMs](https://gonka.ai/host/benchmark-to-choose-optimal-deployment-config-for-llms/)

### Шаг 3: Регистрация через Admin API

На **Network Node сервере** выполните следующую команду:

```bash
curl -X POST http://localhost:9200/admin/v1/nodes \
     -H "Content-Type: application/json" \
     -d '{
       "id": "runpod-node1",
       "host": "https://e6rbg7z3qyr7a4-8080.proxy.runpod.net",
       "inference_port": 5000,
       "poc_port": 8080,
       "max_concurrent": 500,
       "models": {
         "Qwen/Qwen3-235B-A22B-Instruct-2507-FP8": {
           "args": [
             "--max-model-len",
             "240000",
             "--tensor-parallel-size",
             "4"
           ]
         }
       }
     }'
```

**Важные замечания:**

- **`id`** — должен быть уникальным для каждой ноды
- **`host`** — для RunPod используйте формат `https://{RUNPOD_ID}-8080.proxy.runpod.net`
  - Если у вас есть статический IP, можно использовать его: `http://<STATIC_IP>`
- **`inference_port`** — всегда `5000` для RunPod (внутренний порт nginx)
- **`poc_port`** — всегда `8080` для RunPod (внутренний порт nginx)
- **`max_concurrent`** — зависит от вашего GPU и модели, обычно 200-500

### Шаг 4: Проверка успешной регистрации

Если регистрация прошла успешно, вы получите JSON ответ с конфигурацией добавленной ноды:

```json
{
  "id": "runpod-node1",
  "host": "https://e6rbg7z3qyr7a4-8080.proxy.runpod.net",
  "inference_port": 5000,
  "poc_port": 8080,
  ...
}
```

### Примеры регистрации для разных моделей

#### Пример 1: Qwen/Qwen3-32B-FP8 на 1xH100

```bash
curl -X POST http://localhost:9200/admin/v1/nodes \
     -H "Content-Type: application/json" \
     -d '{
       "id": "runpod-node1",
       "host": "https://your-runpod-id-8080.proxy.runpod.net",
       "inference_port": 5000,
       "poc_port": 8080,
       "max_concurrent": 300,
       "models": {
         "Qwen/Qwen3-32B-FP8": {
           "args": []
         }
       }
     }'
```

#### Пример 2: Qwen/Qwen3-32B-FP8 на 8x4090

```bash
curl -X POST http://localhost:9200/admin/v1/nodes \
     -H "Content-Type: application/json" \
     -d '{
       "id": "runpod-node2",
       "host": "https://your-runpod-id-8080.proxy.runpod.net",
       "inference_port": 5000,
       "poc_port": 8080,
       "max_concurrent": 500,
       "models": {
         "Qwen/Qwen3-32B-FP8": {
           "args": [
             "--tensor-parallel-size",
             "4"
           ]
         }
       }
     }'
```

---

## Проверка работоспособности

### 1. Проверка статуса Inference Node

После регистрации нода должна начать обрабатывать запросы. Обычно это занимает несколько минут.

Проверьте логи на RunPod инстансе:

```bash
docker compose -f docker-compose.mlnode.yml logs -f
```

Вы должны видеть успешные подключения и проксирование запросов.

### 2. Проверка списка зарегистрированных нод

На Network Node сервере:

```bash
curl -X GET http://localhost:9200/admin/v1/nodes
```

Это вернет JSON массив со всеми зарегистрированными inference нодами.

### 3. Проверка участия в PoC

После следующего Proof of Computation (обычно 1-3 часа), ваш Network Node должен появиться в списке активных Hosts:

```
http://node2.gonka.ai:8000/v1/epochs/current/participants
```

Вес Network Node будет обновляться на основе работы подключенных inference нод.

---

## Управление нодами

### Получение списка всех нод

```bash
curl -X GET http://localhost:9200/admin/v1/nodes
```

### Удаление ноды

Для удаления inference ноды (без перезапуска Network Node):

```bash
curl -X DELETE "http://localhost:9200/admin/v1/nodes/{id}" \
     -H "Content-Type: application/json"
```

Где `{id}` — это идентификатор ноды, указанный при регистрации.

Пример:

```bash
curl -X DELETE "http://localhost:9200/admin/v1/nodes/runpod-node1" \
     -H "Content-Type: application/json"
```

При успешном удалении вернется `true`.

### Обновление конфигурации ноды

Для обновления конфигурации ноды, сначала удалите её, затем зарегистрируйте заново с новыми параметрами.

---

## Устранение неполадок

### Проблема 1: Inference нода не подключается к Network Node

**Симптомы:**
- Нода зарегистрирована, но не обрабатывает запросы
- Ошибки подключения в логах

**Решения:**

1. **Проверьте доступность Network Node из RunPod:**
   ```bash
   # На RunPod инстансе
   curl http://<NETWORK_NODE_IP>:9100/health
   ```

2. **Проверьте правильность `DAPI_API__POC_CALLBACK_URL`:**
   - Убедитесь, что в `config.env` на Network Node указан правильный IP/домен
   - Не используйте внутренние Docker адреса (`http://api:9100`)

3. **Проверьте порты:**
   - Порт 9100 должен быть открыт на Network Node
   - Порты 8080 и 5000 должны быть открыты на RunPod

4. **Проверьте RUNPOD_ID:**
   - Убедитесь, что `RUNPOD_ID` в `docker-compose.mlnode.yml` правильный
   - Проверьте, что RunPod прокси доступны: `curl https://{RUNPOD_ID}-8080.proxy.runpod.net`

### Проблема 2: Ошибки nginx при запуске

**Симптомы:**
- Контейнер nginx не запускается
- Ошибки в логах о конфигурации nginx

**Решения:**

1. **Проверьте наличие файла шаблона:**
   ```bash
   ls -la nginx-runpod.conf.template
   ```

2. **Проверьте синтаксис nginx конфигурации:**
   ```bash
   docker compose -f docker-compose.mlnode.yml exec inference-node1 nginx -t
   ```

3. **Проверьте переменную RUNPOD_ID:**
   - Убедитесь, что она установлена в `docker-compose.mlnode.yml`
   - Проверьте логи: `docker compose logs inference-node1`

### Проблема 3: Нода не появляется в списке участников PoC

**Симптомы:**
- Нода зарегистрирована, но не видна в списке активных Hosts

**Решения:**

1. **Подождите 1-3 часа** — изменения в списке участников применяются после следующего PoC

2. **Проверьте, что хотя бы одна inference нода подключена:**
   ```bash
   curl -X GET http://localhost:9200/admin/v1/nodes
   ```

3. **Проверьте логи Network Node:**
   ```bash
   docker compose logs -f api
   ```

4. **Убедитесь, что inference нода обрабатывает запросы:**
   - Проверьте логи на RunPod инстансе
   - Должны быть запросы на inference и PoC

### Проблема 4: Ошибки при регистрации ноды

**Симптомы:**
- Ошибка 400/500 при выполнении curl запроса на регистрацию

**Решения:**

1. **Проверьте формат JSON:**
   - Убедитесь, что JSON валидный (можно проверить на https://jsonlint.com/)
   - Проверьте правильность кавычек (должны быть двойные)

2. **Проверьте параметры:**
   - `id` должен быть уникальным
   - `host` должен быть доступным URL
   - `inference_port` и `poc_port` должны быть числами

3. **Проверьте доступность Admin API:**
   ```bash
   curl http://localhost:9200/admin/v1/nodes
   ```

### Проблема 5: RunPod прокси не работает

**Симптомы:**
- Запросы к `{RUNPOD_ID}-8080.proxy.runpod.net` не проходят

**Решения:**

1. **Проверьте правильность RUNPOD_ID:**
   - Убедитесь, что ID правильный (без лишних символов)

2. **Проверьте, что порты открыты в RunPod:**
   - В настройках инстанса должны быть открыты порты 8080 и 5000

3. **Проверьте статус инстанса:**
   - Убедитесь, что инстанс активен и не приостановлен

---

## Дополнительные ресурсы

- [Официальная документация: Multiple nodes](https://gonka.ai/host/multiple-nodes/)
- [Network Node API документация](https://gonka.ai/host/network-node-api/)
- [Benchmark для выбора оптимальной конфигурации](https://gonka.ai/host/benchmark-to-choose-optimal-deployment-config-for-llms/)
- [Hardware specifications](https://gonka.ai/host/hardware-specifications/)

---

## Быстрая справка по командам

### На Network Node сервере

```bash
# Запуск Network Node
cd gonka/deploy/join
source config.env && docker compose -f docker-compose.yml up -d

# Просмотр логов
docker compose -f docker-compose.yml logs -f

# Регистрация inference ноды
curl -X POST http://localhost:9200/admin/v1/nodes \
     -H "Content-Type: application/json" \
     -d @node-config.json

# Список всех нод
curl -X GET http://localhost:9200/admin/v1/nodes

# Удаление ноды
curl -X DELETE "http://localhost:9200/admin/v1/nodes/{id}" \
     -H "Content-Type: application/json"
```

### На RunPod инстансе

```bash
# Запуск inference ноды
cd ~/gonka-runpod
docker compose -f docker-compose.mlnode.yml up -d

# Просмотр логов
docker compose -f docker-compose.mlnode.yml logs -f

# Остановка
docker compose -f docker-compose.mlnode.yml down

# Перезапуск
docker compose -f docker-compose.mlnode.yml restart
```

---

**Примечание:** Эта инструкция дополняет официальную документацию и предоставляет более детальные шаги для развертывания на RunPod. Если вы столкнулись с проблемами, не описанными здесь, обратитесь к официальной документации или свяжитесь с поддержкой.

