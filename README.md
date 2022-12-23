# light-controller-design
Примерное техническое описание системы для изменения параметров устройств и логгирования этих действий с привязкой к пользователям

## Система для контроля за цветовой температурой светильников

### Описание

Система предназначена для регулирования светового потока (СП) и цветовой температуры (ЦТ) светильников, подключенных к контоллеру Wiren Board 7, посредством протокола MQTT. Каждое действие по изменению СП и ЦТ должно логгироваться в базу данных с привязкой к пользователю, который его осуществил. Для этого должна быть предусмотрена регистрация пользователя в системе.


### Примерный перечень технологий

__Backend__

- Python
- Fastapi Framework
- SQLAlchemy ORM or raw database connector

__Frontend__

- HTML
- CSS
- JS
- _[optional]_ React, Vue, etc

__Server__

- SQL-like DBMS (MySQL, MariaDB, etc)

### Приблизительная диаграмма сущностей БД

```
  --------------------------           ---------------------------------------
|            User             |      |                  Log                    |
| --------------------------  |      |---------------------------------------- |
| AI user_id          integer<|---   | PK AI log_id                  integer   |
| PK email            string  |   |  |       lamp_id                 integer   |
| password_sha256     string  |   |  |       adjusted_at             timestamp |
| is_active           boolean |   |  |       light_flow_lm           integer   |
| name                string  |   |  |       color_temperature_k     integer   |
|                             |    --|--- FK user_id                 integer   |
  --------------------------           ---------------------------------------

    AI == Auto increment
    PK == Primary key
    FK == Foreign key
```

### Функции

#### Регистрация

1. Пользователь вводит свою электронную почту, пароль и отображаемое имя.
1. В таблице `user` создается запись, пароль шифруется с помощью алгоритма sha256, поле `is_active` устанавливается в `false`.
1. На почту пользователю отправляется ссылка для активации пользователя, которая содержит его id (автоматически сгенерированный СУБД) и токен активации (можно сделать sha256 от sha256 пароля).

```
POST /user/register

Request body example: {
    "email": "example@email.org",
    "password": "qwerty1234",
    "name": "V. Pupkin"
}

Responses:

201 CREATED  [successful response]
409 CONFLICT [user with this email already exists]
```

### Активация аккаунта
1. При переходе по ссылке проверяется sha256 от сохраненного sha256 пароля и переданный токен, при совпадении поле is_active в БД устанавливается в `true`.

```
POST /user/confirm/{user_id}?code={activation_token}

Path: user_id           [integer]
Query: activation_token [integer]

Responses:

200 OK        [successful response]
404 NOT FOUND [user with this id doesn't exists]
409 CONFLICT  [user already activated]

```
### Вход в систему

1. Пользователь вводит почту и пароль, происходит их проверка, в случае успеха в header'aх ответа отдается токен доступа (также можно взять sha256 от пароля). Этот токен должен передаваться во header'ax всех запросов, требующих авторизации, и сравниваться с токенами, которые лежат в БД. Это необходимо для подтверждения подлинности пользователя.

```
POST /user/signin

Request body example: {
    "email": "example@email.org",
    "password": "qwerty1234"
}

Responses:

200 OK           [successful response], Headers: {"token": "some-access-token"}
401 UNAUTHORIZED [incorrect email or password]


```

### Получение списка светильников

_Нужно определиться с форматом выходных данных. Это должен быть список сущностей для отрисовки на фронте._

```
GET  /lamps

Headers: {"token": "some-access-token"}

Responses:

200 OK             [successful response], Body: List<Lamp>?
401 UNAUTHORIZED   [incorrect access token]
```
### Задание режима
1. Пользователь задает некоторые параметры светильника на фронте и они передаются в Body запроса _(уточнить, какие данные)_.
2. По протоколу MQTT отправляется сообщение для ламп об изменении их режима на основе переданных данных.
3. Данные записываются в таблицу БД `log`.

```
POST /lamps/{lamp_id}/set-mode

Path: lamp_id [int]
Headers: {"token": "some-access-token"}

Responses:

200 OK [successful response]
403 FORBIDDEN   [incorrect access token]
409 CONFLICT [other errors]

```
