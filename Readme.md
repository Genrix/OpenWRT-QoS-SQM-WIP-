# Advanced Layer Cake

Пресет для `sqm-scripts` в OpenWrt 25.12.x: настраиваемая классификация трафика на `nftables` с принудительной записью DSCP поверх CAKE.

Штатный `layer_cake` работает в режиме `diffserv` и доверяет меткам, которые уже стоят в пакетах. Advanced Layer Cake позволяет задавать собственные правила — по IP, подсети, диапазону адресов, протоколу, порту назначения и текущему DSCP — и применять одну и ту же логику к исходящему и входящему трафику.

## Возможности

- Классификация по src/dst IP, CIDR, диапазону адресов и спискам.
- Классификация по протоколу (TCP/UDP/ICMP) и портам, включая диапазоны.
- Принудительная перезапись DSCP — метки клиента не имеют приоритета.
- Классификация один раз на соединение: дальше класс восстанавливается по `ct mark`.
- Автоматический перенос класса на входящее направление через `act_ctinfo`.
- Конфигурация через обычный UCI-файл, до 15 классов.
- Прозрачная диагностика: счётчики `nftables` и `tc`-фильтров.

## Ограничения

| Ограничение | Пояснение |
| --- | --- |
| Только IPv4 | IPv6 не классифицируется и не обрабатывается |
| Только TCP, UDP, ICMP | Для ICMP поле `match_port` игнорируется |
| Максимум 15 классов | Метки `0x0101`–`0x010F`; лишние секции игнорируются с записью в лог |
| Нет независимой классификации ingress | Входящий трафик наследует класс соединения |
| DSCP всегда перезаписывается | При совпадении правила исходная метка пакета не учитывается |
| ECN обнуляется на ingress | `pedit` переписывает весь байт DSField, биты ECT/CE теряются |
| Один инстанс пресета | Все интерфейсы использовали бы один диапазон `ct mark`; для мультиинстанса нужны разные базовые смещения |
| Смена пресета | Для перехода на другой `.qos` рекомендуется перезагрузка роутера |

Пресет не увеличивает пропускную способность канала и не является файрволом — он управляет только очередями и приоритетами.  

## Совместимость с другими пакетами

Пресет занимает `ct mark` в диапазоне `0x0101`–`0x010F`, а на входящем направлении копирует его в `skb->mark` маской `0xffffffff`, полностью перезаписывая метку пакета.

| Пакет / механизм | В чём конфликт | Что делать |
| --- | --- | --- |
| mwan3 | Маска меток по умолчанию `0x3F00` накрывает бит `0x0100` из диапазона пресета | Изменить `mmx_mask` в mwan3 либо базовое смещение меток пресета |
| pbr, policy routing (`ip rule ... fwmark`) | Значения fwmark могут совпасть; на ingress маска `0xffffffff` затирает метки маршрутизации | Развести биты: пресету — младшие, маршрутизации — старшие |
| WireGuard `fwmark`, ручные `mark` в firewall | Совпадение значений и перезапись на ingress | Не использовать `0x0100`–`0x01FF` |

Проверка текущих потребителей меток:

```sh
ip rule show
uci show mwan3 2>/dev/null | grep -i mask
nft list ruleset | grep -i 'mark'
```

Для типового домашнего роутера с одним WAN и без mwan3/pbr конфликтов нет — раздел актуален только приPolicy routing или мультиWAN.

## Как это работает

Классификация выполняется **один раз на исходящем направлении**, а на входящем класс восстанавливается из состояния соединения. Отдельные правила для ingress не нужны и для уже установленных соединений были бы ненадёжны.

### Исходящее направление (LAN → WAN)

| Шаг | Где | Что происходит |
| --- | --- | --- |
| 1 | `nftables`, hook `postrouting`, `oifname <WAN>` | Пакет попадает в таблицу `alc_<iface>` |
| 2 | Правила восстановления | `ct mark ≠ 0` → записать DSCP класса, обработка прекращается |
| 3 | Правила классификации | `ct mark = 0` → первое совпавшее правило задаёт `ct mark` и DSCP |
| 4 | `tc` egress на WAN | CAKE `diffserv` раскладывает пакеты по tins |

