# Megamind API

***
## Общая информация
**Megamind API** предназначен для управления задачами на **платформе Megamind** с помощью обращений к REST-сервису. API имеет единственную точку входа https://megamind.network/api/. Каждый запрос должен содержать заголовок ```api-key```. Значение ключа ```api-key``` можно получить в **Личном кабинете** платформы **Megamind**.

***
#### Обработка ошибок
Каждый запрос при ошибке возвращает следующий ответ
```
{
    "code": 1,  // код ошибки
    "message": "error message"  // текстовое сообщение об ошибке
}
```

***
#### Получение списка свободных исполнителей
Вызов позволяет получить список исполнителей, вернув для каждого из них в т.ч. уникальный идентификатор, характеристики оборудования, стоимость вычислений.
```
GET /v1/gethwlist
```

Запрос вернет при успехе
```
[
    {
        "hwid": "abcdefghabcdefgh",
        "gpus": ["nVidia 1060 6G", "nVidia 1060 6G"],
        "cpus": ["Intel i7 7700K 3.4 GHz"],
        "ram", 32000,
        "network": "100mbit",
        "calculated_perfomance_rating": 4.5,
        "cost": 0.95
    },
    {
        "hwid": "xyzxyzxyz",
        ...
    },
    ...
]
```
где ```hwid``` - уникальный идентификатор исполнителя, ```gpus```, ```cpus``` - списки установленных на исполнителе GPU и CPU, ```ram``` - объем памяти в Мб, ```network``` - измеренная скорость сети, ```calculated_perfomance_rating``` - измеренный рейтинг производительности, ~~```cost``` - цена за операцию~~.

***
#### Создание задачи
Вызов позволяет создать задачу, запустив образ из **реестра образов Megamind** на заданном исполнителе.
```
POST /v1/addtask

{
    "image": "megamind.network/helloworld:latest",
    "params": "{ \"param1\": \"value1\" }",
    "hwid": "abcdefghabcdefgh"
}
```
где ```image``` - имя образа в **реестре Megamind** в формате ```megamind.network/image_name:version_tag``` (см [dev-client-readme.md](./dev-client-readme.md)), ```params``` - строка произвольного формата ~~в base64~~ для передачи параметров в запускаемый процесс в образе (см [dev-client-readme.md](./dev-client-readme.md)), ```hwid``` - ранее полученный идентификатор исполнителя.

Вызов вернет идентификатор запущенной задачи (```taskid```), по которому в дальнейшем идет различение запущенных задач
```
{
    "taskid": "dssfa21344rsad"
}
```

***
#### Остановка задачи
Вызов позволяет остановить задачу, уничтожив ее и убрав из списка выполняемых задач.
```
DELETE /v1/deltask/<taskid>
```

Вызов вернет
``` 
{
    "code": 0,
    "message": "ok"
}
```

***
#### Получение списка активных задач
Вызов позволяет получить для аккаунта список работающих задач и их состояние - задача может быть запущена, в процессе запуска, завершена и т.д.

```
GET /v1/gettasklist
```

Вызов вернет
```
{
    [
        {
            "taskid": "dssfa21344rsad",
            "ip": "192.168.1.100",
            "image": "megamind.network/helloworld:latest",
            "params": "-param1",
            "state": "setup" | "running" | "stopped" | "error" | "unknown", // сейчас цифра вместо текста, unk = 1, setup = 2, running = 3, stopped = 4, error = 5
            "error_msg": ""
        },
        {
        },
        ...
    ]
}
```
где ```taskid``` - уникальный идентификатор задачи, ```ip``` - адрес исполнителя, ```image``` и ```params``` - образ из репозитория Megamind и параметры, переданные ему при запуске, ```state``` - состояние задачи - готовится, работает, ошибка и т.д. В случае ошибки ```error_msg``` будет содержать ошибку, полученную **платформой Megamind** при разворачивании задачи. Задача считается успешно запущенной, если в течение более 5 секунд в качестве статуса был получен только ```running```. Если были получены другие значения и ```running``` в том числе, это означает что у **платформы** возникли какие-то проблемы с развертыванием задачи - неправильно указано имя образа, задача завершается сразу после запуска или неправильно настроен исполнитель.

***
#### Рестарт задачи
Вызов позволяет перезапустить запущенную задачу. Для вызова нужно передать ранее полученный идентификатор задачи.
```
POST /v1/restarttask/<taskid>
```
Вызов вернет
``` 
{
    "code": 0,
    "message": "ok"
}
```
Контролировать процесс перезапуска можно вызовом ```gettasklist``` и наблюдением за параметром ```state```.

***
#### Получение логов задачи
Вызов позволяет получить последние 20 строк из ```stdout``` задачи в хронологическом порядке.
```
GET /v1/gettasklogs/<taskid>
```
Вызов вернет
```
{
    "taskid": "dssfa21344rsad",
    "logs_b64": "SGVsbG8gd29ybGQh"
}
```
где ```taskid``` - уникальный идентификатор задачи, ```logs_b64``` - логи, закодированные в ```base64```.