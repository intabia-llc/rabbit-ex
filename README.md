## Rabbit retry library - библиотека, позволяющая выполнить переданную функцию над сообщением rabbit, а в случае ошибки отправить сообщение в очередь для повторной их обработки.

* При этом можно настроить задержку повторной отправки сообщения в очередь и
  механизм парковки, позволяющий отправить сообщение в отдельную очередь по
  истечении определенного количества попыток.


* Для использования библиотеки необходимо добавить ее в зависимость и в
  используемом классе создать экземпляр класса RabbitMQRetryExecutor
  (можно сделать его бином и заинжектить в объект, где он используется).


* Если необходимо менять настройки задержки и парковки на ходу (без перезапуска
  приложения), то необходимо добавлять сервис через ObjectFactory
  (ObjectFactory<RabbitMQRetryExecutor>). И указать в application.yml следующие
  настройки Spring Boot Actuator:
  ```
  management:
    endpoints:
      web:
        exposure:
          include: "*"
    endpoint:
      env:
        post:
          enabled: true
  ```
  При локальном запуске через IntelliJ Idea помнить, что параметры нужно менять
  в application.yml, который лежит в /target, а не /src директории


* Конфигурация для механизмов задержки и парковки (точка обмена exchange и ключ
  маршрутизации routing key), механизма неприменения логики повторов для части
  ошибок при обработке сообщений и механизма обработки всех остальных ошибок
  устанавливается при создании объекта **RabbitMQRetryExecutor**. Необходимо
  сконфигурировать отдельный exchange для механизма задержки и отдельную очередь и
  exchange для механизма парковки.


* У **RabbitMQRetryExecutor** есть метод **executeWithRetries**, принимающий сообщение и
  функцию для обработки сообщения.


* Для использования стандартных политик, нужно указать в
  application.yml или application.properties параметры для политик.

### Для политики задержки параметр politic.delay-name принимает значения:

1) `increase` - политика увеличения задержки отправки сообщений в очередь.
   Изначальное время задержки 5 секунд, с каждой отправкой время увеличивается,
   максимальное время 15 секунд
2) `custom-increase` - аналогично increase, но можно задать параметры через
   application (изначальное время задержки указывается через параметр
   politic.custom.delay, максимальное время через politic.custom.max-delay)
3) `custom-const`- политика постоянной задержки отправки сообщений в очередь,
   с указанием параметра через application (время задержки указывается
   через параметр politic.custom.delay, по умолчанию 5 секунд)
4) любое другое значение, в том числе его отсутствие - политика постоянной задержки
   отправки сообщений в очередь. Время задержки 5 секунд

* Задержка указывается в миллисекундах

### Для политик парковки параметр politic.parking-name принимает значения:

1) `short` - политика механизма парковки с небольшим количеством попыток (3 шт.)
2) `long` - политика механизма парковки с большим количеством попыток (50 шт.)
3) `custom` - политика, принимающая максимальное количество попыток через
   application (politic.custom.retries) - (по умолчанию 10 шт.)
4) любое другое значение, в том числе его отсутствие - практически без
   парковки (максимальное количество попыток = Long.MAX_VALUE)

* Необходимо использовать версию RabbitMQ, поддерживающую механизм задержки.

### Для политик игнорирования некоторых ошибок, для которых не будет выполняться обычная обработка (логика повтора), параметр politic.failure-ignoring-name принимает значения:

1) `http-code` - политика игнорирования ошибок типа **HttpResponseFaultException**
   по полю **code**. Список http-кодов указывается в параметре
   politic.failure-ignoring.http-codes списком через запятую, например:
   ```
   politic:
     failure-ignoring:
       http-codes: 400,401,403
   ```
   Среди значений можно указать значения вроде `1xx` или `5XX`, чтобы игнорировать
   все коды 100-199 включительно или 500-599 включительно соответственно.
   Для игнорирования всех кодов с 100 по 599 включительно нужно указать значение `all`.
   Регистр букв не имеет значения.
2) любое другое значение, в том числе его отсутствие - политика, при которой
   обрабатываются все возникающие ошибки

