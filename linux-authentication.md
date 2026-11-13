# Механика аутентификации в Linux: SSH, PAM, LDAP и Sudo

Подробная схема того, как происходит вход пользователя на сервер, смена пароля и блокировка учетной записи в домене. Материал полезен для понимания того, как взаимодействуют между собой SSH, модули PAM, локальные файлы конфигурации и внешняя директория LDAP.

---

## 1. Пользователь стучится на сервер (Вход по SSH)

```text
1. Пользователь → SSH-соединение на сервер (порт 22)
   ↓
2. /etc/ssh/sshd_config
   "вообще можно ли root по SSH?" → PermitRootLogin no
   "на каком порту слушать?" → Port 22
   "какие методы аутентификации?" → PasswordAuthentication, PubkeyAuthentication
   ↓
3. /etc/pam.d/sshd  (сценарий PAM для SSH-сервиса)
   Подключает через @include:
   - common-auth      (проверка пароля)
   - common-account   (можно ли аккаунту входить?)
   - common-session   (что делать после входа)
   ↓
4. /etc/pam.d/common-account
   "этому пользователю с этого IP можно?" → pam_access.so
        ↓
        /etc/security/access.conf
        +:root:LOCAL
        +:local-admin:LOCAL
        +:admins-group:ALL
        -:ALL:ALL
   ↓
5. /etc/pam.d/common-auth
   "как проверить пароль?" — по порядку:
   ├─ pam_faillock.so preauth    → не заблокирован ли уже?
   │       ↓
   │       /etc/security/faillock.conf
   │       deny = 5, unlock_time = 900
   │
   ├─ pam_unix.so                → локальный пароль (из /etc/shadow)
   ├─ pam_sss.so                 → пароль в LDAP (через SSSD)
   │       ↓
   │       /etc/sssd/sssd.conf
   │       ldap_uri = ldap://ldap-server:389
   │       ldap_default_bind_dn = cn=ldap-reader,...
   │
   └─ pam_faillock.so authfail / authsucc
           "неудача → увеличить счётчик" / "успех → сбросить счётчик"
                ↓
                /var/lib/faillock/  (файлы счётчиков)
   ↓
6. Пароль верный → пользователь вошёл → создаётся сессия
   ↓
7. /etc/pam.d/common-session
   "что сделать после входа?"
   - создать домашнюю директорию
   - смонтировать /home (если нужно)
   - установить переменные окружения
   - запустить скрипты из /etc/profile.d/
   ↓
8. Пользователь выполняет sudo <команда>
   ↓
9. /etc/pam.d/sudo
   "как проверить, можно ли sudo?"
   Подключает @include common-auth (снова проверка пароля!)
   ↓
10. /etc/sudoers  +  /etc/sudoers.d/ldap-sudo
    "а можно ли ему sudo и на что?"
    %admins-group ALL=(ALL:ALL) ALL
    local-admin ALL=(ALL) /usr/bin/systemctl, /usr/bin/passwd, ...
    ↓
    Проверка: входит ли пользователь в группу %admins-group?
    Откуда берётся группа?
        ↓
        LDAP: cn=admins-group,ou=Groups,dc=example,dc=com
        (через SSSD + NSS: /etc/nsswitch.conf → group: files sss)
   ↓
11. Разрешено → команда выполняется от root
```

---

## 2. Отдельная ветка: Смена пароля

```text
1. Пользователь выполняет passwd (или меняет пароль через GUI/SSH)
   ↓
2. /etc/pam.d/passwd  →  @include common-password
   ↓
3. /etc/pam.d/common-password
   "как проверить новый пароль?" — по порядку:
   ├─ pam_pwquality.so retry=3 enforce_for_root
   │       ↓
   │       /etc/security/pwquality.conf
   │       minlen = 8, minclass = 3, maxrepeat = 3, usercheck = 1
   │
   ├─ pam_pwhistory.so use_authtok
   │       ↓
   │       /etc/security/pwhistory.conf
   │       remember = 3, enforce_for_root
   │
   ├─ pam_unix.so   → сохранить локальный хеш в /etc/shadow
   │       ↓
   │       /etc/login.defs
   │       PASS_MAX_DAYS 180, PASS_MIN_DAYS 0,
   │       PASS_WARN_AGE 7, ENCRYPT_METHOD SHA512
   │
   └─ pam_sss.so use_authtok   → отправить новый пароль в LDAP
           ↓
           LDAP: ldap-server (ou=Users,dc=example,dc=com)
           тут срабатывает ppolicy:
               ↓
               cn=default / pwdPolicyUsers / pwdPolicyService
               pwdMinLength: 8, pwdInHistory: 3,
               pwdMaxAge: 15552000, pwdCheckModule: check_password.so
   ↓
4. Все проверки пройдены → пароль изменён (и локально, и в LDAP)
```

