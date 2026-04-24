# How to get all prefixes from single IP address

## WHOIS the IP

Если известно имя домена, то сначала необходимо определить IP адрес

```
$ nslookup www.ozon.ru
Server:         x.x.x.x
Address:        x.x.x.x#53

Non-authoritative answer:
Name:   www.ozon.ru
Address: 185.73.194.82
Name:   www.ozon.ru
Address: 185.73.193.68
```

Номер автономной системы можно узнать из WHOIS ответа на запрос по IP адресу

```
$ whois 185.73.194.82
% This is the RIPE Database query service.
% The objects are in RPSL format.
%
% The RIPE Database is subject to Terms and Conditions.
% See https://docs.db.ripe.net/terms-conditions.html

% Note: this output has been filtered.
%       To receive output for a database update, use the "-B" flag.

% Information related to '185.73.194.0 - 185.73.194.255'

% Abuse contact for '185.73.194.0 - 185.73.194.255' is 'noc@ozon.ru'

inetnum:        185.73.194.0 - 185.73.194.255
netname:        OZONRU-DCZ1-NET3
descr:          Data-center Network in Zone1
country:        RU
admin-c:        ORMT1-RIPE
tech-c:         ORMT1-RIPE
status:         ASSIGNED PA
mnt-by:         OZONRU-MNT
mnt-lower:      OZONRU-MNT
mnt-routes:     OZONRU-MNT
created:        2014-10-21T06:04:30Z
last-modified:  2014-10-21T06:04:30Z
source:         RIPE

role:           OZON.RU RIPE MANAGEMENT TEAM
address:        Moscow, Presnenskaya embankment 10
abuse-mailbox:  noc@ozon.ru
org:            ORG-LIS21-RIPE
nic-hdl:        ORMT1-RIPE
mnt-by:         OZONRU-MNT
created:        2014-10-17T11:35:48Z
last-modified:  2020-07-28T11:01:21Z
source:         RIPE # Filtered

% Information related to '185.73.194.0/24AS44386'

route:          185.73.194.0/24
descr:          Data-center Network in Zone1
origin:         AS44386
mnt-by:         OZONRU-MNT
created:        2014-10-21T06:22:08Z
last-modified:  2014-10-21T06:22:08Z
source:         RIPE

% This query was served by the RIPE Database Query Service version 1.121.2 (DEXTER)
```
Ищем строку с префиксом `origin`. AS44386 - это тот самый номер автономной системы для адреса принадлежащего сети OZON.

> В выводе whois для IP не всегда есть объект `route` с полем `origin` (зависит от базы данных RIR). Если его нет, можно искать поле `aut-num` или `mnt-by`.

## Get IPv4 prefixes for AS (autonomous system)

Теперь по номеру автономной системы мы сможем узнать все анонсированные префиксы IPv4, для этого используем `curl` и `jq`

```
$ curl -s https://stat.ripe.net/data/announced-prefixes/data.json?resource=AS44386 | jq .
```
В результате выполнения `curl` будет выведен длинный текст, нам же нужно только одно свойство `.data.prefixes[].prefix`

```
curl -s https://stat.ripe.net/data/announced-prefixes/data.json?resource=AS44386 | jq -r '.data.prefixes[].prefix'
```

В результате выполнения этого запроса будут выведены только префиксы:

``` text
195.34.20.0/23
185.73.193.0/24
185.73.192.0/24
185.73.195.0/24
91.212.64.0/24
46.226.122.0/24
185.73.194.0/24
185.73.192.0/22
91.223.93.0/24
```

Теперь все эти префиксы можно добавить в таблицу маршрутизации, чтобы трафик в эти сети сразу заворачивать по кратчайшему маршруту.

В Windows это можно сделать при помощи команды `ROUTE ADD`

```
C:\Windows\System32> route add 195.34.20.0 MASK 255.255.254.0 192.168.1.1 IF 2
```
Эту команду следует повторить для каждого IPv4 префикса (подсети)

> Важно! Укажите свой адрес шлюза и индекс интефейса

Но лучше использовать `PowerShell` и команду `New-NetRoute`, которая понимает префиксы в CIDR-нотации, как их возвращает `curl`

```
# Пример для PowerShell
New-NetRoute -DestinationPrefix "195.34.20.0/23" -InterfaceIndex 2 -NextHop 192.168.1.1
```

Кстати, список префиксов можно скинуть ИИ-шке и попросить написать скрипт на вашем любимом языке

На `powershell` получится что-то типа:

```
$gw = "192.168.1.1"
$ifIndex = (Get-NetIPAddress -IPAddress "192.168.1.2").InterfaceIndex

$subnets = @(
    "195.34.20.0/23", "185.73.193.0/24", "185.73.192.0/24", "185.73.195.0/24", "91.212.64.0/24",
    "46.226.122.0/24", "185.73.194.0/24", "185.73.192.0/22", "91.223.93.0/24"
)

foreach ($subnet in $subnets) {
    New-NetRoute -DestinationPrefix $subnet -InterfaceIndex $ifIndex -NextHop $gw -RouteMetric 1 -ErrorAction SilentlyContinue
}
```

Нужно лишь правильно указать адрес маршрутизатора (в скрипте переменная `192.168.1.1`) и адрес сетевого интерфейса конмпьютера (в скрипте `192.168.1.2`)
