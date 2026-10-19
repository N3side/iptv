# Алгоритм диагностики проблем с IPTV (телевизор не работает / с прерываниями)

Ниже — пошаговый алгоритм для оборудования **Zyxel 1000, Zyxel 3000, Huawei 5600, Zyxel 3528** на основе приведённой документации. Двигайтесь сверху вниз: сначала общие проверки, затем по вендору.

---

## Этап 0. Первичная локализация

1. **Уточнить характер проблемы:**
   - нет картинки совсем (чёрный экран / «нет сигнала»);
   - картинка есть, но рассыпается (артефакты, квадраты);
   - периодические замирания / прерывания;
   - не переключаются каналы;
   - работает только часть каналов.
2. **Уточнить масштаб:** один абонент / подъезд / дом / весь узел. Если массово — смотреть uplink и multicast-VLAN, а не порт абонента.
3. **Проверить, есть ли интернет у абонента** (PPPoE поднят?). Если интернета нет вообще — сначала лечить линию/сессию, IPTV вторично.

---

## Этап 1. Проверка линии ADSL/VDSL (физика)

На DSLAM (Zyxel 1000 / 3000 / Huawei 5600) для порта абонента:

| Что смотреть | Zyxel 1000 | Zyxel 3000 | Huawei 5600 |
|---|---|---|---|
| Состояние линии | `stasctics adsl show 41` (V — линк есть) | `Lcman command 7 show port 41` | `display interface adsl 0/4/5` |
| Параметры линии | `Statictics adsl linerate 41` | `lcman command 7 linerate 41` | `display line operation port 0/1/17` |
| Затухание | до 20 dB — отлично, от 50 dB — потеря синхронизации | то же | до 20 dB — отлично, от 60 dB — работа невозможна |

**Критерии:**
- Линк down / «activating» → проблема физики (кабель, коннектор, сплиттер, модем). IPTV не заработает.
- `Attainable rate` близка к текущей `Downstream rate` → линия на пределе, будут прерывания. Занизить профиль скорости.
- Высокое затухание (близко к порогу) → жалобы на рассыпание картинки.
- `Target margin` низкий / шум высокий → нестабильность, прерывания.

**Действия:**
- Перезагрузить порт: Zyxel 1000 — `adsl disable 41` / `adsl enable 41`; Huawei — `deactive 47` / `activate 47 profile-index 78`.
- При нестабильности — сменить профиль на более консервативный (меньше max rate, выше target margin, `interleave` вместо `fast` для IPTV).
- Проверить кабель/сплиттер на стороне абонента.

---

## Этап 2. Проверка PVC и VLAN для IPTV

### Zyxel 1000
1. `adsl pvc show 41` — убедиться, что PVC для IPTV есть: `vpi 1 / vci 32` с флагом **super**.
2. Если нет — создать:
   ```
   adsl pvc set 41 1 32 super SAGEM_IPTV
   ```
3. `switch vlan portshow 41` — проверить **pvid**: для super channel должен быть VLAN IPTV (например, 1210).
4. При необходимости сменить: `switch vlan pvid 41 1219`.
5. `switch igmpsnoop mvlan show *` — порт должен быть **зафиксирован (F)** на multicast-VLAN:
   ```
   switch igmpsnoop mvlan set 700 41:FU MULTICAST
   ```
6. `switch pktfilter show 41` — фильтр не должен блокировать трафик (`none`).

### Zyxel 3000
1. `lcman port pvc show 7-41` — наличие PVC `0/33` с `super`.
2. Создание: `lcman port pvc set 7-41-0/33 vid=2148 super llc vcprofile=default priority=4`.
3. `lcman port pvid show 7` → `lcman port pvid set 7-41 2148` (VLAN IPTV).

### Huawei 5600
1. `service port vlan 2341 adsl 0/7/41 vpi 1 vci 32 rx-cctr 100 tx-cttr 100` — проверить, что PVC для IPTV создан с нужным VLAN.
2. Настройка BTV:
   ```
   config
   btv
   igmp user add port 0/7/41 adsl 1 32 no-auth max program 1
   ```
3. `display igmp user online slot 0/7` — есть ли online-подписка.

---

## Этап 3. Проверка multicast / IGMP (основная причина «нет картинки» и «часть каналов»)

### Zyxel 1000
- `statistics igmpsnoop group` — список всех multicast-групп.
- `statistics igmpsnoop port group 41` — подписан ли порт на нужные группы.
- `switch igmpsnoop mvlan show *` — порт зафиксирован на mvlan (см. Этап 2).
- `statistics vlan 2316` — статус VLAN (`F` — работает, `U`/`T`/`E` — нет).

### Zyxel 3528 (агрегация)
- `show mvr` — настройки MVR.
- `show multicast` — подписки.
- `show igmp-snooping group all` — активные группы.
- Проверить, что порт в нужном VLAN: `show vlan 2361`, `show interface config 47` (PVID).

### Huawei 5600
- `display igmp user online slot 0/7` — online-подписки.
- `display vlan all` — наличие IPTV VLAN.

