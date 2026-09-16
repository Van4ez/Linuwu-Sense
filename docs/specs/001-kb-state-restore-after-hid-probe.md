---
status: done
date: 2026-09-16
repo: github/Linuwu-Sense, branch anv16s41-hid
---

# Подсветка клавиатуры ANV16S-41 не восстанавливается после ребута

## Зачем

После перезагрузки подсветка не возвращается к тому, что было выставлено до
неё. Причина в порядке инициализации драйвера, не в железе и не в обновлении
пакетов.

Факты, снятые 2026-09-16 (ядро 7.0.0-31, DKMS linuwu-sense/1.2.anv16s41):

1. `four_zone_kb_state_load()` вызывается из `acer_platform_probe()`, а
   `hid_register_driver(&acer_hid_rgb_driver)` идёт после него в
   `acer_wmi_init()`. В журнале: `KB states restored successfully` в
   21:41:22.940561, `Acer HID RGB keyboard controller found` в 21:41:22.941573.
   В момент восстановления `acer_hid_rgb_dev == NULL`, HID-ветка в
   `set_per_zone_color()` пропускается, команда уходит в WMI, который на этой
   модели ничего не делает.
2. Доказательство: WMI-ветка меняет входной буфер на месте
   (`*zones[i] = ... | zone_ids[i]`). Файл `/etc/four_zone_kb_state` хранит
   `ffffff` для всех зон, а `per_zone_mode` в sysfs после загрузки показывает
   `ffffff01,ffffff02,ffffff04,ffffff08,37`. Кэш испорчен именно этой веткой.
3. Отдельно: последняя загрузка перед этой (12.09 04:02 - 16.09 21:40)
   закончилась без штатного shutdown. В журнале нет `Stopping linuwu_sense`
   и `kb states saved`. Файл состояния датирован 12.09 04:01. Сохранение
   происходит только в `rmmod` при выключении, поэтому жёсткий ребут теряет
   всё, что было выставлено после последнего чистого выключения.

## Что делаем

В scope:

- Драйвер: на моделях с `nitro_hid_kb` не выполнять восстановление в
  `four_zone_kb_state_load()`, пока HID-устройство не найдено. Ставить флаг
  `pending`, а в `acer_hid_rgb_probe()` после `acer_hid_rgb_dev = hdev`
  применять кэш (`set_per_zone_color` либо `set_kb_status` по `per_zone`).
- Драйвер: после каждой успешной записи в `four_zone_mode` и
  `per_zone_mode` вызывать `four_zone_kb_state_save()`, чтобы жёсткий
  ребут не терял настройки. Сохранение в rmmod остаётся.
- Пересобрать DKMS для 7.0.0-31, перезагрузить модуль, проверить порядок
  сообщений в журнале и цвет клавиатуры.

Вне scope:

- Камера и микрофон: не изменились, `lsusb` без `0408:*`, `/dev/video*` нет,
  запись с `hw:2,0` и `hw:2,2` даёт 0 ненулевых сэмплов из 288000. Ждём
  ответов по багрепортам, здесь ничего не делаем.

## Неясности

1. Добавлять ли сохранение состояния при каждой записи в sysfs, чтобы жёсткий
   ребут не терял настройки? Ответ: да, сохранять при каждой записи в
   sysfs (`four_zone_mode`, `per_zone_mode`) тоже.
2. Коммитить ли изменение в ветку `anv16s41-hid` и обновлять ли PR #127 в
   upstream? Ответ: пока нет, только локальная сборка. Коммит и PR
   отдельной задачей после проверки на ребуте.
3. Комментарии в коде: в клоне английские, в DKMS-копии русские, логика
   совпадает. Правим клон и копируем в `/usr/src`? Ответ: да. После
   копирования DKMS-копия станет английской, как клон.

## План

1. Правка `src/linuwu_sense.c` в клоне: флаг `kb_state_pending`, ранний
   выход из `four_zone_kb_state_load()` на HID-модели без устройства,
   применение кэша в `acer_hid_rgb_probe()`.
   Плюс вызов `four_zone_kb_state_save()` в конце обоих `_store()` после
   успешного применения.
   Проверка: `make KVER=$(uname -r)` в клоне собирается без предупреждений.
