# 1. Облікові записи, групи, паролі 

Про заготовки (`<USER>` тощо) і позначки 🟢🟡🔴 дивитися [README](README.md).

## Три файли, в яких усе зберігається

| Файл | Що там | Хто читає |
|---|---|---|
| `/etc/passwd` | список користувачів | усі |
| `/etc/shadow` | **хеші паролів** і терміни їх дії | лише root |
| `/etc/group` | список груп і їхніх учасників | усі |

**Хеш** це «відбиток» пароля. Сам пароль не зберігається. Система хешує те, що ти ввів, і порівнює з відбитком. З відбитка пароль зворотно не відновити, його лише підбирають перебором (John the Ripper нижче).

### Поля `/etc/passwd` (розділювач `:`)

anna:x:1001:1001:Anna K.:/home/anna:/bin/bash
1 2 3 4 5 6 7

1. ім'я; 2. `x` = хеш у `/etc/shadow`; 3. UID; 4. GID основної групи; 5. коментар; 6. домашній каталог; 7. оболонка входу (`/usr/sbin/nologin` або `/bin/false` = вхід заборонено).

### Поля `/etc/shadow`

anna:$y$j9T$...хеш...:20734:0:99999:7:::
1 2 3 4 5 6 7 8 9

1. ім'я; 2. хеш (`*`/`!` = заблоковано; порожнє = **пароля немає**); 3. дата останньої зміни (дні від 01.01.1970); 4. мін. днів між змінами; 5. макс. термін дії; 6. попередження (днів); 7. неактивність; 8. дата відключення; 9. резерв.

### Поля `/etc/group`

team:x:1002:anna,bob
1 2 3 4

1. назва; 2. `x`; 3. GID; 4. учасники через кому (користувачі з цією групою як основною тут можуть бути не вказані).

---

## 2.1 Аналіз підсистеми автентифікації

### 2.1.1 Які є облікові записи 🟢
```bash
cut -d: -f1 /etc/passwd
```

### 2.1.2 Хто зараз увійшов 🟢
```bash
who
w
last -n 10
```

### 2.1.3 Локально чи віддалено 🟢
```bash
who -a
```
`(:0)` / `seat0` — локально; `(IP)` + `pts/N` — віддалено (SSH).

### 2.1.4 Акаунти, створені після встановлення 🟢
```bash
grep -E '^UID_MIN|^UID_MAX' /etc/login.defs
awk -F: '$3 >= 1000 && $3 != 65534 {print $1, $3}' /etc/passwd
```

### 2.1.5 Групи 🟢
```bash
cut -d: -f1 /etc/group
```

### 2.1.6 Хто в якій групі 🟢
```bash
getent group | awk -F: '$4 != "" {print $1": "$4}'
for u in $(cut -d: -f1 /etc/passwd); do echo "$u: $(id -Gn "$u")"; done
```

### 2.1.7 Хто лише в одній групі 🟢
```bash
for u in $(cut -d: -f1 /etc/passwd); do
  [ "$(id -G "$u" | wc -w)" -eq 1 ] && echo "$u"
done
```

### 2.1.8 Хто не може увійти 🟢
```bash
grep -E '(/nologin|/false)$' /etc/passwd | cut -d: -f1
```

### 2.1.9 Без пароля 🟢
```bash
sudo awk -F: '$2 == "" {print $1}' /etc/shadow
sudo awk -F: '$2 == "*" || $2 ~ /^!/ {print $1}' /etc/shadow
```

### 2.1.10 Хеші (замасковано) 🟢
```bash
sudo awk -F: 'BEGIN{OFS=":"} { if ($2 ~ /^\$/) $2="***MASKED***"; print }' /etc/shadow
```

### 2.1.11 Яка геш-функція 🟢
```bash
sudo cut -d: -f2 /etc/shadow | grep '^\$' | cut -d'$' -f2 | sort | uniq -c
grep ENCRYPT_METHOD /etc/login.defs
```
`$1$`=MD5, `$5$`=SHA-256, `$6$`=SHA-512, `$y$`=yescrypt, `$2b$`=bcrypt.

### 2.1.12 John the Ripper 🟡
```bash
sudo apt install john
sudo unshadow /etc/passwd /etc/shadow > ~/hashes.txt
chmod 600 ~/hashes.txt
john --format=crypt --wordlist=/usr/share/john/password.lst --rules ~/hashes.txt
john --show --format=crypt ~/hashes.txt
```
Тест: `echo '<USER>:<ПРОСТИЙ_ПАРОЛЬ>' | sudo chpasswd` 🔴
Після — видали `~/hashes.txt` і `~/.john/john.pot`.

Відновлення за маскою:
```bash
john --format=crypt --mask='?l?l?l?d?d' ~/hashes.txt
```
Ітерацій (гірший випадок) = добуток варіантів позицій: 26·26·26·10·10 = 1 757 600.

### 2.1.13 Політика паролів 🟢
```bash
grep -E '^(PASS_|LOGIN_RETRIES|ENCRYPT_METHOD|UMASK)' /etc/login.defs
grep -v '^#' /etc/pam.d/common-password
cat /etc/security/pwquality.conf 2>/dev/null
sudo chage -l <USER>
```

### *2.1.14 Інші файли з хешами 🟢
`/etc/gshadow`, `/etc/shadow-`, `/etc/passwd-`, `/var/backups/*.bak`, `/etc/security/opasswd`, `~/.john/john.pot`.
```bash
sudo grep -rlE '\$(y|6|5|1|2[aby])\$[^:]+' /etc /var/backups 2>/dev/null
```

---

## 2.2 Створення/зміна акаунтів 🔴
```bash
sudo groupadd <GROUP>
sudo groupadd <GROUP2>
sudo useradd -m -s /bin/bash <USER>
sudo useradd -m -s /bin/bash <USER2>
sudo passwd <USER>
sudo passwd <USER2>
sudo usermod -aG <GROUP> <USER>
sudo usermod -aG <GROUP>,<GROUP2> <USER2>
```
Перевірка:
```bash
grep -E '<USER>|<USER2>' /etc/passwd
sudo grep -E '<USER>|<USER2>' /etc/shadow
grep -E '<GROUP>|<GROUP2>' /etc/group
```
Прибрати:
```bash
sudo userdel -r <USER>
sudo userdel -r <USER2>
sudo groupdel <GROUP>
sudo groupdel <GROUP2>
```

## *2.3 PAM
Винесено в [6-pam.md](6-pam.md).
