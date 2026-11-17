<!--
Бланк звіту до лабораторної роботи «TLS та сертифікати».
Скопіювати цей файл у власний публічний репозиторій звіту як README.md і заповнити.
Вивід команд вставляти текстом у блоки коду, а не знімками екрана.
Порожні поля не допускаються: якщо щось не вдалося, записати, що саме і чому.
-->

# Лабораторна робота 2. TLS та сертифікати

| Поле | Значення |
|---|---|
| ПІБ |Трацевський Станіслав Олегович |
| Група |2.01. |
| Ідентифікатор | cs-201-24 |
| Дата виконання |17.11.2026 |
| Середовище | GitHub Codespaces |
| Версія OpenSSL | <!-- вивід openssl version у терміналі стенда --> |

## Параметри варіанта

<!-- Переписати з картки stand/cards/<id>.md. -->

| Параметр | Значення |
|---|---|
| Віддалений сервер |`cdn.cs-201-24.lab` |
| Імена для сертифіката |`www.cs-201-24.lab`, `api.cs-201-24.lab` |
| Ваш сервер (IP) |`10.53.1.1` |

<!-- Якщо стенд перезапускався під час роботи, вказати, після якого завдання. -->

## Завдання 1. HTTP і HTTPS у дампі трафіку

Вивід кроку 1.1:

```text
root@cs-201-24:/lab/work# cd /lab/work
timeout 8 tcpdump -ni eth0 -A -s0 -l 'host 10.53.0.80 and tcp port 80' > t1-http.txt &
sleep 1; curl -s http://cdn.cs-201-24.lab/; wait
grep -a -E 'GET /|Host:|server: web' t1-http.txt
[1] 44
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
server: web
Host: cdn.cs-201-24.lab
SNI: 
protocol: HTTP/1.1 
client: 10.53.1.1
10 packets captured
10 packets received by filter
0 packets dropped by kernel
[1]+  Exit 124                timeout 8 tcpdump -ni eth0 -A -s0 -l 'host 10.53.0.80 and tcp port 80' > t1-http.txt
16:34:26.266345 IP 10.53.1.1.58018 > 10.53.0.80.80: Flags [P.], seq 1:81, ack 1, win 502, options [nop,nop,TS val 1854538724 ecr 449122767], length 80: HTTP: GET / HTTP/1.1
n.......GET / HTTP/1.1
Host: cdn.cs-201-24.lab
server: web
Host: cdn.cs-201-24.lab
```

Вивід кроку 1.2:

```text
root@cs-201-24:/lab/work# timeout 8 tcpdump -ni eth0 -A -s0 -l 'host 10.53.0.80 and tcp port 443' > t1-https.txt &
sleep 1; curl -s --cacert $STAND_CA https://cdn.cs-201-24.lab/; wait
grep -a -c -E 'GET /|server: web' t1-https.txt
grep -a -o 'cdn.cs-201-24.lab' t1-https.txt | wc -l
[1] 57
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
server: web
Host: cdn.cs-201-24.lab
SNI: cdn.cs-201-24.lab
protocol: HTTP/2.0 TLSv1.3
client: 10.53.1.1
23 packets captured
23 packets received by filter
0 packets dropped by kernel
[1]+  Exit 124                timeout 8 tcpdump -ni eth0 -A -s0 -l 'host 10.53.0.80 and tcp port 443' > t1-https.txt
0
1
```

| Що шукали в дампі | HTTP | HTTPS |
|---|---|---|
| рядок запиту `GET /` |так |ні |
| заголовок `Host` |так |ні |
| тіло відповіді (`server: web`) |так |так |
| ім'я віддаленого сервера (кількість входжень) |2 |1 |

1. У якому повідомленні і в якому полі передається ім'я сервера, яке видно в дампі HTTPS?
Воно передається в повідомленні Client Hello у спеціальному розширенні SNI.
2. Чому рядок `SNI:` у відповіді сервера порожній для HTTP і заповнений для HTTPS?
Тому що HTTP працює на 80-му порті без шифрування і TLS-з'єднання не створюється взагалі, відповідно поле SNI там не використовується. Для HTTPS створюється захищене з'єднання, і клієнт обов'язково передає ім'я сервера через SNI, щоб сервер знав, який саме сертифікат показати.
3. Рядок `protocol:` показує HTTP/1.1 для запиту по HTTP і HTTP/2.0 для запиту по HTTPS, хоча обидва запити виконано однією командою `curl` без додаткових ключів. Що визначило версію HTTP у другому випадку?
Версію визначив механізм ALPN під час рукостискання TLS, який узгодив перехід на сучасний протокол HTTP/2 поверх безпечного з'єднання.
## Завдання 2. Рукостискання по записах

