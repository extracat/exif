# exif

Массовая правка EXIF-метаданных у сканов плёночных кадров, управляемая **именем файла** и базой камер/объективов в JSON.

Скрипт разбирает имя файла по фиксированной маске, находит подходящую камеру и объектив в `exif-presets.json` и одной командой `exiftool` проставляет Make/Model/серийники, данные объектива, дату, ISO и комментарий о плёнке — вместо ручного набора отдельных пресет-скриптов под каждую комбинацию камера+объектив.

## Требования

- [`exiftool`](https://exiftool.org/)
- [`jq`](https://stedolan.github.io/jq/)

Оба должны быть в `PATH`. На macOS: `brew install exiftool jq`.

## Установка

```sh
git clone https://github.com/extracat/exif.git ~/GIT/exif
ln -s ~/GIT/exif/exif ~/bin/exif   # ~/bin должен быть в $PATH
```

Скрипт сам находит `exif-presets.json` рядом со своим реальным расположением (симлинки разрешаются), так что путь установки `~/bin/exif` не важен — конфиг всегда ищется в `~/GIT/exif/`.

## Маска имени файла

```
yyyymmdd-nnn-cam-lens-filmbrand-filmtitle-iso[-asACTUALISO]
```

Пример: `20230623-013-FM3A-35f2-Kodak-Ultramax-400-as200.jpg`

| Поле | Пример | Что делает |
|---|---|---|
| `yyyymmdd` | `20230623` | Дата съёмки → `EXIF:AllDates` (`2023:06:23 00:00:00`), перезаписывает всё, что было в файле (обычно мусор от сканера) |
| `nnn` | `013` | Номер кадра на плёнке — только для имени файла и лога, в EXIF не пишется |
| `cam` | `FM3A` | Код камеры, ищется в `presets.cameras` (регистр не важен) |
| `lens` | `35f2` | Код объектива. Для камер типа `interchangeable` ищется в `presets.lenses`. Для камер типа `fixed` это поле **игнорируется** скриптом — данные объектива берутся из пресета камеры, поле нужно только для читаемости имени файла |
| `filmbrand` | `Kodak` | Вместе с `filmtitle` и `iso` пишутся в `EXIF:UserComment` (`"Kodak Ultramax 400"`) |
| `filmtitle` | `Ultramax` | — |
| `iso` | `400` | Номинальная (коробочная) чувствительность плёнки |
| `asACTUALISO` (опционально) | `as200` | Реальная чувствительность съёмки (push/pull) → `EXIF:ISO`. Если поля нет, в `EXIF:ISO` идёт номинальный `iso`. При push/pull в `UserComment` добавляется `(rated ISO N)` |

Все поля разделяются дефисом — значения полей (`filmtitle`, коды камеры/объектива) не должны содержать дефис.

## Использование

```sh
exif FILE...                     # обработать файлы
exif -n FILE...                  # dry-run: показать команду exiftool, ничего не менять
exif -v FILE...                  # показать команду exiftool при реальном запуске
exif --list-cameras              # список известных камер
exif --list-lenses               # список известных сменных объективов
exif --presets /path/to.json ... # использовать другой файл пресетов
exif --help
```

Обычно используется с глобом:

```sh
exif ~/Scans/2023-roll42/*.jpg
```

Файлы, не подходящие под маску, или с неизвестным кодом камеры/объектива — пропускаются с предупреждением, обработка остальных файлов продолжается. В конце выводится сводка `N updated, M skipped`, и если что-то было пропущено — код возврата ненулевой.

Всегда используется `-overwrite_original` (без сохранения бэкапа `_original`), как и в предыдущей версии на отдельных скриптах.

## База камер и объективов (`exif-presets.json`)

### Камеры

| Код | Тип | Камера |
|---|---|---|
| `fm3a` | interchangeable | Nikon FM3A |
| `capios20` | fixed | Minolta Capios 20 |
| `mjuii` | fixed | Olympus mju II |
| `seagull` | interchangeable | Seagull DF-1 |
| `sokolautomat` | fixed | LOMO Sokol Automat |
| `zenite` | interchangeable | Zenit E |

### Сменные объективы

| Код | Объектив |
|---|---|
| `20f28` | Nikkor 20mm f/2.8 Ai-S |
| `35f2` | Nikkor 35mm f/2 Ai-S |
| `50f14` | Nikkor 50mm f/1.4 Ai-S |
| `85f2` | Nikkor 85mm f/2 Ai-S |
| `jupiter135f35` | Jupiter-37A 3.5/135 |
| `helios44_2` | Helios-44-2 58mm f/2 |
| `haiou58f2` | Haiou-64 58mm f/2 |

### Формат пресета

```jsonc
"cameras": {
  "код_камеры": {
    "type": "interchangeable",   // или "fixed"
    "make": "...", "model": "...", "serialNumber": "...",
    "exposureProgram": "...", "exposureMode": "...",
    "lens": { ... }              // только для type: "fixed" — те же поля, что у объекта из "lenses"
  }
},
"lenses": {
  "код_объектива": {
    "make": "...", "serialNumber": "...",
    "lens": "35.0 mm f/2.0", "model": "...", "info": "35mm f/2",
    "focalLength": "35.0 mm", "maxAperture": "2.0", "focalLengthIn35mm": "35 mm"
  }
}
```

Правила:
- Присутствующий ключ — тег пишется. Значение `""` — тег удаляется (`-TAG=`). Отсутствующий ключ — тег не трогается вообще.
- Чтобы добавить новую камеру или объектив, достаточно дописать запись в JSON — менять `exif` не нужно.
- Коды камер и объективов должны быть уникальны в пределах своей таблицы (`cameras` / `lenses` соответственно) и не содержать дефис.