2. Новый каталог `/usr/src/linuwu-sense-1.3.anv16s41/` с `src/`, `Makefile`
   и `dkms.conf` из клона, версия в `dkms.conf` `1.3.anv16s41`. Каталог 1.2
   не трогаем, он нужен для отката. Затем `dkms remove linuwu-sense/1.2.anv16s41
   --all` и `dkms add` + `dkms install linuwu-sense/1.3.anv16s41`.
   Проверка: `dkms status` показывает 1.3 для 7.0.0-31 и 7.0.0-30, 1.2 нет.
3. `rmmod linuwu_sense` (сохранит текущее состояние) и `modprobe
   linuwu_sense`.
   Проверка: в `journalctl -k` строка `restored` идёт после `controller
   found`, `per_zone_mode` показывает чистые `ffffff` без бита зоны,
   клавиатура светится как до rmmod.
4. Выставить заметный цвет через `kb-backlight`, штатно перезагрузиться.
   Проверка: после загрузки цвет тот же без действий пользователя.
5. Выставить другой цвет, проверить mtime `/etc/four_zone_kb_state`
   и сообщение `kb states saved` в журнале без rmmod.
   Проверка: mtime обновился в момент записи.

## Проверка

Снаружи: после штатного ребута клавиатура горит тем цветом и яркостью, что
были до него. Журнал: `Acer HID RGB keyboard controller found` раньше
`KB states restored`.

## Откат

```
sudo dkms remove linuwu-sense/1.3.anv16s41 --all
sudo dkms add linuwu-sense/1.2.anv16s41       # исходники 1.2 остаются в /usr/src
sudo dkms install linuwu-sense/1.2.anv16s41
sudo rmmod linuwu_sense && sudo modprobe linuwu_sense
```

Клон: `git checkout -- src/linuwu_sense.c`.

## Результат

- Шаг 1, 2026-09-16 21:54: патч в клоне, `make KVER=7.0.0-31-generic` без
  ошибок. Пришлось перенести объявление `struct kb_state` выше
  `set_kb_status()` и добавить forward declaration для `_save()`.
- Шаг 2, 21:56: `dkms status` показывает `linuwu-sense/1.3.anv16s41`
  installed для 7.0.0-30 и 7.0.0-31, 1.2 удалён из DKMS, `/usr/src/...1.2`
  на месте. Модуль подписан MOK.
- Шаг 3, 21:58:22: rmmod/modprobe. Журнал: `KB state restore deferred`,
  затем `Acer HID RGB keyboard controller found`, затем `KB states restored
  successfully`. `per_zone_mode` после загрузки `ffffff,ffffff,ffffff,ffffff,37`
  без бита зоны. Клавиатура белая, как в файле состояния.
- Шаг 4/5, 22:00:08: **Oops ядра** при `kb-backlight all FF 60 00 static 60`.
  Трасса: `per_zoned_rgb_kb_store` → `four_zone_kb_state_save` →
  `kernel_write` → `rw_verify_area`, NULL deref по адресу 0x63. RSI =
  `-13` (EACCES): `filp_open` из контекста sysfs-записи идёт с правами
  пользователя, `/etc/four_zone_kb_state` root:root 0644, вернулся
  `ERR_PTR`, а в `_save()` проверка `if(!file)` вместо `IS_ERR()`. Это
  старая ошибка upstream, до сих пор не срабатывала, потому что `_save()`
  звали только из rmmod под root. Цвет применился (HID-запись прошла до
  сохранения), файл состояния не обновился, процесс python3 убит с
  выключенными прерываниями, звук встал на 1-2 с.
  Исправление: сохранение через `schedule_work()` (worker имеет права
  ядра, sysfs-путь не пишет на диск), `IS_ERR()` в `_save()`,
  `cancel_work_sync()` перед финальным сохранением в remove.
- 22:02-22:03: патч с `schedule_work()` внесён в клон, скопирован в
  `/usr/src/...1.3`, DKMS пересобран (ko 22:03:34, srcversion
  8FF57B68D5655E362A24CAB). Модуль **не** перегружался: в памяти остался
  первый билд 1.3, в котором был Oops.