### Входящее направление (WAN → LAN)

| Шаг | Где | Что происходит |
| --- | --- | --- |
| 1 | `tc` ingress на WAN, `prio 1` | `action ctinfo cpmark 0xffffffff continue` восстанавливает `ct mark` в `skb->mark` |
| 2 | `prio 2`, по фильтру на класс | match по `mark` → `pedit` пишет DSCP → `mirred` на `ifb4<iface>` |
| 3 | `prio 3` | Неклассифицированный трафик уходит на IFB без изменения DSCP |
| 4 | `tc` egress на IFB | Входящий CAKE `diffserv` |

> **Почему не `nftables prerouting`.** В ядре `tc ingress` отрабатывает раньше `prerouting`: пакет уходит на IFB через `mirred` до того, как nftables успел бы изменить DSCP. Поэтому восстановление метки на входе делается только средствами `tc` — `act_ctinfo` + `pedit`.

### Метки классов

Каждый класс получает уникальный `ct mark` по формуле `0x0100 + N`, где `N` — порядковый номер класса (1…15): класс 1 → `0x0101`, класс 2 → `0x0102`, …, класс 15 → `0x010F`. Метка живёт всё время жизни соединения, поэтому класс стабилен, а нагрузка на правила минимальна.

### Имя таблицы nftables

Таблица называется `alc_<iface>`, несовместимые с именем символы заменяются на `_`: `eth0` → `alc_eth0`, `eth0.2` → `alc_eth0_2`, `br-lan` → `alc_br_lan`.

## Установка

### Зависимости

```sh
apk update
apk add sqm-scripts luci-app-sqm kmod-sched-cake kmod-ifb kmod-sched-ctinfo kmod-sched nftables tc-full
```

В сборках на `opkg` используйте `opkg update` / `opkg install` с теми же именами пакетов. Имена `tc`/`ip` в разных сборках отличаются (`tc-full`/`tc-tiny`, `ip-full`/`ip-tiny`) — достаточно варианта, который тянет `sqm-scripts`. Действия `tc` иногда разнесены по отдельным пакетам: при ошибке `Failed to load TC action module`, связанной с `pedit`, доустановите `kmod-sched` (или `kmod-sched-act-pedit`, если он есть в вашей версии).

### Файлы

| Файл | Назначение |
| --- | --- |
| `/usr/lib/sqm/Advanced_Layer_Cake.qos` | Исполняемый пресет |
| `/usr/lib/sqm/Advanced_Layer_Cake.qos.help` | Короткая справка в LuCI |
| `/etc/config/advanced_layer_cake` | Конфигурация классов |

```sh
cp Advanced_Layer_Cake.qos Advanced_Layer_Cake.qos.help /usr/lib/sqm/
cp advanced_layer_cake /etc/config/

chmod 755 /usr/lib/sqm/Advanced_Layer_Cake.qos
chmod 644 /usr/lib/sqm/Advanced_Layer_Cake.qos.help
chmod 644 /etc/config/advanced_layer_cake
```

### Включение

1. **Network → SQM QoS → Queue setup script** → `Advanced_Layer_Cake.qos`.
2. Укажите WAN-интерфейс (`eth0`, `eth1`, `pppoe-wan`, `eth0.2` и т. п.).
3. Задайте честную полосу — обычно **90–95 %** от реально измеренной скорости для download и upload. При завышенных значениях очередь не формируется на роутере и шейпер не работает.
4. **Save & Apply**.

## Конфигурация

Файл: `/etc/config/advanced_layer_cake`. Каждый класс — секция `config class 'ClassName'`.

Правила работают как сито: **сверху вниз, первое совпадение побеждает**, дальнейшие правила для пакета не проверяются. Поэтому узкие правила ставятся выше широких, а `Unknown` — последним.

### Параметры класса

