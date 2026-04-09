import fcntl
import hashlib
import os
import shutil
import threading
import time
import traceback
import zipfile
from enum import Enum
from typing import Optional, cast

import requests
from java import dynamic_proxy, jarray
from android.content import Intent
from android.content import DialogInterface
from android.graphics import Color
from android.net import Uri
from android.os import Environment
from android.util import Log
from android.webkit import ValueCallback
from android_utils import run_on_ui_thread
from base_plugin import BasePlugin, MenuItemData, MenuItemType
from client_utils import get_last_fragment
from dalvik.system import InMemoryDexClassLoader
from java.lang import Class, Integer, Long, Runnable, String
from java.nio import ByteBuffer  # ty:ignore[unresolved-import]
from java.util.function import Function
from org.telegram.messenger import ApplicationLoader, LocaleController
from org.telegram.messenger import R as R_tg  # ty:ignore[unresolved-import]
from org.telegram.ui.ActionBar import AlertDialog
from typing_extensions import Any
from ui.bulletin import BulletinHelper
from ui.settings import Divider, Header, Selector, Switch, Text

__id__ = "tg-streaks"
__name__ = "Streaks"
__description__ = "Analog for TikTok streaks for Telegram"
__author__ = "@n08i40k & @RoflPlugins"
__version__ = "2.8.0"
__icon__ = "exteraPlugins/0"
__min_version__ = "12.1.1"

DEBUG_MODE = False
LOGCAT_TAG = "tg-streaks"


REPO_OWNER = "n08i40k"
REPO_NAME = __id__

DEX_URL = f"https://github.com/{REPO_OWNER}/{REPO_NAME}/releases/download/{__version__}/classes.dex"
DEX_SHA256 = "6e5087040eea387536d45a3ab103bca0525be0d9ca904fece14a943bc1c993cc"
RESOURCES_URL = f"https://github.com/{REPO_OWNER}/{REPO_NAME}/releases/download/{__version__}/resources.zip"
RESOURCES_SHA256 = "7cafa8479a39e870f6c785d81a83126fc1b6779c18df9f61447d3b173a35d49d"

PLUGIN_UPDATE_API_URL = (
    f"https://api.github.com/repos/{REPO_OWNER}/{REPO_NAME}/releases/latest"
)
PLUGIN_UPDATE_TG_URL = "tg://resolve?domain=n08i40k_extera&post=3"
UPDATE_CHECK_TIMEOUT_SECONDS = 6
SETTING_UPDATE_CHECK_ENABLED = "update_check_enabled"
SETTING_PET_FAB_SIZE_INDEX = "pet_fab_size_index"
PET_FAB_SIZE_OPTIONS_DP = (64, 80, 96, 112, 128)


def get_plugin_cache_dir(*parts: str) -> str:
    cache_root = ApplicationLoader.applicationContext.getCacheDir().getAbsolutePath()
    return os.path.join(cache_root, __id__, *parts)


I18N_SETTINGS: dict[str, dict[str, str]] = {
    "settings.updates.title": {"en": "Updates", "ru": "Обновления"},
    "settings.updates.auto_check.title": {
        "en": "Update checks",
        "ru": "Проверка обновлений",
    },
    "settings.updates.auto_check.description": {
        "en": "Checks GitHub releases on startup.",
        "ru": "Проверяет релизы GitHub при запуске.",
    },
    "settings.streak_tools.title": {
        "en": "Streak",
        "ru": "Стрик",
    },
    "settings.pet_button.title": {"en": "Streak pet", "ru": "Стрик-пет"},
    "settings.pet_button.size.title": {
        "en": "Button size",
        "ru": "Размер кнопки",
    },
    "settings.pet_button.size.description": {
        "en": "Floating chat button size.",
        "ru": "Размер плавающей кнопки в чате.",
    },
    "settings.streak_tools.rebuild_all_chats.title": {
        "en": "Private chats rebuild",
        "ru": "Пересчёт стриков в личках",
    },
    "settings.streak_tools.rebuild_all_chats.description": {
        "en": "Only user DMs are checked. Bots and groups are skipped.",
        "ru": "Только лички с пользователями. Боты и группы пропускаются.",
    },
    "settings.backups.title": {
        "en": "Backups",
        "ru": "Бэкапы",
    },
    "settings.backups.export.title": {
        "en": "Create backup",
        "ru": "Создать бэкап",
    },
    "settings.backups.restore.title": {
        "en": "Restore backup",
        "ru": "Восстановить бэкап",
    },
    "settings.backups.reset_database.title": {
        "en": "Reset plugin database",
        "ru": "Сбросить базу плагина",
    },
    "settings.backups.description": {
        "en": "Backups are saved to Downloads/tg-streaks. Restore replaces the current database.",
        "ru": "Бэкапы лежат в Downloads/tg-streaks. Восстановление заменяет текущую базу.",
    },
}

I18N_STATUS: dict[str, dict[str, str]] = {
    "status.error.chat.detect_current_failed": {
        "en": "Current chat not found",
        "ru": "Текущий чат не найден",
    },
    "status.error.chat.open_context_failed": {
        "en": "Chat context unavailable",
        "ru": "Контекст чата недоступен",
    },
    "status.error.streak.jump_to_start_failed": {
        "en": "Couldn't open streak start",
        "ru": "Не удалось открыть начало стрика",
    },
    "status.error.backup.export_failed": {
        "en": "Backup export failed",
        "ru": "Не удалось экспортировать бэкап",
    },
    "status.error.backup.not_found": {
        "en": "No backups found",
        "ru": "Бэкапы не найдены",
    },
    "status.error.backup.apply_failed": {
        "en": "Backup restore failed: {reason}",
        "ru": "Не удалось восстановить бэкап: {reason}",
    },
    "status.error.database.delete_failed": {
        "en": "Database reset failed: {reason}",
        "ru": "Не удалось сбросить базу: {reason}",
    },
    "status.error.update.open_link_failed": {
        "en": "Couldn't open update link",
        "ru": "Не удалось открыть ссылку обновления",
    },
    "status.error.rebuild.failed_check_logs": {
        "en": "Rebuild failed, check logs",
        "ru": "Пересчёт не удался, проверьте логи",
    },
    "status.success.backup.exported": {
        "en": "Backup saved: {name}",
        "ru": "Бэкап сохранён: {name}",
    },
    "status.success.backup.imported": {
        "en": "Backup restored: {name}",
        "ru": "Бэкап восстановлен: {name}",
    },
    "status.success.database.reset_started": {
        "en": "Database reset started",
        "ru": "Сброс базы запущен",
    },
    "status.success.streak.jump_to_start_completed": {
        "en": "Opened streak start",
        "ru": "Начало стрика открыто",
    },
    "status.success.streak.restored": {
        "en": "Streak restored",
        "ru": "Стрик восстановлен",
    },
    "status.success.chat.level_messages_enabled": {
        "en": "Service messages on in this chat",
        "ru": "Сервисные сообщения в чате включены",
    },
    "status.success.chat.level_messages_disabled": {
        "en": "Service messages off in this chat",
        "ru": "Сервисные сообщения в чате выключены",
    },
    "status.success.pet_button.enabled": {
        "en": "Streak pet button shown in all chats",
        "ru": "Кнопка стрик-пета показана во всех чатах",
    },
    "status.success.pet_button.disabled": {
        "en": "Streak pet button hidden in all chats",
        "ru": "Кнопка стрик-пета скрыта во всех чатах",
    },
    "status.success.pet.created": {
        "en": "Streak pet created",
        "ru": "Стрик-пет создан",
    },
    "status.success.debug.streak_set_to_3_days": {
        "en": "Debug: streak set to 3 days",
        "ru": "Debug: стрик установлен на 3 дня",
    },
    "status.success.debug.streak_marked_dead": {
        "en": "Debug: streak marked as dead",
        "ru": "Debug: стрик помечен как мёртвый",
    },
    "status.success.debug.streak_upgraded": {
        "en": "Debug: streak upgraded to {days} days",
        "ru": "Debug: стрик улучшен до {days} дней",
    },
    "status.success.debug.streak_frozen": {
        "en": "Debug: streak frozen",
        "ru": "Debug: стрик заморожен",
    },
    "status.success.debug.streak_deleted": {
        "en": "Debug: streak deleted",
        "ru": "Debug: стрик удалён",
    },
    "status.success.debug.pet_deleted": {
        "en": "Debug: streak pet deleted",
        "ru": "Debug: стрик-пет удалён",
    },
    "status.info.chat.private_users_only": {
        "en": "Only for private chats",
        "ru": "Только для личных чатов",
    },
    "status.info.chat.bots_not_supported": {
        "en": "Bots are not supported",
        "ru": "Для ботов недоступно",
    },
    "status.info.chat.deleted_users_not_supported": {
        "en": "Deleted users are not supported",
        "ru": "Для удалённых аккаунтов недоступно",
    },
    "status.info.pet.not_created_for_chat": {
        "en": "No streak pet in this chat yet",
        "ru": "В этом чате ещё нет стрик-пета",
    },
    "status.info.pet.already_exists_for_chat": {
        "en": "Streak pet already exists",
        "ru": "Стрик-пет уже создан",
    },
    "status.info.streak.not_ended_yet": {
        "en": "Streak is still active",
        "ru": "Стрик ещё активен",
    },
    "status.info.streak.restore_unavailable": {
        "en": "Streak can no longer be restored",
        "ru": "Стрик уже нельзя восстановить",
    },
    "status.info.streak.searching_start_message": {
        "en": "Looking for streak start...",
        "ru": "Ищу начало стрика...",
    },
    "status.info.streak.start_message_not_found": {
        "en": "Exact message not found, opened streak day",
        "ru": "Точное сообщение не найдено, открыт день начала",
    },
    "status.info.streak.not_found_for_chat": {
        "en": "No streak in this chat",
        "ru": "В этом чате нет стрика",
    },
    "status.info.rebuild.already_running": {
        "en": "Rebuild already running",
        "ru": "Пересчёт уже идёт",
    },
    "status.info.rebuild.started_all_chats": {
        "en": "Rebuild started",
        "ru": "Пересчёт запущен",
    },
    "status.info.debug.private_users_only": {
        "en": "Debug: only for private chats",
        "ru": "Debug: только для личных чатов",
    },
    "status.info.debug.streak_already_max": {
        "en": "Debug: streak is already maxed",
        "ru": "Debug: стрик уже максимальный",
    },
}

