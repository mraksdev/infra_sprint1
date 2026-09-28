# infra_sprint1 — развёртывание веб-приложения на удалённый сервер

Учебный проект: бэкенд **Kittygram** на Django REST Framework и фронтенд на React,
развёрнутые на одном сервере через **nginx + gunicorn + systemd**.

Ключевая часть репозитория — не само приложение, а **интеграционные тесты, которые
проверяют живой развёрнутый сервис по сети**: регистрация, авторизация, загрузка
изображений, доступность статики и совпадение IP у соседних проектов.

---

## Стек

**Бэкенд:** Django 5.1.1, Django REST Framework 3.15.2, djoser 2.3.1, Pillow, webcolors
**Фронтенд:** React 17, react-scripts 5.0.0, react-router-dom 5
**Сервер:** nginx, gunicorn, systemd, Let's Encrypt
**База данных:** SQLite 3 (по умолчанию)
**Тесты:** pytest 8.3.3, pytest-django, requests

---

## Структура

```
infra_sprint1/
├── backend/
│   ├── manage.py
│   ├── cats/                    # приложение: модели, сериализаторы, viewset'ы
│   └── kittygram_backend/       # settings, urls, wsgi, asgi
├── frontend/
│   ├── package.json
│   └── src/                     # React-компоненты, api.js, context.js
├── infra/
│   ├── default                  # конфигурация nginx: два vhost'а
│   └── gunicorn_kittygram.service   # systemd-юнит
├── tests/
│   ├── conftest.py
│   ├── test_connection.py       # интеграционные: ходят по HTTPS на живой прод
│   ├── test_files.py            # проверка комплектности infra/
│   └── test_settings.py         # статический анализ settings.py
├── pytest.ini
├── requirements.txt
└── .gitignore
```

---

## Установка

```bash
git clone git@github.com:mraksdev/infra_sprint1.git
cd infra_sprint1
```

```bash
python3 -m venv venv
source venv/bin/activate        # Windows PowerShell: .\venv\Scripts\Activate.ps1
pip install --upgrade pip
pip install -r requirements.txt
```

> **Внимание:** `requests` нужен тестам (`tests/test_connection.py`), но в
> `requirements.txt` не указан. Установите его отдельно:
> `pip install requests`. Также в проде нужен `gunicorn` — он ставится на сервере,
> а не из этого файла.

### Переменные окружения

Единственная переменная, которую читает приложение:

| Переменная | Где | Назначение |
|---|---|---|
| `SECRET_KEY` | `backend/kittygram_backend/settings.py` | Django secret key |

В коде задан небезопасный значение по умолчанию. Задайте свой:

```bash
export SECRET_KEY="ваш_секретный_ключ"
```

Других переменных нет: пути к статике и медиа, `DEBUG = False` и `ALLOWED_HOSTS`
захардкожены в `settings.py` под серверные пути.

---

## Локальный запуск

```bash
cd backend
python manage.py migrate
python manage.py runserver        # http://127.0.0.1:8000
```

`DEBUG` в проекте выключен (это проверяет тест `test_settings.py::test_debug`),
поэтому Django не отдаёт трейсбеки и не раздаёт `/media/` через `static()`.
Локально API доступно на `127.0.0.1:8000/api/`.

Фронтенд:

```bash
cd frontend
npm install
npm start                        # http://localhost:3000
```

> Базовый URL API задан в `frontend/src/utils/constants.js` как `URL = ""`, а
> `proxy` в `package.json` не настроен. Чтобы фронтенд увидел бэкенд, укажите
> в этой константе адрес или добавьте `proxy` в `package.json`.

---

## Тесты

```bash
pytest
```

### Что проверяется

| Файл | Тестов | Что делает |
|---|---|---|
| `test_settings.py` | 2 | `DEBUG` выключен; `SECRET_KEY` не зашит литералом в код |
| `test_files.py` | 2 | В `infra/` лежат все нужные файлы; формат файла с данными развёртывания |
| `test_connection.py` | 6 | Живые интеграционные проверки по HTTPS |

Интеграционные тесты делают настоящие запросы на развёрнутый сервис:

- `test_link_connection[name_taski]` и `[name_kittygram]` — доступность обоих сайтов
- `test_projects_on_same_ip` — оба проекта обслуживаются с одного IP
- `test_kittygram_static_is_available` — JS-бандл фронтенда отдаётся с кодом 200
- `test_kittygram_api_available` — API живо (отвечает 400 на пустой пароль, а не 404)
- `test_kittygram_images_availability` — полный сценарий: авторизация →
  создание кота с base64-изображением → скачивание картинки → удаление

Последний тест создаёт и удаляет запись в боевой базе. Он требует доступных
тестовых прав и чистит за собой.

### Файл с данными развёртывания

8 тестов из 10 читают `infra/kittygram_site.txt` с адресами и учётными данными
сервера. **Этот файл не входит в репозиторий** — он в `.gitignore` под маской
`infra/*_site.txt`, потому что содержит логин и пароль.

На свежем клоне из-за этого падают 8 тестов из 10. Чтобы прогнать интеграционные
тесты, создайте файл вручную:

```
infra/kittygram_site.txt
```

Формат — по одному параметру в строке, `имя: значение;`:

```
IP: <ip-сервера>;
name_taski: <https-адрес-первого-проекта>;
name_kittygram: <https-адрес-kittygram>;
login: <пользователь>;
password: <пароль>;
```

Без него пройдут только `test_settings.py` — проверки `DEBUG` и `SECRET_KEY`.

---

## Развёртывание

Два независимых конфига в `infra/`:

**`infra/default`** — nginx. Описывает два vhost'а: один обслуживает соседний
проект из `/var/www/taski` на порту 8000, второй — Kittygram на порту 8080 с
`client_max_body_size 20M` (нужно для загрузки фото), раздачей `/media/` и
сертификатами Let's Encrypt.

**`infra/gunicorn_kittygram.service`** — systemd-юнит:

```ini
[Unit]
Description=gunicorn daemon for kittygram
After=network.target

[Service]
User=ubuntu
WorkingDirectory=/home/ubuntu/infra_sprint1/backend
ExecStart=/home/ubuntu/infra_sprint1/backend/venv/bin/gunicorn --bind 0.0.0.0:8080 kittygram_backend.wsgi
Restart=always

[Install]
WantedBy=multi-user.target
```

`Restart=always` — после перезагрузки сервис поднимается сам.

### Порядок на сервере

```bash
sudo systemctl enable gunicorn_kittygram
sudo systemctl start gunicorn_kittygram

cd backend
python manage.py migrate
python manage.py collectstatic          # пути заданы в settings.py под /var/www/kittygram

sudo cp infra/default /etc/nginx/sites-available/default
sudo ln -s /etc/nginx/sites-available/default /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

Сертификаты выпускаются certbot и добавляются в конфиг nginx.

---

## Известные ограничения

- **Тесты зависят от живого сервера.** Это интеграционные тесты, а не юнит-тесты:
  без доступа к развёрнутому Kittygram и соседнему Taski они не пройдут.
  `test_projects_on_same_ip` и `test_link_connection[name_taski]` требуют, чтобы
  рядом работал второй проект.
- `test_projects_on_same_ip` лезет во внутренности urllib3
  (`response.raw._original_response.fp.raw._sock.getpeername()`) и сломается при
  обновлении библиотеки.
- В `infra/default` нет `location /static/`, поэтому собранная статика Django
  Admin через nginx не отдаётся.
- `STATIC_ROOT` и `MEDIA_ROOT` — абсолютные POSIX-пути, `collectstatic` на Windows
  не отработает.
- Модели не зарегистрированы в `admin.py`, поэтому `/admin/` пуст.
- `psycopg2-binary` и `PyYAML` есть в `requirements.txt`, но в коде не используются.
- `pytest-pythonpath` устарел: начиная с pytest 7 опция `pythonpath` встроена,
  и в `pytest.ini` она уже задана.
- Тестовый фронтенд `frontend/src/components/app/app.test.js` остался шаблоном
  Create React App и ищет текст `/learn react/i`, которого в приложении нет, —
  `npm test` в фронтенде падает.

---

## Лицензия

MIT — см. [LICENSE](LICENSE).
