# Viatrum Merchant API

Документация описывает внешний merchant API проекта Viatrum, который обслуживается backend-контроллерами с префиксом `/api/v1/*` и HMAC-аутентификацией.

Внутренние JWT-эндпоинты `/client/*`, `/admin/*`, `/telegram/*` и провайдерские webhook/integration endpoints в этот документ не входят.

---

## 1. Базовый формат

### Base URL

```text
https://<your-api-domain>
```

Все публичные merchant endpoints начинаются с:

```text
/api/v1
```

### Формат успешного ответа

```json
{
  "success": true,
  "data": {}
}
```

### Формат ошибки

```json
{
  "success": false,
  "error": {
    "message": "invalid Signature",
    "code": 2005
  }
}
```

`message` может быть строкой или массивом сообщений валидации. Все `Decimal`-значения в ответах сериализуются строками.

---

## 2. Аутентификация HMAC

Каждый запрос к `/api/v1/*` должен быть подписан.

### Обязательные headers

| Header | Обязательный | Описание |
|---|---:|---|
| `Content-Type` | Да | `application/json` |
| `Public-Key` | Да | Публичный ключ мерчанта. Также поддерживается legacy header `Api-Key`. |
| `Nonce` | Да | Числовое значение, больше предыдущего. Также поддерживается legacy header `Expires`. |
| `Signature` | Да | HMAC-SHA512 подпись. |
| `X-Environment` | Нет | `PRODUCTION`, `SANDBOX` или `TEST`. По умолчанию `PRODUCTION`. |

### Важное про `Nonce`

В текущей реализации `Nonce` одновременно используется как защита от replay и как время жизни запроса:

1. Значение должно быть числом.
2. Значение должно быть строго больше предыдущего успешного `Nonce` этого мерчанта.
3. Значение должно быть больше текущего времени сервера в миллисекундах, иначе будет ошибка `request timeout`.

Практический вариант: передавайте будущий timestamp в миллисекундах, например `Date.now() + 300000`, и добавляйте небольшой монотонный счетчик для параллельных запросов.

```javascript
let counter = 0;

function generateNonce() {
  const expiresAtMs = Date.now() + 5 * 60 * 1000;
  const suffix = String(counter++ % 1000).padStart(3, '0');
  return `${expiresAtMs}${suffix}`;
}
```

### Строка для подписи

```text
path + bodyString + nonce
```

Где:

| Часть | Описание |
|---|---|
| `path` | Только pathname без домена и query string. Например `/api/v1/pay-in/list`. |
| `bodyString` | Для `POST`, `PUT`, `PATCH` — JSON body с отсортированными ключами. Для `GET` — пустая строка. |
| `nonce` | То же значение, которое передается в header `Nonce`. |

Query параметры в текущей backend-реализации **не входят** в подпись. Например запрос:

```text
GET /api/v1/pay-in/list?offset=0&limit=10
```

подписывается как:

```text
/api/v1/pay-in/list{nonce}
```

### JavaScript пример подписи

```javascript
const crypto = require('crypto');

function sortObjectKeys(value) {
  if (value === null || typeof value !== 'object' || Array.isArray(value)) {
    return value;
  }

  return Object.keys(value)
    .sort()
    .reduce((acc, key) => {
      acc[key] = sortObjectKeys(value[key]);
      return acc;
    }, {});
}

function signRequest({ path, method, body, nonce, privateKey }) {
  const shouldSignBody = method.toUpperCase() !== 'GET' && body;
  const bodyString = shouldSignBody
    ? JSON.stringify(sortObjectKeys(body))
    : '';

  const stringToSign = `${path}${bodyString}${nonce}`;
  const signature = crypto
    .createHmac('sha512', privateKey)
    .update(stringToSign)
    .digest('hex');

  return { stringToSign, signature };
}

const body = {
  amount: '1000',
  bankId: 1,
  callbackURL: 'https://merchant.example/callback',
  currencyId: 1,
  externalID: 'order_10001',
  method: 'CARD'
};

const nonce = generateNonce();
const { stringToSign, signature } = signRequest({
  path: '/api/v1/pay-in',
  method: 'POST',
  body,
  nonce,
  privateKey: 'your_private_key'
});
```