Вивід кроку 2.1 (TLS 1.3):

```text
root@cs-201-24:/lab/work# openssl s_client -connect cdn.cs-201-24.lab:443 -servername cdn.cs-201-24.lab -alpn h2,http/1.1 \
-CAfile $STAND_CA -trace </dev/null > t2-13.txt 2>&1
grep -a -E '^(Sent|Received) Record|Content Type|^      [A-Z][A-Za-z]+, Length' t2-13.txt
grep -a -E '^(ALPN|New,)|\bVerify return' t2-13.txt
Sent Record
  Content Type = Handshake (22)
Received Record
  Content Type = Handshake (22)
Received Record
  Content Type = ChangeCipherSpec (20)
Received Record
  Content Type = ApplicationData (23)
  Inner Content Type = Handshake (22)
Received Record
  Content Type = ApplicationData (23)
  Inner Content Type = Handshake (22)
Received Record
  Content Type = ApplicationData (23)
  Inner Content Type = Handshake (22)
Received Record
  Content Type = ApplicationData (23)
  Inner Content Type = Handshake (22)
Sent Record
  Content Type = ChangeCipherSpec (20)
Sent Record
  Content Type = ApplicationData (23)
  Inner Content Type = Handshake (22)
Sent Record
  Content Type = ApplicationData (23)
  Inner Content Type = Alert (21)
New, TLSv1.3, Cipher is TLS_AES_256_GCM_SHA384
ALPN protocol: h2
Verify return code: 0 (ok)
```

Вивід кроку 2.2 (TLS 1.2):

```text
root@cs-201-24:/lab/work# openssl s_client -connect cdn.cs-201-24.lab:443 -servername cdn.cs-201-24.lab -alpn h2,http/1.1 \
-CAfile $STAND_CA -tls1_2 -trace </dev/null > t2-12.txt 2>&1
grep -a -E '^(Sent|Received) Record|Content Type|^      [A-Z][A-Za-z]+, Length' t2-12.txt
grep -a -E '^(ALPN|New,)|\bVerify return' t2-12.txt
Sent Record
  Content Type = Handshake (22)
Received Record
  Content Type = Handshake (22)
Received Record
  Content Type = Handshake (22)
Received Record
  Content Type = Handshake (22)
Received Record
  Content Type = Handshake (22)
Sent Record
  Content Type = Handshake (22)
Sent Record
  Content Type = ChangeCipherSpec (20)
Sent Record
  Content Type = Handshake (22)
Received Record
  Content Type = Handshake (22)
Received Record
  Content Type = ChangeCipherSpec (20)
Received Record
  Content Type = Handshake (22)
Received Record
  Content Type = ApplicationData (23)
Sent Record
  Content Type = Alert (21)
New, TLSv1.2, Cipher is ECDHE-ECDSA-AES256-GCM-SHA384
ALPN protocol: h2
    Verify return code: 0 (ok)
```

<!-- Один рядок на кожен запис з виводу 2.1, у тому самому порядку. Напрям: К→С або С→К. -->

| № | Напрям | `Content Type` | `Inner Content Type` | Повідомлення | `Length` |
|---|---|---|---|---|---|
| 1 |К→С |Handshake (22) |- |ClientHello |517 |
| 2 |С→К |Handshake (22) |- |ServerHello, EncryptedExtensions, Certificate, CertificateVerify, Finished |1241 |
| 3 |С→К |ChangeCipherSpec (20) |- |ChangeCipherSpec |1 |
| 4 |С→К |ApplicationData (23) |Handshake (22) |NewSessionTicket |134 |
| 5 |С→К |ApplicationData (23) |Handshake (22) |NewSessionTicket |134 |
| 6 |С→К |ApplicationData (23) |Handshake (22) |NewSessionTicket |134 |
| 7 |С→К |ApplicationData (23) |Handshake (22) NewSessionTicket| |134 |
| 8 |К→С |ChangeCipherSpec (20) |- |ChangeCipherSpec |1 |
| 9 |К→С |ApplicationData (23) |Handshake (22) |Finished |52 |
| 10 |К→С |ApplicationData (23) |Alert (21) |CloseNotify |25 |

