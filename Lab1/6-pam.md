# 6. PAM-модулі (*)

Заготовки (`<USER>` тощо) і позначки 🟢🟡🔴 дивись у [README](README.md).

## Терміни

**PAM (Pluggable Authentication Modules)** охоронець на вході. Проміжний шар, через який проходить будь-яка автентифікація, login, su, sudo, passwd. Його можна гнучко налаштовувати без зміни самих програм.

**стек PAM** для кожної дії, вхід, зміна пароля і так далі, є послідовність модулів у `/etc/pam.d/`. Кожен модуль виконує свою перевірку.

**auth / account / password / session** чотири типи правил PAM. auth перевіряє автентичність. account перевіряє обліковий запис, чи не заблокований і чи не прострочений. password відповідає за зміну пароля. session відповідає за дії на початку й кінці сесії.

**required / requisite / sufficient / optional** як результат модуля впливає на весь стек. required означає, що модуль має бути обов'язково успішним, інакше стек провалиться, але перевірка все одно триватиме далі. requisite означає те саме, але стек зупиняється одразу. sufficient означає, що якщо модуль успішний, решту можна пропустити.

## Складність пароля, pam_pwquality 🔴

```bash
sudo apt install libpam-pwquality

sudo nano /etc/pam.d/common-password
# додати рядок перед рядком з pam_unix.so
# password requisite pam_pwquality.so retry=3 minlen=10 dcredit=-1 ucredit=-1 ocredit=-1 lcredit=-1
```

Параметр `minlen=10` означає мінімум 10 символів. Параметри `dcredit`, `ucredit`, `lcredit`, `ocredit`, кожен рівний -1, означають вимогу щонайменше однієї цифри, однієї великої літери, однієї малої літери і одного спецсимвола відповідно. Параметр `retry=3` означає, скільки спроб дати користувачу ввести пароль, що відповідає вимогам.

Перевірка. `sudo passwd <USER>`, спробувати задати простий пароль, система має відхилити з поясненням.

## Інший алгоритм гешування 🔴

```bash
sudo nano /etc/pam.d/common-password
# у рядку з pam_unix.so замінити yescrypt на, наприклад, sha512
# password [success=1 default=ignore] pam_unix.so obscure sha512
```

Ця зміна діє лише на паролі, встановлені після зміни. Старі хеші лишаються у попередньому форматі, доки пароль не перевстановлять.

## Блокування акаунта після кількох невдалих спроб, pam_faillock 🔴

```bash
sudo nano /etc/pam.d/common-auth
# на початку файлу
# auth required pam_faillock.so preauth silent deny=3 unlock_time=600
# рядок з pam_unix.so лишити як є, після нього додати
# auth [default=die] pam_faillock.so authfail deny=3 unlock_time=600

sudo nano /etc/pam.d/common-account
# додати
# account required pam_faillock.so
```

Параметр `deny=3` означає блокування після 3 невдалих спроб. Параметр `unlock_time=600` означає автоматичне розблокування через 600 секунд.

```bash
sudo faillock --user <USER>
# показує лічильник невдалих спроб саме для цього користувача

sudo faillock --user <USER> --reset
# вручну скидає лічильник і розблоковує
```

## Обмеження входу за часом доби, pam_time 🔴

```bash
sudo nano /etc/security/time.conf
# заборонити вхід <USER> по буднях з 00:00 до 06:00
# login;*;<USER>;!Wk0600-0000

sudo nano /etc/pam.d/login
# додати
# account requisite pam_time.so
```

Формат правила в time.conf такий: служба, термінал, користувач, дні й години, розділені крапкою з комою. `Wk` означає будні дні, Weekdays. Знак оклику перед проміжком означає заборону саме в цей час.

## Обмеження ресурсів, pam_limits 🔴

```bash
sudo nano /etc/security/limits.conf
# <USER>  hard  nproc   50       максимум одночасних процесів
# <USER>  hard  fsize   102400   максимальний розмір файлу, у кілобайтах
# <USER>  hard  nofile  100      максимум одночасно відкритих файлів
# <USER>  hard  as      524288   максимум пам'яті на процес, у кілобайтах
```

Це застосовується через `pam_limits.so`, зазвичай уже підключений у `/etc/pam.d/common-session`. Перевірка

```bash
su - <USER>
ulimit -a
# порівняти значення з тими, що задані в limits.conf
```

## Суперкористувач лише для певної групи, без пароля, pam_wheel 🔴

```bash
sudo groupadd wheel
sudo usermod -aG wheel <USER>

sudo nano /etc/pam.d/su
# розкоментувати або додати
# auth required pam_wheel.so group=wheel

sudo visudo
# якщо треба й без пароля саме через sudo
# %wheel ALL=(ALL) NOPASSWD: ALL
```

`pam_wheel.so` обмежує команду `su` лише учасниками групи wheel. Окремий рядок у sudoers потрібен додатково, якщо треба, щоб і sudo не запитував пароль для цієї групи.

## Змінні оточення при вході, pam_env 🟡

```bash
sudo nano /etc/security/pam_env.conf
# MY_VAR DEFAULT=hello
```

`pam_env.so`, зазвичай уже підключений у `/etc/pam.d/common-session`, підхоплює ці змінні щоразу при вході. Перевірка, новий логін через `su - <USER>`, потім `echo $MY_VAR`.