---

## 3. Окружения

Окружение выбирается header-ом:

```http
X-Environment: SANDBOX
```

Поддерживаемые значения:

| Значение | Описание |
|---|---|
| `PRODUCTION` | Продакшн. Используется по умолчанию. |
| `SANDBOX` | Песочница. Также включается через `X-Sandbox-Mode: true`. |
| `TEST` | Тестовое окружение. Также включается через `X-Test-Mode: true`. |

Backend добавляет в ответ диагностические headers:

```http
X-Current-Environment: SANDBOX
X-Test-Mode: true
```

---

## 4. Endpoints

| Метод | Endpoint | Описание |
|---|---|---|
| `GET` | `/api/v1/balance` | Баланс мерчанта. |
| `GET` | `/api/v1/banks` | Активные банки текущего окружения. |
| `GET` | `/api/v1/currencies` | Активные валюты. |
| `GET` | `/api/v1/commissions` | Персональные комиссии мерчанта. |
| `POST` | `/api/v1/pay-in` | Создание PayIn заявки. |
| `GET` | `/api/v1/pay-in/list` | Список PayIn заявок. |
| `GET` | `/api/v1/pay-in/{id}` | Получение PayIn заявки по ID. |
| `PATCH` | `/api/v1/pay-in/{id}/cancel` | Отмена PayIn заявки. |
| `POST` | `/api/v1/pay-in/{id}/send-callback` | Ручная отправка callback по PayIn. |
| `POST` | `/api/v1/pay-in/appeals` | Создание апелляции по PayIn. |
| `GET` | `/api/v1/pay-in/appeals` | Список апелляций. |
| `GET` | `/api/v1/pay-in/appeals/{id}` | Апелляция по ID. |
| `GET` | `/api/v1/pay-in/excel-report` | Данные для Excel-отчета. |
| `POST` | `/api/v1/pay-out` | Создание PayOut заявки. |
| `GET` | `/api/v1/pay-out/list` | Список PayOut заявок. |
| `GET` | `/api/v1/pay-out/{id}` | Получение PayOut заявки по ID. |
| `PUT` | `/api/v1/pay-out/{id}/status/{status}` | Обновление статуса PayOut. |

---

## 5. Справочники

### 5.1. Баланс

```http
GET /api/v1/balance
```

#### Пример ответа

```json
{
  "success": true,
  "data": {
    "balance": {
      "payment": {
        "currency": "RUB",
        "available": "10260.76",
        "frozen": "0"
      },
      "payout": {
        "currency": "RUB",
        "available": "5000.00",
        "frozen": "0.00"
      }
    }
  }
}
```

### 5.2. Банки

```http
GET /api/v1/banks
```

Возвращаются только активные банки выбранного окружения.

#### Пример ответа

```json
{
  "success": true,
  "data": [
    {
      "id": 1,
      "name": "Сбер",
      "key": "SBER",
      "currency": "RUB"
    },
    {
      "id": 2,
      "name": "Т-Банк",
      "key": "TBANK",
      "currency": "RUB"
    }
  ]
}
```

### 5.3. Валюты

```http
GET /api/v1/currencies
```

#### Пример ответа

```json
{
  "success": true,
  "data": [
    {
      "id": 1,
      "name": "Рубль",
      "key": "RUB"
    },
    {
      "id": 2,
      "name": "USDT",
      "key": "USDT"
    }
  ]
}
```

### 5.4. Комиссии

```http
GET /api/v1/commissions
```

Возвращает персональные комиссии текущего мерчанта.

#### Пример ответа

```json
{
  "success": true,
  "data": {
    "payIn": [
      {
        "bank": "Сбер",
        "percent": 10.6,
        "min_amount": 1000,
        "max_amount": 100000
      }
    ],
    "payOut": [
      {
        "bank": "Сбер",
        "percent": 5,
        "min_amount": 1000,
        "max_amount": 100000
      }
    ]
  }
}
```

---

## 6. PayIn

### 6.1. Создание PayIn

```http
POST /api/v1/pay-in
```

#### Body

