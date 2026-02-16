

**Настройка ASUS RT-N66U для локальной сети Windows (IPv6 6in4)**

При использовании туннеля 6in4 и оснасток Windows (MMC, Computer Management) часто возникают задержки из-за того, 
что роутер анонсирует себя как DNS-сервер, не умея разрешать локальные имена.

**1. Подготовка (JFFS)**

Убедитесь, что поддержка скриптов включена:

* **Administration** -> **System** -> **Enable JFFS custom scripts and configs** = Yes.

**2. Скрипт коррекции DNS (dnsmasq.postconf)**

Создайте файл /jffs/scripts/dnsmasq.postconf, чтобы принудительно заменить DNS роутера на ваши локальные контроллеры домена/серверы.


``` bash
*#!/bin/sh*
CONFIG=$1
*# Удаляем стандартные опции DNS роутера (::1) и поиска домена*
sed -i '/option6:23/d' $CONFIG
sed -i '/option6:24/d' $CONFIG
sed -i '/option6:dns-server/d' $CONFIG
sed -i '/ra-param/d' $CONFIG

*# Добавляем локальные IPv6 DNS серверы (замените адреса на свои)*
echo "dhcp-option=lan,option6:dns-server,[2001:470:xxxx::12],[2001:470:xxxx::10]" >> $CONFIG
echo "dhcp-option=lan,option6:24,your.domain.com" >> $CONFIG

*# Принудительно настраиваем Router Advertisement (RA) без само-анонса*
echo "ra-param=br0,10,600" >> $CONFIG
```

Use code with caution.

**3. Активация настроек**

После сохранения файла выполните в консоли:

```bash
chmod a+x /jffs/scripts/dnsmasq.postconf
service restart_dnsmasq
```

Use code with caution.

**4. Настройка WINS и NetBIOS**

Чтобы исключить задержки архаичных протоколов в Windows:

* **USB Application** -> **Network Place (Samba)** -> **Enable WINS Server** = No.
* На клиентах Windows: **Свойства IPv4** -> **Дополнительно** -> **WINS** -> **Отключить NetBIOS через TCP/IP**.

**Результат**

Windows получает через RA и DHCPv6 только валидные локальные DNS. Оснастка «Управление компьютером» подключается мгновенно, так как PTR-запросы (Reverse DNS) уходят сразу на сервер, а не "виснут" на роутере.

**Что еще стоит проверить?**

* **Обратные зоны (PTR)** на ваших DNS-серверах для подсети IPv6.
* **MTU туннеля** (обычно 1480 для 6in4), чтобы избежать фрагментации пакетов RPC.
* **Правила Firewall** на целевом сервере для входящего трафика по IPv6.
