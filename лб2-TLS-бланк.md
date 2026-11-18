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
| Версія OpenSSL | OpenSSL 3.0.13 30 Jan 2024 (Library: OpenSSL 3.0.13 30 Jan 2024) |

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
| тіло відповіді (`server: web`) |так |ні |
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
Sent Record
  Content Type = Handshake (22)
  Length = 332
    ClientHello, Length=328
Received Record
  Content Type = Handshake (22)
  Length = 122
    ServerHello, Length=118
Received Record
  Content Type = ChangeCipherSpec (20)
  Length = 1
Received Record
  Content Type = ApplicationData (23)
  Length = 36
  Inner Content Type = Handshake (22)
    EncryptedExtensions, Length=15
Received Record
  Content Type = ApplicationData (23)
  Length = 1004
  Inner Content Type = Handshake (22)
    Certificate, Length=983
Received Record
  Content Type = ApplicationData (23)
  Length = 95
  Inner Content Type = Handshake (22)
    CertificateVerify, Length=74
Received Record
  Content Type = ApplicationData (23)
  Length = 69
  Inner Content Type = Handshake (22)
    Finished, Length=48
Sent Record
  Content Type = ChangeCipherSpec (20)
  Length = 1
Sent Record
  Content Type = ApplicationData (23)
  Length = 69
  Inner Content Type = Handshake (22)
    Finished, Length=48
Sent Record
  Content Type = ApplicationData (23)
  Length = 19
  Inner Content Type = Alert (21)
ALPN protocol: h2
Verify return code: 0 (ok)
```

Вивід кроку 2.2 (TLS 1.2):

```text
Sent Record
  Content Type = Handshake (22)
  Length = 227
    ClientHello, Length=223
Received Record
  Content Type = Handshake (22)
  Length = 78
    ServerHello, Length=74
Received Record
  Content Type = Handshake (22)
  Length = 982
    Certificate, Length=978
Received Record
  Content Type = Handshake (22)
  Length = 115
    ServerKeyExchange, Length=111
Received Record
  Content Type = Handshake (22)
  Length = 4
    ServerHelloDone, Length=0
Sent Record
  Content Type = Handshake (22)
  Length = 37
    ClientKeyExchange, Length=33
Sent Record
  Content Type = ChangeCipherSpec (20)
  Length = 1
Sent Record
  Content Type = Handshake (22)
  Length = 40
    Finished, Length=12
Received Record
  Content Type = Handshake (22)
  Length = 218
    NewSessionTicket, Length=214
Received Record
  Content Type = ChangeCipherSpec (20)
  Length = 1
Received Record
  Content Type = Handshake (22)
  Length = 40
    Finished, Length=12
Received Record
  Content Type = ApplicationData (23)
  Length = 64
Sent Record
  Content Type = Alert (21)
  Length = 26
ALPN protocol: h2
    Verify return code: 0 (ok)
```

<!-- Один рядок на кожен запис з виводу 2.1, у тому самому порядку. Напрям: К→С або С→К. -->

| № | Напрям | `Content Type` | `Inner Content Type` | Повідомлення | `Length` |
|---|---|---|---|---|---|
| 1 |К→С |Handshake (22) |- |ClientHello |332 |
| 2 |С→К |Handshake (22) |- |ServerHello|122 |
| 3 |С→К |ChangeCipherSpec (20) |- |ChangeCipherSpec |1 |
| 4 |С→К |ApplicationData (23) |Handshake (22) |EncryptedExtensions |36|
| 5 |С→К |ApplicationData (23) |Handshake (22) |Certificate |1004 |
| 6 |С→К |ApplicationData (23) |Handshake (22) |CertificateVerify |95 |
| 7 |С→К |ApplicationData (23) |Handshake (22) |Finished |69 |
| 8 |К→С |ChangeCipherSpec (20) |- |ChangeCipherSpec |1 |
| 9 |К→С |ApplicationData (23) |Handshake (22) |Finished |69 |
| 10 |К→С |ApplicationData (23) |Alert (21) |CloseNotify |19|

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
| Довжина ClientHello, байт |332 |227|
| Номер першого запису, тип якого спостерігач не бачить |4 |11 |
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
root@cs-201-24:/lab/work# acme-root
збережено /lab/tls/acme-root.crt
root@cs-201-24:/lab/work# openssl x509 -in /lab/tls/acme-root.crt -noout -subject -dates
subject=CN = Pebble Root CA 557e75
notBefore=Nov 17 17:18:31 2026 GMT
notAfter=Nov 17 17:18:31 2056 GMT
root@cs-201-24:/lab/work# curl -sS --cacert /lab/tls/acme-root.crt https://www.cs-201-24.lab/
curl: (60) SSL certificate problem: self-signed certificate
More details here: https://curl.se/docs/sslcerts.html

curl failed to verify the legitimacy of the server and therefore could not
establish a secure connection to it. To learn more about this situation and
how to fix it, please visit the web page mentioned above.
```

