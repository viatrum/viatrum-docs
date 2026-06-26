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

Каждый запрос к `/api/v1/*` должен быть подписан. Актуальный контракт — **Merchant HMAC v2**.

> Важно: этот раздел описывает входящие запросы мерчанта к Viatrum API. Исходящие callback-уведомления от Viatrum мерчанту подписываются отдельно заголовками `X-Viatrum-*`, см. [Callback уведомления](#9-callback-уведомления).

### 2.1. Обязательные headers

| Header | Обязательный | Описание |
|---|---:|---|
| `Content-Type` | Да | `application/json` для запросов с JSON body. |
| `Api-Key` | Да | Публичный ключ мерчанта. Также принимается `Public-Key`. |
| `Nonce` | Да | Числовой nonce. Также принимается legacy header `Expires`. |
| `Signature` | Да | HMAC-SHA512 подпись в hex. |
| `X-Environment` | Рекомендуется | `PRODUCTION`, `SANDBOX` или `TEST`. Если не передан, backend использует `PRODUCTION`. |

Рекомендуемый набор headers:

```http
Content-Type: application/json
Api-Key: <merchant_public_key>
Nonce: <nonce>
Signature: <hmac_sha512_hex>
X-Environment: PRODUCTION
```

### 2.2. `Nonce` для HMAC v2

`Nonce` одновременно используется как защита от replay и как время жизни запроса.

Правила:

1. Значение должно состоять только из цифр.
2. Значение должно быть timestamp в миллисекундах в будущем.
3. Значение должно быть не дальше допустимого окна в будущем. В текущей production-конфигурации окно — 5 минут.
4. Значение должно быть строго больше предыдущего успешно принятого nonce этого мерчанта.

Рекомендуемый генератор:

```javascript
let lastNonce = 0;

function generateNonce() {
  const base = Date.now() + 60_000; // +60 секунд
  const next = Math.max(base, lastNonce + 1);
  lastNonce = next;
  return String(next);
}
```

Не добавляйте к nonce длинные суффиксы вида `Date.now() + counter + random` для новых интеграций. Такой формат поддерживается только временно в legacy-режиме HMAC v1.

### 2.3. Canonical string HMAC v2

Подпись считается от строки:

```text
METHOD
PATH
SORTED_QUERY
SHA256_RAW_BODY
NONCE
```

Где:

| Часть | Описание |
|---|---|
| `METHOD` | HTTP method в верхнем регистре: `GET`, `POST`, `PATCH`, `PUT`, `DELETE`. |
| `PATH` | Только pathname без домена. Например `/api/v1/pay-in`. |
| `SORTED_QUERY` | Query string, отсортированный по ключу и значению, без ведущего `?`. Если query нет — пустая строка. |
| `SHA256_RAW_BODY` | SHA256 от raw body в hex. Для пустого body — SHA256 от пустой строки. |
| `NONCE` | То же значение, которое передается в header `Nonce`. |

Если query-параметров нет, третья строка остается пустой. Например:

```text
POST
/api/v1/pay-in

9f4e8e2c7c6f0b6e0f0e0c4a...
1760000060000
```

### 2.4. Правило для body

Подписывается SHA256 от **raw body ровно в том виде, в котором он отправляется в HTTP-запрос**.

Практическое правило:

1. Сформируйте JSON-строку один раз.
2. Посчитайте SHA256 от этой строки.
3. Эту же строку отправьте как request body.

Не сортируйте JSON body для HMAC v2. Сортировка body относится только к legacy HMAC v1.

Для пустого body используйте SHA256 от пустой строки:

```text
e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855
```

### 2.5. JavaScript пример HMAC v2

```javascript
const crypto = require('crypto');

let lastNonce = 0;

function generateNonce() {
  const base = Date.now() + 60_000;
  const next = Math.max(base, lastNonce + 1);
  lastNonce = next;
  return String(next);
}

function buildSortedQuery(urlOrQuery) {
  const url = new URL(urlOrQuery, 'https://api.example');
  const entries = Array.from(url.searchParams.entries()).sort(
    ([keyA, valueA], [keyB, valueB]) =>
      keyA === keyB
        ? valueA.localeCompare(valueB)
        : keyA.localeCompare(keyB),
  );

  return entries
    .map(([key, value]) => `${encodeURIComponent(key)}=${encodeURIComponent(value)}`)
    .join('&');
}

function signRequest({ method, path, query = '', rawBody = '', nonce, secretKey }) {
  const bodyHash = crypto
    .createHash('sha256')
    .update(rawBody, 'utf8')
    .digest('hex');

  const sortedQuery = buildSortedQuery(`${path}${query ? `?${query}` : ''}`);

  const canonicalString = [
    method.toUpperCase(),
    path,
    sortedQuery,
    bodyHash,
    nonce,
  ].join('\n');

  const signature = crypto
    .createHmac('sha512', secretKey)
    .update(canonicalString, 'utf8')
    .digest('hex');

  return { bodyHash, canonicalString, signature };
}

const body = {
  bankId: 1,
  externalID: 'order_10001',
  currencyId: 1,
  callbackURL: 'https://merchant.example/callbacks/payin',
  amount: '6543.00',
  method: 'CARD',
};

const rawBody = JSON.stringify(body);
const nonce = generateNonce();
const { signature } = signRequest({
  method: 'POST',
  path: '/api/v1/pay-in',
  rawBody,
  nonce,
  secretKey: '<merchant_secret_key>',
});

const headers = {
  'Content-Type': 'application/json',
  'Api-Key': '<merchant_public_key>',
  'Nonce': nonce,
  'Signature': signature,
  'X-Environment': 'PRODUCTION',
};
```

### 2.6. Postman pre-request script HMAC v2

```javascript
let lastNonce = Number(pm.environment.get('last_nonce') || '0');

function generateNonce() {
  const base = Date.now() + 60_000;
  const next = Math.max(base, lastNonce + 1);
  lastNonce = next;
  pm.environment.set('last_nonce', String(next));
  return String(next);
}

function buildSortedQuery() {
  const queryItems = pm.request.url.query ? pm.request.url.query.all() : [];

  const activeItems = queryItems
    .filter((item) => !item.disabled && item.key)
    .map((item) => [
      pm.variables.replaceIn(String(item.key)),
      pm.variables.replaceIn(String(item.value ?? '')),
    ])
    .sort((a, b) => {
      if (a[0] === b[0]) return a[1].localeCompare(b[1]);
      return a[0].localeCompare(b[0]);
    });

  if (activeItems.length === 0) return '';

  const params = new URLSearchParams();
  for (const [key, value] of activeItems) {
    params.append(key, value);
  }
  return params.toString();
}

const crypto = require('crypto-js');

const method = pm.request.method.toUpperCase();
const path = pm.variables.replaceIn(pm.request.url.getPath());
const sortedQuery = buildSortedQuery();
const secretKey = pm.environment.get('merchant_secret_key');

let rawBody = '';
if (pm.request.body && pm.request.body.raw) {
  rawBody = pm.variables.replaceIn(pm.request.body.raw);
}

const bodyHash = crypto
  .SHA256(crypto.enc.Utf8.parse(rawBody))
  .toString(crypto.enc.Hex);

const nonce = generateNonce();
const canonicalString = [method, path, sortedQuery, bodyHash, nonce].join('\n');
const signature = crypto
  .HmacSHA512(canonicalString, secretKey)
  .toString(crypto.enc.Hex);

pm.environment.set('nonce', nonce);
pm.environment.set('signature', signature);

console.log('=== VIATRUM HMAC V2 DEBUG ===');
console.log('Canonical string:', canonicalString);
console.log('Signature:', signature);
console.log('=== END DEBUG ===');
```

Headers в Postman:

```http
Content-Type: application/json
Api-Key: {{merchant_public_key}}
Nonce: {{nonce}}
Signature: {{signature}}
X-Environment: PRODUCTION
```

### 2.7. Legacy HMAC v1 во время миграции

Legacy HMAC v1 поддерживается временно только для обратной совместимости. Новые интеграции должны использовать HMAC v2.

Legacy canonical string:

```text
path + sortedJsonBody + nonce
```

Особенности HMAC v1:

- query string не входит в подпись;
- JSON body сортируется по ключам перед подписью;
- используется HMAC-SHA512;
- legacy nonce может иметь вид `<Date.now()><counter><random>`, например `176000000000000042`;
- полный legacy nonce участвует в подписи и replay-check;
- для проверки времени backend использует первые 13 цифр как timestamp в миллисекундах.

Если мерчант уже использовал длинные legacy nonce, при переходе на HMAC v2 может потребоваться сброс nonce-state на стороне Viatrum, потому что новый timestamp nonce будет численно меньше старого legacy nonce. Согласуйте переход с поддержкой Viatrum.

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

### 9.1. Подпись callback от Viatrum

Viatrum подписывает исходящие callback-уведомления HMAC-SHA256. Мерчант должен проверять подпись по raw body, timestamp и event id.

Headers callback:

```http
Content-Type: application/json
X-Viatrum-Timestamp: <unix_timestamp_seconds>
X-Viatrum-Event-Id: <event_id>
X-Viatrum-Key-Id: <key_id>
X-Viatrum-Signature: v1=<hmac_sha256_hex>
```

Canonical string для проверки подписи:

```text
v1
timestamp
eventId
POST
path?sortedQuery
SHA256_RAW_BODY
```

Где:

| Часть | Описание |
|---|---|
| `v1` | Версия схемы подписи. |
| `timestamp` | Значение header `X-Viatrum-Timestamp`. |
| `eventId` | Значение header `X-Viatrum-Event-Id`. |
| `POST` | HTTP method callback-запроса. |
| `path?sortedQuery` | Path callback URL с отсортированным query string. Домен не включается. |
| `SHA256_RAW_BODY` | SHA256 от raw body callback-запроса в hex. |

Подпись:

```text
HMAC-SHA256(callbackSecret, canonicalString)
```

`X-Viatrum-Signature` должен иметь формат:

```text
v1=<hex_signature>
```

Проверяйте, что timestamp находится в допустимом окне, например ±5 минут, и дедуплицируйте события по `X-Viatrum-Event-Id`.

### 9.2. JavaScript пример проверки callback

```javascript
const crypto = require('crypto');

function extractPathWithSortedQuery(callbackUrl) {
  const url = new URL(callbackUrl);
  url.searchParams.sort();
  const query = url.searchParams.toString();
  return query ? `${url.pathname}?${query}` : url.pathname;
}

function verifyViatrumCallback({
  rawBody,
  callbackUrl,
  callbackSecret,
  timestamp,
  eventId,
  signatureHeader,
  toleranceSeconds = 300,
}) {
  const nowSeconds = Math.floor(Date.now() / 1000);
  const timestampSeconds = Number(timestamp);
  if (!Number.isFinite(timestampSeconds)) return false;
  if (Math.abs(nowSeconds - timestampSeconds) > toleranceSeconds) return false;

  const [version, receivedSignature] = String(signatureHeader || '').split('=');
  if (version !== 'v1' || !/^[a-f0-9]{64}$/i.test(receivedSignature || '')) {
    return false;
  }

  const bodyHash = crypto
    .createHash('sha256')
    .update(rawBody, 'utf8')
    .digest('hex');

  const canonicalString = [
    'v1',
    String(timestamp),
    String(eventId),
    'POST',
    extractPathWithSortedQuery(callbackUrl),
    bodyHash,
  ].join('\n');

  const expectedSignature = crypto
    .createHmac('sha256', callbackSecret)
    .update(canonicalString, 'utf8')
    .digest('hex');

  return crypto.timingSafeEqual(
    Buffer.from(expectedSignature, 'hex'),
    Buffer.from(receivedSignature, 'hex'),
  );
}
```

Важно: используйте именно raw body, полученный HTTP-сервером, а не повторно сериализованный JSON-объект.

### 9.3. PayIn callback

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

### 9.4. PayOut callback

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

### 9.5. Обработка callback на стороне мерчанта

1. Проверяйте `X-Viatrum-Signature` до обработки payload.
2. Проверяйте `X-Viatrum-Timestamp`, чтобы отсекать replay старых callback.
3. Делайте обработку идемпотентной по `X-Viatrum-Event-Id` и/или по `id`/`externalID`/`status`.
4. Возвращайте HTTP `200` после успешной обработки.
5. Не считайте порядок callback гарантированным; всегда проверяйте текущий статус заявки через API при спорных ситуациях.
6. Используйте `externalID` как ваш основной ключ, а `id` — как ID заявки Viatrum.

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
| `2001` | `empty Public Key` | Нет `Api-Key` / `Public-Key`. |
| `2002` | `empty NONCE` | Нет `Nonce` / `Expires`. |
| `2003` | `empty Signature` | Нет `Signature`. |
| `2004` | `request timeout` | `Nonce` меньше текущего времени сервера или legacy timestamp устарел. |
| `2005` | `invalid Signature` | Неверная HMAC-подпись. Проверьте canonical string, secret key, raw body и query. |
| `2006` | `invalid Public Key` | Ключ не найден. |
| `2007` | `invalid NONCE` / `NONCE is too far in the future` / `invalid NONCE format` | `Nonce` уже использован, меньше предыдущего, не является числом, слишком длинный или слишком далеко в будущем. |

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

## 12. Полный пример cURL для `PAYMENT_LINK` с HMAC v2

```bash
BODY='{"bankId":1,"externalID":"order_link_10004","currencyId":1,"callbackURL":"https://merchant.example/callbacks/payin","description":"Payment link order #10004","amount":"1000.00","method":"PAYMENT_LINK"}'
NONCE=$(node -e 'console.log(Date.now() + 60000)')
BODY_HASH=$(printf '%s' "$BODY" | openssl dgst -sha256 -hex | awk '{print $2}')

CANONICAL="POST
/api/v1/pay-in

$BODY_HASH
$NONCE"

SIGNATURE=$(printf '%s' "$CANONICAL" \
  | openssl dgst -sha512 -hmac '<merchant_secret_key>' -hex \
  | awk '{print $2}')

curl -X POST 'https://<your-api-domain>/api/v1/pay-in' \
  -H 'Content-Type: application/json' \
  -H 'Api-Key: <merchant_public_key>' \
  -H "Nonce: $NONCE" \
  -H "Signature: $SIGNATURE" \
  -H 'X-Environment: SANDBOX' \
  --data "$BODY"
```

---

## 13. Краткий чек-лист интеграции

- Получите `Api-Key`/public key и secret key.
- Для новых интеграций используйте HMAC v2: `METHOD\nPATH\nSORTED_QUERY\nSHA256_RAW_BODY\nNONCE`.
- Для каждого запроса генерируйте новый `Nonce`: timestamp в миллисекундах в будущем, обычно `Date.now() + 60000`.
- Подписывайте SHA256 от raw body и отправляйте ровно тот же body.
- Query string в HMAC v2 входит в подпись в отсортированном виде.
- Старый HMAC v1 (`path + sortedJsonBody + nonce`) поддерживается только временно на период миграции.
- Для PayIn используйте `payment_link` как единственное публичное поле ссылки на оплату.
- Для `NSPK` и `PAYMENT_LINK` показывайте ссылку из `payment_link`; `receiver` дублирует ее для UI.
- Проверяйте подпись callback от Viatrum по `X-Viatrum-Signature`.
- Обрабатывайте callback идемпотентно и отвечайте HTTP `200`.
