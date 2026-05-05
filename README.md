# API Документация Viatrum

## Краткая логика работы API
API **Viatrum** предоставляет инструменты для интеграции с платёжной системой, обеспечивая управление транзакциями, получение информации о балансе, банках, валютах и комиссиях, а также создание и обработку заявок на приём и выплату средств. Все запросы требуют аутентификации через подпись HMAC-SHA512, включающую путь запроса, тело (при наличии) и уникальный NONCE. Подпись формируется с использованием приватного ключа, а публичный ключ и NONCE передаются в заголовках. Для защиты от повторных запросов NONCE должен быть уникальным и больше предыдущего значения, хранимого в базе. Callback-уведомления отправляются на указанный URL при изменении статуса транзакций.

## Оглавление
- [Введение](#введение)
- [Аутентификация и подпись](#аутентификация-и-подпись)
  - [Генерация подписи](#генерация-подписи)
    - [JavaScript/Node.js](#javascriptnodejs)
    - [PHP](#php)
    - [Python](#python)
  - [Важные особенности NONCE](#важные-особенности-nonce)
  - [Формирование сообщения](#формирование-сообщения)
  - [Обязательные заголовки](#обязательные-заголовки)
- [API Endpoints](#api-endpoints)
  - [Основные эндпоинты](#основные-эндпоинты)
- [Выполнение запросов](#выполнение-запросов)
  - [GET Запрос](#get-запрос)
  - [POST Запрос](#post-запрос)
- [Ответы API](#ответы-api)
  - [Пример успешного ответа](#пример-успешного-ответа)
  - [Пример ответа с ошибкой](#пример-ответа-с-ошибкой)
- [Детальное описание API эндпоинтов](#детальное-описание-api-эндпоинтов)
  - [1. Получение баланса](#1-получение-баланса)
  - [2. Получение списка банков](#2-получение-списка-банков)
  - [3. Получение списка валют](#3-получение-списка-валют)
  - [4. Получение комиссий](#4-получение-комиссий)
  - [5. Создание заявки на прием платежа (PayIn)](#5-создание-заявки-на-прием-платежа-payin)
  - [6. Создание выплаты (PayOut)](#6-создание-выплаты-payout)
  - [7. Получение информации о конкретной заявке](#7-получение-информации-о-конкретной-заявке)
  - [8. Получение заявки PayOut по ID](#8-получение-заявки-payout-по-id)
  - [9. Получение списка заявок](#9-получение-списка-заявок)
- [Статусы транзакций](#статусы-транзакций)
  - [PayIn статусы](#payin-статусы)
  - [PayOut статусы](#payout-статусы)
- [Обработка ошибок](#обработка-ошибок)
  - [Коды ошибок](#коды-ошибок)
- [Тестовые окружения](#тестовые-окружения)
  - [Использование](#использование)
  - [Поддерживаемые методы](#поддерживаемые-методы)
- [Callback уведомления](#callback-уведомления)
  - [Структура callback для PayIn](#структура-callback-для-payin)
  - [Структура callback для PayOut](#структура-callback-для-payout)
  - [Параметры callback](#параметры-callback)
  - [Заголовки callback запроса](#заголовки-callback-запроса)
  - [Безопасность callback уведомлений](#безопасность-callback-уведомлений)
  - [Обработка callback в коде](#обработка-callback-в-коде)
  - [Обработка callback](#обработка-callback)
- [Поддержка](#поддержка)
  

## Введение
Добро пожаловать в документацию по API **Viatrum**. Наш API предназначен для безопасного взаимодействия между внешними системами и сервисами платформы. Он предоставляет доступ к операциям по управлению транзакциями, проверке баланса, созданию платёжных форм и выполнению выплат.

## Аутентификация и Подпись
Для обеспечения безопасности нашего API все запросы должны быть подписаны с использованием подписи (Signature), сгенерированной с помощью приватного ключа (PrivateKey), который мы предоставляем нашим клиентам в Личном Кабинете. Подпись используется для проверки целостности и подлинности запросов.

### Генерация Подписи
Подпись генерируется с использованием алгоритма HMAC-SHA512. Ниже приведены примеры функций генерирования подписи (Signature) на JavaScript, PHP и Python:

#### JavaScript/Node.js
```javascript
const crypto = require('crypto');
// Функция сортировки объекта по ключам
function sortObjectKeys(obj) {
  if (obj === null || typeof obj !== 'object' || Array.isArray(obj)) {
    return obj;
  }
  const sortedObj = {};
  const keys = Object.keys(obj).sort();
  for (const key of keys) {
    sortedObj[key] = sortObjectKeys(obj[key]);
  }
  return sortedObj;
}
function generateSignature(path, body, nonce, privateKey) {
  // Для POST запросов сортируем ключи в body
  let bodyString = '';
  if (body && typeof body === 'object') {
    const sortedBodyObj = sortObjectKeys(body);
    bodyString = JSON.stringify(sortedBodyObj);
  }
  // Формируем строку для подписи: path + body + nonce
  const stringToSign = path + bodyString + nonce;
  // Генерируем подпись HMAC-SHA512
  const signature = crypto.createHmac('sha512', privateKey)
    .update(stringToSign)
    .digest('hex');
  return {
    stringToSign,
    signature,
    body: bodyString
  };
}
// Пример использования для GET запроса
const path = '/api/v1/balance';
const body = null; // Пустое тело для GET запроса
const nonce = generateNonce(); // Уникальное значение
const privateKey = 'your_private_key_here';
const result = generateSignature(path, body, nonce, privateKey);
console.log('String to sign:', result.stringToSign);
console.log('Signature:', result.signature);
// Пример для POST запроса
const postPath = '/api/v1/pay-in';
const postBody = {
  amount: "1000",
  bankId: 1,
  callbackURL: "https://test.com/callback",
  currencyId: 1,
  externalID: "test123",
  method: "CARD"
};
const postResult = generateSignature(postPath, postBody, nonce, privateKey);
console.log('POST String to sign:', postResult.stringToSign);
console.log('POST Signature:', postResult.signature);
```

#### PHP
```php
<?php
function sortObjectKeys($obj) {
    if ($obj === null || !is_array($obj)) {
        return $obj;
    }
    ksort($obj);
    foreach ($obj as $key => $value) {
        $obj[$key] = sortObjectKeys($value);
    }
    return $obj;
}
function generateSignature($path, $body, $nonce, $privateKey) {
    // Для POST запросов сортируем ключи в body
    $bodyString = '';
    if ($body && is_array($body)) {
        $sortedBodyObj = sortObjectKeys($body);
        $bodyString = json_encode($sortedBodyObj, JSON_UNESCAPED_SLASHES);
    }
    // Формируем строку для подписи: path + body + nonce
    $stringToSign = $path . $bodyString . $nonce;
    // Генерируем подпись HMAC-SHA512
    $signature = hash_hmac('sha512', $stringToSign, $privateKey);
    return [
        'stringToSign' => $stringToSign,
        'signature' => $signature,
        'body' => $bodyString
    ];
}
// Пример использования для GET запроса
$path = '/api/v1/balance';
$body = null; // Пустое тело для GET запроса
$nonce = generateNonce(); // Уникальное значение
$privateKey = 'your_private_key_here';
$result = generateSignature($path, $body, $nonce, $privateKey);
echo 'String to sign: ' . $result['stringToSign'] . PHP_EOL;
echo 'Signature: ' . $result['signature'] . PHP_EOL;
// Пример для POST запроса
$postPath = '/api/v1/pay-in';
$postBody = [
    'amount' => "1000",
    'bankId' => 1,
    'callbackURL' => "https://test.com/callback",
    'currencyId' => 1,
    'externalID' => "test123",
    'method' => "CARD"
];
$postResult = generateSignature($postPath, $postBody, $nonce, $privateKey);
echo 'POST String to sign: ' . $postResult['stringToSign'] . PHP_EOL;
echo 'POST Signature: ' . $postResult['signature'] . PHP_EOL;
?>
```

#### Python
```python
import hmac
import hashlib
import json
def sort_object_keys(obj):
    if obj is None or not isinstance(obj, dict):
        return obj
    sorted_obj = {}
    for key in sorted(obj.keys()):
        sorted_obj[key] = sort_object_keys(obj[key])
    return sorted_obj
def generate_signature(path, body, nonce, private_key):
    # Для POST запросов сортируем ключи в body
    body_string = ''
    if body and isinstance(body, dict):
        sorted_body_obj = sort_object_keys(body)
        body_string = json.dumps(sorted_body_obj, separators=(',', ':'))
   
    # Формируем строку для подписи: path + body + nonce
    string_to_sign = path + body_string + str(nonce)
   
    # Генерируем подпись HMAC-SHA512
    signature = hmac.new(
        private_key.encode('utf-8'),
        string_to_sign.encode('utf-8'),
        hashlib.sha512
    ).hexdigest()
   
    return {
        'stringToSign': string_to_sign,
        'signature': signature,
        'body': body_string
    }
# Пример использования для GET запроса
path = '/api/v1/balance'
body = None # Пустое тело для GET запроса
nonce = generate_nonce() # Уникальное значение
private_key = 'your_private_key_here'
result = generate_signature(path, body, nonce, private_key)
print('String to sign:', result['stringToSign'])
print('Signature:', result['signature'])
# Пример для POST запроса
post_path = '/api/v1/pay-in'
post_body = {
    'amount': "1000",
    'bankId': 1,
    'callbackURL': "https://test.com/callback",
    'currencyId': 1,
    'externalID': "test123",
    'method': "CARD"
}
post_result = generate_signature(post_path, post_body, nonce, private_key)
print('POST String to sign:', post_result['stringToSign'])
print('POST Signature:', post_result['signature'])
```

#### C#
```csharp
using System;
using System.Linq;
using Newtonsoft.Json;
using Newtonsoft.Json.Linq;

public static class JsonCanonicalizer
{
    // Канонизация из объекта (POCO → JSON c отсортированными ключами)
    public static string Canonicalize(object body)
    {
        if (body == null) return string.Empty;
        var token = JToken.FromObject(body);
        var sorted = SortToken(token);
        return JsonConvert.SerializeObject(sorted, Formatting.None);
    }

    // Канонизация из JSON-строки
    public static string Canonicalize(string json)
    {
        if (string.IsNullOrWhiteSpace(json)) return string.Empty;
        var token = JToken.Parse(json);
        var sorted = SortToken(token);
        return JsonConvert.SerializeObject(sorted, Formatting.None);
    }

    private static JToken SortToken(JToken token)
    {
        if (token is JObject obj)
        {
            var sortedObj = new JObject();
            foreach (var prop in obj.Properties().OrderBy(p => p.Name, StringComparer.Ordinal))
                sortedObj[prop.Name] = SortToken(prop.Value);
            return sortedObj;
        }
        if (token is JArray arr)
        {
            var newArr = new JArray();
            foreach (var item in arr)
                newArr.Add(SortToken(item));
            return newArr;
        }
        return token; // примитивы как есть
    }
}
```

```csharp
using System;
using System.Security.Cryptography;
using System.Text;

public static class ViatrumSignature
{
    public static string ComputeSignature(string path, string bodyString, string nonce, string privateKey)
    {
        string stringToSign = path + bodyString + nonce;
        using var hmac = new HMACSHA512(Encoding.UTF8.GetBytes(privateKey));
        var hash = hmac.ComputeHash(Encoding.UTF8.GetBytes(stringToSign));
        return BitConverter.ToString(hash).Replace("-", "").ToLowerInvariant();
    }
}
```

### Важные особенности NONCE
**Проблема с NONCE:**
- Система запоминает использованные NONCE для каждого мерчанта в поле `lastNonce` (используется для защиты от replay-атак)
- Повторное использование NONCE приводит к ошибке `invalid NONCE (код 2007)`
- NONCE должен быть больше `lastNonce` в базе данных
- Каждый NONCE может быть использован только один раз

**Решение:**

#### JavaScript/Node.js
```javascript
let counter = 0;
function generateNonce() {
  const timePart = Date.now(); // 13 цифр (мс до 2286)
  const counterPart = (counter++ % 1000).toString().padStart(3, "0"); // 3 цифры
  const randomPart = Math.floor(Math.random() * 100).toString().padStart(2, "0"); // 2 цифры
  return parseInt(`${timePart}${counterPart}${randomPart}`);
}
const nonce = generateNonce();
```

#### PHP
```php
<?php
function generateNonce() {
    static $counter = 0;
    $timePart = (string) round(microtime(true) * 1000); // 13 цифр
    $counterPart = str_pad(($counter++ % 1000), 3, "0", STR_PAD_LEFT); // 3 цифры
    $randomPart = str_pad(mt_rand(0, 99), 2, "0", STR_PAD_LEFT); // 2 цифры
   
    return (int) ($timePart . $counterPart . $randomPart);
}
$nonce = generateNonce();
?>
```

#### Python
```python
import time
import random
def generate_nonce():
    counter = 0
    def inner():
        nonlocal counter
        time_part = str(int(time.time() * 1000)) # 13 цифр
        counter_part = str(counter % 1000).zfill(3) # 3 цифры
        random_part = str(random.randint(0, 99)).zfill(2) # 2 цифры
        counter += 1
        return int(time_part + counter_part + random_part)
    return inner()
nonce = generate_nonce()
```

#### C#
```csharp
using System;
using System.Security.Cryptography;
using System.Threading;

public static class NonceGenerator
{
    private static int _counter = 0; // потокобезопасный счётчик

    // 18-значное значение: TTTTTTTTTTTTT CCC RR → строкой
    public static string GenerateNonce()
    {
        string timePart = DateTimeOffset.UtcNow.ToUnixTimeMilliseconds().ToString(); // 13
        string counterPart = (Interlocked.Increment(ref _counter) % 1000).ToString().PadLeft(3, '0'); // 3
        string randomPart = RandomNumberGenerator.GetInt32(0, 100).ToString().PadLeft(2, '0'); // 2
        return $"{timePart}{counterPart}{randomPart}";
    }

    // Опционально: как long (используйте осторожно)
    public static long GenerateNonceAsLong()
    {
        return long.Parse(GenerateNonce());
    }
}
```

##### Принцип работы
Функция генерирует уникальный идентификатор на основе текущего времени в миллисекундах, счётчика и случайного числа. Это обеспечивает высокую степень уникальности генерируемых значений.

##### Структура nonce
```
TTTTTTTTTTTTTCCC RR
├─ Время (13 цифр) - текущее время в миллисекундах (от 2286 года)
├─ Счетчик (3 цифры) - счётчик (сбрасывается каждый день)
└─ Случайность (2 цифры) - случайное число (от 0 до 99)
```

##### Ключевые параметры
- **Время (timePart):** текущее время в миллисекундах (13 цифр).
- **Счётчик (counterPart):** трёхзначный счётчик, который увеличивается с каждым вызовом функции и сбрасывается после достижения 999.
- **Случайность (randomPart):** двухзначное случайное число (0-99), добавляющее дополнительную энтропию.

##### Возвращаемое значение
Функция возвращает целое число в формате BIGINT UNSIGNED (до 18 цифр), совместимое с большинством СУБД. Пример: ***172325680000000112***.

##### Преимущества подхода
- **Высокая уникальность:**
  - 1,000,000 уникальных значений/мс (1,000 счётчик × 100 случайных)
  - Поддержка до 1,000 RPS без коллизий
- **Совместимость:**
  - Работает с MySQL, PostgreSQL, Redis
  - Автоматически конвертируется в BIGINT
- **Простота реализации:**
  - Не требует синхронизации между серверами
  - Минимальные накладные расходы

##### Минусы использования простого timestamp:
- Не гарантирует уникальность при параллельных запросах
- Может нарушать хронологический порядок
- Уязвим к атакам повторного воспроизведения
- Не масштабируется под высокие нагрузки
- При множественных запросах в одну секунду возникают коллизии
- Отсутствие дополнительной энтропии
- Высокий риск повторного использования NONCE

#### Как работает валидация
1. Система сравнивает новый NONCE с lastNonce в базе данных
2. Новый NONCE должен быть строго больше предыдущего
3. При успешной валидации lastNonce обновляется в базе

### Формирование Сообщения
Сообщение (stringToSign), используемое для генерации подписи, формируется одинаково для всех типов запросов:
**Формула**: `path + body + nonce`
- **path** - путь к эндпоинту (например: `/api/v1/balance`)
- **body** - JSON строка тела запроса (пустая строка для GET запросов)
- **nonce** - уникальное числовое значение

**⚠️ ВАЖНО**: Значение `nonce` ВСЕГДА добавляется в конец сообщения для подписи.

#### <span style="color:red">
*JSON ключи в теле (body) и query параметры запроса должны идти в алфавитном порядке!* </span>

#### Пример формирования сообщения для GET запроса:
- **URL**: `/api/v1/balance`
- **nonce**: `1721585422` - уникальное числовое значение
- **Сообщение для подписи**: `/api/v1/balance1721585422`

#### Пример формирования сообщения для POST запроса:
- **path**: `/api/v1/pay-in`
- **nonce**: `1721585422`
- **Тело запроса**: `{"amount":"1000","bankId":1,"callbackURL":"https://test.com/callback","currencyId":1,"externalID":"test123","method":"CARD"}`
- **Сообщение для подписи**: `/api/v1/pay-in{"amount":"1000","bankId":1,"callbackURL":"https://test.com/callback","currencyId":1,"externalID":"test123","method":"CARD"}1721585422`

### Обязательные заголовки
Каждый запрос к API должен включать следующие заголовки:
- **Content-Type**: `application/json`
- **Public-Key**: Ваш публичный ключ, предоставленный *Viatrum*
- **nonce**: Уникальное числовое значение для предотвращения повторных запросов (должно быть больше предыдущего)
- **Signature**: Подпись HMAC-SHA512, сгенерированная с использованием вашего приватного ключа

#### Пример заголовков
```http
Content-Type: application/json
nonce: 1717025133
Public-Key: your_public_key_here
Signature: 2816894fc8ebe05d47e96eca553ee3ca59863ae8d41a25a42d92b71df5e0e95b4490cfc8ff180e7575c5dbbc643ab3842ca05ae8bbb9f08e57c58cab748f8677
```

## API Endpoints

### Основные эндпоинты
| Метод | Эндпоинт | Описание |
|-------|----------|----------|
| GET | `/api/v1/balance` | Получение баланса |
| GET | `/api/v1/banks` | Получение списка банков |
| GET | `/api/v1/currencies` | Получение списка валют |
| GET | `/api/v1/commissions` | Получение комиссий |
| POST | `/api/v1/pay-in` | Создание заявки на прием платежа |
| GET | `/api/v1/pay-in/list` | Получение списка заявок PayIn |
| GET | `/api/v1/pay-in/{id}` | Получение заявки PayIn по ID |
| POST | `/api/v1/pay-out` | Создание выплаты |
| GET | `/api/v1/pay-out/list` | Получение списка заявок PayOut |
| GET | `/api/v1/pay-out/{id}` | Получение заявки PayOut по ID |

## Выполнение Запросов

### GET Запрос
Пример GET запроса для получения баланса:
```http
GET /api/v1/balance HTTP/1.1
Host: docs.viatrum.pro
Content-Type: application/json
nonce: 1717025133
Public-Key: your_public_key_here
Signature: 2816894fc8ebe05d47e96eca553ee3ca59863ae8d41a25a42d92b71df5e0e95b4490cfc8ff180e7575c5dbbc643ab3842ca05ae8bbb9f08e57c58cab748f8677
```

### POST Запрос
Пример POST запроса для создания PayIn:
```http
POST /api/v1/pay-in HTTP/1.1
Host: docs.viatrum.pro
Content-Type: application/json
nonce: 1717025134
Public-Key: your_public_key_here
Signature: 3336894fc8ebe05d47e96eca553ee3ca59863ae8d41a25a42d92b71df5e0e95b4490cfc8ff180e7575c5dbbc643ab3842ca05ae8bbb9f08e57c58cab748f8677
{
  "bankId": 1,
  "externalID": "test_merchant_id_2",
  "currencyId": 1,
  "callbackURL": "https://example.com/callbacks/payment",
  "description": "Test payment",
  "amount": "1000",
  "method": "CARD"
}
```

## Ответы API
Все ответы от API Viatrum возвращаются в формате JSON. Ответ включает в себя:
- **success** - Статус запроса (true/false)
- **data** - Данные, возвращенные API (только в случае успешного ответа)
- **error** - Данные об ошибке (только в случае ошибочного ответа)

### Пример успешного ответа
```json
{
  "success": true,
  "data": {
    "balance": {
      "payment": {
        "currency": "USDT",
        "available": "10260.76",
        "frozen": "0"
      },
      "payout": {
        "currency": "USDT",
        "available": "0.00",
        "frozen": "0.00"
      }
    }
  }
}
```

### Пример ответа с ошибкой
```json
{
  "success": false,
  "error": {
    "message": "Invalid Signature",
    "code": 2005
  }
}
```

## Детальное описание API эндпоинтов

### 1. Получение баланса
**GET** `/api/v1/balance`
Получение информации о балансе клиента.

#### Параметры запроса
Нет параметров

#### Пример ответа
```json
{
  "success": true,
  "data": {
    "balance": {
      "payment": {
        "currency": "USDT",
        "available": "10260.76",
        "frozen": "0"
      },
      "payout": {
        "currency": "USDT",
        "available": "0.00",
        "frozen": "0.00"
      }
    }
  }
}
```

### 2. Получение списка банков
**GET** `/api/v1/banks`
Получение списка доступных банков для проведения операций.

#### Заголовки
- `X-Environment`: SANDBOX | PRODUCTION (опционально, по умолчанию PRODUCTION)

#### Пример ответа
```json
{
  "success": true,
  "data": [
    {
      "id": 1,
      "name": "МежБанк",
      "key": "ANY_BANK",
      "currency": "RUB"
    }
  ]
}
```

* **ANY_BANK**: Это специальный ключ, обозначающий любой банк. Он позволяет использовать универсальный метод платежа/выплаты, не привязанный к конкретному банку. Рекомендуется для случаев, когда выбор банка не критичен или для автоматизированных систем.


### 3. Получение списка валют
**GET** `/api/v1/currencies`
Получение списка поддерживаемых валют.

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
    },
    {
      "id": 3,
      "name": "USDTУзбекский сом",
      "key": "UZS"
    }
  ]
}
```

### 4. Получение комиссий
**GET** `/api/v1/commissions`
Получение информации о комиссиях (доступно только для ADMIN и SUPER_ADMIN).

#### Заголовки
- `X-Environment`: SANDBOX | PRODUCTION (опционально)

#### Пример ответа
```json
{
  "success": true,
  "data": {
    "payIn": [
      {
        "bank": "МежБанк",
        "percent": 10.6,
        "min_amount": 1000,
        "max_amount": 100000
      },
      {
        "bank": "ПСБ",
        "percent": 10.6,
        "min_amount": 1000,
        "max_amount": 200000
      }
    ],
    "payOut": []
  }
}
```

### 5. Создание заявки на прием платежа (PayIn)
**POST** `/api/v1/pay-in`
Создание новой заявки на прием платежа.

#### Параметры запроса
| Параметр | Тип | Обязательный | Описание | Пример |
|----------|-----|--------------|----------|----------|
| bankId | number | Да | ID банка | 1 |
| externalID | string | Да | Уникальный ID в системе мерчанта (1-64 символа) | "test_merchant_id_2" |
| currencyId | number | Да | ID валюты | 1 |
| callbackURL | string | Нет | URL для получения callback | "https://example.com/callback" |
| description | string | Нет | Описание платежа | "Test payment" |
| amount | string | Да | Сумма платежа | "1000" |
| method | string | Да | Метод платежа: [Поддерживаемые методы](#поддерживаемые-методы) | "CARD" |

#### Пример запроса
```json
{
  "bankId": 1,
  "externalID": "test_merchant_id_2",
  "currencyId": 1,
  "callbackURL": "https://example.com/callbacks/payment",
  "description": "",
  "amount": "6543",
  "method": "CARD"
}
```

#### Описание полей:
- `bankId` (number) - ID банка
- `externalID` (string) - Ваш уникальный ID заявки (1-64 символа, только латинские буквы, цифры, дефис и подчеркивание)
- `currencyId` (number) - ID валюты
- `callbackURL` (string, optional) - URL для получения callback уведомлений
- `description` (string, optional) - Описание платежа
- `amount` (string) - Сумма транзакции (число с не более чем 2 знаками после запятой)
- `method` (string) - Метод перевода: [Поддерживаемые методы](#поддерживаемые-методы)

#### Пример ответа
```json
{
  "success": true,
  "data": {
    "id": "uuid-here",
    "externalID": "test_merchant_id_2",
    "trackerId": "f5ef6b73-0952-4602-a306-82ef1f755f85",
    "status": "PROCESSING",
    "amount": "2296",
    "amountUsdt": "28.4616",
    "commission": "4.2692",
    "rate": "80.67",
    "currency": "RUB",
    "bank": "Озон Банк (Ozon)",
    "method": "CARD",
    "receiver": "2200154965960000",
    "paymentLink": "https://pay.tinkoff.ru/link/1234567890",
    "holder": "Иванов Иван Иванович",
    "description": "Тестовая оплата",
    "createdAt": "2025-01-01T12:00:00Z",
    "updatedAt": "2025-01-01T12:00:00Z"
  }
}
```

### 6. Создание выплаты (PayOut)
**POST** `/api/v1/pay-out`
Создание новой заявки на выплату.

#### Параметры запроса
| Параметр | Тип | Обязательный | Описание | Пример |
|----------|-----|--------------|----------|----------|
| externalID | string | Да | Уникальный ID в системе мерчанта | "test_payout_123" |
| bankId | number | Да | ID банка | 1 |
| method | string | Да | Метод выплаты: CARD, SBP, ACCOUNT | "CARD" |
| currencyId | number | Да | ID валюты | 1 |
| callbackURL | string | Да | URL для получения callback | "https://example.com/callback" |
| amount | string | Да | Сумма выплаты | "5000" |
| receiver | string | Да | Реквизиты получателя | "4000000000000000" |
| holder | string | Да | Имя получателя | "Иванов Иван Иванович" |

#### Пример запроса
```json
{
  "externalID": "test_payout_123",
  "bankId": 1,
  "method": "CARD",
  "currencyId": 1,
  "callbackURL": "https://example.com/callbacks/payout",
  "amount": "5000",
  "receiver": "4000000000000000",
  "holder": "Иванов Иван Иванович"
}
```

#### Описание полей:
- `externalID` (string) - Ваш уникальный ID заявки (1-64 символа, только Латиница, цифры, дефис и подчеркивание)
- `bankId` (number) - ID банка
- `method` (string) - Метод перевода: `CARD`, `SBP`, `ACCOUNT`
- `currencyId` (number) - ID валюты
- `callbackURL` (string) - URL для получения callback уведомлений
- `amount` (string) - Сумма выплаты (число с не более чем 2 знаками после запятой)
- `receiver` (string) - Реквизит получателя (номер карты/телефон/счет)
- `holder` (string) - Имя получателя (3-100 символов)

#### Пример ответа
```json
{
  "success": true,
  "data": {
    "id": "uuid-here",
    "externalID": "test_payout_123",
    "trackerID": "f5ef6b73-0952-4602-a306-82ef1f755f85",
    "status": "PROCESSING",
    "amount": "2296",
    "amountUsdt": "28.4616",
    "commission": "4.2692",
    "rate": "80.67",
    "currency": "RUB",
    "bank": "OZON",
    "method": "CARD",
    "receiver": "4000000000000000",
    "holder": "Иванов Иван Иванович",
    "description": "Тестовая выплата",
    "createdAt": "2025-01-01T12:00:00Z"
  }
}
```

### 7. Получение информации о конкретной заявке
#### Получение заявки PayIn по ID
**GET** `/api/v1/pay-in/{id}`

#### Параметры URL
- `id` - ID заявки в системе Viatrum

#### Пример ответа
```json
{
  "success": true,
  "data": {
    "id": "f5ef6b73-0952-4602-a306-82ef1f755f85",
    "externalID": "test_merchant_id_1",
    "trackerID": "f5ef6b73-0952-4602-a306-82ef1f755f85",
    "status": "COMPLETED",
    "amount": "2296",
    "amountUsdt": "28.4616",
    "commission": "4.2692",
    "rate": "80.67",
    "currency": "RUB",
    "bank": "Озон Банк (Ozon)",
    "method": "CARD",
    "receiver": "2202206212345678",
    "paymentLink": "https://pay.tinkoff.ru/link/1234567890",
    "holder": "IVAN IVANOV",
    "description": "Тестовая оплата",
    "createdAt": "2024-01-01T12:00:00Z",
    "updatedAt": "2024-01-01T12:05:00Z"
  }
}
```

### 8. Получение заявки PayOut по ID
**GET** `/api/v1/pay-out/{id}`

#### Параметры URL
- `id` - ID заявки

#### Пример ответа
```json
{
  "success": true,
  "data": {
    "id": "f5ef6b73-0952-4602-a306-82ef1f755f85",
    "externalID": "test_power_9",
    "trackerID": "f5ef6b73-0952-4602-a306-82ef1f755f85",
    "status": "PENDING",
    "amount": "2296",
    "amountUsdt": "28.4616",
    "commission": "4.2692",
    "rate": "80.67",
    "currency": "RUB",
    "bank": "Озон Банк (Ozon)",
    "method": "CARD",
    "receiver": "2100153962960000",
    "holder": "Иванов Иван Иванович",
    "description": "Тестовая выплата",
    "createdAt": "2025-05-26T20:55:13.968821Z",
    "updatedAt": "2025-05-26T23:55:15.127007+03:00"
  }
}
```

### 9. Получение списка заявок
#### Получение списка PayIn заявок
**GET** `/api/v1/pay-in/list`

#### Получение списка PayOut заявок
**GET** `/api/v1/pay-out/list`

## Статусы транзакций

### PayIn статусы
- `CREATED` - Заявка создана
- `PENDING` - В ожидании реквизитов
- `PROCESSING` - Заявка обрабатывается
- `COMPLETED` - Заявка выполнена
- `TIMEOUT` - Истекло время ожидания оплаты
- `CANCELLED` - Заявка отменена
- `ERROR` - Ошибка при обработке
- `INCORRECT_AMOUNT` - Некорректная сумма
- `REFUNDED` - Возвращен (выполнен откат сделки из состояния COMPLETED)

### PayOut статусы
- `CREATED` - Заявка создана
- `PENDING` - Заявка в обработке
- `PROCESSING` - Заявка обрабатывается
- `COMPLETED` - Заявка выполнена
- `TIMEOUT` - Истекло время ожидания оплаты
- `CANCELLED` - Заявка отменена
- `ERROR` - Ошибка при обработке
- `INCORRECT_AMOUNT` - Некорректная сумма
- `REFUNDED` - Возвращен (выполнен откат сделки из состояния COMPLETED)

## Обработка Ошибок
В случае ошибки ответ будет включать:
- **message**: Описание ошибки
- **code**: Код ошибки

### Коды ошибок
#### Ошибки аутентификации (2000-2999)
| Код ошибки | Сообщение | HTTP статус |
|------------|-----------|-------------|
| 2004 | request timeout | 401 |
| 2005 | invalid Signature | 401 |
| 2007 | invalid NONCE | 401 |

#### Общие ошибки (10000-19999)
| Код ошибки | Сообщение | HTTP статус |
|------------|-----------|-------------|
| 10000 | unauthorized | 401 |

#### Ошибки валидации (20000-29999)
| Код ошибки | Сообщение | HTTP статус |
|------------|-----------|-------------|
| 20000 | wrong input | 400 |
| 20001 | can't bind body to request model | 422 |
| 20002 | can't bind query parameters | 422 |
| 20003 | failed to parse key | 422 |
| 20004 | signature header value missing or malformed | 400 |
| 20005 | public-Key header value missing or malformed | 400 |
| 20006 | nonce header value missing or outdated | 400 |
| 20012 | invalid query params | 400 |
| 20015 | conflict | 409 |
| 20016 | empty external ID | 400 |

#### Ошибки доступа (30000-39999)
| Код ошибки | Сообщение | HTTP статус |
|------------|-----------|-------------|
| 30000 | forbidden | 403 |
| 30001 | no access to requested session | 403 |
| 30002 | requested sessions has expired | 403 |
| 30003 | user doesn't exists | 403 |
| 30004 | zero balance | 403 |
| 30005 | not enough balance | 402 |
| 30006 | amount less than min | 400 |
| 30007 | amount greater than max | 400 |

#### Внутренние ошибки (40000-49999)
| Код ошибки | Сообщение | HTTP статус |
|------------|-----------|-------------|
| 40000 | internal error | 500 |

#### Ошибки ресурсов (60000-69999)
| Код ошибки | Сообщение | HTTP статус |
|------------|-----------|-------------|
| 60003 | empty Public-Key | 401 |
| 60004 | empty nonce | 401 |
| 60005 | empty Signature | 401 |
| 60007 | request timeout | 408 |
| 60008 | invalid Public-Key | 400 |
| 60009 | empty external ID | 400 |
| 60010 | external ID already exists | 409 |
| 60011 | payment doesn't exists | 404 |
| 60012 | payment is finalized | 409 |
| 60013 | commission doesnt exists | 400 |
| 60014 | bank doesnt exists | 400 |
| 60015 | method doesnt exists | 400 |

## Тестовые окружения
Система поддерживает работу с различными окружениями через заголовок `X-Environment`:
- **SANDBOX** - песочница для тестирования
- **PRODUCTION** - продакшн окружение (по умолчанию)

### Использование
Добавьте заголовок в ваши запросы:
```http
X-Environment: SANDBOX
```

### Поддерживаемые методы
#### Методы для PayIn:
- **CARD** - Платежи банковскими картами
- **SBP** - Система быстрых платежей
- **ACCOUNT** - Переводы на банковские счета
- **NSPK** - Платежи через QR код (НСПК)
- **CROSSBORDER_CARD** - Международные переводы банковскими картами
- **CROSSBORDER_SBP** - Международные переводы СБП
- **M2ARM_SBP** - Трансграничный перевод в Армению по номеру телефона
- **M2ABH_SBP** - Трансграничный перевод в Абхазию по СБП
- **M2TJS_SBP** - Трансграничный перевод в Таджикистан по СБП
- **M2ABH_C2C** - Трансграничный перевод в Абхазию по реквизитам карты
- **M2ARM_C2C** - Трансграничный перевод в Армению по реквизитам карты
- **M2TJS_C2C** - Трансграничный перевод в Таджикистан по реквизитам карты
- **PAYMENT_LINK** - Ссылка на оплату
- **C2C_WT** - На карту (белый треугольник)
- **SBP_WT** - СБП (белый треугольник)
- **SBER2SBER** - Сбер2Сбер
- **ALFA2ALFA** - Альфа2Альфа
- **VTB2VTB** - ВТБ2ВТБ
- **TBANK2TBANK** - Тинькофф2Тинькофф
- **OZON2OZON** - Озон2Озон
- **SIM** - SIM
- **ALFA_QR** - Альфа QR

#### Методы для PayOut:
- **CARD** - Выплаты на банковские карты
- **SBP** - Выплаты через Систему быстрых платежей
- **ACCOUNT** - Выплаты на банковские счета

## Callback уведомления
Система отправляет POST запросы на указанный `callbackURL` при изменении статуса транзакции.

### Структура callback для PayIn:
```json
{
  "id": "e42e0768-d913-4b4b-8708-f94cfeaf0777",
  "externalID": "test_merchant_id_1",
  "trackerID": "e42e0768-d913-4b4b-8708-f94cfeaf0777",
  "status": "COMPLETED",
  "amount": "2296",
  "amountUsdt": "28.4616",
  "commission": "4.2692",
  "rate": "80.67",
  "currency": "RUB",
  "bank": "Озон Банк (OZON)",
  "method": "CARD",
  "receiver": "2200154965960000",
  "paymentLink": "https://pay.tinkoff.ru/link/1234567890",
  "holder": "Иванов Иван Иванович",
  "description": "Тестовая транзакция",
  "timestamp": "2024-01-01T12:00:00Z"
}
```

### Структура callback для PayOut:
```json
{
  "id": "f5ef6b73-0952-4602-a306-82ef1f755f85",
  "externalID": "test_merchant_id_2",
  "trackerId": "f5ef6b73-0952-4602-a306-82ef1f755f85",
  "status": "COMPLETED",
  "amount": "2296",
  "amountUsdt": "28.4616",
  "commission": "4.2692",
  "rate": "80.67",
  "currency": "RUB",
  "bank": "Озон Банк (OZON)",
  "method": "CARD",
  "receiver": "4000000000000000",
  "holder": "Иванов Иван Иванович",
  "description": "Тестовая транзакция",
  "timestamp": "2024-01-01T12:00:00Z"
}
```

### Параметры callback
| Параметр | Тип | Описание |
|----------|-----|----------|
| externalID | string | Ваш уникальный ID транзакции |
| trackerId | string | Опциональный ID для отслеживания (если передан) |
| status | string | Новый статус транзакции |
| amount | string | Сумма транзакции |
| amountUsdt | string | Сумма транзакции в USDT |
| commission | string | Комиссия |
| rate | string | Курс |
| currency | string | Валюта |
| bank | string | Банк |
| method | string | Метод платежа/выплаты |
| receiver | string | Реквизиты получателя |
| paymentLink | string | Ссылка на оплату |
| holder | string | Имя держателя карты |
| description | string | Описание транзакции |
| timestamp | string | Время изменения статуса в формате ISO 8601 |

### Заголовки callback запроса
```http
Content-Type: application/json
User-Agent: Viatrum-Callback/1.0
```

### Безопасность callback уведомлений
Для обеспечения безопасности рекомендуется:
1. **Проверка IP-адресов** - Ограничить доступ к callback URL только с IP-адресов Viatrum
2. **HTTPS** - Использовать только защищенные HTTPS URL для callback
3. **Валидация данных** - Проверять корректность полученных данных
4. **Идемпотентность** - Обрабатывать возможные дублирующиеся уведомления
5. **Таймауты** - Отвечать на callback запросы в течение 30 секунд

### Обработка callback в коде
#### Пример для PHP:
```php
<?php
$input = file_get_contents('php://input');
$data = json_decode($input, true);
if ($data && isset($data['externalID'], $data['status'])) {
    // Обработка уведомления
    updateTransactionStatus($data['externalID'], $data['status']);
  
    // Возврат успешного ответа
    http_response_code(200);
    echo json_encode(['status' => 'success']);
} else {
    http_response_code(400);
    echo json_encode(['error' => 'Invalid data']);
}
?>
```

#### Пример для Node.js:
```javascript
app.post('/callback', express.json(), (req, res) => {
    const { externalID, status, amount, currency } = req.body;
  
    if (externalID && status) {
        // Обработка уведомления
        updateTransactionStatus(externalID, status);
      
        res.json({ status: 'success' });
    } else {
        res.status(400).json({ error: 'Invalid data' });
    }
});
```

#### Пример для Python:
```python
from flask import Flask, request, jsonify
app = Flask(__name__)
@app.route('/callback', methods=['POST'])
def callback():
    data = request.get_json()
    if data and 'externalID' in data and 'status' in data:
        # Обработка уведомления
        update_transaction_status(data['externalID'], data['status'])
       
        # Возврат успешного ответа
        return jsonify({'status' => 'success'}), 200
    else:
        return jsonify({'error' => 'Invalid data'}), 400
if __name__ == '__main__':
    app.run(port=3000)
```

### Обработка callback
1. **Ваш сервер должен отвечать HTTP 200** для подтверждения получения
2. **Время ожидания ответа**: 30 секунд
3. **Повторные попытки**: В случае ошибки система выполнит до 3 повторных попыток
4. **Безопасность**: Рекомендуется проверять IP-адрес отправителя

### Пример обработки callback (PHP)
```php
<?php
// Получаем данные callback
$input = file_get_contents('php://input');
$data = json_decode($input, true);
if ($data) {
    $externalID = $data['externalID'];
    $status = $data['status'];
    $amount = $data['amount'];
  
    // Обновляем статус транзакции в вашей системе
    updateTransactionStatus($externalID, $status);
  
    // Возвращаем успешный ответ
    http_response_code(200);
    echo 'OK';
} else {
    http_response_code(400);
    echo 'Invalid data';
}
?>
```

### Пример обработки callback (Node.js)
```javascript
const express = require('express');
const app = express();
app.use(express.json());
app.post('/callback', (req, res) => {
    const { externalID, status, amount, currency, method, timestamp } = req.body;
  
    // Обновляем статус транзакции в вашей системе
    updateTransactionStatus(externalID, status);
  
    // Возвращаем успешный ответ
    res.status(200).send('OK');
});
app.listen(3000, () => {
    console.log('Callback server running on port 3000');
});
```

### Пример обработки callback (Python)
```python
from flask import Flask, request
app = Flask(__name__)
@app.route('/callback', methods=['POST'])
def callback():
    data = request.get_json()
    if data and 'externalID' in data and 'status' in data:
        external_id = data['externalID']
        status = data['status']
        amount = data['amount']
       
        # Обновляем статус транзакции в вашей системе
        update_transaction_status(external_id, status)
       
        # Возвращаем успешный ответ
        return 'OK', 200
    else:
        return 'Invalid data', 400
if __name__ == '__main__':
    app.run(port=3000)
```

---
*Данная документация регулярно обновляется. Следите за изменениями и новыми возможностями API.*