I18N_SHEETS: dict[str, dict[str, str]] = {
    "sheet.streak_info.dialog_title": {
        "en": "Streak",
        "ru": "Стрик",
    },
    "sheet.streak_info.header.title": {
        "en": "You and {name} are on a {days}-day streak!",
        "ru": "У вас с {name} стрик уже {days} дней!",
    },
    "sheet.streak_info.header.description": {
        "en": "Keep it going **:P**",
        "ru": "Продолжайте в том же духе **:P**",
    },
    "sheet.streak_info.feature.how.title": {
        "en": "How it works?",
        "ru": "Как это работает?",
    },
    "sheet.streak_info.feature.how.description": {
        "en": "After three days of communication in a row, you will have a streak!\n It improves depending on duration.",
        "ru": "После трёх дней общения подряд у вас появится стрик!\nОн улучшается в зависимости от длительности.",
    },
    "sheet.streak_info.feature.levels.title": {
        "en": "Levels!",
        "ru": "Уровни!",
    },
    "sheet.streak_info.feature.levels.description": {
        "en": "Levels unlock at 3, 10, 30, 100 and 200+ days.\nA popup appears on level-up.",
        "ru": "Уровни открываются на 3, 10, 30, 100 и 200+ днях.\nПри повышении появляется попап.",
    },
    "sheet.streak_info.feature.keep.title": {
        "en": "Don't drop it ;)",
        "ru": "Не забывайте о нём ;)",
    },
    "sheet.streak_info.feature.keep.description": {
        "en": "If you don't message each other for 24 hours, the streak ends.\nYou get another 24 hours to restore it.",
        "ru": "Если не писать друг другу 24 часа, стрик прервётся.\nНа восстановление будет ещё 24 часа.",
    },
    "sheet.streak_info.feature.fix_duration.title": {
        "en": "Wrong streak length?",
        "ru": "Стрик считается неверно?",
    },
    "sheet.streak_info.feature.fix_duration.description": {
        "en": 'Use "Streak rebuild" to recount it.',
        "ru": 'Используйте "Пересчёт стрика", чтобы пересчитать его.',
    },
    "sheet.pet.streak_days": {"en": "Streak days", "ru": "Дни стрика"},
    "sheet.pet.points_to_next_stage": {
        "en": "{count} points to evolve",
        "ru": "{count} очков до эволюции",
    },
    "sheet.pet.max_level": {"en": "Max", "ru": "Макс."},
    "sheet.pet.locked": {"en": "Locked", "ru": "Заблокировано"},
    "sheet.pet.locked.description": {
        "en": "Open the previous form first",
        "ru": "Сначала откройте прошлую форму",
    },
    "sheet.pet.tasks.title": {"en": "Today's tasks", "ru": "Задания на сегодня"},
    "sheet.pet.badges.title": {"en": "Streak badges", "ru": "Бейджи стрика"},
    "sheet.pet.tasks.exchange_one_message": {
        "en": "1 message each",
        "ru": "По 1 сообщению",
    },
    "sheet.pet.tasks.send_four_messages_each": {
        "en": "4 messages each",
        "ru": "По 4 сообщения",
    },
    "sheet.pet.tasks.send_ten_messages_each": {
        "en": "10 messages each",
        "ru": "По 10 сообщений",
    },
    "sheet.pet.progress.you": {"en": "You", "ru": "Вы"},
    "sheet.pet.progress.partner": {"en": "Partner", "ru": "Партнёр"},
    "sheet.pet.rename.title": {
        "en": "Rename streak pet",
        "ru": "Переименовать стрик-пета",
    },
    "sheet.pet.rename.placeholder": {
        "en": "New name",
        "ru": "Новое имя",
    },
    "sheet.pet.rename.save": {"en": "Save", "ru": "Сохранить"},
    "sheet.pet.rename.cancel": {"en": "Cancel", "ru": "Отмена"},
}

I18N_MENU: dict[str, dict[str, str]] = {
    "menu.chat.create_pet.title": {
        "en": "Streak pet",
        "ru": "Стрик-пет",
    },
    "menu.chat.create_pet.description": {
        "en": "Create it or send an invite",
        "ru": "Создать или отправить инвайт",
    },
    "menu.chat.restore_streak.title": {
        "en": "Streak restore",
        "ru": "Восстановление стрика",
    },
    "menu.chat.restore_streak.description": {
        "en": "Available for 24 hours after it ends",
        "ru": "Доступно 24 часа после обрыва",
    },
    "menu.chat.restore_streak_exact.title": {
        "en": "Streak restore menu",
        "ru": "Меню восстановление стрика",
    },
    "menu.chat.restore_streak_exact.description": {
        "en": "Available anytime, but limited by 2 usages per chat",
        "ru": "Доступно в любое время, но с ограничением по 2 раза на чат",
    },
    "menu.chat.rebuild.streak.title": {
        "en": "Streak rebuild",
        "ru": "Пересчёт стрика",
    },
    "menu.chat.rebuild.streak.description": {
        "en": "Rereads chat history and updates the streak",
        "ru": "Заново проверяет чат и обновляет стрик",
    },
    "menu.chat.rebuild.pet.title": {
        "en": "Streak pet rebuild",
        "ru": "Пересчёт стрик-пета",
    },
    "menu.chat.rebuild.pet.description": {
        "en": "Recounts streak pet tasks and points",
        "ru": "Пересчитывает задачи и очки",
    },
    "menu.chat.open_streak_start.title": {
        "en": "Streak start",
        "ru": "Начало стрика",
    },
    "menu.chat.open_streak_start.description": {
        "en": "Opens the message or day where it began",
        "ru": "Открывает сообщение или день начала",
    },
    "menu.chat.toggle_pet_button.title": {
        "en": "Streak pet button",
        "ru": "Кнопка стрик-пета",
    },
    "menu.chat.toggle_pet_button.description": {
        "en": "Shows or hides the floating button in all chats",
        "ru": "Показывает или скрывает кнопку во всех чатах",
    },
    "menu.chat.toggle_level_messages.title": {
        "en": "Service messages",
        "ru": "Сервисные сообщения",
    },
    "menu.chat.toggle_level_messages.description": {
        "en": "Shows or hides level-up messages in this chat",
        "ru": "Показывает или скрывает сообщения о новых уровнях",
    },
    "menu.debug.create_streak.title": {
        "en": "[DEBUG] 3-day streak",
        "ru": "[DEBUG] Стрик на 3 дня",
    },
    "menu.debug.create_streak.description": {
        "en": "Sets a 3-day streak in this chat",
        "ru": "Ставит стрик на 3 дня в этом чате",
    },
    "menu.debug.crash_plugin.title": {
        "en": "[DEBUG] Plugin crash",
        "ru": "[DEBUG] Краш плагина",
    },
    "menu.debug.crash_plugin.description": {
        "en": "Test crash",
        "ru": "Тестовый краш",
    },
    "menu.debug.kill_streak.title": {
        "en": "[DEBUG] Streak break",
        "ru": "[DEBUG] Обрыв стрика",
    },
    "menu.debug.kill_streak.description": {
        "en": "Breaks the streak in this chat",
        "ru": "Прерывает стрик в этом чате",
    },
    "menu.debug.upgrade_streak.title": {
        "en": "[DEBUG] Streak upgrade",
        "ru": "[DEBUG] Повышение стрика",
    },
    "menu.debug.upgrade_streak.description": {
        "en": "Moves the streak to the next level",
        "ru": "Поднимает стрик до следующего уровня",
    },
    "menu.debug.freeze_streak.title": {
        "en": "[DEBUG] Streak freeze",
        "ru": "[DEBUG] Заморозка стрика",
    },
    "menu.debug.freeze_streak.description": {
        "en": "Sets a frozen streak in this chat",
        "ru": "Ставит замороженный стрик в этом чате",
    },
    "menu.debug.delete_streak.title": {
        "en": "[DEBUG] Streak delete",
        "ru": "[DEBUG] Удаление стрика",
    },
    "menu.debug.delete_streak.description": {
        "en": "Deletes the streak from this chat",
        "ru": "Удаляет стрик из этого чата",
    },
    "menu.debug.delete_pet.title": {
        "en": "[DEBUG] Streak pet delete",
        "ru": "[DEBUG] Удаление стрик-пета",
    },
    "menu.debug.delete_pet.description": {
        "en": "Deletes the streak pet from this chat",
        "ru": "Удаляет стрик-пета из этого чата",
    },
}

