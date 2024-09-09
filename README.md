# pHd (patent helper digital)

Приложение находит ИНН патентообладетелей и показывает статистику по патентам. Его сделала одноимённая команда на хакатоне «Лидеры цифровой трансформации» 2024 года.

Помимо документации в этой организации на Гитхабе есть ещё три репозитория: фронтенд, бэкенд и исследования.

## Компоненты

### Поиск патентообладателей (`research`)

1. Реализован в `research:ch-search.py`. Он получает файл с патентами (ожидается, что его надо скачать из открытых данных Роспатента), подключается базе в Кликхаусе (её надо запустить и наполнить заранее) и ищет в нём каждого патентообладателя. Результат — csv-файл, в котором указаны регистрационный номер и вид патента, а также ИНН патентообладателя (если нашёлся).
2. База готовится на основе Docker-образа ClickHouse и базы от организаторов. Примерный код находится в `research:index.sql` и в `research:Prepare Lookup for CH.ipynb`. В данный момент развёрнута на [ch.patenthelper.digital](http://ch.patenthelper.digital).
3. Алгоритмы поиска — по точному соответствию имени, по имени как подстроке, по совпадению токенов. Для ранжирования результатов используется `ngramDistance` (n=4) между названием патентообладателя из открытых данных и полным наименованием из базы, между почтовым адресом из базы и юридическим или фактическим адресом из базы.
4. Полнота привязки — 84% патентов с российскими патентообладателями. Скорость — 16 тысяч в час на 4CPU cloud.ru (100% загрузка). Точность не оценивалась.
5. Текущий вариант результатов привязки находится на [Яндекс-диске](https://disk.yandex.ru/d/WXmneomSpMj8Jw) (файлы `matched_*`).
6. Мы не стали использовать присланные организаторами Excel-файлы, так как открытые данные удобнее и мало отличаются от них. Исключение — МПК-коды взяты из Excel-файлов.

### Бэкенд

1. Реализован на Python, FastAPI, PostgreSQL.
2. Развёрнут на [backend.patenthelper.digital](http://backend.patenthelper.digital).
3. Есть [автосгенерированная документация](http://backend.patenthelper.digital/docs).
4. Данные, полученные в ходе поиска, загружены в БД. Для загрузки имеется CLI-интерфейс (`backend:app/cli.py`). Режим `load-patents` грузит открытые данные Росреестра, `load-persons` — патентообладателей (для получения загружаемого файла желательно использовать `research:Persons with Patents.ipynb`, который выберет только тех лиц, у которых есть хотя бы один патент), `load-ownership` предназначен для загрузки результатов работы `research:ch-search.py` (можно взять на Яндекс-диске из п. 5 в предыдущем разделе). Предварительно надо запустить Postgres и указать `DATABASE_CLI_URL` в `.env`.
5. Чтобы запустить бэкенд локально, надо установить Python 3.10, выполнить `pip -r requirements.txt`, запустить Postgres, загрузить в него данные (см. следующий пункт), указать `DATABASE_URL` в `.env` и выполнить `uvicorn app.main:app`.
6. Данные для загрузки в БД находятся на [Яндекс-диске](https://disk.yandex.ru/d/WXmneomSpMj8Jw). Загружаются вспомогательным CLI-инструментом `backend:cli.py`. Таблицы из `patent_data.zip` по очереди грузятся через `load-patents`, таблица из `persons_with_patents.zip` — через `load-persons`, таблицы из `matched_csv.zip` — через `load-ownership`. Порядок патенты/лица неважен, но привязки (`load-ownership`) добавляются последними. Скрипт запускается напрямую на сервере. Проверка правильности = нет ошибок SQL при загрузке (или их сравнительно немного при загрузке привязок).

### Фронтенд

1. Написан на React и Vite.
2. Развёрнут на [app.patenthelper.digital](http://app.patenthelper.digital).
3. Для локального запуска надо запустить бэкенд, загрузить туда данные, установить Nodejs 20, потом `npm i` и далее `npm run dev`.

### Docker-образы, CI/CD

1. Хотели сделать, пробовали сделать и развернуть всё как микросервисы с автодеплоем, но так как у Container Apps на cloud.ru много ограничений (нельзя монтировать volumes = при обновлении потеряются данные из БД; максимум 1 Гб памяти — для загрузки данных в Кликхаус надо больше), то помучились и забили, решив запустить всё на паре виртуальных машин и рассудив, что собрать потом при необходимости Docker-образы для подходящей инфраструктуры — дело несложное.
2. Следы экспериментов есть, например, в `backend:Dockerfile` и `research:ch-search-base`.

## Краткое описание интерфейса

### Фильтры

Загрузка Excel-файла со списком ИНН и сохранение этого списка как фильтра, который можно потом использовать на других экранах. Ожидается, что в Excel-файле есть ровно одна колонка с названием `person_tax_number`. Фильтру можно задать имя. Ненужный фильтр можно удалить. Редактировать фильтры нельзя — только загружать новые.

### Патенты

Список патентов с возможностью фильтрации, просмотра информации о каждом патенте, выгрузки в Excel. Фильтровать можно в том числе и по списку ИНН из «Фильтров».

### Правообладатели

Аналогично патентам, только правообладатели и без экспорта.

### Статистика

Более детальная аналитика.
