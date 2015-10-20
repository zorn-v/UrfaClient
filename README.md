URFAClient 1.1.0
==========

Универсальный PHP клиент для биллинговой системы NetUp UTM5 на основе api.xml

## Установка (composer)
```json
{
    "require":{
        "k-shym/urfa-client": "1.*"
    }
}
```

## Зависимости
- UTM 5.2.1-008+
- PHP 5.3+
- Bcmath
- Filter
- Hash
- OpenSSL
- SimpleXML

## Описание файлов
- URFAClient - главный класс библиотеки
- URFAClient_API - объект предоставляет обращение к функциям из api.xml
- URFAClient_Connection - объект соединения с ядром UTM5
- URFAClient_Packet - объект для подготовки получения/отправки бинарных данных ядру
- URFAClient_Collector - сборщик информации для класса API
- URFAClient_Log - журнал с собранными данными
- admin.crt - сертификат для вызова админских функций
- api.xml - файл с описанием api ядра UTM5

## Описание конфига
- login (required) - логин админа или абонента
- password (required) - пароль админа или абонента соответственно
- address (required) - адрес ядра UTM5
- admin (default: TRUE) - указываем какой пользователь подключается. Если TRUE предоставляет сертификат admin.crt для соединения.
- port (default: 11758) - порт ядра UTM5
- timeout (default: 30) - время ожидания ответа от сервера
- api (default: 'api.xml') - полный путь до файла api.xml
- log (default: FALSE) - сборщик логов. Если TRUE, перехватывает исключения из URFAClient_API.

## Пример
Рассмотрим пример использования библиотеки на примере функции rpcf_add_user_new, у нас есть XML описание:
```xml
<function name="rpcf_add_user_new" id="0x2125">
    <input>
        <string name="login"/>
        <string name="password"/>
        <string name="full_name" default=""/>
        <integer name="is_juridical" default="0"/>
        <string name="jur_address" default=""/>
        <string name="act_address" default=""/>
        <string name="flat_number" default=""/>
        <string name="entrance" default=""/>
        <string name="floor" default=""/>
        <string name="district" default=""/>
        <string name="building" default=""/>
        <string name="passport" default=""/>
        <integer name="house_id" default="0"/>
        <string name="work_tel" default=""/>
        <string name="home_tel" default=""/>
        <string name="mob_tel" default=""/>
        <string name="web_page" default=""/>
        <string name="icq_number" default=""/>
        <string name="tax_number" default=""/>
        <string name="kpp_number" default=""/>
        <string name="email" default=""/>
        <integer name="bank_id" default="0"/>
        <string name="bank_account" default=""/>
        <string name="comments" default=""/>
        <string name="personal_manager" default=""/>
        <integer name="connect_date" default="0"/>
        <integer name="is_send_invoice" default="0"/>
        <integer name="advance_payment" default="0"/>

        <integer name="switch_id" default="0"/>
        <integer name="port_number" default="0"/>
        <integer name="binded_currency_id" default="810"/>

        <integer name="parameters_count" default="size(parameter_value)"/>
        <for name="i" from="0" count="size(parameter_value)">
            <integer name="parameter_id" array_index="i"/>
            <string name="parameter_value" array_index="i"/>
        </for>

        <integer name="groups_count" default="size(groups)"/>
        <for name="i" from="0" count="size(groups)">
            <integer name="groups" array_index="i"/>
        </for>

        <integer name="is_blocked" default="0"/>
        <double name="balance" default="0.0"/>
        <double name="credit" default="0.0"/>
        <double name="vat_rate" default="0.0"/>
        <double name="sale_tax_rate" default="0.0"/>
        <integer name="int_status" default="1"/>
    </input>
    <output>
        <integer name="user_id"/>
        <if variable="user_id" value="0" condition="eq">
            <integer name="error_code"/>
            <string name="error_description"/>
        </if>
        <if variable="user_id" value="0" condition="ne">
            <integer name="basic_account"/>
        </if>
    </output>
</function>
```
И так, нам нужно описать входные параметры (элемент input) в ассоциативный массив.
Если в элементе присутствует атрибут _default_, параметр считается необязательным.
Со скалярными значениями все просто:
```
<integer name="user_id" default="0"/> -> array('user_id' => 1)
<string name="login"/> -> array('login' => 'test')
```
И так далее, порядок параметров неважен.

