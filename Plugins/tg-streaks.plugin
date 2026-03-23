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
from android.content import Intent
from android.graphics import Color
from android.net import Uri
from android.webkit import ValueCallback
from android_utils import run_on_ui_thread
from base_plugin import BasePlugin, MenuItemData, MenuItemType
from client_utils import get_last_fragment
from dalvik.system import InMemoryDexClassLoader
from java import dynamic_proxy
from java.lang import Class, Integer, Long, Runnable, String
from java.nio import ByteBuffer  # ty:ignore[unresolved-import]
from java.util.function import Function
from org.telegram.messenger import ApplicationLoader, LocaleController
from org.telegram.messenger import R as R_tg  # ty:ignore[unresolved-import]
from typing_extensions import Any
from ui.bulletin import BulletinHelper
from ui.settings import Divider, Header, Switch, Text

__id__ = "tg-streaks"
__name__ = "Streaks"
__description__ = "Analog for TikTok streaks for Telegram"
__author__ = "@n08i40k"
__version__ = "2.1.2"
__icon__ = "exteraPlugins/0"
__min_version__ = "12.1.1"

DEBUG_MODE = False


REPO_OWNER = "n08i40k"
REPO_NAME = __id__

DEX_URL = f"https://github.com/{REPO_OWNER}/{REPO_NAME}/releases/download/{__version__}/classes.dex"
DEX_SHA256 = "ca6f71245930151b7f15a2bbaca61e92e3b783f4070eb7efa4976c3ee63f973b"
RESOURCES_URL = f"https://github.com/{REPO_OWNER}/{REPO_NAME}/releases/download/{__version__}/resources.zip"
RESOURCES_SHA256 = "cbc31e8459db977596df510d5432e28b5bc39d33df736f29e49c5d2d9fcf19a4"

PLUGIN_UPDATE_API_URL = (
    f"https://api.github.com/repos/{REPO_OWNER}/{REPO_NAME}/releases/latest"
)
PLUGIN_UPDATE_TG_URL = "tg://resolve?domain=n08i40k_extera&post=3"
UPDATE_CHECK_TIMEOUT_SECONDS = 6
SETTING_UPDATE_CHECK_ENABLED = "update_check_enabled"


def get_plugin_cache_dir(*parts: str) -> str:
    cache_root = ApplicationLoader.applicationContext.getCacheDir().getAbsolutePath()
    return os.path.join(cache_root, __id__, *parts)

