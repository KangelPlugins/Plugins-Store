# Plugins Store — Agent Guide

Репозиторий с плагинами для Kangel Plugins Manager (ExteraGram).

## Структура

```
Plugins/              # Текущие версии плагинов (.plugin файлы)
legacy_versions/      # Старые версии плагинов, организованы по папкам
  <plugin_name>/      # Папка для каждого плагина
    <plugin_name>.plugin              # Последняя легаси версия
    <plugin_name>_v<version>.plugin   # Именованные старые версии
store.json            # Метаданные всех плагинов
```

## store.json структура

Корневой объект содержит плагины с ключами-идентификаторами. Каждый плагин — объект с полями:

**Обязательные поля:**
- `url` (string) — URL к актуальному .plugin файлу
- `name` (string) — Отображаемое имя плагина
- `version` (string) — Текущая версия
- `author` (string) — Автор плагина
- `description` (string) — Описание
- `hash` (string) — SHA256 хеш .plugin файла
- `status` (string) — Категория: `"fun"`, `"library"`, `"tools"`, и т.д.

**Опциональные поля:**
- `min_version` (string) — Минимальная версия ExteraGram
- `app_version` (string) — Версия приложения
- `icon` (string) — Путь к иконке
- `dependencies` (array of strings) — Зависимости от других плагинов
- `requirements` (array of strings) — Дополнительные требования
- `legacy_version` (object) — Старые версии плагина

### legacy_version структура

Объект, где ключи — номера версий (string), значения — объекты с:
- `url` (string) — URL к legacy .plugin файлу
- `hash` (string) — SHA256 хеш legacy файла

Пример:
```json
{
  "plugin_id": {
    "url": "https://raw.githubusercontent.com/KangelPlugins/Plugins-Store/main/Plugins/plugin.plugin",
    "name": "Plugin Name",
    "version": "2.0.0",
    "author": "@author",
    "description": "Description text",
    "hash": "sha256hash...",
    "status": "tools",
    "legacy_version": {
      "1.0.0": {
        "url": "https://raw.githubusercontent.com/.../legacy_versions/plugin_id/plugin_id_v1.0.0.plugin",
        "hash": "sha256hash..."
      },
      "1.5.0": {
        "url": "https://raw.githubusercontent.com/.../legacy_versions/plugin_id/plugin_id.plugin",
        "hash": "sha256hash..."
      }
    }
  }
}
```

## Обновление плагинов

При обновлении плагина:

1. Сохрани старую версию в `legacy_versions/<plugin_name>/`:
   - Либо как `<plugin_name>.plugin` (последняя legacy)
   - Либо как `<plugin_name>_v<version>.plugin` (именованная версия)

2. Обнови `Plugins/<plugin_name>.plugin` новой версией

3. Обнови `store.json`:
   - Измени `version` на новую
   - Пересчитай `hash` (SHA256)
   - Добавь запись в `legacy_version` со старой версией

Вычисление хеша:
```python
import hashlib

def get_hash(filepath):
    sha256 = hashlib.sha256()
    with open(filepath, "rb") as f:
        for chunk in iter(lambda: f.read(4096), b""):
            sha256.update(chunk)
    return sha256.hexdigest()
```

## Важно

- Репозиторий обновляется автоматически (см. README.md)
- Не редактируй вручную без необходимости
- Все плагины должны быть в UTF-8
- URL всегда начинается с `https://raw.githubusercontent.com/KangelPlugins/Plugins-Store/main/`
