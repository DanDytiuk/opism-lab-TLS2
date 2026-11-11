<!--
Бланк звіту до лабораторної роботи «TLS та сертифікати».
Скопіювати цей файл у власний публічний репозиторій звіту як README.md і заповнити.
Вивід команд вставляти текстом у блоки коду, а не знімками екрана.
Порожні поля не допускаються: якщо щось не вдалося, записати, що саме і чому.
-->

# Лабораторна робота 2. TLS та сертифікати

| Поле | Значення |
|---|---|
| ПІБ | Дитюк Даніїл |
| Група | ІПЗ 2.01 |
| Ідентифікатор | se-201-04 |
| Дата виконання | 09.11.26 |
| Середовище | GitHub Codespaces |
| Версія OpenSSL | OpenSSL 3.0.13 30 Jan 2024 (Library: OpenSSL 3.0.13 30 Jan 2024) |

## Параметри варіанта

<!-- Переписати з картки stand/cards/<id>.md. -->

| Параметр | Значення |
|---|---|
| Віддалений сервер | cdn.se-201-04.lab |
| Імена для сертифіката | www.se-201-04.lab, shop.se-201-04.lab |
| Ваш сервер (IP) | 10.53.1.1 |

<!-- Якщо стенд перезапускався під час роботи, вказати, після якого завдання. -->

## Завдання 1. HTTP і HTTPS у дампі трафіку

Вивід кроку 1.1:

```text
root@se-201-04:/lab/work# cd /lab/work
timeout 8 tcpdump -ni eth0 -A -s0 -l 'host 10.53.0.80 and tcp port 80' > t1-http.txt &
sleep 1; curl -s http://cdn.se-201-04.lab/; wait
grep -a -E 'GET /|Host:|server: web' t1-http.txt
[1] 37
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
server: web
Host: cdn.se-201-04.lab
SNI: 
protocol: HTTP/1.1 
client: 10.53.1.1
10 packets captured
10 packets received by filter
0 packets dropped by kernel
[1]+  Exit 124                timeout 8 tcpdump -ni eth0 -A -s0 -l 'host 10.53.0.80 and tcp port 80' > t1-http.txt
06:29:45.673428 IP 10.53.1.1.47670 > 10.53.0.80.80: Flags [P.], seq 1:81, ack 1, win 502, options [nop,nop,TS val 3279010050 ecr 2352498166], length 80: HTTP: GET / HTTP/1.1
.q...8E.GET / HTTP/1.1
Host: cdn.se-201-04.lab
server: web
Host: cdn.se-201-04.lab
```

Вивід кроку 1.2:

```text
root@se-201-04:/lab/work# timeout 8 tcpdump -ni eth0 -A -s0 -l 'host 10.53.0.80 and tcp port 443' > t1-https.txt &
sleep 1; curl -s --cacert $STAND_CA https://cdn.se-201-04.lab//; wait
grep -a -c -E 'GET /|server: web' t1-https.txt
grep -a -o 'cdn.se-201-04.lab/' t1-https.txt | wc -l
[1] 43
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
server: web
Host: cdn.se-201-04.lab
SNI: cdn.se-201-04.lab
protocol: HTTP/2.0 TLSv1.3
client: 10.53.1.1
22 packets captured
22 packets received by filter
0 packets dropped by kernel
[1]+  Exit 124                timeout 8 tcpdump -ni eth0 -A -s0 -l 'host 10.53.0.80 and tcp port 443' > t1-https.txt
0
0
```

| Що шукали в дампі | HTTP | HTTPS |
|---|---|---|
| рядок запиту `GET /` | є | немає |
| заголовок `Host` | є | немає |
| тіло відповіді (`server: web`) | є | немає |
| ім'я віддаленого сервера (кількість входжень) | 2 | 0 |

1. У якому повідомленні і в якому полі передається ім'я сервера, яке видно в дампі HTTPS? Передається у SNI це все 

