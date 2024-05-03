# Smart Pay
#### Описание API для подключения кассы
----
## Общие сведения
Платёжный сервис KD Pay/Smart Pay обеспечивает проведение платежей на основе технологий СБП “me-2-me pull” и Сервиса привязки счета (“Подписка”). Для этого сервис   реализует два основных API: клиентский API и кассовый API. Данный документ описывает кассовый API. 

## Тестовое окружение
Для проведения разработки и отладки сервис использует следующие тестовые окружения: `http://kassa-api.test.r-proc.net`
Для авторизации используется клиентский ssl-сертификат. Чтобы получить сертификат для тестовой среды, напишите запрос на адрес kkt-integration@r-proc.ru

## Описание API
Наиболее полное описание API в формате OpenAPI 3.0 доступно тут [http://kassa-api.r-proc.net/api/docs/v1/](http://kassa-api.test.r-proc.net/api/docs/v1/)
Цель данного документа описать основные сценарии использования кассового API и уточнить детали его использования, первоисточником следуем считать описание в формате OpenAPI 3.0, при расхождения верным следует считать последнее.

## Описание сценариев
### Авторизация
Перед вызовом всех прочих ме.тодов API касса должна авторизоваться с помощью. метода /v1/auth. В ответ касса получит авторизационную куку, которую необходимо передавать при вызове остальных методов API.
### Оплата
Для проведения платежа касса должна считать QR-код, предъявленный покупателем. Полученные данные используются кассой как для взаимодействия с процессингом лояльности, так и для проведения платежа. Структура QR-кода описана ниже.
Сценарий оплаты состоит из нескольких активных действий со стороны кассы:
1. При предъявлении QR-кода касса должна как можно быстрее “зафиксировать” его, чтобы исключить предъявление копии того же QR-кода на другой кассе. Так как покупатель может предъявить QR-код в любой момент (до того, как кассир начал пробивать покупки, в середине этого процесса или в конце), то данная операция не оперирует суммой покупки.
2. Для “фиксации” кода используется метод `/v1/purchase/code/commit`. В качестве параметров необходимо передать данные полученные из QR-кода и уникальный идентификатор данной покупки на данной кассе `kassa_purchase_id`. Он должен быть уникален в рамках каждой кассы. В качестве такого идентификатора можно использовать unix timestamp момента создания нового фискального документа.
3. В ответ касса получает идентификатор покупки на стороне платёжного сервиса, данный идентификатор используется на следующих шагах.
4. После того, как кассир закончил пробивать все покупки касса должна инициировать оплату с помощью метода “KD Pay”/“Smart Pay”. Для этого используется метод `/v1/purchase/release`, важно отметить, что в этот метод в качестве параметра передается сумма покупки.
5. После того, как касса успешна инициировала оплату, она должна в цикле запрашивать статус платежа с помощью метода `/v1/purchase/status`. После того, как касса получила успешный статус платежа, кассир может отдавать покупки покупателю.
6. Касса должна сообщить платежному сервису, что транзакция успешна завершена со стороны кассы. Используется метод `/v1/purchase/kassa/status`.
7. Для отображения истории покупок в мобильном приложении касса должна сохранить данные фискального чека в платежном сервисе. Используется метод `/v1/purchase/fiscal`. В качестве параметров передаются регистрационный номер кассы и фискальный признак чека.

### Структура QR-кода
QR-код, формируется мобильным приложением (МП). Базово QR должен содержать идентификатор в программе лояльности и ОТР для подтверждения платежной транзакции. Так как в прикассовой зоне может не быть стабильного мобильного интернета, то ОТP получаются приложением заранее при каждом запуске. Для уменьшения рисков несанкционированного использования QR-кода OTP передается в зашифрованном виде внутри структуры с метаданными.
QR код представляет JSON структуру следующего вида:

`{
  "id_loyalty": "string",
  "encrypted": "string",
  "sign": "string"
}`

* `id_loyalty` - идентификатор пользователя в программе лояльности, формат определяется мобильным приложением.

* `encrypted` - строка сформированная как результат base64 преобразования зашифрованной строки вида `<otp>.<timestamp>.<user_id>`, где otp - пароль подтверждающий конкретную транзакцию, timestamp - временная метка utc полученная, в момент демонстрации qr в МП. Для шифрования используется публичный ключ платежного сервиса.

* `sign` - строка сформированная как результат base64 преобразования электронной подписи строки вида `<otp>.<timestamp>.<user_id>`, где otp - пароль подтверждающий конкретную транзакцию, timestamp - временная метка utc полученная, в момент демонстрации QR-кода в МП. Для подписи используется приватный ключ МП.

### Отмена оплаты
Если в процессе оплаты покупатель или кассир решают прервать процесс ожидания подтверждения оплаты от платежного сервиса, то касса должна явно уведомить об этом платежный сервис, с помощью метода '/v1/purchase/kassa/status' отправив статус cancel. После получения этого статуса со стороны кассы платежный сервис самостоятельно отменит платеж на своей стороне.

### Возврат
Возврат происходит по стандартным правилам магазина (торговой сети). 
Для непосредственного осуществления возврата касса должна выполнить следующие действия:
1. Запросить возврат с помощью метода `/v1/purchase/refund`. Возврат может быть сделан только в рамках какого-то, ранее проведенного платежа. Поэтому в качестве параметра в метод передается идентификатор платежа, по которому необходимо сделать возврат. Параметр `kassa_refund_id` представляет собой уникальный идентификатор возврата на стороне кассы, по смыслу совпадает с параметром `kassa_purchase_id` и может быть сформирован таким же образом.
2. В ответ касса получает уникальный идентификатор возврата со стороны платежного сервиса, он используется в последующих запросах.
3. Платежный сервис со своей стороны контролирует, что платеж, по которому запрошен возврат существует, был ранее завершен в статусе “Успешен” и общая сумма возвратов не больше исходной суммы платежа.
4. После того, как возврат запрошен, касса в цикле опрашивает платежный сервис, для получения статуса возврата с помощью метода `/v1/purchase/refund/status`.
5. После получения статуса возврата “Успешен” касса должна подтвердить, что возврат осуществлен отправив соответствующий статус в платежный сервис с помощью метода `/v1/purchase/refund/kassa/status`. Касса может прервать ожидание статуса возврата отправис статус cancel (см. Отмена оплаты)

### Отправка данных фискального чека
Кроме непосредственно платежных операций сервис KD Pay/Smart Pay позволяет сохранять на своей стороне данные фискальных чеков, чтобы впоследствии отображать их в мобильном приложении торговой сети. Это позволяет значительно сократить затраты на печать бумажных чеков.
	Для сохранения данных фискального чека используется метод `/v2/purchase/fiscal`. Передается два основных блока данных - данные полученные из QR-кода и данные самого чека. 
Данные чека могут быть переданы независимо от того, были ди покупка оплачена с помощью KD Pay/Smart Pay или другим способом оплаты. 

### Вспомогательные методы
Платежный сервис также реализует несколько вспомогательных методов.
1. Касса может проверить доступность и работоспособность платежного сервиса с помощью метода `/v1/ping`
2. Список всех возможных ошибок можно получить с помощью метода `/v1/error_answers`