Вивід кроку 3.3 (останні рядки `lego` і рядки журналу nginx):

```text
2026/11/17 17:32:55 [INFO] [www.cs-201-24.lab, api.cs-201-24.lab] acme: Obtaining SAN certificate
2026/11/17 17:32:55 [INFO] [www.cs-201-24.lab] AuthURL: https://ca.lab:14000/authZ/VJL8wetqZXSyynqrJ5aMY4M7vmjUQbzpnKT3qYI_k4s
2026/11/17 17:32:55 [INFO] [api.cs-201-24.lab] AuthURL: https://ca.lab:14000/authZ/qxC6MF231YkarHeCD5qZ_uG7fJxhCiXuCRuPXlnsQow
2026/11/17 17:32:55 [INFO] [api.cs-201-24.lab] acme: Could not find solver for: tls-alpn-01
2026/11/17 17:32:55 [INFO] [api.cs-201-24.lab] acme: use http-01 solver
2026/11/17 17:32:55 [INFO] [www.cs-201-24.lab] acme: Could not find solver for: tls-alpn-01
2026/11/17 17:32:55 [INFO] [www.cs-201-24.lab] acme: use http-01 solver
2026/11/17 17:32:55 [INFO] [api.cs-201-24.lab] acme: Trying to solve HTTP-01
2026/11/17 17:33:01 [INFO] [api.cs-201-24.lab] The server validated our request
2026/11/17 17:33:01 [INFO] [www.cs-201-24.lab] acme: Trying to solve HTTP-01
2026/11/17 17:33:08 [INFO] [www.cs-201-24.lab] The server validated our request
2026/11/17 17:33:08 [INFO] [www.cs-201-24.lab, api.cs-201-24.lab] acme: Validations succeeded; requesting certificates
2026/11/17 17:33:08 [INFO] Wait for certificate [timeout: 30s, interval: 500ms]
2026/11/17 17:33:08 [INFO] [www.cs-201-24.lab] Server responded with a certificate.
10.53.0.10 - - [17/Nov/2026:17:30:06 +0000] "GET /.well-known/acme-challenge/9rfcgUQKM4LnA4Tymeof6c3rci3XwtVJRVd9BW-EDfQ HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
10.53.0.10 - - [17/Nov/2026:17:30:06 +0000] "GET /.well-known/acme-challenge/9rfcgUQKM4LnA4Tymeof6c3rci3XwtVJRVd9BW-EDfQ HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
10.53.0.10 - - [17/Nov/2026:17:30:06 +0000] "GET /.well-known/acme-challenge/9rfcgUQKM4LnA4Tymeof6c3rci3XwtVJRVd9BW-EDfQ HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
10.53.0.10 - - [17/Nov/2026:17:32:55 +0000] "GET /.well-known/acme-challenge/Ng7KSJTb-2UzozG8Eb63-3kCiw_fnWGFuyMQjSpopp4 HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
10.53.0.10 - - [17/Nov/2026:17:32:55 +0000] "GET /.well-known/acme-challenge/Ng7KSJTb-2UzozG8Eb63-3kCiw_fnWGFuyMQjSpopp4 HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
10.53.0.10 - - [17/Nov/2026:17:32:55 +0000] "GET /.well-known/acme-challenge/Ng7KSJTb-2UzozG8Eb63-3kCiw_fnWGFuyMQjSpopp4 HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
10.53.0.10 - - [17/Nov/2026:17:33:01 +0000] "GET /.well-known/acme-challenge/ig-kx88-p8KGhVDy4hDwSEq6lNW5ibgG4BIoJVVghSk HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
10.53.0.10 - - [17/Nov/2026:17:33:01 +0000] "GET /.well-known/acme-challenge/ig-kx88-p8KGhVDy4hDwSEq6lNW5ibgG4BIoJVVghSk HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
10.53.0.10 - - [17/Nov/2026:17:33:01 +0000] "GET /.well-known/acme-challenge/ig-kx88-p8KGhVDy4hDwSEq6lNW5ibgG4BIoJVVghSk HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
```