| Параметр | Значение | Назначение |
| --- | --- | --- |
| `match_dscp` | `0`–`63` или `''` | Совпадение по текущему DSCP |
| `match_src_ip` | IP / CIDR / диапазон / список / `''` | Совпадение по адресу источника |
| `match_dst_ip` | IP / CIDR / диапазон / список / `''` | Совпадение по адресу назначения |
| `match_protocol` | `TCP`, `UDP`, `ICMP`, список / `''` | Совпадение по протоколу |
| `match_port` | порт / диапазон / список / `''` | Совпадение по порту назначения |
| `set_dscp` | `0`–`63` | Обязательный: какой DSCP записать |

Пустое значение `''` означает «не проверять это поле». Класс без условий ловит весь трафик.

### Форматы значений

| Поле | Допустимые формы | Пример |
| --- | --- | --- |
| `match_port` | порт, список, диапазон, комбинация | `'53, 67, 8000-9000, 9100'` |
| `match_src_ip`, `match_dst_ip` | адрес, подсеть, диапазон, список | `'192.168.1.0/24'`, `'192.168.1.10-192.168.1.50'` |
| `match_protocol` | один протокол или список | `'TCP, UDP'` |
| `match_dscp` | число 0–63 | `'46'` |

### Рекомендуемый порядок классов

1. Конкретные хосты, подсети, игровые серверы.
2. Трафик с уже корректными метками (EF, CS5, CS1).
3. Служебные сервисы: DNS, DHCP, NTP.
4. VoIP и видеозвонки.
5. Видео и стриминг.
6. Широкие правила: протоколы, диапазоны портов, браузерный трафик.
7. `Unknown` — catch-all с `set_dscp '0'`.

Если catch-all класса нет, скрипт добавит неявный `Unknown` с DSCP 0 — при условии, что лимит в 15 классов не исчерпан.

### Пример конфигурации

```text
# /etc/config/advanced_layer_cake
# Sieve principle: top-to-bottom, first match wins. '' = do not check the field.

config class 'Downloads'
    option match_dscp      '8'
    option match_protocol  ''
    option match_port      ''
    option set_dscp        '8'

config class 'Games'
    option match_dscp      '46'
    option set_dscp        '46'

config class 'IPCall'
    option match_dscp      '40'
    option match_protocol  'UDP'
    option set_dscp        '40'

config class 'Network'
    option match_port      '53, 67, 68, 123, 853, 5222, 5228, 5353'
    option set_dscp        '46'

config class 'IPTV'
    option match_dst_ip    '10.10.1.20'
    option set_dscp        '34'

config class 'Stream'
    option match_protocol  'TCP'
    option match_port      '1935'
    option set_dscp        '34'

config class 'Browser'
    option match_protocol  'TCP'
    option match_port      '80, 443, 8443'
    option set_dscp        '20'

config class 'Unknown'
    option set_dscp        '0'
```

Пропущенные параметры эквивалентны `''` — поле не проверяется.

### Ошибки в конфигурации

- Некорректная секция пропускается, в лог пишется причина, остальные классы применяются.
- Если корректных классов нет, пресет переходит в упрощённый режим, близкий к обычному `layer_cake`.

## DSCP и CAKE

Пресет сам запускает CAKE в режиме **`diffserv4`** — дописывать что-либо в настройках SQM не требуется. Четыре тина нужны потому, что пример конфигурации использует все четыре категории: без Video отдельного тина для IPTV/стриминга просто не существует.

| Класс примера | DSCP | Tin в `diffserv4` |
| --- | --- | --- |
| Downloads | 8 (CS1) | Bulk |
| Unknown | 0 (BE) | Best Effort |
| Browser | 20 (AF22) | Best Effort |
| IPTV, Stream | 34 (AF41) | Video |
| IPCall | 40 (CS5) | Voice |
| Games, Network | 46 (EF) | Voice |

Голый CAKE без параметров работает в `diffserv3` (Bulk / Best Effort / Voice) — там DSCP 34 обрабатывается как обычный трафик. Поэтому режим в пресете задан явно, а не полагается на значение по умолчанию.

