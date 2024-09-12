# Подключение GigaChat и запуск Node.js TypeScript сервера

В этом руководстве мы рассмотрим, как настроить и запустить сервер на Node.js с использованием TypeScript, а также как подключиться к API GigaChat.

## Шаг 1: Клонирование репозитория и установка зависимостей

Первым шагом в настройке вашего проекта будет клонирование репозитория. Это позволит вам получить все необходимые файлы и структуру проекта, чтобы начать работу. Для этого выполните следующую команду в терминале:

```bash
git clone https://github.com/gHashTag/gigachat-typescript-server.git
```

Эта команда создаст локальную копию репозитория на вашем компьютере. 
После успешного клонирования репозитория, перейдите в директорию проекта:

```bash
cd gigachat-typescript-server
```

Теперь, когда вы находитесь в корневой директории проекта, вам нужно установить все необходимые зависимости. Эти зависимости включают в себя библиотеки, которые будут использоваться в вашем проекте для различных задач, таких как создание сервера, выполнение HTTP-запросов и работа с переменными окружения. Для установки зависимостей выполните следующую команду:


```bash
npm install axios dotenv express
```

## Шаг 2: Создайте проект GigaChat API

Проект GigaChat API нужен для получения авторизационных данных, а также для управления платными опциями. С помощью авторизационных данных вы можете получить токен доступа для авторизации запросов к API.

> Авторизационные данные — строка, полученная в результате кодирования в Base64 клиентского идентификатора (Client ID) и ключа (Client Secret) API. Вы можете использовать готовые данные из личного кабинета или самостоятельно закодировать идентификатор и ключ.

Подробнее о том, как получить авторизационные данные, можно узнать в [документации GigaChat](https://developers.sber.ru/docs/ru/gigachat/individuals-quickstart).

Создайте файл `.env` в корневой директории проекта и добавьте в него следующие переменные:

```bash
GIGACHAT_API_URL=https://gigachat.devices.sberbank.ru/api/v1/chat/completions
GIGACHAT_CLIENT_ID=
GIGACHAT_CLIENT_SECRET=
```


## Шаг 3: Создание сертификата
Использование сертификатов НУЦ Минцифры является важным шагом для обеспечения безопасности вашего приложения при работе с GigaChat API. Следуя этим инструкциям, вы сможете настроить ваше приложение для безопасного обмена данными с сервисом GigaChat.

Чтобы установить публичный сертификат НУЦ Минцифры в коде вашего приложения, выполните следующие шаги:

1. Перейдите на портал Госуслуг и скачайте сертификат для вашей операционной системы.
2. Скачайте сертификат `russiantrustedca.pem` и сохраните его в корневой директории проекта `/src/russiantrustedca.pem`.
3. Укажите в коде вашего приложения путь к сертификату. Для Node.js это можно сделать следующим образом:

```typescript
const certPath = path.resolve(__dirname, 'russiantrustedca.pem');
const cert = fs.readFileSync(certPath);

const httpsAgent = new https.Agent({
    ca: cert
});
```

Для получения дополнительной информации и подробных инструкций, посетите [документацию GigaChat](https://developers.sber.ru/docs/ru/gigachat/certificates).


## Шаг 4: Запуск сервера
В этом разделе мы рассмотрим, как запустить сервер для вашего проекта GigaChat API.
В файле `/src/index.ts` добавьте следующий код:

```typescript
import dotenv from 'dotenv';
import express from 'express';
import axios from 'axios';
import https from 'https';
import fs from 'fs';
import path from 'path';
import querystring from 'querystring';

dotenv.config();

const app = express();
const port = 3000;

app.use(express.json());

// Укажите путь к вашему сертификату
const certPath = path.resolve(__dirname, 'russiantrustedca.pem');
const cert = fs.readFileSync(certPath);

const httpsAgent = new https.Agent({
    ca: cert
});

if(!process.env.GIGACHAT_API_URL || !process.env.GIGACHAT_CLIENT_ID || !process.env.GIGACHAT_CLIENT_SECRET) {
    console.error('GIGACHAT_API_URL, GIGACHAT_CLIENT_ID или GIGACHAT_CLIENT_SECRET не установлены');
    process.exit(1);
}

async function getAccessToken() {
    try {
        const data = querystring.stringify({
            scope: 'GIGACHAT_API_PERS'
        });

        const clientSecret = process.env.GIGACHAT_CLIENT_SECRET;
        const authHeader = `Basic ${clientSecret}`;
        const rqUid = process.env.RQ_UID;
        const response = await axios.post('https://ngw.devices.sberbank.ru:9443/api/v2/oauth', data, {
            headers: {
                'Content-Type': 'application/x-www-form-urlencoded',
                'Accept': 'application/json',
                'RqUID': rqUid,
                'Authorization': authHeader
            },
            httpsAgent
        });

        return response.data.access_token;
    } catch (error: any) {
        console.error('Ошибка при получении токена доступа:', error.response ? error.response.data : error.message);
        throw error;
    }
}



app.post('/chat', async (req, res) => {
    const { message } = req.body;

    try {
        const token = await getAccessToken();
        if(!process.env.GIGACHAT_API_URL) {
            console.error('GIGACHAT_API_URL не установлен');
            process.exit(1);
            
        }
        const response = await axios.post(process.env.GIGACHAT_API_URL, {
            model: 'GigaChat',
            messages: [
                {
                    role: 'user',
                    content: message
                }
            ],
            stream: false,
            repetition_penalty: 1
        }, {
            headers: {
                'Content-Type': 'application/json',
                'Accept': 'application/json',
                'Authorization': `Bearer ${token}`
            },
            httpsAgent
        });

        res.json(response.data);
    } catch (error: any) {
        console.error(error);
        res.status(500).send('Ошибка при обращении к GigaChat API');
    }
});

app.listen(port, () => {
    console.log(`Сервер запущен на http://localhost:${port}`);
});
```

Запустите сервер с помощью команды:

```bash
npm run start:dev 
```                         

## Шаг 5: Проверка работы сервера

Откройте браузер и перейдите по адресу `http://localhost:3000`. Если все сделано правильно, вы увидите ответ от сервера.


## Шаг 7: Использование API

Теперь вы можете использовать API для получения ответов на ваши запросы.

```bash
curl -X POST http://localhost:3000/chat \
     -H "Content-Type: application/json" \
     -d '{"message": "Привет, как дела?"}'
```

## Шаг 9: Заключение

Вы успешно настроили и запустили сервер на Node.js с использованием TypeScript и подключили его к API GigaChat.

Теперь вы можете использовать API для получения ответов на ваши запросы.

Если у вас возникнут какие-либо вопросы или проблемы, обратитесь в чат нашей поддержки [НейроКодер](https://t.me/neuro_coder_group).