- 22:04:25: штатный reboot завис. `linuwu_sense.service` (ExecStop=rmmod)
  дошёл до `Thermal states saved successfully` и встал на секции
  `four_zone_kb`: `sysfs_remove_group()` не может снять атрибут
  `per_zone_mode`, чей обработчик упал в Oops и не отпустил kernfs.
  systemd: `Stopping timed out` 22:05:55, SIGKILL rmmod 22:07:26 (не
  помогает, задача в ядре). Выключение кнопкой. `kb states saved` не было.
- 22:09: загрузка с новым билдом (srcversion совпадает с файлом). Порядок в
  журнале правильный: `restore deferred` → `HID RGB controller found` →
  `KB states restored successfully`. `per_zone_mode` = `ffffff x4, 37`.
  Клавиатура белая, потому что `/etc/four_zone_kb_state` датирован
  21:58:22 и хранит белый: цвет от 22:00 не сохранился из-за Oops, а
  сохранение при rmmod не случилось из-за зависания. Восстановление
  работает, восстановило то, что лежало на диске.
- Осталось: шаг 5 (запись цвета, проверить mtime файла и `kb states saved`
  без rmmod) и шаг 4 (штатный ребут) на новом билде.
- Шаг 5, 22:18:58: `kb-backlight all FF 60 00 static 60` на новом билде.
  Без Oops, `per_zone_mode` = `ff6000 x4, 60`, mtime файла 22:18:58.569,
  журнал `kb states saved successfully` 22:18:58.571. Путь `four_zone_mode`
  (22:19:09, breathing) тоже сохраняет. Чтение `four_zone_mode` по-прежнему
  показывает мусор WMI, это известно и вне scope.
- Перед шагом 4 выставлен static ff6000/60. Ждём штатный ребут.
- Точка возобновления (22:2x): выполняется шаг 4. Ожидание штатного ребута
  на билде srcversion 8FF57B68D5655E362A24CAB, контрольный цвет static
  ff6000/60. После загрузки проверить: цвет клавиатуры, `journalctl -k -b 0
  | grep linuwu` (порядок `deferred` → `controller found` → `restored`),
  в `journalctl -b -1` строку `kb states saved` при выключении и что
  `linuwu_sense.service` остановился без `Stopping timed out`.
- Шаг 4, ребут 22:22:50 → 22:23:34, билд srcversion 8FF57B68D5655E362A24CAB.
  Выключение (`journalctl -b -1`): `Stopping linuwu_sense.service` 22:22:50.277,
  `Thermal states saved`, `kb states saved successfully` 22:22:50.311,
  `Deactivated successfully` 22:22:50.332. Без `Stopping timed out`.
  Файл `/etc/four_zone_kb_state` mtime 22:22:50.310, совпадает с rmmod.
  Загрузка (`journalctl -k -b 0`): `KB state restore deferred` 22:23:35.078831
  → `Acer HID RGB keyboard controller found` .079879 → `KB states restored
  successfully` .079903. `per_zone_mode` = `ff6000,ff6000,ff6000,ff6000,60`,
  без бита зоны. Клавиатура оранжевая, как выставлено до ребута, без действий
  пользователя. Проверка снаружи пройдена.
- Побочное, вне scope: в этом ребуте появилась строка `Set Device Status
  failed: 0x0 - 0x5` в 22:23:37.39. Это `wmid3_set_device_status()`, путь
  rfkill (wireless/bluetooth через WMID_GUID3), совпадает по времени с
  подъёмом `mt7921e`. В двух предыдущих ребутах её не было, к клавиатуре и
  к патчу отношения не имеет.
- Итог: оба изменения работают. Восстановление идёт после появления HID,
  сохранение при каждой записи в sysfs через workqueue, чистый ребут
  сохраняет и возвращает цвет. Осталось отдельной задачей (Неясность 2):
  коммит в `anv16s41-hid` и обновление PR #127. В рабочем дереве клона
  лежат артефакты сборки (`*.o`, `*.ko`, `*.cmd`, `Module.symvers`),
  перед коммитом их надо исключить.
