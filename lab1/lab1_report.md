University: [ITMO University](https://itmo.ru/ru/)

Faculty: [FICT](https://fict.itmo.ru)

Course: [Introduction in routing](https://github.com/itmo-ict-faculty/introduction-in-routing)

Year: 2022/2023

Group: K33212

Author: Glebov Ivan Igorevich

Lab: Lab1

Date of create: 03.10.2022

Date of finished: 06.10.2022

# Стартовый конфиг

Стартовый конфиг представлен в файле topology.clab.yml. Файл представляет собой топологию сети и описание каждой ноды с адресом в управляющей сети для подключения.

## Разворачивание

Первое разворачивание группы контейнеров производится командой `clab deploy` из дериктории рядом с файлом topology.clab.yml.

Если есть необходимость перезапустить лабораторную работу со сбросом всех устройств и сети, то используется команда `clab deploy --reconfigure`.

# Конфигурация устройств

## Настройка главного маршрутизатора

Настройку удобнее всего начинать с верхней точки дерева сети и спускаться к листьям.

На этом устройстве надо настроить 2 dhcp сервера на две подсети и настроить маршрутизацию трафика между vlan.

### Настройка vlan

#### Добавление vlan

`interface vlan add interface=ether2 name=vlan20 vlan-id=20`

- interface - название интерфейса, который будет находиться во vlan
- name - название vlan
- vlan-id - номерной идентификатор vlan (если внутри устройства будет два vlan с одним id, но разным name, то трафик внутри них будет восприниматься, как трафик из одного vlan.

Важно отметить - на один физический интерфейс можно повесить сколько угодно vlan. Данный механизм используется для передачи по одному физическому каналу трафика с несколькими тегами vlan.

*Часть конфигурации для настройки vlan*

```
/interface vlan
add interface=ether2 name=vlan10 vlan-id=10
add interface=ether2 name=vlan20 vlan-id=20

/ip address
add address=192.168.10.1/24 interface=vlan10 network=192.168.10.0
add address=192.168.20.1/24 interface=vlan20 network=192.168.20.0
```

### Настройка dhcp

Для создания и включения dhcp сервера необходимо выполнить следующие шаги:
- Добавить пул адресов
- Добавить dhcp-server network
- Создать dhcp-server на нужный интерфейс
- Включить dhcp-server

#### Добавление пула адресов

`ip pool add name=poolName ranges=192.168.10.10-192.168.10.250`

- poolName - название пула адресов
- ranges - диапазон адресов, которые выделяются сервером

#### Добавление dhcp-server network

`ip dhcp-server network add address=192.168.10.0/24 gateway=192.168.10.1`

- address - адрес сети с укороченной маской
- gateway - шлюз по умолчанию для этой сети

#### Создание dhcp-server и его включение

`ip dhcp-server add name=dhcpServername address-pool=poolName interface=vlan10 disabled=no`

- name - название сервера
- address-pool - название пула адресов, которые будут на этом сервере
- interface - имя интерфейса на который будет работать сервер
- disabled - yes/no. Со старта сервер выключен (disabled=yes). Для автоматического включения сервера надо использовать параметр 'no'

*Часть конфигурации для настройки dhcp*

```
/ip pool
add name=pool1 ranges=192.168.10.10-192.168.10.250
add name=pool2 ranges=192.168.20.10-192.168.20.250
/ip dhcp-server
add address-pool=pool1 disabled=no interface=vlan10 name=server1
add address-pool=pool2 disabled=no interface=vlan20 name=server2
/ip dhcp-server network
add address=192.168.10.0/24 gateway=192.168.10.1
add address=192.168.20.0/24 gateway=192.168.20.1
```

### Полная конфигурация главного машррутизатора

```
/interface vlan
add interface=ether2 name=vlan10 vlan-id=10
add interface=ether2 name=vlan20 vlan-id=20
/interface wireless security-profiles
set [ find default=yes ] supplicant-identity=MikroTik
/ip pool
add name=pool1 ranges=192.168.10.10-192.168.10.250
add name=pool2 ranges=192.168.20.10-192.168.20.250
/ip dhcp-server
add address-pool=pool1 disabled=no interface=vlan10 name=server1
add address-pool=pool2 disabled=no interface=vlan20 name=server2
/ip address
add address=172.31.255.30/30 interface=ether1 network=172.31.255.28
add address=192.168.10.1/24 interface=vlan10 network=192.168.10.0
add address=192.168.20.1/24 interface=vlan20 network=192.168.20.0
/ip dhcp-client
add disabled=no interface=ether1
/ip dhcp-server network
add address=192.168.10.0/24 gateway=192.168.10.1
add address=192.168.20.0/24 gateway=192.168.20.1
/system identity
set name=R01.TEST
```

## Настройка центрального коммутатора

Данный коммутатор имеет 3 линка - к основному маршрутизатору и по одному линку к соседним коммутаторам.

На этом устройстве будет самый длинный конфиг, но после осознания он не воспринимается очень сложным.

Идея конфигурации этого устройства в том, что в него приходит трафик тегированный двумя разными vlan и перенаправляется к маршрутизатору через один линк. Для реализации этого надо будет создать два интерфейса bridge внутри коммутатора.

### Создание vlan и их настройка

Для коммутации трафика внутри коммутатора надо создать 4-е интерфейса vlan. Они должны парами иметь одинаковые vlan-id.
Пусть это будут vlan10 и vlan103 для vlan-id=10 и vlan20 и vlan203 для vlan-id=20.

Каждый vlan надо повесить на какой-то интерфейс - vlan10 и vlan20 подключим к ether2, который соединён с основным маршрутизатором.
vlan103 подключим к ether3, который идёт к левому коммутатору, а vlan203 к ether4, которые подключен к правому коммутатору.

Физические интерфейсы и их vlan'ы:
- ether2: vlan10, vlan20
- ether3: vlan103
- ether4: vlan203

Для каждой пар vlan надо создать bridge - виртуальный неуправляемый(в данном случае) коммутатор.

### Создание bridge и подключение к ним нужных интерфейсов

Пусть это будут bridge10 и bridge20 соответственно.

Команда для создания bridge: `itnerface bridge add name=bridge10`

После создания двух bridge к ним надо подключить необходимые интерфейсы.

Команда для подключения интерфейса в bridge: `interface bridge port add interface=vlan10 bridge=bridge10`

- interface - название подключаемого интерфейса
- bridge - название bridge к которому подключается интерфейс

Такми образом мы подключаем к bridge10 vlan10 и vlan103, а к bridge20 vlan20 и vlan203.

Bridge и их vlan'ы:
- bridge10: vlan10, vlan103
- bridge20: vlan20, vlan203

### Настройка dhcp-client

После добавления vlan'ов в bridge должна быть уже связная сеть и мы сразу можем настроить автоматическое получение адресов на наши bridge используя dhcp-server'а на основном маршрутизаторе.

Для этого необходимо использовать следующуюю команду: `ip dhcp-client add interface=bridge10 disabled=no`
- interface - название интерфейса, который будет получать адрес от dhcp-server
- disabled - yes/no. (Со старта состояние `disabled=yes`.) Чтобы сразу запустить dhcp-client надо указать флаг `disabled=no`

Подобная конфигурация повторяется для bridge20. По итогу мы должны получить адреса из указанных ранее на основном маршрутизаторе пулов.

*Для получения информации об адресах внутри устройства можно использовать команду `ip address print`. Она выведет список всех адресов для этого устройства.*


### Полная конфигурация центрального коммутатора

```
/interface bridge
add name=bridge
add name=bridge10
add name=bridge20
/interface vlan
add interface=ether2 name=vlan10 vlan-id=10
add interface=ether2 name=vlan20 vlan-id=20
add interface=ether3 name=vlan103 vlan-id=10
add interface=ether4 name=vlan203 vlan-id=20
/interface wireless security-profiles
set [ find default=yes ] supplicant-identity=MikroTik
/interface bridge port
add bridge=bridge interface=ether2
add bridge=bridge10 interface=vlan10
add bridge=bridge10 interface=vlan103
add bridge=bridge20 interface=vlan20
add bridge=bridge20 interface=vlan203
/ip address
add address=172.31.255.30/30 interface=ether1 network=172.31.255.28
/ip dhcp-client
add disabled=no interface=ether1
add disabled=no interface=bridge10
add disabled=no interface=bridge20
/system identity
set name=SW01.01.TEST
```

## Конфигурация левого коммутатора

Конфигурация этого устройства будет самой простой - в данном случае надо создать vlan, bridge и dhcp-client на bridge.

### Создание vlan и настройка

Так как в конфигурации центрального коммутатора порт ether3 соединён с vlan, который имеет vlan-id=10, то и здесь он должен быть таким.

`interface add vlan name=vlan10 vlan-id=10 interface=ether2`

### Создание bridge и настройка

Далее необходимо создать неуправляемый (в данной конфигурации) коммутатор вида bridge.
Используемая команда: `interface bridge add name=bridge10`.

И подключить к нему все необходимые интерфейсы. Подключение интерфейсов производится командой: `interface bridge port add interface=ether3 bridge=bridge10`.

*Как это работает? Весь трафик внутри bridge идёт без меток vlan, но когда он отправляется через ehter3, который помечен vlan10, то он будет автоматически получать соответствующую метку vlan.*

### Полная конфигурация левого коммутатора

```
/interface bridge
add name=bridge10
/interface vlan
add interface=ether2 name=vlan10 vlan-id=10
/interface wireless security-profiles
set [ find default=yes ] supplicant-identity=MikroTik
/interface bridge port
add bridge=bridge10 interface=vlan10
add bridge=bridge10 interface=ether3
/ip address
add address=172.31.255.30/30 interface=ether1 network=172.31.255.28
/ip dhcp-client
add disabled=no interface=ether1
add disabled=no interface=bridge10
/system identity
set name=SW02.01.TEST
```

## Конфигурация правого коммутатора

Конфигурация этого устройства идентична предыдущему за исключением vlan-id и названий соответствующих интерфейсов и подробное её описание будет опущено за ненадобностью.

### Полная конфигурация правого коммутатора

```
/interface bridge
add name=bridge20
/interface vlan
add interface=ether2 name=vlan20 vlan-id=20
/interface wireless security-profiles
set [ find default=yes ] supplicant-identity=MikroTik
/interface bridge port
add bridge=bridge20 interface=vlan20
add bridge=bridge20 interface=ether3
/ip address
add address=172.31.255.30/30 interface=ether1 network=172.31.255.28
/ip dhcp-client
add disabled=no interface=ether1
add disabled=no interface=bridge20
/system identity
set name=SW02.02.TEST
```