Вивід кроку 3.4:

```text
root@cs-201-24:/lab/work#ccd /lab/work/lego/certificates
openssl x509 -in www.cs-201-24.lab.crt -noout -subject -issuer -dates -serial -ext subjectAltName
openssl x509 -in www.cs-201-24.lab.issuer.crt -noout -subject -issuer
subject=CN = www.cs-201-24.lab
issuer=CN = Pebble Intermediate CA 0242ac
notBefore=Nov 17 17:33:08 2026 GMT
notAfter=Nov 17 17:33:07 2031 GMT
serial=6C59BF1FA7D5C89E
X509v3 Subject Alternative Name: 
    DNS:www.cs-201-24.lab, DNS:api.cs-201-24.lab
subject=CN = Pebble Intermediate CA 0242ac
issuer=CN = Pebble Root CA 557e75
```

Вивід кроку 3.5:

```text
root@cs-201-24:/lab/work/lego/certificates# cp www.cs-201-24.lab.crt /lab/tls/server.crt
cp www.cs-201-24.lab.key /lab/tls/server.key
web-reload
nginx: the configuration file /lab/nginx.conf syntax is ok
nginx: configuration file /lab/nginx.conf test is successful
2026/11/17 17:37:46 [notice] 63#63: signal process started
root@cs-201-24:/lab/work/lego/certificates# curl -sS --cacert /lab/tls/acme-root.crt https://www.cs-201-24.lab/
curl: (60) SSL certificate problem: unable to get local issuer certificate
More details here: https://curl.se/docs/sslcerts.html

curl failed to verify the legitimacy of the server and therefore could not
establish a secure connection to it. To learn more about this situation and
how to fix it, please visit the web page mentioned above.
root@cs-201-24:/lab/work/lego/certificates# openssl s_client -connect www.cs-201-24.lab:443 -servername www.cs-201-24.lab -CAfile /lab/tls/acme-root.crt \
</dev/null 2>/dev/null | sed -n '/^Certificate chain/,/^---/p;/^Verify return code/p'
Certificate chain
 0 s:CN = www.cs-201-24.lab
   i:CN = Pebble Intermediate CA 0242ac
   a:PKEY: id-ecPublicKey, 256 (bit); sigalg: RSA-SHA256
   v:NotBefore: Nov 17 17:33:08 2026 GMT; NotAfter: Nov 17 17:33:07 2031 GMT
---
Verify return code: 21 (unable to verify the first certificate)
```

Вивід кроку 3.6:

```text
root@cs-201-24:/lab/work/lego/certificates# cat www.cs-201-24.lab.crt www.cs-201-24.lab.issuer.crt > /lab/tls/server.crt
web-reload
nginx: the configuration file /lab/nginx.conf syntax is ok
nginx: configuration file /lab/nginx.conf test is successful
2026/11/17 17:41:01 [notice] 71#71: signal process started

root@cs-201-24:/lab/work/lego/certificates# curl -sS --cacert /lab/tls/acme-root.crt https://www.cs-201-24.lab/
curl -sS --cacert /lab/tls/acme-root.crt https://api.cs-201-24.lab/
openssl s_client -connect www.cs-201-24.lab:443 -servername www.cs-201-24.lab -CAfile /lab/tls/acme-root.crt \
</dev/null 2>/dev/null | sed -n '/^Certificate chain/,/^---/p;/^Verify return code/p'
server: cs-201-24 (https)
Host: www.cs-201-24.lab
SNI: www.cs-201-24.lab
protocol: TLSv1.3
server: cs-201-24 (https)
Host: api.cs-201-24.lab
SNI: api.cs-201-24.lab
protocol: TLSv1.3
Certificate chain
 0 s:CN = www.cs-201-24.lab
   i:CN = Pebble Intermediate CA 0242ac
   a:PKEY: id-ecPublicKey, 256 (bit); sigalg: RSA-SHA256
   v:NotBefore: Nov 17 17:33:08 2026 GMT; NotAfter: Nov 17 17:33:07 2031 GMT
 1 s:CN = Pebble Intermediate CA 0242ac
   i:CN = Pebble Root CA 557e75
   a:PKEY: rsaEncryption, 2048 (bit); sigalg: RSA-SHA256
   v:NotBefore: Nov 17 17:18:31 2026 GMT; NotAfter: Nov 17 17:18:31 2056 GMT
---
Verify return code: 0 (ok)
```

