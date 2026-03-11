import hashlib
import json
import os
import threading
import time
from dataclasses import dataclass
from enum import Enum
from functools import wraps
from typing import (
    Callable,
    Concatenate,
    Optional,
    ParamSpec,
    TypeVar,
    cast,
)

import requests
from android.app import Dialog
from android.content import DialogInterface, Intent
from android.graphics import Color, Typeface
from android.graphics.drawable import ColorDrawable, GradientDrawable
from android.net import Uri
from android.view import Gravity, WindowManager
from android.webkit import ValueCallback
from android.widget import ImageView, LinearLayout, TextView
from android_utils import jclass, log, run_on_ui_thread
from base_plugin import (
    BasePlugin,
    HookResult,
    MenuItemData,
    MenuItemType,
    MethodHook,
)
from client_utils import (
    get_account_instance,
    get_connections_manager,
    get_last_fragment,
    get_messages_controller,
)
from dalvik.system import InMemoryDexClassLoader
from hook_utils import get_private_field
from java import (
    dynamic_proxy,
    jarray,
    jlong,
)
from java.lang import (
    Boolean,
    Class,
    Integer,
    Long,
    Object,
    String,
    System,
)
from java.util import Calendar, TimeZone
from java.util.concurrent import ConcurrentHashMap
from java.util.function import Function
from org.telegram.messenger import (
    AndroidUtilities,
    ApplicationLoader,
    DialogObject,
    LocaleController,
    MessagesController,
    NotificationCenter,
    UserConfig,
    UserObject,
)
from org.telegram.messenger import R as R_tg  # ty:ignore[unresolved-import]
from org.telegram.tgnet import TLRPC, RequestDelegate, TLObject
from org.telegram.ui import ChatActivity, LaunchActivity
from org.telegram.ui.Components import AnimatedEmojiDrawable
from typing_extensions import Any
from ui.bulletin import BulletinHelper
from ui.settings import Divider, Header, Switch, Text

__id__ = "tg-streaks"
__name__ = "Streaks"
__description__ = "Analog for TikTok streaks for Telegram"
__author__ = "@n08i40k"
__version__ = "1.4.0"
__icon__ = "exteraPlugins/0"
__min_version__ = "12.2.10"

DEBUG_MODE = False

REPO_OWNER = "n08i40k"
REPO_NAME = __id__

SECONDS_IN_DAY = 86400
DAY_CHECK_TIMEOUT_SECONDS = 15
DAY_CHECK_TIMEOUT_RETRIES = 5
DAY_CHECK_REQUEST_DELAY_SECONDS = 0.35
DAY_CHECK_RETRY_DELAY_SECONDS = 0.8
DAY_CHECK_MESSAGE_TABLE_CANDIDATES = ("messages_v2", "messages")

DEX_URL = f"https://github.com/{REPO_OWNER}/{REPO_NAME}/releases/download/{__version__}/classes.dex"
DEX_SHA256 = "013c15fa3aa795553160ffea1b69b2429b44b58c2cc26530daaeceb0ce012b65"

PLUGIN_UPDATE_TG_URL = "tg://resolve?domain=n08i40k_extera&post=3"
UPDATE_CHECK_TIMEOUT_SECONDS = 6
SETTING_UPDATE_CHECK_ENABLED = "update_check_enabled"

_day_check_messages_table_name: Optional[str] = None
_day_check_messages_table_resolved = False

I18N_STRINGS: dict[str, dict[str, str]] = {
    "settings_updates": {"en": "Updates", "ru": "Обновления"},
    "settings_check_updates": {
        "en": "Check for plugin updates",
        "ru": "Проверять обновления плагина",
    },
    "settings_check_updates_hint": {
        "en": "Checks the latest GitHub release in the background when the plugin loads.",
        "ru": "При загрузке плагина в фоне проверяет последний релиз на GitHub.",
    },
    "settings_streak_tools": {"en": "Streak Tools", "ru": "Инструменты стрика"},
    "settings_force_check_all_private_chats": {
        "en": "Force check all private chats",
        "ru": "Принудительно проверить все личные чаты",
    },
    "settings_only_private_hint": {
        "en": "Only private dialogs with real users are checked. Bots and groups are skipped.",
        "ru": "Проверяются только личные диалоги с реальными пользователями. Боты и группы пропускаются.",
    },
    "settings_db_backups": {"en": "Database Backups", "ru": "Резервные копии базы"},
    "settings_export_backup_now": {
        "en": "Export backup now",
        "ru": "Экспортировать бэкап сейчас",
    },
    "settings_import_latest_backup": {
        "en": "Import latest backup",
        "ru": "Импортировать последний бэкап",
    },
    "settings_db_backups_hint": {
        "en": "Daily backups are created automatically. Import replaces current streak database.",
        "ru": "Ежедневные бэкапы создаются автоматически. Импорт заменяет текущую базу стриков.",
    },
    "err_users_db_not_ready": {
        "en": "Users database is not ready yet",
        "ru": "База пользователей ещё не готова",
    },
    "err_cannot_detect_current_chat": {
        "en": "Cannot detect current chat",
        "ru": "Не удалось определить текущий чат",
    },
    "err_force_check_failed_logs": {
        "en": "Force check failed. See plugin logs.",
        "ru": "Принудительная проверка завершилась ошибкой. Смотрите логи плагина.",
    },
    "err_cannot_open_chat_context": {
        "en": "Cannot open chat context for jump",
        "ru": "Не удалось открыть контекст чата для перехода",
    },
    "err_failed_jump_to_streak_start": {
        "en": "Failed to jump to streak start",
        "ru": "Не удалось перейти к началу стрика",
    },
    "err_failed_export_backup": {
        "en": "Failed to export backup",
        "ru": "Не удалось экспортировать бэкап",
    },
    "err_backup_export_failed": {
        "en": "Backup export failed",
        "ru": "Ошибка экспорта бэкапа",
    },
    "err_backup_import_failed": {
        "en": "Backup import failed",
        "ru": "Ошибка импорта бэкапа",
    },
    "err_failed_open_update_link": {
        "en": "Failed to open update link",
        "ru": "Не удалось открыть ссылку обновления",
    },
    "ok_backup_exported": {
        "en": "Backup exported: {name}",
        "ru": "Бэкап экспортирован: {name}",
    },
    "ok_backup_imported": {
        "en": "Backup imported: {name}",
        "ru": "Бэкап импортирован: {name}",
    },
    "ok_jumped_to_streak_start_message": {
        "en": "Jumped to streak start message",
        "ru": "Переход к сообщению начала стрика выполнен",
    },
    "ok_debug_streak_set_3": {
        "en": "Debug: streak set to 3 days",
        "ru": "Debug: стрик установлен на 3 дня",
    },
    "ok_debug_streak_marked_dead": {
        "en": "Debug: streak marked as dead",
        "ru": "Debug: стрик помечен как мёртвый",
    },
    "ok_debug_streak_upgraded": {
        "en": "Debug: streak upgraded to {days} days",
        "ru": "Debug: стрик улучшен до {days} дней",
    },
    "info_private_user_only": {
        "en": "This action works only for private user chats",
        "ru": "Это действие работает только в личных чатах с пользователями",
    },
    "info_force_check_already_running": {
        "en": "Force check is already running",
        "ru": "Принудительная проверка уже выполняется",
    },
    "info_force_check_started_chat": {
        "en": "Force check started for this chat",
        "ru": "Принудительная проверка запущена для этого чата",
    },
    "info_force_check_started_all": {
        "en": "Force check started",
        "ru": "Принудительная проверка запущена",
    },
    "info_no_streak_record_for_chat": {
        "en": "No streak record for this chat",
        "ru": "Для этого чата нет записи стрика",
    },
    "info_searching_streak_start_message": {
        "en": "Searching streak start message...",
        "ru": "Ищу сообщение начала стрика...",
    },
    "info_exact_start_message_not_found": {
        "en": "Exact start message not found, jumped to streak start day",
        "ru": "Точное сообщение начала не найдено, выполнен переход к дню начала стрика",
    },
    "info_debug_mode_disabled": {
        "en": "Debug mode is disabled",
        "ru": "Debug-режим отключён",
    },
    "info_debug_private_user_only": {
        "en": "Debug actions work only for private user chats",
        "ru": "Debug-действия работают только в личных чатах с пользователями",
    },
    "info_debug_streak_already_max": {
        "en": "Debug: streak is already at max level",
        "ru": "Debug: стрик уже на максимальном уровне",
    },
    "popup_streak_ended_title": {"en": "Streak Ended", "ru": "Стрик завершён"},
    "popup_streak_started_title": {"en": "Streak Started", "ru": "Стрик начат"},
    "popup_streak_upgraded_title": {"en": "{days} DAYS!!!", "ru": "{days} ДНЕЙ!!!"},
    "popup_streak_ended_subtitle": {
        "en": "Your streak with {name} ended after {days} days!",
        "ru": "Ваш стрик с {name} завершился после {days} дней!",
    },
    "popup_streak_started_subtitle": {
        "en": "You started a streak with {name}!",
        "ru": "Вы начали стрик с {name}!",
    },
    "popup_streak_upgraded_subtitle": {
        "en": "Your streak with {name} leveled up!",
        "ru": "Ваш стрик с {name} повысил уровень!",
    },
    "dex_sheet_feature_how_title": {
        "en": "How does this work?",
        "ru": "Как это работает?",
    },
    "dex_sheet_feature_how_subtitle": {
        "en": "After three days of communication in a row, you will have a streak that improves depending on its duration!",
        "ru": "После трёх дней общения подряд у вас появится стрик, который улучшается в зависимости от длительности!",
    },
    "dex_sheet_feature_levels_title": {
        "en": "What levels are there?",
        "ru": "Какие уровни есть?",
    },
    "dex_sheet_feature_levels_subtitle": {
        "en": "There are levels for 3, 10, 30, 100, and 200 consecutive days of communication. A pop-up will appear when your streak level improves.",
        "ru": "Есть уровни за 3, 10, 30, 100 и 200 дней общения подряд. При повышении уровня стрика появится всплывающее окно.",
    },
    "dex_sheet_feature_keep_title": {
        "en": "Don't forget about it ;)",
        "ru": "Не забывайте про него ;)",
    },
    "dex_sheet_feature_keep_subtitle": {
        "en": "If you don't manage to message each other within 24 hours, the streak will be terminated without the possibility of recovery.",
        "ru": "Если вы не успеете написать друг другу в течение 24 часов, стрик завершится без возможности восстановления.",
    },
    "dex_sheet_feature_incorrect_title": {
        "en": "Streak duration is incorrect?",
        "ru": "Длительность стрика неверная?",
    },
    "dex_sheet_feature_incorrect_subtitle": {
        "en": 'This can be fixed! Click the "Force check this chat" button to recalculate the streak length.',
        "ru": 'Это можно исправить! Нажмите кнопку "Принудительно проверить этот чат", чтобы пересчитать длину стрика.',
    },
    "dex_sheet_title": {
        "en": "You and {name} have been on a streak for {days} days now!",
        "ru": "У Вас и {name} стрик уже более {days} дней!",
    },
    "dex_sheet_subtitle": {
        "en": "Keep chatting and keep the streak going **:P**",
        "ru": "Продолжайте общаться и не прерывайте стрик **:P**",
    },
    "menu_force_check_chat_text": {
        "en": "Force check this chat",
        "ru": "Принудительно проверить этот чат",
    },
    "menu_force_check_chat_subtext": {
        "en": "Rebuild streak for this private chat",
        "ru": "Пересобрать стрик для этого личного чата",
    },
    "menu_go_to_streak_start_text": {
        "en": "Go to streak start",
        "ru": "Перейти к началу стрика",
    },
    "menu_go_to_streak_start_subtext": {
        "en": "Jump to the day when this streak started",
        "ru": "Перейти к дню, когда начался этот стрик",
    },
    "menu_debug_create_streak_text": {
        "en": "[DEBUG] Create 3-day streak",
        "ru": "[DEBUG] Создать стрик на 3 дня",
    },
    "menu_debug_create_streak_subtext": {
        "en": "Set current chat streak to 3 days",
        "ru": "Установить стрик текущего чата на 3 дня",
    },
    "menu_debug_kill_streak_text": {
        "en": "[DEBUG] Kill streak",
        "ru": "[DEBUG] Убить стрик",
    },
    "menu_debug_kill_streak_subtext": {
        "en": "Force streak death for current chat",
        "ru": "Принудительно завершить стрик в текущем чате",
    },
    "menu_debug_upgrade_streak_text": {
        "en": "[DEBUG] Upgrade streak",
        "ru": "[DEBUG] Улучшить стрик",
    },
    "menu_debug_upgrade_streak_subtext": {
        "en": "Upgrade current chat to next streak level",
        "ru": "Повысить стрик текущего чата до следующего уровня",
    },
    "force_check_day_mode_today": {"en": "today pass", "ru": "проверка сегодня"},
    "force_check_day_mode_yesterday": {"en": "yesterday pass", "ru": "проверка вчера"},
    "force_check_day_state_active": {"en": "active", "ru": "активно"},
    "force_check_day_state_stop": {"en": "stop", "ru": "стоп"},
    "force_check_day_progress_chat": {
        "en": "Force check day progress: {days_checked} days (offset={day_offset}, {state}, {mode})",
        "ru": "Прогресс проверки по дням: {days_checked} дней (смещение={day_offset}, {state}, {mode})",
    },
    "force_check_day_progress_all": {
        "en": "Force check day progress: {days_checked} days for {peer_name} [{checked_chats}/{total_chats}] (offset={day_offset}, {state}, {mode})",
        "ru": "Прогресс проверки по дням: {days_checked} дней для {peer_name} [{checked_chats}/{total_chats}] (смещение={day_offset}, {state}, {mode})",
    },
    "force_check_progress_zero": {
        "en": "Force check progress: 0/0",
        "ru": "Прогресс проверки: 0/0",
    },
    "force_check_progress": {
        "en": "Force check progress: {checked}/{total} (updated={updated}, removed={removed}, unchanged={unchanged})",
        "ru": "Прогресс проверки: {checked}/{total} (обновлено={updated}, удалено={removed}, без изменений={unchanged})",
    },
    "force_check_summary_chat": {
        "en": "Force check done for this chat: checked={checked}, updated={updated}, removed={removed}, unchanged={unchanged}",
        "ru": "Проверка этого чата завершена: проверено={checked}, обновлено={updated}, удалено={removed}, без изменений={unchanged}",
    },
    "force_check_summary_all": {
        "en": "Force check done: checked={checked}, updated={updated}, removed={removed}, unchanged={unchanged}",
        "ru": "Проверка завершена: проверено={checked}, обновлено={updated}, удалено={removed}, без изменений={unchanged}",
    },
    "db_err_no_backups_found": {"en": "No backups found", "ru": "Бэкапы не найдены"},
    "db_err_failed_list_backups": {
        "en": "Failed to list backups: {reason}",
        "ru": "Не удалось получить список бэкапов: {reason}",
    },
    "db_err_failed_read_backup": {
        "en": "Failed to read backup: {reason}",
        "ru": "Не удалось прочитать бэкап: {reason}",
    },
    "db_err_failed_apply_backup": {
        "en": "Failed to apply backup: {reason}",
        "ru": "Не удалось применить бэкап: {reason}",
    },
    "update_bulletin_text": {
        "en": "Streaks plugin update available!",
        "ru": "Доступна обновление плагина Streaks!",
    },
    "update_bulletin_button": {
        "en": "Update",
        "ru": "Обновить",
    },
}