2. Чому рядок `SNI:` у відповіді сервера порожній для HTTP і заповнений для HTTPS? У HTTP використовується TCP-з'єднання без TLS і SNI пустий, а HTTPS він вже більш захищенний ніж HTTP і вже встановлюється TCP-з'єднання з TLS при цьому потрібно назва сервера і SNI вже не пустий

3. Рядок `protocol:` показує HTTP/1.1 для запиту по HTTP і HTTP/2.0 для запиту по HTTPS, хоча обидва запити виконано однією командою `curl` без додаткових ключів. Що визначило версію HTTP у другому випадку?

## Завдання 2. Рукостискання по записах

Вивід кроку 2.1 (TLS 1.3):

```text
root@se-201-04:/lab/work# openssl s_client -connect cdn.se-201-04.lab:443 -servername cdn.se-201-04.lab -alpn h2,http/1.1 
\                                                     
  -CAfile $STAND_CA -trace </dev/null > t2-13.txt 2>&1
grep -a -E '^(Sent|Received) Record|Content Type|^    [A-Z][A-Za-z]+, Length' t2-13.txt
grep -a -E '^(ALPN|New,)|Verify return' t2-13.txt
Sent Record
  Content Type = Handshake (22)
    ClientHello, Length=328
Received Record
  Content Type = Handshake (22)
    ServerHello, Length=118
Received Record
  Content Type = ChangeCipherSpec (20)
Received Record
  Content Type = ApplicationData (23)
  Inner Content Type = Handshake (22)
    EncryptedExtensions, Length=15
Received Record
  Content Type = ApplicationData (23)
  Inner Content Type = Handshake (22)
    Certificate, Length=981
Received Record
  Content Type = ApplicationData (23)
  Inner Content Type = Handshake (22)
    CertificateVerify, Length=76
Received Record
  Content Type = ApplicationData (23)
  Inner Content Type = Handshake (22)
    Finished, Length=48
Sent Record
  Content Type = ChangeCipherSpec (20)
Sent Record
  Content Type = ApplicationData (23)
  Inner Content Type = Handshake (22)
    Finished, Length=48
Sent Record
  Content Type = ApplicationData (23)
  Inner Content Type = Alert (21)
New, TLSv1.3, Cipher is TLS_AES_256_GCM_SHA384
ALPN protocol: h2
Verify return code: 0 (ok)
```

Вивід кроку 2.2 (TLS 1.2):

```text
root@se-201-04:/lab/work# openssl s_client -connect cdn.se-201-04.lab:443 -servername cdn.se-201-04.lab -alpn h2,http/1.1 \
  -CAfile $STAND_CA -trace -tls1_2 </dev/null > t2-12.txt 2>&1
grep -a -E '^(Sent|Received) Record|Content Type|^    [A-Z][A-Za-z]+, Length' t2-12.txt
grep -a -E '^(ALPN|New,)|Verify return' t2-12.txt
Sent Record
  Content Type = Handshake (22)
    ClientHello, Length=223
Received Record
  Content Type = Handshake (22)
    ServerHello, Length=74
Received Record
  Content Type = Handshake (22)
    Certificate, Length=976
Received Record
  Content Type = Handshake (22)
    ServerKeyExchange, Length=111
Received Record
  Content Type = Handshake (22)
    ServerHelloDone, Length=0
Sent Record
  Content Type = Handshake (22)
    ClientKeyExchange, Length=33
Sent Record
  Content Type = ChangeCipherSpec (20)
Sent Record
  Content Type = Handshake (22)
    Finished, Length=12
Received Record
  Content Type = Handshake (22)
    NewSessionTicket, Length=214
Received Record
  Content Type = ChangeCipherSpec (20)
Received Record
  Content Type = Handshake (22)
    Finished, Length=12
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
| 1 | К→С | Handshake | - | ClientHello | Length=328 |
| 2 | С→К | Handshake | - | ServerHello | Length=118 |
| 3 | С→К | ChangeCipherSpec | - | - | - |
| 4 | С→К | ApplicationData | Handshake | EncryptedExtensions | Length=15 |
| 5 | С→К | ApplicationData | Handshake | Certificate | Length=981 |
| 6 | С→К | ApplicationData | Handshake | CertificateVerify | Length=76 |
| 7 | С→К | ApplicationData | Handshake | Finished | Length=48 |
| 8 | К→С | ChangeCipherSpec | - | - | - |
| 9 | К→С | ApplicationData | Handshake | Finished | Length=48 |
| 10 | К→С | ApplicationData | Alert | - | - |

Вивід кроків 2.3 і 2.4:

```text
root@se-201-04:/lab/work# grep -a -A3 'extension_type=server_name' t2-13.txt | head -4
        extension_type=server_name(0), length=22
          0000 - 00 14 00 00 11 63 64 6e-2e 73 65 2d 32 30 31   .....cdn.se-201
          000f - 2d 30 34 2e 6c 61 62                           -04.lab
        extension_type=ec_point_formats(11), length=4