| Параметр | Значення |
|---|---|
| Видавець сертифіката сервера |Pebble Intermediate CA 0242ac |
| Видавець сертифіката видавця |Pebble Root CA 557e75 |
| Імена в SAN |www.cs-201-24.lab, api.cs-201-24.lab |
| Строк дії (від – до) |Nov 17 17:33:08 2026 GMT – Nov 17 17:33:07 2031 GMT |
| Тип ключа сервера і алгоритм підпису |id-ecPublicKey, 256 біт,  |
| Кількість сертифікатів, які надсилає сервер, у кроці 3.5 і в кроці 3.6 |У кроці 3.5 — 1 сертифікат, у кроці 3.6 — 2 сертифікати |

1. Хто і як перевірив, що ви контролюєте обидва імена? Вкажіть рядки виводу, які це підтверджують.
Перевірив сам центр сертифікації ACME. Програма lego поклала на мій сайт спеціальні файли-паролі, а центр сертифікації зайшов на обидва сайти і прочитав їх. Так він переконався, що сайти справді мої.
Рядки підтвердження (з виводів lego і nginx):
[api.cs-201-24.lab] The server validated our request
[www.cs-201-24.lab] The server validated our request
"GET /.well-known/acme-challenge/... HTTP/1.1" 200
3. Чому в кроці 3.5 перевірка не пройшла, хоча корінь центру сертифікації вказано в `--cacert`, а сертифікат виданий правильно?
Мій сертифікат підписав посередник, а не сам головний центр. У кроці 3.5 сервер віддав тільки свій сертифікат, а про посередника забув. Через це програма не змогла простежити зв'язок від мого сайту до головного центру.
4. У виводі ланцюжка для сертифіката сервера вказано `id-ecPublicKey` і `RSA-SHA256`. Що означає кожне з цих значень і чому вони не суперечать одне одному?
id-ecPublicKey — це тип мого ключа. RSA-SHA256 — це тип підпису, яким центр сертифікації завірив мій документ своїм ключем.
Вони не суперечать одне одному, бо це два різні ключі: я використовую свій, а центр сертифікації ставить печатку своїм.
5. Що станеться з перевіркою вашого сервера після перезапуску стенда і чому? Що в реальній інфраструктурі відповідає цій ситуації?
Після перезапуску перевірка видасть помилку, тому що стенд при кожному запуску створює новий головний центр сертифікації, а мій збережений файл acme-root.crt залишиться від старого.
У реальному житті таке буває, коли центр сертифікації оновлює свої головні сертифікати або коли людина заходить на сайт зі старого телефона чи комп'ютера, де ще немає оновленої бази сертифікатів.
## Завдання 4. Діагностика несправностей

Вивід кроку 4.1:

