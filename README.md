# TON Deep Documentation
Документация по внутренним компонентам ТОНа, которая поможет разобраться, как все работает.

## ADNL
Это протокол нижнего уровня, на котором построено все взаимодействие в сети TON, он может работать поверх любого протокола, но чаще всего применяется поверх TCP и UDP. UDP применяется для общения между нодами, а TCP для коммуникации с lite-клиентами. В данный момент в документации делается упор только на TCP реализацию.

Вместо адресов, узлы сети используют публичные ключи ed25519 и устанавливают соединение, используя общий ключ, полученный с помощью процедуры Диффи-Хелмана для эллиптических кривых - ECDH.

### Структура пакетов
Каждый ADNL пакет, кроме хэндшейка, имеет структуру:
* 4 байта размера пакета в little endian (N)
* 32 байта nonce [[?]](## "Случайные байты, для защиты от атак на чексумму")
* (N - 64) байт полезных данных
* 32 байта чексумма SHA256 от nonce и полезных данных

Весь пакет, включая размер, зашифрован **AES-CTR**.
После расшифровки - нужно обязательно проверить, сходится ли чексумма с данными, для проверки нужно просто посчитать чексумму самостоятельно и сравнить результат с тем, что у нас в пакете.

Хэндшейк пакет - исключение, он передается в частично открытом виде и описан в следующей главе.


### Установка соединения
Для установки соединения нам нужно знать ip, порт и публичный ключ сервера, и сгенерировать свой приватный и публичный ключ ed25519. 

Данные публичных серверов такие как ip, порт и ключ можно получить из [конфига](https://ton-blockchain.github.io/global.config.json). Айпи в конфиге в числовом виде, в нормальный вид его можно привести используя, например [этот инстурмент](https://www.browserling.com/tools/dec-to-ip). Публичный ключ в конфиге в base64 формате.

Клиент генерирует 160 случайных байт, часть из которых будет использоваться сторонами в качестве основы для AES шифрования. 

Из них создаются 2 постоянных AES-CTR шифра, которые будут использоваться сторонами для шифрования/дешифрования сообщений после хэндшейка.
* Шифр A - ключ 0 - 31 байты, iv 64 - 79 байты
* Шифр B - ключ 32 - 63 байты, iv 80 - 95 байты

Шифры применяются в таком порядке:
* Шифр A используется сервером для шифрования отправляемых сообщений.
* Шифр A используется клиентом для дешифрования полученных сообщений.
* Шифр B используется клиентом для шифрования отправляемых сообщений.
* Шифр B используется сервером для дешифрования полученных сообщений.

Для установки соединения клиент должен отправить хэндшейк пакет, содержащий:
* [32 байта] **Айди ключа сервера** [[Подробнее]](#%D0%BF%D0%BE%D0%BB%D1%83%D1%87%D0%B5%D0%BD%D0%B8%D0%B5-%D0%B0%D0%B9%D0%B4%D0%B8-%D0%BA%D0%BB%D1%8E%D1%87%D0%B0)
* [32 байта] **Наш публичный ключ ed25519**
* [32 байта] **SHA256 хэш от наших 160 байт**
* [160 байт] **Наши 160 байт в зашифрованом виде** [[Подробнее]](#%D1%88%D0%B8%D1%84%D1%80%D0%BE%D0%B2%D0%B0%D0%BD%D0%B8%D0%B5-%D0%B4%D0%B0%D0%BD%D0%BD%D1%8B%D1%85-handshake-%D0%BF%D0%B0%D0%BA%D0%B5%D1%82%D0%B0)


При получении хэндшейк пакета, сервер проделает те же самые действия у себя, получит ECDH ключ, расшифрует 160 байт и создаст 2 постоянных ключа. Если все получится, сервер ответит пустым ADNL пакетом, без полезных данных, для дешифровки которого (а также последующих) нужно использовать один из постоянных шифров. 

С этого момента можно считать соединение установленным.

## Обмен данными и TL схемы
После того, как мы установили соединение, мы можем приступать к получению информации, для сериализации данных используется язык TL.

TL (Type Language) - это язык для описания структур данных.

Для структуризации полезных данных, при общении используются [TL схемы](https://github.com/ton-blockchain/ton/tree/master/tl/generate/scheme) и в качестве префикса - айди схемы в little endian, который вычисляется, как crc32 с таблицей IEEEE от схемы.

TL оперирует 32 битными блоками. Соответственно размер данных в TL должен быть кратен 4 байтам. Если размер объекта не кратен 4, нам нужно добавить нужное количество нулевых байт до кратности. 

Для кодирования чисел всегда используется порядок Little Endian.

Более детально TL можно изучить в [документации Telegram](https://core.telegram.org/mtproto/TL) (опционально)

### Ping&Pong
Ping пакет оптимально отправлять примерно раз в 5 секунд. Это нужно для поддержания соединения, пока обмен данными не происходит, иначе сервер оборвет соединение.

Ping пакет, как и все остальные, строится по стандартной схеме, описанной [выше](#структура-пакетов), и в качестве полезных данных несет идентификатор и айди запроса. 

Найдем нужную схему для пинг запроса [тут](https://github.com/ton-blockchain/ton/blob/master/tl/generate/scheme/ton_api.tl#L35) и вычислим айди схемы, как
`crc32_IEEEE("tcp.ping random_id:long = tcp.Pong")`. При конвертации в байты с порядком little endian получим **9a2b084d**.

Таким образом наш ADNL ping пакет будет выглядеть так:
* 4 байта размера пакета в little endian -> 64 + (4+8) = **76**
* 32 байта nonce -> случайные 32 байта
* 4 байта ID TL схемы -> **9a2b084d**
* 8 байт айди запроса -> случайное число uint64 
* 32 байта чексумма SHA256 от nonce и полезных данных

Отправляем наш пакет и в ответ ждем [tcp.pong](https://github.com/ton-blockchain/ton/blob/master/tl/generate/scheme/ton_api.tl#L23), `random_id` будет равен тому, который мы отправили в ping.

### Получение информации от лайт сервера
Все запросы, которые направлены на получение информации из блокчеина, обернуты в [LiteServer Query](https://github.com/ton-blockchain/ton/blob/master/tl/generate/scheme/lite_api.tl#L83) схему, которая в свою очередь обернута в [ADNL Query](https://github.com/ton-blockchain/ton/blob/master/tl/generate/scheme/lite_api.tl#L22) схему.

LiteQuery:
`liteServer.query data:bytes = Object`, айди **df068c79**

ADNLQuery:
`adnl.message.query query_id:int256 query:bytes = adnl.Message`, айди **7af98bb4**

LiteQuery передается внутри ADNLQuery, как `query:bytes`, а конечный запрос передается внутри LiteQuery, как `data:bytes`.

##### Кодирование bytes в TL
Для кодирования массива байт нам нужно сначала определить его размер. Если он меньше чем 254 байта, то используется кодирование с 1 байтом в качестве размера. Если больше, то в качестве первого байта пишется 0xFE, как индикатор большого массива, и уже после него следуют 3 байта размера.

Например, мы кодируем массив `[0xAA, 0xBB]`, его размер 2. Мы используем 1 байт размера и далее пишем сами данные, получаем `[0x02, 0xAA, 0xBB]`, готово, но мы видим, что финальный равен 3 и не кратен 4м байтам, тогда нам нужно добавить 1 байт паддинга, чтобы было 4. Получаем: `[0x02, 0xAA, 0xBB, 0x00]`.

В случае, если нам нужно закодировать массив, размер которого будет равен, например, 396, мы поступаем следующим образом: 396 >= 254, значит мы используем 3 байта для кодирования размера и 1 байт индикатор повышенного размера, получаем: `[0xFE, 0x8C, 0x01, 0x00, байты массива]`, 396+4 = 400, что кратно 4, выравнивать не нужно.

#### Получение полезных данных
Теперь, так как мы уже умеем формировать TL пакеты для Lite API, мы можем запросить информацию о текущем блоке мастерчеина TON. Блок мастерчеина используется во многих дальнейших запросах, как входящий параметр, для индикации состояния (момента), в котором нам нужна информация.

##### getMasterchainInfo
Ищем нужную нам [TL схему](https://github.com/ton-blockchain/ton/blob/master/tl/generate/scheme/lite_api.tl#L60), вычисляем ее айди и строим пакет:

* 4 байта размера пакета в little endian -> 64 + (4+32+(1+4+(1+4+3)+3)) = **116**
* 32 байта nonce -> случайные 32 байта
* 4 байта ID ADNLQuery схемы -> **7af98bb4**
* 32 байта `query_id:int256` -> случайные 32 байта
* * 1 байт размер массива -> **12**
* * 4 байта ID LiteQuery схемы -> **df068c79**
* * * 1 байт размер массива -> **4**
* * * 4 байта ID getMasterchainInfo схемы -> **2ee6b589**
* * * 3 нулевых байта падинга (выравнивание к 8)
* * 3 нулевых байта падинга (выравнивание к 16)
* 32 байта чексумма SHA256 от nonce и полезных данных

Пример пакета в hex:
```
74000000                                                             -> размер пакета (116)
5fb13e11977cb5cff0fbf7f23f674d734cb7c4bf01322c5e6b928c5d8ea09cfd     -> nonce
  7af98bb4                                                           -> ADNLQuery
  77c1545b96fa136b8e01cc08338bec47e8a43215492dda6d4d7e286382bb00c4   -> query_id
    0c                                                               -> размер массива
    df068c79                                                         -> LiteQuery
      04                                                             -> размер массива
      2ee6b589                                                       -> getMasterchainInfo
      000000                                                         -> 3 байта падинг
    000000                                                           -> 3 байта падинг
ac2253594c86bd308ed631d57a63db4ab21279e9382e416128b58ee95897e164     -> sha256
```

В ответ мы ожидаем получить [liteServer.masterchainInfo](https://github.com/ton-blockchain/ton/blob/master/tl/generate/scheme/lite_api.tl#L30), состоящий из last:[ton.blockIdExt](https://github.com/ton-blockchain/ton/blob/master/tl/generate/scheme/tonlib_api.tl#L51) state_root_hash:int256 и init:[tonNode.zeroStateIdExt](https://github.com/ton-blockchain/ton/blob/master/tl/generate/scheme/ton_api.tl#L359).

Полученый пакет десериализуется тем же самым образом, что и отправленый, - тот же алгоритм, но в обратную сторону, разве что ответ завернут только в [ADNLAnswer](https://github.com/ton-blockchain/ton/blob/master/tl/generate/scheme/lite_api.tl#L23).

После расшифровки ответа, получаем пакет вида:
```
20010000                                                                  -> размер пакета (288)
5558b3227092e39782bd4ff9ef74bee875ab2b0661cf17efdfcd4da4e53e78e6          -> nonce
  1684ac0f                                                                -> ADNLAnswer
  77c1545b96fa136b8e01cc08338bec47e8a43215492dda6d4d7e286382bb00c4        -> query_id (идентичен запросу)
    b8                                                                    -> размер массива
    81288385                                                              -> liteServer.masterchainInfo
                                                                          last:tonNode.blockIdExt
        ffffffff                                                          -> workchain:int
        0000000000000080                                                  -> shard:long
        27405801                                                          -> seqno:int   
        e585a47bd5978f6a4fb2b56aa2082ec9deac33aaae19e78241b97522e1fb43d4  -> root_hash:int256
        876851b60521311853f59c002d46b0bd80054af4bce340787a00bd04e0123517  -> file_hash:int256
      8b4d3b38b06bb484015faf9821c3ba1c609a25b74f30e1e585b8c8e820ef0976    -> state_root_hash:int256
                                                                          init:tonNode.zeroStateIdExt 
        ffffffff                                                          -> workchain:int
        17a3a92992aabea785a7a090985a265cd31f323d849da51239737e321fb05569  -> root_hash:int256      
        5e994fcf4d425c0a6ce6a792594b7173205f740a39cd56f537defd28b48a0f6e  -> file_hash:int256
    000000                                                                -> падинг 3 байта
520c46d1ea4daccdf27ae21750ff4982d59a30672b3ce8674195e8a23e270d21          -> sha256
```

##### runSmcMethod
Мы уже умеем получать блок мастерчеина, значит теперь мы можем вызывать любые методы лайт-сервера.
Разберем **runSmcMethod** - это метод, который вызывает функцию из смарт контракта и возвращает результат. Здесь нам потребуется понять некоторые новые типы данных, такие как [TL-B](https://ton.org/docs/#/overviews/TL-B), [Cell](https://ton.org/docs/#/overviews/Cells) и [BoC](#bag-of-cells).

Для выполнения метода смарт-контракта нам нужно отправить запрос по TL схеме:
`liteServer.runSmcMethod mode:# id:tonNode.blockIdExt account:liteServer.accountId method_id:long params:bytes = liteServer.RunMethodResult`

И ждать ответ вида:
`liteServer.runMethodResult mode:# id:tonNode.blockIdExt shardblk:tonNode.blockIdExt shard_proof:mode.0?bytes proof:mode.0?bytes state_proof:mode.1?bytes init_c7:mode.3?bytes lib_extras:mode.4?bytes exit_code:int result:mode.2?bytes = liteServer.RunMethodResult;`

В запросе мы видим такие поля:
1. mode:# - uint32 битовая маска того, что мы хотим видеть в ответе, например, result:mode.2?bytes будет присутствовать в ответе только, если бит с индексом 2 равен единице.
2. id:tonNode.blockIdExt - наш стейт мастер блока, который мы получили в прошлой главе.
3. account:[liteServer.accountId](https://github.com/ton-blockchain/ton/blob/master/tl/generate/scheme/lite_api.tl#L27) - воркчеин и данные адреса смарт контракта.
4. method_id:long - 8 байт, в которых пишется crc16 с таблицей XMODEM от имени вызываемого метода и установленый 17й бит [[Расчет]](https://github.com/xssnick/tonutils-go/blob/88f83bc3554ca78453dd1a42e9e9ea82554e3dd2/ton/runmethod.go#L16)
5. params:bytes - [Stack](https://github.com/ton-blockchain/ton/blob/master/crypto/block/block.tlb#L783) сериализованый в [BoC](#bag-of-cells), содержащий аргументы для вызова метода. [[Пример реализации]](https://github.com/xssnick/tonutils-go/blob/88f83bc3554ca78453dd1a42e9e9ea82554e3dd2/tlb/stack.go)

Например, нам нужен только `result:mode.2?bytes`, тогда наш mode будет равен 0b100, то есть 4. В ответ мы получим:
1. mode:# -> то, что и отправляли, - 4.
2. id:tonNode.blockIdExt -> наш мастер блок, относительно которого был выполнен метод
3. shardblk:tonNode.blockIdExt -> шард блок, в котором находится аккаунт контракта
4. exit_code:int -> 4 байта, которые являются кодом выхода при выполнении метода. Если все успешно, то = 0, если нет - равен коду исключения
5. result:mode.2?bytes -> [Stack](https://github.com/ton-blockchain/ton/blob/master/crypto/block/block.tlb#L783) сериализованый в [BoC](#bag-of-cells), содержащий возвращенные методом значения.

Разберем вызов и получение результата от метода `a2` контракта `EQBL2_3lMiyywU17g-or8N7v9hDmPCpttzBPE2isF2GTzpK4`:

Код метода на FunC:
```
(cell, cell) a2() method_id {
  cell a = begin_cell().store_uint(0xAABBCC8, 32).end_cell();
  cell b = begin_cell().store_uint(0xCCFFCC1, 32).end_cell();
  return (a, b);
}
```

Заполняем наш запрос:
* `mode` = 4, нам нужен только результат -> `04000000`
* `id` = результат выполнения getMasterchainInfo
* `account` = воркчеин 0 (4 байта `00000000`), и int256 [полученный из адреса нашего контракта](#Адреса), то есть 32 байта `4bdbfde5322cb2c14d7b83ea2bf0deeff610e63c2a6db7304f1368ac176193ce`
* `method_id` = [вычисленый](https://github.com/xssnick/tonutils-go/blob/88f83bc3554ca78453dd1a42e9e9ea82554e3dd2/ton/runmethod.go#L16) id от `a2` -> `0a2e010000000000`
* `params:bytes` = Наш метод не принимает входных параметров, значит нам нужно передать ему пустой стек (`000000`, ячейка 3 байта - стек 0 глубины) сериализованый в [BoC](#bag-of-cells) -> `b5ee9c72010101010005000006000000` -> сериализуем в bytes и получаем `10b5ee9c72410101010005000006000000000000` 0x10 - размер, 3 байта в конце - падинг.

В ответе получаем:
* `mode:#` -> не интересен
* `id:tonNode.blockIdExt` -> не интересен
* `shardblk:tonNode.blockIdExt` -> не интересен
* `exit_code:int` -> равен 0, если все успешно
* `result:mode.2?bytes` -> [Stack](https://github.com/ton-blockchain/ton/blob/master/crypto/block/block.tlb#L783) содержащий возвращенные методом данные в формате BoC, его мы распакуем.

Внутри `result` мы получили `b5ee9c7201010501001b000208000002030102020203030400080ccffcc1000000080aabbcc8`, это [BoC](#bag-of-cells) содержащий стек с данными. Когда мы десериализуем его, мы получим ячейку:
```
32[00000203] -> {
  8[03] -> {
    0[],
    32[0AABBCC8]
  },
  32[0CCFFCC1]
}
```
Если мы ее распарсим, то получим 2 значения типа cell, которые возвращает наш FunC метод. 
Первые 3 байта корневой ячейки `000002` - это глубина стека, то есть 2. Значит метод вернул нам 2 значения. 

Продолжаем парсинг, следующие 8 бит (1 байт) - это тип значения на текущем уровне стека. Для некоторых типов он может занимать 2 байта. Возможные варианты можно посмотреть в [схеме](https://github.com/ton-blockchain/ton/blob/master/crypto/block/block.tlb#L766). В нашем случае мы имеем `03`, что означает:
```
vm_stk_cell#03 cell:^Cell = VmStackValue;
```
Значит тип нашего значения - cell, и, судя по схеме, он хранит само значение, как ссылку. Но, если мы посмотрим на схему хранения элементов стека, -
```
vm_stk_cons#_ {n:#} rest:^(VmStackList n) tos:VmStackValue = VmStackList (n + 1);
```
То увидим, что первая ссылка `rest:^(VmStackList n)` - это ячейка следующего значения на стеке, а наше значение `tos:VmStackValue` идет вторым, значит для получения значения нам нужно читать вторую ссылку, то есть `32[0CCFFCC1]` - это наш первый cell, который вернул контракт.

Теперь мы можем залезть глубже и достать второй элемент стека, проходим по первой ссылке, теперь мы имеем:
```
8[03] -> {
    0[],
    32[0AABBCC8]
  }
```
Повторяем тот же процесс. Первые 8 бит = `03` - то есть опять cell. Вторая ссылка - это значение `32[0AABBCC8]` и, так как глубина нашего стека равна 2, мы завершаем проход. Итого, мы имеем 2 значения, возвращенные контрактом, - `32[0CCFFCC1]` и `32[0AABBCC8]`. 

Обратите внимание, что они идут в обратном порядке. Точно так же нужно передавать и аргументы при вызове функции - в обратном порядке от того, что мы видим в коде FunC. 

[Пример реализации](https://github.com/xssnick/tonutils-go/blob/master/ton/runmethod.go#L24)

### Адреса

Стандартный адрес вида `EQBL2_3lMiyywU17g-or8N7v9hDmPCpttzBPE2isF2GTzpK4` является байтами в кодировке base64 uri. 
Его длина составляет 36 байт, последние 2 из которых - чексумма crc16 с таблицей XMODEM. Первый байт - флаги, второй - воркчеин. 
32 байта посередине - данные самого адреса, в схемах часто представлены как int256.

[Пример декодирования](https://github.com/xssnick/tonutils-go/blob/3d9ee052689376061bf7e4a22037ff131183afad/address/addr.go#L156)

### Bag of Cells
Bag of Cells - формат сериализации ячеек (cells) в массив байт, описанный в виде [TL-B схемы](https://github.com/ton-blockchain/ton/blob/24dc184a2ea67f9c47042b4104bbb4d82289fac1/crypto/tl/boc.tlb#L25).

#### Сериализация ячеек
Разберем ячейку вида:
```
1[8_] -> {
  24[0AAAAA],
  7[FE] -> {
    24[0AAAAA]
  }
}
```
Тут мы имеем корневую ячейку размером 1 бит, которая имеет 2 ссылки: первая на ячейку размером 24 бита и вторая на ячейку размером 7 бит, которая имеет 1 ссылку на ячейку размером 24 бита.

Нам нужно превратить ячейки в плоский набор байтов, для этого нам сначала нам нужно оставить только уникальные ячейки - их у нас 3 из 4х. Имеем:
```
1[8_]
24[0AAAAA]
7[FE]
```
Теперь расположим их в таком порядке, чтобы родительские ячейки не указывали назад. Та ячейка, на которую указывают остальные, должна быть в списке после тех, которые на нее указывают. Получаем:
```
1[8_]      -> индекс 0 (корневая ячейка)
7[FE]      -> индекс 1
24[0AAAAA] -> индекс 2
```

Посчитаем дескрипторы для каждой из них. Это 2 байта, хранящие флаги, информацию о длине данных и количестве ссылок. Флаги будут опущены в текущем разборе, они почти всегда 0. Первый байт содержит 5 битов флагов и 3 бита количества ссылок. Второй байт - длину полных 4х битных групп (но минимум 1, если не пусто). Получаем:
```
1[8_]      -> 0201 -> 2 ссылки, длина 1 
7[FE]      -> 0101 -> 1 ссылка, длина 1
24[0AAAAA] -> 0006 -> 0 ссылок, длина 6
```
Для данных с неполными 4х битными группами добавляется 1 бит в конец. Он означает конечный бит группы и служит для определения настоящего размера неполных групп. Добавим его:
```
1[8_]      -> C0     -> 0b10000000->0b11000000
7[FE]      -> FF     -> 0b11111110->0b11111111
24[0AAAAA] -> 0AAAAA -> не меняем (полные группы)
```

Теперь добавим индексы ссылок:
```
0 1[8_]      -> 0201 -> ссылается на 2 ячейки с такими индексами
1 7[FE]      -> 02 -> ссылается на ячейки с индексом 2
2 24[0AAAAA] -> нет ссылок
```

И соберем все вместе:
```
0201 C0     0201  
0101 AA     02
0006 0AAAAA 
```

И склеим в массив байт:
`0201c002010101ff0200060aaaaa`, размер 14 байт.

[Пример сериализации](https://github.com/xssnick/tonutils-go/blob/3d9ee052689376061bf7e4a22037ff131183afad/tvm/cell/serialize.go#L205)

#### Упаковка в BoC
Упакуем ячейку из прошлой главы. Мы уже сериализовали ее в плоский массив байт длиной 14.

Строим заголовок согласно [схеме](https://github.com/ton-blockchain/ton/blob/24dc184a2ea67f9c47042b4104bbb4d82289fac1/crypto/tl/boc.tlb#L25).

```
b5ee9c72                      -> id tl-b структуры BoC
01                            -> флаги и размер size:(## 3), в нашем случае флаги все 0, 
                                 и количество байтов нужное для хранения количества ячеек - 1.
                                 получаем - 0b0_0_0_00_001
01                            -> кол-во байтов для хранения размера сериализованых ячеек
03                            -> количество ячеек, 1 байт (определяется 3 битами size:(## 3), равный 3м.
01                            -> количество корневых ячеек - 1
00                            -> absent, всегда 0 (в текущих имплементациях)
0e                            -> размер сериализованых ячеек, 1 байт (размер определен выше), равный 14
00                            -> индекс корневой ячейки, размером 1 (определяется 3 битами size:(## 3) из заголовка), 
                                 всегда 0
0201c002010101ff0200060aaaaa  -> сериализованые ячейки
```

Всё, что выше, склеим в массив байтов и получим:
`b5ee9c7201010301000e000201c002010101ff0200060aaaaa` Это наш финальный BoC!

Примеры реализации BoC: [Сериализация](https://github.com/xssnick/tonutils-go/blob/master/tvm/cell/serialize.go), [Десериализация](https://github.com/xssnick/tonutils-go/blob/master/tvm/cell/parse.go)

## Дополнительные технические детали хендшейка

#### Получение айди ключа
Айди ключа - это SHA256 хэш от 4 магических байт **[0xC6, 0xB4, 0x13, 0x48]** и **публичного ключа**

[Пример кода](https://github.com/xssnick/tonutils-go/blob/2b5e5a0e6ceaf3f28309b0833cb45de81c580acc/liteclient/crypto.go#L16)

#### Шифрование данных Handshake пакета
Хэндшейк пакет отправляется в полуоткрытом виде, зашифрованы только 160 байт, содержащие информацию о постоянных шифрах.

Чтобы их зашифровать, нам нужен AES-CTR шифр, для его получения нам нужен SHA256 хэш от 160 байт и [общий ключ ECDH](#%D0%BF%D0%BE%D0%BB%D1%83%D1%87%D0%B5%D0%BD%D0%B8%D0%B5-%D0%BE%D0%B1%D1%89%D0%B5%D0%B3%D0%BE-%D0%BA%D0%BB%D1%8E%D1%87%D0%B0-%D0%BF%D0%BE-ecdh)

Шифр собирается следующим образом:
* ключ = (0 - 15 байты общего ключа) + (16 - 31 байты хэша)
* iv   = (0 - 3 байты хэша) + (20 - 31 байты общего ключа)

После того, как шифр собран, мы шифруем им наши 160 байт.

[Пример кода](https://github.com/xssnick/tonutils-go/blob/2b5e5a0e6ceaf3f28309b0833cb45de81c580acc/liteclient/connection.go#L361)

#### Получение общего ключа по ECDH
Для расчета общего ключа нам понадобится наш приватный ключ и публичный ключ сервера. 

Суть DH в получении общего секретного ключа, без разглашения приватной информации. Приведу пример, как это происходит, в максимально упрощенном виде.
Предположим, что нужно сгенерировать общий ключ между нами и сервером, процесс будет выглядеть так:
1. Мы генерируем секретное и публичное числа, например **6** и **7**
2. Сервер генерирует секретное и публичное числа, например **5** и **15**
3. Мы с сервером обмениваемся публичными числами, отправляем серверу **7**, он нам отправляет **15**.
4. Мы высчитываем: **7^6 mod 15 = 4**
5. Сервер высчитывает: **7^5 mod 15 = 7**
6. Обмениваемся полученными числами, мы серверу **4**, он нам **7**
7. Мы высчитываем **7^6 mod 15 = 4**
8. Сервер высчитывает: **4^5 mod 15 = 4**
9. Общий ключ = **4**

Детали самого ECDH будут опущены, чтобы не усложнять прочтение. Он вычисляется с помощью 2 ключей, приватного и публичного, путем нахождения общей точки на кривой. Если интересно, то лучше почитать про это отдельно.

[Пример кода](https://github.com/xssnick/tonutils-go/blob/2b5e5a0e6ceaf3f28309b0833cb45de81c580acc/liteclient/crypto.go#L32)