Вивід кроків 2.3 і 2.4:ІІ

```text
root@cs-201-24:/lab/work# grep -a -A3 'extension_type=server_name' t2-13.txt | head -4
        extension_type=server_name(0), length=22
          0000 - 00 14 00 00 11 63 64 6e-2e 63 73 2d 32 30 31   .....cdn.cs-201
          000f - 2d 32 34 2e 6c 61 62                           -24.lab
        extension_type=ec_point_formats(11), length=4
root@cs-201-24:/lab/work# for v in 1.3 1.2; do
timeout 6 tcpdump -ni eth0 -A -s0 -l 'host 10.53.0.80 and tcp port 443' > t2-dump-$v.txt &
sleep 1; curl -s -o /dev/null --tlsv1.2 --tls-max $v --cacert $STAND_CA https://cdn.cs-201-24.lab/; wait
echo "TLS $v: $(grep -a -c 'OPISM Stand Intermediate' t2-dump-$v.txt)"
done
[1] 73
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
22 packets captured
22 packets received by filter
0 packets dropped by kernel
[1]+  Exit 124                timeout 6 tcpdump -ni eth0 -A -s0 -l 'host 10.53.0.80 and tcp port 443' > t2-dump-$v.txt
TLS 1.3: 0
[1] 79
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
21 packets captured
21 packets received by filter
0 packets dropped by kernel
[1]+  Exit 124                timeout 6 tcpdump -ni eth0 -A -s0 -l 'host 10.53.0.80 and tcp port 443' > t2-dump-$v.txt
TLS 1.2: 2
```

| Показник | TLS 1.3 | TLS 1.2 |
|---|---|---|
| Довжина ClientHello, байт |517 |517 |
| Номер першого запису, тип якого спостерігач не бачить |4 |7 |
| Кількість RTT від ClientHello до першого HTTP-запиту |1 |2 |
| Чи видно сертифікат у дампі (результат 2.4) |ні |так |
| Обраний протокол ALPN |h2 |h2 |

1. За якими рядками виводу 2.1 визначено, з якого запису починається шифрування? Що бачить спостерігач замість типу кожного наступного запису?
Шифрування починається там, де з'являється тип вмісту ApplicationData (23). Сторонній спостерігач бачить загальний тип ApplicationData, а реальний тип справжнього повідомлення прихований у полі Inner Content Type.
2. Чому ClientHello у TLS 1.3 довший, ніж у TLS 1.2? Назвіть розширення, яке за рахунок цього скорочує рукостискання на один RTT.
ClientHello у TLS 1.3 містить попередні налаштування. Завдяки розширенню key_share клієнт одразу надсилає свої криптографічні параметри, що дозволяє зекономити один повний цикл обміну.
3. Довжина розширення `server_name` у виводі 2.3 становить 17 байт для імені з 12 символів. З яких частин складається це значення і яким воно буде для вашого імені?
Воно складається з 2 байтів типу розширення, 2 байтів загальної довжини, 1 байта типу імені, 2 байтів довжини самого імені та 12 байтів доменного імені cdn.cs-201-24.lab.
4. Навіщо в TLS 1.3 передається запис ChangeCipherSpec, якщо в рукостисканні він нічого не змінює?
Він передається виключно для забезпечення зворотної сумісності, щоб старі мережеві екрани або посередники, які очікують на цей запис зі старої версії TLS, не заблокували з'єднання.
## Завдання 3. Сертифікат для власного сервера

Вивід кроків 3.1 і 3.2:

```text

```

Вивід кроку 3.3 (останні рядки `lego` і рядки журналу nginx):

```text

```

Вивід кроку 3.4:

```text

```

Вивід кроку 3.5:

```text

```

Вивід кроку 3.6:

```text

```

