# credit-cards/list. Список поданных клиентом ВИ/статусов/черновиков

## Цель
{--Сервис предназначен для получения списка поданных клиентом ВИ, их статусов, и созданных черновиков.Сервис работает в трех режимах:--}
{++Сервис предназначен для другого++}
1. Получение {~~списка ВИ~>массива ~~} и их статусов - по ВИ, отправленным на исполнение. В этом случае, активных черновиков уже не будет, поэтому в ответе сервиса не предполагается вывод информации о черновиках.
2. Получение списка ВИ и черновиков по ним - по ВИ, оформление которых еще не завешено. В этом случае, статусы по заявке от системы-исполнения еще не получены, поэтому в ответе сервиса не предполагается вывод информации о статусах ВИ.
3. Получение всей найденной информации о ВИ - в ответе могут быть ВИ, как отправленные на исполнение, так и ВИ с черновиками.
## Общие требования и ограничения
{==Сервис разрабатывается с учётом общих требований ко всем универсальным сервисам (УС) омниканальной платформы (ОКП) ==}(см. УС. Общие требования ко всем сервисам ОКП). Особое внимание к разделу Нефункциональные требования.
При возникновении ошибок должно происходить логгирование с описанием ошибки (на латинице). При возникновении исключительной ситуации логировать как можно больше информации о ситуации, приведшей к ошибке.
Логирование осуществляется с помощью библиотеки логирования (logging-starter). Лог-запись должна содержать параметры, описанные в библиотеке логирования (см. Система логирования).
Библиотека логирования должна быть настроена так, чтобы осуществлялся сбор всех логов УС (см. /omni-logging. Система логирования).
В качестве идентификатора клиента указывается значение заголовка _GPB-guid_, если оно указано во входных данных. Правила логирования сервиса:
1. Первое логирование обязательно выполняется в точке старта работы сервиса. Логируются входные параметры сервиса в соответствии с правилами, описанными в спецификации сервиса.
2. Рекомендуется логировать факты (т.е. минимум информации) обращений к таким внешним источникам как базы данных (указывать информацию о таблицах, к которым идёт обращение и совершаемом действии (чтение, запись...)) и in-memmory кеш (указывать информацию об имени map, к которой идет обращение и совершаемом действии (чтение, запись...)).
3. Обязательно логировать обращение к внешним системам. При уровне _debug_ логируются все параметры. При уровне _info_ логируются все параметры за исключением массива _data/wills/values _
4. Обязательно логировать ответ от внешней системы. При уровне debug ответ от внешней системы логируется полностью. При уровне _info_ логируется ответ без объекта data (указывается как _"data":\<disguised>_)
5. Последнее логирование обязательно выполняется в точке выхода из сервиса. При уровне _debug_ ответ логируется полностью.

## Связанные документы и ссылки

| Краткое назначение документа / ссылки  | Ссылка на документ                             |
|----------------------------------------|------------------------------------------------|
| Сервис, который вызывает данный сервис | client/orders. Получение списка заявок клиента |
|                                        |                                                |
|                                        |                                                |

## Спецификация сервиса
Код сервиса: credit-cards/list  
HTTP метод POST: ```https://<endpoint>/omni-cc-orders/<version>/credit-cards/list```, где <version> включает в себя информацию о версии сервиса.

!!! note "Входные и выходные параметры описаны в OpenApi спецификации"
    [OpenAPI credit-card/list](openApi.md)


## Подробное описание алгоритма работы