А вот про циклы расскажу более подробно. Как было замечено, разработчики биллинга не пришли к единому формату описания.
В нашем примере используется count="size(parameter_value)", в других можно встретить название поля счетчика. Отсюда возникает вопрос, какое имя давать параметру для массива?
Поэтому было принято решение использовать имя атрибута счетчика в качестве имени для параметра массива. В нашем случае будет так:
```php
array(
    // ...
    'parameters_count' => array(
        array(
            'parameter_id' => 0,
            'parameter_value' => 'м',
        ),
        array(
            'parameter_id' => 1,
            'parameter_value' => '13.06.2014',
        ),
    ),
    'groups_count' => array(
        array(
            'groups' => 1000,
        ),
        array(
            'groups' => 1001,
        ),
    ),
    // ...
)
```
Если попадется элемент error будет выброшено исключение _XML Described error:_, а далее атрибуты ошибки.

C условиями тоже все просто, если истина, то заходим внутрь. И содержание обрабатывается как описано выше.

В итоге получаем минимальный набор параметров для создания пользователя:
```php
include 'URFAClient/init.php';

$urfa = URFAClient::init(array(
    'login'    => 'init',
    'password' => 'init',
    'address'  => 'localhost',
));

$result = $urfa->rpcf_add_user_new(array(
    'login'=>'test',
    'password'=>'test',
));
```
В переменную $result попадут данные которые описаны в элементе output. Более расширенные примеры смотри в example.php.

## Возможные проблемы
- Тестировалось на версиях биллинга UTM-5.2.1-008-update6 и UTM-5.3-002-update18
- Тестировались не все функции из api.xml
- Не реализована передача типа long для PHP x32
- При обновлении api.xml обязательно проверяйте используемые функции, в них бывают ошибки.

По возникшим проблемам присылайте лог(URFAClient::trace_log()), api.xml и версию биллинговой системы. Удачи!

## История изменений

**v1.1.0**
- Добавлен консольный помощник cmd.php (описание функций api.xml в php array)
- Добавлены XML с описанием API из более ранних версий и отдельные тесты к ним
- Исправлена проблема при одновременном использовании от двух и более экземпляров библиотеки (с поддержкой и без поддержки IPv6)

**v1.0.11**
- Исправлена некорректная работа параметра timeout

**v1.0.10**
- Если данные от ядра биллинговой системы не получены, функции возвращают NULL
- Обновлен api.xml до версии ядра 5.3-002-update18

**v1.0.9**
- Входные параметры тега for стали необязательными
- Исправлено приведение типа при NULL значениях

**v1.0.8**
- Исправлена ошибка при использовании PHP 5.6 (SSLv3)

**v1.0.7**
- Исправлена проблема с проверкой сертификата при использовании PHP 5.6
- Обновлен api.xml до версии ядра 5.3-002-update12

**v1.0.6**
- Добавлен параметр timeout
- Добавлено время выполнения функций в логе

**v1.0.5**
- Преданные аргументы функциям приводятся к нужному типу данных (например для типа integer теперь можно передать строку _array('user_id' => '13')_ )
- Доработана поддержка получения типа long для PHP x32 x64
- Доработана поддержка отправки типа long для PHP x64

**v1.0.4**
- Исправлена ошибка получения отрицательного типа int при использовании PHP x64 и UTM5 x32

**v1.0.3**
- Обновлен api.xml до версии ядра 5.3-002-update9
- Исправлена ошибка для типа данных double
- Доработана поддержка типа данных long в элементе input

**v1.0.2**
- Поправлена поддержка IPv6
- Обновлен api.xml до версии ядра 5.3-002-update8

**v1.0.1**
- Поправлена обработка элемента output