I18N_DIALOGS: dict[str, dict[str, str]] = {
    "dialog.create_pet.title": {
        "en": "Streak pet",
        "ru": "Стрик-пет",
    },
    "dialog.create_pet.message": {
        "en": "Does the other person use the plugin?",
        "ru": "У собеседника есть плагин?",
    },
    "dialog.create_pet.confirm": {"en": "Yes", "ru": "Да"},
    "dialog.create_pet.cancel": {"en": "No", "ru": "Нет"},
    "dialog.backup_restore.title": {
        "en": "Choose backup",
        "ru": "Выберите бэкап",
    },
    "dialog.calendar_fix.manual_revive.title": {
        "en": "Mark restore day?",
        "ru": "Пометить день восстановлением?",
    },
    "dialog.calendar_fix.manual_revive.message_gap": {
        "en": "This day looks like a streak restore day. Mark it as restored?",
        "ru": "Этот день выглядит как день восстановления стрика. Пометить его как восстановленный?",
    },
    "dialog.calendar_fix.manual_revive.message_dead_chain": {
        "en": "This day looks like a restore point in a dead streak chain. Mark it as restored?",
        "ru": "Этот день выглядит как точка восстановления в мёртвой цепочке стрика. Пометить его как восстановленный?",
    },
    "dialog.calendar_fix.rebuild.title": {
        "en": "Rebuild streak?",
        "ru": "Пересчитать стрик?",
    },
    "dialog.calendar_fix.rebuild.message": {
        "en": "The restore mark is saved. Rebuild the streak now?",
        "ru": "Пометка восстановления сохранена. Пересчитать стрик сейчас?",
    },
    "dialog.calendar_fix.warning_next_day.title": {
        "en": "Better tap the next day",
        "ru": "Лучше нажмите на следующий день",
    },
    "dialog.calendar_fix.warning_next_day.message": {
        "en": "This day looks like the death day. The restore should be placed on the next day.",
        "ru": "Этот день больше похож на день смерти стрика. Восстановление лучше ставить на следующий день.",
    },
    "dialog.calendar_fix.limit_reached.title": {
        "en": "Restore limit reached",
        "ru": "Лимит восстановлений достигнут",
    },
    "dialog.calendar_fix.limit_reached.message": {
        "en": "You have already used the maximum number of manual calendar restores for this chat.",
        "ru": "Для этого чата уже использовано максимальное количество ручных восстановлений через календарь.",
    },
    "dialog.calendar_fix.confirm": {"en": "Confirm", "ru": "Подтвердить"},
    "dialog.calendar_fix.cancel": {"en": "Cancel", "ru": "Отмена"},
    "dialog.calendar_fix.rebuild_now": {
        "en": "Rebuild",
        "ru": "Пересчитать",
    },
    "dialog.calendar_fix.later": {"en": "Later", "ru": "Позже"},
    "dialog.calendar_fix.ok": {"en": "OK", "ru": "Ок"},
}

I18N_REBUILD: dict[str, dict[str, str]] = {
    "rebuild.streak.progress.chat": {
        "en": "Streak rebuild: {peer_name} • {days_checked} d.",
        "ru": "Пересчёт стрика: {peer_name} • {days_checked} д.",
    },
    "rebuild.streak.progress.all_chats": {
        "en": "Streak rebuild: {peer_name} [{checked_chats}/{total_chats}] • {days_checked} d.",
        "ru": "Пересчёт стрика: {peer_name} [{checked_chats}/{total_chats}] • {days_checked} д.",
    },
    "rebuild.streak.summary.all_chats": {
        "en": "Rebuilt {checked} private chats",
        "ru": "Пересчитано {checked} личных чатов",
    },
    "rebuild.streak.summary.chat": {
        "en": "Streak with {peer_name}: {days} d., {revives} revives.",
        "ru": "Стрик с {peer_name}: {days} д., восстановлений: {revives}.",
    },
    "rebuild.streak.retry_delay": {
        "en": "Telegram didn't respond. Retry in {seconds}s.",
        "ru": "Telegram не ответил. Повтор через {seconds} сек.",
    },
}

I18N_UPDATE: dict[str, dict[str, str]] = {
    "update.available.message": {
        "en": "Update available: {current} -> {latest}",
        "ru": "Есть обновление: {current} -> {latest}",
    },
    "update.available.action": {"en": "Update", "ru": "Обновить"},
}

I18N_DOWNLOAD: dict[str, dict[str, str]] = {
    "download.engine.started": {
        "en": "Downloading engine...",
        "ru": "Скачиваю движок...",
    },
    "download.engine.completed": {
        "en": "Engine downloaded",
        "ru": "Движок скачан",
    },
    "download.assets.started": {
        "en": "Downloading assets...",
        "ru": "Скачиваю ресурсы...",
    },
    "download.assets.completed": {
        "en": "Assets downloaded",
        "ru": "Ресурсы скачаны",
    },
    "download.progress.known_total": {
        "en": "{percent}% • {downloaded}/{total} • ETA {eta}",
        "ru": "{percent}% • {downloaded}/{total} • ETA {eta}",
    },
    "download.progress.unknown_total": {
        "en": "Downloaded {downloaded} • ETA...",
        "ru": "Скачано {downloaded} • ETA...",
    },
}

I18N_SERVICE_MESSAGES: dict[str, dict[str, str]] = {
    "service.streak.started.text": {
        "en": "Streak started!",
        "ru": "Стрик начался!",
    },
    "service.streak.level_up.text": {
        "en": "Streak: {days} days!",
        "ru": "Стрик: {days} дней!",
    },
    "service.streak.ended.title": {
        "en": "Streak ended!",
        "ru": "Стрик прервался!",
    },
    "service.streak.ended.subtitle": {
        "en": "You have 24 hours to restore it",
        "ru": "На восстановление есть 24 часа",
    },
    "service.streak.ended.hint": {
        "en": "Tap below to restore it",
        "ru": "Нажмите ниже, чтобы восстановить",
    },
    "service.streak.ended.action": {"en": "Restore", "ru": "Восстановить"},
    "service.streak.restored.self": {
        "en": "You restored the streak!",
        "ru": "Вы восстановили стрик!",
    },
    "service.streak.restored.peer": {
        "en": "{name} restored the streak!",
        "ru": "{name} восстановил(а) стрик!",
    },
    "service.pet.invite.title": {
        "en": "Join streak pet?",
        "ru": "Вступить в стрик-пета?",
    },
    "service.pet.invite.description": {
        "en": "Shared streak pet for this chat",
        "ru": "Общий стрик-пет для этого чата",
    },
    "service.pet.invite.hint": {
        "en": "Tap below to accept",
        "ru": "Нажмите ниже, чтобы принять",
    },
    "service.pet.invite.action": {"en": "Accept", "ru": "Принять"},
    "service.pet.invite.sent.self": {
        "en": "You sent a streak-pet invite!",
        "ru": "Вы отправили инвайт в стрик-пета!",
    },
    "service.pet.invite.accepted.peer": {
        "en": "{name} accepted the invite!",
        "ru": "{name} принял(а) инвайт!",
    },
    "service.pet.invite.accepted.self": {
        "en": "You accepted the invite!",
        "ru": "Вы приняли инвайт!",
    },
    "service.pet.rename.self": {
        "en": "You named the streak pet {petName}!",
        "ru": "Вы назвали стрик-пета {petName}!",
    },
    "service.pet.rename.peer": {
        "en": "{peerName} named the streak pet {petName}!",
        "ru": "{peerName} назвал(а) стрик-пета {petName}!",
    },
}

I18N_STRINGS: dict[str, dict[str, str]] = {
    **I18N_SETTINGS,
    **I18N_STATUS,
    **I18N_SHEETS,
    **I18N_MENU,
    **I18N_DIALOGS,
    **I18N_REBUILD,
    **I18N_UPDATE,
    **I18N_DOWNLOAD,
    **I18N_SERVICE_MESSAGES,
}


class StreakLevel:
    length: int
    document_id: int
    popup_resource_name: str
    text_color: Color
    text_color_int: int

    def __init__(
        self,
        length: int,
        document_id: int,
        popup_resource_name: str,
        text_color: tuple[int, int, int],
    ):
        self.length = int(length)
        self.document_id = int(document_id)
        self.popup_resource_name = str(popup_resource_name)
        self.text_color = Color.valueOf(
            text_color[0] / 255,
            text_color[1] / 255,
            text_color[2] / 255,
            1.0,
        )
        self.text_color_int = Color.rgb(
            int(text_color[0]),
            int(text_color[1]),
            int(text_color[2]),
        )


class StreakLevels(Enum):
    COLD = StreakLevel(0, 5285071881815235305, "", (175, 175, 175))
    DAYS_3 = StreakLevel(3, 5285079178964672780, "3.webm", (255, 154, 0))
    DAYS_10 = StreakLevel(10, 5285274844789777412, "10.webm", (255, 100, 0))
    DAYS_30 = StreakLevel(30, 5285076623459129616, "30.webm", (255, 61, 0))
    DAYS_100 = StreakLevel(100, 5285003347022093599, "100.webm", (255, 0, 200))
    DAYS_200 = StreakLevel(200, 5285514817497504375, "200.webm", (176, 0, 255))

    @staticmethod
    def pick_by_length(length: int, cold: bool = False) -> StreakLevel:
        if cold:
            return StreakLevels.COLD.value

        if length < 10:
            return StreakLevels.DAYS_3.value
        if length < 30:
            return StreakLevels.DAYS_10.value
        if length < 100:
            return StreakLevels.DAYS_30.value
        if length < 200:
            return StreakLevels.DAYS_100.value

        return StreakLevels.DAYS_200.value


class StreakPetLevel:
    max_points: int
    image_resource_path: str
    gradient_start: str
    gradient_end: str
    pet_start: str
    pet_end: str
    accent: str
    accent_secondary: str

    def __init__(
        self,
        max_points: int,
        image_resource_path: str,
        gradient_start: str,
        gradient_end: str,
        pet_start: str,
        pet_end: str,
        accent: str,
        accent_secondary: str,
    ):
        self.max_points = int(max_points)
        self.image_resource_path = str(image_resource_path)
        self.gradient_start = str(gradient_start)
        self.gradient_end = str(gradient_end)
        self.pet_start = str(pet_start)
        self.pet_end = str(pet_end)
        self.accent = str(accent)
        self.accent_secondary = str(accent_secondary)


