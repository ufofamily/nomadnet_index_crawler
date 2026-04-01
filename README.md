# NomadNet Search

**NomadNet Search** — это локальный индексатор и поисковый веб-интерфейс для узлов **NomadNet / Reticulum**. Проект слушает announce-пакеты в сети Reticulum, обнаруживает доступные destination, ставит страницы в очередь обхода, пытается получить контент ресурсов вроде `/page/index.mu`, извлекает внутренние ссылки, рекурсивно расширяет граф обхода и строит локальный индекс, по которому затем можно искать через простой Google-подобный веб-интерфейс.

Проект не является глобальным поисковиком всей сети. Он индексирует только ту часть NomadNet, которую реально видит конкретная нода через доступные transport/path в Reticulum. Поэтому это скорее **локальный поисковый шлюз по видимому сегменту сети**, чем централизованный поисковый сервис.

## Возможности

- обнаружение нод через **Reticulum announces**
- хранение найденных destination в локальной базе
- очередь страниц на обход
- запрос стартовых страниц нод
- извлечение внутренних ссылок из содержимого страниц
- рекурсивная постановка найденных путей в очередь
- локальное хранение страниц и метаданных
- поиск по проиндексированному содержимому
- веб-интерфейс для поиска и просмотра сохранённых страниц

## Архитектура

Проект состоит из нескольких модулей:

- `main.py` — точка входа, запуск Reticulum, DB writer, crawler worker
- `crawler/announce_listener.py` — обработчик announce-пакетов
- `crawler/fetcher.py` — worker, забирающий страницы из очереди и индексирующий их
- `crawler/page_client.py` — сетевой клиент для загрузки страниц с нод
- `parser/link_extractor.py` — извлечение внутренних ссылок из текста страницы
- `search/indexer.py` — работа с SQLite, нодами, страницами и поиском
- `search/db_writer.py` — сериализованная запись в БД через очередь
- `web/app.py` — веб-интерфейс поиска и просмотра страниц
- `config/reticulum.conf` — конфигурация Reticulum
- `data/indexer.db` — локальная база данных

## Зависимости

Минимальные зависимости:

- **Python 3.11+**
- **rns**
- **lxmf**
- **flask**

SQLite отдельно устанавливать не нужно — он входит в стандартную библиотеку Python.

## Установка

### 1. Клонирование проекта

```bat
git clone <URL_РЕПОЗИТОРИЯ>
cd nomad-search
2. Создание виртуального окружения
python -m venv .venv
3. Активация окружения в Windows CMD
.venv\Scripts\activate
Для PowerShell:
.venv\Scripts\Activate.ps1
4. Установка зависимостей
pip install rns lxmf flask
pip freeze > requirements.txt
Структура проекта
nomad-search/
│
├── .venv/
├── main.py
├── requirements.txt
├── config/
│   └── reticulum.conf
├── data/
│   └── indexer.db
├── crawler/
│   ├── __init__.py
│   ├── announce_listener.py
│   ├── fetcher.py
│   └── page_client.py
├── parser/
│   ├── __init__.py
│   └── link_extractor.py
├── search/
│   ├── __init__.py
│   ├── db_writer.py
│   └── indexer.py
└── web/
    ├── __init__.py
    └── app.py
Конфигурация Reticulum
Создай файл config/reticulum.conf:
[reticulum]
  enable_transport = yes
  share_instance = no
  shared_instance_port = 37428
  instance_control_port = 37429

[[Default TCP Interface]]
  type = TCPClientInterface
  enabled = yes
  target_host = amsterdam.connect.reticulum.network
  target_port = 4965
При необходимости можно заменить bootstrap-узел на другой доступный transport.
Запуск индексатора
python main.py
После запуска индексатор:
1.	инициализирует базу данных
2.	запускает Reticulum
3.	регистрирует обработчик announce
4.	запускает DB writer
5.	запускает fetcher worker
6.	начинает обнаруживать ноды и пытаться индексировать страницы
Пример сообщений в консоли:
DB writer started
Fetcher worker started
Indexer started, waiting for announces...
Discovered: 1da0dfacb470708c45fbb91e8f4e5ecd hops=4 iface=TCPInterface[...]
[FETCH] 1da0dfacb470708c45fbb91e8f4e5ecd:/page/index.mu
[SUCCESS] 1da0dfacb470708c45fbb91e8f4e5ecd:/page/index.mu title='Home' body_len=2048 links=12
[ENQUEUE] 1da0dfacb470708c45fbb91e8f4e5ecd:/page/about.mu
Запуск веб-интерфейса
В отдельном окне, в том же виртуальном окружении:
python web\app.py
После этого интерфейс будет доступен по адресу:
http://127.0.0.1:5000
Как это работает
Обнаружение нод
AnnounceHandler слушает announce-пакеты Reticulum и добавляет найденные ноды в локальную БД.
Очередь страниц
Для новых нод создаётся стартовая запись страницы, обычно:
/page/index.mu
Загрузка страниц
Fetcher забирает страницы со статусом pending, переводит их в processing, пытается получить содержимое страницы с удалённой ноды и при успехе сохраняет его в БД.
Извлечение ссылок
После загрузки страницы парсер ищет внутренние ссылки вида:
•	/page/about.mu
•	page:about.mu
•	nomadnet:/page/about.mu
•	hash:/page/index.mu
Найденные пути ставятся в очередь на дальнейший обход.
Поиск
Веб-интерфейс ищет по локально сохранённым страницам и показывает:
•	путь страницы
•	заголовок
•	сниппет
•	переход на полное сохранённое содержимое
База данных
Проект использует SQLite. Основные сущности:
nodes
Содержит найденные ноды:
•	destination_hash
•	first_seen
•	last_seen
•	hop_count
•	interface
•	status
pages
Содержит известные страницы:
•	node_id
•	path
•	title
•	body
•	fetched_at
•	content_hash
•	status
•	error_text
Текущие ограничения
•	индексируется только та часть сети, которая реально достижима с текущего transport
•	не все announce соответствуют NomadNet page nodes
•	часть destination может таймаутиться или не иметь page handler
•	полнотекстовый поиск сейчас реализован в упрощённом виде
•	нет распределённого обмена индексом между индексаторами
•	нет нормализации Micron-разметки в полноценную структурированную модель
•	нет robots/noindex-политики
•	нет ограничения глубины обхода и per-node rate limiting в завершённом виде
Типичные проблемы
ModuleNotFoundError: No module named 'flask'
Запущен не тот Python. Нужно использовать виртуальное окружение:
.venv\Scripts\activate
python web\app.py
database is locked
Причина — конкурентная запись в SQLite. В проекте запись должна идти только через DBWriter.
database disk image is malformed
База была повреждена на предыдущих этапах разработки. Удали:
del data\indexer.db
del data\indexer.db-wal
del data\indexer.db-shm
и запусти проект заново.
timeout while fetching /page/index.mu
Причины могут быть такие:
•	destination не является NomadNet page node
•	путь до ноды отсутствует
•	нода не отвечает
•	выбран неверный service/aspect для page request
•	страница не опубликована