```text
root@cs-201-24:/lab/work# for i in 1 2 3 4 5; do
  n=f$i.cs-201-24.lab; echo "=== $n"
  curl -sS --cacert $STAND_CA https://$n/ 2>&1 | head -1
  openssl s_client -connect $n:443 -servername $n -CAfile $STAND_CA </dev/null 2>/dev/null \
    | sed -n '/^Certificate chain/,/^---/p;/^Verify return code/p' | grep -v PKEY
done
=== f1.cs-201-24.lab
curl: (60) SSL certificate problem: certificate has expired
Certificate chain
 0 s:CN = f1.cs-201-24.lab
   i:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   v:NotBefore: May  1 16:28:20 2026 GMT; NotAfter: Oct 28 16:28:20 2026 GMT
 1 s:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   i:O = OPISM Lab, CN = OPISM Stand Root CA
   v:NotBefore: Nov 17 16:28:19 2026 GMT; NotAfter: Nov 16 16:28:19 2031 GMT
---
Verify return code: 10 (certificate has expired)
=== f2.cs-201-24.lab
curl: (60) SSL: no alternative certificate subject name matches target host name 'f2.cs-201-24.lab'
Certificate chain
 0 s:CN = intra.cs-201-24.lab
   i:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   v:NotBefore: Nov 17 16:28:20 2026 GMT; NotAfter: Feb 15 16:28:20 2027 GMT
 1 s:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   i:O = OPISM Lab, CN = OPISM Stand Root CA
   v:NotBefore: Nov 17 16:28:19 2026 GMT; NotAfter: Nov 16 16:28:19 2031 GMT
---
Verify return code: 0 (ok)
=== f3.cs-201-24.lab
server: web, f3
Certificate chain
 0 s:CN = f3.cs-201-24.lab
   i:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   v:NotBefore: Nov 17 16:28:20 2026 GMT; NotAfter: Feb 15 16:28:20 2027 GMT
 1 s:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   i:O = OPISM Lab, CN = OPISM Stand Root CA
   v:NotBefore: Nov 17 16:28:19 2026 GMT; NotAfter: Nov 16 16:28:19 2031 GMT
---
Verify return code: 0 (ok)
=== f4.cs-201-24.lab
curl: (60) SSL certificate problem: unable to get local issuer certificate
Certificate chain
 0 s:CN = f4.cs-201-24.lab
   i:O = Foreign Org, CN = Foreign Intermediate CA
   v:NotBefore: Nov 17 16:28:20 2026 GMT; NotAfter: Feb 15 16:28:20 2027 GMT
 1 s:O = Foreign Org, CN = Foreign Intermediate CA
   i:O = Foreign Org, CN = Foreign Root CA
   v:NotBefore: Nov 17 16:28:19 2026 GMT; NotAfter: Nov 16 16:28:19 2031 GMT
---
Verify return code: 20 (unable to get local issuer certificate)
=== f5.cs-201-24.lab
curl: (60) SSL certificate problem: self-signed certificate
Certificate chain
 0 s:CN = f5.cs-201-24.lab
   i:CN = f5.cs-201-24.lab
   v:NotBefore: Nov 17 16:28:20 2026 GMT; NotAfter: Feb 15 16:28:20 2027 GMT
---
Verify return code: 18 (self-signed certificate)
```

| Сервер | Помилка `curl` | `Verify return code` | Яка перевірка не пройшла | Рядок виводу, що це доводить | Що має змінити власник сервера |
|---|---|---|---|---|---|
| f1 |certificate has expired |10 (certificate has expired) |прострочений сертифікат |v:NotAfter: Oct 28 16:28:20 2026 GMT |Оновити та перевипустити сертифікат |
| f2 |no alternative certificate subject name matches... |0 (ok) |ім'я відсутнє в SAN |0 s:CN = intra.cs-201-24.lab |Перевипустити сертифікат, додавши правильне ім'я домену f2.cs-201-24.lab у список SAN |
| f3 |успішно |0 (ok) |справний сервер |Verify return code: 0 (ok) |Нічого |
| f4 |unable to get local issuer certificate |20 (unable to get local issuer certificate) |сертифікат виданий центром, якому клієнт не довіряє |i:CN = Foreign Intermediate CA |Налаштувати довіру до стороннього ЦС на клієнті або замінити сертифікат на виданий довіреним стендовим ЦС |
| f5 |self-signed certificate |18 (self-signed certificate) |самопідписаний сертифікат |i:CN = f5.cs-201-24.lab |Замінити самопідписаний сертифікат на дійсний, виданий центром сертифікації |