class StreakPetLevels(Enum):
    POINTS_100 = StreakPetLevel(
        100,
        "points-100.webm",
        "#F9B746",
        "#FFF8E8",
        "#FFCB68",
        "#FF9C24",
        "#8D4A00",
        "#FFF2C8",
    )
    POINTS_300 = StreakPetLevel(
        300,
        "points-300.webm",
        "#FEA386",
        "#FFF2EC",
        "#FFC0A9",
        "#F9724F",
        "#8A2E19",
        "#FFE1D6",
    )
    POINTS_500 = StreakPetLevel(
        500,
        "points-500.webm",
        "#FF8EFA",
        "#FFF0FF",
        "#FFB6FC",
        "#FF63E3",
        "#842C7A",
        "#FFE3FB",
    )
    POINTS_900 = StreakPetLevel(
        900,
        "points-900.webm",
        "#6873FF",
        "#EEF0FF",
        "#98A1FF",
        "#4A56F0",
        "#2230A3",
        "#DFE3FF",
    )


class JvmPluginBridge:
    klass: Optional[Class]

    def __init__(self, plugin: "TgStreaksPlugin"):
        self.plugin = plugin
        self.klass = None
        self.cache_dir = get_plugin_cache_dir("plugins_dex_cache")
        os.makedirs(self.cache_dir, exist_ok=True)
        self.dex_path = os.path.join(self.cache_dir, f"{__id__}.dex")
        self._download_lock = threading.Lock()

    def load(self):
        if DEBUG_MODE:
            self.plugin.log(
                "Debug mode enabled. Downloading DEX without SHA256 checks..."
            )
            dex_data = self._download_bytes(show_bulletins=True)
            if dex_data is not None:
                self._write_dex_file(dex_data)
                self._load(dex_data)
                return

            self.plugin.log(
                "DEX download failed in debug mode. Falling back to cached DEX if available..."
            )
            self._load_cached_file()
            return

        expected_sha256 = str(DEX_SHA256).strip().lower()
        cached_sha256 = self._compute_file_sha256(self.dex_path)

        if cached_sha256 == expected_sha256:
            self._load_cached_file()
            return

        if cached_sha256 is not None:
            self.plugin.log(
                f"Cached DEX SHA256 mismatch (cached={cached_sha256}, expected={expected_sha256}). Loading cached DEX and refreshing in background..."
            )
            self._load_cached_file()

            if self.klass is not None:
                self._refresh_async()
                return

            self.plugin.log(
                "Cached DEX exists but failed to load. Downloading replacement synchronously..."
            )
        else:
            self.plugin.log("Cached DEX not found. Downloading new file...")

        dex_data = self._download_bytes(show_bulletins=True)
        if dex_data is None:
            return

        downloaded_sha256 = self._compute_sha256(dex_data)
        if downloaded_sha256 != expected_sha256:
            raise RuntimeError("Downloaded DEX SHA256 mismatch")

        self._write_dex_file(dex_data)
        self._load(dex_data)

    def _load_cached_file(self):
        try:
            with open(self.dex_path, "rb") as f:
                self._load(f.read())
        except Exception as e:
            self.plugin.log_exception("Failed to load cached DEX", e)

    def _compute_sha256(self, data: bytes) -> str:
        return hashlib.sha256(data).hexdigest().lower()

    def _compute_file_sha256(self, path: str) -> Optional[str]:
        if not os.path.exists(path):
            return None

        try:
            with open(path, "rb") as f:
                return self._compute_sha256(f.read())
        except Exception as e:
            self.plugin.log_exception("Failed to read cached DEX for SHA256", e)
            return None

    def _write_dex_file(self, dex_data: bytes):
        with open(self.dex_path, "wb") as f:
            f.write(dex_data)

    def _download_bytes(self, show_bulletins: bool) -> Optional[bytes]:
        try:
            return self.plugin._download_with_progress(
                url=DEX_URL,
                started_key="download.engine.started",
                completed_key="download.engine.completed",
                show_bulletins=show_bulletins,
            )
        except Exception as e:
            self.plugin.log_exception("Failed to download DEX", e)
            return None

    def _refresh_async(self):
        def worker():
            if not self._download_lock.acquire(blocking=False):
                return

            try:
                dex_data = self._download_bytes(show_bulletins=True)
                if dex_data is None:
                    return

                downloaded_sha256 = self._compute_sha256(dex_data)
                if downloaded_sha256 != str(DEX_SHA256).strip().lower():
                    raise RuntimeError("Downloaded DEX SHA256 mismatch")

                self._write_dex_file(dex_data)
            except Exception as e:
                self.plugin.log_exception("Failed to refresh DEX", e)
            finally:
                self._download_lock.release()

        threading.Thread(
            target=worker, name="tg-streaks-dex-refresh", daemon=True
        ).start()

    def _load(self, dex_data: bytes):
        class_path = "ru.n08i40k.streaks.Plugin"

        try:
            loader = InMemoryDexClassLoader(
                ByteBuffer.wrap(dex_data),  # ty:ignore[unresolved-attribute]
                ApplicationLoader.applicationContext.getClassLoader(),
            )
            self.klass = loader.loadClass(String(class_path))
        except Exception as e:
            self.plugin.log_exception("Failed to load DEX", e)


class ZipResourcesBridge:
    def __init__(self, plugin: "TgStreaksPlugin"):
        self.plugin = plugin
        self.cache_dir = get_plugin_cache_dir("plugins_resources_cache")
        os.makedirs(self.cache_dir, exist_ok=True)
        self.zip_path = os.path.join(self.cache_dir, f"{__id__}-resources.zip")
        self.resources_root = os.path.join(self.cache_dir, "resources")
        self.lock_path = f"{self.zip_path}.lock"

    def load(self) -> Optional[str]:
        if DEBUG_MODE:
            return self._load_debug()

        expected_sha256 = str(RESOURCES_SHA256).strip().lower()
        cached_sha256 = self._compute_file_sha256(self.zip_path)

        if cached_sha256 == expected_sha256 and os.path.isdir(self.resources_root):
            return self.resources_root

        if cached_sha256 is not None and os.path.isdir(self.resources_root):
            self.plugin.log(
                f"Cached resources ZIP SHA256 mismatch (cached={cached_sha256}, expected={expected_sha256}). Using current resources and refreshing in background..."
            )
            self._refresh_async()
            return self.resources_root

        return self._load_release_slow_path(expected_sha256)

    def _load_debug(self) -> Optional[str]:
        lock_fd = self._acquire_download_lock(blocking=False)
        if lock_fd is not None:
            try:
                self.plugin.log(
                    "Debug mode enabled. Downloading resources ZIP without SHA256 checks..."
                )
                zip_data = self._download_bytes(show_bulletins=True)
                if zip_data is not None:
                    self._write_zip_file(zip_data)
                    self._extract_zip()
                    return (
                        self.resources_root
                        if os.path.isdir(self.resources_root)
                        else None
                    )
            finally:
                self._release_download_lock(lock_fd)
        else:
            self.plugin.log(
                "Resources ZIP download is already in progress. Waiting for the existing run to finish..."
            )
            wait_lock_fd = self._acquire_download_lock(blocking=True)
            try:
                pass
            finally:
                self._release_download_lock(wait_lock_fd)

        if os.path.isdir(self.resources_root):
            return self.resources_root

        if os.path.exists(self.zip_path):
            extract_lock_fd = self._acquire_download_lock(blocking=True)
            try:
                if not os.path.isdir(self.resources_root):
                    self._extract_zip()
            finally:
                self._release_download_lock(extract_lock_fd)
            if os.path.isdir(self.resources_root):
                return self.resources_root

        self.plugin.log(
            "Resources ZIP download failed in debug mode. Falling back to current extracted resources if available..."
        )
        return self.resources_root if os.path.isdir(self.resources_root) else None

    def _load_release_slow_path(self, expected_sha256: str) -> Optional[str]:
        refresh_needed = False

        lock_fd = self._acquire_download_lock(blocking=False)
        if lock_fd is not None:
            try:
                result_path, refresh_needed = self._load_release_slow_path_locked(
                    expected_sha256,
                    allow_download=True,
                )
            finally:
                self._release_download_lock(lock_fd)
        else:
            self.plugin.log(
                "Resources ZIP update is already in progress. Waiting for the existing run to finish..."
            )
            wait_lock_fd = self._acquire_download_lock(blocking=True)
            try:
                result_path, refresh_needed = self._load_release_slow_path_locked(
                    expected_sha256,
                    allow_download=False,
                )
            finally:
                self._release_download_lock(wait_lock_fd)

        if refresh_needed:
            self._refresh_async()

        return result_path

    def _load_release_slow_path_locked(
        self,
        expected_sha256: str,
        allow_download: bool,
    ) -> tuple[Optional[str], bool]:
        cached_sha256 = self._compute_file_sha256(self.zip_path)

        if cached_sha256 == expected_sha256:
            if not os.path.isdir(self.resources_root):
                self.plugin.log(
                    "Resources ZIP is cached, but unpacked files are missing. Extracting..."
                )
                self._extract_zip()
            return self.resources_root if os.path.isdir(
                self.resources_root
            ) else None, False

        if cached_sha256 is not None:
            if not os.path.isdir(self.resources_root):
                self.plugin.log(
                    "Cached resources ZIP is outdated, but unpacked files are missing. Extracting cached version first..."
                )
                self._extract_zip()

            if os.path.isdir(self.resources_root):
                self.plugin.log(
                    f"Cached resources ZIP SHA256 mismatch (cached={cached_sha256}, expected={expected_sha256}). Using current resources and refreshing in background..."
                )
                return self.resources_root, allow_download

            if not allow_download:
                return None, False

            self.plugin.log(
                "Cached resources ZIP exists but there are no usable extracted files. Downloading replacement synchronously..."
            )
        else:
            if not allow_download:
                return None, False

            self.plugin.log("Cached resources ZIP not found. Downloading new file...")

        zip_data = self._download_bytes(show_bulletins=True)
        if zip_data is None:
            return self.resources_root if os.path.isdir(
                self.resources_root
            ) else None, False

        downloaded_sha256 = self._compute_sha256(zip_data)
        if downloaded_sha256 != expected_sha256:
            raise RuntimeError("Downloaded resources ZIP SHA256 mismatch")

        self._write_zip_file(zip_data)
        self._extract_zip()

        return self.resources_root if os.path.isdir(
            self.resources_root
        ) else None, False

    def _compute_sha256(self, data: bytes) -> str:
        return hashlib.sha256(data).hexdigest().lower()

    def _compute_file_sha256(self, path: str) -> Optional[str]:
        if not os.path.exists(path):
            return None

        try:
            with open(path, "rb") as f:
                return self._compute_sha256(f.read())
        except Exception as e:
            self.plugin.log_exception(
                "Failed to read cached resources ZIP for SHA256", e
            )
            return None

    def _write_zip_file(self, zip_data: bytes):
        with open(self.zip_path, "wb") as f:
            f.write(zip_data)

    def _acquire_download_lock(self, blocking: bool) -> Optional[int]:
        lock_fd = os.open(self.lock_path, os.O_CREAT | os.O_RDWR, 0o666)
        lock_flags = fcntl.LOCK_EX
        if not blocking:
            lock_flags |= fcntl.LOCK_NB

        try:
            fcntl.flock(lock_fd, lock_flags)
            return lock_fd
        except BlockingIOError:
            os.close(lock_fd)
            return None
        except Exception:
            os.close(lock_fd)
            raise

    def _release_download_lock(self, lock_fd: int):
        try:
            fcntl.flock(lock_fd, fcntl.LOCK_UN)
        finally:
            os.close(lock_fd)

    def _download_bytes(self, show_bulletins: bool) -> Optional[bytes]:
        try:
            return self.plugin._download_with_progress(
                url=RESOURCES_URL,
                started_key="download.assets.started",
                completed_key="download.assets.completed",
                show_bulletins=show_bulletins,
            )
        except Exception as e:
            self.plugin.log_exception("Failed to download resources ZIP", e)
            return None

    def _refresh_async(self):
        def worker():
            lock_fd = self._acquire_download_lock(blocking=False)
            if lock_fd is None:
                return

            try:
                expected_sha256 = str(RESOURCES_SHA256).strip().lower()
                cached_sha256 = self._compute_file_sha256(self.zip_path)
                if cached_sha256 == expected_sha256 and os.path.isdir(
                    self.resources_root
                ):
                    return

                zip_data = self._download_bytes(show_bulletins=True)
                if zip_data is None:
                    return

                downloaded_sha256 = self._compute_sha256(zip_data)
                if downloaded_sha256 != expected_sha256:
                    raise RuntimeError("Downloaded resources ZIP SHA256 mismatch")

                self._write_zip_file(zip_data)
                self._extract_zip()
            except Exception as e:
                self.plugin.log_exception("Failed to refresh resources ZIP", e)
            finally:
                self._release_download_lock(lock_fd)

        threading.Thread(
            target=worker, name="tg-streaks-resources-refresh", daemon=True
        ).start()

    def _extract_zip(self):
        staging_root = os.path.join(self.cache_dir, "resources-staging")

        if os.path.isdir(staging_root):
            shutil.rmtree(staging_root)

        os.makedirs(staging_root, exist_ok=True)

        try:
            with zipfile.ZipFile(self.zip_path) as zip_file:
                for member in zip_file.namelist():
                    normalized = member.replace("\\", "/")
                    if not normalized.startswith("resources/"):
                        raise RuntimeError(f"Unexpected ZIP entry: {member}")

                    target_path = os.path.abspath(
                        os.path.join(staging_root, normalized)
                    )
                    if not target_path.startswith(
                        os.path.abspath(staging_root) + os.sep
                    ):
                        raise RuntimeError(f"Unsafe ZIP entry: {member}")

                zip_file.extractall(staging_root)

            extracted_root = os.path.join(staging_root, "resources")
            if not os.path.isdir(extracted_root):
                raise RuntimeError("Resources ZIP does not contain resources/ root")

            if os.path.isdir(self.resources_root):
                shutil.rmtree(self.resources_root)

            os.replace(extracted_root, self.resources_root)
            self.plugin.log("Resources ZIP extracted successfully")
        except Exception as e:
            self.plugin.log_exception("Failed to extract resources ZIP", e)
        finally:
            if os.path.isdir(staging_root):
                shutil.rmtree(staging_root)