root@se-201-04:/lab/work# for v in 1.3 1.2; do
  timeout 6 tcpdump -ni eth0 -A -s0 -l 'host 10.53.0.80 and tcp port 443' > t2-dump-$v.txt &
  sleep 1; curl -s -o /dev/null --tlsv1.2 --tls-max $v --cacert $STAND_CA https://cdn.se-201-04.lab/; wait
  echo "TLS $v: $(grep -a -c 'OPISM Stand Intermediate' t2-dump-$v.txt)"
done
[1] 83
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
21 packets captured
21 packets received by filter
0 packets dropped by kernel
[1]+  Exit 124                timeout 6 tcpdump -ni eth0 -A -s0 -l 'host 10.53.0.80 and tcp port 443' > t2-dump-$v.txt
TLS 1.3: 0
[1] 89
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
| Довжина ClientHello, байт | Length=328 | Length=223 |
| Номер першого запису, тип якого спостерігач не бачить | 4 | 11 |
| Кількість RTT від ClientHello до першого HTTP-запиту | 1 | 2 |
| Чи видно сертифікат у дампі (результат 2.4) | 0 TLS 1.3: 0 | 2 сертифіката TLS 1.2: 2 |
| Обраний протокол ALPN | h2 | h2 |

1. За якими рядками виводу 2.1 визначено, з якого запису починається шифрування? Що бачить спостерігач замість типу кожного наступного запису? Там де ApplicationData і за цією назвою ховається certificate та інші якщо дивитися різницю 1.2 і 1.3 tls

2. Чому ClientHello у TLS 1.3 довший, ніж у TLS 1.2? Назвіть розширення, яке за рахунок цього скорочує рукостискання на один RTT. Є шифрування через це він більший наче як

3. Довжина розширення `server_name` у виводі 2.3 становить 17 байт для імені з 12 символів. З яких частин складається це значення і яким воно буде для вашого імені? У мене 22 байти, а самі вони складаються з імені і типу серверу 

4. Навіщо в TLS 1.3 передається запис ChangeCipherSpec, якщо в рукостисканні він нічого не змінює? Сигнал що шифруємося

## Завдання 3. Сертифікат для власного сервера

Вивід кроків 3.1 і 3.2:

```text
root@se-201-04:/lab/work# acme-root                                                                                        
openssl x509 -in /lab/tls/acme-root.crt -noout -subject -dates
збережено /lab/tls/acme-root.crt
subject=CN = Pebble Root CA 048ce0
notBefore=Nov 10 18:47:03 2026 GMT
notAfter=Nov 10 18:47:03 2056 GMT

root@se-201-04:/lab/work# curl -sS --cacert /lab/tls/acme-root.crt https://www.se-201-04.lab/
curl: (60) SSL certificate problem: self-signed certificate
More details here: https://curl.se/docs/sslcerts.html

curl failed to verify the legitimacy of the server and therefore could not
establish a secure connection to it. To learn more about this situation and
how to fix it, please visit the web page mentioned above.

```

