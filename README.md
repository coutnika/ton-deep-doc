# TON Deep Documentation
Документация по внутренним компонентам ТОНа, которая поможет разобраться, как все работает.

## ADNL
Это протокол нижнего уровня, на котором построено все взаимодействие в сети TON, он может работать поверх любого протокола, но чаще всего применяется поверх TCP и UDP. UDP применяется для общения между нодами, а TCP для коммуникации с lite-клиентами. В данный момент в документации делается упор только на TCP реализацию.

Вместо адресов, узлы сети используют публичные ключи ed25519 и устанавливают соединение, используя общий ключ, полученный с помощью процедуры Диффи-Хелмана для эллиптических кривых - ECDH.

### Структура пакетов
Каждый ADNL пакет, кроме хэндшейка, имеет структуру:
* 4 байта размера пакета (N)
* 32 байта nonce [[?]](## "Случайные байты, нужны для защиты от атак на чексумму")
* (N - 64) байт полезных данных
* 32 байта чексумма [[?]](## "SHA256 от nonce и полезных данных")

Весь пакет, включая размер, зашифрован **AES-CTR**.
После расшифровки - нужно обязательно проверить, сходится ли чексумма с данными, для проверки нужно просто посчитать чексумму самостоятельно и сравнить результат с тем, что у нас в пакете.

Хэндшейк пакет - исключение, он передается в частично открытом виде и описан в следующей главе.


### Установка соединения
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
* **Айди ключа сервера** [[Подробнее]](#%D0%BF%D0%BE%D0%BB%D1%83%D1%87%D0%B5%D0%BD%D0%B8%D0%B5-%D0%B0%D0%B9%D0%B4%D0%B8-%D0%BA%D0%BB%D1%8E%D1%87%D0%B0)
* **Свой публичный ключ ed25519** [[?]](## "Можно генерировать новую пару на каждое соединение")
* **SHA256 хэш от наших 160 байт**
* **Наши 160 байт в зашифрованом виде** [[Подробнее]](#%D1%88%D0%B8%D1%84%D1%80%D0%BE%D0%B2%D0%B0%D0%BD%D0%B8%D0%B5-%D0%B4%D0%B0%D0%BD%D0%BD%D1%8B%D1%85-handshake-%D0%BF%D0%B0%D0%BA%D0%B5%D1%82%D0%B0)


При получении хэндшейк пакета, сервер проделает те же самые действия у себя, получит ECDH ключ, расшифрует 160 байт и создаст 2 постоянных ключа. Если все получится, сервер ответит пустым ADNL пакетом, без полезных данных, для дешифровки которого (а также последующих) нужно использовать один из постоянных ключей. 

С этого момента можно считать соединение установленным.

## Технические детали реализации клиента

#### Получение айди ключа
Айди ключа - это SHA256 хэш от 4 магических байт [0xC6, 0xB4, 0x13, 0x48] и публичного ключа

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

Детали самого ECDH будут опущены, чтобы не усложнять прочтение. Если интересно, то лучше почитать про это отдельно.

[Пример кода](https://github.com/xssnick/tonutils-go/blob/2b5e5a0e6ceaf3f28309b0833cb45de81c580acc/liteclient/crypto.go#L32)
