---
layout: post
title: "Личное облако на Proxmox: нейронки, LLM и embedding"
date: 2025-11-17 21:58:46 +0000
tags: ["proxmox", "lxc", "openvino", "ml", "gpu passthrough", "embeddings"]
description: "В прошлой статье я описывал как построить сетевую часть самодержавного сервера, пора принести в него что-то отдаленно разумное. Это руководство описывает весь процесс: от подготовки хоста Proxmox и настройки LXC-контейнера до поиска, конвертации и запуска embedding-моделей (на примере BAAI/bge-large-en-v1.5 ) с использованием Intel GPU для работы модели. Но будет легко запустить не одну модель или полноценного чатбота на этой основе. Главное, что будет ясно как использовать даже простое имеющееся железо домашнего сервера для этого."
---

В прошлой [статье]({{ site.baseurl }}{% post_url 2025-11-03-proxmox-network %}) я описывал как построить сетевую часть самодержавного сервера, пора принести в него что-то отдаленно разумное. Это руководство описывает весь процесс: от подготовки хоста Proxmox и настройки LXC-контейнера до поиска, конвертации и запуска embedding-моделей (на примере `BAAI/bge-large-en-v1.5`) с использованием встроенного Intel iGPU для работы модели. Но будет легко запустить не одну модель или полноценного чатбота на этой основе. Главное, что будет ясно как использовать даже простое имеющееся железо домашнего сервера для этого.

------------------------------------------------------------------------

Небольшое отступление по поводу того, что у нас получилось после прошлой статьи:

1.  Есть Asus NUC у которого процессор двенадцатого поколения Intel и Intel Xe графикой, NPU и 12 ядрами CPU. Встройка работает с ОЗУ, а значит медленно, но верно сможет работать даже с приличного размера моделями, но все проверять надо, пока, первый заход.

2.  Сетевая инфраструктура, которая позволяет нам поднимать в локальной сети за мостом services, сервисы, которые мы можем легко выставить в интернет через Caddy реверс-прокси.