Вивід кроку 3.3 (останні рядки `lego` і рядки журналу nginx):

```text
root@se-201-04:/lab/work# lego --server $ACME_DIR --email admin@se-201-04.lab --accept-tos --path /lab/work/lego \
  --http --http.webroot /var/www/acme \
  --domains www.se-201-04.lab --domains shop.se-201-04.lab run --no-bundle
grep acme-challenge /var/log/nginx/access.log
2026/11/10 19:40:38 No key found for account admin@se-201-04.lab. Generating a P256 key.
2026/11/10 19:40:38 Saved key to /lab/work/lego/accounts/ca.lab_14000/admin@se-201-04.lab/keys/admin@se-201-04.lab.key
2026/11/10 19:40:38 [INFO] acme: Registering account for admin@se-201-04.lab
!!!! HEADS UP !!!!

Your account credentials have been saved in your Let's Encrypt
configuration directory at "/lab/work/lego/accounts".

You should make a secure backup of this folder now. This
configuration directory will also contain certificates and
private keys obtained from Let's Encrypt so making regular
backups of this folder is ideal.
2026/11/10 19:40:38 [INFO] [www.se-201-04.lab, shop.se-201-04.lab] acme: Obtaining SAN certificate
2026/11/10 19:40:38 [INFO] [shop.se-201-04.lab] AuthURL: https://ca.lab:14000/authZ/XL0aj9CuHbhoGOMmgQDp4HJsrut2sUwKk02pQCkFg8E
2026/11/10 19:40:38 [INFO] [www.se-201-04.lab] AuthURL: https://ca.lab:14000/authZ/U-hiCfF2IqVSpXoRZNPt8msG4yEDeB-VTVaYIbcpLto
2026/11/10 19:40:38 [INFO] [shop.se-201-04.lab] acme: Could not find solver for: tls-alpn-01
2026/11/10 19:40:38 [INFO] [shop.se-201-04.lab] acme: use http-01 solver
2026/11/10 19:40:38 [INFO] [www.se-201-04.lab] acme: Could not find solver for: tls-alpn-01
2026/11/10 19:40:38 [INFO] [www.se-201-04.lab] acme: use http-01 solver
2026/11/10 19:40:38 [INFO] [shop.se-201-04.lab] acme: Trying to solve HTTP-01
2026/11/10 19:40:46 [INFO] [shop.se-201-04.lab] The server validated our request
2026/11/10 19:40:46 [INFO] [www.se-201-04.lab] acme: Trying to solve HTTP-01
2026/11/10 19:40:51 [INFO] [www.se-201-04.lab] The server validated our request
2026/11/10 19:40:51 [INFO] [www.se-201-04.lab, shop.se-201-04.lab] acme: Validations succeeded; requesting certificates
2026/11/10 19:40:51 [INFO] Wait for certificate [timeout: 30s, interval: 500ms]
2026/11/10 19:40:51 [INFO] [www.se-201-04.lab] Server responded with a certificate.
10.53.0.10 - - [10/Nov/2026:19:40:38 +0000] "GET /.well-known/acme-challenge/7FI58Dy3bqKHIPsEk57IRrkxYVvuCABm-QivM5Wr1JQ HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
10.53.0.10 - - [10/Nov/2026:19:40:38 +0000] "GET /.well-known/acme-challenge/7FI58Dy3bqKHIPsEk57IRrkxYVvuCABm-QivM5Wr1JQ HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
10.53.0.10 - - [10/Nov/2026:19:40:38 +0000] "GET /.well-known/acme-challenge/7FI58Dy3bqKHIPsEk57IRrkxYVvuCABm-QivM5Wr1JQ HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
10.53.0.10 - - [10/Nov/2026:19:40:46 +0000] "GET /.well-known/acme-challenge/jb9a_-Q9X27Q_28J4GBi3wi_lg6kCCM-uXCYMWsO48E HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
10.53.0.10 - - [10/Nov/2026:19:40:46 +0000] "GET /.well-known/acme-challenge/jb9a_-Q9X27Q_28J4GBi3wi_lg6kCCM-uXCYMWsO48E HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
10.53.0.10 - - [10/Nov/2026:19:40:46 +0000] "GET /.well-known/acme-challenge/jb9a_-Q9X27Q_28J4GBi3wi_lg6kCCM-uXCYMWsO48E HTTP/1.1" 200 87 "-" "LetsEncrypt-Pebble-VA (linux; amd64)"
```

