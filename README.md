# Эмуляция виртуальной лаборатории VirtualBox в Cisco Packet Tracer

Данный репозиторий содержит конфигурации для эмуляции сетевой топологии виртуальной лаборатории (VirtualBox) в симуляторе Cisco Packet Tracer. 

Стенд демонстрирует каскадную маршрутизацию (тройной NAT), разделение на уровни доступа и работу DHCP-сервера.

## Архитектура сети

Трафик проходит следующую цепочку:
`Клиенты (Server-PT) -> DC01 (RRAS/NAT) -> Vbox_NAT (VirtualBox NAT) -> Dlink_my_router (Домашний роутер) -> Интернет`

## Адресация

| Сеть / Устройство | Интерфейс | IP-адрес | Маска | Шлюз |
| :--- | :--- | :--- | :--- | :--- |
| **WAN (Интернет)** | Gi0/0/1 (Dlink) | 79.1.1.2 | 255.255.255.0 | 79.1.1.1 |
| **Домашняя сеть** | Gi0/0/0 (Dlink) | 192.168.24.1 | 255.255.255.0 | - |
| | Gi0/0/1 (Vbox_NAT) | 192.168.24.50 | 255.255.255.0 | 192.168.24.1 |
| **VirtualBox NAT** | Gi0/0/0 (Vbox_NAT) | 10.0.2.2 | 255.255.255.0 | - |
| | Gi0/0/1 (DC01) | 10.0.2.15 | 255.255.255.0 | 10.0.2.2 |
| **LAN (Host-Only)** | Gi0/0/0 (DC01) | 192.168.200.10 | 255.255.255.0 | - |
| | VLAN 1 (HostOnly_Switch) | 192.168.200.1 | 255.255.255.0 | 192.168.200.10 |
| **DHCP Пул** | - | 192.168.200.30 - 254 | - | 192.168.200.10 |

---

## Конфигурации устройств

### 1. Dlink_my_router (ISR4331)
*Эмуляция реального домашнего роутера. Выпускает трафик в интернет (Cloud-PT).*

```cisco
enable
configure terminal
no ip domain-lookup

! WAN (в сторону Ufanet)
interface GigabitEthernet0/0/1
 ip address 79.1.1.2 255.255.255.0
 ip nat outside
 no shutdown
 exit

! LAN (в сторону Vbox_NAT)
interface GigabitEthernet0/0/0
 ip address 192.168.24.1 255.255.255.0
 ip nat inside
 no shutdown
 exit

! NAT (PAT)
access-list 1 permit 192.168.24.0 0.0.0.255
ip nat inside source list 1 interface GigabitEthernet0/0/1 overload

! Маршрут по умолчанию в сторону провайдера
ip route 0.0.0.0 0.0.0.0 79.1.1.1

end
write memory
```

### 2. Vbox_NAT (ISR4331)
*Эмуляция встроенного NAT VirtualBox. Транслирует трафик из 10.0.2.0/24 в домашнюю сеть.*

```cisco
enable
configure terminal
no ip domain-lookup

! WAN (в сторону Dlink)
interface GigabitEthernet0/0/1
 ip address 192.168.24.50 255.255.255.0
 ip nat outside
 no shutdown
 exit

! LAN (в сторону DC01)
interface GigabitEthernet0/0/0
 ip address 10.0.2.2 255.255.255.0
 ip nat inside
 no shutdown
 exit

! NAT (PAT)
access-list 1 permit 10.0.2.0 0.0.0.255
ip nat inside source list 1 interface GigabitEthernet0/0/1 overload

! Маршрут по умолчанию
ip route 0.0.0.0 0.0.0.0 192.168.24.1

end
write memory
```

### 3. DC01 (ISR4331)
*Эмуляция Windows Server (RRAS, DHCP, DNS). Шлюз для всех виртуальных машин.*

```cisco
enable
configure terminal
no ip domain-lookup

! WAN (в сторону Vbox_NAT)
interface GigabitEthernet0/0/1
 ip address 10.0.2.15 255.255.255.0
 ip nat outside
 no shutdown
 exit

! LAN (в сторону HostOnly_Switch)
interface GigabitEthernet0/0/0
 ip address 192.168.200.10 255.255.255.0
 ip nat inside
 no shutdown
 exit

! NAT (PAT)
access-list 1 permit 192.168.200.0 0.0.0.255
ip nat inside source list 1 interface GigabitEthernet0/0/1 overload

! Маршрут по умолчанию
ip route 0.0.0.0 0.0.0.0 10.0.2.2

! Настройка DHCP (исключаем адреса шлюза и статики)
ip dhcp excluded-address 192.168.200.1 192.168.200.29
ip dhcp pool LAN_CLIENTS
 network 192.168.200.0 255.255.255.0
 default-router 192.168.200.10
 dns-server 192.168.200.10
 exit

end
write memory
```

### 4. HostOnly_Switch (Multilayer Switch 3560)
*Эмуляция виртуального адаптера физического ПК. Работает как L2-коммутатор с IP-адресом.*

```cisco
enable
configure terminal

! Отключаем маршрутизацию (чтобы не мешал DC01)
no ip routing

interface vlan 1
 ip address 192.168.200.1 255.255.255.0
 no shutdown
 exit

! Шлюз для самого коммутатора
ip default-gateway 192.168.200.10

end
write memory
```

### 5. Нижние коммутаторы (Switch3, Switch4, Switch5)
*Устройства 2960-24TT. Настройка не требуется, работают как прозрачные L2-коммутаторы.*

---

## Инструкция по запуску и проверке
### Настройка клиентов (Server0 - Server5)
1. Кликните на любой `Server-PT` (например, `Server0`).
2. Перейдите в **Desktop** -> **IP Configuration**.
3. Выберите **DHCP**.
4. Устройство должно получить адрес `192.168.200.30`, маску `255.255.255.0`, шлюз `192.168.200.10`.

### Проверка связности (Ping)
С любого клиента (через Command Prompt) проверьте прохождение пакетов по всей цепочке:
```cmd
ping 192.168.200.10    # DC01 (шлюз)
ping 10.0.2.2          # Vbox_NAT
ping 192.168.24.1      # Dlink (Домашний роутер)
ping 79.1.1.2          # WAN (Интернет)
```