3.  Лично у меня на этот сервер переехал мой инстанс [Mastodon](https://mtdn.anyqn.com/home). Содержать его у хостера было не дешево.

4.  Помимо прочего я распилил изолированный мост services на набор отдельных VLAN для групп сервисов что бы изолировать их дополнительно, теперь OPNSense настоящий шлюз между этими сетями. Включил на OPNSense такую штуку как Suricata, которая показывает когда сервисы сканируют автоботы на предмет открытых уязвимостей, не сказать еще хужей. Если бы самодержавные хостеры знали сколько их сканят, они бы беспокоились о безопасности побольше, но о безопасности в другой раз.

Чего мы хотим:

1.  Счастья.

2.  Пробросить Intel GPU из Proxmox хоста внутрь LXC контейнера, а там и внутрь Docker контейнера. Вложенность такая потому, что запуск среды обработки неронных моделей без Docker это довольно сложное мероприятие.

3.  Запустить LXC контейнер с OpenVINO models server (OVMS), который будет движком, обеспечивающим embedding или inference, зависит от наших потребностей. Движек от Intel для их железок, очень оптимизированный и много, что может. Как запуск большого количества моделей, так и маскировку под разные API доступа.

4.  В принципе хотим еще и WebUI на это все натянуть, но это за рамками статьи потому, что мне это не нужно и сделаь это просто, так как OVMS поддерживает самые разные распространенные API, например от OpenAI, с которыми работает любая WebUI. Это в его документации описано довольно ясно.

5.  Получить возможность сделать curl запрос к нейронке и получить embedding, что мне в данном случае и нужно, так как хочется построить свой RAG и давать к нему доступ всяким нейронкам в виде MCP, если вы понимаете о чем я.

Штош, хватит отступлений, поехали. Напомню только, что если что-то не ясно, спросите нейронку, они отлично пояснят.

### Краткое содержание следующих серий

Вот [тут](https://docs.openvino.ai/2025/documentation/compatibility-and-support/supported-models.html) и [тут](https://huggingface.co/OpenVINO/models) можно найти море готовых моделей. После чего вот так их запустить:

Команды

``` bash
i3draven@ovms:~/ovms$ tree .
.
└── models

2 directories, 0 files
i3draven@ovms:~/ovms$ docker ps
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
i3draven@ovms:~/ovms$ docker run --rm -u 1000   -v ~/ovms/models:/models:rw   openvino/model_server:latest-gpu   --pull   --model_name llm   --source_model OpenVINO/DeepSeek-R1-Distill-Qwen-1.5B-int4-ov --target_device GPU --task text_generation --model_repository_path /models
remote: Enumerating objects: 19, done.
remote: Counting objects: 100% (16/16), done.
remote: Compressing objects: 100% (16/16), done.
remote: Total 19 (delta 0), reused 0 (delta 0), pack-reused 3 (from 1)
Receiving objects: 100% (19/19), 17.19 KiB | 0.00 B/s, done.
Checking out files:   0% (0/14)
Downloading lfs size: 2.09 MiB file: /models/OpenVINO/DeepSeek-R1-Distill-Qwen-1.5B-int4-ov/openvino_detokenizer.bin
Progress: [##################################################] 100.00% [2.09 MiB/s]  
Checking out files:  42% (6/14) 
Downloading lfs size: 1.30 GiB file: /models/OpenVINO/DeepSeek-R1-Distill-Qwen-1.5B-int4-ov/openvino_model.bin
Progress: [##################################################] 100.00% [66.60 MiB/s] 
Checking out files:  57% (8/14) 
Downloading lfs size: 5.33 MiB file: /models/OpenVINO/DeepSeek-R1-Distill-Qwen-1.5B-int4-ov/openvino_tokenizer.bin
Progress: [##################################################] 100.00% [5.33 MiB/s] 
Checking out files:  71% (10/14)
Downloading lfs size: 10.89 MiB file: /models/OpenVINO/DeepSeek-R1-Distill-Qwen-1.5B-int4-ov/tokenizer.json
Progress: [##################################################] 100.00% [10.89 MiB/s] 
Model: OpenVINO/DeepSeek-R1-Distill-Qwen-1.5B-int4-ov downloaded to: /models/OpenVINO/DeepSeek-R1-Distill-Qwen-1.5B-int4-ov
Graph: graph.pbtxt created in: /models/OpenVINO/DeepSeek-R1-Distill-Qwen-1.5B-int4-ov
i3draven@ovms:~/ovms$ docker run --rm -u 1000   -v ~/ovms/models:/models:rw   openvino/model_server:latest   --list_models --model_repository_path /models
OpenVINO/DeepSeek-R1-Distill-Qwen-1.5B-int4-ov
i3draven@ovms:~/ovms$ docker run --rm -u 1000   -v ~/ovms/models:/models:rw   openvino/model_server:latest   --add_to_config /models   --model_name DeepSeek-R1-Distill-Qwen-1.5B-int4-ov --model_repository_path /models/OpenVINO
Config updated: /models/config.json
i3draven@ovms:~/ovms$ docker run -d --rm   --user $(id -u):$(id -g)   --device /dev/dri:/dev/dri   --group-add $(stat -c "%g" /dev/dri/render* | head -n 1)   -v ~/ovms/models:/models:ro   -p 9000:9000 -p 8000:8000   openvino/model_server:latest-gpu   --config_path /models/config.json   --port 9000   --rest_port 8000
c4cd80757c4c8abe9f85a9c08806f1691b2a123e7e0e582ae37a968f89c86bc7
i3draven@ovms:~/ovms$ docker ps
CONTAINER ID   IMAGE                              COMMAND                  CREATED         STATUS         PORTS                                                                                      NAMES
c4cd80757c4c   openvino/model_server:latest-gpu   "/ovms/bin/ovms --co…"   3 seconds ago   Up 3 seconds   0.0.0.0:8000->8000/tcp, [::]:8000->8000/tcp, 0.0.0.0:9000->9000/tcp, [::]:9000->9000/tcp   keen_diffie
i3draven@ovms:~/ovms$ docker logs c4cd80757c4c
[2025-11-25 19:52:43.325][1][serving][info][server.cpp:91] OpenVINO Model Server 2025.3.0.6e2e910de
[2025-11-25 19:52:43.325][1][serving][info][server.cpp:92] OpenVINO backend 2025.3.0.dev20250826
[2025-11-25 19:52:43.325][1][serving][info][pythoninterpretermodule.cpp:37] PythonInterpreterModule starting
[2025-11-25 19:52:43.381][1][serving][info][pythoninterpretermodule.cpp:50] PythonInterpreterModule started
[2025-11-25 19:52:43.423][1][modelmanager][info][modelmanager.cpp:156] Available devices for Open VINO: CPU, GPU
[2025-11-25 19:52:43.424][1][serving][info][capimodule.cpp:40] C-APIModule starting
[2025-11-25 19:52:43.424][1][serving][info][capimodule.cpp:42] C-APIModule started
[2025-11-25 19:52:43.424][1][serving][info][grpcservermodule.cpp:102] GRPCServerModule starting
[2025-11-25 19:52:43.424][1][serving][info][grpcservermodule.cpp:179] GRPCServerModule started
[2025-11-25 19:52:43.424][1][serving][info][grpcservermodule.cpp:180] Started gRPC server on port 9000
[2025-11-25 19:52:43.424][1][serving][info][httpservermodule.cpp:35] HTTPServerModule starting
[2025-11-25 19:52:43.424][1][serving][info][httpservermodule.cpp:39] Will start 16 REST workers
[2025-11-25 19:52:43.475][1][serving][info][drogon_http_server.cpp:158] REST server listening on port 8000 with 16 unary threads and 16 streaming threads
[2025-11-25 19:52:43.475][1][serving][info][httpservermodule.cpp:61] HTTPServerModule started
[2025-11-25 19:52:43.475][1][serving][info][httpservermodule.cpp:62] Started REST server at 0.0.0.0:8000
[2025-11-25 19:52:43.475][1][serving][info][servablemanagermodule.cpp:51] ServableManagerModule starting
[2025-11-25 19:52:43.477][1][serving][info][mediapipegraphdefinition.cpp:421] MediapipeGraphDefinition initializing graph nodes
[2025-11-25 19:52:43.477][1][modelmanager][info][servable_initializer.cpp:463] Initializing Language Model Continuous Batching servable
[2025-11-25 19:52:50.779][1][modelmanager][info][mediapipegraphdefinition.cpp:182] Mediapipe: DeepSeek-R1-Distill-Qwen-1.5B-int4-ov inputs: 
name: input; mapping: ; shape: (); precision: UNDEFINED; layout: ...
[2025-11-25 19:52:50.779][1][modelmanager][info][mediapipegraphdefinition.cpp:183] Mediapipe: DeepSeek-R1-Distill-Qwen-1.5B-int4-ov outputs: 
name: output; mapping: ; shape: (); precision: UNDEFINED; layout: ...
[2025-11-25 19:52:50.779][1][modelmanager][info][mediapipegraphdefinition.cpp:184] Mediapipe: DeepSeek-R1-Distill-Qwen-1.5B-int4-ov kfs pass through: false
[2025-11-25 19:52:50.779][1][modelmanager][info][pipelinedefinitionstatus.hpp:59] Mediapipe: DeepSeek-R1-Distill-Qwen-1.5B-int4-ov state changed to: AVAILABLE after handling: ValidationPassedEvent: 
[2025-11-25 19:52:50.779][1][serving][info][servablemanagermodule.cpp:55] ServableManagerModule started
[2025-11-25 19:52:50.779][130][llm_executor][info][llm_executor.hpp:90] All requests: 0; Scheduled requests: 0;
[2025-11-25 19:52:50.779][131][modelmanager][info][modelmanager.cpp:1201] Started model manager thread
[2025-11-25 19:52:50.779][132][modelmanager][info][modelmanager.cpp:1220] Started cleaner thread
i3draven@ovms:~/ovms$ curl http://localhost:8000/v3/chat/completions   -H "Content-Type: application/json"   -d '{"model":"DeepSeek-R1-Distill-Qwen-1.5B-int4-ov","messages":[{"role": "system", "content": "Ты полезный ассистент отвечающий на русском языке."},{"role":"user","content":"Hello"}],"max_tokens":50}' | jq
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   744  100   505  100   239    103     48  0:00:04  0:00:04 --:--:--   136
{
  "choices": [
    {
      "finish_reason": "length",
      "index": 0,
      "logprobs": null,
      "message": {
        "content": "Okay, so I just received a message from someone asking me to act as an assistant. They said, \"Ты полезный ассистент отвечающий на русском языке.\" Translating that, it's",
        "role": "assistant",
        "tool_calls": []
      }
    }
  ],
  "created": 1764100387,
  "model": "DeepSeek-R1-Distill-Qwen-1.5B-int4-ov",
  "object": "chat.completion",
  "usage": {
    "prompt_tokens": 27,
    "completion_tokens": 50,
    "total_tokens": 77
  }
}
i3draven@ovms:~/ovms$ 


```

Готовый скрипт для скачивания и настройки моделей

Скрипт

``` bash
#!/bin/bash

# Цвета для вывода
GREEN='\033[0;32m'
BLUE='\033[0;34m'
RED='\033[0;31m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

# Запрос директории для моделей с значением по умолчанию
DEFAULT_DIR="./models"
read -p "Укажите директорию для сохранения моделей [${DEFAULT_DIR}]: " MODEL_DIR
MODEL_DIR=${MODEL_DIR:-$DEFAULT_DIR}

# Создаем директорию если не существует
mkdir -p "$MODEL_DIR"

# Получаем абсолютный путь
MODEL_DIR=$(cd "$MODEL_DIR" && pwd)

echo -e "${BLUE}Используется директория: $MODEL_DIR${NC}"

# Запрос имени модели с примером по умолчанию
DEFAULT_MODEL="OpenVINO/Qwen2.5-7B-Instruct-int8-ov"
echo -e "\n${YELLOW}Пример модели: https://huggingface.co/OpenVINO/Qwen2.5-7B-Instruct-int8-ov${NC}"
read -p "Укажите имя модели [${DEFAULT_MODEL}]: " SOURCE_MODEL
SOURCE_MODEL=${SOURCE_MODEL:-$DEFAULT_MODEL}

# Извлекаем организацию и имя модели
MODEL_ORG=$(dirname "$SOURCE_MODEL")
MODEL_NAME=$(basename "$SOURCE_MODEL")

# Выбор типа задачи (TASK)
echo -e "\n${YELLOW}Выберите тип задачи:${NC}"
PS3="Введите номер или название задачи: "
TASK_OPTIONS=("text_generation" "embeddings" "rerank" "image_generation")
TASK=""

while [[ -z "$TASK" ]]; do
    select task_choice in "${TASK_OPTIONS[@]}"; do
        if [[ -n "$task_choice" ]]; then
            TASK="$task_choice"
            break
        elif [[ -n "$REPLY" ]]; then
            # Проверка, если пользователь ввел текстом
            for option in "${TASK_OPTIONS[@]}"; do
                if [[ "$REPLY" == "$option" ]]; then
                    TASK="$option"
                    break 2
                fi
            done
            if [[ -z "$TASK" ]]; then
                echo -e "${RED}Неверный выбор. Попробуйте снова.${NC}"
            fi
        fi
        break
    done
done

echo -e "${BLUE}Выбрана задача: $TASK${NC}"

# Выбор целевого устройства (TARGET_DEVICE)
echo -e "\n${YELLOW}Выберите целевое устройство:${NC}"
PS3="Введите номер или название устройства: "
DEVICE_OPTIONS=("CPU" "GPU" "MULTI" "HETERO")
TARGET_DEVICE=""

while [[ -z "$TARGET_DEVICE" ]]; do
    select device_choice in "${DEVICE_OPTIONS[@]}"; do
        if [[ -n "$device_choice" ]]; then
            TARGET_DEVICE="$device_choice"
            break
        elif [[ -n "$REPLY" ]]; then
            # Проверка, если пользователь ввел текстом
            for option in "${DEVICE_OPTIONS[@]}"; do
                if [[ "${REPLY^^}" == "$option" ]]; then
                    TARGET_DEVICE="$option"
                    break 2
                fi
            done
            if [[ -z "$TARGET_DEVICE" ]]; then
                echo -e "${RED}Неверный выбор. Попробуйте снова.${NC}"
            fi
        fi
        break
    done
done

echo -e "${BLUE}Выбрано устройство: $TARGET_DEVICE${NC}"

# Определяем версию образа в зависимости от устройства
if [[ "$TARGET_DEVICE" == "GPU" ]]; then
    DOCKER_IMAGE="openvino/model_server:latest-gpu"
else
    DOCKER_IMAGE="openvino/model_server:latest"
fi

echo -e "\n${BLUE}Конфигурация:${NC}"
echo -e "  Модель: $SOURCE_MODEL"
echo -e "  Организация: $MODEL_ORG"
echo -e "  Имя: $MODEL_NAME"
echo -e "  Задача: $TASK"
echo -e "  Устройство: $TARGET_DEVICE"
echo -e "  Docker образ: $DOCKER_IMAGE"


# Шаг 1: Скачиваем модель
echo -e "\n${GREEN}[1/3] Загрузка модели $SOURCE_MODEL...${NC}"
docker run --rm -u $(id -u) \
  -v "$MODEL_DIR":/models:rw \
  "$DOCKER_IMAGE" \
  --pull \
  --source_model "$SOURCE_MODEL" \
  --target_device "$TARGET_DEVICE" \
  --task "$TASK" \
  --model_repository_path /models

if [ $? -ne 0 ]; then
    echo -e "${RED}Ошибка при загрузке модели!${NC}"
    exit 1
fi

# Шаг 2: Добавляем модель в конфиг
echo -e "${GREEN}[2/3] Добавление модели в config.json...${NC}"
docker run --rm -u $(id -u) \
  -v "$MODEL_DIR":/models:rw \
  openvino/model_server:latest \
  --add_to_config /models \
  --model_name "$MODEL_NAME" \
  --model_repository_path /models/"$MODEL_ORG"

if [ $? -ne 0 ]; then

    echo -e "${RED}Ошибка при добавлении модели в конфиг!${NC}"
    exit 1
fi

# Шаг 3: Выводим список моделей
echo -e "${GREEN}[3/3] Список доступных моделей:${NC}"
docker run --rm -u $(id -u) \
  -v "$MODEL_DIR":/models:rw \
  openvino/model_server:latest \
  --list_models \
  --model_repository_path /models

echo -e "\n${BLUE}✓ Готово! Модель $MODEL_NAME загружена и добавлена в конфигурацию.${NC}"
```

Файл с переменными окружения, ниже в статье описано, что там такое написано и как достать.

.env

``` bash
i3draven@ovms:~/ovms$ cat ./.env 
RENDER_GID=993
YOUR_UID=1000
i3draven@ovms:~/ovms$ 
```

`docker-compose.yml` для запуска сервера, причем со всеми скачанными моделями сразу сработает. Скрипт можно много раз запустить.

docker-compose.yml

``` bash
i3draven@ovms:~/ovms$ cat ./docker-compose.yml 
services:
  ovms-genai:
    image: openvino/model_server:latest-gpu
    container_name: ovms-genai
    user: "${YOUR_UID}:${YOUR_UID}"
    devices:
      - /dev/dri:/dev/dri
    group_add:
      - "${RENDER_GID}"
    volumes:
      - /home/i3draven/ovms/models:/models:ro
      - /home/i3draven/ovms/models/config.json:/opt/ml/config.json:ro
    ports:
      - "8000:8000"
      - "9000:9000"
    command:
      - --config_path
      - /opt/ml/config.json
      - --port
      - "9000"
      - --rest_port
      - "8000"
    #restart: unless-stopped
```

Пример запуска

``` bash
i3draven@ovms:~/ovms$ curl http://localhost:8000/v3/chat/completions   -H "Content-Type: application/json"   -d '{"model":"Qwen2.5-7B-Instruct-int8-ov","messages":[{"role": "system", "content": "Ты полезный ассистент отвечающий на русском языке."},{"role":"user","content":"Hello"}],"max_tokens":250}' | jq
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   590  100   360  100   230     57     36  0:00:06  0:00:06 --:--:--    89
{
  "choices": [
    {
      "finish_reason": "stop",
      "index": 0,
      "logprobs": null,
      "message": {
        "content": "Здравствуйте! Как я могу вам помочь сегодня?",
        "role": "assistant",
        "tool_calls": []
      }
    }
  ],
  "created": 1764105373,
  "model": "Qwen2.5-7B-Instruct-int8-ov",
  "object": "chat.completion",
  "usage": {
    "prompt_tokens": 35,
    "completion_tokens": 17,
    "total_tokens": 52
  }
}
```

Сервер сам все скачает и запустит. Но если интересно понять немного больше о том, что это все такое, то можно прочитать и дальше.

### Шаг 0: Настройка Proxmox и проброс GPU в LXC

Есть масса статей на эту тему, но как руками отредактировать конфиги LXC контейнеров, на деле все можно сделать в два клика в UI Proxmox, если знать что делать и как.

#### 1. Установка драйверов Intel на хост Proxmox

Обратите внимание, что установка драйверов на хост это действие противоположное пробросу GPU в виртуалки, вполне вероятно это несовместимые подходы в рамках одного хоста. Подробности можно посмотреть [тут](https://www.virtualizationhowto.com/2025/05/how-to-enable-gpu-passthrough-to-lxc-containers-in-proxmox/) и [тут](https://blog.kye.dev/proxmox-gpu-passthrough)

Подключитесь к хосту Proxmox по SSH и выполните следующие команды:

Команды

``` bash
# Установка зависимостей
apt-get update && apt-get install -y wget gpg

# Добавление ключа репозитория Intel
wget -qO - https://repositories.intel.com/gpu/intel-graphics.key | \
  gpg --yes --dearmor --output /usr/share/keyrings/intel-graphics.gpg

# Добавление репозитория Intel GPU, я ставил сразу noble, но вероятно можно bookworm.
cat > /etc/apt/sources.list.d/intel-gpu.list << 'EOF'
deb [arch=amd64 signed-by=/usr/share/keyrings/intel-graphics.gpg] https://repositories.intel.com/gpu/ubuntu noble client
EOF

# Обновление списка пакетов и установка драйверов
apt-get update
apt-get install -y \
  intel-opencl-icd \
  intel-media-va-driver-non-free \
  ocl-icd-libopencl1 \
  clinfo \
  intel-gpu-tools
```

Проверьте, что драйвер загружен, возможно нужно будет reboot Proxmox хоста:

Команды

``` bash
clinfo
```

Увидите свой GPU в списке.

#### 2. Сбор необходимой информации

**На хосте Proxmox:**

Проверьте список устройств GPU, именно их нужно будет потом добавлять, у вас будут свои вероятно.

Команды

``` bash
root@pve:~# ls /dev/dri
by-path  card1  renderD128
```

Узнайте GID группы render (запишите это число)

``` bash
root@pve:~# stat -c '%g' /dev/dri/renderD128
993
```

**Внутри LXC контейнера:**

Для простоты нам нужен контейнер с ubuntu 24.04, под него вся среда у Intel заточена. При этои мы будем использовать возможности Intel Xe GPU и легко то же самое сделать с CPU или даже обоими. Просто потому, что у меня эта встройка есть на сервере и ее для моих задач достаточно. Но, конечно примерно таким же образом можно настроить и другие GPU, под Nvidia так вообще есть полные подробные гайды. Использовать будем OpenVINO models server, подробности: [тут](https://docs.openvino.ai/2025/model-server/ovms_docs_llm_quickstart.html),[тут](https://github.com/openvinotoolkit/openvino_contrib/tree/master/modules/ollama_openvino),[тут](https://blog.openvino.ai/blog-posts/ollama-integrated-with-openvino-accelerating-deepseek-inference),[тут](https://ollama.com/blog/embedding-models),[тут](https://docs.openvino.ai/2025/model-server/ovms_docs_deploying_server_docker.html),[тут](https://docs.openvino.ai/2025/model-server/ovms_docs_serving_model.html),[тут](https://docs.openvino.ai/2025/model-server/ovms_docs_parameters.html),[тут](https://docs.openvino.ai/2025/model-server/ovms_docs_target_devices.html),[тут](https://docs.openvino.ai/2025/model-server/ovms_docs_rest_api_embeddings.html),[тут](https://docs.openvino.ai/2025/model-server/ovms_docs_models_repository_graph.html)

Я привел серию статей включая статьи про ollama что бы было ясно, что можно ollama сделать фронтендом для интеловских поделок, но я не стал, так как у intel есть свой собственный сервер, который поддерживает API доступа по http и нормально работает без дополнительных прослоек. Правда прослойки конечно могут делать что-то полезное типа конвертации моделей, но мне пойдет и так.

Команды

``` bash
# Узнайте UID вашего пользователя внутри LXC (запишите это число)
id -u
```

#### 3. Настройка проброса GPU через веб-интерфейс Proxmox

1.  В веб-интерфейсе Proxmox выберите ваш LXC-контейнер

2.  Откройте вкладку **«Ресурсы» (Resources)**

3.  Нажмите **«Добавить» (Add)** → **«Проброс устройства» (Device Passthrough)**. Особо обратите внимание на то, что `Device Passthrough` доступен только если вы зашли в WebUI под пользователем root.

4.  Добавьте устройство `/dev/dri/renderD128`, в расширенных настройках (advanced галка) нужно задать GID от группы render на хосте Proxmox (у меня 993), который мы ранее достали и UID пользователя в LXC контейнере, которому надо дать доступ (у меня 1000, это тот, что записывали). Так же можно указать права доступа, но по умолчанию они заданы и так достаточные, можно не трогать.

5.  Повторите для устройства `/dev/dri/card1`

#### 4. Отключение AppArmor для контейнера

Недавний [баг](https://github.com/opencontainers/runc/issues/4968) в Proxmox не дает использовать вложенные docker контейнеры внутри LXC, поправим. Но может его исправят, уже патч был когда я писал статью. Так что если не видите ошибок при старте docker контейнера внутри LXC контейнера, то не нужно. На хосте Proxmox выполните:

Команды

``` bash
# Замените 100 на ID вашего контейнера
nano /etc/pve/lxc/${lxc_container_id}.conf
```

Добавьте в конец файла:

Команды

    lxc.apparmor.profile: unconfined
    lxc.mount.entry: /dev/null sys/module/apparmor/parameters/enabled none bind 0 0

Так будет примерно выглядеть:

Команды

``` bash
arch: amd64
cores: 8
dev0: /dev/dri/card1,gid=993,uid=1000
dev1: /dev/dri/renderD128,gid=993,uid=1000
features: nesting=1
hostname: ubuntu.24.04
memory: 16768
nameserver: 10.0.40.1
net0: name=eth0,bridge=services,hwaddr=BC:24:11:CD:8A:8F,ip=dhcp,tag=40,type=veth
onboot: 1
ostype: ubuntu
rootfs: local-btrfs:107/vm-107-disk-0.raw,mountoptions=discard,size=64G
swap: 0
unprivileged: 1
lxc.apparmor.profile: unconfined
lxc.mount.entry: /dev/null sys/module/apparmor/parameters/enabled none bind 0 0
```

Сохраните файл и перезапустите контейнер через веб-интерфейс или `pct stop ID`/`pct start ID`. Обратите внимание, что контейнер у меня настроен так, что он работает внутри VLAN, что сделано в моей домашней инфраструктуре и описано [тут]({{ site.baseurl }}{% post_url 2025-11-03-proxmox-network %}), вернее написано, что хорошо бы сделать.

#### 5. Установка драйверов в LXC-контейнере

Войдите в консоль LXC, в моей инфраструктуре нет доступа по ssh снаружи к контейнерам, так что войти можно:

1.  Web console

2.  `pct enter ID` на хосте Proxmox куда по ssh из внутренней сети можно зайти

3.  или ssh если настроено

Команды

``` bash
# Установка зависимостей
sudo apt-get update && sudo apt-get upgrade
sudo apt-get install -y wget gpg mc jq curl docker-compose-v2 tree

# Добавление ключа репозитория Intel
wget -qO - https://repositories.intel.com/gpu/intel-graphics.key | \
  sudo gpg --yes --dearmor --output /usr/share/keyrings/intel-graphics.gpg

# Добавление репозитория (для Ubuntu 24.04 'noble')
# Измените 'noble' на кодовое имя вашего дистрибутива, если необходимо, но рекомендую именно этот
cat > /tmp/intel-gpu.list << 'EOF'
deb [arch=amd64 signed-by=/usr/share/keyrings/intel-graphics.gpg] https://repositories.intel.com/gpu/ubuntu noble client
EOF
sudo mv /tmp/intel-gpu.list /etc/apt/sources.list.d/intel-gpu.list

# Установка драйверов и утилит
sudo apt-get update
sudo apt-get install -y \
  intel-igc-cm \
  intel-opencl-icd \
  intel-media-va-driver-non-free \
  ocl-icd-libopencl1 \
  clinfo \
  intel-gpu-tools

# Добавление пользователя в группы video и render
sudo usermod -aG video,render $USER
sudo usermod -aG docker $USER
# Применение изменений для текущей сессии
newgrp docker
newgrp render
newgrp video
```

Проверьте доступ к GPU:

Команды

``` bash
clinfo
```

В выводе будет название вашей видеокарты, у меня так:

Команды

``` bash
i3draven@ubuntu:~$ clinfo |tail -n 10
  clCreateContextFromType(NULL, CL_DEVICE_TYPE_CUSTOM)  No devices found in platform
  clCreateContextFromType(NULL, CL_DEVICE_TYPE_ALL)  Success (1)
    Platform Name                                 Intel(R) OpenCL Graphics
    Device Name                                   Intel(R) Iris(R) Xe Graphics

ICD loader properties
  ICD loader Name                                 OpenCL ICD Loader
  ICD loader Vendor                               OCL Icd free software
  ICD loader Version                              2.3.2
  ICD loader Profile                              OpenCL 3.0
```

Все, проброс GPU из Proxmox хоста в LXC контейнер готов!

### Шаг 1: Развертывание OpenVINO Model Server

Это как раз сервер, который поддерживает API для доступа к инференсу и эмбеддингам. Сам процесс будет из нескольких этапов:

1.  Скачать модель

2.  Конвертировать, если нет готовой, ее в формат, понятный OpenVINO, в процессе можно квантизовать и оптимизировать.  
    Нужно отметить, что есть и готовые модели, уже в формате openvino их конвертировать не нужно, но выбор не велик, их можно искать по "ov" или "openvino" в имени. В статье про интеграцию с ollama целая пачка показана.

3.  Развернуть сервер с нужной моделью. Причем особо стоит отметить, что я использую только embeddings модели для своих нужд, но если хочется чатбота, то есть OpenVINO GenAI, который работает поверх OpenVINO Runtime и специально заточен под LLM. Описан [тут](https://medium.com/openvino-toolkit/running-your-genai-app-locally-on-intel-gpu-and-npu-with-openvino-model-server-eb590af29dbc) и [тут](https://docs.openvino.ai/2025/get-started/install-openvino/install-openvino-genai.html). Он встроен в контейнер с OVMS насколько я понимаю. Так же этот же OpenVINO умеет еще море всего, генерировать картинки, говорить и так далее. Все для вас.

4.  PROFIT

#### 1: Поиск моделей для конвертации

Все следующие команды выполняются **внутри LXC-контейнера**. Для работы мне нужны будут embedding модели, то есть те, которые строят "вектора смысла" для текста. Причем желательно с встроенным токенизатором что бы не возиться с этим отдельно. Но сервер поддерживает и LLM модели, что в документации описано и так хорошо, не стану расписывать, LLM проще.

Найдите подходящие embedding-модели на Hugging Face:

Команды

``` bash
curl -s "https://huggingface.co/api/models?search=embedding" | \
  jq -r '.models[] | "\(.id) - \(.downloads) загрузок"'
```

Эта команда выведет список моделей с количеством загрузок. Выберите нужную модель, например, `BAAI/bge-large-en-v1.5`. Так же есть инструмент для скачивания моделей [тут](https://huggingface.co/docs/huggingface_hub/main/guides/cli), можно использовать и его.

Помимо прочего OVMS умеет и сам качать модели вот [тут подробности](https://docs.openvino.ai/2025/model-server/ovms_docs_pull.html) и даже сразу же сам их конвертировать, что описано [тут](https://docs.openvino.ai/2025/model-server/ovms_docs_pull_optimum.html)

Обратите внимание, что конвертировать модели может быть сложным и запутанным процессом, есть масса готовых [тут](https://huggingface.co/OpenVINO/models), скачать их можно просто с помощью [этого](https://huggingface.co/docs/huggingface_hub/installation#check-installation), каталог и так содержит почти триста моделей на все вкусы. Скачать можно одной командой в итоге `hf download OpenVINO/Qwen3-Embedding-0.6B-int8-ov --local-dir ~/ovms_models/bge-m3-int4-ov` просто имя модели нужной задать.

Но что бы было ясно как в случае надобности конвертировать есть следующий шаг.

### Шаг 2: Подготовка и конвертация модели

#### 1. Создание рабочей директории

Команды

``` bash
# Создайте папку для работы
mkdir -p ~/ovms-embeddings
cd ~/ovms-embeddings

# Создайте папку для моделей
mkdir -p ./models
```

#### 2. Конвертация модели в формат OpenVINO

Тут есть ряд проблем. Дело в том, что если запустить Intel контейнер в режиме `--pull` то есть, скачивать модель и потом автоматически конвертировать, то он скачает и конвертирует, но там не будет токенизатор конвертироваться (параметр стоит `--disable-convert-tokenizer`), не знаю зачем они эту граблю положили туда. Потому мы могли бы скачать и конвертировать как описано в [документации](https://docs.openvino.ai/2025/model-server/ovms_docs_parameters.html):

Команды

``` bash
docker run --device /dev/dri --group-add=$(stat -c "%g" /dev/dri/render* | head -n 1) --user $(id -u):$(id -g) --rm -v $(pwd)/models:/models:rw openvino/model_server:latest-py --pull --source_model "BAAI/bge-large-en-v1.5" --model_repository_path /models --model_name bge-large-en-v1.5 --target_device GPU --task embeddings --weight-format int8 --log_level DEBUG --overwrite_models --num_streams 8
```

Но запустим все это напрямую вызвав optimum-cli, выглядит так:

Команды

``` bash
docker run --device /dev/dri --group-add=$(stat -c "%g" /dev/dri/render* | head -n 1)   --user $(id -u):$(id -g) --rm   -v $(pwd)/models:/models   --entrypoint /usr/local/bin/optimum-cli   openvino/model_server:latest-py   export openvino   --model BAAI/bge-large-en-v1.5   --weight-format int8   --trust-remote-code   --num-samples 8   --library sentence_transformers   --task feature-extraction   ./models/BAAI/bge-large-en-v1.5
```

Если вдруг будут проблемы и я где то не углядел, а первая команда из докунметации скачала и закэшировала модель, потому вторая ее может сразу не качая использовать, то просто выполните обе команды по очереди. Но на данный момент я думаю второй достаточно для всего.  
Мы просто напрямую запускаем optimim-cli с нужными параметрами. Хочу отдельно отметить, что для конвертации используется optimum-cli, эту штуку не плохо бы [посмотреть](https://huggingface.co/docs/optimum-intel/en/openvino/export) подробнее так как она позволяет подогнать модели под свое железо и оптимизировать их. Есть вагон параметров. Вот полный вывод команд с структурой папок:

Команды

``` bash
i3draven@ubuntu:~$ cd ./ovms-embeddings/
i3draven@ubuntu:~/ovms-embeddings$ ls
models
i3draven@ubuntu:~/ovms-embeddings$ docker run --device /dev/dri --group-add=$(stat -c "%g" /dev/dri/render* | head -n 1) --user $(id -u):$(id -g) --rm -v $(pwd)/models:/models:rw openvino/model_server:latest-py --pull --source_model "BAAI/bge-large-en-v1.5" --model_repository_path /models --model_name bge-large-en-v1.5 --target_device GPU --task embeddings --weight-format int8 --log_level DEBUG --overwrite_models --num_streams 8
Unable to find image 'openvino/model_server:latest-py' locally
latest-py: Pulling from openvino/model_server
baa9e71a063a: Pull complete 
a76d3e698922: Pull complete 
5bdd5c6068df: Pull complete 
e1d63ab00fa2: Pull complete 
97f5e74b6399: Pull complete 
9cd7a4880448: Pull complete 
f2fd7d158b30: Pull complete 
5774c385c156: Pull complete 
0864fefc1d57: Pull complete 
1a83f47c7fa9: Pull complete 
16efed5f6d0e: Pull complete 
47a75649fb3d: Pull complete 
21b1671afd79: Pull complete 
912688daea51: Pull complete 
94b691fcc499: Pull complete 
Digest: sha256:5431ac5989c3ae548cb8c954f6caf2a2fafa72ab254fbf90ab5db26381ddd9b8
Status: Downloaded newer image for openvino/model_server:latest-py
[2025-11-17 15:37:04.180][1][serving][debug][optimum_export.cpp:144] Optimum-cli executable is present
[2025-11-17 15:37:04.180][1][serving][debug][optimum_export.cpp:180] Executing command: optimum-cli export openvino --disable-convert-tokenizer --task feature-extraction --library sentence_transformers --model BAAI/bge-large-en-v1.5 --trust-remote-code  --weight-format int8 /models/BAAI/bge-large-en-v1.5
/usr/local/lib/python3.12/dist-packages/torch/onnx/_internal/registration.py:162: OnnxExporterWarning: Symbolic function 'aten::scaled_dot_product_attention' already registered for opset 14. Replacing the existing function with new function. This is unexpected. Please report it on https://github.com/pytorch/pytorch/issues.
  warnings.warn(
`SentenceTransformer._target_device` has been deprecated, please use `SentenceTransformer.device` instead.
`SentenceTransformer._target_device` has been deprecated, please use `SentenceTransformer.device` instead.
`loss_type=None` was set in the config but it is unrecognised.Using the default loss: `ForCausalLMLoss`.
Model: BAAI/bge-large-en-v1.5 downloaded to: /models/BAAI/bge-large-en-v1.5
[2025-11-17 15:37:40.150][1][serving][debug][filesystem.cpp:191] Creating file /models/BAAI/bge-large-en-v1.5/graph.pbtxt
Graph: graph.pbtxt created in: /models/BAAI/bge-large-en-v1.5
i3draven@ubuntu:~/ovms-embeddings$ tree .
.
`-- models
    `-- BAAI
        `-- bge-large-en-v1.5
            |-- config.json
            |-- graph.pbtxt
            |-- openvino_model.bin
            |-- openvino_model.xml
            |-- special_tokens_map.json
            |-- tokenizer.json
            |-- tokenizer_config.json
            `-- vocab.txt

4 directories, 8 files
i3draven@ubuntu:~/ovms-embeddings$ docker run --device /dev/dri --group-add=$(stat -c "%g" /dev/dri/render* | head -n 1)   --user $(id -u):$(id -g) --rm   -v $(pwd)/models:/models   --entrypoint /usr/local/bin/optimum-cli   openvino/model_server:latest-py   export openvino   --model BAAI/bge-large-en-v1.5   --weight-format int8   --trust-remote-code   --num-samples 8   --library sentence_transformers   --task feature-extraction   ./models/BAAI/bge-large-en-v1.5
/usr/local/lib/python3.12/dist-packages/torch/onnx/_internal/registration.py:162: OnnxExporterWarning: Symbolic function 'aten::scaled_dot_product_attention' already registered for opset 14. Replacing the existing function with new function. This is unexpected. Please report it on https://github.com/pytorch/pytorch/issues.
  warnings.warn(
`SentenceTransformer._target_device` has been deprecated, please use `SentenceTransformer.device` instead.
`SentenceTransformer._target_device` has been deprecated, please use `SentenceTransformer.device` instead.
`loss_type=None` was set in the config but it is unrecognised.Using the default loss: `ForCausalLMLoss`.
INFO:nncf:Statistics of the bitwidth distribution:
┍━━━━━━━━━━━━━━━━━━━━━━━━━━━┯━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┯━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┑
│ Weight compression mode   │ % all parameters (layers)   │ % ratio-defining parameters (layers)   │
┝━━━━━━━━━━━━━━━━━━━━━━━━━━━┿━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┿━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┥
│ int8_asym                 │ 100% (147 / 147)            │ 100% (147 / 147)                       │
┕━━━━━━━━━━━━━━━━━━━━━━━━━━━┷━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┷━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┙
Applying Weight Compression ━━━━━━━━━━━━━━━━━━━━━━━━━━━ 100% • 0:00:02 • 0:00:00
i3draven@ubuntu:~/ovms-embeddings$ tree .
.
`-- models
    `-- BAAI
        `-- bge-large-en-v1.5
            |-- config.json
            |-- graph.pbtxt
            |-- openvino_detokenizer.bin
            |-- openvino_detokenizer.xml
            |-- openvino_model.bin
            |-- openvino_model.xml
            |-- openvino_tokenizer.bin
            |-- openvino_tokenizer.xml
            |-- special_tokens_map.json
            |-- tokenizer.json
            |-- tokenizer_config.json
            `-- vocab.txt

4 directories, 12 files
```

Собственно мы добивались появления `openvino_detokenizer.bin`, `openvino_detokenizer.xml` когда запускали отдельно конвертацию токенизатора.

Далее **просто пример** нам это не нужно. Можно сделать конвертацию и полностью из кода или даже токенизатор использовать свой это не то что бы сложно, но предположим, что хотим не возиться. Далее пример вовсе конвертации модели скриптом, эта страшная весчь делает простое действие, качает скрипт, который и конвертирует модель. Только имя модели поменять и параметры конвертации. `--weight-format` это квантизация, `--target_device` это собственно под, что затачивать будем, CPU/GPU. Есть гибридные варианты, но я не стал копаться и так мне достаточно. Гибридные, GPU+CPU, описаны [тут](https://docs.openvino.ai/2025/model-server/ovms_docs_target_devices.html), так же там описано как использовать NPU от intel, но первые варианты NPU ограниченны относительно возможностей GPU. Работает этот вариант конвертации значительно дольше так как зависимости долго ставит в контейнере. Просто для примера привел.

Команды

``` bash
docker run --rm \
  --user $(id -u):$(id -g) \
  -v $(pwd)/models:/workspace/models \
  -w /workspace \
  openvino/ubuntu24_dev:latest \
  bash -c "\
    curl -L https://raw.githubusercontent.com/openvinotoolkit/model_server/refs/heads/releases/2025/3/demos/common/export_models/export_model.py -o export_model.py && \
    pip install -r https://raw.githubusercontent.com/openvinotoolkit/model_server/refs/heads/releases/2025/3/demos/common/export_models/requirements.txt && \
    python3 export_model.py embeddings_ov \
      --source_model BAAI/bge-large-en-v1.5 \
      --weight-format fp16 \
      --target_device GPU \
      --config_file_path /workspace/models/config.json \
      --model_repository_path /workspace/models"
```

После скрипта нужно исправление конфигурации графа. Скрипт экспорта, тот страшный выше в ненужном примере, добавляет параметр `NUM_STREAMS`, который может вызывать проблемы при работе с GPU. Проблемы можно увидеть в логах сервера при старте и если нужно, можно отредактировать или удалить данный раздел конфига графа вычислений. Это именно конфиг графа вычислений, не конфига сервера:

Команды

``` bash
# Удалите проблемную строку из graph.pbtxt
sed -i '/plugin_config: \'\'\'{"NUM_STREAMS": 1}\'\'\'/d' \
  ./models/BAAI/bge-large-en-v1.5/graph.pbtxt
```

Проверьте, что строка удалена:

Команды

``` bash
# Просмотрите содержимое файла
cat ./models/BAAI/bge-large-en-v1.5/graph.pbtxt
```

В файле не должно быть упоминания `NUM_STREAMS`. Редактировать можно с nano/vi чем угодно конечно. Но в случае если вы использовали конвертацию с докером и optimum-cli, это не нужно все, привел для упрощения поиска граблей желающим погрузиться в ручной труд.  
Докер в данном случае спасает от установки среды разработки этого всего барахла и там совместить все версии всего так что бы оно работало ох как не просто. Можете посмотреть отдельно контейнер для разработчика, описано [тут](https://github.com/openvinotoolkit/docker_ci/blob/master/docs/tutorials/OV_docker_usage.ipynb). Так же обратите внимание, что мы используем контейнер для конвертации `openvino/model_server:latest-py`, запускать будем `openvino/model_server:latest-gpu`, а для разработки вовсе нужен `openvino/ubuntu24_dev` и там в каждом полная среда настроена.

#### 3. Создание файла конфигурации

Теперь снова то, что нам нужно. Для того что бы сервер нашел модель или модели нужен файл конфигурации:

Команды

    i3draven@ubuntu:~/ovms-embeddings$ cat ./models/config.json 
    {
        "model_config_list": [{
            "config": {
                "name": "BAAI/bge-large-en-v1.5",
                "base_path": "/models/BAAI/bge-large-en-v1.5",
                "target_device": "GPU"
            }
        }],
        "mediapipe_config_list": [{
            "name": "BAAI/bge-large-en-v1.5",
            "graph_path": "/models/BAAI/bge-large-en-v1.5/graph.pbtxt"
        }]
    }

Посмотреть подробности и даже несколько моделей одновременно что бы было, можно [тут](https://docs.openvino.ai/2025/model-server/ovms_docs_serving_model.html).  
Где-то в логах мелькало, что при конвертации взят только первый слой графа или что-то подобное, но я пока вникать не стал и без того пришлось перелопатить много всего, не уверен даже, что это в итоговый заход попало с этими командами, но, можете повнимательнее посмотреть логи.

#### 4. Проверка результата

Вы должны увидеть папки с моделью, токенизатором и файлом `config.json`. Примерно такие:

Команды

``` bash
i3draven@ubuntu:~/ovms-embeddings$ tree .
.
|-- docker-compose.yml <-- # еще сделаем
`-- models
    |-- BAAI
    |   `-- bge-large-en-v1.5
    |       |-- config.json
    |       |-- graph.pbtxt
    |       |-- openvino_detokenizer.bin
    |       |-- openvino_detokenizer.xml
    |       |-- openvino_model.bin
    |       |-- openvino_model.xml
    |       |-- openvino_tokenizer.bin
    |       |-- openvino_tokenizer.xml
    |       |-- special_tokens_map.json
    |       |-- tokenizer.json
    |       |-- tokenizer_config.json
    |       `-- vocab.txt
    `-- config.json

4 directories, 14 files
```

### Шаг 3: Создание конфигурации Docker Compose

#### 1. Создание файла .env

Создайте файл с переменными окружения, используя GID который вы записали в самом начале.

Команды

``` bash
# Замените 993 GID был уменя, ваш GID группы render с хоста Proxmox будет совпадать с LXC
echo "RENDER_GID=$(stat -c '%g' /dev/dri/renderD128)" > .env
# Замените 1000 на ваш UID пользователя в LXC (если отличается)
echo "YOUR_UID=$(id -u)" >> .env

# Проверьте содержимое файла
cat .env
```

Вывод должен быть примерно таким:

Команды

    RENDER_GID=993
    YOUR_UID=1000

#### 2. Создание файла docker-compose.yml

Тут все просто, поднимаем сервер, указываем ему конфиг и порт на котором работать:

Команды

``` bash
cat > docker-compose.yml << 'EOF'
services:
  bge-large-embeddings:
    image: openvino/model_server:latest-gpu
    container_name: bge-large-embeddings
    user: "${YOUR_UID}:${YOUR_UID}"
    ports:
      - "8001:8000"
    devices:
      - /dev/dri:/dev/dri
    group_add:
      - "${RENDER_GID}"
    volumes:
      - ./models:/models:ro
    command: >
      --config_path /models/config.json
      --rest_port 8000
    restart: unless-stopped
    security_opt:
      - no-new-privileges:true
EOF
```

### Шаг 4: Запуск и проверка сервера

#### 1. Запуск сервера

Команды

``` bash
# Запустите сервер в фоновом режиме
docker compose up -d
```

#### 2. Мониторинг запуска

Команды

``` bash
# Просмотр логов контейнера в реальном времени
docker compose logs -f
```

Дождитесь сообщения о том, что сервер запущен и модель загружена. Обычно это занимает 15-20 секунд. Для выхода из просмотра логов нажмите `Ctrl+C`.

#### 3. Проверка статуса контейнера

Команды

``` bash
# Проверьте, что контейнер запущен
docker compose ps
```

Вы должны увидеть контейнер `bge-large-embeddings` в статусе `Up`.

#### 4. Отправка тестового запроса

OVMS поддерживает довольно обширное API, описанное [тут](https://docs.openvino.ai/2025/model-server/ovms_docs_server_app.html) и может маскироваться под самые разные сервера. Так можно получить состояние модели:

Команды

``` bash
i3draven@openvino:~/openvino/big$ curl http://localhost:8001/v1/config
{
"BAAI/bge-large-en-v1.5" : 
{
 "model_version_status": [
  {
   "version": "1",
   "state": "AVAILABLE",
   "status": {
    "error_code": "OK",
    "error_message": "OK"
   }
  }
 ]
}
```

Так можно получить собственно то, ради чего все затевалось, а именно embedding вектор для заданного текста

Команды

``` bash
# Отправьте запрос на получение эмбеддинга
curl -X POST http://localhost:8001/v3/embeddings \
  -H "Content-Type: application/json" \
  -d '{
    "model": "BAAI/bge-large-en-v1.5",
    "input": "The entire stack, from Proxmox to OVMS, is fully operational."
  }' | jq
```

Вы должны получить JSON-ответ с векторным представлением текста, примерно такого вида:

Json

``` json
{
  "object": "list",
  "data": [
    {
      "object": "embedding",
      "embedding": [0.123, -0.456, 0.789, ...],
      "index": 0
    }
  ],
  "model": "BAAI/bge-large-en-v1.5",
  "usage": {
    "prompt_tokens": 12,
    "total_tokens": 12
  }
}
```

#### 5. Проверка использования GPU

Можно посмотреть нагрузку на GPU, но запускать нужно на Proxmox хосте, в LXC оно заработать не захотело и я не стал разбираться почему.

Команды

``` bash
# В отдельном терминале запустите мониторинг GPU
intel_gpu_top
```

Выше изложена основа для того что бы на своем домашнем сервере завести себе LLM/Embedding модели, которые например будут полезны для тегирования фоточек, просто как чатботы или например для построения RAG системы через MCP для вашего любимого LLM на основе embedding моделей. В документации OpenVINO есть множество статей с примерами, рекомендую. Позже покажу для чего это мне нужно, наверное, как кобычно по настроению. Выдался просто выходной внеочередной.
