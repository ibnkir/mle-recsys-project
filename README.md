<<<<<<< HEAD
# Проект: "Создание рекомендательной системы для музыкального сервиса"
=======
# Яндекс Практикум, Курс "Инженер машинного обучения" (2024 г.)
# Учебный проект 4-го спринта: "Создание рекомендательной системы"
>>>>>>> 2fc5bd6babe12245095a0f19721964d3cc03c274
## Выполнил: [Кирилл Н.](mailto:ibnkir@yandex.ru)

## Описание проекта
Целью проекта является создание системы рекомендаций аудиотреков для
пользователей музыкального сервиса на основе их истории прослушиваний.
В рамках проекта был разработан пайплайн для расчета нескольких типов рекомендаций,
который был интегрирован в веб-сервис на базе FastAPI.

Основные инструменты и Python-библиотеки:
- Visual Studio Code
- Jupyter Lab
- implicit.ALS
- S3
- FastAPI
- uvicorn

## Как воспользоваться репозиторием
1. Перейдите в домашнюю папку и склонируйте репозиторий на ваш компьютер
   ```bash
   cd ~
   git clone https://github.com/ibnkir/mle-recsys-project
   ```

2. Создайте виртуальное окружение и установите в него необходимые Python-пакеты
    ```
    python3 -m venv env_recsys_start
    . env_recsys_start/bin/activate
    pip install -r requirements.txt
    ```