class TimeUtils:
    @staticmethod
    def get_server_timestamp() -> int:
        local_timestamp = int(time.time())

        try:
            timestamp = int(get_connections_manager().getCurrentTime())
            if timestamp > 0 and abs(timestamp - local_timestamp) <= (
                2 * SECONDS_IN_DAY
            ):
                return timestamp
        except Exception:
            pass

        return local_timestamp

    @staticmethod
    def get_server_timezone() -> TimeZone:
        return TimeZone.getTimeZone(String("UTC"))

    @staticmethod
    def get_user_timezone() -> TimeZone:
        return TimeZone.getDefault()

    @staticmethod
    def strip_timestamp_in_timezone(timestamp: int, timezone: TimeZone) -> int:
        calendar = Calendar.getInstance(timezone)
        calendar.setTimeInMillis(int(timestamp) * 1000)
        calendar.set(Calendar.MILLISECOND, 0)
        calendar.set(Calendar.SECOND, 0)
        calendar.set(Calendar.MINUTE, 0)
        calendar.set(Calendar.HOUR_OF_DAY, 0)

        return int(calendar.getTimeInMillis() / 1000)

    @staticmethod
    def get_stripped_timestamp_in_timezone(
        day_offset: int, timezone: TimeZone, now_timestamp: Optional[int] = None
    ) -> int:
        if now_timestamp is None:
            now_timestamp = TimeUtils.get_server_timestamp()

        calendar = Calendar.getInstance(timezone)
        calendar.setTimeInMillis(int(now_timestamp) * 1000)
        calendar.set(Calendar.MILLISECOND, 0)
        calendar.set(Calendar.SECOND, 0)
        calendar.set(Calendar.MINUTE, 0)
        calendar.set(Calendar.HOUR_OF_DAY, 0)

        if day_offset != 0:
            calendar.add(Calendar.DAY_OF_MONTH, int(day_offset))

        return int(calendar.getTimeInMillis() / 1000)

    @staticmethod
    def strip_timestamp(timestamp: int) -> int:
        # User-local day bucket. Used for streak freeze logic.
        return TimeUtils.strip_timestamp_in_timezone(
            timestamp, TimeUtils.get_user_timezone()
        )

    @staticmethod
    def strip_server_timestamp(timestamp: int) -> int:
        # Server day bucket. Used for request ranges and comparisons.
        return TimeUtils.strip_timestamp_in_timezone(
            timestamp, TimeUtils.get_server_timezone()
        )

    @staticmethod
    def get_stripped_timestamp(day_offset: int = 0) -> int:
        # User-local day bucket. Freeze must happen at local midnight.
        return TimeUtils.get_stripped_timestamp_in_timezone(
            day_offset, TimeUtils.get_user_timezone(), now_timestamp=int(time.time())
        )

    @staticmethod
    def get_server_stripped_timestamp(day_offset: int = 0) -> int:
        # Server day bucket for history checks.
        return TimeUtils.get_stripped_timestamp_in_timezone(
            day_offset, TimeUtils.get_server_timezone()
        )

    @staticmethod
    def get_stripped_boundaries(day_offset: int) -> tuple[int, int]:
        return (
            TimeUtils.get_stripped_timestamp(day_offset),
            TimeUtils.get_stripped_timestamp(day_offset + 1),
        )

    @staticmethod
    def get_server_stripped_boundaries(day_offset: int) -> tuple[int, int]:
        return (
            TimeUtils.get_server_stripped_timestamp(day_offset),
            TimeUtils.get_server_stripped_timestamp(day_offset + 1),
        )

    @staticmethod
    def get_server_day_offset(timestamp: int) -> int:
        server_today = TimeUtils.get_server_stripped_timestamp()
        timestamp_day = TimeUtils.strip_server_timestamp(timestamp)
        return int((timestamp_day - server_today) // SECONDS_IN_DAY)

    @staticmethod
    def get_local_day_offset(timestamp: int) -> int:
        local_today = TimeUtils.get_stripped_timestamp()
        timestamp_day = TimeUtils.strip_timestamp(timestamp)
        return int((timestamp_day - local_today) // SECONDS_IN_DAY)


class JvmPluginBridge:
    klass: Optional[Class]

    def __init__(self, plugin: BasePlugin):
        self.plugin = plugin
        self.klass = None

        self.cache_dir = os.path.join(
            ApplicationLoader.applicationContext.getExternalFilesDir(
                None  # ty:ignore[invalid-argument-type]
            ).getAbsolutePath(),
            "plugins_dex_cache",
        )

        if not os.path.exists(self.cache_dir):
            os.makedirs(self.cache_dir)

        self.dex_path = os.path.join(self.cache_dir, f"{__id__}.dex")

    def load(self):
        if DEBUG_MODE:
            self._download()
            self._load_cached_file()
            return

        expected_sha256 = str(DEX_SHA256).strip().lower()

        cached_sha256 = self._compute_file_sha256(self.dex_path)
        if cached_sha256 is not None and cached_sha256 == expected_sha256:
            self.plugin.log("Using cached DEX (SHA256 matched)")
            self._load_cached_file()
            return

        if cached_sha256 is None:
            self.plugin.log("Cached DEX not found. Downloading new file...")
        else:
            self.plugin.log(
                f"Cached DEX SHA256 mismatch (cached={cached_sha256}, expected={expected_sha256}). Downloading new file..."
            )

        dex_data = self._download_bytes()
        if dex_data is None:
            return

        downloaded_sha256 = self._compute_sha256(dex_data)
        if downloaded_sha256 != expected_sha256:
            self._force_disable_plugin(
                "Downloaded DEX SHA256 mismatch. Plugin was disabled for safety."
            )
            return

        self._write_dex_file(dex_data)
        self._load(dex_data)

    def _load_cached_file(self):
        try:
            with open(self.dex_path, "rb") as f:
                self._load(f.read())
        except Exception as e:
            self.plugin.log(str(e))

    def _compute_sha256(self, data: bytes) -> str:
        return hashlib.sha256(data).hexdigest().lower()

    def _compute_file_sha256(self, path: str) -> Optional[str]:
        if not os.path.exists(path):
            return None

        try:
            with open(path, "rb") as f:
                return self._compute_sha256(f.read())
        except Exception as e:
            self.plugin.log(f"Failed to read cached DEX for SHA256: {e}")
            return None

    def _write_dex_file(self, dex_data: bytes):
        with open(self.dex_path, "wb") as f:
            f.write(dex_data)

    def _download_bytes(self) -> Optional[bytes]:
        try:
            self.plugin.log("Downloading DEX...")
            r = requests.get(DEX_URL, timeout=10)
            if r.status_code == 200:
                return cast("bytes", r.content)
            self.plugin.log(f"Failed to download DEX: {r.status_code}")
            return None
        except Exception as e:
            self.plugin.log(f"Failed to download DEX: {e}")
            return None

    def _force_disable_plugin(self, reason: str):
        self.plugin.log(reason)

        try:
            setattr(self.plugin, "enabled", False)
        except Exception:
            pass

        raise RuntimeError(reason)

    def _download(self):
        dex_data = self._download_bytes()
        if dex_data is None:
            return
        self._write_dex_file(dex_data)

    def _load(self, dex_data):
        class_path = "ru.n08i40k.streaks.Plugin"

        try:
            loader = InMemoryDexClassLoader(
                jclass("java.nio.ByteBuffer").wrap(dex_data),  # ty:ignore[unresolved-attribute]
                ApplicationLoader.applicationContext.getClassLoader(),
            )

            self.klass = loader.loadClass(String(class_path))

        except Exception as e:
            self.plugin.log(f"Failed to load DEX: {e}")


def _resolve_day_check_messages_table_name(db: Any) -> Optional[str]:
    global _day_check_messages_table_name
    global _day_check_messages_table_resolved

    if _day_check_messages_table_resolved:
        return _day_check_messages_table_name

    for table_name in DAY_CHECK_MESSAGE_TABLE_CANDIDATES:
        cursor = None
        try:
            cursor = db.queryFinalized(f"SELECT out FROM {table_name} LIMIT 1")
            _day_check_messages_table_name = str(table_name)
            _day_check_messages_table_resolved = True
            return _day_check_messages_table_name
        except Exception:
            pass
        finally:
            if cursor is not None:
                try:
                    cursor.dispose()
                except Exception:
                    pass

    _day_check_messages_table_name = None
    _day_check_messages_table_resolved = True
    return None


def _try_get_cached_day_activity(
    peer: TLRPC.InputPeer, start_ts: int, end_ts: int
) -> Optional[bool]:
    try:
        dialog_id = int(getattr(peer, "user_id", 0))
    except Exception:
        return None

    if dialog_id <= 0:
        return None

    try:
        storage = get_account_instance().getMessagesStorage()
        db = storage.getDatabase()
    except Exception:
        return None

    table_name = _resolve_day_check_messages_table_name(db)
    if table_name is None:
        return None

    cursor = None

    try:
        cursor = db.queryFinalized(
            (
                String(
                    f"SELECT "
                    f"MAX(CASE WHEN out = 1 THEN 1 ELSE 0 END), "
                    f"MAX(CASE WHEN out = 0 THEN 1 ELSE 0 END) "
                    f"FROM {table_name} "
                    f"WHERE uid = ? AND date >= ? AND date < ?"
                )
            ),
            Integer(int(dialog_id)),
            Integer(int(start_ts)),
            Integer(int(end_ts)),
        )

        if cursor is None or not bool(cursor.next()):
            return None

        has_out = 0 if cursor.isNull(0) else int(cursor.intValue(0))
        has_in = 0 if cursor.isNull(1) else int(cursor.intValue(1))

        if has_out == 1 and has_in == 1:
            return True

        return None
    except Exception:
        return None
    finally:
        if cursor is not None:
            try:
                cursor.dispose()
            except Exception:
                pass


def was_chat_active(
    peer: TLRPC.InputPeer, day_offset: int, fail_on_error: bool = False
) -> bool:
    start_ts, end_ts = TimeUtils.get_stripped_boundaries(day_offset)
    _, server_end_ts = TimeUtils.get_server_stripped_boundaries(day_offset)
    initial_offset_date = int(max(end_ts, server_end_ts))

    cached_result = _try_get_cached_day_activity(peer, start_ts, end_ts)
    if cached_result is not None:
        return bool(cached_result)

    attempts = int(DAY_CHECK_TIMEOUT_RETRIES) + 1
    last_state: Optional[dict[str, Any]] = None

    def run_single_attempt() -> dict[str, Any]:
        done = threading.Event()
        state = {
            "offset_id": 0,
            "offset_date": initial_offset_date,
            "from_self": False,
            "from_peer": False,
            "failed": False,
            "timeout": False,
            "failure_reason": "",
        }

        def load():
            req = TLRPC.TL_messages_getHistory()
            req.peer = peer
            req.offset_id = int(state["offset_id"])
            req.offset_date = int(state["offset_date"])
            req.limit = 10

            class Delegate(dynamic_proxy(RequestDelegate)):
                def run(self, response: TLObject, error: TLRPC.TL_error | None) -> None:
                    if error is not None:
                        state["failed"] = True
                        try:
                            state["failure_reason"] = f"request error: {error.text}"
                        except Exception:
                            state["failure_reason"] = f"request error: {error}"
                        done.set()
                        log(error)
                        return

                    response = cast("TLRPC.messages_Messages", response)
                    msgs = response.messages

                    if msgs is None or msgs.size() == 0:
                        done.set()
                        return

                    for i in range(msgs.size()):
                        m = msgs.get(i)
                        d = int(m.date)

                        if start_ts <= d < end_ts:
                            if m.from_id.user_id == peer.user_id:
                                state["from_peer"] = True
                            else:
                                state["from_self"] = True

                            if state["from_self"] and state["from_peer"]:
                                done.set()
                                return

                    oldest = msgs.get(msgs.size() - 1)
                    state["offset_id"] = int(oldest.id)
                    state["offset_date"] = int(oldest.date)

                    if int(oldest.date) >= start_ts:
                        load()
                    else:
                        done.set()

            conn_m = get_connections_manager()
            conn_m.sendRequest(req, Delegate())

        load()

        if not done.wait(timeout=int(DAY_CHECK_TIMEOUT_SECONDS)):
            state["failed"] = True
            state["timeout"] = True
            state["failure_reason"] = (
                f"day check timed out after {DAY_CHECK_TIMEOUT_SECONDS}s "
                f"(attempt {attempt + 1}/{attempts})"
            )

        return state

    for attempt in range(attempts):
        state = run_single_attempt()
        last_state = state

        if bool(state.get("timeout", False)) and attempt < (attempts - 1):
            if DAY_CHECK_RETRY_DELAY_SECONDS > 0:
                time.sleep(float(DAY_CHECK_RETRY_DELAY_SECONDS))
            continue

        if fail_on_error and bool(state["failed"]):
            reason = cast("str", state.get("failure_reason", "")) or "unknown"
            raise RuntimeError(f"day activity check failed: {reason}")

        return state["from_peer"] and state["from_self"]

    if fail_on_error:
        reason = "unknown"
        if last_state is not None:
            reason = cast("str", last_state.get("failure_reason", "")) or "unknown"
        raise RuntimeError(f"day activity check failed: {reason}")

    return False


def get_streak_length(peer_id: int):
    peer = get_messages_controller().getInputPeer(peer_id)  # ty:ignore[no-matching-overload]

    streak_days = 0

    while True:
        if not was_chat_active(peer, -streak_days):
            break

        streak_days += 1

    return streak_days


def get_streak_die_date(peer_id: int, day_offset: int) -> Optional[int]:
    if day_offset > 0:
        raise Exception("Day offset must be negative or zero")

    peer = get_messages_controller().getInputPeer(peer_id)  # ty:ignore[no-matching-overload]

    while True:
        if not was_chat_active(peer, day_offset):
            break

        day_offset += 1

        if day_offset > 0:
            return None

    return day_offset


def get_streak_die_date_from_freeze_ts(peer_id: int, freeze_ts: int) -> Optional[int]:
    return get_streak_die_date(peer_id, TimeUtils.get_local_day_offset(freeze_ts))


class StreakLevel:
    document_id: int
    text_color: Color
    text_color_int: int

    def __init__(self, document_id: int, text_color: tuple[int, int, int]):
        self.document_id = document_id
        self.text_color = Color.valueOf(
            text_color[0] / 255, text_color[1] / 255, text_color[2] / 255, 1.0
        )
        self.text_color_int = Color.rgb(
            int(text_color[0]), int(text_color[1]), int(text_color[2])
        )


class StreakLevels(Enum):
    COLD = StreakLevel(5285071881815235305, (175, 175, 175))
    DAYS_3 = StreakLevel(5285079178964672780, (255, 154, 0))
    DAYS_10 = StreakLevel(5285274844789777412, (255, 100, 0))
    DAYS_30 = StreakLevel(5285076623459129616, (255, 61, 0))
    DAYS_100 = StreakLevel(5285003347022093599, (255, 0, 200))
    DAYS_200 = StreakLevel(5285514817497504375, (176, 0, 255))

    def pick_by_length(length: int, cold: bool = False) -> StreakLevel:
        if cold:
            return StreakLevels.COLD.value

        if length < 10:
            return StreakLevels.DAYS_3.value
        elif length < 30:
            return StreakLevels.DAYS_10.value
        elif length < 100:
            return StreakLevels.DAYS_30.value
        elif length < 200:
            return StreakLevels.DAYS_100.value

        return StreakLevels.DAYS_200.value


class SenderType(Enum):
    SELF = 0
    PEER = 1


P = ParamSpec("P")
R = TypeVar("R")
S = TypeVar("S", bound=object)


def cache_by_field_value(
    *,
    field_name: str,
) -> Callable[[Callable[Concatenate[S, P], R]], Callable[Concatenate[S, P], R]]:
    def inner(method: Callable[Concatenate[S, P], R]) -> Callable[Concatenate[S, P], R]:
        cache_attr = f"__cache__{method.__name__}__by_{field_name}"  # ty:ignore[unresolved-attribute]
        lock_attr = f"__lock__{method.__name__}__by_{field_name}"  # ty:ignore[unresolved-attribute]

        @wraps(method)
        def wrapper(self: S, *args: P.args, **kwargs: P.kwargs) -> R:
            field_value = getattr(self, field_name, None)

            val_cache = getattr(self, cache_attr, None)

            if val_cache is None:
                val_cache = {}
                setattr(self, cache_attr, val_cache)

            val_lock: Optional[threading.RLock] = getattr(self, lock_attr, None)

            if val_lock is None:
                val_lock = threading.RLock()
                setattr(self, lock_attr, val_lock)

            with val_lock:
                if field_value in val_cache:
                    return val_cache[field_value]

                value = method(self, *args, **kwargs)
                val_cache.clear()
                val_cache[field_value] = value
                return value

        return wrapper

    return inner


@dataclass
class UserRecord:
    user_id: int

    started_at: int
    freezes_at: int

    last_sended_at: Optional[int]
    last_received_at: Optional[int]

    def to_dict(self) -> dict[str, Optional[int]]:
        return {
            "user_id": int(self.user_id),
            "started_at": int(self.started_at),
            "freezes_at": int(self.freezes_at),
            "last_sended_at": None
            if self.last_sended_at is None
            else int(self.last_sended_at),
            "last_received_at": None
            if self.last_received_at is None
            else int(self.last_received_at),
        }

    def from_dict(payload: dict[str, Any]) -> Optional["UserRecord"]:
        try:
            user_id = int(payload["user_id"])
            started_at = int(payload["started_at"])
            freezes_at = int(payload["freezes_at"])
        except Exception:
            return None

        if user_id <= 0:
            return None

        last_sended_at_raw = payload.get("last_sended_at")
        last_received_at_raw = payload.get("last_received_at")

        return UserRecord(
            user_id=user_id,
            started_at=started_at,
            freezes_at=freezes_at,
            last_sended_at=(
                None if last_sended_at_raw is None else int(last_sended_at_raw)
            ),
            last_received_at=(
                None if last_received_at_raw is None else int(last_received_at_raw)
            ),
        )

    def new(peer_id: int, sender: SenderType, event_day: Optional[int] = None):
        day = (
            TimeUtils.get_stripped_timestamp()
            if event_day is None
            else TimeUtils.strip_timestamp(event_day)
        )

        self = UserRecord(
            user_id=peer_id,
            started_at=day,
            freezes_at=(day + SECONDS_IN_DAY),
            last_sended_at=(day if sender == SenderType.SELF else None),
            last_received_at=(day if sender == SenderType.PEER else None),
        )

        return self

    def get_length(self) -> int:
        length = (
            TimeUtils.get_stripped_timestamp() - self.started_at
        ) // SECONDS_IN_DAY

        if not self.is_freezed():
            length += 1

        return length

    def is_freezed(self) -> bool:
        return TimeUtils.get_stripped_timestamp() >= self.freezes_at

    def should_die(self) -> bool:
        return (
            (TimeUtils.get_stripped_timestamp() - self.freezes_at) // SECONDS_IN_DAY
        ) > 0

    @cache_by_field_value(field_name="freezes_at")
    def is_alive(self) -> bool:  # "hungry" function, shouldn't be called frequently
        if not self.is_freezed():
            return True

        if not self.should_die():
            return True

        # History checks are server-day based; freeze timestamp itself is user-day based.
        death_date = get_streak_die_date_from_freeze_ts(self.user_id, self.freezes_at)

        return (
            death_date is None or death_date == 0
        )  # can be freezed, is death date is today

    def update_freeze_date(self):
        self.freezes_at = TimeUtils.get_stripped_timestamp(
            1
        )  # will be freezed from tomorrow

    def update_from(
        self, sender_type: SenderType, event_day: Optional[int] = None
    ) -> bool:
        day = (
            TimeUtils.get_stripped_timestamp()
            if event_day is None
            else TimeUtils.strip_timestamp(event_day)
        )

        changed = False

        if sender_type == SenderType.SELF:
            if self.last_sended_at is None or day > self.last_sended_at:
                self.last_sended_at = day
                changed = True
        else:
            if self.last_received_at is None or day > self.last_received_at:
                self.last_received_at = day
                changed = True

        if self.last_sended_at is None or self.last_received_at is None:
            return changed

        if self.last_sended_at == self.last_received_at:
            # Move freeze date to the next day of the latest day with both-side activity.
            next_freeze = max(self.freezes_at, self.last_sended_at + SECONDS_IN_DAY)
            if next_freeze != self.freezes_at:
                self.freezes_at = next_freeze
                changed = True

        return changed


def reset_drawable_cache(jvm_plugin: JvmPluginBridge):
    klass = jvm_plugin.klass

    if klass is None:
        return

    klass.getDeclaredMethod(String("clearCaches")).invoke(None)  # ty:ignore[invalid-argument-type]


def get_authorized_user_ids() -> list[int]:
    result = []
    for i in range(UserConfig.MAX_ACCOUNT_COUNT):
        cfg = UserConfig.getInstance(i)
        if cfg.isClientActivated():
            result.append(cfg.getClientUserId())
    return result


class UsersDatabase:
    def __init__(
        self,
        jvm_plugin: JvmPluginBridge,
        logger: Callable[[str], None],
        storage_path: Optional[str] = None,
    ):
        self._map: dict[int, UserRecord] = dict()
        self._lock = threading.RLock()
        self._jvm_plugin = jvm_plugin
        self._logger = logger
        self._storage_path = storage_path or self._build_storage_path()
        self._storage_dir = os.path.dirname(self._storage_path)
        self._backups_dir = os.path.join(self._storage_dir, "backups")
        os.makedirs(self._backups_dir, exist_ok=True)
        self._load()
        self.ensure_daily_backup()

    def _build_storage_path(self) -> str:
        files_dir = ApplicationLoader.applicationContext.getFilesDir().getAbsolutePath()
        storage_dir = os.path.join(files_dir, "chaquopy", "plugins", __id__)
        if not os.path.exists(storage_dir):
            os.makedirs(storage_dir, exist_ok=True)
        return os.path.join(storage_dir, "users_db.json")

    def _build_payload_locked(self) -> dict[str, Any]:
        return {
            "version": 1,
            "saved_at": int(time.time()),
            "users": [record.to_dict() for record in self._map.values()],
        }

    def _write_payload_to_path(self, payload: dict[str, Any], target_path: str):
        tmp_path = f"{target_path}.tmp"
        with open(tmp_path, "w", encoding="utf-8") as f:
            json.dump(payload, f, ensure_ascii=True, separators=(",", ":"))
        os.replace(tmp_path, target_path)

    def _load_map_from_payload(self, payload: dict[str, Any]) -> dict[int, UserRecord]:
        users_payload = payload.get("users", [])
        loaded_map: dict[int, UserRecord] = {}

        for row in users_payload:
            if not isinstance(row, dict):
                continue

            record = UserRecord.from_dict(cast("dict[str, Any]", row))
            if record is None:
                continue

            loaded_map[record.user_id] = record

        return loaded_map

    def _daily_backup_path(self, day_key: str) -> str:
        return os.path.join(self._backups_dir, f"users_db.daily.{day_key}.json")

    def _get_local_day_key(self) -> str:
        return time.strftime("%Y-%m-%d", time.localtime())

    def _ensure_daily_backup_locked(self) -> Optional[str]:
        day_key = self._get_local_day_key()
        target_path = self._daily_backup_path(day_key)

        if os.path.exists(target_path):
            return target_path

        payload = self._build_payload_locked()
        self._write_payload_to_path(payload, target_path)
        return target_path

    def _collect_backup_paths_locked(self) -> list[str]:
        if not os.path.exists(self._backups_dir):
            return []

        backup_paths = []
        for name in os.listdir(self._backups_dir):
            if not name.endswith(".json"):
                continue
            backup_paths.append(os.path.join(self._backups_dir, name))

        backup_paths.sort(key=lambda path: os.path.getmtime(path), reverse=True)
        return backup_paths

    def ensure_daily_backup(self):
        try:
            with self._lock:
                self._ensure_daily_backup_locked()
        except Exception as e:
            self._logger(f"UsersDatabase daily backup failed: {e}")

    def export_backup_now(self) -> Optional[str]:
        with self._lock:
            payload = self._build_payload_locked()
            stamp = time.strftime("%Y-%m-%d_%H-%M-%S", time.localtime())
            target_path = os.path.join(
                self._backups_dir, f"users_db.export.{stamp}.json"
            )
            self._write_payload_to_path(payload, target_path)
            return target_path

    def import_latest_backup(self) -> tuple[bool, str]:
        try:
            with self._lock:
                backup_paths = self._collect_backup_paths_locked()
        except Exception as e:
            return (False, f"Failed to list backups: {e}")

        if len(backup_paths) == 0:
            return (False, "No backups found")

        latest_path = backup_paths[0]
        return self.import_backup(latest_path)

    def import_backup(self, backup_path: str) -> tuple[bool, str]:
        try:
            with open(backup_path, "r", encoding="utf-8") as f:
                payload = cast("dict[str, Any]", json.load(f))
            loaded_map = self._load_map_from_payload(payload)
        except Exception as e:
            return (False, f"Failed to read backup: {e}")

        changed = False
        try:
            with self._lock:
                if self._map != loaded_map:
                    self._map = loaded_map
                    self._persist_locked()
                    changed = True
        except Exception as e:
            return (False, f"Failed to apply backup: {e}")

        if changed:
            reset_drawable_cache(self._jvm_plugin)

        self._logger(
            f"UsersDatabase backup imported: {backup_path}, records={len(loaded_map)}"
        )
        return (True, backup_path)

    def get_backups_dir(self) -> str:
        return self._backups_dir

    def _persist_locked(self):
        payload = self._build_payload_locked()
        self._write_payload_to_path(payload, self._storage_path)
        self._ensure_daily_backup_locked()

    def _load(self):
        if not os.path.exists(self._storage_path):
            return

        try:
            with open(self._storage_path, "r", encoding="utf-8") as f:
                payload = json.load(f)
            loaded_map = self._load_map_from_payload(cast("dict[str, Any]", payload))

            with self._lock:
                self._map = loaded_map

            self._logger(f"UsersDatabase loaded: {len(loaded_map)} records")
        except Exception as e:
            self._logger(f"UsersDatabase load failed: {e}")

    def save(self):
        try:
            with self._lock:
                self._persist_locked()
            reset_drawable_cache(self._jvm_plugin)
        except Exception as e:
            self._logger(f"UsersDatabase save failed: {e}")

    def get_user(
        self, user_id: int, include_authorized: bool = False
    ) -> Optional[UserRecord]:
        if user_id <= 0:
            return None

        if not include_authorized and user_id in get_authorized_user_ids():
            return None

        with self._lock:
            return self._map.get(user_id)

    def list_user_ids(self) -> list[int]:
        with self._lock:
            return list(self._map.keys())

    def list_users(self) -> list[UserRecord]:
        with self._lock:
            return list(self._map.values())

    def set_user_record(
        self,
        record: UserRecord,
        persist: bool = True,
        reset_cache: bool = True,
    ) -> bool:
        changed = False
        with self._lock:
            user_id = int(record.user_id)
            current = self._map.get(user_id)
            if current is None or current.to_dict() != record.to_dict():
                self._map[user_id] = record
                if persist:
                    self._persist_locked()
                changed = True

        if changed and reset_cache:
            reset_drawable_cache(self._jvm_plugin)

        return changed

    def remove_user_record(
        self,
        user_id: int,
        persist: bool = True,
        reset_cache: bool = True,
    ) -> bool:
        changed = False
        with self._lock:
            if self._map.pop(int(user_id), None) is not None:
                if persist:
                    self._persist_locked()
                changed = True

        if changed and reset_cache:
            reset_drawable_cache(self._jvm_plugin)

        return changed


class StreaksController:
    def __init__(
        self,
        users_db: UsersDatabase,
        logger: Callable[[str], None],
    ):
        self._users_db = users_db
        self._logger = logger

    def _copy_record(self, record: UserRecord) -> Optional[UserRecord]:
        return UserRecord.from_dict(record.to_dict())

    def on_dialog_event(
        self, peer_id: int, sender_type: SenderType, event_day: Optional[int] = None
    ):
        if peer_id <= 0:
            return

        day = (
            TimeUtils.get_stripped_timestamp()
            if event_day is None
            else TimeUtils.strip_timestamp(event_day)
        )
        current = self._users_db.get_user(peer_id, include_authorized=True)
        changed = False

        if current is None:
            record = UserRecord.new(peer_id, sender_type, day)
            changed = True
        else:
            record = UserRecord.from_dict(current.to_dict())
            if record is None:
                record = UserRecord.new(peer_id, sender_type, day)
                changed = True

        changed = record.update_from(sender_type, day) or changed
        if changed:
            self._users_db.set_user_record(record)

    def _build_recent_record(
        self,
        user_id: int,
        include_today: bool,
        on_day_progress: Optional[Callable[[int, bool, bool], None]] = None,
        fail_on_error: bool = False,
    ) -> Optional[UserRecord]:
        peer = get_messages_controller().getInputPeer(user_id)  # ty:ignore[no-matching-overload]
        if peer is None:
            return None

        first_day_check = True

        def check_day(day_offset: int) -> bool:
            nonlocal first_day_check

            if not first_day_check and DAY_CHECK_REQUEST_DELAY_SECONDS > 0:
                time.sleep(float(DAY_CHECK_REQUEST_DELAY_SECONDS))

            first_day_check = False
            return was_chat_active(peer, day_offset, fail_on_error=fail_on_error)

        day_offset = 0 if include_today else -1
        is_active = check_day(day_offset)
        if on_day_progress is not None:
            on_day_progress(day_offset, is_active, include_today)

        if not is_active:
            return None

        streak_days = 1

        while True:
            day_offset = -streak_days if include_today else -(streak_days + 1)
            is_active = check_day(day_offset)
            if on_day_progress is not None:
                on_day_progress(day_offset, is_active, include_today)

            if not is_active:
                break
            streak_days += 1

        today = TimeUtils.get_stripped_timestamp()
        started_at = today - (
            (streak_days - 1) * SECONDS_IN_DAY
            if include_today
            else streak_days * SECONDS_IN_DAY
        )
        active_day = today if include_today else today - SECONDS_IN_DAY
        freeze_day = today + SECONDS_IN_DAY if include_today else today

        return UserRecord(
            user_id=user_id,
            started_at=started_at,
            freezes_at=freeze_day,
            last_sended_at=active_day,
            last_received_at=active_day,
        )

    def _build_recent_record_any(
        self,
        user_id: int,
        on_day_progress: Optional[Callable[[int, int, bool, bool], None]] = None,
        fail_on_error: bool = False,
    ) -> Optional[UserRecord]:
        checked_days = 0

        def day_progress(day_offset: int, is_active: bool, include_today: bool):
            nonlocal checked_days
            checked_days += 1

            if on_day_progress is not None:
                on_day_progress(checked_days, day_offset, is_active, include_today)

        record = self._build_recent_record(
            user_id,
            include_today=True,
            on_day_progress=day_progress,
            fail_on_error=fail_on_error,
        )

        if record is not None:
            return record

        return self._build_recent_record(
            user_id,
            include_today=False,
            on_day_progress=day_progress,
            fail_on_error=fail_on_error,
        )

    def reconcile_on_plugin_load(self) -> tuple[int, int, int]:
        user_ids = self._users_db.list_user_ids()
        checked = 0
        updated = 0
        removed = 0
        changed = False

        for user_id in user_ids:
            record = self._users_db.get_user(user_id, include_authorized=True)
            if record is None:
                continue

            if not record.should_die():
                continue

            checked += 1

            try:
                death_date = get_streak_die_date_from_freeze_ts(
                    user_id, record.freezes_at
                )
            except Exception as e:
                self._logger(f"reconcile failed for {user_id}: {e}")
                continue

            if death_date in (None, 0):
                updated_record = self._copy_record(record)
                if updated_record is None:
                    continue

                if death_date is None:
                    updated_record.update_freeze_date()
                else:
                    updated_record.freezes_at = TimeUtils.get_stripped_timestamp()

                if self._users_db.set_user_record(
                    updated_record, persist=False, reset_cache=False
                ):
                    updated += 1
                    changed = True
                continue

            rebuilt = self._build_recent_record(user_id, include_today=True)
            if rebuilt is None:
                rebuilt = self._build_recent_record(user_id, include_today=False)

            if rebuilt is not None:
                if self._users_db.set_user_record(
                    rebuilt, persist=False, reset_cache=False
                ):
                    updated += 1
                    changed = True
            else:
                if self._users_db.remove_user_record(
                    user_id, persist=False, reset_cache=False
                ):
                    removed += 1
                    changed = True

        if changed:
            self._users_db.save()

        return checked, updated, removed

    def force_check_user_ids(
        self,
        user_ids: list[int],
        on_progress: Optional[Callable[[int, int, int, int, int], None]] = None,
        on_day_progress: Optional[Callable[[int, int, int, bool, bool], None]] = None,
    ) -> tuple[int, int, int, int]:
        authorized_ids = set(get_authorized_user_ids())
        valid_user_ids = [
            user_id
            for user_id in user_ids
            if user_id > 0 and user_id not in authorized_ids
        ]
        total = len(valid_user_ids)
        checked = 0
        updated = 0
        removed = 0
        unchanged = 0
        changed = False

        if on_progress is not None:
            on_progress(0, total, updated, removed, unchanged)

        for user_id in valid_user_ids:
            checked += 1

            try:
                rebuilt = self._build_recent_record_any(
                    user_id,
                    fail_on_error=True,
                    on_day_progress=(
                        None
                        if on_day_progress is None
                        else (
                            lambda day_checked, day_offset, is_active, include_today: (
                                on_day_progress(
                                    user_id,
                                    day_checked,
                                    day_offset,
                                    is_active,
                                    include_today,
                                )
                            )
                        )
                    ),
                )
            except Exception as e:
                self._logger(f"force check failed for {user_id}: {e}")
                if on_progress is not None:
                    on_progress(checked, total, updated, removed, unchanged)
                continue

            current = self._users_db.get_user(user_id, include_authorized=True)
            if rebuilt is None:
                if self._users_db.remove_user_record(
                    user_id, persist=False, reset_cache=False
                ):
                    removed += 1
                    changed = True
                else:
                    unchanged += 1

                if on_progress is not None:
                    on_progress(checked, total, updated, removed, unchanged)
                continue

            if current is None or current.to_dict() != rebuilt.to_dict():
                self._users_db.set_user_record(
                    rebuilt, persist=False, reset_cache=False
                )
                updated += 1
                changed = True
            else:
                unchanged += 1

            if on_progress is not None:
                on_progress(checked, total, updated, removed, unchanged)

        if changed:
            self._users_db.save()

        return checked, updated, removed, unchanged


class TgStreaksPlugin(BasePlugin):
    def create_settings(self) -> list[Any]:
        return [
            Header(text=self._t("settings_updates")),
            Switch(
                key=SETTING_UPDATE_CHECK_ENABLED,
                text=self._t("settings_check_updates"),
                default=self._is_update_check_enabled(),
                subtext=self._t("settings_check_updates_hint"),
                icon="msg_retry",
                on_change=lambda value: self._on_update_check_setting_changed(value),
            ),
            Header(text=self._t("settings_streak_tools")),
            Text(
                text=self._t("settings_force_check_all_private_chats"),
                icon="msg_retry",
                on_click=lambda _: self._on_force_check_all_chats_clicked(),
            ),
            Divider(text=self._t("settings_only_private_hint")),
            Header(text=self._t("settings_db_backups")),
            Text(
                text=self._t("settings_export_backup_now"),
                on_click=lambda _: self._on_export_backup_clicked(),
            ),
            Text(
                text=self._t("settings_import_latest_backup"),
                on_click=lambda _: self._on_import_latest_backup_clicked(),
            ),
            Divider(text=self._t("settings_db_backups_hint")),
        ]

    def _show_info(self, message: str):
        run_on_ui_thread(lambda: BulletinHelper.show_info(message))

    def _show_success(self, message: str):
        run_on_ui_thread(lambda: BulletinHelper.show_success(message))

    def _show_error(self, message: str):
        run_on_ui_thread(lambda: BulletinHelper.show_error(message))

    def _is_update_check_enabled(self) -> bool:
        try:
            return bool(self.get_setting(SETTING_UPDATE_CHECK_ENABLED, True))
        except Exception:
            return True

    def _on_update_check_setting_changed(self, enabled: bool):
        enabled = bool(enabled)

        try:
            self.set_setting(SETTING_UPDATE_CHECK_ENABLED, enabled)
        except Exception as e:
            self.log(f"Failed to persist update check setting: {e}")

        if enabled:
            self._start_update_check()
            return

        try:
            self._update_check_stop.set()
        except Exception:
            pass

    def _normalize_version_tag(self, value: Optional[str]) -> str:
        if value is None:
            return ""

        normalized = str(value).strip()
        if len(normalized) == 0:
            return ""

        return normalized.removeprefix("v").removeprefix("V")

    def _open_external_url(self, url: str):
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
                self.log(f"Failed to open external url {url}: {e}")
                self._show_error(self._t("err_failed_open_update_link"))

        run_on_ui_thread(open_url)

    def _show_update_bulletin(self):
        text = self._t("update_bulletin_text")
        button_text = self._t("update_bulletin_button")

        def show():
            if self._update_check_stop.is_set():
                return

            try:
                BulletinHelper.show_with_button(
                    text,
                    R_tg.raw.ic_download,
                    button_text,
                    lambda: self._open_external_url(PLUGIN_UPDATE_TG_URL),
                    duration=int(BulletinHelper.DURATION_PROLONG),
                )
            except Exception as e:
                self.log(f"Failed to show update bulletin: {e}")
                self._show_info(text)

        run_on_ui_thread(show)

    def _fetch_latest_release_tag(self) -> Optional[str]:
        response = requests.get(
            f"https://api.github.com/repos/{REPO_OWNER}/{REPO_NAME}/releases/latest",
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

    def _start_update_check(self):
        if DEBUG_MODE:
            self.log("Update check skipped: debug mode is enabled")
            return

        if not self._is_update_check_enabled():
            self.log("Update check skipped: disabled in plugin settings")
            return

        with self._update_check_lock:
            if self._update_check_inflight:
                return
            self._update_check_inflight = True

        self._update_check_stop.clear()

        def worker():
            try:
                latest_tag = self._fetch_latest_release_tag()
                if latest_tag is None:
                    self.log("Update check skipped: latest release tag_name is missing")
                    return

                latest_version = self._normalize_version_tag(latest_tag)
                current_version = self._normalize_version_tag(__version__)

                if len(latest_version) == 0:
                    self.log("Update check skipped: latest release version is empty")
                    return

                if latest_version == current_version:
                    self.log(f"Plugin is up to date: {current_version}")
                    return

                if self._update_check_stop.is_set():
                    return

                self.log(
                    f"Plugin update available: current={__version__}, latest={latest_tag}"
                )
                self._show_update_bulletin()
            except Exception as e:
                self.log(f"Plugin update check failed: {e}")
            finally:
                with self._update_check_lock:
                    self._update_check_inflight = False

        threading.Thread(target=worker, daemon=True).start()

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

    def _localize_db_import_error(self, message: str) -> str:
        if message == "No backups found":
            return self._t("db_err_no_backups_found")

        prefix_map = {
            "Failed to list backups:": "db_err_failed_list_backups",
            "Failed to read backup:": "db_err_failed_read_backup",
            "Failed to apply backup:": "db_err_failed_apply_backup",
        }

        for prefix, key in prefix_map.items():
            if message.startswith(prefix):
                reason = message[len(prefix) :].strip()
                return self._t(key, reason=reason)

        return message

    def _on_export_backup_clicked(self):
        if not hasattr(self, "users_db"):
            self._show_error(self._t("err_users_db_not_ready"))
            return

        try:
            backup_path = self.users_db.export_backup_now()
            if backup_path is None:
                self._show_error(self._t("err_failed_export_backup"))
                return

            self.log(f"Backup exported: {backup_path}")
            self._show_success(
                self._t("ok_backup_exported", name=os.path.basename(backup_path))
            )
        except Exception as e:
            self.log(f"Backup export failed: {e}")
            self._show_error(self._t("err_backup_export_failed"))

    def _on_import_latest_backup_clicked(self):
        if not hasattr(self, "users_db"):
            self._show_error(self._t("err_users_db_not_ready"))
            return

        try:
            ok, result = self.users_db.import_latest_backup()
            if not ok:
                self._show_error(self._localize_db_import_error(result))
                return

            self._patch_cached_users_streak_emoji_status()
            self._sync_dead_states_snapshot()
            self.log(f"Backup imported: {result}")
            self._show_success(
                self._t("ok_backup_imported", name=os.path.basename(result))
            )
        except Exception as e:
            self.log(f"Backup import failed: {e}")
            self._show_error(self._t("err_backup_import_failed"))

    def _resolve_popup_context(self):
        fragment = None
        context = None

        try:
            fragment = get_last_fragment()
        except Exception:
            fragment = None

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

    def _create_popup_emoji_drawable(
        self,
        parent_view: Any,
        emoji_size_dp: int,
        emoji_document_id: Optional[int],
        color: int,
    ):
        if emoji_document_id is None or int(emoji_document_id) <= 0:
            return None

        try:
            drawable = AnimatedEmojiDrawable.SwapAnimatedEmojiDrawable(
                parent_view, int(emoji_size_dp)
            )
            drawable.setParentView(parent_view)
            drawable.setBounds(
                0,
                0,
                AndroidUtilities.dp(float(emoji_size_dp)),
                AndroidUtilities.dp(float(emoji_size_dp)),
            )

            if self.jvm_plugin.klass is not None:
                self.jvm_plugin.klass.getDeclaredMethod(
                    String("enableParticles"),
                    AnimatedEmojiDrawable.SwapAnimatedEmojiDrawable.getClass(),
                    Integer.TYPE,
                ).invoke(None, drawable, color)  # ty:ignore[no-matching-overload]

            try:
                drawable.center = True
            except Exception:
                pass

            cache_type = int(AnimatedEmojiDrawable.getCacheTypeForEnterView())
            if not bool(drawable.set(jlong(int(emoji_document_id)), cache_type, True)):  # ty:ignore[no-matching-overload]
                return None

            drawable.attach()
            drawable.play()
            return drawable
        except Exception as e:
            self.log(f"Failed to build popup streak emoji: {e}")
            return None

    def _show_streak_popup(
        self,
        title: str,
        subtitle: str,
        accent_color: int,
        emoji_document_id: Optional[int] = None,
    ):
        def show():
            context = self._resolve_popup_context()
            if context is None:
                self.log("Skip streak popup: no context")
                return

            try:
                dialog = Dialog(context)
                dialog.requestWindowFeature(1)
                dialog.setCancelable(True)
                dialog.setCanceledOnTouchOutside(True)

                container = LinearLayout(context)
                container.setOrientation(1)
                container.setGravity(Gravity.CENTER_HORIZONTAL)
                container.setPadding(
                    AndroidUtilities.dp(16),
                    AndroidUtilities.dp(14),
                    AndroidUtilities.dp(16),
                    AndroidUtilities.dp(14),
                )

                bg = GradientDrawable()
                bg.setShape(0)
                bg.setColor(Color.argb(200, 20, 20, 24))
                bg.setCornerRadius(float(AndroidUtilities.dp(24)))
                bg.setStroke(AndroidUtilities.dp(1), accent_color)
                container.setBackground(bg)

                emoji_view = None
                emoji_drawable = None
                emoji_size = 0
                if emoji_document_id is not None and int(emoji_document_id) > 0:
                    emoji_view = ImageView(context)
                    emoji_view.setScaleType(ImageView.ScaleType.CENTER_INSIDE)  # ty:ignore[unresolved-attribute]
                    emoji_size_dp = 88 * 2
                    emoji_size = AndroidUtilities.dp(float(emoji_size_dp))
                    emoji_drawable = self._create_popup_emoji_drawable(
                        emoji_view,
                        emoji_size_dp,
                        emoji_document_id,
                        Integer(accent_color),
                    )

                if emoji_drawable is not None and emoji_view is not None:
                    emoji_view.setImageDrawable(emoji_drawable)
                    emoji_lp = LinearLayout.LayoutParams(
                        emoji_size, int(emoji_size / 2)
                    )
                    emoji_lp.bottomMargin = AndroidUtilities.dp(10)
                    container.addView(emoji_view, emoji_lp)

                title_view = TextView(context)
                title_view.setText(String(title))
                title_view.setTextColor(accent_color)
                title_view.setTextSize(24.0)
                title_view.setTypeface(AndroidUtilities.bold())
                title_view.setGravity(Gravity.CENTER_HORIZONTAL)
                title_view.setTypeface(
                    Typeface.create(Typeface.DEFAULT_BOLD, 900, False)
                )
                container.addView(title_view, LinearLayout.LayoutParams(-2, -2))

                subtitle_view = TextView(context)
                subtitle_view.setText(String(subtitle))
                subtitle_view.setTextColor(Color.WHITE)
                subtitle_view.setTextSize(14.0)
                subtitle_view.setGravity(Gravity.CENTER_HORIZONTAL)
                subtitle_lp = LinearLayout.LayoutParams(-2, -2)
                subtitle_lp.topMargin = AndroidUtilities.dp(6)
                container.addView(subtitle_view, subtitle_lp)

                dialog.setContentView(
                    container, LinearLayout.LayoutParams(AndroidUtilities.dp(320), -2)
                )

                window = dialog.getWindow()
                if window is not None:
                    window.setBackgroundDrawable(ColorDrawable(Color.TRANSPARENT))
                    window.setLayout(AndroidUtilities.dp(320), -2)
                    window.setGravity(Gravity.CENTER)
                    window.clearFlags(WindowManager.LayoutParams.FLAG_DIM_BEHIND)

                dialog.show()

                def release_emoji():
                    if emoji_drawable is None:
                        return

                    try:
                        emoji_drawable.detach()
                    except Exception:
                        pass

                class DismissListener(dynamic_proxy(DialogInterface.OnDismissListener)):
                    def onDismiss(self, var1):
                        release_emoji()

                dialog.setOnDismissListener(DismissListener())

                def dismiss_popup():
                    try:
                        release_emoji()
                        if dialog.isShowing():
                            dialog.dismiss()
                    except Exception:
                        pass

                threading.Timer(9.2, lambda: run_on_ui_thread(dismiss_popup)).start()
            except Exception as e:
                self.log(f"Failed to show streak popup: {e}")

        run_on_ui_thread(show)

    def _resolve_open_chat_dialog_id(self) -> Optional[int]:
        try:
            fragment = get_last_fragment()
        except Exception:
            return None

        if fragment is None:
            return None

        try:
            dialog_id = int(fragment.getDialogId())
            if dialog_id == 0:
                return None
            return dialog_id
        except Exception:
            return None

    def _enqueue_streak_popup_for_open_chat(
        self,
        peer_id: int,
        key: str,
        title: str,
        subtitle: str,
        accent_color: int,
        emoji_document_id: Optional[int] = None,
    ):
        if not self._can_show_streak_popup(key):
            return

        current_dialog_id = self._resolve_open_chat_dialog_id()

        if current_dialog_id is not None and int(current_dialog_id) == int(peer_id):
            self._show_streak_popup(
                title,
                subtitle,
                accent_color,
                emoji_document_id=emoji_document_id,
            )
            return

        popup_payload = {
            "title": str(title),
            "subtitle": str(subtitle),
            "accent_color": int(accent_color),
            "emoji_document_id": (
                None if emoji_document_id is None else int(emoji_document_id)
            ),
        }

        with self._streak_popup_queue_lock:
            queue = self._pending_streak_popups.get(int(peer_id))
            if queue is None:
                queue = []
                self._pending_streak_popups[int(peer_id)] = queue

            queue.append(popup_payload)
            if len(queue) > 5:
                del queue[:-5]

    def _flush_pending_streak_popup_for_open_chat(self):
        current_dialog_id = self._resolve_open_chat_dialog_id()
        if current_dialog_id is None:
            return

        popup_payload = None

        with self._streak_popup_queue_lock:
            queue = self._pending_streak_popups.get(int(current_dialog_id))
            if queue is None or len(queue) == 0:
                return

            popup_payload = queue.pop(0)
            if len(queue) == 0:
                self._pending_streak_popups.pop(int(current_dialog_id), None)

        if popup_payload is None:
            return

        self._show_streak_popup(
            cast("str", popup_payload["title"]),
            cast("str", popup_payload["subtitle"]),
            int(popup_payload["accent_color"]),
            emoji_document_id=cast("Optional[int]", popup_payload["emoji_document_id"]),
        )

    def _resolve_peer_name(self, peer_id: int) -> str:
        try:
            user = self._resolve_dialog_user(get_messages_controller(), int(peer_id))
            if user is not None:
                name = cast("str", UserObject.getUserName(user))
                if name is not None and len(name) > 0:
                    return name
        except Exception:
            pass

        return f"id:{peer_id}"

    def _can_show_streak_popup(self, key: str, cooldown_seconds: int = 20) -> bool:
        now = int(time.time())
        last = int(self._streak_popup_last.get(key, 0))

        if now - last < cooldown_seconds:
            return False

        self._streak_popup_last[key] = now
        return True

    def _resolve_level_text_color(self, level_id: int) -> int:
        target_level_id = int(level_id)

        for level in StreakLevels:
            if int(level.value.document_id) == target_level_id:
                return int(level.value.text_color_int)

        return Color.rgb(255, 173, 51)

    def _record_state(self, record: Optional[UserRecord]) -> Optional[dict[str, int]]:
        if record is None:
            return None

        length = int(record.get_length())
        dead = 1 if record.should_die() else 0
        level_id = int(
            StreakLevels.pick_by_length(length, record.is_freezed()).document_id
        )

        return {
            "length": length,
            "dead": dead,
            "level_id": level_id,
        }

    def _enqueue_streak_ended_popup(
        self, peer_id: int, days: int, name: Optional[str] = None
    ):
        days = int(days)
        if days < 3:
            return

        if name is None:
            name = self._resolve_peer_name(peer_id)

        self._enqueue_streak_popup_for_open_chat(
            int(peer_id),
            f"die:{peer_id}:{days}",
            self._t("popup_streak_ended_title"),
            self._t("popup_streak_ended_subtitle", name=name, days=days),
            StreakLevels.COLD.value.text_color_int,
            emoji_document_id=int(StreakLevels.COLD.value.document_id),
        )

    def _force_check_day_labels(
        self, include_today: bool, is_active: bool
    ) -> tuple[str, str]:
        mode = (
            self._t("force_check_day_mode_today")
            if include_today
            else self._t("force_check_day_mode_yesterday")
        )
        state = (
            self._t("force_check_day_state_active")
            if is_active
            else self._t("force_check_day_state_stop")
        )
        return mode, state

    def _maybe_notify_streak_transition(
        self,
        peer_id: int,
        before: Optional[dict[str, int]],
        after: Optional[dict[str, int]],
    ):
        if after is None:
            return

        name = self._resolve_peer_name(peer_id)
        after_length = int(after["length"])
        after_dead = bool(after["dead"])

        if after_dead:
            if (
                before is not None
                and not bool(before["dead"])
                and int(before["length"]) >= 3
            ):
                self._enqueue_streak_ended_popup(
                    int(peer_id), int(before["length"]), name=name
                )
                self._reload_user_in_messages_cache(int(peer_id))
                try:
                    self._rerender_dialog_cells()
                except Exception as e:
                    self.log(f"Failed to rerender dialogs after streak death: {e}")
            return

        was_started = after_length >= 3 and (
            before is None or bool(before["dead"]) or int(before["length"]) < 3
        )

        if was_started:
            key = f"start:{peer_id}:{after_length}"
            self._enqueue_streak_popup_for_open_chat(
                int(peer_id),
                key,
                self._t("popup_streak_started_title"),
                self._t("popup_streak_started_subtitle", name=name),
                StreakLevels.DAYS_3.value.text_color_int,
                emoji_document_id=int(after["level_id"]),
            )
            try:
                self._rerender_dialog_cells()
            except Exception as e:
                self.log(f"Failed to rerender dialogs after streak start: {e}")
            return

        if before is None:
            return

        upgraded = (
            after_length == 3
            or after_length == 10
            or after_length == 30
            or after_length == 100
            or after_length == 200
        )

        if int(before["level_id"]) != int(after["level_id"]) and upgraded:
            key = f"upgrade:{peer_id}:{after['level_id']}"
            upgraded_level_color = self._resolve_level_text_color(
                int(after["level_id"])
            )
            self._enqueue_streak_popup_for_open_chat(
                int(peer_id),
                key,
                self._t("popup_streak_upgraded_title", days=after_length),
                self._t("popup_streak_upgraded_subtitle", name=name),
                upgraded_level_color,
                emoji_document_id=int(after["level_id"]),
            )

    def _notify_for_dialog_event(
        self, peer_id: int, sender_type: SenderType, event_day: Optional[int] = None
    ):
        before = self._record_state(self.users_db.get_user(peer_id))
        self.streaks.on_dialog_event(peer_id, sender_type, event_day)
        self._repair_record_from_cached_today_activity(peer_id, event_day)
        after = self._record_state(self.users_db.get_user(peer_id))
        self._maybe_notify_streak_transition(peer_id, before, after)

        try:
            if self._should_refresh_streak_emoji(before, after):
                self._refresh_streak_emoji_for_peer(int(peer_id))
        except Exception as e:
            self.log(f"Failed to refresh streak emoji for peer {peer_id}: {e}")

    def _should_refresh_streak_emoji(
        self,
        before: Optional[dict[str, int]],
        after: Optional[dict[str, int]],
    ) -> bool:
        if before is None and after is None:
            return False

        before_has = (
            before is not None
            and not bool(before["dead"])
            and int(before["length"]) >= 3
        )
        after_has = (
            after is not None and not bool(after["dead"]) and int(after["length"]) >= 3
        )

        if before_has != after_has:
            return True

        if not after_has:
            return False

        if before is None or after is None:
            return True

        if int(before["length"]) != int(after["length"]):
            return True

        if int(before["level_id"]) != int(after["level_id"]):
            return True

        return False

    def _refresh_streak_emoji_for_peer(self, peer_id: int):
        user = None

        try:
            user = get_messages_controller().getUser(Long(int(peer_id)))
        except Exception:
            user = None

        if user is None:
            try:
                users = cast(
                    "ConcurrentHashMap[Long, TLRPC.User]",
                    get_private_field(get_messages_controller(), "users"),
                )
                user = users.get(Long(int(peer_id)))
            except Exception:
                user = None

        record = self.users_db.get_user(int(peer_id))
        has_streak_emoji = (
            record is not None
            and not bool(record.should_die())
            and int(record.get_length()) >= 3
        )

        if user is not None:
            if has_streak_emoji:
                self._patch_user_streak_emoji_status(user)
            else:
                try:
                    user.emoji_status = None
                except Exception:
                    pass
                self._post_user_emoji_status_updated(user)
                self._post_emoji_status_interface_update()
        else:
            self._post_emoji_status_interface_update()

        self._post_emoji_loaded_notification()
        self._rerender_dialog_cells()

    def _reload_user_in_messages_cache(self, peer_id: int):
        try:
            controller = get_messages_controller()
            controller.reloadUser(int(peer_id))
        except Exception as e:
            self.log(f"Failed to reload user {peer_id} in MessagesController: {e}")
            return

        def after_reload():
            try:
                self._post_emoji_loaded_notification()
                self._post_emoji_status_interface_update()
                self._rerender_dialog_cells()
            except Exception as e:
                self.log(f"Post-reload refresh failed for user {peer_id}: {e}")

        threading.Timer(0.8, lambda: run_on_ui_thread(after_reload)).start()

    def _sync_dead_states_snapshot(self):
        self._streak_dead_state = {}
        try:
            for record in self.users_db.list_users():
                self._streak_dead_state[int(record.user_id)] = bool(record.should_die())
        except Exception as e:
            self.log(f"Failed to init dead-state snapshot: {e}")

    def _check_dead_state_transitions(self):
        try:
            self.users_db.ensure_daily_backup()
        except Exception:
            pass

        try:
            records = self.users_db.list_users()
        except Exception:
            return

        seen: set[int] = set()

        for record in records:
            user_id = int(record.user_id)
            seen.add(user_id)
            is_dead_now = bool(record.should_die())
            was_dead = bool(self._streak_dead_state.get(user_id, is_dead_now))

            if not was_dead and is_dead_now:
                self._enqueue_streak_ended_popup(user_id, int(record.get_length()))
                self._reload_user_in_messages_cache(int(user_id))
                try:
                    self._rerender_dialog_cells()
                except Exception as e:
                    self.log(f"Failed to rerender dialogs on dead-state monitor: {e}")

            self._streak_dead_state[user_id] = is_dead_now

        missing = [
            user_id for user_id in self._streak_dead_state.keys() if user_id not in seen
        ]
        for user_id in missing:
            self._streak_dead_state.pop(user_id, None)

    def _start_dead_state_monitor(self):
        self._streak_dead_monitor_stop.clear()

        def worker():
            while not self._streak_dead_monitor_stop.wait(30):
                try:
                    self._check_dead_state_transitions()
                except Exception as e:
                    self.log(f"Dead-state monitor error: {e}")

        threading.Thread(target=worker, daemon=True).start()

    def _start_streak_popup_dispatch_monitor(self):
        self._streak_popup_dispatch_stop.clear()

        def worker():
            while not self._streak_popup_dispatch_stop.wait(1.0):
                try:
                    self._flush_pending_streak_popup_for_open_chat()
                except Exception as e:
                    self.log(f"Popup dispatch monitor error: {e}")

        threading.Thread(target=worker, daemon=True).start()

    def _register_chat_action_menu_items(self):
        try:
            item_id = self.add_menu_item(
                MenuItemData(
                    menu_type=MenuItemType.CHAT_ACTION_MENU,
                    text=self._t("menu_force_check_chat_text"),
                    icon="msg_retry",
                    subtext=self._t("menu_force_check_chat_subtext"),
                    on_click=lambda payload: self._on_force_check_current_chat_clicked(
                        payload
                    ),
                    priority=1000,
                )
            )
            self._chat_force_check_menu_item_id = item_id
        except Exception as e:
            self._chat_force_check_menu_item_id = None
            self.log(f"Failed to register chat action menu item: {e}")

        try:
            item_id = self.add_menu_item(
                MenuItemData(
                    menu_type=MenuItemType.CHAT_ACTION_MENU,
                    text=self._t("menu_go_to_streak_start_text"),
                    icon="other_chats",
                    subtext=self._t("menu_go_to_streak_start_subtext"),
                    on_click=lambda payload: self._on_go_to_streak_start_clicked(
                        payload
                    ),
                    priority=999,
                )
            )
            self._chat_go_to_streak_start_menu_item_id = item_id
        except Exception as e:
            self._chat_go_to_streak_start_menu_item_id = None
            self.log(f"Failed to register go-to-streak-start menu item: {e}")

        if not DEBUG_MODE:
            return

        try:
            item_id = self.add_menu_item(
                MenuItemData(
                    menu_type=MenuItemType.CHAT_ACTION_MENU,
                    text=self._t("menu_debug_create_streak_text"),
                    subtext=self._t("menu_debug_create_streak_subtext"),
                    on_click=lambda payload: self._on_debug_create_streak_clicked(
                        payload
                    ),
                    priority=998,
                )
            )
            self._chat_debug_create_streak_menu_item_id = item_id
        except Exception as e:
            self._chat_debug_create_streak_menu_item_id = None
            self.log(f"Failed to register debug-create menu item: {e}")

        try:
            item_id = self.add_menu_item(
                MenuItemData(
                    menu_type=MenuItemType.CHAT_ACTION_MENU,
                    text=self._t("menu_debug_kill_streak_text"),
                    subtext=self._t("menu_debug_kill_streak_subtext"),
                    on_click=lambda payload: self._on_debug_kill_streak_clicked(
                        payload
                    ),
                    priority=997,
                )
            )
            self._chat_debug_kill_streak_menu_item_id = item_id
        except Exception as e:
            self._chat_debug_kill_streak_menu_item_id = None
            self.log(f"Failed to register debug-kill menu item: {e}")

        try:
            item_id = self.add_menu_item(
                MenuItemData(
                    menu_type=MenuItemType.CHAT_ACTION_MENU,
                    text=self._t("menu_debug_upgrade_streak_text"),
                    subtext=self._t("menu_debug_upgrade_streak_subtext"),
                    on_click=lambda payload: self._on_debug_upgrade_streak_clicked(
                        payload
                    ),
                    priority=996,
                )
            )
            self._chat_debug_upgrade_streak_menu_item_id = item_id
        except Exception as e:
            self._chat_debug_upgrade_streak_menu_item_id = None
            self.log(f"Failed to register debug-upgrade menu item: {e}")

    def _remove_menu_item_safely(self, item_id: Optional[str], item_name: str):
        if not item_id:
            return

        try:
            self.remove_menu_item(item_id)
        except Exception as e:
            self.log(f"Failed to remove {item_name} menu item: {e}")

    def _unregister_chat_action_menu_items(self):
        self._remove_menu_item_safely(
            getattr(self, "_chat_force_check_menu_item_id", None), "force-check"
        )
        self._remove_menu_item_safely(
            getattr(self, "_chat_go_to_streak_start_menu_item_id", None),
            "go-to-streak-start",
        )
        self._remove_menu_item_safely(
            getattr(self, "_chat_debug_create_streak_menu_item_id", None),
            "debug-create",
        )
        self._remove_menu_item_safely(
            getattr(self, "_chat_debug_kill_streak_menu_item_id", None), "debug-kill"
        )
        self._remove_menu_item_safely(
            getattr(self, "_chat_debug_upgrade_streak_menu_item_id", None),
            "debug-upgrade",
        )
        self._chat_force_check_menu_item_id = None
        self._chat_go_to_streak_start_menu_item_id = None
        self._chat_debug_create_streak_menu_item_id = None
        self._chat_debug_kill_streak_menu_item_id = None
        self._chat_debug_upgrade_streak_menu_item_id = None

    def _extract_dialog_id_from_menu_payload(self, payload: Any) -> Optional[int]:
        if payload is None:
            return None

        payload_getter = None

        if isinstance(payload, dict):
            payload_getter = payload.get
        else:
            for key in (
                "dialog_id",
                "dialogId",
                "peer_id",
                "peerId",
                "chat_id",
                "chatId",
                "user_id",
                "userId",
            ):
                try:
                    value = payload.get(key)
                    payload_getter = payload.get
                except Exception:
                    break

                if value is None:
                    continue

                try:
                    dialog_id = int(value)
                    if dialog_id != 0:
                        return dialog_id
                except Exception:
                    continue

        if payload_getter is not None:
            for key in (
                "dialog_id",
                "dialogId",
                "peer_id",
                "peerId",
                "chat_id",
                "chatId",
                "user_id",
                "userId",
            ):
                value = payload_getter(key)
                if value is None:
                    continue

                try:
                    dialog_id = int(value)
                    if dialog_id != 0:
                        return dialog_id
                except Exception:
                    continue

            for key in ("chatActivity", "fragment", "chatFragment", "target", "object"):
                value = payload_getter(key)
                if value is None:
                    continue

                try:
                    dialog_id = int(value.getDialogId())
                    if dialog_id != 0:
                        return dialog_id
                except Exception:
                    continue

        try:
            dialog_id = int(payload.getDialogId())
            if dialog_id != 0:
                return dialog_id
        except Exception:
            return None

        return None

    def _extract_chat_activity_from_menu_payload(
        self, payload: Any
    ) -> Optional[ChatActivity]:
        if payload is None:
            return None

        candidates: list[Any] = [payload]
        payload_getter = None

        if isinstance(payload, dict):
            payload_getter = payload.get
        else:
            try:
                payload_getter = payload.get
            except Exception:
                payload_getter = None

        if payload_getter is not None:
            for key in ("chatActivity", "fragment", "chatFragment", "target", "object"):
                value = payload_getter(key)
                if value is not None:
                    candidates.append(value)

        for value in candidates:
            if value is None:
                continue

            try:
                dialog_id = int(value.getDialogId())
                if dialog_id != 0:
                    return cast("ChatActivity", value)
            except Exception:
                continue

        return None

    def _can_force_check_dialog_id(self, dialog_id: int) -> bool:
        if dialog_id <= 0:
            return False

        if not DialogObject.isUserDialog(dialog_id):
            return False

        if dialog_id in get_authorized_user_ids() or UserObject.isReplyUser(dialog_id):
            return False

        if UserObject.isService(dialog_id):
            return False

        user = self._resolve_dialog_user(get_messages_controller(), dialog_id)

        if user is None:
            return True

        if UserObject.isUserSelf(user):
            return False

        if UserObject.isDeleted(user):
            return False

        if UserObject.isBot(user):
            return False

        if UserObject.isReplyUser(user):
            return False

        return True

    def _find_first_message_id_in_range(
        self, peer: TLRPC.InputPeer, start_ts: int, end_ts: int
    ) -> Optional[int]:
        done = threading.Event()
        state = {
            "offset_id": 0,
            "offset_date": int(end_ts),
            "first_id": 0,
            "first_date": 0,
        }

        def load():
            req = TLRPC.TL_messages_getHistory()
            req.peer = peer
            req.offset_id = int(state["offset_id"])
            req.offset_date = int(state["offset_date"])
            req.limit = 100

            class Delegate(dynamic_proxy(RequestDelegate)):
                def run(self, response: TLObject, error: TLRPC.TL_error | None) -> None:
                    if error is not None:
                        done.set()
                        log(error)
                        return

                    response = cast("TLRPC.messages_Messages", response)
                    msgs = response.messages

                    if msgs is None or msgs.size() == 0:
                        done.set()
                        return

                    for i in range(msgs.size()):
                        message = msgs.get(i)
                        message_date = int(message.date)

                        if not (start_ts <= message_date < end_ts):
                            continue

                        message_id = int(message.id)
                        current_first_date = int(state["first_date"])
                        current_first_id = int(state["first_id"])

                        if (
                            current_first_id == 0
                            or message_date < current_first_date
                            or (
                                message_date == current_first_date
                                and message_id < current_first_id
                            )
                        ):
                            state["first_id"] = message_id
                            state["first_date"] = message_date

                    oldest = msgs.get(msgs.size() - 1)
                    oldest_date = int(oldest.date)
                    state["offset_id"] = int(oldest.id)
                    state["offset_date"] = oldest_date

                    if oldest_date >= start_ts:
                        load()
                    else:
                        done.set()

            get_connections_manager().sendRequest(req, Delegate())

        load()
        done.wait(timeout=60)

        first_id = int(state["first_id"])
        if first_id <= 0:
            return None

        return first_id

    def _on_force_check_current_chat_clicked(self, payload: Any):
        if not hasattr(self, "users_db"):
            self._show_error(self._t("err_users_db_not_ready"))
            return

        dialog_id = self._extract_dialog_id_from_menu_payload(payload)

        if dialog_id is None:
            self.log(f"Force check chat click payload missing dialog id: {payload}")
            self._show_error(self._t("err_cannot_detect_current_chat"))
            return

        if not self._can_force_check_dialog_id(dialog_id):
            self._show_info(self._t("info_private_user_only"))
            return

        if not self._try_start_force_check("info_force_check_started_chat"):
            return

        def worker():
            try:
                day_progress_state = {
                    "last_reported": 0,
                }

                def on_day_progress(
                    progress_user_id: int,
                    day_checked: int,
                    day_offset: int,
                    is_active: bool,
                    include_today: bool,
                ):
                    if progress_user_id != dialog_id:
                        return

                    if day_progress_state["last_reported"] == day_checked and is_active:
                        return

                    day_progress_state["last_reported"] = day_checked
                    mode, state = self._force_check_day_labels(include_today, is_active)
                    message = self._t(
                        "force_check_day_progress_chat",
                        days_checked=day_checked,
                        day_offset=day_offset,
                        state=state,
                        mode=mode,
                    )
                    self.log(message)
                    self._show_info(message)

                checked, updated, removed, unchanged = (
                    self.streaks.force_check_user_ids(
                        [dialog_id],
                        on_day_progress=on_day_progress,
                    )
                )
                self._patch_cached_users_streak_emoji_status()
                summary = self._t(
                    "force_check_summary_chat",
                    checked=checked,
                    updated=updated,
                    removed=removed,
                    unchanged=unchanged,
                )
                self.log(summary)
                self._show_success(summary)
            except Exception as e:
                self.log(f"Force check failed for chat {dialog_id}: {e}")
                self._show_error(self._t("err_force_check_failed_logs"))
            finally:
                self._finish_force_check()

        threading.Thread(target=worker, daemon=True).start()

    def _on_go_to_streak_start_clicked(self, payload: Any):
        if not hasattr(self, "users_db"):
            self._show_error(self._t("err_users_db_not_ready"))
            return

        dialog_id = self._extract_dialog_id_from_menu_payload(payload)

        if dialog_id is None:
            self.log(f"Go-to-streak-start click payload missing dialog id: {payload}")
            self._show_error(self._t("err_cannot_detect_current_chat"))
            return

        if not self._can_force_check_dialog_id(dialog_id):
            self._show_info(self._t("info_private_user_only"))
            return

        record = self.users_db.get_user(dialog_id)
        if record is None:
            self._show_info(self._t("info_no_streak_record_for_chat"))
            return

        chat_activity = self._extract_chat_activity_from_menu_payload(payload)
        if chat_activity is None:
            self.log(f"Go-to-streak-start payload missing ChatActivity: {payload}")
            self._show_error(self._t("err_cannot_open_chat_context"))
            return

        start_date = int(record.started_at)
        end_date = start_date + SECONDS_IN_DAY
        self._show_info(self._t("info_searching_streak_start_message"))

        def worker():
            message_id: Optional[int] = None

            try:
                peer = get_messages_controller().getInputPeer(dialog_id)  # ty:ignore[no-matching-overload]
                if peer is not None:
                    message_id = self._find_first_message_id_in_range(
                        peer, start_date, end_date
                    )
            except Exception as e:
                self.log(f"Go-to-streak-start lookup failed for {dialog_id}: {e}")

            def jump():
                try:
                    if message_id is not None and message_id > 0:
                        try:
                            chat_activity.scrollToMessageId(
                                message_id, 0, True, 0, True, 0
                            )
                            self._show_success(
                                self._t("ok_jumped_to_streak_start_message")
                            )
                            return
                        except Exception as e:
                            self.log(
                                f"Go-to-streak-start scroll failed for {dialog_id}: {e}"
                            )

                    chat_activity.jumpToDate(start_date)
                    self._show_info(self._t("info_exact_start_message_not_found"))
                except Exception as e:
                    self.log(f"Go-to-streak-start failed for {dialog_id}: {e}")
                    self._show_error(self._t("err_failed_jump_to_streak_start"))

            run_on_ui_thread(jump)

        threading.Thread(target=worker, daemon=True).start()

    def _extract_debug_target_dialog_id(self, payload: Any) -> Optional[int]:
        if not DEBUG_MODE:
            self._show_info(self._t("info_debug_mode_disabled"))
            return None

        if not hasattr(self, "users_db"):
            self._show_error(self._t("err_users_db_not_ready"))
            return None

        dialog_id = self._extract_dialog_id_from_menu_payload(payload)
        if dialog_id is None:
            self._show_error(self._t("err_cannot_detect_current_chat"))
            return None

        if not self._can_force_check_dialog_id(dialog_id):
            self._show_info(self._t("info_debug_private_user_only"))
            return None

        return dialog_id

    def _make_alive_record(self, dialog_id: int, target_length: int) -> UserRecord:
        target_length = max(int(target_length), 1)
        today = TimeUtils.get_stripped_timestamp()
        return UserRecord(
            user_id=int(dialog_id),
            started_at=today - ((target_length - 1) * SECONDS_IN_DAY),
            freezes_at=today + SECONDS_IN_DAY,
            last_sended_at=today,
            last_received_at=today,
        )

    def _persist_debug_record(
        self, before: Optional[dict[str, int]], record: UserRecord
    ):
        self.users_db.set_user_record(record)
        self._patch_cached_users_streak_emoji_status()
        after_record = self.users_db.get_user(record.user_id)
        after = self._record_state(after_record)
        self._maybe_notify_streak_transition(record.user_id, before, after)

        if after_record is not None:
            self._streak_dead_state[int(record.user_id)] = bool(
                after_record.should_die()
            )

    def _on_debug_create_streak_clicked(self, payload: Any):
        dialog_id = self._extract_debug_target_dialog_id(payload)
        if dialog_id is None:
            return

        before = self._record_state(self.users_db.get_user(dialog_id))
        record = self._make_alive_record(dialog_id, 3)
        self._persist_debug_record(before, record)
        self._show_success(self._t("ok_debug_streak_set_3"))

    def _on_debug_kill_streak_clicked(self, payload: Any):
        dialog_id = self._extract_debug_target_dialog_id(payload)
        if dialog_id is None:
            return

        current = self.users_db.get_user(dialog_id)
        if current is None or current.should_die() or current.get_length() < 3:
            current = self._make_alive_record(dialog_id, 3)

        before = self._record_state(current)
        today = TimeUtils.get_stripped_timestamp()
        dead_record = UserRecord(
            user_id=int(dialog_id),
            started_at=int(current.started_at),
            freezes_at=today - SECONDS_IN_DAY,
            last_sended_at=current.last_sended_at,
            last_received_at=current.last_received_at,
        )
        self._persist_debug_record(before, dead_record)
        self._show_success(self._t("ok_debug_streak_marked_dead"))

    def _on_debug_upgrade_streak_clicked(self, payload: Any):
        dialog_id = self._extract_debug_target_dialog_id(payload)
        if dialog_id is None:
            return

        current = self.users_db.get_user(dialog_id)
        if current is None or current.should_die():
            current = self._make_alive_record(dialog_id, 3)

        before = self._record_state(current)
        current_length = max(int(current.get_length()), 3)

        if current_length < 10:
            target_length = 10
        elif current_length < 30:
            target_length = 30
        elif current_length < 100:
            target_length = 100
        elif current_length < 200:
            target_length = 200
        else:
            self._show_info(self._t("info_debug_streak_already_max"))
            return

        upgraded = self._make_alive_record(dialog_id, target_length)
        self._persist_debug_record(before, upgraded)
        self._show_success(self._t("ok_debug_streak_upgraded", days=target_length))

    def _resolve_dialog_user(
        self, controller: MessagesController, dialog_id: int
    ) -> Optional[TLRPC.User]:
        user = None

        try:
            user = controller.getUser(Long(dialog_id))
        except Exception:
            user = None

        if user is not None:
            return user

        try:
            users = controller.getUsers()
            if users is not None:
                user = users.get(Long(dialog_id))
        except Exception:
            user = None

        return user

    def _collect_real_private_dialog_user_ids(self) -> list[int]:
        result: set[int] = set()
        controller = get_messages_controller()
        dialogs = getattr(controller, "allDialogs", None)

        if dialogs is None:
            dialogs = controller.getDialogs(0)

        if dialogs is None:
            return []

        self_user_ids = set(get_authorized_user_ids())

        for i in range(dialogs.size()):
            dialog = dialogs.get(i)

            if dialog is None:
                continue

            dialog_id = int(getattr(dialog, "id", 0))

            if dialog_id == 0:
                try:
                    DialogObject.initDialog(dialog)
                    dialog_id = int(getattr(dialog, "id", 0))
                except Exception:
                    dialog_id = 0

            if dialog_id == 0:
                try:
                    peer = getattr(dialog, "peer", None)
                    if peer is not None:
                        dialog_id = int(DialogObject.getPeerDialogId(peer))
                    else:
                        dialog_id = 0
                except Exception:
                    dialog_id = 0

            if dialog_id <= 0:
                continue

            if not DialogObject.isUserDialog(dialog_id):
                continue

            if dialog_id in self_user_ids or UserObject.isReplyUser(dialog_id):
                continue

            if UserObject.isService(dialog_id):
                continue

            user = self._resolve_dialog_user(controller, dialog_id)

            if user is not None:
                if UserObject.isUserSelf(user):
                    continue

                if UserObject.isDeleted(user):
                    continue

                if UserObject.isBot(user):
                    continue

                if UserObject.isReplyUser(user):
                    continue

            result.add(dialog_id)

        return sorted(result)

    def _on_force_check_all_chats_clicked(self):
        if not hasattr(self, "users_db"):
            self._show_error(self._t("err_users_db_not_ready"))
            return

        if not self._try_start_force_check("info_force_check_started_all"):
            return

        def worker():
            try:
                user_ids = self._collect_real_private_dialog_user_ids()
                total = len(user_ids)
                self.log(f"Force check targets: {total}")

                progress_lock = threading.Lock()
                progress_state = {
                    "last_bucket": -1,
                    "last_checked": -1,
                    "checked_chats": 0,
                    "total": total,
                }
                day_progress_state: dict[int, int] = {}
                peer_name_cache: dict[int, str] = {}

                def on_progress(
                    checked: int,
                    progress_total: int,
                    updated: int,
                    removed: int,
                    unchanged: int,
                ):
                    with progress_lock:
                        total_local = int(progress_total)
                        checked_local = int(checked)
                        progress_state["total"] = total_local
                        progress_state["checked_chats"] = checked_local

                        if total_local <= 0:
                            if progress_state["last_checked"] != 0:
                                progress_state["last_checked"] = 0
                                self.log(self._t("force_check_progress_zero"))
                            return

                        bucket = (checked_local * 10) // total_local

                        if (
                            checked_local != total_local
                            and checked_local > 0
                            and bucket == progress_state["last_bucket"]
                        ):
                            return

                        progress_state["last_bucket"] = bucket
                        progress_state["last_checked"] = checked_local

                    progress_message = self._t(
                        "force_check_progress",
                        checked=checked_local,
                        total=total_local,
                        updated=updated,
                        removed=removed,
                        unchanged=unchanged,
                    )
                    self.log(progress_message)
                    self._show_info(progress_message)

                def on_day_progress(
                    progress_user_id: int,
                    day_checked: int,
                    day_offset: int,
                    is_active: bool,
                    include_today: bool,
                ):
                    if (
                        day_progress_state.get(progress_user_id, 0) == int(day_checked)
                        and is_active
                    ):
                        return

                    day_progress_state[progress_user_id] = int(day_checked)
                    mode, state = self._force_check_day_labels(include_today, is_active)
                    checked_chats = int(progress_state.get("checked_chats", 0))
                    total_chats = int(progress_state.get("total", total))
                    peer_name = peer_name_cache.get(progress_user_id)

                    if peer_name is None:
                        peer_name = self._resolve_peer_name(progress_user_id)
                        peer_name_cache[progress_user_id] = peer_name

                    day_message = self._t(
                        "force_check_day_progress_all",
                        days_checked=day_checked,
                        peer_name=peer_name,
                        checked_chats=checked_chats,
                        total_chats=total_chats,
                        day_offset=day_offset,
                        state=state,
                        mode=mode,
                    )
                    self.log(day_message)
                    self._show_info(day_message)

                checked, updated, removed, unchanged = (
                    self.streaks.force_check_user_ids(
                        user_ids,
                        on_progress=on_progress,
                        on_day_progress=on_day_progress,
                    )
                )
                self._patch_cached_users_streak_emoji_status()
                summary = self._t(
                    "force_check_summary_all",
                    checked=checked,
                    updated=updated,
                    removed=removed,
                    unchanged=unchanged,
                )
                self.log(summary)
                self._show_success(summary)
            except Exception as e:
                self.log(f"Force check failed: {e}")
                self._show_error(self._t("err_force_check_failed_logs"))
            finally:
                self._finish_force_check()

        threading.Thread(target=worker, daemon=True).start()

    def _try_start_force_check(self, started_message_key: str) -> bool:
        with self._force_check_lock:
            if self._force_check_running:
                self._show_info(self._t("info_force_check_already_running"))
                return False
            self._force_check_running = True

        self._show_info(self._t(started_message_key))
        return True

    def _finish_force_check(self):
        with self._force_check_lock:
            self._force_check_running = False

    def _get_tl_name(self, value: Any) -> str:
        try:
            return cast("str", value.getClass().getSimpleName())
        except Exception:
            return cast("str", value.__class__.__name__)

    def _get_self_user_id(self, account: int) -> int:
        account = int(account)
        cache = cast("dict[int, int]", getattr(self, "_self_user_ids", {}))
        cached_id = cache.get(account, 0)

        if cached_id > 0:
            return cached_id

        try:
            user_config = UserConfig.getInstance(account)
            user_id = int(user_config.getClientUserId())

            if user_id > 0:
                cache[account] = user_id
                self._self_user_ids = cache

            return user_id
        except Exception:
            return 0

    def _extract_private_peer_id(self, peer: Any) -> Optional[int]:
        if peer is None:
            return None

        if isinstance(peer, int):
            return int(peer) if int(peer) > 0 else None

        try:
            class_name = cast("str", peer.getClass().getName())
        except Exception:
            class_name = ""

        if class_name in ("java.lang.Integer", "java.lang.Long"):
            try:
                peer_id = int(peer)
                if peer_id > 0:
                    return peer_id
            except Exception:
                pass

        user_id = getattr(peer, "user_id", None)

        if user_id is None:
            return None

        try:
            user_id = int(user_id)
        except Exception:
            return None

        if user_id <= 0:
            return None

        # Explicitly reject groups/channels.
        if int(getattr(peer, "chat_id", 0)) != 0:
            return None

        if int(getattr(peer, "channel_id", 0)) != 0:
            return None

        return user_id

    def _query_cached_day_activity_flags(
        self, dialog_id: int, day_start: int, day_end: int
    ) -> Optional[tuple[bool, bool]]:
        if int(dialog_id) <= 0:
            return None

        try:
            storage = get_account_instance().getMessagesStorage()
            db = storage.getDatabase()
        except Exception:
            return None

        table_name = _resolve_day_check_messages_table_name(db)
        if table_name is None:
            return None

        cursor = None

        try:
            cursor = db.queryFinalized(
                String(
                    f"SELECT "
                    f"MAX(CASE WHEN out = 1 THEN 1 ELSE 0 END), "
                    f"MAX(CASE WHEN out = 0 THEN 1 ELSE 0 END) "
                    f"FROM {table_name} "
                    f"WHERE uid = ? AND date >= ? AND date < ?"
                ),
                Integer(int(dialog_id)),
                Integer(int(day_start)),
                Integer(int(day_end)),
            )

            if cursor is None or not bool(cursor.next()):
                return None

            has_out = (0 if cursor.isNull(0) else int(cursor.intValue(0))) == 1
            has_in = (0 if cursor.isNull(1) else int(cursor.intValue(1))) == 1

            return (bool(has_out), bool(has_in))
        except Exception:
            return None
        finally:
            if cursor is not None:
                try:
                    cursor.dispose()
                except Exception:
                    pass

    def _repair_record_from_cached_today_activity(
        self, peer_id: int, event_day: Optional[int] = None
    ):
        current = self.users_db.get_user(int(peer_id), include_authorized=True)
        if current is None:
            return

        day = (
            TimeUtils.get_stripped_timestamp()
            if event_day is None
            else TimeUtils.strip_timestamp(event_day)
        )
        day_end = int(day + SECONDS_IN_DAY)

        flags = self._query_cached_day_activity_flags(int(peer_id), int(day), day_end)
        if flags is None:
            return

        has_out, has_in = flags
        if not has_out and not has_in:
            return

        updated = UserRecord.from_dict(current.to_dict())
        if updated is None:
            return

        changed = False
        if has_out:
            changed = updated.update_from(SenderType.SELF, int(day)) or changed
        if has_in:
            changed = updated.update_from(SenderType.PEER, int(day)) or changed

        if changed:
            self.users_db.set_user_record(updated)

    def _extract_private_peer_id_from_send_params(self, params: Any) -> Optional[int]:
        candidates: list[Any] = []

        direct_attrs = (
            "peer",
            "peer_id",
            "dialogId",
            "dialog_id",
            "uid",
            "user_id",
        )

        for attr_name in direct_attrs:
            try:
                value = getattr(params, attr_name, None)
            except Exception:
                value = None
            if value is not None:
                candidates.append(value)

        nested_roots = ("messageObject", "message", "obj", "messageOwner")
        for root_name in nested_roots:
            try:
                root = getattr(params, root_name, None)
            except Exception:
                root = None

            if root is None:
                continue

            candidates.append(root)

            for attr_name in ("peer", "peer_id", "dialogId", "dialog_id", "uid"):
                try:
                    value = getattr(root, attr_name, None)
                except Exception:
                    value = None
                if value is not None:
                    candidates.append(value)

            try:
                owner = getattr(root, "messageOwner", None)
            except Exception:
                owner = None

            if owner is not None:
                candidates.append(owner)
                for attr_name in ("peer_id", "from_id", "dialog_id", "dialogId"):
                    try:
                        value = getattr(owner, attr_name, None)
                    except Exception:
                        value = None
                    if value is not None:
                        candidates.append(value)

        for candidate in candidates:
            peer_id = self._extract_private_peer_id(candidate)
            if peer_id is not None and peer_id > 0:
                return int(peer_id)

        for candidate in candidates:
            try:
                dialog_id = int(candidate)
            except Exception:
                continue

            if dialog_id <= 0:
                continue

            try:
                if bool(DialogObject.isUserDialog(dialog_id)):
                    return int(dialog_id)
            except Exception:
                pass

        return None

    def _consume_message(self, account: int, message: Any):
        peer_id = self._extract_private_peer_id(getattr(message, "peer_id", None))

        if peer_id is None:
            return

        self_user_id = self._get_self_user_id(account)

        if self_user_id > 0 and peer_id == self_user_id:
            return

        sender_type = SenderType.PEER
        from_id = self._extract_private_peer_id(getattr(message, "from_id", None))
        message_out = bool(getattr(message, "out", False))

        if from_id is not None and self_user_id > 0:
            sender_type = (
                SenderType.SELF if from_id == self_user_id else SenderType.PEER
            )
        elif message_out:
            sender_type = SenderType.SELF

        message_date = int(getattr(message, "date", int(time.time())))
        self._notify_for_dialog_event(
            peer_id, sender_type, TimeUtils.strip_timestamp(message_date)
        )

    def _consume_update(self, account: int, update_name: str, update: Any):
        if update_name in ("TL_updateNewMessage", "TL_updateNewChannelMessage"):
            message = getattr(update, "message", None)

            if message is not None:
                self._consume_message(account, message)

            return

        if update_name == "TL_updateShortMessage":
            peer_id = int(getattr(update, "user_id", 0))

            if peer_id <= 0:
                return

            self_user_id = self._get_self_user_id(account)

            if self_user_id > 0 and peer_id == self_user_id:
                return

            sender_type = (
                SenderType.SELF
                if bool(getattr(update, "out", False))
                else SenderType.PEER
            )
            message_date = int(getattr(update, "date", int(time.time())))
            self._notify_for_dialog_event(
                peer_id, sender_type, TimeUtils.strip_timestamp(message_date)
            )
            return

        if update_name == "TL_updateShort":
            nested_update = getattr(update, "update", None)

            if nested_update is not None:
                self._consume_update(
                    account, self._get_tl_name(nested_update), nested_update
                )

            return

        if update_name in ("TL_updates", "TL_updatesCombined"):
            updates = getattr(update, "updates", None)

            if updates is None:
                return

            for i in range(updates.size()):
                nested_update = updates.get(i)
                self._consume_update(
                    account, self._get_tl_name(nested_update), nested_update
                )

    def _patch_cached_users_streak_emoji_status(self):
        users = cast(
            "ConcurrentHashMap[Long, TLRPC.User]",
            get_private_field(get_messages_controller(), "users"),
        )
        users_iter = users.values().iterator()

        for i in range(users.size()):
            self._patch_user_streak_emoji_status(users_iter.next())

    def _patch_user_streak_emoji_status(self, user: TLRPC.User):
        user_record = self.users_db.get_user(user.id)

        if user_record is None:
            return

        length = int(user_record.get_length())
        if length < 3:
            return

        user.emoji_status = TLRPC.TL_emojiStatus()
        user.emoji_status.document_id = StreakLevels.COLD.value.document_id
        user.premium = True
        self._post_user_emoji_status_updated(user)
        self._post_emoji_status_interface_update()

    def _post_user_emoji_status_updated(self, user: Optional[TLRPC.User]):
        if user is None:
            return

        def post():
            try:
                for i in range(UserConfig.MAX_ACCOUNT_COUNT):
                    cfg = UserConfig.getInstance(i)
                    if not bool(cfg.isClientActivated()):
                        continue
                    NotificationCenter.getInstance(i).postNotificationName(
                        NotificationCenter.userEmojiStatusUpdated, user
                    )
            except Exception as e:
                self.log(f"Failed to post userEmojiStatusUpdated: {e}")

        run_on_ui_thread(post)

    def _post_emoji_status_interface_update(self):
        def post():
            try:
                for i in range(UserConfig.MAX_ACCOUNT_COUNT):
                    cfg = UserConfig.getInstance(i)
                    if not bool(cfg.isClientActivated()):
                        continue
                    NotificationCenter.getInstance(i).postNotificationName(
                        NotificationCenter.updateInterfaces,
                        Integer(MessagesController.UPDATE_MASK_EMOJI_STATUS),
                    )
            except Exception as e:
                self.log(f"Failed to post emoji status updateInterfaces: {e}")

        run_on_ui_thread(post)

    def _post_emoji_loaded_notification(self):
        def post():
            try:
                for i in range(UserConfig.MAX_ACCOUNT_COUNT):
                    cfg = UserConfig.getInstance(i)
                    if not bool(cfg.isClientActivated()):
                        continue
                    NotificationCenter.getInstance(i).postNotificationName(
                        NotificationCenter.emojiLoaded
                    )
            except Exception as e:
                self.log(f"Failed to post emojiLoaded: {e}")

        run_on_ui_thread(post)

    def _post_dialogs_need_reload(self):
        def post():
            try:
                for i in range(UserConfig.MAX_ACCOUNT_COUNT):
                    cfg = UserConfig.getInstance(i)
                    if not bool(cfg.isClientActivated()):
                        continue
                    NotificationCenter.getInstance(i).postNotificationName(
                        NotificationCenter.dialogsNeedReload, Boolean(True)
                    )
            except Exception as e:
                self.log(f"Failed to post dialogsNeedReload: {e}")

        run_on_ui_thread(post)

    def hook_user_emoji_status_assign(self, patch_existing: bool):
        users = cast(
            "ConcurrentHashMap[Long, TLRPC.User]",
            get_private_field(get_messages_controller(), "users"),
        )
        users_hash = System.identityHashCode(users)

        def patch_user(user: TLRPC.User):
            self._patch_user_streak_emoji_status(user)

        class ConcurrentHashMap_putValHook(MethodHook):
            def __init__(self, plugin: TgStreaksPlugin, target_hash: int):
                self.plugin = plugin
                self.target_hash = target_hash

            def before_hooked_method(self, param):
                this_hash = System.identityHashCode(param.thisObject)

                if self.target_hash != this_hash:
                    return

                user = param.args[1]

                if user is None:
                    return

                user = cast("TLRPC.User", user)

                patch_user(user)

            def attach(plugin: TgStreaksPlugin, target_hash: int):
                # method name is randomized bc of proguard
                for method in users.getClass().getDeclaredMethods():
                    if method.getReturnType() != Object.getClass():
                        continue

                    if method.getParameterCount() != 3:
                        continue

                    if method.getParameterTypes() != [
                        Object.getClass(),
                        Object.getClass(),
                        Boolean.TYPE,
                    ]:
                        continue

                    self.hook_method(
                        method, ConcurrentHashMap_putValHook(self, target_hash)
                    )

        ConcurrentHashMap_putValHook.attach(self, users_hash)

        if not patch_existing:
            return

        users_iter = users.values().iterator()

        for i in range(users.size()):
            patch_user(users_iter.next())

    def _rerender_dialog_cells(self):
        def _invoke_title_action_for_all_dialogs_activities() -> int:
            def invalidate_dialogs_view_pages(fragment: Any) -> int:
                updated_pages = 0

                def invalidate_all_recycler_children(list_view: Any):
                    if list_view is None:
                        return

                    try:
                        child_count = int(list_view.getChildCount())
                    except Exception:
                        child_count = 0

                    for child_index in range(child_count):
                        try:
                            child = list_view.getChildAt(child_index)
                            if child is not None:
                                child.invalidate()
                        except Exception:
                            pass

                    recycler_groups = [
                        ("getCachedChildCount", "getCachedChildAt"),
                        ("getHiddenChildCount", "getHiddenChildAt"),
                        ("getAttachedScrapChildCount", "getAttachedScrapChildAt"),
                    ]

                    for count_method_name, get_method_name in recycler_groups:
                        try:
                            count_method = getattr(list_view, count_method_name)
                            get_method = getattr(list_view, get_method_name)
                            extra_count = int(count_method())
                        except Exception:
                            continue

                        for child_index in range(extra_count):
                            try:
                                child = get_method(child_index)
                                if child is not None:
                                    child.invalidate()
                            except Exception:
                                pass

                try:
                    pages = get_private_field(fragment, "viewPages")
                except Exception:
                    pages = None

                if pages is None:
                    return updated_pages

                try:
                    pages_len = int(len(pages))
                except Exception:
                    pages_len = 0

                for i in range(pages_len):
                    try:
                        page = pages[i]
                    except Exception:
                        continue

                    if page is None:
                        continue

                    try:
                        list_view = get_private_field(page, "listView")
                    except Exception:
                        list_view = None

                    if list_view is not None:
                        try:
                            list_view.stopScroll()
                        except Exception:
                            pass

                        try:
                            list_view.setItemViewCacheSize(0)
                        except Exception:
                            pass

                        try:
                            list_view.invalidateViews()
                        except Exception:
                            pass

                        try:
                            invalidate_all_recycler_children(list_view)
                        except Exception:
                            pass

                        try:
                            pool = list_view.getRecycledViewPool()
                            if pool is not None:
                                pool.clear()
                        except Exception:
                            pass

                        try:
                            adapter_to_reattach = list_view.getAdapter()
                        except Exception:
                            adapter_to_reattach = None

                        try:
                            if adapter_to_reattach is not None:
                                list_view.setAdapter(None)
                                list_view.setAdapter(adapter_to_reattach)
                        except Exception:
                            pass

                        try:
                            list_view.setItemViewCacheSize(2)
                        except Exception:
                            pass

                        try:
                            list_view.invalidate()
                            list_view.requestLayout()
                        except Exception:
                            pass

                    try:
                        adapter = get_private_field(page, "dialogsAdapter")
                    except Exception:
                        adapter = None

                    if adapter is not None:
                        try:
                            adapter.notifyDataSetChanged()
                        except Exception:
                            pass

                    updated_pages += 1

                return updated_pages

            launch = LaunchActivity.instance

            if launch is None:
                return 0

            applied = 0
            seen: set[int] = set()

            def apply_to_layout(field_name: str):
                nonlocal applied

                try:
                    layout = get_private_field(launch, field_name)
                    if layout is None:
                        return
                    stack = layout.getFragmentStack()
                    if stack is None:
                        return
                except Exception:
                    return

                try:
                    size = int(stack.size())
                except Exception:
                    size = 0

                for i in range(size):
                    try:
                        fragment = stack.get(i)
                    except Exception:
                        continue

                    if fragment is None:
                        continue

                    try:
                        class_name = cast("str", fragment.getClass().getName())
                    except Exception:
                        class_name = ""

                    if class_name != "org.telegram.ui.DialogsActivity":
                        continue

                    try:
                        fragment_hash = int(System.identityHashCode(fragment))
                    except Exception:
                        fragment_hash = id(fragment)

                    if fragment_hash in seen:
                        continue
                    seen.add(fragment_hash)

                    try:
                        action_bar = fragment.getActionBar()
                    except Exception:
                        action_bar = None

                    try:
                        if action_bar is not None and not bool(
                            action_bar.isSearchFieldVisible()
                        ):
                            runnable = get_private_field(
                                action_bar, "titleActionRunnable"
                            )
                            if runnable is not None:
                                runnable.run()
                                applied += 1
                    except Exception:
                        pass

                    try:
                        invalidate_dialogs_view_pages(fragment)
                    except Exception:
                        pass

            apply_to_layout("actionBarLayout")
            apply_to_layout("rightActionBarLayout")
            apply_to_layout("layersActionBarLayout")
            return applied

        def post_reload():
            try:
                self._post_emoji_loaded_notification()
                self._post_dialogs_need_reload()
                self._post_emoji_status_interface_update()
                try:
                    AnimatedEmojiDrawable.updateAll()
                except Exception:
                    pass
                applied = _invoke_title_action_for_all_dialogs_activities()
                self.log(
                    f"Posted emoji status update via NotificationCenter; executed ActionBar title action for DialogsActivity count={applied}"
                )
            except Exception as e:
                self.log(f"Failed to post emoji status NotificationCenter update: {e}")

        # run_on_ui_thread(post_reload)
        threading.Timer(0.5, lambda: run_on_ui_thread(post_reload)).start()

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
            self.log(f"Failed to infer JVM plugin version: {e}")
            return

        try:
            ref = self

            class Logger(dynamic_proxy(ValueCallback)):
                def onReceiveValue(self, var1):
                    ref.log(var1)

            class UserResolver(dynamic_proxy(Function)):
                def apply(self, t: Long):
                    if t <= 0:
                        return None

                    record = ref.users_db.get_user(t)

                    if record is not None:
                        length = record.get_length()
                        if int(length) < 3:
                            return None

                        streak_level = StreakLevels.pick_by_length(
                            length, record.is_freezed()
                        )

                        return jarray(Object)(
                            [
                                Integer(length),
                                Long(streak_level.document_id),
                                streak_level.text_color,
                            ],
                        )

                    return None

            class Translator(dynamic_proxy(Function)):
                def apply(self, t: String):
                    if t is None:
                        return ""
                    return ref._t(str(t))

            self.jvm_plugin.klass.getDeclaredMethod(
                String("inject"),
                ValueCallback.getClass(),  # ty:ignore[unresolved-attribute]
                Function.getClass(),  # ty:ignore[unresolved-attribute]
                Function.getClass(),  # ty:ignore[unresolved-attribute]
            ).invoke(None, Logger(), UserResolver(), Translator())  # ty:ignore[no-matching-overload]
            self.log("JVM plugin injected successfully")
        except Exception as e:
            self.log(f"Failed to inject JVM plugin: {e}")
            return

    def on_plugin_load(self):
        self._force_check_lock = threading.Lock()
        self._force_check_running = False
        self._chat_force_check_menu_item_id = None
        self._chat_go_to_streak_start_menu_item_id = None
        self._chat_debug_create_streak_menu_item_id = None
        self._chat_debug_kill_streak_menu_item_id = None
        self._chat_debug_upgrade_streak_menu_item_id = None
        self._streak_popup_last: dict[str, int] = {}
        self._streak_popup_queue_lock = threading.Lock()
        self._pending_streak_popups: dict[int, list[dict[str, Any]]] = {}
        self._streak_popup_dispatch_stop = threading.Event()
        self._streak_dead_state: dict[int, bool] = {}
        self._streak_dead_monitor_stop = threading.Event()
        self._update_check_stop = threading.Event()
        self._update_check_lock = threading.Lock()
        self._update_check_inflight = False

        self._load_jvm_plugin()

        self.add_on_send_message_hook()
        self.add_hook("TL_updateNewMessage")
        self.add_hook("TL_updateNewChannelMessage")
        self.add_hook("TL_updateShortMessage")
        self.add_hook("TL_updateShort")
        self.add_hook("TL_updates")
        self.add_hook("TL_updatesCombined")

        try:
            self.users_db = UsersDatabase(jvm_plugin=self.jvm_plugin, logger=self.log)
            self.streaks = StreaksController(users_db=self.users_db, logger=self.log)
            self._self_user_ids = {}

            checked, updated, removed = self.streaks.reconcile_on_plugin_load()
            if checked > 0:
                self.log(
                    f"Streak startup check done: checked={checked}, updated={updated}, removed={removed}"
                )

            self.hook_user_emoji_status_assign(True)
            self._rerender_dialog_cells()
            self._register_chat_action_menu_items()
            self._sync_dead_states_snapshot()
            self._start_dead_state_monitor()
            self._start_streak_popup_dispatch_monitor()
            self._start_update_check()
        except Exception as e:
            self.log(f"Exception: {e}")

    def on_plugin_unload(self):
        try:
            self._streak_dead_monitor_stop.set()
        except Exception:
            pass

        try:
            self._streak_popup_dispatch_stop.set()
        except Exception:
            pass

        try:
            self._update_check_stop.set()
        except Exception:
            pass

        self._unregister_chat_action_menu_items()

        if self.jvm_plugin.klass is None:
            return

        try:
            self.jvm_plugin.klass.getDeclaredMethod(String("eject")).invoke(None)  # ty:ignore[invalid-argument-type]
            self.log("JVM plugin ejected successfully")
        except Exception as e:
            self.log(f"Failed to eject JVM plugin: {e}")

        self.jvm_plugin.klass = None

    def on_send_message_hook(self, account: int, params: Any) -> HookResult:
        try:
            peer_id = self._extract_private_peer_id_from_send_params(params)

            if peer_id is None:
                return HookResult()

            self_user_id = self._get_self_user_id(account)

            if self_user_id > 0 and peer_id == self_user_id:
                return HookResult()

            self._notify_for_dialog_event(peer_id, SenderType.SELF)
        except Exception as e:
            self.log(f"send hook error: {e}")

        return HookResult()

    def on_update_hook(self, update_name: str, account: int, update: Any) -> HookResult:
        try:
            self._consume_update(account, update_name, update)
        except Exception as e:
            self.log(f"update hook error ({update_name}): {e}")

        return HookResult()
