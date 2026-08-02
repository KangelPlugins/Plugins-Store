# Plugins Store — структура репозитория

Репозиторий с плагинами для Kangel Plugins Manager (ExteraGram). Обновляется автоматически
ботом; вручную обычно ничего не трогать.

## Структура

```
Plugins/                 # текущие версии плагинов
  <id>.plugin            # обычный плагин (один Python-файл)
  <id>.eaf               # elyx-плагин (ZIP-архив)
legacy_versions/         # старые версии, по папке на плагин
  <id>/                  # папка плагина
    <base>_v<version>.<ext>  # именованная версия (основной паттерн)
    <base>.<ext>              # старая запись без версии в имени
store.json               # метаданные всех плагинов (ключ = id)
```

## store.json

Ключ корневого объекта — `id` плагина. Поля (пишутся только если есть):

- `url` — `https://raw.githubusercontent.com/KangelPlugins/Plugins-Store/main/Plugins/<файл>` (обязательное)
- `name`, `version`, `author`, `description`, `hash` — SHA256 файла
- `icon`, `min_version`, `app_version`
- `status` — один из: `utilities`, `customization`, `fun`, `informational`, `messages`, `library`
- `requirements`, `requires`, `dependencies`
- `description_<lang>` — локализованные описания; `<lang>` = язык из elyx-локалей
- `legacy_version` — объект `{ "<version>": { "url": "...", "hash": "sha256" } }`

Пример:
```json
{
  "custom_profile": {
    "url": "https://raw.githubusercontent.com/KangelPlugins/Plugins-Store/main/Plugins/custom_profile.plugin",
    "name": "Custom Profile",
    "version": "1.6",
    "author": "@RoflPlugins",
    "description": "Описание на дефолтном языке",
    "description_ru": "Русское описание",
    "description_en": "English description",
    "hash": "sha256...",
    "status": "customization",
    "legacy_version": {
      "1.1": {
        "url": "https://raw.githubusercontent.com/KangelPlugins/Plugins-Store/main/legacy_versions/custom_profile/Custom Profile_v1.1.plugin",
        "hash": "sha256..."
      }
    }
  }
}
```

Hash — SHA256 по байтам файла:
```python
import hashlib
def get_hash(path):
    return hashlib.sha256(open(path, "rb").read()).hexdigest()
```

## legacy_versions: правило коллизий

Старая версия всегда сохраняется под версионированным именем `<base>_v<version>.<ext>`
и не перезаписывает уже существующий файл в `legacy_versions/<id>/`. Если файл с таким
именем уже есть, но хеш другой — добавляется суффикс (`<base>_v<version>_1.<ext>`).
Это защита от потери версий при коллизии имён.

## Elyx (.eaf / .elyx)

Elyx — структурированный формат плагинов ExteraGram (https://plugins.exteragram.app/docs/elyx).
Файл — ZIP-архив; в корне `refmap` (`.yml`/`.yaml`/`.json`), указывающий на `metainfo`,
`main`, `strings`, `assets`. Метаданные — из `metainfo.json`/`metainfo.yml`/`meta.json`/`meta.yml`
(путь из refmap, fallback — поиск по архиву). В `store.json` поля те же, что и для `.plugin`.
