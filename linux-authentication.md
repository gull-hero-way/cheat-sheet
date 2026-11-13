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

---

## 6. Где находится pam_pwquality.so

Модуль — часть пакета libpam-pwquality.

> `pam_pwquality.so` - это «движок», который умеет проверять. Он не знает, каким должен быть параметр. Его задача — выполнить проверку по заданным правилам. `/etc/security/pwquality.conf` - это файл с правилами (политика). Именно здесь написано, что «пароль должен быть не короче 8 символов» (minlen = 8) и «содержать минимум 3 класса символов» (minclass = 3).

В разных дистрибутивах Linux путь может немного отличаться, но обычно он лежит в директории с PAM-модулями:

В Debian/Ubuntu: 
```text
    /lib/x86_64-linux-gnu/security/pam_pwquality.so или /usr/lib/x86_64-linux-gnu/security/pam_pwquality.so.
```
В RHEL/Fedora/CentOS: 
```text
    /usr/lib64/security/pam_pwquality.so.
```

---

## 7. Взаимодействие /etc/pam.d/ и /etc/security/

Выполняют разные роли в одном механизме - `архитектуре PAM`:

`/etc/pam.d/` `(Сценарий проверки):`
Здесь лежат файлы для каждого сервиса (sshd, login, su, sudo, common-*). Эти файлы — сценарий того, какие шаги проверки нужно выполнить для этого сервиса. Например, в common-auth написано: «сначала проверь блокировку (pam_faillock), потом пароль по локальной базе (pam_unix), потом по LDAP (pam_sss)». Это порядок действий и список модулей.


`/etc/security/` `(Правила игры):` 
Здесь лежат конфигурационные файлы для этих модулей. Они определяют детали работы: faillock.conf говорит «блокировать после 5 попыток», pwquality.conf говорит «минимальная длина 8 символов», access.conf говорит «кому можно входить». Это параметры и правила.

---

## 8. Справка по файлам в `/etc/ssh/`, `/etc/pam.d/`, `/etc/security/`


### `/etc/ssh/`

*   **`sshd_config`**: Конфигурация **SSH-сервера**. Определяет, как сервер принимает входящие подключения (порт, разрешён ли вход root, методы аутентификации).
*   **`ssh_config`**: Конфигурация **SSH-клиента**. Определяет, как клиент подключается к другим серверам (алгоритмы шифрования, сжатие).


### `/etc/pam.d/` (Сценарии для сервисов)

Каждый файл здесь — это **сценарий**, какие проверки выполнить для конкретного сервиса. Файлы `common-*` — это общие сценарии, которые подключаются (`@include`) из других файлов.

*   **`common-auth`**: Сценарий **проверки пароля при входе**. Определяет, какие модули (например, `pam_unix.so`, `pam_sss.so`, `pam_faillock.so`) и в каком порядке должны проверить подлинность пользователя.
*   **`common-account`**: Сценарий **проверки допустимости входа**. Определяет, можно ли аккаунту входить в систему (не истёк ли срок действия пароля, не заблокирован ли он, разрешён ли вход по времени/группе через `pam_access.so`).
*   **`common-password`**: Сценарий **смены пароля**. Определяет, какие модули (например, `pam_pwquality.so`, `pam_pwhistory.so`) должны проверить новый пароль перед его установкой.
*   **`common-session`**: Сценарий **действий после входа**. Определяет, что нужно сделать после успешной аутентификации (создать домашнюю директорию, смонтировать файловые системы, установить переменные окружения).
*   **`sshd`**: Сценарий PAM специально для **SSH-сервера**. Обычно просто подключает `common-auth`, `common-account` и `common-session`.
*   **`su` / `su-l`**: Сценарий для команды **`su`** (переключение пользователя). Может содержать дополнительные проверки (например, через `pam_wheel.so`, разрешая `su` только членам группы `sudo`).
*   **`sudo`**: Сценарий для команды **`sudo`**. Подключает `common-auth` для повторной проверки пароля пользователя.
*   **`login`**: Сценарий для входа через **консоль** (tty).
*   **`cron`**: Сценарий для заданий **планировщика**.
*   **`other`**: Сценарий по умолчанию для сервисов, у которых нет собственного файла.


### `/etc/security/` (Правила для модулей PAM)

Каждый файл здесь — это **конфигурация** для конкретного модуля PAM, определяющая параметры его работы.

*   **`access.conf`**: Правила **доступа к системе**. Определяет, каким пользователям и группам разрешён или запрещён вход на хост. Используется модулем `pam_access.so`.
*   **`faillock.conf`**: Правила **блокировки после неудачных попыток входа**. Определяет, после скольких неудач блокировать учётную запись и на какое время. Используется модулем `pam_faillock.so`.
*   **`pwquality.conf`**: Правила **сложности пароля**. Определяет минимальную длину, требуемые классы символов и другие параметры. Используется модулем `pam_pwquality.so`.
*   **`pwhistory.conf`**: Правила **истории паролей**. Определяет, сколько последних паролей нужно помнить и запрещать к повторному использованию. Используется модулем `pam_pwhistory.so`.
*   **`limits.conf`**: Ограничения **системных ресурсов** для пользователей (например, максимальное количество процессов, размер файлов).
*   **`group.conf`**: Ограничения, применяемые к **группам** пользователей.
*   **`time.conf`**: Ограничения по **времени входа** для определённых сервисов.
*   **`pam_env.conf`**: Установка **переменных окружения** при входе пользователя.