I18N_STRINGS: dict[str, dict[str, str]] = {
    "settings.updates": {"en": "Updates", "ru": "Обновления"},
    "settings.check_updates": {
        "en": "Check for plugin updates",
        "ru": "Проверять обновления плагина",
    },
    "settings.check_updates.hint": {
        "en": "Checks the latest GitHub release in the background when the plugin loads.",
        "ru": "При загрузке плагина в фоне проверяет последний релиз на GitHub.",
    },
    "settings.streak_tools": {"en": "Streak Tools", "ru": "Инструменты стрика"},
    "settings.force_check_all_private_chats": {
        "en": "Recalculate streaks in all private chats",
        "ru": "Пересчитать стрики во всех личных чатах",
    },
    "settings.only_private.hint": {
        "en": "Only private dialogs with real users are checked. Bots and groups are skipped.",
        "ru": "Проверяются только личные диалоги с реальными пользователями. Боты и группы пропускаются.",
    },
    "settings.db_backups": {"en": "Database Backups", "ru": "Резервные копии базы"},
    "settings.export_backup_now": {
        "en": "Create backup now",
        "ru": "Создать резервную копию сейчас",
    },
    "settings.import_latest_backup": {
        "en": "Restore from latest backup",
        "ru": "Восстановить из последней копии",
    },
    "settings.db_backups.hint": {
        "en": "Daily backups are created automatically. Import replaces current streak database.",
        "ru": "Ежедневные бэкапы создаются автоматически. Импорт заменяет текущую базу стриков.",
    },
    "err.cannot_detect_current_chat": {
        "en": "Cannot detect current chat",
        "ru": "Не удалось определить текущий чат",
    },
    "err.cannot_open_chat_context": {
        "en": "Cannot open chat context for jump",
        "ru": "Не удалось открыть контекст чата для перехода",
    },
    "err.failed_jump_to_streak_start": {
        "en": "Failed to jump to streak start",
        "ru": "Не удалось перейти к началу стрика",
    },
    "err.backup_export_failed": {
        "en": "Backup export failed",
        "ru": "Ошибка экспорта бэкапа",
    },
    "err.failed_open_update_link": {
        "en": "Failed to open update link",
        "ru": "Не удалось открыть ссылку обновления",
    },
    "ok.backup_exported": {
        "en": "Backup exported: {name}",
        "ru": "Бэкап экспортирован: {name}",
    },
    "ok.backup_imported": {
        "en": "Backup imported: {name}",
        "ru": "Бэкап импортирован: {name}",
    },
    "ok.jumped_to_streak_start_message": {
        "en": "Jumped to streak start message",
        "ru": "Переход к сообщению начала стрика выполнен",
    },
    "ok.debug_streak_set_3": {
        "en": "Debug: streak set to 3 days",
        "ru": "Debug: стрик установлен на 3 дня",
    },
    "ok.debug_streak_marked_dead": {
        "en": "Debug: streak marked as dead",
        "ru": "Debug: стрик помечен как мёртвый",
    },
    "ok.debug_streak_upgraded": {
        "en": "Debug: streak upgraded to {days} days",
        "ru": "Debug: стрик улучшен до {days} дней",
    },
    "ok.debug_streak_frozen": {
        "en": "Debug: streak frozen",
        "ru": "Debug: стрик заморожен",
    },
    "ok.debug_streak_deleted": {
        "en": "Debug: streak deleted",
        "ru": "Debug: стрик удалён",
    },
    "ok.upgrade_service_messages_enabled": {
        "en": "Upgrade service messages enabled for this chat",
        "ru": "Сервисные сообщения об апгрейде включены для этого чата",
    },
    "ok.upgrade_service_messages_disabled": {
        "en": "Upgrade service messages disabled for this chat",
        "ru": "Сервисные сообщения об апгрейде выключены для этого чата",
    },
    "ok.streak_restored": {"en": "Streak restored", "ru": "Стрик восстановлен"},
    "info.private_user_only": {
        "en": "This action works only for private user chats",
        "ru": "Это действие работает только в личных чатах с пользователями",
    },
    "info.action_not_available_for_bots": {
        "en": "This action is not available in chats with bots",
        "ru": "Это действие недоступно в чатах с ботами",
    },
    "info.action_not_available_for_deleted_users": {
        "en": "This action is not available for deleted accounts",
        "ru": "Это действие недоступно для удалённых аккаунтов",
    },
    "info.streak_not_ended_yet": {
        "en": "This streak hasn't ended yet",
        "ru": "Этот стрик ещё не завершился",
    },
    "info.streak_restore_unavailable": {
        "en": "This streak can no longer be restored",
        "ru": "Этот стрик больше нельзя восстановить",
    },
    "info.force_check_already_running": {
        "en": "Force check is already running",
        "ru": "Принудительная проверка уже выполняется",
    },
    "info.force_check_started_all": {
        "en": "Force check started",
        "ru": "Принудительная проверка запущена",
    },
    "info.searching_streak_start_message": {
        "en": "Searching streak start message...",
        "ru": "Ищу сообщение начала стрика...",
    },
    "info.exact_start_message_not_found": {
        "en": "Exact start message not found, jumped to streak start day",
        "ru": "Точное сообщение начала не найдено, выполнен переход к дню начала стрика",
    },
    "info.no_streak_record_for_chat": {
        "en": "No streak record for this chat",
        "ru": "Для этого чата нет записи стрика",
    },
    "info.debug_private_user_only": {
        "en": "Debug actions work only for private user chats",
        "ru": "Debug-действия работают только в личных чатах с пользователями",
    },
    "info.debug_streak_already_max": {
        "en": "Debug: streak is already at max level",
        "ru": "Debug: стрик уже на максимальном уровне",
    },
    "dex_sheet.feature_how.title": {
        "en": "How does this work?",
        "ru": "Как это работает?",
    },
    "dex_sheet.feature_how.subtitle": {
        "en": "After three days of communication in a row, you will have a streak that improves depending on its duration!",
        "ru": "После трёх дней общения подряд у вас появится стрик, который улучшается в зависимости от длительности!",
    },
    "dex_sheet.feature_levels.title": {
        "en": "What levels are there?",
        "ru": "Какие уровни есть?",
    },
    "dex_sheet.feature_levels.subtitle": {
        "en": "There are levels for 3, 10, 30, 100, and 200+ consecutive days of communication. A pop-up will appear when your streak level improves.",
        "ru": "Есть уровни за 3, 10, 30, 100 и 200+ дней общения подряд. При повышении уровня стрика появится всплывающее окно.",
    },
    "dex_sheet.feature_keep.title": {
        "en": "Don't forget about it ;)",
        "ru": "Не забывайте про него ;)",
    },
    "dex_sheet.feature_keep.subtitle": {
        "en": "If you don't manage to message each other within 24 hours, the streak will end. It can be restored only within the next 24 hours.",
        "ru": "Если вы не успеете написать друг другу в течение 24 часов, стрик завершится. Его можно восстановить только в течение следующих 24 часов.",
    },
    "dex_sheet.feature_incorrect.title": {
        "en": "Streak duration is incorrect?",
        "ru": "Длительность стрика неверная?",
    },
    "dex_sheet.feature_incorrect.subtitle": {
        "en": 'This can be fixed! Tap "Recalculate streak in this chat" to recount the streak length.',
        "ru": 'Это можно исправить! Нажмите "Пересчитать стрик в этом чате", чтобы заново посчитать его длину.',
    },
    "dex_sheet.title": {
        "en": "You and {name} have been on a streak for {days} days now!",
        "ru": "У Вас и {name} стрик уже более {days} дней!",
    },
    "dex_sheet.subtitle": {
        "en": "Keep chatting and keep the streak going **:P**",
        "ru": "Продолжайте общаться и не прерывайте стрик **:P**",
    },
    "menu.force_check_chat.text": {
        "en": "Recalculate streak in this chat",
        "ru": "Пересчитать стрик в этом чате",
    },
    "menu.force_check_chat.subtext": {
        "en": "Check the chat history again and update the streak length",
        "ru": "Ещё раз проверить историю чата и обновить длину стрика",
    },
    "menu.go_to_streak_start.text": {
        "en": "Open where the streak began",
        "ru": "Открыть начало стрика",
    },
    "menu.go_to_streak_start.subtext": {
        "en": "Jump to the message or day where the current streak started",
        "ru": "Перейти к сообщению или дню, с которого начался текущий стрик",
    },
    "menu.upgrade_service_messages.text": {
        "en": "Toggle service messages",
        "ru": "Переключить сервисные сообщения",
    },
    "menu.upgrade_service_messages.subtext": {
        "en": "Show or hide service messages in this chat when the streak reaches a new level",
        "ru": "Показывать или скрывать в этом чате служебные сообщения о новом уровне стрика",
    },
    "menu.restore_streak.text": {"en": "Restore streak", "ru": "Восстановить стрик"},
    "menu.restore_streak.subtext": {
        "en": "Available only during the first 24 hours after the streak ends",
        "ru": "Доступно только в течение первых 24 часов после прерывания стрика",
    },
    "menu.debug_create_streak.text": {
        "en": "[DEBUG] Create 3-day streak",
        "ru": "[DEBUG] Создать стрик на 3 дня",
    },
    "menu.debug_create_streak.subtext": {
        "en": "Set current chat streak to 3 days",
        "ru": "Установить стрик текущего чата на 3 дня",
    },
    "menu.debug_crash_plugin.text": {
        "en": "[DEBUG] Crash plugin",
        "ru": "[DEBUG] Крашнуть плагин",
    },
    "menu.debug_crash_plugin.subtext": {
        "en": ":)",
        "ru": ":)",
    },
    "menu.debug_kill_streak.text": {
        "en": "[DEBUG] Kill streak",
        "ru": "[DEBUG] Убить стрик",
    },
    "menu.debug_kill_streak.subtext": {
        "en": "Force streak death for current chat",
        "ru": "Принудительно завершить стрик в текущем чате",
    },
    "menu.debug_upgrade_streak.text": {
        "en": "[DEBUG] Upgrade streak",
        "ru": "[DEBUG] Улучшить стрик",
    },
    "menu.debug_upgrade_streak.subtext": {
        "en": "Upgrade current chat to next streak level",
        "ru": "Повысить стрик текущего чата до следующего уровня",
    },
    "menu.debug_freeze_streak.text": {
        "en": "[DEBUG] Freeze streak",
        "ru": "[DEBUG] Заморозить стрик",
    },
    "menu.debug_freeze_streak.subtext": {
        "en": "Simulate a frozen streak in the current chat",
        "ru": "Симулировать замороженный стрик в текущем чате",
    },
    "menu.debug_delete_streak.text": {
        "en": "[DEBUG] Delete streak",
        "ru": "[DEBUG] Удалить стрик",
    },
    "menu.debug_delete_streak.subtext": {
        "en": "Delete the streak record for the current chat",
        "ru": "Удалить запись стрика для текущего чата",
    },
    "force_check.day_progress_chat": {
        "en": "Recalculating streak with {peer_name}: checked {days_checked} d.",
        "ru": "Пересчитываю стрик с {peer_name}: проверено {days_checked} д.",
    },
    "force_check.day_progress_all_simple": {
        "en": "Recalculating streaks: {peer_name} [{checked_chats}/{total_chats}], checked {days_checked} d.",
        "ru": "Пересчитываю стрики: {peer_name} [{checked_chats}/{total_chats}], проверено {days_checked} д.",
    },
    "force_check.summary_all_simple": {
        "en": "Rebuild completed for {checked} private chats",
        "ru": "Ребилд завершён для {checked} личных чатов",
    },
    "force_check.retry_delay": {
        "en": "Telegram did not respond. Next attempt in {seconds} s.",
        "ru": "Telegram не ответил. Следующая попытка через {seconds} сек.",
    },
    "db.err.no_backups_found": {"en": "No backups found", "ru": "Бэкапы не найдены"},
    "db.err.failed_apply_backup": {
        "en": "Failed to apply backup: {reason}",
        "ru": "Не удалось применить бэкап: {reason}",
    },
    "update.bulletin.text": {
        "en": "Plugin update available: {current} -> {latest}",
        "ru": "Доступно обновление плагина: {current} -> {latest}",
    },
    "update.bulletin.button": {"en": "Update", "ru": "Обновить"},
    "download.dex.started": {
        "en": "Downloading plugin engine...",
        "ru": "Скачивание движка плагина...",
    },
    "download.dex.completed": {
        "en": "Plugin engine download completed",
        "ru": "Движок плагина скачан",
    },
    "download.resources.started": {
        "en": "Downloading streak resources...",
        "ru": "Скачивание ресурсов стриков...",
    },
    "download.resources.completed": {
        "en": "Streak resources download completed",
        "ru": "Ресурсы стриков скачаны",
    },
    "download.progress.subtitle": {
        "en": "{percent}% • {downloaded}/{total} • ETA {eta}",
        "ru": "{percent}% • {downloaded}/{total} • ETA {eta}",
    },
    "download.progress.subtitle_unknown": {
        "en": "{downloaded} downloaded • ETA calculating...",
        "ru": "Скачано {downloaded} • ETA рассчитывается...",
    },
    "service_message.create.text": {
        "en": "Streak started!",
        "ru": "Стрик начался!",
    },
    "service_message.upgrade.text": {
        "en": "Streak upgraded to {days} days!",
        "ru": "Стрик достиг {days} дней!",
    },
    "service_message.death.title": {
        "en": "Your streak has ended!",
        "ru": "Ваш стрик завершился!",
    },
    "service_message.death.subtitle": {
        "en": "You can restore it within the next 24 hours",
        "ru": "Вы можете восстановить его в течение следующих 24 часов",
    },
    "service_message.death.hint": {
        "en": "Tap the button below to restore your streak",
        "ru": "Нажмите кнопку ниже, чтобы восстановить стрик",
    },
    "service_message.death.button": {"en": "Restore", "ru": "Восстановить"},
    "service_message.restore.text.self": {
        "en": "You restored the streak!",
        "ru": "Вы восстановили стрик!",
    },
    "service_message.restore.text.peer": {
        "en": "{name} restored the streak!",
        "ru": "{name} восстановил(а) стрик!",
    },
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
    DAYS_3 = StreakLevel(3, 5285079178964672780, "3.webp", (255, 154, 0))
    DAYS_10 = StreakLevel(10, 5285274844789777412, "10.webp", (255, 100, 0))
    DAYS_30 = StreakLevel(30, 5285076623459129616, "30.webp", (255, 61, 0))
    DAYS_100 = StreakLevel(100, 5285003347022093599, "100.webp", (255, 0, 200))
    DAYS_200 = StreakLevel(200, 5285514817497504375, "200.webp", (176, 0, 255))

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
            self.plugin.log("Debug mode enabled. Downloading DEX without SHA256 checks...")
            dex_data = self._download_bytes(show_bulletins=True)
            if dex_data is not None:
                self._write_dex_file(dex_data)
                self._load(dex_data)
                return

            self.plugin.log("DEX download failed in debug mode. Falling back to cached DEX if available...")
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

            self.plugin.log("Cached DEX exists but failed to load. Downloading replacement synchronously...")
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
                started_key="download.dex.started",
                completed_key="download.dex.completed",
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

        threading.Thread(target=worker, name="tg-streaks-dex-refresh", daemon=True).start()

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
                self.plugin.log("Debug mode enabled. Downloading resources ZIP without SHA256 checks...")
                zip_data = self._download_bytes(show_bulletins=True)
                if zip_data is not None:
                    self._write_zip_file(zip_data)
                    self._extract_zip()
                    return self.resources_root if os.path.isdir(self.resources_root) else None
            finally:
                self._release_download_lock(lock_fd)
        else:
            self.plugin.log("Resources ZIP download is already in progress. Waiting for the existing run to finish...")
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

        self.plugin.log("Resources ZIP download failed in debug mode. Falling back to current extracted resources if available...")
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
            self.plugin.log("Resources ZIP update is already in progress. Waiting for the existing run to finish...")
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
                self.plugin.log("Resources ZIP is cached, but unpacked files are missing. Extracting...")
                self._extract_zip()
            return self.resources_root if os.path.isdir(self.resources_root) else None, False

        if cached_sha256 is not None:
            if not os.path.isdir(self.resources_root):
                self.plugin.log("Cached resources ZIP is outdated, but unpacked files are missing. Extracting cached version first...")
                self._extract_zip()

            if os.path.isdir(self.resources_root):
                self.plugin.log(
                    f"Cached resources ZIP SHA256 mismatch (cached={cached_sha256}, expected={expected_sha256}). Using current resources and refreshing in background..."
                )
                return self.resources_root, allow_download

            if not allow_download:
                return None, False

            self.plugin.log("Cached resources ZIP exists but there are no usable extracted files. Downloading replacement synchronously...")
        else:
            if not allow_download:
                return None, False

            self.plugin.log("Cached resources ZIP not found. Downloading new file...")

        zip_data = self._download_bytes(show_bulletins=True)
        if zip_data is None:
            return self.resources_root if os.path.isdir(self.resources_root) else None, False

        downloaded_sha256 = self._compute_sha256(zip_data)
        if downloaded_sha256 != expected_sha256:
            raise RuntimeError("Downloaded resources ZIP SHA256 mismatch")

        self._write_zip_file(zip_data)
        self._extract_zip()

        return self.resources_root if os.path.isdir(self.resources_root) else None, False

    def _compute_sha256(self, data: bytes) -> str:
        return hashlib.sha256(data).hexdigest().lower()

    def _compute_file_sha256(self, path: str) -> Optional[str]:
        if not os.path.exists(path):
            return None

        try:
            with open(path, "rb") as f:
                return self._compute_sha256(f.read())
        except Exception as e:
            self.plugin.log_exception("Failed to read cached resources ZIP for SHA256", e)
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
                started_key="download.resources.started",
                completed_key="download.resources.completed",
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
                if cached_sha256 == expected_sha256 and os.path.isdir(self.resources_root):
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

        threading.Thread(target=worker, name="tg-streaks-resources-refresh", daemon=True).start()

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

                    target_path = os.path.abspath(os.path.join(staging_root, normalized))
                    if not target_path.startswith(os.path.abspath(staging_root) + os.sep):
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
    REBUILD = "rebuild"
    TOGGLE_SERVICE_MESSAGES = "serviceMessages.toggle"
    GO_TO_STREAK_START = "goToStreakStart"
    REVIVE_NOW = "reviveNow"

    DEBUG_CREATE = "debug.create"
    DEBUG_UPGRADE = "debug.upgrade"
    DEBUG_FREEZE = "debug.freeze"
    DEBUG_KILL = "debug.kill"
    DEBUG_DELETE = "debug.delete"
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
                "key": cls.REBUILD,
                "text_key": "menu.force_check_chat.text",
                "subtext_key": "menu.force_check_chat.subtext",
                "icon": "msg_retry",
                "priority": 1000,
            },
            {
                "key": cls.GO_TO_STREAK_START,
                "text_key": "menu.go_to_streak_start.text",
                "subtext_key": "menu.go_to_streak_start.subtext",
                "icon": "other_chats",
                "priority": 999,
            },
            {
                "key": cls.TOGGLE_SERVICE_MESSAGES,
                "text_key": "menu.upgrade_service_messages.text",
                "subtext_key": "menu.upgrade_service_messages.subtext",
                "icon": "msg_settings",
                "priority": 998,
            },
            {
                "key": cls.REVIVE_NOW,
                "text_key": "menu.restore_streak.text",
                "subtext_key": "menu.restore_streak.subtext",
                "icon": "msg_reactions",
                "priority": 997,
            },
            {
                "key": cls.DEBUG_CREATE,
                "text_key": "menu.debug_create_streak.text",
                "subtext_key": "menu.debug_create_streak.subtext",
                "priority": 996,
                "debug_only": True,
            },
            {
                "key": cls.DEBUG_UPGRADE,
                "text_key": "menu.debug_upgrade_streak.text",
                "subtext_key": "menu.debug_upgrade_streak.subtext",
                "priority": 995,
                "debug_only": True,
            },
            {
                "key": cls.DEBUG_FREEZE,
                "text_key": "menu.debug_freeze_streak.text",
                "subtext_key": "menu.debug_freeze_streak.subtext",
                "priority": 994,
                "debug_only": True,
            },
            {
                "key": cls.DEBUG_KILL,
                "text_key": "menu.debug_kill_streak.text",
                "subtext_key": "menu.debug_kill_streak.subtext",
                "priority": 993,
                "debug_only": True,
            },
            {
                "key": cls.DEBUG_DELETE,
                "text_key": "menu.debug_delete_streak.text",
                "subtext_key": "menu.debug_delete_streak.subtext",
                "priority": 992,
                "debug_only": True,
            },
            {
                "key": cls.DEBUG_CRASH,
                "text_key": "menu.debug_crash_plugin.text",
                "subtext_key": "menu.debug_crash_plugin.subtext",
                "priority": 991,
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
            self.plugin._show_error(self.plugin._t("err.cannot_detect_current_chat"))
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
    IMPORT_LATEST_BACKUP = "importLatestBackup"

    def __init__(self, plugin: "TgStreaksPlugin"):
        self.plugin = plugin

    def build_settings(self) -> list[Any]:
        return [
            Header(text=self.plugin._t("settings.streak_tools")),
            Text(
                text=self.plugin._t("settings.force_check_all_private_chats"),
                icon="msg_retry",
                on_click=lambda _: self._on_click(self.REBUILD_ALL),
            ),
            Divider(text=self.plugin._t("settings.only_private.hint")),
            Header(text=self.plugin._t("settings.db_backups")),
            Text(
                text=self.plugin._t("settings.export_backup_now"),
                icon="msg_save",
                on_click=lambda _: self._on_click(self.EXPORT_BACKUP_NOW),
            ),
            Text(
                text=self.plugin._t("settings.import_latest_backup"),
                icon="msg_reset",
                on_click=lambda _: self._on_click(self.IMPORT_LATEST_BACKUP),
            ),
            Divider(text=self.plugin._t("settings.db_backups.hint")),
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
            "update.bulletin.text",
            current=__version__,
            latest=latest_version,
        )
        button_text = self.plugin._t("update.bulletin.button")

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
    _reload_lock = threading.Lock()

    settings_actions: SettingsActions

    def log_exception(self, message: str, exception: BaseException):
        self.log(f"{message}: {exception}")

        for chunk in traceback.format_exception(
            type(exception),
            exception,
            exception.__traceback__,
        ):
            for line in chunk.rstrip().splitlines():
                if len(line) > 0:
                    self.log(line)

    def create_settings(self) -> list[Any]:
        if not hasattr(self, "settings_actions"):
            self.settings_actions = SettingsActions(self)

        return [
            Header(text=self._t("settings.updates")),
            Switch(
                key=SETTING_UPDATE_CHECK_ENABLED,
                text=self._t("settings.check_updates"),
                default=self._is_update_check_enabled(),
                subtext=self._t("settings.check_updates.hint"),
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
                    "download.progress.subtitle",
                    percent=str(int(downloaded * 100 / total_bytes)),
                    downloaded=self._format_download_size(downloaded),
                    total=self._format_download_size(total_bytes),
                    eta=self._format_eta(remaining_seconds),
                )
            else:
                subtitle = self._t(
                    "download.progress.subtitle_unknown",
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
                self._show_error(self._t("err.failed_open_update_link"))

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

            class ReloadCallback(dynamic_proxy(Runnable)):
                def run(self):
                    ref._schedule_full_reload("Reload requested by JVM plugin")

            self.jvm_plugin.klass.getDeclaredMethod(
                String("inject"),
                ValueCallback.getClass(),  # ty:ignore[unresolved-attribute]
                Function.getClass(),  # ty:ignore[unresolved-attribute]
                String.getClass(),  # ty:ignore[unresolved-attribute]
                Runnable.getClass(),  # ty:ignore[unresolved-attribute]
            ).invoke(
                None,
                Logger(),
                TranslationResolver(),
                String(self.resources_bridge.resources_root),
                ReloadCallback(),
            )  # ty:ignore[no-matching-overload]
            self.log("JVM plugin injected successfully")
        except Exception as e:
            self.log_exception("Failed to inject JVM plugin", e)

    def _schedule_full_reload(self, reason: str):
        if not self._reload_lock.acquire(blocking=False):
            self.log(f"Skipped duplicate plugin reload request: {reason}")
            return

        def worker():
            try:
                self.log(f"Starting full plugin reload: {reason}")

                try:
                    self.on_plugin_unload()
                except BaseException as e:
                    self.log_exception("Failed during plugin unload while reloading", e)

                try:
                    self.on_plugin_load()
                except BaseException as e:
                    self.log_exception("Failed during plugin load while reloading", e)
                    return

                self.log("Full plugin reload completed")
            finally:
                self._reload_lock.release()

        threading.Thread(
            target=worker,
            name="tg-streaks-full-reload",
            daemon=True,
        ).start()

    def on_plugin_load(self):
        self.resources_bridge = ZipResourcesBridge(self)
        self._load_jvm_plugin()

        self._register_streak_levels()

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
