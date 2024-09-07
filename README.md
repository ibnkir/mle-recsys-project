# Проект: "Разработка системы рекомендаций аудиотреков"
## Выполнил: [Кирилл Н.](mailto:ibnkir@yandex.ru)

## Описание проекта
Целью проекта является создание рекомендательной системы для
пользователей музыкального сервиса на основе их истории прослушиваний.
В рамках проекта были проведены предобработка и анализ данных, 
сгенерированы рекомендации нескольких типов, посчитаны метрики на тестовой выборке и разработан веб-сервис на базе FastAPI для получения смешанных рекомендаций.

Основные использованные инструменты и Python-библиотеки:
- Visual Studio Code
- Jupyter Lab
- implicit.ALS
- Catboost
- S3-хранилище на Yandex Cloud
- FastAPI
- uvicorn

## Как воспользоваться репозиторием
1. Перейдите в домашнюю папку и склонируйте репозиторий на ваш компьютер
   ```bash
   cd ~
   git clone https://github.com/ibnkir/music-recsys-project
   ```

2. Создайте виртуальное окружение и установите в него необходимые Python-пакеты
    ```
    python3 -m venv .venv_music_recsys
    source .venv_music_recsys/bin/activate
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
1. __Предобработка и анализ данных__
    - Проверили исходные данные на наличие пропусков, дубликатов, некорректных типов, 
    значений и связей между таблицами. Для экономии ресурсов оставили только треки, 
    где были заполнены все четыре категории: жанр, артист, альбом и название трека. 
    Из истории прослушиваний убрали строки, соответствующие удаленным трекам;
    - По той же причине сократили датасет событий примерно в 3 раза с 5.4Gb до 1.5-2Gb, 
    для этого у всех пользователей удалили около 2/3 прослушиваний, оставив 
    в истории каждого пользователя только треки на позициях 1,4,7 и т.д., 
    после чего удалили колонку `track_seq`;
    - Также для экономии памяти оставили тип int32 у идентификаторов пользователей и треков в таблице `interactions`. Соответствующую колонку в таблице `tracks` привели к тому же типу;
    - Предобработанные данные о треках и прослушиваниях сохранены в файлах `items.parquet` и `events.parquet`
    и загружены в S3-хранилище.
    
    Код для предварительной обработки и анализа данных представлен в файле `recommendations.ipynb`.

2. __Расчёт рекомендаций__
    
    Ниже перечислены посчитанные рекомендации и их метрики на тестовой выборке:
    - Случайные рекомендации (baseline 1):
        - Precision@5: 0.0
        - Recall@5: 0.0
    
    - Рекомендации на основе топ-100 популярных треков (baseline 2):
        - Precision@5: 0.00124
        - Recall@5: 0.00117
        - F1@5: 0.00120
        - Покрытие "холодных" пользователей: 0.38
        - Среднее количество совпавших рекомендаций на одного «холодного» пользователя: 1.8

    - Персональные рекомендации на основе коллаборативной фильтрации с помощью ALS 
    (по 50 рекомендаций на каждого пользователя):
        - Precision@5: 0.00222
        - Recall@5: 0.00360
        - F1@5: 0.00274
        - Novelty@5: 0.792
        - Покрытие объектов: 0.004

    - Персональные рекомендации на основе ранжирующей catboost-модели по нескольким признакам, 
    включая коллаборативные оценки, количество треков, прослушанных каждым пользователем, 
    и жанровые оценки (по 50 рекомендаций на каждого пользователя):
        - Precision@5: 0.00221
        - Recall@5: 0.00358
        - F1@5: 0.00273
        - Novelty@5: 0.781
        - Покрытие объектов: 0.004
        
    Также были рассчитаны контентные i2i-рекомендации с помощью ALS на основе жанровой близости треков для генерации онлайн-рекомендаций (по 10 на каждый трек).
    
    Помимо расчета метрик все рекомендации были оценены визуально на случайно выбранных примерах. Код для расчёта рекомендаций представлен в файле `recommendations.ipynb`.

3. __Запуск и тестирование сервиса__
    
    Исходный код основного и вспомогательных сервисов содержится в файлах:
    - `recommendations_service.py` - основной сервис для генерации офлайн- и онлайн-рекомендаций;
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
    произвольно выбранного пользователя и объектов, как показано ниже:

    - Получение топ-10 рекомендаций по умолчанию<br>
    ```python test_service.py```
    - Получение первых 10 персональных рекомендаций только по офлайн-истории пользователя с user_id=617032<br>
    ```python test_service.py --offline --user_id 617032 --k 10```
    - Добавление item_id=99262 в онлайн-историю пользователя с user_id=617032<br>
    ```python test_service.py --add_online --user_id 617032 --item_id 99262```
    - Добавление item_id=590303 в онлайн-историю пользователя с user_id=617032<br>
    ```python test_service.py --add_online --user_id 617032 --item_id 590303```
    - Добавление item_id=597196 в онлайн-историю пользователя с user_id=617032<br>
    ```python test_service.py --add_online --user_id 617032 --item_id 597196```
    - Получение первых 10 персональных рекомендаций только по онлайн-истории пользователя с user_id=617032<br>
    ```python test_service.py --online --user_id 617032 --k 10```
    - Получение первых 10 смешанных персональных рекомендаций по офлайн- и онлайн-истории пользователя с user_id=617032 (первые на четных позициях, вторые на нечетных)<br>
    ```python test_service.py --blended --user_id 617032 --k 10```

    Также для тестирования можно использовать Jupyter-ноутбук `tests.ipynb`
