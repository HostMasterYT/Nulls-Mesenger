# Nulls Messenger (UI MVP+)

Обновлённый локальный прототип мессенджера с **огромным разделом настроек** и расширенным backend API.

## Запуск фронтенда

```bash
python3 -m http.server 4173
```

Откройте: <http://localhost:4173>

## Запуск backend

```bash
cd server
npm install
FACEBOOK_APP_ID=1437460264576640 \
FACEBOOK_APP_SECRET=... \
FACEBOOK_REDIRECT_URI=http://localhost:8080/auth/facebook/callback \
FRONTEND_ORIGIN=http://localhost:4173 \
FRONTEND_ORIGINS=http://localhost:4173,http://127.0.0.1:4173,http://0.0.0.0:4173 \
SESSION_SECRET=replace_me \
npm start
```

## Что переписано

- Настройки разделены на крупные секции и вынесены в отдельное модальное окно:
  - профиль,
  - внешний вид,
  - приватность,
  - уведомления,
  - защита,
  - чаты и сообщения.
- Настройки теперь синхронизируются через backend API (`GET/PUT /settings`), а не только в localStorage.
- Добавлены API-эндпоинты для метаданных и безопасности сессии.
- Расширена API-модель чатов:
  - `GET /chats/:chatId/messages?limit&before`
  - `GET /chats/:chatId/stats`
- Добавлена полноценная регистрация локального аккаунта по номеру телефона:
  - `POST /auth/register/request-code` создаёт 6-значный код подтверждения;
  - `POST /auth/register/confirm` активирует аккаунт и выдаёт cookie-сессию;
  - вход выполняется по номеру телефона/username/email и password.


## Работа только через backend (без demo-mode)

Теперь UI не переключается в оффлайн-демо автоматически.
Для регистрации/логина/чатов/синхронизации настроек backend должен быть запущен и доступен по `AUTH_API_BASE` (по умолчанию `http://localhost:8080`).

## Регистрация по номеру телефона

Фронтенд больше не создаёт аккаунт сразу после кнопки «Регистрация». Сначала пользователь вводит username, телефон и пароль, нажимает **«Получить код»**, затем вводит код в поле подтверждения и нажимает **«Подтвердить»**.

В локальной разработке внешний SMS-провайдер не подключён, поэтому backend:

- возвращает код в поле `devCode` ответа `POST /auth/register/request-code`;
- печатает тот же код в консоль сервера строкой `[REGISTRATION_CODE] +7999...: 123456`;
- ограничивает код временем жизни и количеством попыток.

Для production вместо `devCode` нужно подключить SMS-шлюз в функции отправки кода, а `devCode` из ответа убрать.

После подтверждения номера можно:

- входить в аккаунт по `phone + password`;
- искать друзей по номеру через кнопку «＋» и `POST /chats/by-phone`;
- писать сообщения в серверно-синхронизированных чатах.

Для быстрого теста уже есть seed-друзья: `+79990000001` и `+79990000002`, пароль у обоих `123456`.

## OAuth Facebook

- Facebook: `/auth/facebook/start` → официальный `facebook.com`
- App ID по умолчанию в сервере: `1437460264576640`
- Статус Facebook провайдера: `GET /auth/providers/status`

> Важно: зарегистрировать Facebook App может только владелец аккаунта в Meta Developers: <https://developers.facebook.com/>.


### Facebook вход (без JSSDK кнопки)

В приложении используется серверный OAuth-редирект через кнопку `Facebook`:
- UI переводит на `GET /auth/facebook/start`,
- сервер перенаправляет на Facebook и после callback создаёт backend-сессию.

Если в кабинете Meta вы видите сообщение *"Вход через JSSDK отключен"*,
включите параметр **"Вход с SDK JavaScript"** в developers.facebook.com.
Это не требуется для текущего redirect-flow, но необходимо именно для `<fb:login-button>` сценариев.


### Автопоиск backend API

Если `window.__AUTH_API_BASE__` не задан или API недоступен, фронтенд теперь пробует несколько адресов backend (`localhost/127.0.0.1:8080` и текущий host:8080) и показывает список проверенных адресов в статусе `API metadata`.


### Facebook JavaScript SDK Login

На странице подключен JS SDK входа через Facebook с ID приложения `1437460264576640`:
- `FB.init({ appId: '1437460264576640', cookie: true, xfbml: true, version: 'v25.0' })`
- `FB.getLoginStatus(...)`
- `<fb:login-button scope="public_profile,email" onlogin="checkLoginState();">`
- при статусе `connected` запускается серверный OAuth redirect для синхронизации backend-сессии.

Если в Meta показано «Вход через JSSDK отключен», включите параметр **«Вход с помощью SDK для JavaScript»** и добавьте корректные домены/redirect URI в developers.facebook.com.

## API

### Системные
- `GET /api/meta`
- `GET /auth/providers/status`

### Авторизация
- `POST /auth/register/request-code`
- `POST /auth/register/confirm`
- `POST /auth/register` — совместимость: возвращает ошибку `registration_requires_code`
- `POST /auth/login`
- `POST /auth/logout`
- `GET /me`
- `GET /auth/facebook/start`
- `GET /auth/facebook/callback`

### Настройки и безопасность
- `GET /settings`
- `PUT /settings`
- `GET /security/sessions`
- `POST /security/sessions/refresh`

### Чаты
- `POST /chats/by-phone`
- `GET /chats`
- `GET /chats/:chatId/messages`
- `POST /chats/:chatId/messages`
- `GET /chats/:chatId/stats`