1. Для сервера з помилкою `unable to get local issuer certificate` визначте, яка це несправність: сервер не надсилає проміжний сертифікат чи сертифікат виданий центром, якому клієнт не довіряє. Чим ці дві несправності відрізняються у виводі `openssl s_client`?
Для сервера з помилкою unable to get local issuer certificate у нашому випадку (f4) це несправність, коли сертифікат видано центром, якому клієнт не довіряє. У виводі openssl s_client ці дві несправності відрізняються тим, що при відсутності проміжного сертифіката сервер передає лише один власний сертифікат і ланцюжок обривається на знайомому знайомому кореневому сертифікаті, тоді як у випадку недовіреного центру сервер надсилає повний ланцюжок із двох сертифікатів, але в кінці фігурує сторонній кореневий центр, якого немає в базі довірених клієнта.
2. Якщо серед ваших серверів є сервер з невідповідним ім'ям, чому `openssl s_client` повертає для нього `Verify return code: 0 (ok)`, а `curl` повідомляє про помилку? Якщо такого сервера немає, дайте відповідь для загального випадку.
Сервер з невідповідним ім'ям (f2) проходить перевірку в OpenSSL, але видає помилку в curl, тому що openssl s_client за замовчуванням перевіряє лише криптографічну правильність підпису і ланцюжка сертифікатів, тоді як curl додатково суворо перевіряє, чи збігається ім'я сайту, до якого звертаються, із записами SAN у сертифікаті.
3. Які з п'яти серверів відповіли б на запит `curl -k`? Що при цьому втрачається порівняно з `--cacert`?
На запит із ключем -k відповіли б усі п'ять серверів, оскільки цей ключ повністю примусово вимикає перевірку сертифікатів, але при цьому втрачається безпека і захищеність з'єднання, бо клієнт перестає перевіряти справжність сервера, роблячи канал вразливим до перехоплення.
## Завдання 5. Ціна нового з'єднання

**Прогноз** (записати до вимірювання). Скільки RTT мине до завершення кожного етапу:

| Етап | TLS 1.3, нове з'єднання | TLS 1.2, нове з'єднання | Повторне використання з'єднання |
|---|---|---|---|
| `time_connect` |1 RTT |1 RTT |0 RTT |
| `time_appconnect` |2 RTT |2 RTT |0 RTT |
| `time_starttransfer` |2 RTT |3 RTT |1 RTT |

Вивід кроків 5.2 і 5.3:

```text
0.035670 0.035782 0.040711 0.040843 0.040881 1.1 1
0.035511 0.035605 0.039003 0.039183 0.039220 1.1 1
0.035457 0.035548 0.038879 0.038992 0.039038 1.1 1
0.035461 0.035548 0.039098 0.039207 0.039245 1.1 1
0.035488 0.035587 0.038946 0.039053 0.039099 1.1 1
0.035795 0.035887 0.039205 0.039322 0.039362 1.1 1
0.035480 0.035568 0.038932 0.039045 0.039085 1.1 1
0.000015 0.000000 0.000000 0.000105 0.000131 1.1 0
0.000010 0.000000 0.000000 0.000113 0.000131 1.1 0І
```

<!-- Значення в мілісекундах. Для нових з'єднань — медіана трьох вимірювань. -->

| Запит | `namelookup` | `connect` | `appconnect` | `starttransfer` | `total` | `num_connects` |
|---|---|---|---|---|---|---|
| TLS 1.3, нове |35,511 мс |35,605 мс|39,003	мс |39,183	мс |39,220	мс |1 мс|
| TLS 1.2, нове |35,488	мс |35,587 мс |39,098	 мс|39,207	мс |39,245 мс|1 мс |
| 5.3, перший запит |35,480 мс|35,568	 мс|38,932 мс|39,045 мс |39,085	 мс|1 мс|
| 5.3, другий запит |0,015 мс|0 мс|0 мс|0,105 мс |0,131 мс|0 мс|
| 5.3, третій запит |0,010 мс|0 мс|0 мс|0,113 мс|0,131	 мс|0 мс|

RTT, визначений вимірюванням, і з якого значення його визначено: приблизно 35,6 мс за значенням медіани трьох вимірювань, що показує час встановлення TCP-з'єднання.