Вивід кроку 3.4:

```text
root@se-201-04:/lab/work# cd /lab/work/lego/certificates
openssl x509 -in www.se-201-04.lab.crt -noout -subject -issuer -dates -serial -ext subjectAltName
openssl x509 -in www.se-201-04.lab.issuer.crt -noout -subject -issuer
subject=CN = www.se-201-04.lab
issuer=CN = Pebble Intermediate CA 3ed25a
notBefore=Nov 10 19:40:51 2026 GMT
notAfter=Nov 10 19:40:50 2031 GMT
serial=54C957474916CB0F
X509v3 Subject Alternative Name: 
    DNS:www.se-201-04.lab, DNS:shop.se-201-04.lab
subject=CN = Pebble Intermediate CA 3ed25a
issuer=CN = Pebble Root CA 048ce0
```

Вивід кроку 3.5:

```text
root@se-201-04:/lab/work/lego/certificates# cp www.se-201-04.lab.crt /lab/tls/server.crt
cp www.se-201-04.lab.key /lab/tls/server.key
web-reload
curl -sS --cacert /lab/tls/acme-root.crt https://www.se-201-04.lab/
openssl s_client -connect www.se-201-04.lab:443 -servername www.se-201-04.lab -CAfile /lab/tls/acme-root.crt \
  </dev/null 2>/dev/null | sed -n '/^Certificate chain/,/^---/p;/^Verify return code/p'
nginx: the configuration file /lab/nginx.conf syntax is ok
nginx: configuration file /lab/nginx.conf test is successful
2026/11/10 19:43:38 [notice] 114#114: signal process started
curl: (60) SSL certificate problem: unable to get local issuer certificate
More details here: https://curl.se/docs/sslcerts.html

curl failed to verify the legitimacy of the server and therefore could not
establish a secure connection to it. To learn more about this situation and
how to fix it, please visit the web page mentioned above.
Certificate chain
 0 s:CN = www.se-201-04.lab
   i:CN = Pebble Intermediate CA 3ed25a
   a:PKEY: id-ecPublicKey, 256 (bit); sigalg: RSA-SHA256
   v:NotBefore: Nov 10 19:40:51 2026 GMT; NotAfter: Nov 10 19:40:50 2031 GMT
---
Verify return code: 21 (unable to verify the first certificate)
```

Вивід кроку 3.6:

```text
root@se-201-04:/lab/work/lego/certificates# cat www.se-201-04.lab.crt www.se-201-04.lab.issuer.crt > /lab/tls/server.crt
web-reload
nginx: the configuration file /lab/nginx.conf syntax is ok
nginx: configuration file /lab/nginx.conf test is successful
2026/11/10 19:46:40 [notice] 123#123: signal process started
```

| Параметр | Значення |
|---|---|
| Видавець сертифіката сервера | Pebble Intermediate CA 3ed25a |
| Видавець сертифіката видавця | Pebble Root CA 048ce0 |
| Імена в SAN | www.se-201-04.lab, shop.se-201-04.lab |
| Строк дії (від – до) | v:NotBefore: Nov 10 19:40:51 2026 GMT; NotAfter: Nov 10 19:40:50 2031 GMT |
| Тип ключа сервера і алгоритм підпису | id-ecPublicKey, 256 (bit); sigalg: RSA-SHA256 |
| Кількість сертифікатів, які надсилає сервер, у кроці 3.5 і в кроці 3.6 | 1 і 2? |