### Этап чтения информации из кэша
УС обращается к кэшу приложения Spring Cache [clientWillList](../../storedge/cache/spring/clientWillList.md), находящемуся в памяти, по ключу, состоящему из параметров \<guid> и \<hash>, разделённых между собой | (U+007C; Dec 124), где (далее по тексту ключ упоминается как _keyWillList_):
{
"guid":
{
"value": "GPB-guid"
},
"hash":
{
"value": "SHA-512(data{})"
}
}
УС анализирует результат работы с кэшем:
Если запись по указанному ключу в кэше найдена, то УС переходит на [Этап формирования ответа](#этап-формирования-ответа)
Если в кэше отсутствуют данные по указанному ключу (или взаимодействие с кэшем не удалось), то в зависимости от значения входного параметра data/mode:
* если _data/mode == "delivery_, то УС переходит на [Этап получения информации об оформленных ВИ](#этап-получения-информации-об-оформленных-ви)
* если _data/mode == "draft"_ то УС переходит на [Этап получения информации о черновиках](#этап-получения-информации-о-черновиках)
* если _data/mode == "all"_, то УС последовательно выполняет логику [Этап получения информации об оформленных ВИ](#этап-получения-информации-об-оформленных-ви) и [Этап получения информации о черновиках](#этап-получения-информации-о-черновиках)

## Этап получения информации об оформленных ВИ

!!! note "**Важно!**"
    УС выполняет данный этап, если _data/mode_ == "delivery" **ИЛИ** _data/mode_ == "all"


### Шаг получения списка оформленных ВИ

УС формирует список rowsacceptedWill[], составленных из данных, полученных в результате следующего запроса к БД (список требуемых параметров описан в таблице ниже):

* УС обращается к omni_cc_orders.accepted_will (далее _acceptedWill_) по ключам SQL ```client_guid == "GPB-guid" AND (create_time BETWEEN "data/period/from"  AND "data/period/to")```
* УС добавляет данные таблицы omni_cc_orders.scenario_will (далее _scenarioWill_) по ключу ```id == acceptedWill.scenario_id```
* УС дополнительно фильтрует данные по записям, где _scenarioWill/id_ находится в перечне [gpb.path.scredit-card-list.scenario.filter][application]
* УС добавляет данные таблицы omni_cc_orders.employee_will  (далее _employee_) по ключу ```order_id == acceptedWill.order_id```
* если ```data/includeValues == true```, то УС добавляет данные таблицы omni_cc_orders.client_order_property  (далее _clientOrderProperty_) по ключу _order_id == acceptedWill.order_id_ И _partition_key == acceptedWill.partition_key_ (поиск по **partition_key** осуществляется только для партиционированных таблиц)

??? abstract "Маппинг данных ответов от БД в rowsacceptedWill[]"
    
    | Поле                        | Источник                       | Комментарий                   |
    |-----------------------------|--------------------------------|-------------------------------|
    | orderId                     | acceptedWill.order_id          | Идентификатор ВИ              |
    | relationOrderId             | acceptedWill.relation_order_id | Идентификатор предыдущего ВИ  |
    | channel                     | acceptedWill.channel           | Код канала                    |
    | scenarioId                  | acceptedWill.scenario_id       | Идентификатор сценария        |
    | scenarioName                | scenarioWill.name              | Наименование сценария         |
    | kindCode                    | scenarioWill.kind_code         |                               |
    | kindName                    | scenarioWill.kind_name         |                               |
    | typeCode                    | scenarioWill.type_code         |                               |
    | typeName                    | scenarioWill.type_name         |                               |
    | order4client                | acceptedWill.order4client      |                               |
    | createTime                  | acceptedWill.accepted_time     |                               |
    | lastModifyTime              | acceptedWill.last_modify_time  |                               | 
    | partitionKey                | acceptedWill.partition_key     |                               |
    | employeeLogin               | employeeWIll.login             |                               |
    | willListConfig              | scenarioWill.will_list_config  |                               |
    | clientOrderPropertyValues[] |                                | массив объектов из параметров |
    | clientOrderProperty.id      |                                |                               |
    | clientOrderProperty.title   |                                |                               |
    | clientOrderProperty.value   |                                |                               |


Если УС полученный rowsacceptedWill[] пуст, то УС переходит на следующий этап. Иначе УС переходит к следующему шагу.

### Шаг получения дополнительной информации о ВИ
УС обогащает массив rowsacceptedWill[] данными таблиц omni_cc_orders.access_will (далее _access_), omni_cc_orders.sign_will (далее _signWill_), omni_cc_orders.label_will(далее _labelWill_) по ключам _order_id == rowsacceptedWill[]/orderId И partition_key == acceptedWill.partitionKey_
<div class="info">
Реализация: переиспользуются требования к <a href="https://confluence.int.gazprombank.ru/pages/viewpage.action?pageId=206736160#inner/will/list.ПолучениеспискаВИичерновиковклиентадляиспользованиявнутреннимисервисами-ТребованияксоставлениюSQLзапросов">inner/will/list.Получение списка ВИ и черновиков клиента для использования внутренними сервисами -Требования к составлению SQL запросов</a>
</div>
#### алгоритм наполнения rowsacceptedWill[]
plantuml
skinparam DefaultFontSize 20

json "rowsacceptedWill[]" as rowsacceptedWill {
"access" :
{
"permission": {
"source:": "<b>access.permission"
},
"lastModifyTime": {
"source:": "<b>access.access_last_modify_time"
}
},
"signWill" :{
"action": {
"source": "signWill.action"
},
"confirmTime": {
"source": "signWill.confirmTime"
}
},
"labels[]":
{
"source": "labelWill.label",
"feature": "в виде массива"
}
}
После обработки всех записей _rowsacceptedWill[]_ УС переходит на следующий шаг.
### Шаг формирования массива ВИ
УС для каждого элемента _rowsacceptedWill[]_ формирует список _deliveryResult[]_ в итоговой структуре данных:
orderId <- rowsacceptedWill[]/orderId :: Идентификатор волеизъявления ::
relationOrderId <- rowsacceptedWill[]/relationOrderId :: Идентификатор связанного волеизъявления ::
channelCode <- rowsacceptedWill[]/channel :: Код канала из справочника кодов каналов ::
isDelivery <- false :: Признак того, что ВИ доставлено до системы исполнения ::
scenarioId <- rowsacceptedWill[]/scenarioId :: Уникальный идентификатор сценария ::
scenarioName <- rowsacceptedWill[]/scenarioName :: Наименование сценария ::
categoryCode <- rowsacceptedWill[]/categoryCode :: Код категории, к которой принадлежит сценарий ::
categoryName <- rowsacceptedWill[]/categoryName :: Наименование категории сценария ::
kindCode <- rowsacceptedWill[]/kindCode :: Код вида сценария ::
kindName <- rowsacceptedWill[]/kindName :: Наименование вида сценария ::
typeCode <- rowsacceptedWill[]/typeCode :: Код типа сценария ::
typeName <- rowsacceptedWill[]/typeName :: Наименование кода сценария ::
order4client <- rowsacceptedWill[]/order4client :: Номер для клиента ::
date <- rowsacceptedWill[]/createTime :: Дата приема ВИ от клиента ::
employee
* login <- rowsacceptedWill[]/employee/Login :: Идентификатор сотрудника, ответственного за ВИ в настоящий момент времени ::
  access
  permission <- rowsacceptedWill[]/access/permission :: Участник процесса подачи ВИ, для которого разрешен доступ к ВИ. ::
  lastModifyTime <- rowsacceptedWill[]/access/last_modify_time :: Время последнего обновления типа доступа (Unix timestamp в мс) ::
  sign <- rowsacceptedWill[]/signWill :: Информация о подписи ВИ. Не заполняется при отсутствии параметра. ::
  labels[] <- rowsacceptedWill[]/labels :: Список меток ВИ ::
  11:51 AM
* values[] <- clientOrderPropertyValues[] _Заполняется только если data/includeValues == true_ :: Абстракция WillConfigValue Массив данных ВИ ::
### Шаг получения статуса ВИ
УС для списка _deliveryResult[]_ обращается в таблицs omni_cc_orders.last_state_will и omni_cc_orders.aggregated_state_params_will  по ключам _order_id==deliveryResult[]/orderId И channel = meta/channel И partition_key = = acceptedWill.partition_key_ за записью (далее _state_ и _stateParams_, соответственно).
Если такой записи не найдено, то УС ищет по ключам, _order_id==deliveryResult[]/orderId И channel = "#EMPTY#" И partition_key = = acceptedWill.partition_key_ (далее _state_ и _stateParams_, соответственно):
Обогащение _deliveryResult[]_
isDelivery <- true :: Признак того, что ВИ доставлено до системы исполнения ::
state :: онтейнер с информацией о текущем статусе ВИ ::
code <- state/code ::  ::
name <- state/name ::  ::
isFinal <- state/is_final ::  ::
flow <- state/flow ::  ::
notification <- state/notification ::  ::
rawId <- state/raw_id ::  ::
setTime <- state/set_time ::  ::
params[]
key <- stateParams[]/param_key ::  ::
value <- stateParams[]/param_value ::  ::
type <- stateParams[]/param_type ::  ::
Иначе УС обогащает _deliveryResult[]_ следующими данными
* isTrash <- true, если найдена запись в таблице omni_operation.client_will_trash по ключу deliveryResult[]/orderId или false при отсутствии записи :: Признак того, что ВИ находится в корзине. ::
  УС переходит на следующий шаг
### Шаг формирования объекта для ответа
УС определяет список уникальных _deliveryResult[]/orderId_ и формирует ассоциативный массив _wills\<key,object>_ у которого ключом key является значение _deliveryResult[]/orderId_, а в качестве значения выступает объект _deliveryResult[]_ у которого  _deliveryResult[]/date_ максимален среди всех найденных _deliveryResult[]_ с указанным _orderId_.
* wills :: Тег-контейнер, вида "уникальный ключ - значение", где в качестве уникального ключа \<key> выступает идентификатор волеизъявления (orderId), а в качестве значения выступает объект \<object> ::
    * \<key> <- deliveryResult[]/order_id :: Идентификатор волеизъявления ::
        * \<object> <- deliveryResult :: Данные волеизъявления ::
          УС переходит на следующий этап.
<div class="note">
Если _data/mode == "delivery"_, то переходит на Этап сохранения результатов запроса в кэш
</div>
## Этап получения информации о черновиках
<div class="note">
УС выполняет данный этап, если _data/mode == "draft"_ <b>ИЛИ</b> _data/mode == "all"_
</div>
### Шаг получения списка черновиков
УС формирует список rowsdraftWill[], составленных из данных, полученных в результате следующего запроса к БД (список требуемых параметров описан в таблице ниже):
* УС обращается к omni_cc_orders.draft_will (далее draftWill) по ключам:
SQL
client_guid == "GPB-guid" AND (create_time BETWEEN "data/period/from"  AND "data/period/to") AND expire_time > now() AND active == true
УС добавляет данные таблицы omni_cc_orders.scenario_will (далее scenarioWill) по ключу id == draftWill.scenario_id
Если во входных параметрах передан объект data/scenario и он не пустой, то УС дополнительно фильтрует данные по записям, где scenarioWill/id == data/scenario/[i]/id, scenarioWill/category_code == data/scenario/[i]/categoryCode, scenarioWill/kind_code == data/scenario/[i]/kindCode, scenarioWill/type_code == data/scenario/[i]/typeCode
УС добавляет данные таблицы omni_cc_orders.employee_will  (далее employee) по ключу order_id == draftWill.order_id
УС добавляет данные таблицы omni_cc_orders.draft_state_will  (далее draftState) по ключу access_key == draftWill.access_key с наиболее поздним значением create_time,
11:51 AM
* УС добавляет данные таблицы omni_cc_orders.client_order_property (далее clientOrderProperty) по ключу order_id ==  draftWill.order_id И partition_key == draftWill.partition_key
<details>
<summary>Маппинг данных ответов от БД в <i>rowsdraftWill[]</i></summary>
accessKey <- draftWill.access_key
orderId <- draftWill.order_id
channel <- draftWill.channel
scenarioId <- draftWill.scenario_id
scenarioName <- scenarioWill.name
categoryCode <- scenarioWill.category_code
categoryName <- scenarioWill.category_name
kindCode <- scenarioWill.kind_code
kindName <- scenarioWill.kind_name
typeCode <- scenarioWill.type_code
typeName <- scenarioWill.type_name
expiryTime <- draftWill.expiry_time
expiryClientTime <- draftWill.expiry_client_time
createTime <- draftWill.create_time 
lastModifyTime <- draftWill.last_modify_time 
partitionKey <- draftWill.partition_key
employeeLogin <- employeeWIll.login
willListConfig <- scenarioWill.will_list_config
clientOrderPropertyValues[] <- clientOrderProperty.json_values в виде массива
stateCode <- draftState.code
stateName <- draftState.name
stateSetTime <- draftState.set_time
</details>
Если итоговый rowsdraftWill[] пуст, то УС переходит на следующий этап. Иначе УС переходит к следующему шагу.
### Шаг получения дополнительной информации о черновике
УС обогащает массив rowsdraftWill[] данными таблиц omni_cc_orders.access_will (далее access), omni_cc_orders.label_will (далее labelWill) по ключам order_id == rowsdraftWill[]/orderId И partition_key == rowsdraftWill.partitionKey
<div class="info">
Реализация: переиспользуются требования к <a href="https://confluence.int.gazprombank.ru/pages/viewpage.action?pageId=206736160#inner/will/list.ПолучениеспискаВИичерновиковклиентадляиспользованиявнутреннимисервисами-ТребованияксоставлениюSQLзапросов">inner/will/list.Получение списка ВИ и черновиков клиента для использования внутренними сервисами -Требования к составлению SQL запросов</a>
</div>
Обогащение rowsdraftWill[]
* access
    * permission <- access.permission
    * lastModifyTime <- access.lastModifyTime
    * labels <- labelWill.label в виде массива
УС переходит на следующий шаг.
### Шаг формирования массива черновиков
УС для каждого элемента rowsdraftWill[] формирует список draftResult[] в итоговой структуре данных:
Наполнение draftResult[]
orderId <-  rowsdraftWill[]/orderId :: Идентификатор волеизъявления ::
channelCode <-  rowsdraftWill[]/channel :: Код канала из справочника кодов каналов ::
isDelivery <- false :: Признак того, что ВИ доставлено до системы исполнения ::
scenarioId <-  rowsdraftWill[]/scenarioId :: Уникальный идентификатор сценария ::
scenarioName <-  rowsdraftWill[]/scenarioName :: Наименование сценария ::
categoryCode <-  rowsdraftWill[]/categoryCode :: Код категории, к которой принадлежит сценарий ::
categoryName <-  rowsdraftWill[]/categoryName :: Наименование категории сценария ::
kindCode <-  rowsdraftWill[]/kindCode :: Код вида сценария ::
kindName <-  rowsdraftWill[]/kindName :: Наименование вида сценария ::
typeCode <-  rowsdraftWill[]/typeCode :: Код типа сценария ::
typeName <-  rowsdraftWill[]/typeName :: Наименование типа сценария ::
date <-  rowsdraftWill[]/createTime  :: Дата создания черновика ::
draft :: Контейнер с информацией о черновике ::
accessKey <-  rowsdraftWill[]/accessKey :: Ключ доступа к черновику ::
expiryDateTime <-  rowsdraftWill[]/expiryTime :: Дата и время до которого черновик является активным (unix время в миллисекундах) ::
expiryClientTime <-  rowsdraftWill[]/expiryClientTime :: Дата и время до которого разрешено поднятие черновика пользователем (unix время в миллисекундах) ::
state :: Контейнер с информацией о статусе черновика ::
code <-  rowsdraftWill[]/stateCode :: Код статуса ::
name <-  rowsdraftWill[]/stateName :: Наименование статуса ::
11:51 AM
setTime <-  rowsdraftWill[]/stateSetTime :: Время установки статуса ::
employee
* login <-   rowsdraftWill[]/employeeLogin :: Идентификатор сотрудника, ответственного за ВИ в настоящий момент времени ::
access :: Информация о праве доступа к ВИ в настоящий момент времени Не заполняется при отсутствии параметров. ::
permission <-   rowsdraftWill[]/access/permission :: Участник процесса подачи ВИ, для которого разрешен доступ к ВИ. ::
lastModifyTime <-   rowsdraftWill[]/access/last_modify_time  :: Время последнего обновления типа доступа (Unix timestamp в мс) ::
labels[] <-  rowsdraftWill[]/labels :: Список меток ВИ ::
values[] <- clientOrderPropertyValues /Заполняется только если data/includeValues == true :: Абстракция WillConfigValue Массив данных ВИ ::
### Шаг формирования объекта для ответа
УС определяет список уникальных _deliveryResult[]/orderId_ и формирует ассоциативный массив _wills\<key,object>_ у которого ключом key является значение _draftResult[]/orderId_, а в качестве значения выступает объект _draftResult[]_
* wills :: Тег-контейнер, вида "уникальный ключ - значение", где в качестве уникального ключа \<key> выступает идентификатор волеизъявления (orderId), а в качестве значения выступает объект \<object> ::
    * \<key> <- draftResult[]/order_id :: Идентификатор волеизъявления ::
        * \<object> <- draftResult :: Данные черновика ::
УС переходит на следующий этап.
## Этап сохранения результатов запроса в кэш
УС сохраняет итоговый список wills в кэш [clientWillList](../../storedge/cache/spring/clientWillList.md) по ключу keyWillList и переходит на следующий этап.
## Этап формирования ответа
УС завершает работу, формируя ответ с параметрами:
status <- success
actualTimestamp <- время формирования ответа
data <- Запись кэша [clientWillList](../../storedge/cache/spring/clientWillList.md) по ключу keyWillList
# Особенности тестирования
1. Учесть описанные особенности переданного во входных параметрах кода канала
:(../../support/styles.md)