| Параметр | Значення |
|---|---|
| Видавець сертифіката сервера | |
| Видавець сертифіката видавця | |
| Імена в SAN | |
| Строк дії (від – до) | |
| Тип ключа сервера і алгоритм підпису | |
| Кількість сертифікатів, які надсилає сервер, у кроці 3.5 і в кроці 3.6 | |

1. Хто і як перевірив, що ви контролюєте обидва імена? Вкажіть рядки виводу, які це підтверджують.

2. Чому в кроці 3.5 перевірка не пройшла, хоча корінь центру сертифікації вказано в `--cacert`, а сертифікат виданий правильно?

3. У виводі ланцюжка для сертифіката сервера вказано `id-ecPublicKey` і `RSA-SHA256`. Що означає кожне з цих значень і чому вони не суперечать одне одному?

4. Що станеться з перевіркою вашого сервера після перезапуску стенда і чому? Що в реальній інфраструктурі відповідає цій ситуації?

## Завдання 4. Діагностика несправностей

Вивід кроку 4.1:

```text

```

| Сервер | Помилка `curl` | `Verify return code` | Яка перевірка не пройшла | Рядок виводу, що це доводить | Що має змінити власник сервера |
|---|---|---|---|---|---|
| f1 | | | | | |
| f2 | | | | | |
| f3 | | | | | |
| f4 | | | | | |
| f5 | | | | | |

1. Для сервера з помилкою `unable to get local issuer certificate` визначте, яка це несправність: сервер не надсилає проміжний сертифікат чи сертифікат виданий центром, якому клієнт не довіряє. Чим ці дві несправності відрізняються у виводі `openssl s_client`?

2. Якщо серед ваших серверів є сервер з невідповідним ім'ям, чому `openssl s_client` повертає для нього `Verify return code: 0 (ok)`, а `curl` повідомляє про помилку? Якщо такого сервера немає, дайте відповідь для загального випадку.

3. Які з п'яти серверів відповіли б на запит `curl -k`? Що при цьому втрачається порівняно з `--cacert`?

## Завдання 5. Ціна нового з'єднання

**Прогноз** (записати до вимірювання). Скільки RTT мине до завершення кожного етапу:

| Етап | TLS 1.3, нове з'єднання | TLS 1.2, нове з'єднання | Повторне використання з'єднання |
|---|---|---|---|
| `time_connect` | | | |
| `time_appconnect` | | | |
| `time_starttransfer` | | | |

Вивід кроків 5.2 і 5.3:

```text

```

<!-- Значення в мілісекундах. Для нових з'єднань — медіана трьох вимірювань. -->

| Запит | `namelookup` | `connect` | `appconnect` | `starttransfer` | `total` | `num_connects` |
|---|---|---|---|---|---|---|
| TLS 1.3, нове | | | | | | |
| TLS 1.2, нове | | | | | | |
| 5.3, перший запит | | | | | | |
| 5.3, другий запит | | | | | | |
| 5.3, третій запит | | | | | | |

RTT, визначений вимірюванням, і з якого значення його визначено:

1. Виразіть кожне виміряне значення через RTT. Де вимірювання відрізняється від цілого числа RTT і чим це пояснюється?

2. Чому `time_namelookup` не близький до нуля, хоча резолвер уже має ім'я в кеші? За яких умов він був би приблизно вдвічі більшим?

3. Сервіс виконує 200 послідовних запитів до цього сервера. Розрахуйте, скільки часу займе лише очікування мережі, якщо кожен запит встановлює нове з'єднання TLS 1.3, нове з'єднання TLS 1.2 і якщо всі запити йдуть через одне з'єднання.

## Висновки

**Що виявилося несподіваним.** Результат, який не збігся з очікуванням, і рядок виводу, де це видно:

**Яке питання залишилося без відповіді.** Що саме не вдалося пояснити; до якого рядка виводу це стосується; що вже перевірено і що з'ясувати не вдалося:

## Декларація використання ШІ

Використовувалися інструменти ШІ: <!-- так / ні -->

<!-- Якщо так, заповнити таблицю: один рядок на кожне звернення, що вплинуло на звіт. Промпт наводити дослівно. -->

| № | Інструмент | Промпт (дослівно) | Що використано у звіті і що змінено самостійно |
|---|---|---|---|
| 1 | | | |
