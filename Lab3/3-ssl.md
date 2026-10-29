# Частина 3. SSL openssl 

Усе виконується на одній машині, мережа між VM не потрібна.

## 3.1 Ключ і самопідписаний сертифікат 🟡

```bash
openssl genrsa -aes256 -out priv.key 2048
# genrsa створює таємний RSA ключ довжиною 2048 біт
# -aes256 шифрує сам файл ключа парольною фразою

openssl req -new -key priv.key -out certif.csr -subj "/C=UA/O=<NAME>/OU=<GROUP>/CN=<NAME> <GROUP> zalikova <NUMBER>"
# req -new будує запит на сертифікат, CSR
# -subj задає дані власника без інтерактивних питань
# обов'язково вказати прізвище, групу і номер залікової книжки
# латиниця надійніша за кирилицю, яку openssl без -utf8 може зіпсувати

openssl x509 -req -sha256 -signkey priv.key -in certif.csr -out certif.crt -days 1830
# x509 -req перетворює запит на готовий сертифікат
# -signkey означає самопідпис, той самий ключ підписує сам себе
# -days 1830 це термін дії сертифіката

openssl x509 -text -in certif.crt
openssl x509 -text -in certif.crt > certiftxt.crt
# -text виводить вміст сертифіката у читабельному вигляді, поле Subject містить дані власника

openssl x509 -in certif.crt -pubkey -noout > pubkey.pem
# -pubkey виводить лише відкритий ключ із сертифіката
# -noout прибирає сам текст сертифіката з виводу
```

## 3.2 Перевірка і додавання в довірені 🔴

```bash
openssl verify certif.crt
# перевіряє ланцюжок довіри сертифіката
# для самопідписаного дасть помилку self-signed certificate, це очікувано

sudo cp certif.crt /usr/local/share/ca-certificates/certif.crt
sudo update-ca-certificates
# копіює сертифікат у системне сховище довірених сертифікатів Debian Kali
# update-ca-certificates перебудовує загальний список довірених

openssl verify certif.crt
# тепер має показати certif.crt OK
```

## 3.3 Шифрування файлу 🟡

```bash
openssl rand -out key.bin 64
head -c 1000000 /dev/urandom > largfile.dat
# key.bin це випадковий симетричний ключ
# largfile.dat імітує великий файл для шифрування

openssl enc -aes-256-cbc -salt -pbkdf2 -in largfile.dat -out largfile.dat.enc -pass file:./key.bin
# enc шифрує великий файл швидким симетричним алгоритмом AES
# пароль для шифрування це вміст key.bin, а не текст з клавіатури

openssl pkeyutl -encrypt -pubin -inkey pubkey.pem -in key.bin -out key.bin.enc
# шифрує сам симетричний ключ повільним асиметричним RSA, відкритим ключем
# так великий файл шифрується швидко, а лише короткий ключ шифрується повільно

openssl pkeyutl -decrypt -inkey priv.key -in key.bin.enc -out key.bin2
openssl enc -d -aes-256-cbc -pbkdf2 -in largfile.dat.enc -out largfile.dat2 -pass file:./key.bin2
# decrypt розшифровує ключ таємним ключем власника сертифіката
# enc -d розшифровує сам файл уже цим відновленим ключем

cmp largfile.dat largfile.dat2 && echo FILE_SAME
# cmp мовчить, якщо файли побайтово однакові
# FILE_SAME підтверджує, що розшифрований файл точно збігається з оригіналом
```

У методичці також пропонується зашифрувати ключ через сертифікат (`-certin -inkey certif.crt`) замість окремого `pubkey.pem`. Результат інший файл байт у байт, бо RSA додає випадкове доповнення, але після розшифрування обидва дають той самий `key.bin`.

## 3.4 Цифровий підпис 🟡

```bash
date +%s > example.txt
sha256sum example.txt
# створює тестовий файл і рахує його SHA-256 хеш для довідки

openssl dgst -sha256 -sign priv.key -out example.sha example.txt
# dgst -sign рахує хеш файлу і одразу шифрує його таємним ключем, це і є підпис

openssl dgst -sha256 -verify pubkey.pem -signature example.sha example.txt
# -verify рахує хеш файлу заново і звіряє з розшифрованим підписом
# збіг означає Verified OK
```

Другий, двоетапний спосіб з методички

```bash
openssl dgst -sha256 -binary example.txt > example.hash
openssl pkeyutl -sign -inkey priv.key -pkeyopt digest:sha256 -in example.hash -out example.hash.sign
openssl pkeyutl -verify -pubin -inkey pubkey.pem -pkeyopt digest:sha256 -in example.hash -sigfile example.hash.sign
# тут хеш спочатку рахується окремо, у двійковому вигляді, -binary
# pkeyutl -sign шифрує вже готовий хеш, а не рахує його сам
# pkeyutl -verify перевіряє підпис і виводить Signature Verified Successfully
```

## Прибрати

```bash
sudo rm /usr/local/share/ca-certificates/certif.crt
sudo update-ca-certificates --fresh
# прибирає тестовий сертифікат з довірених системи
```