| Поле | Тип | Обязательное | Описание |
|---|---|---:|---|
| `bankId` | number | Да | ID банка из `/api/v1/banks`. |
| `externalID` | string | Да | Уникальный ID заявки в системе мерчанта. 1–64 символа: латиница, цифры, `_`, `-`. |
| `currencyId` | number | Да | ID валюты из `/api/v1/currencies`. |
| `callbackURL` | string | Нет | URL для callback-уведомлений. Должен быть `http` или `https`. |
| `description` | string | Нет | Описание заявки. |
| `amount` | string | Да | Сумма в нативной валюте, максимум 2 знака после точки. |
| `method` | string | Да | Метод PayIn. См. [методы PayIn](#62-методы-payin). |

#### Пример запроса

```json
{
  "bankId": 1,
  "externalID": "order_10001",
  "currencyId": 1,
  "callbackURL": "https://merchant.example/callbacks/payin",
  "description": "Order #10001",
  "amount": "6543.00",
  "method": "CARD"
}
```

#### Универсальный ответ PayIn

```json
{
  "success": true,
  "data": {
    "id": "e42e0768-d913-4b4b-8708-f94cfeaf0777",
    "externalID": "order_10001",
    "trackerID": "provider-operation-id",
    "method": "CARD",
    "amount": "6543",
    "amountUsdt": "80.6683",
    "commission": "4.2692",
    "rate": "81.17",
    "currency": "RUB",
    "bank": "Сбер",
    "status": "PROCESSING",
    "holder": "Иванов Иван Иванович",
    "receiver": "2200154965960000",
    "cardNumber": "2200154965960000",
    "phoneNumber": null,
    "accountNumber": null,
    "payment_link": null,
    "image_qr": null,
    "timeoutAt": "2026-05-12T14:30:00.000Z",
    "description": "Order #10001",
    "createdAt": "2026-05-12T14:15:00.000Z",
    "updatedAt": "2026-05-12T14:15:02.000Z"
  }
}
```

### 6.2. Методы PayIn

| Метод | Описание | Основной реквизит в ответе |
|---|---|---|
| `CARD` | Перевод на карту. | `receiver` / `cardNumber` |
| `SBP` | Перевод по СБП. | `receiver` / `phoneNumber` |
| `ACCOUNT` | Перевод на банковский счет. | `receiver` / `accountNumber` |
| `NSPK` | Оплата по QR/NSPK-ссылке. | `payment_link`; для UI ссылка также дублируется в `receiver`. |
| `PAYMENT_LINK` | Ссылка на оплату / внутрибанковский QR-link. | `payment_link`; для UI ссылка также дублируется в `receiver`. |
| `CROSSBORDER_CARD` | Трансграничный перевод картой. | `receiver` / `cardNumber` |
| `CROSSBORDER_SBP` | Трансграничный перевод по телефону. | `receiver` / `phoneNumber` |
| `M2ARM_SBP` | Армения, перевод по телефону. | `receiver` / `phoneNumber` |
| `M2ABH_SBP` | Абхазия, СБП. | `receiver` / `phoneNumber` |
| `M2TJS_SBP` | Таджикистан, перевод по телефону. | `receiver` / `phoneNumber` |
| `M2ABH_C2C` | Абхазия, карта. | `receiver` / `cardNumber` |
| `M2ARM_C2C` | Армения, карта. | `receiver` / `cardNumber` |
| `M2TJS_C2C` | Таджикистан, карта. | `receiver` / `cardNumber` |
| `C2C_WT` | Карта, white triangle. | `receiver` / `cardNumber` |
| `SBP_WT` | СБП, white triangle. | `receiver` / `phoneNumber` |
| `SBER2SBER` | Внутрибанковский Сбер → Сбер. | `receiver` |
| `ALFA2ALFA` | Внутрибанковский Альфа → Альфа. | `receiver` |
| `VTB2VTB` | Внутрибанковский ВТБ → ВТБ. | `receiver` |
| `TBANK2TBANK` | Внутрибанковский Т-Банк → Т-Банк. | `receiver` |
| `OZON2OZON` | Внутрибанковский Озон → Озон. | `receiver` |
| `SIM` | SIM-реквизит. | `receiver` |
| `ALFA_QR` | Альфа QR. | `payment_link` / `image_qr` |

### 6.3. Контракт ссылок и QR

В публичном API нет отдельного поля `nspk_url`. Любая ссылка для оплаты возвращается в `payment_link`.

| Поле | Назначение |
|---|---|
| `payment_link` | Главная ссылка, которую мерчант может открыть или показать плательщику. Для `NSPK` сюда попадает provider `payment_url`; для `PAYMENT_LINK` — provider `payment_link`. |
| `receiver` | Реквизит получателя. Для `NSPK` и `PAYMENT_LINK` текущий контракт дублирует сюда `payment_link`, чтобы ссылка была видна в UI как реквизит. |
| `image_qr` | Ссылка на изображение QR-кода, если провайдер его возвращает. |
| `cardNumber`, `phoneNumber`, `accountNumber` | Нормализованные реквизиты для классических методов, если доступны. |

Правило для интеграций:

```text
merchant-visible payment link = payment_link
```

Не используйте `paymentLink`, `nspkURL`, `NSPKurl` во внешнем контракте merchant API — это внутренние или старые поля.

### 6.4. Пример PayIn `CARD`

```json
{
  "success": true,
  "data": {
    "id": "9c1f74d7-9b45-4372-aee7-026102d8e8cc",
    "externalID": "order_card_10001",
    "trackerID": "tr_123",
    "method": "CARD",
    "amount": "5000",
    "amountUsdt": "61.58",
    "commission": "3.69",
    "rate": "81.20",
    "currency": "RUB",
    "bank": "Сбер",
    "status": "PROCESSING",
    "holder": "Иванов Иван Иванович",
    "receiver": "2200154965960000",
    "cardNumber": "2200154965960000",
    "phoneNumber": null,
    "accountNumber": null,
    "payment_link": null,
    "image_qr": null,
    "timeoutAt": "2026-05-12T14:30:00.000Z",
    "description": "Order #10001",
    "createdAt": "2026-05-12T14:15:00.000Z",
    "updatedAt": "2026-05-12T14:15:02.000Z"
  }
}
```

### 6.5. Пример PayIn `SBP`

```json
{
  "success": true,
  "data": {
    "id": "93ef6cbf-d1bd-49b0-bc81-31bf1c98f7d2",
    "externalID": "order_sbp_10002",
    "trackerID": "tr_124",
    "method": "SBP",
    "amount": "3500",
    "amountUsdt": "43.10",
    "commission": "2.59",
    "rate": "81.20",
    "currency": "RUB",
    "bank": "Т-Банк",
    "status": "PROCESSING",
    "holder": "Петров Петр Петрович",
    "receiver": "79991234567",
    "cardNumber": null,
    "phoneNumber": "79991234567",
    "accountNumber": null,
    "payment_link": null,
    "image_qr": null,
    "timeoutAt": "2026-05-12T14:30:00.000Z",
    "description": "Order #10002",
    "createdAt": "2026-05-12T14:15:00.000Z",
    "updatedAt": "2026-05-12T14:15:02.000Z"
  }
}
```

### 6.6. Пример PayIn `NSPK`

Для `NSPK` provider `payment_url` отдается мерчанту как `payment_link`. Поле `receiver` содержит ту же ссылку для отображения в UI.

#### Request

```json
{
  "bankId": 1,
  "externalID": "order_nspk_10003",
  "currencyId": 1,
  "callbackURL": "https://merchant.example/callbacks/payin",
  "description": "NSPK order #10003",
  "amount": "1000.00",
  "method": "NSPK"
}
```

#### Response

```json
{
  "success": true,
  "data": {
    "id": "1e6ba508-4b9d-45a0-b297-9f8b14ce7d9b",
    "externalID": "order_nspk_10003",
    "trackerID": "tr_nspk_125",
    "method": "NSPK",
    "amount": "1000",
    "amountUsdt": "12.31",
    "commission": "0.74",
    "rate": "81.20",
    "currency": "RUB",
    "bank": "Сбер",
    "status": "PROCESSING",
    "holder": null,
    "receiver": "https://qr.nspk.ru/BS1A0000000000000000000000000000?type=01&bank=100000000111&crc=ABCD",
    "cardNumber": null,
    "phoneNumber": null,
    "accountNumber": null,
    "payment_link": "https://qr.nspk.ru/BS1A0000000000000000000000000000?type=01&bank=100000000111&crc=ABCD",
    "image_qr": null,
    "timeoutAt": "2026-05-12T14:30:00.000Z",
    "description": "NSPK order #10003",
    "createdAt": "2026-05-12T14:15:00.000Z",
    "updatedAt": "2026-05-12T14:15:02.000Z"
  }
}
```

### 6.7. Пример PayIn `PAYMENT_LINK`

Для `PAYMENT_LINK` provider `payment_link` отдается мерчанту как `payment_link`. Поле `receiver` содержит ту же ссылку для отображения в UI.

#### Request

```json
{
  "bankId": 1,
  "externalID": "order_link_10004",
  "currencyId": 1,
  "callbackURL": "https://merchant.example/callbacks/payin",
  "description": "Payment link order #10004",
  "amount": "1000.00",
  "method": "PAYMENT_LINK"
}
```

#### Response

```json
{
  "success": true,
  "data": {
    "id": "3bff84f7-b54c-4308-81db-a2e7697c7003",
    "externalID": "order_link_10004",
    "trackerID": "tr_link_126",
    "method": "PAYMENT_LINK",
    "amount": "1000",
    "amountUsdt": "12.31",
    "commission": "0.74",
    "rate": "81.20",
    "currency": "RUB",
    "bank": "Альфа-Банк",
    "status": "PROCESSING",
    "holder": null,
    "receiver": "https://pay.example.com/order/3bff84f7-b54c-4308-81db-a2e7697c7003",
    "cardNumber": null,
    "phoneNumber": null,
    "accountNumber": null,
    "payment_link": "https://pay.example.com/order/3bff84f7-b54c-4308-81db-a2e7697c7003",
    "image_qr": "https://pay.example.com/order/3bff84f7-b54c-4308-81db-a2e7697c7003/qr.png",
    "timeoutAt": "2026-05-12T14:30:00.000Z",
    "description": "Payment link order #10004",
    "createdAt": "2026-05-12T14:15:00.000Z",
    "updatedAt": "2026-05-12T14:15:02.000Z"
  }
}
```

### 6.8. Получение списка PayIn

```http
GET /api/v1/pay-in/list?offset=0&limit=10
```

Query параметры:

| Параметр | Тип | По умолчанию | Описание |
|---|---|---:|---|
| `offset` | number | `0` | Смещение. |
| `limit` | number | `10` | Количество записей, максимум `100`. |

#### Пример ответа

```json
{
  "success": true,
  "data": {
    "total": 2,
    "items": [
      {
        "id": "1e6ba508-4b9d-45a0-b297-9f8b14ce7d9b",
        "externalID": "order_nspk_10003",
        "currencyId": 1,
        "status": "PROCESSING",
        "method": "NSPK",
        "bank": "Сбер",
        "amount": "1000",
        "commission": "100",
        "holder": null,
        "receiver": "https://qr.nspk.ru/BS1A...",
        "payment_link": "https://qr.nspk.ru/BS1A...",
        "image_qr": null,
        "createdTime": "2026-05-12T14:15:00.000Z",
        "updatedTime": "2026-05-12T14:15:02.000Z"
      }
    ]
  }
}
```

### 6.9. Получение PayIn по ID

```http
GET /api/v1/pay-in/{id}
```

#### Пример ответа

```json
{
  "success": true,
  "data": {
    "id": "1e6ba508-4b9d-45a0-b297-9f8b14ce7d9b",
    "externalID": "order_nspk_10003",
    "trackerID": "tr_nspk_125",
    "amount": "1000",
    "commission": "100",
    "currency": "RUB",
    "callback_url": "https://merchant.example/callbacks/payin",
    "status": "PROCESSING",
    "method": "NSPK",
    "bank": "Сбер",
    "merchant": {
      "id": "merchant-id",
      "username": "merchant-login"
    },
    "receiver": "https://qr.nspk.ru/BS1A...",
    "payment_link": "https://qr.nspk.ru/BS1A...",
    "image_qr": null,
    "holder": null,
    "cardNumber": null,
    "phoneNumber": null,
    "accountNumber": null,
    "timeoutAt": "2026-05-12T14:30:00.000Z",
    "createdTime": "2026-05-12T14:15:00.000Z",
    "updatedTime": "2026-05-12T14:15:02.000Z"
  }
}
```

### 6.10. Отмена PayIn

```http
PATCH /api/v1/pay-in/{id}/cancel
```

Endpoint переводит заявку в `CANCELLED`, если отмена доступна.

#### Пример ответа

```json
{
  "success": true,
  "data": {
    "id": "1e6ba508-4b9d-45a0-b297-9f8b14ce7d9b",
    "status": "CANCELLED"
  }
}
```

### 6.11. Ручная отправка PayIn callback

```http
POST /api/v1/pay-in/{id}/send-callback
```

Отправляет callback на `callbackURL`, сохраненный в PayIn заявке.

---

## 7. PayOut

### 7.1. Создание PayOut

```http
POST /api/v1/pay-out
```

#### Body

| Поле | Тип | Обязательное | Описание |
|---|---|---:|---|
| `externalID` | string | Да | Уникальный ID выплаты в системе мерчанта. |
| `bank` | string | Да | Название банка. |
| `method` | string | Да | `card`, `sbp`, `account`. |
| `currencyId` | string | Да | Код валюты из 3 заглавных букв, например `RUB`. |
| `callbackURL` | string | Да | URL для callback. |
| `amount` | string | Да | Сумма выплаты, максимум 2 знака после точки. |
| `receiver` | string | Да | Реквизит получателя. |
| `holder` | string | Да | Имя получателя, 3–100 символов. |

> В текущем backend-коде PayOut DTO и сервис используют разные представления банка/валюты. Таблица выше отражает DTO-контракт, который валидируется на входе. Если PayOut используется в production, нужно синхронизировать DTO, validators и service lookup.

#### Пример запроса

```json
{
  "externalID": "payout_10001",
  "bank": "Сбер",
  "method": "card",
  "currencyId": "RUB",
  "callbackURL": "https://merchant.example/callbacks/payout",
  "amount": "5000.00",
  "receiver": "2200154965960000",
  "holder": "Иванов Иван Иванович"
}
```

#### Пример ответа

```json
{
  "success": true,
  "data": {
    "id": "f5ef6b73-0952-4602-a306-82ef1f755f85",
    "externalID": "payout_10001",
    "currencyId": 1,
    "method": "card",
    "amount": "5000",
    "commission": "250",
    "status": "CREATED",
    "receiver": "2200154965960000",
    "holder": "Иванов Иван Иванович",
    "bank": "Сбер"
  }
}
```

### 7.2. Получение списка PayOut

```http
GET /api/v1/pay-out/list?offset=0&limit=10
```

#### Пример ответа

```json
{
  "success": true,
  "data": {
    "total": 1,
    "items": [
      {
        "id": "f5ef6b73-0952-4602-a306-82ef1f755f85",
        "externalID": "payout_10001",
        "currencyId": 1,
        "status": "CREATED",
        "method": "card",
        "bank": "Сбер",
        "amount": "5000",
        "commission": "250",
        "receiver": "2200154965960000",
        "createdTime": "2026-05-12T14:15:00.000Z"
      }
    ]
  }
}
```

### 7.3. Получение PayOut по ID

```http
GET /api/v1/pay-out/{id}
```

#### Пример ответа

```json
{
  "success": true,
  "data": {
    "id": "f5ef6b73-0952-4602-a306-82ef1f755f85",
    "externalID": "payout_10001",
    "trackerID": null,
    "currencyId": 1,
    "method": "card",
    "amount": "5000",
    "commission": "250",
    "status": "CREATED",
    "receiver": "2200154965960000",
    "holder": "Иванов Иван Иванович",
    "bank": "Сбер",
    "createdTime": "2026-05-12T14:15:00.000Z",
    "merchant": {
      "id": "merchant-id",
      "username": "merchant-login",
      "email": "merchant@example.com",
      "callbackUrl": null,
      "callbackSecret": null
    }
  }
}
```

---

## 8. Апелляции PayIn

### 8.1. Создание апелляции

```http
POST /api/v1/pay-in/appeals
```

#### Body

| Поле | Тип | Обязательное | Описание |
|---|---|---:|---|
| `transactionId` | string UUID | Да | ID PayIn заявки. |
| `appealReason` | string | Да | Причина апелляции. |
| `appealText` | string | Нет | Текст до 1000 символов. Обязателен для причины `OTHER`. |
| `receiptUrl` | string | Нет | URL чека/скриншота. |
| `receiptType` | string | Нет | Тип файла, например `jpg`, `png`, `pdf`. |

#### `appealReason`

```text
TRADER_NOT_CONFIRM_PAYMENT
INVOICE_EXPIRED_WITH_PAYMENT
PAYMENT_NOT_RECIEVED
NEW_AMOUNT
OVERPAYMENT
BUYER_PAID_LESS
CONFIRMATION_DOCUMENTS
BANK_ACCOUNT_FROZEN
FRAUD
MALICIOUS_ORDER_CANCELLATION
OTHER
```

#### Пример запроса

```json
{
  "transactionId": "1e6ba508-4b9d-45a0-b297-9f8b14ce7d9b",
  "appealReason": "PAYMENT_NOT_RECIEVED",
  "appealText": "Плательщик отправил оплату, но статус не изменился",
  "receiptUrl": "https://merchant.example/files/receipt-10003.jpg",
  "receiptType": "jpg"
}
```

#### Пример ответа

```json
{
  "success": true,
  "data": {
    "id": "a2b76f3f-97fb-4d17-b9ce-53d5f3fa44f1",
    "transactionId": "1e6ba508-4b9d-45a0-b297-9f8b14ce7d9b",
    "appealState": "APPEALED",
    "appealReason": "PAYMENT_NOT_RECIEVED",
    "appealText": "Плательщик отправил оплату, но статус не изменился",
    "receiptUrl": "https://merchant.example/files/receipt-10003.jpg",
    "receiptType": "jpg",
    "createdAt": "2026-05-12T14:20:00.000Z",
    "updatedAt": "2026-05-12T14:20:00.000Z"
  }
}
```

### 8.2. Список апелляций

```http
GET /api/v1/pay-in/appeals?offset=0&limit=10
```

Query параметры:

| Параметр | Тип | Описание |
|---|---|---|
| `id` | UUID | Фильтр по ID апелляции. |
| `transactionId` | UUID | Фильтр по PayIn ID. |
| `appealState` | string | `NOT_SET`, `APPEALED`, `USER_SUCCESS`, `TRADER_SUCCESS`. |
| `appealReason` | string | Причина апелляции. |
| `offset` | number | Смещение. |
| `limit` | number | Количество, максимум `100`. |

---

## 9. Callback уведомления

Callback отправляется POST-запросом на `callbackURL`, указанный при создании заявки.

### Headers callback

```http
Content-Type: application/json
```

Текущая реализация не подписывает callback отдельным HMAC header-ом. Для безопасности рекомендуется проверять HTTPS endpoint, IP-allowlist и идемпотентность по `id`/`externalID`/`status`.

### PayIn callback

Основной PayIn callback содержит те же ключевые поля, что и ответ создания/получения PayIn.

```json
{
  "id": "1e6ba508-4b9d-45a0-b297-9f8b14ce7d9b",
  "externalID": "order_nspk_10003",
  "trackerID": "tr_nspk_125",
  "trackerId": "1e6ba508-4b9d-45a0-b297-9f8b14ce7d9b",
  "method": "NSPK",
  "amount": "1000",
  "amountUsdt": "12.31",
  "commission": "0.74",
  "rate": "81.20",
  "currency": "RUB",
  "bank": "Сбер",
  "status": "COMPLETED",
  "holder": null,
  "receiver": "https://qr.nspk.ru/BS1A...",
  "payment_link": "https://qr.nspk.ru/BS1A...",
  "image_qr": null,
  "description": "NSPK order #10003",
  "timestamp": "2026-05-12T14:25:00.000Z"
}
```

`trackerId` может дополнительно добавляться сервисом отправки callback. Значение зависит от места отправки: при ручной отправке это может быть ID PayIn, при автоматической — provider tracker.

### PayOut callback

```json
{
  "externalID": "payout_10001",
  "status": "COMPLETED",
  "amount": "5000",
  "currency": 1,
  "method": "card",
  "receiver": "2200154965960000",
  "timestamp": "2026-05-12T14:25:00.000Z",
  "trackerId": "pay-out-f5ef6b73-0952-4602-a306-82ef1f755f85"
}
```

### Обработка callback на стороне мерчанта

1. Возвращайте HTTP `200` после успешной обработки.
2. Делайте обработку идемпотентной: один и тот же статус может прийти повторно.
3. Не считайте порядок callback гарантированным; всегда проверяйте текущий статус заявки через API при спорных ситуациях.
4. Используйте `externalID` как ваш основной ключ, а `id` — как ID заявки Viatrum.

---

## 10. Статусы

### PayIn / PayOut статусы

| Статус | Описание |
|---|---|
| `CREATED` | Заявка создана. |
| `PENDING` | Ожидание получения реквизитов или промежуточная обработка. |
| `PROCESSING` | Реквизиты выданы, ожидается оплата/исполнение. |
| `COMPLETED` | Успешно завершена. |
| `TIMEOUT` | Истекло время оплаты/исполнения. |
| `CANCELLED` | Отменена. |
| `DISPUTE` | Спор. |
| `ERROR` | Ошибка. |
| `INCORRECT_AMOUNT` | Оплата прошла на некорректную сумму. |
| `REFUNDED` | Возврат/откат из завершенного состояния. |

---

## 11. Ошибки

### Частые ошибки аутентификации

| Код | Сообщение | Причина |
|---:|---|---|
| `2001` | `empty Public Key` | Нет `Public-Key` / `Api-Key`. |
| `2002` | `empty NONCE` | Нет `Nonce` / `Expires`. |
| `2003` | `empty Signature` | Нет `Signature`. |
| `2004` | `request timeout` | `Nonce` меньше текущего времени сервера. |
| `2005` | `invalid Signature` | Неверная подпись. |
| `2006` | `invalid Public Key` | Ключ не найден. |
| `2007` | `invalid NONCE` | `Nonce` уже использован или меньше предыдущего. |

### Частые бизнес-ошибки

| Код | Сообщение | Причина |
|---:|---|---|
| `30003` | `Merchant not found` | Мерчант не найден по ключу. |
| `60010` | `externalID already exists` | Дубликат `externalID`. |
| `60011` | `payment doesn't exists` | PayIn не найден. |
| `60013` | `commission doesnt exists` / provider error | Нет комиссии или ошибка обработки. |
| `60014` | `bank doesnt exists` | Банк не найден или недоступен. |
| `60015` | `method doesnt exists` | Метод не поддерживается. |

### Пример ошибки валидации

```json
{
  "success": false,
  "error": {
    "message": [
      "ID может содержать только латинские буквы, цифры, дефис и подчеркивание"
    ],
    "code": 20000
  }
}
```

---

## 12. Полный пример cURL для `PAYMENT_LINK`

```bash
curl -X POST 'https://<your-api-domain>/api/v1/pay-in' \
  -H 'Content-Type: application/json' \
  -H 'Public-Key: your_public_key' \
  -H 'Nonce: 1770000000000001' \
  -H 'Signature: calculated_hmac_sha512_signature' \
  -H 'X-Environment: SANDBOX' \
  -d '{
    "bankId": 1,
    "externalID": "order_link_10004",
    "currencyId": 1,
    "callbackURL": "https://merchant.example/callbacks/payin",
    "description": "Payment link order #10004",
    "amount": "1000.00",
    "method": "PAYMENT_LINK"
  }'
```

---

## 13. Краткий чек-лист интеграции

- Получите `Public-Key` и private/secret key.
- Для каждого запроса генерируйте новый будущий `Nonce`.
- Подписывайте `path + sortedBody + nonce`; query string не включайте.
- Для PayIn используйте `payment_link` как единственное публичное поле ссылки на оплату.
- Для `NSPK` и `PAYMENT_LINK` показывайте ссылку из `payment_link`; `receiver` дублирует ее для UI.
- Обрабатывайте callback идемпотентно и отвечайте HTTP `200`.