1. Хто і як перевірив, що ви контролюєте обидва імена? Вкажіть рядки виводу, які це підтверджують.
```text
2026/11/10 19:40:38 [INFO] [shop.se-201-04.lab] acme: Trying to solve HTTP-01
2026/11/10 19:40:46 [INFO] [shop.se-201-04.lab] The server validated our request
2026/11/10 19:40:46 [INFO] [www.se-201-04.lab] acme: Trying to solve HTTP-01
2026/11/10 19:40:51 [INFO] [www.se-201-04.lab] The server validated our request
2026/11/10 19:40:51 [INFO] [www.se-201-04.lab, shop.se-201-04.lab] acme: Validations succeeded; requesting certificates
```
2. Чому в кроці 3.5 перевірка не пройшла, хоча корінь центру сертифікації вказано в `--cacert`, а сертифікат виданий правильно?
```text
Certificate chain
 0 s:CN = www.se-201-04.lab
   i:CN = Pebble Intermediate CA 3ed25a
   a:PKEY: id-ecPublicKey, 256 (bit); sigalg: RSA-SHA256
   v:NotBefore: Nov 10 19:40:51 2026 GMT; NotAfter: Nov 10 19:40:50 2031 GMT

Через промежний сертифікат?
```

3. У виводі ланцюжка для сертифіката сервера вказано `id-ecPublicKey` і `RSA-SHA256`. Що означає кожне з цих значень і чому вони не суперечать одне одному? 1 значення означає відкритий ключ і його тип ес. 2 значення означає алгоритм шифрування ключа і розмір

4. Що станеться з перевіркою вашого сервера після перезапуску стенда і чому? Що в реальній інфраструктурі відповідає цій ситуації? Після перезапуску стенда перевірка сертифіката може перестати проходити, через те що Pebble використовує тимчасову інфраструктуру, в реальності зміна сертифіката більш доцільна реальність з ситуацією на стенді

## Завдання 4. Діагностика несправностей

Вивід кроку 4.1:

```text
root@se-201-04:/lab/work/lego/certificates# for i in 1 2 3 4 5; do
  n=f$i.se-201-04.lab; echo "=== $n"
  curl -sS --cacert $STAND_CA https://$n/ 2>&1 | head -1
  openssl s_client -connect $n:443 -servername $n -CAfile $STAND_CA </dev/null 2>/dev/null \
    | sed -n '/^Certificate chain/,/^---/p;/^Verify return code/p' | grep -v PKEY
done
=== f1.se-201-04.lab
curl: (60) SSL certificate problem: unable to get local issuer certificate
Certificate chain
 0 s:CN = f1.se-201-04.lab
   i:O = Foreign Org, CN = Foreign Intermediate CA
   v:NotBefore: Nov 10 18:47:01 2026 GMT; NotAfter: Feb  8 18:47:01 2027 GMT
 1 s:O = Foreign Org, CN = Foreign Intermediate CA
   i:O = Foreign Org, CN = Foreign Root CA
   v:NotBefore: Nov 10 18:47:01 2026 GMT; NotAfter: Nov  9 18:47:01 2031 GMT
---
Verify return code: 20 (unable to get local issuer certificate)
=== f2.se-201-04.lab
curl: (60) SSL: no alternative certificate subject name matches target host name 'f2.se-201-04.lab'
Certificate chain
 0 s:CN = staging.se-201-04.lab
   i:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   v:NotBefore: Nov 10 18:47:01 2026 GMT; NotAfter: Feb  8 18:47:01 2027 GMT
 1 s:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   i:O = OPISM Lab, CN = OPISM Stand Root CA
   v:NotBefore: Nov 10 18:47:01 2026 GMT; NotAfter: Nov  9 18:47:01 2031 GMT
---
Verify return code: 0 (ok)
=== f3.se-201-04.lab
curl: (60) SSL certificate problem: unable to get local issuer certificate
Certificate chain
 0 s:CN = f3.se-201-04.lab
   i:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   v:NotBefore: Nov 10 18:47:01 2026 GMT; NotAfter: Feb  8 18:47:01 2027 GMT
---
Verify return code: 21 (unable to verify the first certificate)
=== f4.se-201-04.lab
curl: (60) SSL certificate problem: self-signed certificate
Certificate chain
 0 s:CN = f4.se-201-04.lab
   i:CN = f4.se-201-04.lab
   v:NotBefore: Nov 10 18:47:01 2026 GMT; NotAfter: Feb  8 18:47:01 2027 GMT
---
Verify return code: 18 (self-signed certificate)
=== f5.se-201-04.lab
server: web, f5
Certificate chain
 0 s:CN = f5.se-201-04.lab
   i:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   v:NotBefore: Nov 10 18:47:01 2026 GMT; NotAfter: Feb  8 18:47:01 2027 GMT
 1 s:O = OPISM Lab, CN = OPISM Stand Intermediate CA
   i:O = OPISM Lab, CN = OPISM Stand Root CA
   v:NotBefore: Nov 10 18:47:01 2026 GMT; NotAfter: Nov  9 18:47:01 2031 GMT
---
Verify return code: 0 (ok)
```

