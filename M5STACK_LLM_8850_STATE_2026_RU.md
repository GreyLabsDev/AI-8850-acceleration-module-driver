# M5Stack LLM-8850 / AX8850: состояние поддержки и практическое руководство

> Актуальность: 24 марта 2026 года. Документ ориентирован на M5Stack LLM-8850 Card/Kit с 8 ГБ памяти и Raspberry Pi 5, но большая часть сведений применима к другим Linux-хостам с PCIe.

## Краткий вывод

LLM-8850 уже не экспериментальная плата без программной экосистемы. Для нее есть актуальный PCIe-стек AXCL, C/C++ и Python API, специализированный LLM/VLM-рантайм, OpenAI-совместимый сервер, готовые модели компьютерного зрения, распознавания речи, синтеза речи и генерации изображений. AXERA и M5Stack регулярно добавляют новые порты.

Главные ограничения остаются существенными:

- это закрытый NPU-стек с собственным форматом `.axmodel`, а не CUDA/ROCm;
- произвольную модель нельзя запустить непосредственно из PyTorch, ONNX или GGUF: ее нужно портировать либо найти готовую сборку;
- память карты отделена от RAM Raspberry Pi; для 8-гигабайтной версии реально доступны примерно 7 ГиБ CMM, часть памяти занята системой и рантаймом;
- совместимость конкретной архитектуры и операции важнее заявленных 24 TOPS;
- независимое сообщество пока невелико, большинство готовых портов сделано AXERA, M5Stack или близкими контрибьюторами;
- опубликованные скорости обычно измерены самой AXERA/M5Stack и не всегда являются полным end-to-end тестом на Pi 5.

Для 8 ГБ наиболее практичный набор на сегодня:

| Задача | Первый выбор | Почему |
|---|---|---|
| Универсальная локальная LLM | [Qwen3.5-4B Int4](https://huggingface.co/AXERA-TECH/Qwen3.5-4B-GPTQ-Int4) | Наиболее сильная из свежих моделей, которая укладывается в карту; около 5.7 ток/с в опубликованном тесте |
| Быстрая LLM/агент | [Qwen3-1.7B Int4](https://huggingface.co/AXERA-TECH/Qwen3-1.7B-GPTQ-Int4) | Хороший баланс качества, скорости и запаса памяти; около 12.7 ток/с |
| Минимальная задержка | [Qwen3-0.6B Int4](https://huggingface.co/AXERA-TECH/Qwen3-0.6B-GPTQ-Int4) | Около 20.2 ток/с, удобна для классификации, команд и простых агентов |
| Изображения и OCR | [Qwen3.5-2B VLM Int4](https://huggingface.co/AXERA-TECH/Qwen3.5-2B-GPTQ-Int4) или [Qwen3-VL-2B](https://huggingface.co/AXERA-TECH/Qwen3-VL-2B-Instruct) | 2B дает лучший рабочий баланс; Qwen3.5 в тесте около 13.4 ток/с |
| Видео с малой задержкой | [FastVLM-0.5B](https://huggingface.co/AXERA-TECH/FastVLM-0.5B) или [SmolVLM2-500M](https://huggingface.co/AXERA-TECH/SmolVLM2-500M-Video-Instruct) | Компактные VLM, рассчитанные на edge/video сценарии |
| Русская речь в текст | [Whisper Small](https://huggingface.co/M5Stack/whisper-small-axmodel) | Поддерживает русский и заметно точнее Tiny/Base; для длинной речи нужен внешний VAD и сегментация |
| Максимальная скорость ASR | [Whisper Turbo](https://huggingface.co/AXERA-TECH/whisper-turbo) | Самая производительная готовая Whisper-сборка, но требует проверки памяти и качества на своем аудио |
| Синтез речи | [CosyVoice2](https://docs.m5stack.com/en/guide/ai_accelerator/llm-8850/m5_llm_8850_cosy_voice2) | Наиболее функциональный готовый TTS-порт, есть API-обвязка |
| Генерация изображений | [LCM-LoRA SD 1.5](https://huggingface.co/AXERA-TECH/lcm-lora-sdv1-5) | Четыре шага UNet занимают около 1.74 с; практичнее обычной SD 1.5 |
| Детекция/видеонаблюдение | [YOLO11](https://docs.m5stack.com/en/guide/ai_accelerator/llm-8850/m5_llm_8850_yolo11) | Зрелый и быстрый сценарий; есть интеграция с Frigate |

## Что представляет собой ускоритель

M5Stack LLM-8850 построен на SoC AXERA AX8850. В документации и выводе программ он часто называется `AX650N`, а при компиляции моделей применяется цель `AX650`. Это ожидаемое наследование имен внутри SDK, а не признак неверной платы.

Основные характеристики:

- до 24 TOPS INT8;
- NPU с тремя вычислительными ядрами;
- восемь Cortex-A55 до 1.7 ГГц на самой карте;
- 8 ГБ LPDDR4x в рассматриваемой версии;
- M.2 2242, интерфейс PCIe 2.0 x2;
- аппаратные видео- и мультимедийные блоки.

В PCIe-сценарии Raspberry Pi является хостом, а карта отдельным вычислительным устройством. RAM Pi не расширяет память NPU. Вес модели, KV-кеш LLM, входы/выходы и рабочие буферы должны помещаться в DDR карты. Поэтому «4B Int4 занимает примерно 2 ГБ» недостаточно для оценки: фактическое потребление может быть значительно выше из-за контекста и буферов.

## Аппаратная установка на Raspberry Pi 5

Перед подключением полностью обесточьте Raspberry Pi. Горячая установка карты запрещена.

### Питание

- С фирменной LLM-8850 PiHat вся система питается через PD-вход платы. Нужен источник мощнее `9 В x 3 А`, то есть не менее 27 Вт.
- С официальной Raspberry Pi M.2 HAT+ M5Stack рекомендует импульсный источник `5 В x 3 А` без PD; из-за согласования некоторых PD-блоков возможна нехватка мощности.
- Для длительного LLM, VLM, ASR или SD-инференса нужно активное охлаждение Raspberry Pi и обдув карты.
- Нестабильное питание может выглядеть как ошибка PCIe, драйвера или внезапный сбой инференса.

Официальная инструкция с фотографиями: [LLM-8850 Card Hardware Installation](https://docs.m5stack.com/en/guide/ai_accelerator/llm-8850/m5_llm_8850_hardware_install).

## Установка AXCL

Поддерживаемые хосты включают aarch64 и x86_64 под Ubuntu/Debian, Raspberry Pi 5 и Windows. macOS, WSL и виртуальные машины официально не поддерживаются. Для Pi 5 самый прямой путь — 64-битная Raspberry Pi OS/Debian.

### 1. Обновить систему и EEPROM

```bash
sudo apt update && sudo apt full-upgrade
sudo rpi-eeprom-update
```

Если EEPROM старее 6 декабря 2023 года, в `sudo raspi-config` выберите `Advanced Options` → `Bootloader Version` → `Latest`, затем:

```bash
sudo rpi-eeprom-update -a
sudo reboot
```

После перезагрузки карта должна быть видна как устройство AXERA:

```bash
lspci | grep -i -E 'axera|0650'
```

Ожидаемая строка содержит `Axera Semiconductor ... Device 0650`.

### 2. Установить драйвер и AXCL host runtime

Для DKMS нужны `gcc`, `make`, `patch` и заголовки текущего ядра. Официальный репозиторий M5Stack устанавливается так:

```bash
sudo apt install -y gcc make patch linux-headers-$(uname -r) dkms
sudo wget -qO /etc/apt/keyrings/StackFlow.gpg \
  https://repo.llm.m5stack.com/m5stack-apt-repo/key/StackFlow.gpg
echo 'deb [signed-by=/etc/apt/keyrings/StackFlow.gpg] https://repo.llm.m5stack.com/m5stack-apt-repo axclhost main' \
  | sudo tee /etc/apt/sources.list.d/axclhost.list
sudo apt update
sudo apt install -y axclhost
source /etc/profile
```

Проверка:

```bash
axcl-smi
```

Команда должна показать карту, температуру, загрузку CPU/NPU, CMM и процессы. На странице документации в марте 2026 года еще встречается старый пример `V3.6.4`, но актуальная ветка документации AXCL — `V3.16.0`. Для свежего `ax-llm` нужен SDK не ниже 3.16.0. Проверяйте реально установленную версию, а не снимок экрана из руководства.

Полезная диагностика:

```bash
axcl-smi
watch -n 1 axcl-smi
dmesg | grep -i -E 'axcl|axera|pcie'
```

Официальные источники: [установка M5Stack](https://docs.m5stack.com/en/guide/ai_accelerator/llm-8850/m5_llm_8850_software_install), [AXCL Quick Start](https://axcl-docs.readthedocs.io/en/latest/doc_guide_quick_start.html), [документация AXCL V3.16.0](https://axcl-docs.readthedocs.io/en/latest/).

### 3. Первый тест NPU

```bash
wget https://m5stack.oss-cn-shenzhen.aliyuncs.com/resource/linux/ax8850_card/yolo11s.axmodel
axcl_run_model -m yolo11s.axmodel -r 10
```

В опубликованном примере M5Stack чистый прогон модели занимает в среднем около 3.4 мс. Это проверка NPU-графа без полного декодирования, препроцессинга и постпроцессинга приложения.

## Способы работы

### Готовые приложения StackFlow

Это самый простой путь: APT-пакеты M5Stack устанавливают сервис и модели. Например, локальный OpenAI-совместимый сервер:

```bash
sudo apt install lib-llm llm-sys llm-llm llm-openai-api
sudo apt install llm-model-qwen3-1.7b-int8-ctx-axcl
sudo systemctl restart llm-openai-api
curl http://127.0.0.1:8000/v1/models
```

После установки каждой новой модели сервис нужно перезапускать. Пример запроса:

```bash
curl http://127.0.0.1:8000/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer sk-local' \
  -d '{
    "model": "qwen3-1.7B-Int8-ctx-axcl",
    "messages": [{"role": "user", "content": "Кратко объясни, что такое NPU"}]
  }'
```

Инструкция: [M5Stack OpenAI API](https://docs.m5stack.com/en/guide/ai_accelerator/llm-8850/m5_llm_8850_openai).

### ax-llm для LLM и VLM

[AXERA-TECH/ax-llm](https://github.com/AXERA-TECH/ax-llm) — основной современный рантайм. Он поддерживает AX8850 через AXCL aarch64, интерактивный CLI, OpenAI-совместимый сервер, изображения/видео для VLM, управление контекстом и памятью.

```bash
git clone https://github.com/AXERA-TECH/ax-llm.git
cd ax-llm
./install.sh
./build_axcl_aarch64.sh

axllm run /path/to/model_directory
axllm serve /path/to/model_directory --port 8000
```

Используйте инструкции и параметры именно из версии репозитория, которой соответствуют скачанные модели. Старые каталоги моделей могут содержать отдельные `.axmodel` по слоям и конфигурацию, поэтому один файл нельзя произвольно подставить в любую версию рантайма.

### C/C++ AXCL API

Низкоуровневый путь подходит для минимальной задержки и сложных мультимедийных пайплайнов:

- инициализация AXCL и выбор устройства;
- выделение device/host памяти и DMA;
- загрузка `.axmodel`, создание execution context;
- копирование входа, запуск, чтение выхода;
- аппаратный decode/encode, IVPS, IVE и другие блоки.

Начинать лучше с [AXERA-TECH/axcl-samples](https://github.com/AXERA-TECH/axcl-samples), где LLM-8850 и Raspberry Pi 5 указаны явно, и с [AXERA-TECH/axcl-pi5-examples](https://github.com/AXERA-TECH/axcl-pi5-examples). Справочник API: [AXCL SDK API](https://axcl-docs.readthedocs.io/en/latest/doc_guide_axcl_api.html).

### Python

[AXERA-TECH/pyaxcl](https://github.com/AXERA-TECH/pyaxcl) предоставляет Python-обертки для runtime, NPU-инференса, codec, IVPS, IVE и DMA. Для некоторых готовых моделей используется [pyaxengine](https://github.com/AXERA-TECH/pyaxengine) с `AXCLRTExecutionProvider`.

Python удобен для оркестрации и препроцессинга, но критические циклы лучше оставлять в NPU/C++ и не гонять промежуточные тензоры через PCIe после каждого слоя.

## LLM: актуальные готовые модели

Ниже `tok/s` — опубликованная AXERA скорость decode/generation для соответствующей сборки. Она полезна для сравнения моделей между собой, но не включает одинаково все расходы токенизации, prefill, сетевого API и конкретного хоста.

| Модель | Квантование | Опубликованная скорость | Оценка для 8 ГБ | Применение |
|---|---:|---:|---|---|
| [Qwen3-0.6B](https://huggingface.co/AXERA-TECH/Qwen3-0.6B-GPTQ-Int4) | Int4 | 20.21 ток/с | Отлично | Команды, классификация, простые агенты |
| [Qwen3-1.7B](https://huggingface.co/AXERA-TECH/Qwen3-1.7B-GPTQ-Int4) | Int4 | 12.72 ток/с | Лучший баланс скорости | Чат, русский язык, extraction, tool calling |
| [Qwen3-4B](https://huggingface.co/AXERA-TECH/Qwen3-4B-GPTQ-Int4) | Int4 | 6.50 ток/с | Работает, контекст контролировать | Более качественный чат и рассуждение |
| [Qwen3.5-0.8B](https://huggingface.co/AXERA-TECH/Qwen3.5-0.8B-GPTQ-Int4) | Int4 | 23.8 ток/с | Отлично | Свежая быстрая text/VLM-архитектура |
| [Qwen3.5-2B](https://huggingface.co/AXERA-TECH/Qwen3.5-2B-GPTQ-Int4) | Int4 | 13.4 ток/с | Рекомендуется | Свежий универсальный баланс |
| [Qwen3.5-4B](https://huggingface.co/AXERA-TECH/Qwen3.5-4B-GPTQ-Int4) | Int4 | 5.7 ток/с | Лучшее качество из практичных | Универсальный ассистент, умеренный контекст |
| [Qwen2.5-1.5B-Instruct](https://huggingface.co/AXERA-TECH/Qwen2.5-1.5B-Instruct) | Int4/Int8 варианты | зависит от сборки | Зрелый запасной вариант | Стабильный мультиязычный instruct |
| [DeepSeek-R1-Distill-Qwen-1.5B](https://huggingface.co/AXERA-TECH/DeepSeek-R1-Distill-Qwen-1.5B) | Int4 | зависит от сборки | Работает, ответы многословны | Компактное reasoning, не для малой задержки |
| [MiniCPM4-0.5B](https://huggingface.co/AXERA-TECH/MiniCPM4-0.5B) | оптимизированная сборка | зависит от сборки | Очень легко | Edge-агент с низким расходом памяти |
| [SmolLM2-1.7B](https://huggingface.co/AXERA-TECH/SmolLM2-1.7B-Instruct) | Int4 | зависит от сборки | Хорошо | Англоязычные компактные задачи |
| [TinyLlama-1.1B](https://huggingface.co/AXERA-TECH/TinyLlama-1.1B-Chat-v1.0) | Int4 | зависит от сборки | Легко | Старый baseline, не основной выбор |

Полный регулярно обновляемый каталог и таблицы производительности находятся в [AXERA-TECH/awesome-docs](https://github.com/AXERA-TECH/awesome-docs). Перед скачиванием проверяйте, что в карточке есть AX650/AX8850/AXCL-вариант, а не только AX630C/AX620E/AX650 SoC.

### Как выбрать LLM для 8 ГБ

1. Начните с Qwen3-1.7B Int4 как с контрольной модели.
2. Для качества сравните ее с Qwen3.5-4B Int4 на своих 20–50 типичных запросах.
3. Для голосового ассистента и автоматизации предпочитайте 0.6B–2B: задержка важнее небольшого выигрыша качества.
4. Для RAG держите документы и embedding-индекс на Pi, а на NPU отправляйте только сформированный контекст.
5. Ограничивайте контекст. KV-кеш растет с длиной истории и способен вытеснить модель, которая формально помещалась.
6. Не выбирайте Int8 автоматически: на 8 ГБ Int4 часто дает лучший общий результат за счет доступного контекста.

## VLM: изображения, OCR и видео

| Модель | Размер/скорость | Сильная сторона | Рекомендация |
|---|---|---|---|
| [Qwen3.5-0.8B/2B/4B](https://huggingface.co/AXERA-TECH) | около 23.8/13.4/5.7 ток/с Int4 | Самое свежее семейство, текст + изображение | 2B как основной баланс, 4B для качества |
| [Qwen3-VL-2B-Instruct](https://huggingface.co/AXERA-TECH/Qwen3-VL-2B-Instruct) | 2B | OCR, документы, общий visual QA | Практичный зрелый выбор |
| [Qwen3-VL-4B Int4](https://huggingface.co/AXERA-TECH/Qwen3-VL-4B-Instruct-GPTQ-Int4) | 4B | Более точное описание и OCR | Использовать с коротким контекстом |
| [Qwen2.5-VL-3B](https://huggingface.co/AXERA-TECH/Qwen2.5-VL-3B-Instruct) | 3B | Проверенная VLM | Хороший запасной вариант |
| [InternVL3-1B](https://huggingface.co/AXERA-TECH/InternVL3-1B) | 1B | Компактное visual QA | Когда важна скорость и память |
| [FastVLM-0.5B](https://huggingface.co/AXERA-TECH/FastVLM-0.5B) | 0.5B | Низкая задержка | Камера и частые кадры |
| [SmolVLM2-500M](https://huggingface.co/AXERA-TECH/SmolVLM2-500M-Video-Instruct) | 0.5B | Видео и малый размер | Edge video demo/аналитика |
| [MiniCPM-V-2.6](https://huggingface.co/AXERA-TECH/MiniCPM-V-2_6) | крупнее | Сильный visual QA | Проверять память конкретной сборки |
| [Janus-Pro-1B](https://huggingface.co/AXERA-TECH/Janus-Pro-1B) | 1B | Современное мультимодальное понимание | Интересный порт, менее зрелый путь |
| [LibCLIP](https://docs.m5stack.com/en/guide/ai_accelerator/llm-8850/m5_llm_8850_clip) | embedding | Поиск похожих изображений | Полезнее VLM для Immich/каталогов |

Для непрерывного видео не стоит запускать тяжелую VLM на каждом кадре. Более эффективный пайплайн: аппаратный декодер → YOLO/трекер → выбор интересного кадра → VLM. Это снижает PCIe-трафик и дает приемлемую задержку.

## Speech-to-text и аудио

### Whisper

Готовая экосистема состоит из двух уровней:

- [ml-inory/whisper.axcl](https://github.com/ml-inory/whisper.axcl) — исходный community-порт, сборка и конвертация;
- [AXERA-TECH/ax_asr_api](https://github.com/AXERA-TECH/ax_asr_api) — готовый ASR-стек с Whisper Tiny, Base, Small, Turbo и SenseVoice.

M5Stack также публикует готовые [Tiny](https://huggingface.co/M5Stack/whisper-tiny-axmodel), [Base](https://huggingface.co/M5Stack/whisper-base-axmodel) и [Small](https://huggingface.co/M5Stack/whisper-small-axmodel). Базовая сборка community-примера:

```bash
git clone https://github.com/ml-inory/whisper.axcl.git
cd whisper.axcl
mkdir -p build && cd build
cmake -DCMAKE_INSTALL_PREFIX=../install -DCMAKE_BUILD_TYPE=Release ..
make install -j4
cd ../install
./whisper -w ../demo.wav
```

В опубликованном M5Stack примере для Small: encoder около 190 мс, декодирование около 31.8 ток/с и примерно 735 мс суммарно для короткого тестового файла после загрузки моделей. Загрузка трех графов при этом занимает несколько секунд, поэтому процесс нужно держать запущенным, а не стартовать для каждой фразы.

Для русского:

- явно задавайте `language=ru`, если интерфейс конкретного приложения это позволяет;
- начинайте с Small; Tiny/Base быстрее, но хуже переносят шум, имена и длинные фразы;
- Turbo стоит тестировать для пакетной расшифровки и малой задержки;
- приводите аудио к mono PCM 16 кГц;
- длинную запись делите через VAD на перекрывающиеся сегменты, сохраняйте временные метки и склеивайте текст;
- измеряйте WER на собственных записях, особенно для телефонного звука и профессиональной лексики.

Whisper работает с окнами до 30 секунд, поэтому «часовой файл» всегда является задачей внешнего пайплайна. Для streaming полезны ring buffer, VAD и сохранение состояния приложения, но готовый порт не превращает Whisper в истинно потоковую архитектуру.

### SenseVoice

[SenseVoice](https://github.com/AXERA-TECH/ax_asr_api) быстрый и умеет дополнительные аудиособытия/эмоции, но официальный набор языков не включает русский. Для русской речи он не заменяет Whisper.

### TTS и speaker recognition

- [M5Stack CosyVoice2](https://docs.m5stack.com/en/guide/ai_accelerator/llm-8850/m5_llm_8850_cosy_voice2) — основной современный TTS-вариант, включая отдельный API-пример.
- [MeloTTS](https://docs.m5stack.com/en/guide/ai_accelerator/llm-8850/m5_llm_8850_melotts) — более простой TTS-порт.
- [3D-Speaker-MT](https://docs.m5stack.com/en/guide/ai_accelerator/llm-8850/m5_llm_8850_3d_speaker_mt) — embedding/идентификация говорящего.
- [sherpa-onnx](https://docs.m5stack.com/en/guide/ai_accelerator/llm-8850/m5_llm_8850_sherpa-onnx) — еще один путь сборки голосового приложения.

Поддержку русского голоса нужно проверять по конкретному checkpoint, а не только по названию TTS-движка.

## Генеративные модели

### LCM-LoRA Stable Diffusion 1.5

[AXERA-TECH/lcm-lora-sdv1-5](https://huggingface.co/AXERA-TECH/lcm-lora-sdv1-5) — наиболее практичный генератор изображений для этой карты. В опубликованном запуске:

- text encoder: примерно 3–4.5 с;
- загрузка моделей: примерно 14–23 с;
- один шаг UNet: около 433 мс;
- четыре шага: около 1.74 с;
- VAE decode: около 914 мс.

После прогрева одна картинка формируется за несколько секунд, но первый запуск заметно дольше. Команды:

```bash
git clone https://huggingface.co/AXERA-TECH/lcm-lora-sdv1-5
cd lcm-lora-sdv1-5
python -m venv sd
source sd/bin/activate
sudo apt install -y cmake
pip install -r requirements.txt
pip install https://github.com/AXERA-TECH/pyaxengine/releases/download/0.1.3.rc2/axengine-0.1.3-py3-none-any.whl
python run_txt2img_axe_infer.py
```

### Обычная SD 1.5 и LivePortrait

- [M5Stack/SD1.5-LLM8850](https://huggingface.co/M5Stack/SD1.5-LLM8850) содержит 10/20-step backend и Web UI. Он функционален, но медленнее LCM-LoRA.
- [LivePortrait](https://docs.m5stack.com/en/guide/ai_accelerator/llm-8850/m5_llm_8850_liveportrait) анимирует портрет по driving video и показывает, что карта годится не только для LLM.

Современных портов SDXL, FLUX и diffusion-видео с сопоставимой готовностью для 8 ГБ AX8850 пока нет. Здесь ограничения памяти и неподдерживаемых операций особенно заметны.

## Компьютерное зрение и готовые приложения

CV — самая зрелая часть экосистемы. Доступны:

- YOLOv5/YOLOv8/[YOLO11](https://docs.m5stack.com/en/guide/ai_accelerator/llm-8850/m5_llm_8850_yolo11);
- [YOLO-World-V2](https://docs.m5stack.com/en/guide/ai_accelerator/llm-8850/m5_llm_8850_yolo_world_v2) для open-vocabulary detection;
- face detection, pose, segmentation, OCR и классификация;
- [Depth Anything V2](https://docs.m5stack.com/en/guide/ai_accelerator/llm-8850/m5_llm_8850_depth_anything_v2);
- [MixFormer-V2](https://docs.m5stack.com/en/guide/ai_accelerator/llm-8850/m5_llm_8850_mixformer_v2) для tracking;
- [Real-ESRGAN](https://docs.m5stack.com/en/guide/ai_accelerator/llm-8850/m5_llm_8850_real_esrgan), SuperResolution и RIFE;
- [Frigate NVR](https://docs.m5stack.com/en/guide/ai_accelerator/llm-8850/m5_llm_8850_frigate) и [Immich](https://docs.m5stack.com/en/guide/ai_accelerator/llm-8850/m5_llm_8850_immich).

В полном M5Stack-примере YOLOv5s обработка изображения занимает около 8.1 мс, тогда как чистый benchmark YOLO11s показывает около 3.4 мс. Разница хорошо иллюстрирует, почему скорость одного NPU-графа нельзя считать FPS готового приложения.

Для камеры выгодно декодировать видео аппаратным блоком карты и держать resize/color conversion/inference на устройстве. Возвращать на Pi следует bbox, классы, embeddings или готовый кадр, а не все промежуточные тензоры.

## Как адаптировать свою модель

### Обычная ONNX-модель

Типичный путь:

```text
PyTorch/TensorFlow
  → экспорт ONNX с фиксированными формами
  → onnxsim/onnxslim и проверка ONNX Runtime
  → подготовка репрезентативной calibration-выборки
  → Pulsar2, target_hardware: AX650
  → .axmodel
  → axcl_run_model
  → интеграция через AXCL/pyaxcl
```

Основная документация: [Pulsar2 English docs](https://github.com/AXERA-TECH/pulsar2-docs-en), [Pulsar2 User Guide](https://pulsar2-docs.readthedocs.io/en/latest/), [ONNX operator support](https://pulsar2-docs.readthedocs.io/en/latest/appendix/op_support_list.html).

Практический порядок:

1. Экспортируйте inference-only ONNX без training-веток и случайных операций.
2. Зафиксируйте формы входов, если dynamic shape не является строго необходимой.
3. Проверьте численное совпадение PyTorch и ONNX Runtime.
4. Упростите граф через `onnxslim`; после упрощения снова сравните результаты.
5. Сверьте операции с таблицей Pulsar2. Неподдерживаемый слой замените эквивалентным подграфом или вынесите на CPU.
6. Подготовьте реальные calibration-данные после того же preprocessing, который будет в приложении. Для CV обычно нужны десятки/сотни репрезентативных кадров.
7. Сначала соберите небольшой вариант модели в INT8. Для чувствительных узлов применяйте mixed precision, если это поддерживает текущий Pulsar2.
8. Запустите `.axmodel` через `axcl_run_model`, затем сравните метрики исходника, ONNX и NPU.
9. Только после проверки точности интегрируйте препроцессинг и постпроцессинг в приложение.

Для AX8850 в конфигурациях и именах артефактов обычно указывается `AX650`. Не меняйте цель на выдуманное `AX8850`, если документация конкретной версии Pulsar2 требует AX650.

### LLM и VLM

Портирование трансформера сложнее обычной ONNX-модели. Нужны:

- поддерживаемая `ax-llm` архитектура и шаблон конфигурации;
- разбиение prefill/decode и слоев на NPU-графы;
- совместимый tokenizer/chat template;
- преобразование и часто GPTQ Int4-квантование весов;
- правильные KV-cache shapes и лимит контекста;
- host-код для sampling и multimodal encoder;
- проверка не только perplexity, но и длинных диалогов, stopping tokens и tool calls.

Лучший путь для своей fine-tuned модели — брать уже поддерживаемую базовую архитектуру, например Qwen3/Qwen3.5, делать LoRA/QLoRA fine-tuning на обычной GPU-машине, сливать adapter с базой, затем повторять официальный pipeline конвертации. Обучение на AX8850 не является рабочим сценарием; это inference-ускоритель.

Руководство по LLM-компиляции: [Pulsar2 LLM Build](https://pulsar2-docs.readthedocs.io/en/latest/appendix/build_llm.html). В качестве эталона структуры используйте готовую модель той же архитектуры и текущий [ax-llm](https://github.com/AXERA-TECH/ax-llm).

### Когда портирование невыгодно

Не начинайте с конвертации, если:

- архитектуры нет среди поддерживаемых примеров;
- граф содержит custom ops, динамические циклы или сложный control flow;
- ожидается частая смена моделей;
- качество нужно проверить до длительной работы с toolchain;
- модель почти заполняет 8 ГБ только весами.

Сначала проверьте задачу на готовом Qwen/Whisper/YOLO-порте. Собственный порт оправдан, когда есть измеримый выигрыш качества, задержки или специализации.

## Состояние сообщества

Экосистема открыта частично. Исходники runtime-примеров, оберток и многих приложений доступны, но компилятор и низкоуровневые компоненты остаются vendor-specific. Поэтому «community model» часто означает открытую исходную модель, которую портировала AXERA, а не независимую реализацию рантайма.

Полезные проекты:

| Проект | Роль | Зрелость |
|---|---|---|
| [AXERA-TECH/ax-llm](https://github.com/AXERA-TECH/ax-llm) | LLM/VLM runtime и API | Основной, активно обновляется |
| [AXERA-TECH/awesome-docs](https://github.com/AXERA-TECH/awesome-docs) | Каталог моделей, документация и benchmark | Основной индекс |
| [AXERA-TECH/axcl-samples](https://github.com/AXERA-TECH/axcl-samples) | PCIe C/C++ samples | Зрелая база разработки |
| [AXERA-TECH/pyaxcl](https://github.com/AXERA-TECH/pyaxcl) | Python bindings | Полезен для прототипов |
| [AXERA-TECH/pyaxengine](https://github.com/AXERA-TECH/pyaxengine) | Python execution provider | Используется в готовых приложениях |
| [AXERA-TECH/ax_asr_api](https://github.com/AXERA-TECH/ax_asr_api) | Whisper/SenseVoice API | Лучший старт для ASR |
| [ml-inory/whisper.axcl](https://github.com/ml-inory/whisper.axcl) | Независимый Whisper-порт | Важный community-проект |
| [M5Stack CosyVoice2](https://docs.m5stack.com/en/guide/ai_accelerator/llm-8850/m5_llm_8850_cosy_voice2) | CosyVoice2 TTS | Специализированный готовый порт |
| [AXERA-TECH/axcl-pi5-examples](https://github.com/AXERA-TECH/axcl-pi5-examples) | Примеры именно для Pi 5 | Практичная точка входа |
| [M5Stack Hugging Face](https://huggingface.co/M5Stack) | Готовые артефакты для LLM-8850 | Удобно, но часть моделей старее AXERA-каталога |
| [AXERA-TECH Hugging Face](https://huggingface.co/AXERA-TECH) | Самый широкий набор новых портов | Проверять AXCL и целевой чип |

## Известные ограничения и ловушки

1. **Gemma 4 сейчас нельзя рекомендовать для LLM-8850 по PCIe.** В `ax-llm` отмечена ошибка корректности Gemma 4 под AXCL. Наличие модели и benchmark в общем каталоге не означает рабочий PCIe-сценарий.
2. **Названия AX650N/AX650 нормальны.** AX8850 может отображаться в `axcl-smi` как AX650N и компилироваться с target AX650.
3. **Версии должны совпадать.** Свежие модели могут требовать новый AXCL, firmware, `ax-llm` и Pulsar2. После обновления драйвера проверяйте `axcl-smi` и пересобирайте приложения.
4. **8 ГБ на карте — не 8 ГБ для модели.** Часть памяти зарезервирована. В опубликованном `axcl-smi` доступный CMM близок к 7040 МиБ.
5. **Контекст LLM дорог.** Уменьшайте максимальную длину и число одновременных запросов прежде, чем заключать, что модель не помещается.
6. **PCIe 2.0 x2 ограничивает обмен.** Держите pipeline на карте; частые host-device копирования могут съесть выигрыш NPU.
7. **Первый запуск медленный.** Загрузка нескольких `.axmodel` занимает секунды. Для сервиса загружайте модель один раз и прогревайте ее.
8. **Не все Hugging Face-репозитории готовы для AXCL.** Читайте README и ищите `AX650`, `AX8850`, `AXCL` и aarch64; SoC-пример может обращаться к локальным устройствам иначе.
9. **Опубликованный benchmark не равен приложению.** Отдельно измеряйте prefill, decode, preprocessing, PCIe copy, postprocessing и полную задержку запроса.
10. **Безопасность API остается вашей задачей.** Не выставляйте OpenAI-совместимый порт в Интернет без reverse proxy, настоящей аутентификации, TLS и rate limiting.

## Рекомендуемый план первого запуска

1. Установить карту с корректным питанием и охлаждением.
2. Обновить EEPROM, убедиться, что `lspci` видит Device 0650.
3. Установить актуальный `axclhost`, проверить карту через `axcl-smi`.
4. Запустить `yolo11s.axmodel` через `axcl_run_model` и записать температуру/время.
5. Установить Qwen3-1.7B Int4 и проверить CLI, затем OpenAI API.
6. Сравнить Qwen3.5-2B и 4B на своем наборе запросов, контролируя CMM и контекст.
7. Для речи запустить Whisper Small на чистом и шумном русском аудио, затем измерить real-time factor и WER.
8. Для камеры собрать YOLO → tracker → VLM pipeline вместо VLM на каждом кадре.
9. Только после появления baseline начинать конвертацию собственной модели.

Минимальная таблица собственных измерений:

| Модель | AXCL/runtime | Контекст/вход | Загрузка | Prefill | Decode/FPS/RTF | CMM peak | Температура | Качество |
|---|---|---:|---:|---:|---:|---:|---:|---|
| | | | | | | | | |

## Основные ссылки

- [M5Stack LLM-8850 overview](https://docs.m5stack.com/en/guide/ai_accelerator/overview)
- [M5Stack hardware installation](https://docs.m5stack.com/en/guide/ai_accelerator/llm-8850/m5_llm_8850_hardware_install)
- [M5Stack environment setup](https://docs.m5stack.com/en/guide/ai_accelerator/llm-8850/m5_llm_8850_software_install)
- [M5Stack quick experience](https://docs.m5stack.com/en/guide/ai_accelerator/llm-8850/m5_llm_8850_quick_start)
- [AXCL V3.16.0 documentation](https://axcl-docs.readthedocs.io/en/latest/)
- [AXERA ax-llm](https://github.com/AXERA-TECH/ax-llm)
- [AXERA awesome-docs](https://github.com/AXERA-TECH/awesome-docs)
- [AXERA axcl-samples](https://github.com/AXERA-TECH/axcl-samples)
- [AXERA pyaxcl](https://github.com/AXERA-TECH/pyaxcl)
- [Pulsar2 documentation](https://pulsar2-docs.readthedocs.io/en/latest/)
- [Pulsar2 docs repository](https://github.com/AXERA-TECH/pulsar2-docs-en)
- [AXERA Hugging Face](https://huggingface.co/AXERA-TECH)
- [M5Stack Hugging Face](https://huggingface.co/M5Stack)

## Итоговая оценка

LLM-8850 особенно удачен как автономный edge-ускоритель для нескольких четких задач: локальная LLM до 4B Int4, компактная VLM, Whisper, быстрый CV-пайплайн, TTS и SD 1.5/LCM. По абсолютной гибкости он уступает GPU: ассортимент моделей уже, портирование сложнее, а версии vendor toolchain связаны между собой. Зато в поддерживаемых сценариях карта дает Raspberry Pi 5 уровень инференса, недостижимый на одном CPU, при компактном форм-факторе и умеренном энергопотреблении.

Наиболее рациональная стратегия — строить продукт вокруг готовых Qwen3/Qwen3.5, Whisper и YOLO-портов, измерить полный pipeline, а собственную конвертацию начинать только для доказанного узкого преимущества.