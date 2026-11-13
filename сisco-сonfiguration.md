# 🌐 Cisco Network Configuration: VLAN, NAT & Routing

![Cisco](https://img.shields.io/badge/Vendor-Cisco-blue.svg)
![Networking](https://img.shields.io/badge/Topic-Networking-orange.svg)
![Type](https://img.shields.io/badge/Type-Cheat%20Sheet-green.svg)

> Настройка корпоративной сети на оборудовании Cisco (Packet Tracer / GNS3). Включает настройку VLAN, транковых портов, маршрутизации на палочке (Router-on-a-Stick), DHCP, ACL и трансляции адресов (SNAT/DNAT). Инструкцию писал давно, тут важны больше последовательность шагов и команды.

---

## 📑 Содержание
- [1. Настройка конечных устройств](#1-настройка-конечных-устройств)
- [2. Настройка коммутаторов уровня L2](#2-настройка-коммутаторов-уровня-l2)
- [3. Настройка центрального коммутатора (L3 Core)](#3-настройка-центрального-коммутатора-l3-core)
- [4. Настройка центрального роутера](#4-настройка-центрального-роутера)
- [5. Сводные таблицы конфигурации](#5-сводные-таблицы-конфигурации)

---

## 1. Настройка конечных устройств

##### Настройка ПК (получение IP по DHCP)
```bash
PC> ip dhcp
PC> show ip
PC> ping 8.8.8.8
```

##### Альтернатива: Настройка сервера (статический IP)
```bash
Server> ip 192.168.50.194 255.255.255.240 192.168.50.193  # IP, Маска, Шлюз
Server> ip dns 8.8.8.8
```

---

## 2. Настройка коммутаторов уровня L2

### Коммутатор отдела кадров (`sw-hr`)
```bash
sw-hr> enable
sw-hr# configure terminal
sw-hr(config)# hostname sw-hr
```

##### Создание VLAN
```bash
sw-hr(config)# vlan 10
sw-hr(config-vlan)# name HR
sw-hr(config-vlan)# exit
```

##### Настройка портов доступа (Access)
```bash
sw-hr(config)# interface range e0/1-3
sw-hr(config-if-range)# switchport mode access
sw-hr(config-if-range)# switchport access vlan 10
sw-hr(config-if-range)# description Users_HR
sw-hr(config-if-range)# exit
```

##### Настройка магистрального порта (Trunk)
```bash
sw-hr(config)# interface e0/0
sw-hr(config-if)# switchport trunk encapsulation dot1q
sw-hr(config-if)# switchport mode trunk
sw-hr(config-if)# switchport trunk allowed vlan 10
sw-hr(config-if)# description Trunk_to_Core
sw-hr(config-if)# end
```

##### Сохранение конфигурации
```bash
sw-hr# write memory
```

---

## 3. Настройка центрального коммутатора (L3 Core)

```bash
sw-core> enable
sw-core# configure terminal
```

##### Создание VLAN
```bash
sw-core(config)# vlan 10
sw-core(config-vlan)# name HR
sw-core(config)# vlan 20
sw-core(config-vlan)# name Finance
sw-core(config)# vlan 30
sw-core(config-vlan)# name R&D
sw-core(config)# vlan 40
sw-core(config-vlan)# name DataCenter
sw-core(config-vlan)# exit
```

##### Настройка Trunk-портов к нижестоящим коммутаторам
```bash
sw-core(config)# interface e0/1
sw-core(config-if)# switchport trunk encapsulation dot1q
sw-core(config-if)# switchport mode trunk
sw-core(config-if)# switchport trunk allowed vlan 10
sw-core(config-if)# description Link_to_HR
sw-core(config-if)# exit

sw-core(config)# interface e0/2
sw-core(config-if)# switchport trunk encapsulation dot1q
sw-core(config-if)# switchport mode trunk
sw-core(config-if)# switchport trunk allowed vlan 20
sw-core(config-if)# description Link_to_Finance
sw-core(config-if)# exit

sw-core(config)# interface e0/3
sw-core(config-if)# switchport trunk encapsulation dot1q
sw-core(config-if)# switchport mode trunk
sw-core(config-if)# switchport trunk allowed vlan 30
sw-core(config-if)# description Link_to_R&D
sw-core(config-if)# exit
```

##### Trunk к роутеру
```bash
sw-core(config)# interface e0/0
sw-core(config-if)# switchport trunk encapsulation dot1q
sw-core(config-if)# switchport mode trunk
sw-core(config-if)# switchport trunk allowed vlan 10,20,30,40
sw-core(config-if)# description Trunk_to_Router
sw-core(config-if)# exit
```

##### Порты для серверов (Access)
```bash
sw-core(config)# interface range e1/0-1
sw-core(config-if-range)# switchport mode access
sw-core(config-if-range)# switchport access vlan 40
sw-core(config-if-range)# description Server_Connection
sw-core(config-if-range)# end
```

##### Сохранение конфигурации
```bash
sw-core# write memory
```

---

## 4. Настройка центрального роутера

```bash
rou-core> enable
rou-core# configure terminal
```

### 4.1 Базовая настройка интерфейсов и субинтерфейсов

##### Внутренний интерфейс к коммутатору
```bash
rou-core(config)# interface ethernet 0/1
rou-core(config-if)# no shutdown
rou-core(config-if)# description Link_to_Switch_Core
rou-core(config-if)# exit
```

##### Субинтерфейс VLAN 10 (HR)
```bash
rou-core(config)# interface ethernet 0/1.10
rou-core(config-subif)# encapsulation dot1Q 10
rou-core(config-subif)# ip address 192.168.50.1 255.255.255.128
rou-core(config-subif)# description Subif_VLAN10_HR
rou-core(config-subif)# no shutdown
rou-core(config-subif)# exit
```

##### Субинтерфейс VLAN 20 (Finance)
```bash
rou-core(config)# interface ethernet 0/1.20
rou-core(config-subif)# encapsulation dot1Q 20
rou-core(config-subif)# ip address 192.168.50.129 255.255.255.224
rou-core(config-subif)# description Subif_VLAN20_Finance
rou-core(config-subif)# no shutdown
rou-core(config-subif)# exit
```

##### Субинтерфейс VLAN 30 (R&D)
```bash
rou-core(config)# interface ethernet 0/1.30
rou-core(config-subif)# encapsulation dot1Q 30
rou-core(config-subif)# ip address 192.168.50.161 255.255.255.224
rou-core(config-subif)# description Subif_VLAN30_RD
rou-core(config-subif)# no shutdown
rou-core(config-subif)# exit
```

##### Субинтерфейс VLAN 40 (DataCenter)
```bash
rou-core(config)# interface ethernet 0/1.40
rou-core(config-subif)# encapsulation dot1Q 40
rou-core(config-subif)# ip address 192.168.50.193 255.255.255.240
rou-core(config-subif)# description Subif_VLAN40_DataCenter
rou-core(config-subif)# no shutdown
rou-core(config-subif)# exit
```

##### Внешний интерфейс к провайдеру
```bash
rou-core(config)# interface ethernet 0/0
rou-core(config-if)# ip address 203.0.113.2 255.255.255.252
rou-core(config-if)# description Link_to_ISP
rou-core(config-if)# no shutdown
rou-core(config-if)# exit
```

### 4.2 Настройка маршрутизации

##### Default route к провайдеру
```bash
rou-core(config)# ip route 0.0.0.0 0.0.0.0 203.0.113.1
```

### 4.3 Настройка DHCP (пулы для VLAN)

##### Пул для отдела кадров (HR)
```bash
rou-core(config)# ip dhcp pool VLAN10_HR
rou-core(dhcp-config)# network 192.168.50.0 255.255.255.128
rou-core(dhcp-config)# default-router 192.168.50.1
rou-core(dhcp-config)# dns-server 8.8.8.8
rou-core(dhcp-config)# lease infinite
rou-core(dhcp-config)# exit
rou-core(config)# ip dhcp excluded-address 192.168.50.1
```

##### Пул для финансового отдела (Finance)
```bash
rou-core(config)# ip dhcp pool VLAN20_Finance
rou-core(dhcp-config)# network 192.168.50.128 255.255.255.224
rou-core(dhcp-config)# default-router 192.168.50.129
rou-core(dhcp-config)# dns-server 8.8.8.8
rou-core(dhcp-config)# lease infinite
rou-core(dhcp-config)# exit
rou-core(config)# ip dhcp excluded-address 192.168.50.129
```

##### Пул для отдела разработки (R&D)
```bash
rou-core(config)# ip dhcp pool VLAN30_RD
rou-core(dhcp-config)# network 192.168.50.160 255.255.255.224
rou-core(dhcp-config)# default-router 192.168.50.161
rou-core(dhcp-config)# dns-server 8.8.8.8
rou-core(dhcp-config)# lease infinite
rou-core(dhcp-config)# exit
rou-core(config)# ip dhcp excluded-address 192.168.50.161
```

### 4.4 Настройка ACL (ограничение доступа к VLAN 30)
> **Логика:** Разрешаем доступ в R&D только из HR, блокируем всем остальным (Finance, DataCenter), а весь остальной трафик (в интернет) пропускаем.

##### Создание расширенного ACL
```bash
rou-core(config)# ip access-list extended RD_ACCESS
# Правило 1: Разрешить из HR (VLAN 10) в R&D (VLAN 30)
rou-core(config-ext-nacl)# permit ip 192.168.50.0 0.0.0.127 192.168.50.160 0.0.0.31
# Правило 2: Запретить всем остальным входить в R&D
rou-core(config-ext-nacl)# deny ip any 192.168.50.160 0.0.0.31
# Правило 3: Разрешить весь остальной трафик
rou-core(config-ext-nacl)# permit ip any any
rou-core(config-ext-nacl)# exit
```

##### Применение ACL на интерфейс
```bash
rou-core(config)# interface ethernet 0/1.30
rou-core(config-subif)# ip access-group RD_ACCESS out
rou-core(config-subif)# exit
```
*Обратная маска (wildcard): `0.0.0.127` = `/25` (HR), `0.0.0.31` = `/27` (R&D).*

### 4.5 Настройка SNAT (Source NAT)

##### Указываем inside и outside интерфейсы
```bash
rou-core(config)# interface ethernet 0/0
rou-core(config-if)# ip nat outside
rou-core(config-if)# exit

rou-core(config)# interface ethernet 0/1.10
rou-core(config-subif)# ip nat inside
rou-core(config-subif)# exit
rou-core(config)# interface ethernet 0/1.20
rou-core(config-subif)# ip nat inside
rou-core(config-subif)# exit
rou-core(config)# interface ethernet 0/1.30
rou-core(config-subif)# ip nat inside
rou-core(config-subif)# exit
rou-core(config)# interface ethernet 0/1.40
rou-core(config-subif)# ip nat inside
rou-core(config-subif)# exit
```

##### Создаем ACL для NAT (какие сети пускать в интернет)
```bash
rou-core(config)# access-list 10 permit 192.168.50.0 0.0.0.127    # VLAN 10
rou-core(config)# access-list 10 permit 192.168.50.128 0.0.0.31   # VLAN 20
rou-core(config)# access-list 10 permit 192.168.50.160 0.0.0.31   # VLAN 30
rou-core(config)# access-list 10 permit 192.168.50.192 0.0.0.15   # VLAN 40
```

##### Включаем SNAT (overload = PAT)
```bash
rou-core(config)# ip nat inside source list 10 interface e0/0 overload
```

### 4.6 Настройка DNAT (Проброс порта)
Проброс 80 порта к внутреннему серверу `192.168.50.194`.

```bash
rou-core(config)# ip nat inside source static tcp 192.168.50.194 80 203.0.113.2 80 extendable
```

### 4.7 Команды для проверки

##### Проверка конфигурации
```bash
rou-core# show ip interface brief       # Статус интерфейсов
rou-core# show ip route                 # Таблица маршрутизации
rou-core# show ip dhcp binding          # Выданные DHCP-адреса
rou-core# show ip access-lists          # Правила ACL
rou-core# show ip nat translations      # Активные NAT-сессии
```

##### Тесты
```bash
PC> ping 8.8.8.8
Internet_Server> rlogin 203.0.113.2 80  # Проверка DNAT
```

---

## 5. Сводные таблицы конфигурации

### 5.1 VLAN и подсети
| VLAN ID | Название | Подсеть | Маска | CIDR | Шлюз (Gateway) |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **10** | HR | 192.168.50.0 | 255.255.255.128 | /25 | 192.168.50.1 |
| **20** | Finance | 192.168.50.128 | 255.255.255.224 | /27 | 192.168.50.129 |
| **30** | R&D | 192.168.50.160 | 255.255.255.224 | /27 | 192.168.50.161 |
| **40** | DataCenter | 192.168.50.192 | 255.255.255.240 | /28 | 192.168.50.193 |

### 5.2 Устройства L2 (Коммутаторы)
| Устройство | Роль | VLAN на access-портах | Uplink-порт | Trunk VLAN |
| :--- | :--- | :--- | :--- | :--- |
| `sw-hr` | Коммутатор HR | VLAN 10 (e0/1-3) | e0/0 | VLAN 10 |
| `sw-fin` | Коммутатор Finance | VLAN 20 (e0/1-3) | e0/0 | VLAN 20 |
| `sw-rd` | Коммутатор R&D | VLAN 30 (e0/1-3) | e0/0 | VLAN 30 |
| `sw-core` | Центральный L3 | VLAN 40 (e1/0-1) | e0/0 (к роутеру) | 10,20,30,40 |

### 5.3 Устройство L3 (Роутер)
| Интерфейс | Тип | VLAN | IP-адрес | Роль | Назначение |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `e0/1.10` | Субинтерфейс | 10 | 192.168.50.1/25 | `ip nat inside` | Шлюз HR |
| `e0/1.20` | Субинтерфейс | 20 | 192.168.50.129/27 | `ip nat inside` | Шлюз Finance |
| `e0/1.30` | Субинтерфейс | 30 | 192.168.50.161/27 | `ip nat inside` | Шлюз R&D |
| `e0/1.40` | Субинтерфейс | 40 | 192.168.50.193/28 | `ip nat inside` | Шлюз DataCenter |
| `e0/0` | Физический | - | 203.0.113.2/30 | `ip nat outside` | К провайдеру |

### 5.4 DHCP-пулы (на `rou-core`)
| Пул | VLAN | Диапазон | Шлюз | DNS | Исключённый адрес |
| :--- | :--- | :--- | :--- | :--- | :--- |
| VLAN10_HR | 10 | 192.168.50.0/25 | 192.168.50.1 | 8.8.8.8 | 192.168.50.1 |
| VLAN20_Finance | 20 | 192.168.50.128/27 | 192.168.50.129 | 8.8.8.8 | 192.168.50.129 |
| VLAN30_RD | 30 | 192.168.50.160/27 | 192.168.50.161 | 8.8.8.8 | 192.168.50.161 |

### 5.5 Сводная таблица (Где что настроено)
| Функция | Устройство | Интерфейс(ы) |
| :--- | :--- | :--- |
| **VLAN 10 (HR)** | `sw-hr`, `sw-core` | access: e0/1-3, trunk: e0/0 |
| **VLAN 20 (Finance)** | `sw-fin`, `sw-core` | access: e0/1-3, trunk: e0/0 |
| **VLAN 30 (R&D)** | `sw-rd`, `sw-core` | access: e0/1-3, trunk: e0/0 |
| **VLAN 40 (DataCenter)**| `sw-core` | access: e1/0-1 |
| **МежVLAN-маршрутизация** | `rou-core` | субинтерфейсы e0/1.10-40 |
| **Маршрут по умолчанию** | `rou-core` | e0/0 → 203.0.113.1 |
| **DHCP-сервер** | `rou-core` | глобально (пулы для VLAN 10,20,30) |
| **ACL (ограничение доступа)**| `rou-core` | e0/1.30 out → RD_ACCESS |
| **SNAT** | `rou-core` | inside: e0/1.10-40, outside: e0/0 |
| **DNAT** | `rou-core` | статический проброс 80 порта |