| Сервер | Помилка `curl` | `Verify return code` | Яка перевірка не пройшла | Рядок виводу, що це доводить | Що має змінити власник сервера |
|---|---|---|---|---|---|
| f1 | unable to get local issuer certificate | 20 | неможливо отримати сертифікат місцевого емітента | s:O = Foreign Org, CN = Foreign Intermediate CA | отримати сертифікат від довіренного органу |
| f2 | no alternative certificate subject name matches target host name 'f2.se-201-04.lab' | 0 | немає альтернативної назви предмета сертифіката | Certificate chain | Заміна сертифіката |
| f3 | unable to get local issuer certificate | 21 | перевірка повноти сертифікатів | Verify return code: 21 (unable to verify the first certificate) | проміжний сертифікат додати |
| f4 | self-signed certificate | 18 | самопідписаний сертифікат | s:CN = f4.se-201-04.lab
   i:CN = f4.se-201-04.lab | отримати сертифікат від довіренного органу |
| f5 | - | 0 | - | Verify return code: 0 (ok) | нічого |

1. Для сервера з помилкою `unable to get local issuer certificate` визначте, яка це несправність: сервер не надсилає проміжний сертифікат чи сертифікат виданий центром, якому клієнт не довіряє. Чим ці дві несправності відрізняються у виводі `openssl s_client`? f1 сертифікат виданий центром, якому клієнт не довіряє, а f3 сервер не надсилає проміжний сертифікат. Вони відрізняються тим що у 1 випадку сервер передає якийсь сертифікат який не подобається клієнту, а у 3 випадку немає одного проміжного сертифікату і через це помилка

2. Якщо серед ваших серверів є сервер з невідповідним ім'ям, чому `openssl s_client` повертає для нього `Verify return code: 0 (ok)`, а `curl` повідомляє про помилку? Якщо такого сервера немає, дайте відповідь для загального випадку. Не той домен перевіряється

3. Які з п'яти серверів відповіли б на запит `curl -k`? Що при цьому втрачається порівняно з `--cacert`? Наче всі

## Завдання 5. Ціна нового з'єднання

**Прогноз** (записати до вимірювання). Скільки RTT мине до завершення кожного етапу:

| Етап | TLS 1.3, нове з'єднання | TLS 1.2, нове з'єднання | Повторне використання з'єднання |
|---|---|---|---|
| `time_connect` | 1 | 1 | 0 |
| `time_appconnect` | 2 | 3 | 0 |
| `time_starttransfer` | 3 | 4 | 1 |

Вивід кроків 5.2 і 5.3:

```text
root@se-201-04:/lab/work/lego/certificates# F='%{time_namelookup} %{time_connect} %{time_appconnect} %{time_starttransfer} %
{time_total} %{http_version} %{num_connects}\n'
U=https://cdn.se-201-04.lab/
for i in 1 2 3; do curl -s -o /dev/null --cacert $STAND_CA -w "$F" $U; done
for i in 1 2 3; do curl -s -o /dev/null --tls-max 1.2 --cacert $STAND_CA -w "$F" $U; done
0.071633 0.106827 0.145911 0.181237 0.181296 2 1
0.035466 0.071389 0.110682 0.146051 0.146128 2 1
0.035536 0.070700 0.110004 0.145305 0.145371 2 1
0.035443 0.070570 0.144217 0.179581 0.179638 2 1
0.035436 0.070597 0.144904 0.180244 0.180304 2 1
0.035460 0.070641 0.144923 0.180337 0.180418 2 1

root@se-201-04:/lab/work/lego/certificates# curl -s --cacert $STAND_CA -w "$F" -o /dev/null $U -o /dev/null $U -o /dev/null $U
0.035548 0.070692 0.109020 0.144390 0.144453 2 1
0.000017 0.000000 0.000000 0.037558 0.037612 2 0
0.000017 0.000000 0.000000 0.035408 0.035453 2 0

```

<!-- Значення в мілісекундах. Для нових з'єднань — медіана трьох вимірювань. -->

| Запит | `namelookup` | `connect` | `appconnect` | `starttransfer` | `total` | `num_connects` |
|---|---|---|---|---|---|---|
| TLS 1.3, нове | 0.071633 | 0.106827 | 0.145911 | 0.181237 | 0.181296 | 1 |
| TLS 1.2, нове | 0.035443 | 0.070570 | 0.144217 | 0.179581 | 0.179638 | 1 |
| 5.3, перший запит | 0.035548 | 0.070692 | 0.109020 | 0.144390 | 0.144453 | 1 |
| 5.3, другий запит | 0.000017 | 0 | 0 | 0.037558 | 0.037612 | 0 |
| 5.3, третій запит | 0.000017 | 0 | 0 | 0.035408 | 0.035453 | 0 |

RTT, визначений вимірюванням, і з якого значення його визначено: time_starttransfer беремо значення і буде 1 RTT = 0.035453

1. Виразіть кожне виміряне значення через RTT. Де вимірювання відрізняється від цілого числа RTT і чим це пояснюється?

| Запит | `namelookup` | `connect` | `appconnect` | `starttransfer` | `total` | `num_connects` |
|---|---|---|---|---|---|---|
| TLS 1.3, нове | 2 | 3 | 4 | 4 | 5 | 1 |
| TLS 1.2, нове | 1 | 2 | 4 | 5 | 5 | 1 |
| 5.3, перший запит | 1 | 2 | 3 | 4 | 4 | 1 |
| 5.3, другий запит | 0 | 0 | 0 | 1 | 1 | 0 |
| 5.3, третій запит | 0 | 0 | 0 | 1 | 1 | 0 |

2. Чому `time_namelookup` не близький до нуля, хоча резолвер уже має ім'я в кеші? За яких умов він був би приблизно вдвічі більшим? DNS запит надсилається в любому випадку тому не 0


3. Сервіс виконує 200 послідовних запитів до цього сервера. Розрахуйте, скільки часу займе лише очікування мережі, якщо кожен запит встановлює нове з'єднання TLS 1.3, нове з'єднання TLS 1.2 і якщо всі запити йдуть через одне з'єднання.

## Висновки

**Що виявилося несподіваним.** Результат, який не збігся з очікуванням, і рядок виводу, де це видно:

**Яке питання залишилося без відповіді.** Що саме не вдалося пояснити; до якого рядка виводу це стосується; що вже перевірено і що з'ясувати не вдалося:

## Декларація використання ШІ

Використовувалися інструменти ШІ: ні