class ChatContextMenu:
    TOGGLE_PET_FAB = "togglePetFab"
    REBUILD = "rebuild"
    REBUILD_PET = "rebuildPet"
    CREATE_PET = "createPet"
    TOGGLE_SERVICE_MESSAGES = "serviceMessages.toggle"
    GO_TO_STREAK_START = "goToStreakStart"
    REVIVE_NOW = "reviveNow"
    REVIVE_EXACT = "reviveExact"

    DEBUG_CREATE = "debug.create"
    DEBUG_UPGRADE = "debug.upgrade"
    DEBUG_FREEZE = "debug.freeze"
    DEBUG_KILL = "debug.kill"
    DEBUG_DELETE = "debug.delete"
    DEBUG_DELETE_PET = "debug.deletePet"
    DEBUG_CRASH = "debug.crash"

    MENU_PAYLOAD_DIALOG_KEYS = (
        "dialog_id",
        "dialogId",
        "peer_id",
        "peerId",
        "chat_id",
        "chatId",
        "user_id",
        "userId",
    )
    MENU_PAYLOAD_FRAGMENT_KEYS = (
        "chatActivity",
        "fragment",
        "chatFragment",
        "target",
        "object",
    )

    def __init__(self, plugin: "TgStreaksPlugin"):
        self.plugin = plugin
        self._item_ids: dict[str, str] = {}
        self._callbacks: dict[str, Any] = {}

    @classmethod
    def _menu_items(cls) -> tuple[dict[str, Any], ...]:
        return (
            {
                "key": cls.CREATE_PET,
                "text_key": "menu.chat.create_pet.title",
                "subtext_key": "menu.chat.create_pet.description",
                "icon": "menu_premium_main",
                "priority": 1001,
            },
            {
                "key": cls.REVIVE_NOW,
                "text_key": "menu.chat.restore_streak.title",
                "subtext_key": "menu.chat.restore_streak.description",
                "icon": "msg_reactions",
                "priority": 1000,
            },
            {
                "key": cls.REVIVE_EXACT,
                "text_key": "menu.chat.restore_streak_exact.title",
                "subtext_key": "menu.chat.restore_streak_exact.description",
                "icon": "msg_reactions",
                "priority": 999,
            },
            {
                "key": cls.REBUILD,
                "text_key": "menu.chat.rebuild.streak.title",
                "subtext_key": "menu.chat.rebuild.streak.description",
                "icon": "msg_retry",
                "priority": 998,
            },
            {
                "key": cls.REBUILD_PET,
                "text_key": "menu.chat.rebuild.pet.title",
                "subtext_key": "menu.chat.rebuild.pet.description",
                "icon": "msg_retry",
                "priority": 997,
            },
            {
                "key": cls.GO_TO_STREAK_START,
                "text_key": "menu.chat.open_streak_start.title",
                "subtext_key": "menu.chat.open_streak_start.description",
                "icon": "other_chats",
                "priority": 996,
            },
            {
                "key": cls.TOGGLE_PET_FAB,
                "text_key": "menu.chat.toggle_pet_button.title",
                "subtext_key": "menu.chat.toggle_pet_button.description",
                "icon": "menu_premium_main",
                "priority": 995,
            },
            {
                "key": cls.TOGGLE_SERVICE_MESSAGES,
                "text_key": "menu.chat.toggle_level_messages.title",
                "subtext_key": "menu.chat.toggle_level_messages.description",
                "icon": "msg_settings",
                "priority": 994,
            },
            {
                "key": cls.DEBUG_CREATE,
                "text_key": "menu.debug.create_streak.title",
                "subtext_key": "menu.debug.create_streak.description",
                "priority": 993,
                "debug_only": True,
            },
            {
                "key": cls.DEBUG_UPGRADE,
                "text_key": "menu.debug.upgrade_streak.title",
                "subtext_key": "menu.debug.upgrade_streak.description",
                "priority": 992,
                "debug_only": True,
            },
            {
                "key": cls.DEBUG_FREEZE,
                "text_key": "menu.debug.freeze_streak.title",
                "subtext_key": "menu.debug.freeze_streak.description",
                "priority": 991,
                "debug_only": True,
            },
            {
                "key": cls.DEBUG_KILL,
                "text_key": "menu.debug.kill_streak.title",
                "subtext_key": "menu.debug.kill_streak.description",
                "priority": 990,
                "debug_only": True,
            },
            {
                "key": cls.DEBUG_DELETE,
                "text_key": "menu.debug.delete_streak.title",
                "subtext_key": "menu.debug.delete_streak.description",
                "priority": 989,
                "debug_only": True,
            },
            {
                "key": cls.DEBUG_DELETE_PET,
                "text_key": "menu.debug.delete_pet.title",
                "subtext_key": "menu.debug.delete_pet.description",
                "priority": 988,
                "debug_only": True,
            },
            {
                "key": cls.DEBUG_CRASH,
                "text_key": "menu.debug.crash_plugin.title",
                "subtext_key": "menu.debug.crash_plugin.description",
                "priority": 987,
                "debug_only": True,
            },
        )

    def register(self):
        self.unregister()

        for item in self._menu_items():
            if item.get("debug_only", False) and not DEBUG_MODE:
                continue

            key = str(item["key"])

            try:
                menu_item_args = {
                    "menu_type": MenuItemType.CHAT_ACTION_MENU,
                    "text": self.plugin._t(str(item["text_key"])),
                    "subtext": self.plugin._t(str(item["subtext_key"])),
                    "on_click": lambda payload, button=key: self._on_click(
                        button, payload
                    ),
                    "priority": int(item["priority"]),
                }

                if "icon" in item:
                    menu_item_args["icon"] = item["icon"]

                item_id = self.plugin.add_menu_item(
                    MenuItemData(
                        **menu_item_args,
                    )
                )

                self._item_ids[key] = str(item_id)
            except Exception as e:
                self.plugin.log_exception(
                    f"Failed to register chat context menu item {key}",
                    e,
                )

    def unregister(self):
        for key, item_id in tuple(self._item_ids.items()):
            try:
                self.plugin.remove_menu_item(item_id)
            except Exception as e:
                self.plugin.log_exception(
                    f"Failed to remove chat context menu item {key}",
                    e,
                )

        self._item_ids.clear()
        self._callbacks.clear()

    def _on_click(self, key: str, payload: Any):
        dialog_id = self._extract_dialog_id_from_payload(payload)
        if dialog_id is None:
            self.plugin.log(
                f"Chat context menu click payload missing dialog id for {key}: {payload}"
            )
            self.plugin._show_error(
                self.plugin._t("status.error.chat.detect_current_failed")
            )
            return

        try:
            self._invoke_callback(String(key), Long(dialog_id))
        except Exception as e:
            self.plugin.log_exception(
                f"Failed to execute chat context menu callback {key} for {dialog_id}",
                e,
            )

    def _invoke_callback(self, key: String, value: Long):
        if self.plugin.jvm_plugin.klass is None:
            self.plugin.log(
                f"Chat context menu callback {key} is unavailable: JVM plugin is not loaded"
            )
            return

        try:
            self.plugin.jvm_plugin.klass.getDeclaredMethod(
                String("invokeChatContextMenuCallback"),
                String.getClass(),
                Long.TYPE,
            ).invoke(None, key, value)  # ty:ignore[no-matching-overload]
        except Exception as e:
            self.plugin.log_exception(
                f"Failed to resolve chat context menu callback {key}",
                e,
            )

    def _extract_dialog_id_from_payload(self, payload: Any) -> Optional[int]:
        if payload is None:
            return None

        def parse_dialog_id(value: Any) -> Optional[int]:
            if value is None:
                return None

            try:
                dialog_id = int(value)
            except Exception:
                return None

            return dialog_id or None

        if isinstance(payload, dict):
            payload_getter = payload.get
        else:
            try:
                payload_getter = payload.get
            except Exception:
                payload_getter = None

        if payload_getter is not None:
            for candidate_key in self.MENU_PAYLOAD_DIALOG_KEYS:
                dialog_id = parse_dialog_id(payload_getter(candidate_key))
                if dialog_id is not None:
                    return dialog_id

            for candidate_key in self.MENU_PAYLOAD_FRAGMENT_KEYS:
                value = payload_getter(candidate_key)
                if value is None:
                    continue

                try:
                    dialog_id = parse_dialog_id(value.getDialogId())
                except Exception:
                    dialog_id = None

                if dialog_id is not None:
                    return dialog_id

        try:
            return parse_dialog_id(payload.getDialogId())
        except Exception:
            return None