Режим можно переопределить через **Advanced option string** в LuCI (`eqdisc_opts` / `iqdisc_opts` в `/etc/config/sqm`): эти параметры добавляются после параметров пресета, и в CAKE побеждает последнее значение. Проверка фактического режима — `tc qdisc show dev <iface>`.

## Примеры правил

```text
# Конкретный игровой сервер
config class 'GameServer'
    option match_dst_ip    '203.0.113.50'
    option match_protocol  'UDP'
    option set_dscp        '46'

# Устройство в фон
config class 'NasBackground'
    option match_src_ip    '192.168.1.50'
    option set_dscp        '8'

# DNS и NTP наверх
config class 'NetServices'
    option match_port      '53, 123'
    option set_dscp        '46'

# Стриминговая подсеть
config class 'StreamingHost'
    option match_dst_ip    '198.51.100.0/24'
    option set_dscp        '34'
```

## Диагностика

| Задача | Команда |
| --- | --- |
| Лог пресета | `logread \| grep ALC` |
| CAKE на WAN | `tc qdisc show dev eth0` |
| CAKE на IFB | `tc qdisc show dev ifb4eth0` |
| Ingress-фильтры со статистикой | `tc -s filter show dev eth0 parent ffff:` |
| Счётчики правил классификации | `nft -s list chain ip alc_eth0 egress_classify` |
| Таблица целиком | `nft list table ip alc_eth0` |

Успешный запуск выглядит так:

```text
ALC: Class 1 'Downloads': mark=0x0101 set_dscp=8
ALC: Class 2 'Games': mark=0x0102 set_dscp=46
...
ALC: Advanced Layer Cake applied: iface=eth0 ifb=ifb4eth0 table=alc_eth0 classes=8
```

Ожидаемая картина: на обоих интерфейсах `cake` с `diffserv`; на ingress WAN — фильтр `prio 1` с `ctinfo`, фильтры `prio 2` по числу классов и дефолтный `prio 3`.

Быстрая проверка на живом трафике: `ping 8.8.8.8`, затем посмотреть счётчики цепочки `egress_classify` и статистику ingress-фильтров — прирост должен быть у правила ожидаемого класса и у соответствующего фильтра.

## Устранение неполадок

| Симптом | Причина | Решение |
| --- | --- | --- |
| `Failed to load TC action module` | Нет модуля нужного `tc action` (обычно `pedit`) | `kmod-sched`, при наличии — `kmod-sched-act-pedit` |
| `act_ctinfo kernel module missing` | Нет модуля `ctinfo` | `kmod-sched-ctinfo` |
| `invalid match_port` | Синтаксис: `'80,,443'`, `'abc'`, `'9000-8000'` | Привести к формату `'80, 443, 8443'` |
| `invalid match_src_ip` / `invalid match_dst_ip` | Неверный формат или IPv6 | Только IPv4: адрес, CIDR, диапазон, список |
| Правила есть, но счётчики нулевые | Порядок правил, не тот интерфейс в SQM, несовпадение `oifname`, выше лежит более общее правило | Сверить интерфейс и порядок, поднять узкое правило выше |
| Классы после 15-го не работают | Исчерпан диапазон меток | Сократить число классов, объединить правила списками |
| Пресет не переключается на другой `.qos` | Известное ограничение SQM | Перезагрузить роутер |

---

### Идея на будущее: убрать ограничение с ECN.

`pedit` переписывает весь байт DSField, поэтому ECT/CE теряются. `act_ctinfo` умеет восстанавливать DSCP **масочной** записью, сохраняя ECN, если хранить DSCP в старших битах connmark:

- на egress правило класса пишет константу `0x01000000 | (DSCP << 18) | class_id` (например, EF: `0x01B80101`);
- на ingress вместо `pedit`:

```sh
tc filter add dev eth0 parent ffff: protocol ip prio 1 u32 match u32 0 0 \
    action ctinfo dscpmask 0xfc000000 statemask 0x01000000 \
    action mirred egress redirect dev ifb4eth0
```

Бонусом уходят per-class ingress-фильтры и зависимость от `pedit`, а младшие биты остаются свободны для чужих меток — то есть частично решается и пункт 3.