**Типовые причины:**
- Порт не зафиксирован (F) на multicast-VLAN → группы не приходят.
- Неверный PVID → трафик уходит в интернет-VLAN, а не в IPTV.
- IGMP snooping не видит запросов от модема (проблема на стороне абонента/модема).
- Multicast-VLAN не протянут через агрегацию (Zyxel 3528) — смотреть `show vlan`, `show running-config interface port-channel 47`.

---

## Этап 4. Проверка трафика и MAC/DHCP

### Zyxel 1000
- `statistics port 41 0 33` — трафик именно на IPTV-PVC (tx/rx rate). Если tx есть, а rx нет — проблема на стороне абонента/модема.
- `statistics mac 41` — виден ли MAC модема/приставки.
- `statistics dhcp snoop 41` — какой IP и на каком VLAN получила приставка. **Важно:** список хранится сутки и не обновляется при падении линка — очистить `switch dhcp snoop flush 41` и перезагрузить порт, чтобы увидеть свежий IP.
- `switch pktfilter show 41` — фильтр не должен резать трафик.

### Zyxel 3528
- `show mac address-table port 4` / `show mac address-table vlan 2361` — виден ли MAC приставки в IPTV VLAN.
- `show dhcp snooping binding` — привязки IP.
- `show interface 47` — link up, скорости tx/rx.

### Huawei 5600
- `display mac-address adsl 0/7/41` (или `port`, если `adsl` не работает).
- `display traffic 0/7/41` — счётчики.
- `display location xxxx-xxxx-xxxx` — поиск порта по MAC приставки.

**Признак:** если MAC приставки не виден в IPTV VLAN — проблема в PVC/PVID/фильтре (Этап 2). Если виден, но трафика нет — multicast/IGMP (Этап 3).

---

## Этап 5. Проверка агрегации и uplink (Zyxel 3528)

1. `show interface 47` — link up, нет ошибок, скорость.
2. `cable-diagnostics 47` — диагностика кабеля до DSLAM/абонента.
3. `show vlan` / `show running-config interface port-channel 47` — порт в нужных VLAN, PVID корректен, `untagged`/`tagged` выставлены правильно (абонентские — untagged, аплинки — tagged).
4. `show lldp info remote` — виден ли соседний коммутатор.
5. `show logging` — ошибки линка, flap, STP.
6. Если массовые жалобы — проверить загрузку uplink и multicast-VLAN на всём пути.

---

## Этап 6. Проверка PPPoE Intermediate Agent (если приставка не получает IP / не авторизуется)

На Zyxel 3528:
- `pppoe intermediate-agent` включён;
- `pppoe intermediate-agent format-type identifier-string string xxx-FTTH-a.b.c.d option spv delimiter /` — корректный префикс города и IP коммутатора;
- `pppoe intermediate-agent vlan yyy` + `circuit-id` / `remote-id` для нужного VLAN;
- на абонентских портах `pppoe intermediate-agent trust`.

Если формат/префикс неверный — приставка может не получать сессию, хотя линк есть.

---

## Этап 7. Энергосбережение (частые «прерывания»)

На Zyxel 1000:
- `adsl pmm show 41` — если энергосбережение включено, при простое линк может «засыпать» и давать рывки/подвисания.
- `adsl pmm disable 41` — отключить для IPTV-портов.

---

## Этап 8. Логи и финальная проверка

- Zyxel 1000: `alarm history show`, `statistics port 41`.
- Zyxel 3528: `show logging`, `show system information`.
- Huawei: `display board 0`, `display traffic table from-index 0` (ATM-профиль).
- Диагностика PVC:
  - Zyxel 1000: `adsl loopback 41 f5 0 33`;
  - Huawei: `atm ping x 0 33`.

После изменений:
- Zyxel 1000: `config save`;
- Zyxel 3528: `write memory 1`;
- Huawei: сохранить конфиг.

---

## Краткая блок-схема (что проверять в первую очередь)

```
Жалоба на IPTV
   │
   ├─ Линк DSL down/activating? ──► Этап 1 (физика, профиль, перезагрузка порта)
   │
   ├─ Линк есть, но нет картинки:
   │     ├─ PVC super для IPTV есть? ──► Этап 2 (создать/проверить PVC, PVID, super)
   │     ├─ Порт зафиксирован на mvlan (F)? ──► Этап 3 (igmp snoop, mvlan)
   │     ├─ MAC приставки виден в IPTV VLAN? ──► Этап 4 (MAC/DHCP/фильтр)
   │     └─ Трафик на PVC идёт? ──► Этап 4 (statistics port … 0 33)
   │
   ├─ Картинка рассыпается/прерывается:
   │     ├─ Затухание/шум/attainable rate ──► Этап 1 (занизить профиль, interleave)
   │     ├─ Энергосбережение включено? ──► Этап 7 (adsl pmm disable)
   │     └─ Ошибки/флапы на агрегации ──► Этап 5 (cable-diagnostics, show logging)
   │
   └─ Часть каналов не работает ──► Этап 3 (конкретные multicast-группы, mvlan, IGMP)
```

**Правило приоритета:** сначала физика линии (Этап 1), затем PVC/VLAN (Этап 2), затем multicast (Этап 3) — это покрывает ~90% случаев «не работает телевизор» и «прерывания» на данном оборудовании.