class SettingsActions:
    REBUILD_ALL = "rebuildAllPrivateChats"
    EXPORT_BACKUP_NOW = "exportBackupNow"
    DELETE_DB_AND_RELOAD = "deleteDbAndReload"

    def __init__(self, plugin: "TgStreaksPlugin"):
        self.plugin = plugin

    def build_settings(self) -> list[Any]:
        return [
            Header(text=self.plugin._t("settings.pet_button.title")),
            Selector(
                key=SETTING_PET_FAB_SIZE_INDEX,
                text=self.plugin._t("settings.pet_button.size.title"),
                default=self.plugin._get_pet_fab_size_index(),
                items=[f"{size} dp" for size in PET_FAB_SIZE_OPTIONS_DP],
                icon="msg_customize",
                on_change=lambda value: self.plugin._on_pet_fab_size_changed(value),
            ),
            Divider(text=self.plugin._t("settings.pet_button.size.description")),
            Header(text=self.plugin._t("settings.streak_tools.title")),
            Text(
                text=self.plugin._t("settings.streak_tools.rebuild_all_chats.title"),
                icon="msg_retry",
                on_click=lambda _: self._on_click(self.REBUILD_ALL),
            ),
            Divider(
                text=self.plugin._t(
                    "settings.streak_tools.rebuild_all_chats.description"
                )
            ),
            Header(text=self.plugin._t("settings.backups.title")),
            Text(
                text=self.plugin._t("settings.backups.export.title"),
                icon="msg_save",
                on_click=lambda _: self._on_click(self.EXPORT_BACKUP_NOW),
            ),
            Text(
                text=self.plugin._t("settings.backups.restore.title"),
                icon="msg_reset",
                on_click=lambda _: self.plugin._show_restore_backup_file_dialog(),
            ),
            Text(
                text=self.plugin._t("settings.backups.reset_database.title"),
                icon="msg_delete",
                on_click=lambda _: self.plugin._schedule_database_reset_reinitialize(),
            ),
            Divider(text=self.plugin._t("settings.backups.description")),
        ]

    def _on_click(self, key: str):
        try:
            self._invoke_callback(String(key))
        except Exception as e:
            self.plugin.log_exception(
                f"Failed to execute settings callback {key}",
                e,
            )

    def _invoke_callback(self, key: String):
        if self.plugin.jvm_plugin.klass is None:
            self.plugin.log(
                f"Settings callback {key} is unavailable: JVM plugin is not loaded"
            )
            return

        try:
            self.plugin.jvm_plugin.klass.getDeclaredMethod(
                String("invokeSettingsActionCallback"),
                String.getClass(),
            ).invoke(None, key)  # ty:ignore[no-matching-overload]
        except Exception as e:
            self.plugin.log_exception(
                f"Failed to resolve settings callback {key}",
                e,
            )


class PluginUpdateChecker:
    def __init__(self, plugin: "TgStreaksPlugin"):
        self.plugin = plugin
        self._stop = threading.Event()
        self._lock = threading.Lock()
        self._inflight = False

    def is_enabled(self) -> bool:
        try:
            return bool(self.plugin.get_setting(SETTING_UPDATE_CHECK_ENABLED, True))
        except Exception:
            return True

    def set_enabled(self, enabled: bool):
        enabled = bool(enabled)

        try:
            self.plugin.set_setting(SETTING_UPDATE_CHECK_ENABLED, enabled)
        except Exception as e:
            self.plugin.log_exception("Failed to persist update check setting", e)

        if enabled:
            self.start()
            return

        self.stop()

    def stop(self):
        self._stop.set()

    def start(self):
        if not self.is_enabled():
            self.plugin.log("Update check skipped: disabled in plugin settings")
            return

        with self._lock:
            if self._inflight:
                return
            self._inflight = True

        self._stop.clear()

        def worker():
            try:
                latest_tag = self._fetch_latest_release_tag()
                if latest_tag is None:
                    self.plugin.log(
                        "Update check skipped: latest release tag_name is missing"
                    )
                    return

                latest_version = self._normalize_version_tag(latest_tag)
                current_version = self._normalize_version_tag(__version__)

                if len(latest_version) == 0:
                    self.plugin.log(
                        "Update check skipped: latest release version is empty"
                    )
                    return

                if latest_version == current_version:
                    self.plugin.log(f"Plugin is up to date: {current_version}")
                    return

                if self._stop.is_set():
                    return

                self.plugin.log(
                    f"Plugin update available: current={__version__}, latest={latest_tag}"
                )
                self._show_update_bulletin(latest_tag)
            except Exception as e:
                self.plugin.log_exception("Plugin update check failed", e)
            finally:
                with self._lock:
                    self._inflight = False

        threading.Thread(target=worker, daemon=True).start()

    def _normalize_version_tag(self, value: Optional[str]) -> str:
        if value is None:
            return ""

        normalized = str(value).strip()
        if len(normalized) == 0:
            return ""

        return normalized.removeprefix("v").removeprefix("V")

    def _fetch_latest_release_tag(self) -> Optional[str]:
        response = requests.get(
            PLUGIN_UPDATE_API_URL,
            headers={
                "Accept": "application/vnd.github+json",
                "X-GitHub-Api-Version": "2022-11-28",
            },
            timeout=UPDATE_CHECK_TIMEOUT_SECONDS,
        )
        response.raise_for_status()
        payload = cast("dict[str, Any]", response.json())
        tag_name = payload.get("tag_name")

        if tag_name is None:
            return None

        normalized = str(tag_name).strip()
        if len(normalized) == 0:
            return None

        return normalized

    def _show_update_bulletin(self, latest_version: str):
        text = self.plugin._t(
            "update.available.message",
            current=__version__,
            latest=latest_version,
        )
        button_text = self.plugin._t("update.available.action")

        def show():
            if self._stop.is_set():
                return

            try:
                BulletinHelper.show_with_button(
                    text,
                    R_tg.raw.ic_download,
                    button_text,
                    lambda: self.plugin._open_telegram_url(PLUGIN_UPDATE_TG_URL),
                    duration=int(BulletinHelper.DURATION_PROLONG),
                )
            except Exception as e:
                self.plugin.log_exception("Failed to show update bulletin", e)
                self.plugin._show_info(text)

        run_on_ui_thread(show)


