

# 🐧 Samba + LDAP: интеграция и разграничение доступа

![Samba](https://img.shields.io/badge/Software-Samba-blue.svg)
![LDAP](https://img.shields.io/badge/Protocol-LDAP-orange.svg)
![Security](https://img.shields.io/badge/Topic-Access%20Control-red.svg)

Эта шпаргалка описывает два архитектурных подхода к связке Samba и OpenLDAP, разбирает их "подводные камни", а также показывает, какие атрибуты и схемы обязательно должны быть в каталоге, чтобы всё заработало.

---

## 🗺️ Главная схема: Как Samba видит пользователей

Когда Samba пытается проверить, можно ли пользователю зайти в папку, она действует так:

```text
Пользователь -> Samba -> NSS (getent/id) -> SSSD -> LDAP
```
*Samba — это обычное Linux-приложение. Она не ходит в LDAP напрямую (если не настроен `ldapsam`). Она спрашивает у системы: «В каких группах состоит этот юзер?», а система через **NSS** и **SSSD** достает это из LDAP.*

---

## ⚖️ Два пути настройки: Выбор архитектуры

Существует два принципиально разных способа подружить Samba с LDAP.

### Путь 1: NSS/SSSD + `tdbsam` (Лабораторный / Простой)
Samba использует локальную базу паролей, а группы и юзеров берет из системы (через SSSD).

*   **Плюсы:** Быстро настраивается, работает для простых файловых серверов. Не нужно расширять схему LDAP под Samba.
*   **Минусы:** Пароль пользователя хранится дважды (в LDAP и локально в Samba). Samba не умеет сама менять пароли в LDAP.

### Путь 2: Прямой `ldapsam` (Production / Правильный)
Samba напрямую общается с LDAP и хранит там свои Samba-специфичные атрибуты (пароли, SID'ы).

*   **Плюсы:** Единая точка входа. Samba может менять пароли в LDAP и управлять машинными учетками.
*   **Минусы:** Сложно настроить. Требует расширения схемы LDAP (добавления атрибутов `sambaSID`, `sambaNTPassword`). Требует прав на запись в LDAP.

---

## 🛠️ Путь 1: NSS/SSSD + `tdbsam` (Пошаговая схема)

### 1. Подготовка LDAP (Схема `rfc2307` + `posixGroup`)
Чтобы Linux (и Samba) увидели группы, они должны быть типа `posixGroup` и иметь числовой GID. Группы типа `groupOfNames` (где участники — это DN) для NSS бесполезны.

**Обязательные поля в LDAP для группы:**
```ldif
dn: cn=grp-files-rw,ou=Groups,dc=example,dc=com
objectClass: top
objectClass: posixGroup
cn: grp-files-rw
gidNumber: 8001               # Обязательно! Без GID NSS не отдаст группу
memberUid: user1              # Просто логины, а не полные DN
memberUid: user2
```

**Обязательные поля в LDAP для пользователя:**
```ldif
dn: cn=Иван Иванов,ou=Users,dc=example,dc=com
objectClass: top
objectClass: posixAccount
cn: Иван Иванов
uid: user1
uidNumber: 1001
gidNumber: 8001               # Первичная группа (должна существовать в LDAP!)
homeDirectory: /home/user1
```

### 2. Настройка SSSD
Файл `/etc/sssd/sssd.conf` должен быть строго настроен на нужную схему.
```ini
[domain/LDAP]
id_provider = ldap
ldap_uri = ldap://ldap-server:389
ldap_search_base = dc=example,dc=com
ldap_default_bind_dn = cn=admin,dc=example,dc=com
ldap_default_authtok = PASSWORD

# ⚠️ КРИТИЧЕСКИ ВАЖНО:
ldap_schema = rfc2307
ldap_group_object_class = posixGroup
ldap_group_member = memberUid
```
*После правки: `pkill -9 sssd` и запуск заново (в Docker без systemd: `docker exec -d srv_fs /usr/sbin/sssd -D`).*

### 3. Настройка Samba (`smb.conf`)
```ini
[global]
   workgroup = WORKGROUP
   server role = standalone server
   security = user
   passdb backend = tdbsam
   map to guest = never

[shared_folder]
   path = /share/folder
   valid users = @grp-files-rw   # Samba спросит у NSS, есть ли юзер в этой группе
   
   # ⚠️ КОСТЫЛЬ (для лабораторных):
   # Если Samba не может построить токен доступа (UID/GID не совпадают), 
   # используется force user. Файлы пишутся от рута, но аудит (%U) покажет реального юзера.
   force user = root 
```

### 4. Добавление пользователей в Samba
Так как мы используем `tdbsam`, Samba ничего не знает о паролях в LDAP. Нужно создать локального Samba-пользователя (с тем же паролем):
```bash
smbpasswd -a user1
```
*Теперь Samba проверит пароль по `passdb.tdb`, а доступ к папке — через группу `@grp-files-rw` (через SSSD).*

---

## 🏢 Путь 2: Прямой `ldapsam` (Для Production)

Если вы хотите, чтобы Samba сама управляла паролями в LDAP, без SSSD.

### 1. Расширение LDAP-схемы
Вам понадобятся Samba-атрибуты. Создается объект домена:
```ldif
dn: sambaDomainName=WORKGROUP,dc=example,dc=com
objectClass: sambaDomain
objectClass: top
sambaDomainName: WORKGROUP
sambaSID: S-1-5-21-XYZ...  # SID из `net getlocalsid`
```

Пользователю добавляются классы и атрибуты:
```ldif
dn: cn=Иван Иванов,ou=Users,dc=example,dc=com
changetype: modify
add: objectClass
objectClass: sambaSamAccount
-
add: sambaSID
sambaSID: S-1-5-21-XYZ-1001
-
add: sambaNTPassword
sambaNTPassword: 8846F7EA... # Хэш пароля
```

Группе добавляются маппинги:
```ldif
dn: cn=grp-files-rw,ou=Groups,dc=example,dc=com
changetype: modify
add: objectClass
objectClass: sambaGroupMapping
-
add: sambaSID
sambaSID: S-1-5-21-XYZ-8001
-
add: sambaGroupType
sambaGroupType: 2
```

### 2. Настройка Samba
```ini
[global]
   workgroup = WORKGROUP
   security = user
   passdb backend = ldapsam:ldap://ldap-server:389
   ldap admin dn = cn=admin,dc=example,dc=com
   ldap suffix = dc=example,dc=com
   ldap group suffix = ou=Groups
   ldap user suffix = ou=Users

   # ⚠️ Опция "trusted" требует, чтобы у ВСЕХ групп (включая Domain Users) 
   # были прописаны sambaSID. Иначе Samba упадет с ошибкой NT_STATUS_INTERNAL_DB_CORRUPTION
   ldapsam:trusted = yes
```

---

## 🕵️ Полезные команды для отладки

Если что-то не работает, проверяем по цепочке:

| Что проверяем | Команда | Ожидаемый результат |
| :--- | :--- | :--- |
| **NSS видит группу?** | `getent group grp-files-rw` | `grp-files-rw:*:8001:user1,user2` |
| **SSSD видит юзера и его группы?** | `id user1` | `uid=1001(user1) gid=8001(grp-files-rw) groups=8001(...)` |
| **Самба видит юзера?** | `pdbedit -L` | Список локальных юзеров Samba |
| **Самба видит группы LDAP?** | `net groupmap list` | (В `ldapsam` покажет группы) |
| **Тест доступа** | `smbclient //localhost/share -U user1` | Подключение и команда `ls` |

---

## 🚨 Архитектурные грабли (Топ ошибок)

1. **Смешивание `groupOfNames` и `posixGroup`** 
   Если SSSD настроен на `rfc2307bis` (группы `groupOfNames`), он ожидает атрибут `member` (с DN). Если вы создали `posixGroup` с `memberUid`, SSSD их просто не увидит, и `getent` вернет пустоту. **Выбирайте что-то одно.**
2. **Отсутствие `gidNumber`** 
   Если у группы нет числового `gidNumber`, NSS не отдаст её системе. Samba не сможет проверить `valid users = @group`.
3. **Миф о `tdbsam` + PAM** 
   Многие думают, что `passdb backend = tdbsam` будет проверять пароли через SSSD/PAM. **Это не так.** `tdbsam` использует свой файл `/var/lib/samba/private/passdb.tdb`. Чтобы пароль работал, юзера надо добавить через `smbpasswd -a`.
4. **Права 777** 
   Если в `smb.conf` нет нормального разграничения на уровне ФС, а папки открыты на 777, любой локальный пользователь (или скомпрометированный процесс) прочитает файлы минуя Samba. Для production нужны ACL (`setfacl`) или правильный маппинг UID/GID.

---

## 📝 Бонус: Аудит действий (full_audit)

Чтобы видеть, кто и что делал в папках:

**В `smb.conf`:**
```ini
[shared_folder]
   vfs objects = full_audit
   full_audit:prefix = %U|%I|%S  # %U - реальный юзер (не root!)
   full_audit:success = all
   full_audit:facility = local5
   full_audit:priority = notice
```
**В `/etc/rsyslog.d/49-samba-audit.conf`:**
```text
local5.*    /var/log/samba/audit.log
& stop
```
*Не забудь создать файл лога и дать права: `touch /var/log/samba/audit.log && chown syslog:adm ...`*.