### Для политик обработки неуспешной попытки (если её решено НЕ игнорировать) параметр politic.failure-processing-name принимает значения:

1) `delay-and-parking` - политика, при которой выполняется повтор неуспешного
   действия, а после того, как будет сделано определённое кол-во попыток
   (зависящее от политики парковки **ParkingPolitic**), событие отправится
   в очередь ожиданий (на "парковку")
2) любое другое значение, в том числе его отсутствие - политика, при которой
   повтор неуспешного действия НЕ выполняется, событие сразу отправится
   в очередь ожиданий (на "парковку")

### Build и Push в Nexus

#### Build проекта

```shell
mvn clean install
```

#### Push в nexus

```shell
mvn deploy
```

#### NOTE

Настройки для пуша в nexus можно найти в confluence.

#### Build Docker image:

```shell
docker build --tag=rabbitmq-stage:1.0.0 .
```

#### Запуск RabbitMQ в Docker:

```shell
docker run -d --name rabbit -p 5672:5672 -p 15672:15672 rabbitmq-stage:1.0.0
```

#### Команда для изменения параметров при работающем приложении:

```shell
curl -H "Content-Type: application/json" -X POST -d '{"name":NAME, "value":VALUE}' http://localhost:8080/actuator/env/
```

#### Вместо NAME и VALUE подставляются необходимые параметры. Например, чтобы задать время задержки 20 секунд и переключиться на кастомную политику задержки выполняются следующие команды:

```shell
curl -H "Content-Type: application/json" -X POST -d '{"name":"politic.custom.delay", "value":"20000"}' http://localhost:8080/actuator/env/
```

```shell
curl -H "Content-Type: application/json" -X POST -d '{"name":"politic.delay-name", "value":"custom-const"}' http://localhost:8080/actuator/env/
```

### Настройка десериализатора ответов RabbitMQ

Для игнорирования ошибок во время десериализации ответов из кролика добавьте в
конфигурацию приложения следующее свойство:

```yaml
deserialization-config:
  ignore-deserialization-exceptions: true
```

Вышеупомянутая конфигурация отвечает за создание
**DeserializationProblemHandler**'а, ответственного за обработку ошибок во время
десериализации ответа из **RabbitMQ**.

Данный handler имеет логику игнорирования ошибок во время десериализации
входного сообщения.

По умолчанию свойство **
deserialization-config.ignore-deserialization-exceptions**
имеет значение '**false**'.

### Предложения по доработке библиотеки:
1) Добавить политику игнорирования ошибок по классу исключения
2) Добавить политики, получающие параметры не из application.yml, а через конструктор.

Задача: Внешний сервис может вернуть успех с кодом 200 или ошибку с 4xx или 5xx кодом.
Какой бы статус ни вернулся, после принятия сообщения нужно отправить сообщение в Кролик во внутренние сервисы со статусом ответа от внешнего сервиса.
При этом для статусов 200 или 4xx нужно отправить сообщение сразу, без retry-логики, а для 5xx - только после исчерпания всех retry-попыток.

Проблема: разветвление на 4xx и 5xx происходит внутри retry-библиотеки,
а разветвление на успешный статус (200) и ошибки (4xx, 5xx) приходится делать вне кода библиотеки из-за того,
чтобы библиотека смогла прочитать http-код ошибки, нам приходится перехватывать ошибку
в классе IgnoringErrorsResponseErrorHandler: DefaultResponseErrorHandler(),
пойманное от rest-template и оборачивать его в собственное исключение.
Вопрос: как отправить http-статус в Кролик во внутренние сервисы для каждого сценария?
Была мысль для 4xx и 5xx кодов передавать лямбду с отправкой в Кролик внутрь библиотеки, 
но эта лямбда слишком зависит от контекста, который вне библиотеки - даже если получится, 
то будет очень громоздко - придётся сохранять в исключении дополнительные данные.
Плюс исключение не позволяет хранить в себе поля generic-типов (была мысль хранить в исключении ResponseEntity<Any>)