class TgStreaksPlugin(BasePlugin):
    _reinitialize_lock = threading.Lock()

    settings_actions: SettingsActions

    def log(self, message: Any):
        text = str(message)
        super().log(text)

        try:
            Log.i(LOGCAT_TAG, text)
        except Exception:
            pass

    def log_exception(self, message: str, exception: BaseException):
        text = f"{message}: {exception}"
        self.log(text)

        if DEBUG_MODE:
            try:
                Log.e(LOGCAT_TAG, text)
            except Exception:
                pass

        for chunk in traceback.format_exception(
            type(exception),
            exception,
            exception.__traceback__,
        ):
            for line in chunk.rstrip().splitlines():
                if len(line) > 0:
                    self.log(line)
                    if DEBUG_MODE:
                        try:
                            Log.e(LOGCAT_TAG, line)
                        except Exception:
                            pass

    def create_settings(self) -> list[Any]:
        if not hasattr(self, "settings_actions"):
            self.settings_actions = SettingsActions(self)

        return [
            Header(text=self._t("settings.updates.title")),
            Switch(
                key=SETTING_UPDATE_CHECK_ENABLED,
                text=self._t("settings.updates.auto_check.title"),
                default=self._is_update_check_enabled(),
                subtext=self._t("settings.updates.auto_check.description"),
                icon="msg_retry",
                on_change=lambda value: self._on_update_check_setting_changed(value),
            ),
            *self.settings_actions.build_settings(),
        ]

    def _show_info(self, message: str):
        run_on_ui_thread(lambda: BulletinHelper.show_info(message))

    def _show_error(self, message: str):
        run_on_ui_thread(lambda: BulletinHelper.show_error(message))

    def _show_success(self, message: str):
        run_on_ui_thread(lambda: BulletinHelper.show_success(message))

    def _show_download_progress(self, title: str, subtitle: str):
        def show():
            try:
                BulletinHelper.show_two_line(title, subtitle, R_tg.raw.ic_download)
            except Exception as e:
                self.log_exception("Failed to show download progress bulletin", e)
                self._show_info(f"{title}\n{subtitle}")

        run_on_ui_thread(show)

    def _format_download_size(self, byte_count: float) -> str:
        value = float(max(byte_count, 0.0))
        units = ("B", "KB", "MB", "GB")
        unit_index = 0

        while value >= 1024.0 and unit_index < len(units) - 1:
            value /= 1024.0
            unit_index += 1

        if unit_index == 0:
            return f"{int(value)} {units[unit_index]}"

        return f"{value:.1f} {units[unit_index]}"

    def _format_eta(self, seconds: float) -> str:
        total_seconds = int(max(seconds, 0.0))
        minutes, secs = divmod(total_seconds, 60)
        hours, mins = divmod(minutes, 60)

        if hours > 0:
            return f"{hours}h {mins:02d}m"
        if minutes > 0:
            return f"{minutes}m {secs:02d}s"
        return f"{secs}s"

    def _download_with_progress(
        self,
        url: str,
        started_key: str,
        completed_key: str,
        show_bulletins: bool,
    ) -> Optional[bytes]:
        if show_bulletins:
            self._show_info(self._t(started_key))

        response = requests.get(url, timeout=10, stream=True)
        if response.status_code != 200:
            self.log(f"Failed to download {url}: {response.status_code}")
            return None

        total_bytes = int(response.headers.get("content-length", "0") or "0")
        downloaded = 0
        chunks: list[bytes] = []
        started_at = time.monotonic()
        last_progress_at = 0.0

        for chunk in response.iter_content(chunk_size=64 * 1024):
            if not chunk:
                continue

            chunks.append(cast("bytes", chunk))
            downloaded += len(chunk)

            if not show_bulletins:
                continue

            now = time.monotonic()
            if total_bytes > 0 and downloaded >= total_bytes:
                continue

            if (now - last_progress_at) < 0.8:
                continue

            elapsed = max(now - started_at, 0.001)
            speed = downloaded / elapsed
            title = self._t(started_key)

            if total_bytes > 0 and speed > 1.0:
                remaining_seconds = (total_bytes - downloaded) / speed
                subtitle = self._t(
                    "download.progress.known_total",
                    percent=str(int(downloaded * 100 / total_bytes)),
                    downloaded=self._format_download_size(downloaded),
                    total=self._format_download_size(total_bytes),
                    eta=self._format_eta(remaining_seconds),
                )
            else:
                subtitle = self._t(
                    "download.progress.unknown_total",
                    downloaded=self._format_download_size(downloaded),
                )

            self._show_download_progress(title, subtitle)
            last_progress_at = now

        payload = b"".join(chunks)

        if show_bulletins:
            self._show_success(self._t(completed_key))

        return payload

    def _is_update_check_enabled(self) -> bool:
        try:
            return bool(self.get_setting(SETTING_UPDATE_CHECK_ENABLED, True))
        except Exception:
            return True

    def _get_pet_fab_size_index(self) -> int:
        default_index = 1

        try:
            raw_value = int(self.get_setting(SETTING_PET_FAB_SIZE_INDEX, default_index))
        except Exception:
            return default_index

        return max(0, min(raw_value, len(PET_FAB_SIZE_OPTIONS_DP) - 1))

    def _get_pet_fab_size_dp(self) -> int:
        return PET_FAB_SIZE_OPTIONS_DP[self._get_pet_fab_size_index()]

    def _apply_pet_fab_size_dp(self, size_dp: int):
        if self.jvm_plugin.klass is None:
            self.log("Pet FAB size update skipped: JVM plugin is not loaded")
            return

        try:
            self.jvm_plugin.klass.getDeclaredMethod(
                String("setPetFabSizeDp"),
                Integer.TYPE,
            ).invoke(
                None,
                Integer(int(size_dp)),
            )
        except Exception as e:
            self.log_exception("Failed to apply pet FAB size", e)

    def _on_pet_fab_size_changed(self, value: int):
        size_index = max(0, min(int(value), len(PET_FAB_SIZE_OPTIONS_DP) - 1))

        try:
            self.set_setting(SETTING_PET_FAB_SIZE_INDEX, size_index)
        except Exception as e:
            self.log_exception("Failed to persist pet FAB size setting", e)

        self._apply_pet_fab_size_dp(PET_FAB_SIZE_OPTIONS_DP[size_index])

    def _on_update_check_setting_changed(self, enabled: bool):
        enabled = bool(enabled)

        if hasattr(self, "update_checker"):
            self.update_checker.set_enabled(enabled)
            return

        try:
            self.set_setting(SETTING_UPDATE_CHECK_ENABLED, enabled)
        except Exception as e:
            self.log_exception("Failed to persist update check setting", e)

    def _get_app_language_code(self) -> str:
        try:
            locale_controller = LocaleController.getInstance()
        except Exception:
            locale_controller = None

        lang_code = None

        if locale_controller is not None:
            try:
                locale_info = locale_controller.getCurrentLocaleInfo()
            except Exception:
                locale_info = None

            if locale_info is not None:
                try:
                    lang_code = cast("str", locale_info.getLangCode())
                except Exception:
                    lang_code = None

                if lang_code is None or len(str(lang_code)) == 0:
                    try:
                        lang_code = cast("str", locale_info.shortName)
                    except Exception:
                        lang_code = None

            if lang_code is None or len(str(lang_code)) == 0:
                try:
                    current_locale = locale_controller.getCurrentLocale()
                except Exception:
                    current_locale = None

                if current_locale is not None:
                    try:
                        lang_code = cast("str", current_locale.getLanguage())
                    except Exception:
                        lang_code = None

        if lang_code is None:
            return "en"

        normalized = str(lang_code).strip().lower()
        if len(normalized) == 0:
            return "en"

        normalized = normalized.replace("-", "_")
        base_code = normalized.split("_", 1)[0]
        if len(base_code) == 0:
            return "en"

        return base_code

    def _t(self, key: str, **kwargs: Any) -> str:
        values = I18N_STRINGS.get(key, None)
        if values is None:
            text = key
        else:
            lang = self._get_app_language_code()
            text = values.get(lang) or values.get("en") or key

        try:
            return str(text).format(**kwargs)
        except Exception:
            return str(text)

    def _resolve_popup_context(self):
        try:
            fragment = get_last_fragment()
        except Exception:
            fragment = None

        context = None

        if fragment is not None:
            try:
                context = fragment.getParentActivity()
            except Exception:
                context = None

            if context is None:
                try:
                    context = fragment.getContext()
                except Exception:
                    context = None

        if context is None:
            try:
                context = ApplicationLoader.applicationContext
            except Exception:
                context = None

        return context

    def _open_telegram_url(self, url: str):
        def open_url():
            try:
                context = self._resolve_popup_context()
                if context is None:
                    raise RuntimeError("no context")

                intent = Intent()
                intent.setAction(String(Intent.ACTION_VIEW))
                intent.setData(Uri.parse(String(url)))
                intent.setPackage(String(context.getPackageName()))
                intent.addFlags(int(Intent.FLAG_ACTIVITY_NEW_TASK))
                context.startActivity(intent)
            except Exception as e:
                self.log_exception(f"Failed to open Telegram url {url}", e)
                self._show_error(self._t("status.error.update.open_link_failed"))

        run_on_ui_thread(open_url)

    def _register_streak_levels(self):
        if self.jvm_plugin.klass is None:
            return

        try:
            register_method = self.jvm_plugin.klass.getDeclaredMethod(
                String("registerStreakLevel"),
                Integer.TYPE,
                Color.getClass(),
                Long.TYPE,
                String.getClass(),  # ty:ignore[unresolved-attribute]
            )

            for level in StreakLevels:
                streak_level = cast("StreakLevel", level.value)
                register_method.invoke(
                    None,  # ty:ignore[invalid-argument-type]
                    Integer(streak_level.length),
                    streak_level.text_color,
                    Long(streak_level.document_id),
                    String(streak_level.popup_resource_name),
                )

            self.log(f"Registered {len(StreakLevels)} streak levels")
        except Exception as e:
            self.log_exception("Failed to register streak levels", e)

    def _register_streak_pet_levels(self):
        if self.jvm_plugin.klass is None:
            return

        try:
            register_method = self.jvm_plugin.klass.getDeclaredMethod(
                String("registerStreakPetLevel"),
                Integer.TYPE,
                String.getClass(),  # ty:ignore[unresolved-attribute]
                String.getClass(),  # ty:ignore[unresolved-attribute]
                String.getClass(),  # ty:ignore[unresolved-attribute]
                String.getClass(),  # ty:ignore[unresolved-attribute]
                String.getClass(),  # ty:ignore[unresolved-attribute]
                String.getClass(),  # ty:ignore[unresolved-attribute]
                String.getClass(),  # ty:ignore[unresolved-attribute]
            )

            for level in StreakPetLevels:
                streak_pet_level = cast("StreakPetLevel", level.value)
                register_method.invoke(
                    None,  # ty:ignore[invalid-argument-type]
                    Integer(streak_pet_level.max_points),
                    String(streak_pet_level.image_resource_path),
                    String(streak_pet_level.gradient_start),
                    String(streak_pet_level.gradient_end),
                    String(streak_pet_level.pet_start),
                    String(streak_pet_level.pet_end),
                    String(streak_pet_level.accent),
                    String(streak_pet_level.accent_secondary),
                )

            self.log(f"Registered {len(StreakPetLevels)} streak pet levels")
        except Exception as e:
            self.log_exception("Failed to register streak pet levels", e)

    def _finalize_jvm_plugin_inject(self):
        if self.jvm_plugin.klass is None:
            return

        try:
            self.jvm_plugin.klass.getDeclaredMethod(String("finalizeInject")).invoke(
                None  # ty:ignore[invalid-argument-type]
            )
            self.log("JVM plugin finalizeInject completed")
        except Exception as e:
            self.log_exception("Failed to finalize JVM plugin inject", e)

    def _load_jvm_plugin(self):
        self.jvm_plugin = JvmPluginBridge(self)
        self.jvm_plugin.load()

        if self.jvm_plugin.klass is None:
            return

        try:
            build_date = self.jvm_plugin.klass.getDeclaredMethod(
                String("getBuildDate")
            ).invoke(None)  # ty:ignore[invalid-argument-type]
            self.log(f"Loading JVM plugin {build_date}")
        except Exception as e:
            self.log_exception("Failed to infer JVM plugin version", e)

        try:
            ref = self

            class Logger(dynamic_proxy(ValueCallback)):
                def onReceiveValue(self, var1):
                    ref.log(str(var1))

            class TranslationResolver(dynamic_proxy(Function)):
                def apply(self, t: String):
                    if t is None:
                        return ""
                    return ref._t(str(t))

            self.jvm_plugin.klass.getDeclaredMethod(
                String("inject"),
                ValueCallback.getClass(),  # ty:ignore[unresolved-attribute]
                Function.getClass(),  # ty:ignore[unresolved-attribute]
                String.getClass(),  # ty:ignore[unresolved-attribute]
            ).invoke(
                None,
                Logger(),
                TranslationResolver(),
                String(self.resources_bridge.resources_root),
            )  # ty:ignore[no-matching-overload]
            self.log("JVM plugin injected successfully")
            self._apply_pet_fab_size_dp(self._get_pet_fab_size_dp())
        except Exception as e:
            self.log_exception("Failed to inject JVM plugin", e)

    def _database_file_paths(self) -> list[str]:
        base_path = ApplicationLoader.applicationContext.getDatabasePath(
            String("tg-streaks")
        ).getAbsolutePath()
        return [
            str(base_path),
            f"{base_path}-wal",
            f"{base_path}-shm",
            f"{base_path}-journal",
        ]

    def _backups_dir(self) -> str:
        downloads_dir = Environment.getExternalStoragePublicDirectory(
            Environment.DIRECTORY_DOWNLOADS
        ).getAbsolutePath()
        backups_dir = os.path.join(str(downloads_dir), "tg-streaks")
        os.makedirs(backups_dir, exist_ok=True)
        return backups_dir

    def _list_backup_files(self) -> list[str]:
        backups_dir = self._backups_dir()
        files = []

        try:
            for name in os.listdir(backups_dir):
                path = os.path.join(backups_dir, name)
                if os.path.isfile(path) and name.endswith(".sqlite3"):
                    files.append(path)
        except Exception as e:
            self.log_exception("Failed to list backup files", e)
            return []

        files.sort(key=os.path.getmtime, reverse=True)
        return files

    def _show_restore_backup_file_dialog(self):
        backup_files = self._list_backup_files()
        if len(backup_files) == 0:
            self._show_error(self._t("status.error.backup.not_found"))
            return

        def show():
            try:
                fragment = get_last_fragment()
            except Exception:
                fragment = None

            if fragment is None:
                self._show_error(
                    self._t("status.error.backup.apply_failed").format(
                        reason="No UI context"
                    )
                )
                return

            names = [os.path.basename(path) for path in backup_files]

            try:

                class BackupClickListener(
                    dynamic_proxy(DialogInterface.OnClickListener)
                ):
                    def onClick(self, _dialog, which):
                        self_outer._schedule_restore_backup_reinitialize(
                            backup_files[int(which)]
                        )

                self_outer = self
                fragment.showDialog(
                    AlertDialog.Builder(fragment.getContext())
                    .setTitle(self._t("dialog.backup_restore.title"))
                    .setItems(
                        jarray(String)([String(name) for name in names]),
                        BackupClickListener(),
                    )
                    .create()
                )
            except Exception as e:
                self.log_exception("Failed to show restore backup dialog", e)
                self._show_error(
                    self._t("status.error.backup.apply_failed").format(reason=str(e))
                )

        run_on_ui_thread(show)

    def _schedule_restore_backup_reinitialize(self, backup_path: str):
        reason = (
            f"Backup restore requested from settings: {os.path.basename(backup_path)}"
        )

        if not self._reinitialize_lock.acquire(blocking=False):
            self.log(f"Skipped duplicate backup restore request: {reason}")
            return

        def worker():
            try:
                self.log(f"Starting backup restore: {reason}")

                if not os.path.isfile(backup_path):
                    self._show_error(self._t("status.error.backup.not_found"))
                    return

                try:
                    self.on_plugin_unload()
                except BaseException as e:
                    self.log_exception(
                        "Failed during plugin unload before backup restore", e
                    )

                try:
                    target_path = self._database_file_paths()[0]
                    os.makedirs(os.path.dirname(target_path), exist_ok=True)

                    for path in self._database_file_paths()[1:]:
                        if os.path.exists(path):
                            os.remove(path)

                    if os.path.exists(target_path):
                        os.remove(target_path)

                    shutil.copy2(backup_path, target_path)
                    self.log(f"Database restored from backup: {backup_path}")
                except BaseException as e:
                    self.log_exception("Failed to apply backup file", e)
                    self._show_error(
                        self._t("status.error.backup.apply_failed").format(
                            reason=str(e)
                        )
                    )
                    return

                try:
                    self.on_plugin_load()
                except BaseException as e:
                    self.log_exception(
                        "Failed during plugin load after backup restore", e
                    )
                    return

                self._show_success(
                    self._t(
                        "status.success.backup.imported",
                        name=os.path.basename(backup_path),
                    )
                )
                self.log("Backup restore and plugin reinitialization completed")
            finally:
                self._reinitialize_lock.release()

        threading.Thread(
            target=worker,
            name="tg-streaks-backup-restore-reinitialize",
            daemon=True,
        ).start()

    def _schedule_database_reset_reinitialize(self):
        reason = "Database reset requested from settings"

        if not self._reinitialize_lock.acquire(blocking=False):
            self.log(f"Skipped duplicate database reset request: {reason}")
            return

        def worker():
            try:
                self.log(f"Starting database reset: {reason}")

                try:
                    self.on_plugin_unload()
                except BaseException as e:
                    self.log_exception("Failed during plugin unload before DB reset", e)

                try:
                    for path in self._database_file_paths():
                        if os.path.exists(path):
                            os.remove(path)
                            self.log(f"Deleted database file: {path}")
                except BaseException as e:
                    self.log_exception("Failed to delete plugin database", e)
                    self._show_error(
                        self._t("status.error.database.delete_failed").format(
                            reason=str(e)
                        )
                    )
                    return

                self._show_success(self._t("status.success.database.reset_started"))

                try:
                    self.on_plugin_load()
                except BaseException as e:
                    self.log_exception("Failed during plugin load after DB reset", e)
                    return

                self.log("Database reset and plugin reinitialization completed")
            finally:
                self._reinitialize_lock.release()

        threading.Thread(
            target=worker,
            name="tg-streaks-db-reset-reinitialize",
            daemon=True,
        ).start()

    def on_plugin_load(self):
        self.resources_bridge = ZipResourcesBridge(self)
        self._load_jvm_plugin()

        self._register_streak_levels()
        self._register_streak_pet_levels()

        self.settings_actions = SettingsActions(self)
        self.chat_context_menu = ChatContextMenu(self)
        self.chat_context_menu.register()

        self._finalize_jvm_plugin_inject()

        self.update_checker = PluginUpdateChecker(self)
        self.update_checker.start()

        threading.Thread(
            target=self.resources_bridge.load,
            name="tg-streaks-resources-init",
            daemon=True,
        ).start()

    def on_plugin_unload(self):
        try:
            self.update_checker.stop()
        except Exception:
            pass

        try:
            self.chat_context_menu.unregister()
        except Exception as e:
            self.log_exception("Failed to unregister chat context menu", e)

        if self.jvm_plugin.klass is None:
            return

        try:
            self.jvm_plugin.klass.getDeclaredMethod(String("eject")).invoke(None)  # ty:ignore[invalid-argument-type]
            self.log("JVM plugin ejected successfully")
        except Exception as e:
            self.log_exception("Failed to eject JVM plugin", e)

        self.jvm_plugin.klass = None