1. Виразіть кожне виміряне значення через RTT. Де вимірювання відрізняється від цілого числа RTT і чим це пояснюється?
Нехай виміряний RTT приблизно за 35,6 мс. Значення time_namelookup і time_connect становлять приблизно 1 RTT. Значення time_appconnect, time_starttransfer і time_total — приблизно 1,1 RTT. Відхилення від цілого числа RTT пояснюються додатковим часом на обробку запитів, встановлення TLS-з'єднання та формування відповіді сервером.
2. Чому `time_namelookup` не близький до нуля, хоча резолвер уже має ім'я в кеші? За яких умов він був би приблизно вдвічі більшим?
Тому що кеш DNS знаходиться на резолвері, а клієнту все одно потрібно надіслати йому запит та отримати відповідь через мережу. На це витрачається приблизно один RTT.
Час був би приблизно вдвічі більшим, якби запису не було в кеші резолвера і йому довелося б додатково звертатися до авторитетного DNS-сервера.
3. Сервіс виконує 200 послідовних запитів до цього сервера. Розрахуйте, скільки часу займе лише очікування мережі, якщо кожен запит встановлює нове з'єднання TLS 1.3, нове з'єднання TLS 1.2 і якщо всі запити йдуть через одне з'єднання.
Якщо сервіс виконує 200 послідовних запитів, кожного разу встановлюючи нове з'єднання TLS 1.3, загальний час очікування становитиме приблизно 7,844 секунди. Для TLS 1.2 — приблизно 7,849 секунди. Якщо ж усі запити виконуються через одне спільне з'єднання, час становитиме лише приблизно 0,065 секунди. Використання одного постійного з'єднання значно економить час, оскільки не потрібно щоразу встановлювати нове TCP-з'єднання та виконувати TLS-рукостискання.
## Висновки

**Що виявилося несподіваним.** Результат, який не збігся з очікуванням, і рядок виводу, де це видно:
Найбільш несподіваним було те (5 завдання), наскільки сильно відрізняється час виконання запитів через нове та вже встановлене з'єднання. Перший запит займав приблизно 39 мс, а наступні — лише 0,131 мс. Це видно в останніх двох рядках виводу, де num_connects дорівнює 0. Також виявилося, що різниця між TLS 1.2 і TLS 1.3 за результатами вимірювань дуже незначна. Рядки, що підтверджують:
0.000015 0.000000 0.000000 0.000105 0.000131 1.1 0
0.000010 0.000000 0.000000 0.000113 0.000131 1.1 0
**Яке питання залишилося без відповіді.** Що саме не вдалося пояснити; до якого рядка виводу це стосується; що вже перевірено і що з'ясувати не вдалося:
Залишилося питання (завдання 5), наскільки результати вимірювання залежать від навантаження на сервер і мережу. Було перевірено час DNS-запиту, встановлення TCP- і TLS-з'єднань, а також повторне використання з'єднання, але не вдалося точно визначити, як зміняться ці показники за великої кількості одночасних клієнтів. Рядки, що підтверджують: 
0.035670 0.035782 0.040711 0.040843 0.040881 1.1 1
0.035461 0.035548 0.039098 0.039207 0.039245 1.1 1
## Декларація використання ШІ

Використовувалися інструменти ШІ: <!-- так / ні -->

<!-- Якщо так, заповнити таблицю: один рядок на кожне звернення, що вплинуло на звіт. Промпт наводити дослівно. -->

| № | Інструмент | Промпт (дослівно) | Що використано у звіті і що змінено самостійно |
|---|---|---|---|
| 1 |ChatGPT (OpenAI) |допоможи із висновком |Використано допомогу у формулюванні висновку, самостійно перевірено отримані результати, підібрано відповідні рядки виводу та відредаговано текст висновку. |
| 1 |ChatGPT (OpenAI) |поясни, як заповнити таблицю вимірювання часу роботи мережевих запитів за отриманими результатами у 5 заваднні і допоможи із розрахунками у 3 питанні цього ж завдання  |Використано пояснення показників time_namelookup, time_connect, time_appconnect, time_starttransfer і time_total, допомогу з математичними розрахунками та порівнянням часу виконання запитів. Самостійно виконано команди та отримано результати вимірювань, перевірено отримані результати та внесено їх у звіт.|