---

## 3. Отдельная ветка: Блокировка учётки в LDAP (ppolicy)

Это происходит **не на хосте**, а **внутри ldap-server**, когда любой хост пытается проверить пароль доменной учётки:

```text
1. Пользователь вводит пароль на workstation
   ↓
2. workstation: /etc/pam.d/common-auth → pam_sss.so
   "спроси у LDAP, верный ли пароль?"
   ↓
3. По сети: ldap://ldap-server:389
   ↓
4. ldap-server: OpenLDAP + overlay ppolicy
   "проверяю пароль + смотрю счётчик неудач"
        ↓
        /opt/ldap-policy/*.ldif  (шаблоны, применены через start.sh)
        ↓
        База mdb (реальные политики):
        - cn=default,ou=Policies
        - cn=pwdPolicyUsers,ou=Policies
        - cn=pwdPolicyService,ou=Policies
        ↓
   Если пароль неверный:
        pwdFailureCount++ (в атрибуте pwdFailureTime учётки)
   Если pwdFailureCount >= pwdMaxFailure (5):
        учётка блокируется на pwdLockoutDuration (900 сек)
   ↓
5. Ответ в workstation: "OK" или "заблокирован"
```

---

## 4. Ключевые файлы и их роли (сводка)

| Путь | Что определяет |
| :--- | :--- |
| `/etc/ssh/ssh_config` | Конфигурация клиента (программы ssh). Она определяет, как клиент подключается к другим серверам: какие алгоритмы шифрования предпочитать, использовать ли сжатие, с какими настройками по умолчанию идти на определенные хосты. Этот файл на безопасность самого сервера никак не влияет.|
| `/etc/ssh/sshd_config` | Конфигурация сервера (демона sshd). Он определяет, как сервер принимает входящие подключения: на каком порту слушать, можно ли входить под root, какие методы аутентификации разрешены, какие логи писать.) |
| `/etc/pam.d/sshd` | Сценарий PAM для SSH (подключает common-*) |
| `/etc/pam.d/common-account` | Проверка: можно ли аккаунту входить (`pam_access`) |
| `/etc/security/access.conf` | Белый список «кто откуда может входить» |
| `/etc/pam.d/common-auth` | Проверка пароля при входе (`pam_faillock`, `pam_unix`, `pam_sss`) |
| `/etc/security/faillock.conf` | Параметры блокировки (5 попыток, 900 сек) |
| `/var/lib/faillock/` | Файлы счётчиков неудачных попыток (по пользователям) |
| `/etc/pam.d/common-session` | Действия после входа (домашняя папка, окружение) |
| `/etc/pam.d/sudo` | Сценарий PAM для `sudo` (переиспользует common-auth) |
| `/etc/sudoers` + `/etc/sudoers.d/ldap-sudo` | Кто и что может через `sudo` |
| `/etc/sssd/sssd.conf` | Связь с LDAP (URI, bind DN, схема) |
| `/etc/nsswitch.conf` | Откуда брать пользователей/группы (`files sss`) |
| `/etc/pam.d/passwd` | Сценарий PAM для смены пароля |
| `/etc/pam.d/common-password` | Смена пароля (`pam_pwquality`, `pam_pwhistory`, `pam_sss`) |
| `/etc/security/pwquality.conf` | Требования к сложности пароля |
| `/etc/security/pwhistory.conf` | История паролей |
| `/etc/login.defs` | Сроки действия пароля, метод шифрования |
| `/opt/ldap-policy/*.ldif` | Шаблоны LDAP-политик (применяет `start.sh`) |
| База `mdb` (LDAP) | Рабочие политики `ppolicy` (`cn=default`, `pwdPolicyUsers`, `pwdPolicyService`) |
| `/usr/bin/start.sh` | Что происходит при старте контейнера `ldap-server` |

---

## 5. Как запомнить порядок (мнемоника)

**Вход по SSH:**
> **ssh** → **sshd_config** → **PAM (account → auth → session)** → **sudo**

**Смена пароля:**
> **passwd** → **common-password** → **pwquality → pwhistory → SSSD → LDAP**

**Блокировка в LDAP:**
> **любой хост** → **pam_sss** → **LDAP** → **ppolicy** → **счётчик + lockout**


