3. Скачайте три файла с исходными данными в папку репозитория
    - Данные о треках - [tracks.parquet](https://storage.yandexcloud.net/mle-data/ym/tracks.parquet)
    - Имена артистов, названия альбомов, треков и жанров - [catalog_names.parquet](https://storage.yandexcloud.net/mle-data/ym/catalog_names.parquet)
    - Данные о том, какие пользователи прослушали тот или иной трек - [interactions.parquet](https://storage.yandexcloud.net/mle-data/ym/interactions.parquet)
 
    Для удобства можно воспользоваться командой wget:
    ```
    wget https://storage.yandexcloud.net/mle-data/ym/tracks.parquet
    wget https://storage.yandexcloud.net/mle-data/ym/catalog_names.parquet
    wget https://storage.yandexcloud.net/mle-data/ym/interactions.parquet
    ```

4. Запустите Jupyter Lab в командной строке
    ```
    jupyter lab --ip=0.0.0.0 --no-browser
    ```

5. Чтобы не выполнять подготовительный код в Jupyter-ноутбуке и сразу перейти к запуску и тестированию проекта, скачайте готовые файлы с рекомендациями по ссылкам:
    - Топ-треки [top_popular.parquet](https://disk.yandex.ru/d/nTcukpqOtQLDsg)
    - Похожие треки [similar.parquet](https://disk.yandex.ru/d/dsXfq-ZLVMmUTQ)
    - Финальные рекомендации [recommendations.parquet](https://disk.yandex.ru/d/9Y__uW1wLRtzuA)

## Этапы и результаты выполнения проекта
1. __Предобработка данных и EDA__
    - Проверили исходные данные на наличие пропусков, дубликатов, некорректных типов, 
    значений и связей между таблицами. Для экономии ресурсов оставили только треки, 
    где были заполнены все четыре категории жанр/артист/альбом/название трека. 
    Из истории прослушиваний убрали строки, соответствующие удаленным трекам;
    - По той же причине сократили датасет событий примерно в 3 раза с 5.4G до 1.5-2G, 
    для этого у всех пользователей удалили около 2/3 прослушиваний 
    (в истории каждого пользователя оставили только треки на позициях 1,4,7 итд), 
    после чего удалили колонку `track_seq`;
    - Также для экономии памяти оставили тип int32 у идентификаторов пользователей и треков в таблице `interactions`, соответствующую колонку в таблице `tracks` привели к тому же типу;
    - Предобработанные данные о треках и прослушиваниях сохранены в файлах `items.parquet` и `events.parquet`
    и загружены в S3-хранилище.
    
    Код для предварительной обработки и EDA представлен в файле `recommendations.ipynb`.

2. __Расчёт рекомендаций__
    
    Ниже перечислены реализованные подходы для генерации рекомендаций и их метрики на валидации:
    - Рекомендации по умолчанию на основе топ-100 популярных треков по количеству прослушиваний:
        - Доля событий "холодных" пользователей, совпавших с рекомендациями по умолчанию: 0.056
        - Доля "холодных" пользователей без релевантных рекомендаций: 0.620
        - Среднее покрытие "холодных" пользователей треками: 1.819
    - Персональные рекомендации на основе коллаборативного подхода с использованием ALS 
    (по 50 рекомендаций на каждого пользователя):
        - coverage: 0.004
        - novelty@5: 0.792
        - precision: 0.0022
        - recall: 0.0036
    - Рекомендации на основе жанровой близости треков с использованием ALS (по 10 на каждый трек);
    - Финальные рекомендации на основе ранжирующей catboost-модели по нескольким признакам, 
    включая коллаборативные оценки, количество треков, прослушанных каждым пользователем, 
    и жанровые оценки (по 50 рекомендаций на каждого пользователя):
        - coverage: 0.004
        - novelty@5: 0.781
        - precision: 0.0022
        - recall: 0.0036

    Также все рекомендации были оценены визуально на случайно выбранных примерах.
    
    Код для расчёта рекомендаций представлен в файле `recommendations.ipynb`.

3. __Запуск и тестирование сервиса__
    
    Исходный код основного и вспомогательных сервисов содержится в файлах:
    - `recommendations_service.py` - основной сервис для генерации оффлайн- и онлайн-рекомендаций;
    - `features_service.py` - вспомогательный сервис для поиска похожих треков;
    - `events_service.py` - вспомогательный сервис для сохранения и получения 
    последних прослушанных треков пользователя.
    
    Перед запуском убедитесь, что в репозитории находятся все необходимые parquet-файлы 
    с рекомендациями (см. выше), а также конфигурационный файл `config.ini`, в котором
    прописаны url всех трех сервисов. 
    Далее выполните 3 команды (по одной на каждый сервис) в 3-х разных терминалах, находясь в корневой папке проекта:
    ```
    uvicorn recommendations_service:app
    uvicorn events_service:app --port 8020
    uvicorn features_service:app --port 8010
    ```
    
    Для отправки тестовых запросов откройте 4-й терминал, перейдите на нем в папку проекта
    и выполните команды, соответствующие различным сценариям для
    произвольно выбранного пользователя и объектов:

    - Получение топ-10 рекомендаций по умолчанию<br>
    ```python test_service.py```
    - Получение первых 10 персональных рекомендаций только по оффлайн-истории пользователя с user_id=617032<br>
    ```python test_service.py --offline --user_id 617032 --k 10```
    - Добавление item_id=99262 в онлайн-историю пользователя с user_id=617032<br>
    ```python test_service.py --add_online --user_id 617032 --item_id 99262```
    - Добавление item_id=590303 в онлайн-историю пользователя с user_id=617032<br>
    ```python test_service.py --add_online --user_id 617032 --item_id 590303```
    - Добавление item_id=597196 в онлайн-историю пользователя с user_id=617032<br>
    ```python test_service.py --add_online --user_id 617032 --item_id 597196```
    - Получение первых 10 персональных рекомендаций только по онлайн-истории пользователя с user_id=617032<br>
    ```python test_service.py --online --user_id 617032 --k 10```
    - Получение первых 10 смешанных персональных рекомендаций по оффлайн- и онлайн-истории пользователя с user_id=617032 (первые на четных позициях, вторые на нечетных)<br>
    ```python test_service.py --blended --user_id 617032 --k 10```

    Также для тестирования можно использовать Jupyter-ноутбук `tests.ipynb`
