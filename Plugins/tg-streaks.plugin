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
from java import dynamic_proxy, jarray, jbyte
from android.content import Intent
from android.content import DialogInterface
from android.graphics import Color
from android.net import Uri
from android.os import Environment, Process
from android.util import Log
from android.webkit import ValueCallback
from android_utils import copy_to_clipboard, run_on_ui_thread
from base_plugin import BasePlugin, MenuItemData, MenuItemType, MethodHook
from client_utils import get_last_fragment
from dalvik.system import InMemoryDexClassLoader
from java.lang import Class, Integer, Long, String
from java.nio import ByteBuffer  # ty:ignore[unresolved-import]
from java.util import Locale
from org.telegram.messenger import ApplicationLoader, LocaleController
from org.telegram.messenger import R as R_tg  # ty:ignore[unresolved-import]
from org.telegram.ui.ActionBar import AlertDialog
from typing_extensions import Any
from ui.bulletin import BulletinHelper
from ui.settings import Divider, Header, Selector, Switch, Text

# fmt: off

__id__ = "tg-streaks"
__name__ = "Streaks"
__description__ = "Аналог стриков TikTok для Telegram"
__author__ = "@n08i40k_extera & @RoflPlugins"
__version__ = "2.16.0"
__icon__ = "tiktok_streak/4"
__min_version__ = "12.1.1"

# Core constants

DEBUG_MODE = False
LOGCAT_TAG = __id__

# Helper constants

REPO_OWNER = "n08i40k"
REPO_NAME = __id__

# External resource urls

DEX_URL = f"https://github.com/{REPO_OWNER}/{REPO_NAME}/releases/download/{__version__}/classes.dex"
RESOURCES_URL = f"https://github.com/{REPO_OWNER}/{REPO_NAME}/releases/download/{__version__}/resources.zip"

# Resource hashes

DEX_SHA256 = "c9df4f90e479f73ef5b7fa3e153316ed28f1eac46f6f0943127ee47a481c54af"
RESOURCES_SHA256 = "268cde673c98d59778384f16ca356cde3203faf328d2fdaa747200e9b6ca52b5"

# Plugin official resource links

PLUGIN_UPDATE_TG_URL = "tg://resolve?domain=n08i40k_extera&post=3"
PLUGIN_CHAT_TG_URL = "tg://resolve?domain=n08i40k_extera_chat"
PLUGIN_UPDATE_API_URL = f"https://api.github.com/repos/{REPO_OWNER}/{REPO_NAME}/releases/latest"
PLUGIN_DOCS_URL = f"https://{REPO_OWNER}.github.io/{REPO_NAME}/"

# Other constants

UPDATE_CHECK_TIMEOUT_SECONDS = 6
SETTING_UPDATE_CHECK_ENABLED = "update_check_enabled"
SETTING_LAST_LOADED_VERSION = "last_loaded_version"
SETTING_PET_FAB_SIZE_INDEX = "pet_fab_size_index"
PET_FAB_SIZE_OPTIONS_DP = (64, 80, 96, 112, 128)
VALID_ISO_LANGUAGES = frozenset(str(code) for code in Locale.getISOLanguages())


def get_plugin_cache_dir(*parts: str) -> str:
    cache_root = ApplicationLoader.applicationContext.getCacheDir().getAbsolutePath()
    return os.path.join(cache_root, __id__, *parts)


def _sha256_hex(data: bytes) -> str:
    return hashlib.sha256(data).hexdigest().lower()


I18N_SETTINGS: dict[str, dict[str, str]] = {
    "settings.backups.description": {
        "en": "Backups are saved to Downloads/tg-streaks. Restore replaces the current database.",
        "ru": "Бэкапы лежат в Downloads/tg-streaks. Восстановление заменяет текущую базу.",
    },
    "settings.backups.export.title": {"en": "Create backup", "ru": "Создать бэкап"},
    "settings.backups.reset_database.title": {
        "en": "Reset plugin database",
        "ru": "Сбросить базу плагина",
    },
    "settings.backups.restore.title": {
        "en": "Restore backup",
        "ru": "Восстановить бэкап",
    },
    "settings.backups.title": {"en": "Backups", "ru": "Бэкапы"},
    "settings.help.docs.description": {
        "en": "Open the plugin user guide in your browser.",
        "ru": "Открыть руководство пользователя в браузере.",
    },
    "settings.help.docs.title": {"en": "Documentation", "ru": "Документация"},
    "settings.help.title": {"en": "Help", "ru": "Справка"},
    "settings.pet_button.size.description": {
        "en": "Floating chat button size.",
        "ru": "Размер плавающей кнопки в чате.",
    },
    "settings.pet_button.size.title": {"en": "Button size", "ru": "Размер кнопки"},
    "settings.pet_button.title": {"en": "Streak pet", "ru": "Серийчик"},
    "settings.streak_tools.rebuild_all_chats.description": {
        "en": "Only user DMs are checked. Bots and groups are skipped.",
        "ru": "Только лички с пользователями. Боты и группы пропускаются.",
    },
    "settings.streak_tools.rebuild_all_chats.title": {
        "en": "Private chats rebuild",
        "ru": "Пересчёт стриков в личках",
    },
    "settings.streak_tools.title": {"en": "Streak", "ru": "Стрик"},
    "settings.updates.auto_check.description": {
        "en": "Checks GitHub releases on startup.",
        "ru": "Проверяет релизы GitHub при запуске.",
    },
    "settings.updates.auto_check.title": {
        "en": "Update checks",
        "ru": "Проверка обновлений",
    },
    "settings.updates.title": {"en": "Updates", "ru": "Обновления"},
}

I18N_STATUS: dict[str, dict[str, str]] = {
    "status.error.backup.apply_failed": {
        "en": "Backup restore failed: {reason}",
        "ru": "Не удалось восстановить бэкап: {reason}",
    },
    "status.error.backup.not_found": {
        "en": "No backups found",
        "ru": "Бэкапы не найдены",
    },
    "status.error.chat.detect_current_failed": {
        "en": "Current chat not found",
        "ru": "Текущий чат не найден",
    },
    "status.error.database.delete_failed": {
        "en": "Database reset failed: {reason}",
        "ru": "Не удалось сбросить базу: {reason}",
    },
    "status.error.update.open_link_failed": {
        "en": "Couldn't open update link",
        "ru": "Не удалось открыть ссылку обновления",
    },
    "status.error.update.restart_failed": {
        "en": "Couldn't restart client",
        "ru": "Не удалось перезапустить клиент",
    },
    "status.success.backup.imported": {
        "en": "Backup restored: {name}",
        "ru": "Бэкап восстановлен: {name}",
    },
    "status.success.database.reset_started": {
        "en": "Database reset started",
        "ru": "Сброс базы запущен",
    },
}

I18N_MENU: dict[str, dict[str, str]] = {
    "menu.chat.control_panel.description": {
        "en": "Streak, pet, sync and time zone settings",
        "ru": "Настройки стрика, серийчика, синхронизации и часового пояса",
    },
    "menu.chat.control_panel.title": {"en": "Control panel", "ru": "Панель управления"},
    "menu.chat.restore_streak_exact.description": {
        "en": "Available anytime, but limited by 2 usages per chat",
        "ru": "Доступно в любое время, но с ограничением по 2 раза на чат",
    },
    "menu.chat.restore_streak_exact.title": {
        "en": "Streak restore menu",
        "ru": "Меню восстановление стрика",
    },
    "menu.debug.crash_plugin.description": {"en": "Test crash", "ru": "Тестовый краш"},
    "menu.debug.crash_plugin.title": {
        "en": "[DEBUG] Plugin crash",
        "ru": "[DEBUG] Краш плагина",
    },
    "menu.debug.create_streak.description": {
        "en": "Sets a 3-day streak in this chat",
        "ru": "Ставит стрик на 3 дня в этом чате",
    },
    "menu.debug.create_streak.title": {
        "en": "[DEBUG] 3-day streak",
        "ru": "[DEBUG] Стрик на 3 дня",
    },
    "menu.debug.delete_pet.description": {
        "en": "Deletes the streak pet from this chat",
        "ru": "Удаляет серийчика из этого чата",
    },
    "menu.debug.delete_pet.title": {
        "en": "[DEBUG] Streak pet delete",
        "ru": "[DEBUG] Удаление серийчика",
    },
    "menu.debug.delete_streak.description": {
        "en": "Deletes the streak from this chat",
        "ru": "Удаляет стрик из этого чата",
    },
    "menu.debug.delete_streak.title": {
        "en": "[DEBUG] Streak delete",
        "ru": "[DEBUG] Удаление стрика",
    },
    "menu.debug.freeze_streak.description": {
        "en": "Sets a frozen streak in this chat",
        "ru": "Ставит замороженный стрик в этом чате",
    },
    "menu.debug.freeze_streak.title": {
        "en": "[DEBUG] Streak freeze",
        "ru": "[DEBUG] Заморозка стрика",
    },
    "menu.debug.kill_streak.description": {
        "en": "Breaks the streak in this chat",
        "ru": "Прерывает стрик в этом чате",
    },
    "menu.debug.kill_streak.title": {
        "en": "[DEBUG] Streak break",
        "ru": "[DEBUG] Обрыв стрика",
    },
    "menu.debug.upgrade_streak.description": {
        "en": "Moves the streak to the next level",
        "ru": "Поднимает стрик до следующего уровня",
    },
    "menu.debug.upgrade_streak.title": {
        "en": "[DEBUG] Streak upgrade",
        "ru": "[DEBUG] Повышение стрика",
    },
}

I18N_DIALOGS: dict[str, dict[str, str]] = {
    "dialog.backup_restore.browse": {
        "en": "Browse from device...",
        "ru": "Выбрать с устройства...",
    },
    "dialog.backup_restore.title": {"en": "Choose backup", "ru": "Выберите бэкап"},
    "dialog.download_failed.cancel": {"en": "Cancel", "ru": "Отмена"},
    "dialog.download_failed.message": {
        "en": "Failed to download {filename}. Check your internet connection or try enabling a VPN, then try again.",
        "ru": "Не удалось скачать {filename}. Проверьте подключение к интернету или попробуйте включить ВПН, затем повторите попытку.",
    },
    "dialog.download_failed.retry": {"en": "Retry", "ru": "Повторить"},
    "dialog.download_failed.title": {
        "en": "Couldn't download plugin data",
        "ru": "Не удалось скачать данные плагина",
    },
    "dialog.load_crash.message": {
        "en": "tg-streaks failed to load at stage '{stage}'.\n\nA crash report has been copied to the clipboard — you can send it to the plugin's chat.",
        "ru": "Не удалось загрузить tg-streaks на этапе «{stage}».\n\nОтчёт скопирован в буфер обмена — вы можете отправить его в чат плагина.",
    },
    "dialog.load_crash.ok": {"en": "OK", "ru": "Ок"},
    "dialog.load_crash.open_chat": {"en": "Open chat", "ru": "Открыть чат"},
    "dialog.load_crash.title": {
        "en": "Plugin load failed",
        "ru": "Не удалось загрузить плагин",
    },
    "dialog.sha256_mismatch.message": {
        "en": "Checksum of downloaded {filename} does not match.\nThe plugin has been disabled for security.\n\nPlugin version: {version}\nTarget file: {filename}\nHash: {hash} ({expected_hash} expected)",
        "ru": "Контрольная сумма скачанного {filename} не совпадает.\nПлагин отключён в целях безопасности.\n\nВерсия плагина: {version}\nФайл: {filename}\nХеш: {hash} (ожидался {expected_hash})",
    },
    "dialog.sha256_mismatch.ok": {"en": "OK", "ru": "Ок"},
    "dialog.sha256_mismatch.title": {
        "en": "Streaks plugin disabled",
        "ru": "Плагин Streaks отключён",
    },
    "dialog.update_restart.later": {"en": "Later", "ru": "Позже"},
    "dialog.update_restart.message": {
        "en": "Streaks was updated from {previous} to {current}. Restart the client to finish the update.",
        "ru": "Streaks обновлён с {previous} до {current}. Перезапустите клиент, чтобы завершить обновление.",
    },
    "dialog.update_restart.restart": {"en": "Restart", "ru": "Перезапустить"},
    "dialog.update_restart.title": {
        "en": "Restart client?",
        "ru": "Перезапустить клиент?",
    },
}

I18N_UPDATE: dict[str, dict[str, str]] = {
    "update.available.action": {"en": "Update", "ru": "Обновить"},
    "update.available.message": {
        "en": "Update available: {current} -> {latest}",
        "ru": "Есть обновление: {current} -> {latest}",
    },
}

I18N_DOWNLOAD: dict[str, dict[str, str]] = {
    "download.assets.completed": {"en": "Assets downloaded", "ru": "Ресурсы скачаны"},
    "download.assets.started": {
        "en": "Downloading assets...",
        "ru": "Скачиваю ресурсы...",
    },
    "download.engine.completed": {"en": "Engine downloaded", "ru": "Движок скачан"},
    "download.engine.started": {
        "en": "Downloading engine...",
        "ru": "Скачиваю движок...",
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

I18N_STRINGS: dict[str, dict[str, str]] = {
    **I18N_SETTINGS,
    **I18N_STATUS,
    **I18N_MENU,
    **I18N_DIALOGS,
    **I18N_UPDATE,
    **I18N_DOWNLOAD,
}

# fmt: on


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


class DownloadFailedError(Exception):
    """Raised by TgStreaksPlugin._download_with_progress on any download
    failure. The user-facing dialog is shown right there, at the point the
    download failed; callers only need to handle fallback-to-cache logic, if
    any."""


class JvmPluginBridge:
    klass: Optional[Class]

    def __init__(self, plugin: "TgStreaksPlugin"):
        self.plugin = plugin
        self.klass = None
        self.cache_dir = get_plugin_cache_dir("plugins_dex_cache")
        os.makedirs(self.cache_dir, exist_ok=True)
        self.dex_path = os.path.join(self.cache_dir, f"{__id__}.dex")

    def load(self):
        if DEBUG_MODE:
            self.plugin.log(
                "Debug mode enabled. Downloading DEX without SHA256 checks..."
            )
            try:
                dex_data = self._download(show_bulletins=True)
            except DownloadFailedError:
                self.plugin.log(
                    "DEX download failed in debug mode. Falling back to cached DEX if available..."
                )
                self._load_cached_file()
                return

            self._write_dex_file(dex_data)
            self._load(dex_data)
            return

        expected_sha256 = str(DEX_SHA256).strip().lower()
        cached_sha256 = self._compute_file_sha256(self.dex_path)

        if cached_sha256 == expected_sha256:
            self._load_cached_file()
            return

        if cached_sha256 is not None:
            self.plugin.log(
                f"Cached DEX SHA256 mismatch (cached={cached_sha256}, expected={expected_sha256}). Downloading correct version..."
            )
        else:
            self.plugin.log("Cached DEX not found. Downloading...")

        try:
            dex_data = self._download(show_bulletins=True)
        except DownloadFailedError:
            if cached_sha256 is not None:
                self.plugin.log(
                    "DEX download failed. Falling back to stale cached DEX (version mismatch possible)..."
                )
                self._load_cached_file()
            return

        downloaded_sha256 = _sha256_hex(dex_data)
        if downloaded_sha256 != expected_sha256:
            self.plugin._show_sha256_mismatch_dialog(
                "classes.dex", downloaded_sha256, expected_sha256
            )
            return

        self._write_dex_file(dex_data)
        self._load(dex_data)

    def _load_cached_file(self):
        try:
            with open(self.dex_path, "rb") as f:
                self._load(f.read())
        except Exception as e:
            self.plugin.log_exception("Failed to load cached DEX", e)

    def _compute_file_sha256(self, path: str) -> Optional[str]:
        if not os.path.exists(path):
            return None

        try:
            with open(path, "rb") as f:
                return _sha256_hex(f.read())
        except Exception as e:
            self.plugin.log_exception("Failed to read cached DEX for SHA256", e)
            return None

    def _write_dex_file(self, dex_data: bytes):
        with open(self.dex_path, "wb") as f:
            f.write(dex_data)

    def _download(self, show_bulletins: bool) -> bytes:
        return self.plugin._download_with_progress(
            url=DEX_URL,
            label="classes.dex",
            started_key="download.engine.started",
            completed_key="download.engine.completed",
            show_bulletins=show_bulletins,
        )

    def _load(self, dex_data: bytes):
        class_path = "ru.n08i40k.streaks.Plugin"

        try:
            loader = InMemoryDexClassLoader(
                ByteBuffer.wrap(dex_data),
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

        if cached_sha256 == expected_sha256:
            if not os.path.isdir(self.resources_root):
                self.plugin.log(
                    "Resources ZIP is cached, but unpacked files are missing. Extracting..."
                )
                lock_fd = self._acquire_lock(blocking=True)
                try:
                    if not os.path.isdir(self.resources_root):
                        self._extract_zip()
                finally:
                    self._release_lock(lock_fd)
            return self.resources_root if os.path.isdir(self.resources_root) else None

        if cached_sha256 is not None:
            self.plugin.log(
                f"Cached resources ZIP SHA256 mismatch (cached={cached_sha256}, expected={expected_sha256}). Downloading correct version..."
            )
        else:
            self.plugin.log("Cached resources ZIP not found. Downloading...")

        lock_fd = self._acquire_lock(blocking=False)
        if lock_fd is None:
            self.plugin.log("Resources ZIP update is already in progress. Waiting...")
            lock_fd = self._acquire_lock(blocking=True)
            self._release_lock(lock_fd)
            # Another process finished: re-check what it produced
            return self.resources_root if os.path.isdir(self.resources_root) else None

        try:
            try:
                zip_data = self._download(show_bulletins=True)
            except DownloadFailedError:
                if os.path.isdir(self.resources_root):
                    self.plugin.log(
                        "Resources ZIP download failed. Falling back to stale extracted resources (version mismatch possible)..."
                    )
                    return self.resources_root
                return None

            downloaded_sha256 = _sha256_hex(zip_data)
            if downloaded_sha256 != expected_sha256:
                self.plugin._show_sha256_mismatch_dialog(
                    "resources.zip", downloaded_sha256, expected_sha256
                )
                return None

            self._write_zip_file(zip_data)
            self._extract_zip()
        finally:
            self._release_lock(lock_fd)

        return self.resources_root if os.path.isdir(self.resources_root) else None

    def _load_debug(self) -> Optional[str]:
        lock_fd = self._acquire_lock(blocking=False)
        if lock_fd is not None:
            try:
                self.plugin.log(
                    "Debug mode enabled. Downloading resources ZIP without SHA256 checks..."
                )
                try:
                    zip_data = self._download(show_bulletins=True)
                    self._write_zip_file(zip_data)
                    self._extract_zip()
                except DownloadFailedError:
                    pass
            finally:
                self._release_lock(lock_fd)
        else:
            self.plugin.log("Resources ZIP download is already in progress. Waiting...")
            wait_lock_fd = self._acquire_lock(blocking=True)
            self._release_lock(wait_lock_fd)

        return self.resources_root if os.path.isdir(self.resources_root) else None

    def _compute_file_sha256(self, path: str) -> Optional[str]:
        if not os.path.exists(path):
            return None

        try:
            with open(path, "rb") as f:
                return _sha256_hex(f.read())
        except Exception as e:
            self.plugin.log_exception(
                "Failed to read cached resources ZIP for SHA256", e
            )
            return None

    def _write_zip_file(self, zip_data: bytes):
        with open(self.zip_path, "wb") as f:
            f.write(zip_data)

    def _acquire_lock(self, blocking: bool) -> Optional[int]:
        lock_fd = os.open(self.lock_path, os.O_CREAT | os.O_RDWR, 0o666)
        lock_flags = fcntl.LOCK_EX | (0 if blocking else fcntl.LOCK_NB)

        try:
            fcntl.flock(lock_fd, lock_flags)
            return lock_fd
        except BlockingIOError:
            os.close(lock_fd)
            return None
        except Exception:
            os.close(lock_fd)
            raise

    def _release_lock(self, lock_fd: int):
        try:
            fcntl.flock(lock_fd, fcntl.LOCK_UN)
        finally:
            os.close(lock_fd)

    def _download(self, show_bulletins: bool) -> bytes:
        return self.plugin._download_with_progress(
            url=RESOURCES_URL,
            label="resources.zip",
            started_key="download.assets.started",
            completed_key="download.assets.completed",
            show_bulletins=show_bulletins,
        )

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
    CONTROL_MENU = "controlMenu"
    RESTORE_EXACT = "restoreExact"

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
                "key": cls.CONTROL_MENU,
                "text_key": "menu.chat.control_panel.title",
                "subtext_key": "menu.chat.control_panel.description",
                "icon": "msg_settings",
                "priority": 1001,
            },
            {
                "key": cls.RESTORE_EXACT,
                "text_key": "menu.chat.restore_streak_exact.title",
                "subtext_key": "menu.chat.restore_streak_exact.description",
                "icon": "msg_reactions",
                "priority": 999,
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
                return int(value) or None
            except Exception:
                return None

        payload_getter = getattr(payload, "get", None)
        if payload_getter is not None:
            for key in self.MENU_PAYLOAD_DIALOG_KEYS:
                if (dialog_id := parse_dialog_id(payload_getter(key))) is not None:
                    return dialog_id

            for key in self.MENU_PAYLOAD_FRAGMENT_KEYS:
                value = payload_getter(key)
                if value is None:
                    continue
                try:
                    if (dialog_id := parse_dialog_id(value.getDialogId())) is not None:
                        return dialog_id
                except Exception:
                    pass

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
            ).invoke(None, key)  # ty:ignore[invalid-argument-type]
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
        return self.plugin._is_update_check_enabled()

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

                if not latest_version:
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
        if not normalized:
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
        if not normalized:
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
    _full_load_lock = threading.Lock()

    settings_actions: SettingsActions

    def log(self, message: Any):
        text = str(message)
        super().log(text)

        if getattr(self, "_load_logging_active", False):
            self._load_log_buffer.append(text)

        try:
            Log.i(cast("String", LOGCAT_TAG), cast("String", text))
        except Exception:
            pass

    def log_exception(self, message: str, exception: BaseException):
        def _log_e(text: str):
            if DEBUG_MODE:
                try:
                    Log.e(cast("String", LOGCAT_TAG), cast("String", text))
                except Exception:
                    pass

        text = f"{message}: {exception}"
        self.log(text)
        _log_e(text)

        for chunk in traceback.format_exception(
            type(exception),
            exception,
            exception.__traceback__,
        ):
            for line in chunk.rstrip().splitlines():
                if line:
                    self.log(line)
                    _log_e(line)

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
            Header(text=self._t("settings.help.title")),
            Text(
                text=self._t("settings.help.docs.title"),
                icon="msg_info",
                on_click=lambda _: self._open_browser_url(PLUGIN_DOCS_URL),
            ),
            Divider(text=self._t("settings.help.docs.description")),
        ]

    def _show_info(self, message: str):
        run_on_ui_thread(lambda: BulletinHelper.show_info(message))

    def _show_error(self, message: str):
        run_on_ui_thread(lambda: BulletinHelper.show_error(message))

    def _show_success(self, message: str):
        run_on_ui_thread(lambda: BulletinHelper.show_success(message))

    def _get_last_loaded_version(self) -> str:
        previous_version = ""

        try:
            previous_version = str(self.get_setting(SETTING_LAST_LOADED_VERSION, ""))
        except Exception as e:
            self.log_exception("Failed to read last loaded plugin version", e)

        return previous_version

    def _persist_current_loaded_version(self):
        try:
            self.set_setting(SETTING_LAST_LOADED_VERSION, __version__)
        except Exception as e:
            self.log_exception("Failed to persist last loaded plugin version", e)

    def _should_pause_full_load_for_update(self) -> bool:
        previous_version = self._get_last_loaded_version()

        if len(previous_version) == 0 or previous_version == __version__:
            self._persist_current_loaded_version()
            return False

        self._show_update_restart_dialog(previous_version)
        return True

    def _show_sha256_mismatch_dialog(
        self, filename: str, hash: str, expected_hash: str
    ):
        self.log(f"SHA256 mismatch for {filename}: plugin disabled")

        def show():
            try:
                fragment = get_last_fragment()
            except Exception:
                fragment = None

            title = self._t("dialog.sha256_mismatch.title")
            message = self._t(
                "dialog.sha256_mismatch.message",
                filename=filename,
                version=__version__,
                hash=f"{hash[:4]}...{hash[-6:]}",
                expected_hash=f"{expected_hash[:4]}...{expected_hash[-6:]}",
            )

            if fragment is None:
                self._show_error(message)
                return

            try:
                fragment.showDialog(
                    AlertDialog.Builder(fragment.getContext())
                    .setTitle(String(title))
                    .setMessage(String(message))
                    .setPositiveButton(
                        String(self._t("dialog.sha256_mismatch.ok")),
                        None,
                    )
                    .create()
                )
            except Exception as e:
                self.log_exception("Failed to show SHA256 mismatch dialog", e)
                self._show_error(message)

        run_on_ui_thread(show)

    def _show_download_failed_dialog(self, filename: str):
        self.log(f"Download failed for {filename}: plugin load aborted")

        def show():
            try:
                fragment = get_last_fragment()
            except Exception:
                fragment = None

            message = self._t("dialog.download_failed.message", filename=filename)

            if fragment is None:
                self.log("Download failed dialog deferred: UI context is unavailable")
                self._schedule_download_failed_dialog_retry(filename)
                return

            self_outer = self

            class RetryClickListener(dynamic_proxy(AlertDialog.OnButtonClickListener)):
                def onClick(self, _dialog: AlertDialog, _which: int) -> None:  # ty: ignore[invalid-method-override]
                    self_outer._schedule_continue_plugin_load()

            try:
                fragment.showDialog(
                    AlertDialog.Builder(fragment.getContext())
                    .setTitle(String(self._t("dialog.download_failed.title")))
                    .setMessage(String(message))
                    .setPositiveButton(
                        String(self._t("dialog.download_failed.retry")),
                        RetryClickListener(),
                    )
                    .setNegativeButton(
                        String(self._t("dialog.download_failed.cancel")),
                        None,
                    )
                    .create()
                )
            except Exception as e:
                self.log_exception("Failed to show download failed dialog", e)
                self._show_error(message)

        run_on_ui_thread(show)

    def _schedule_download_failed_dialog_retry(self, filename: str):
        timer = threading.Timer(
            1.0,
            lambda: self._show_download_failed_dialog(filename),
        )
        timer.daemon = True
        timer.start()

    def _show_update_restart_dialog(self, previous_version: str):
        def show():
            try:
                fragment = get_last_fragment()
            except Exception:
                fragment = None

            if fragment is None:
                self.log("Update restart dialog deferred: UI context is unavailable")
                self._schedule_update_restart_dialog_retry(previous_version)
                return

            self_outer = self

            class RestartClickListener(
                dynamic_proxy(AlertDialog.OnButtonClickListener)
            ):
                def onClick(self, _dialog: AlertDialog, _which: int) -> None:  # ty: ignore[invalid-method-override]
                    self_outer._persist_current_loaded_version()
                    self_outer._restart_client()

            class LaterClickListener(dynamic_proxy(AlertDialog.OnButtonClickListener)):
                def onClick(self, _dialog: AlertDialog, _which: int) -> None:  # ty: ignore[invalid-method-override]
                    self_outer._persist_current_loaded_version()
                    self_outer._schedule_continue_plugin_load()

            try:
                fragment.showDialog(
                    AlertDialog.Builder(fragment.getContext())
                    .setTitle(String(self._t("dialog.update_restart.title")))
                    .setMessage(
                        String(
                            self._t(
                                "dialog.update_restart.message",
                                previous=previous_version,
                                current=__version__,
                            )
                        )
                    )
                    .setPositiveButton(
                        String(self._t("dialog.update_restart.restart")),
                        RestartClickListener(),
                    )
                    .setNegativeButton(
                        String(self._t("dialog.update_restart.later")),
                        LaterClickListener(),
                    )
                    .create()
                )
            except Exception as e:
                self.log_exception("Failed to show update restart dialog", e)
                self._show_info(
                    self._t(
                        "dialog.update_restart.message",
                        previous=previous_version,
                        current=__version__,
                    )
                )
                self._schedule_update_restart_dialog_retry(previous_version)

        run_on_ui_thread(show)

    def _schedule_update_restart_dialog_retry(self, previous_version: str):
        timer = threading.Timer(
            1.0,
            lambda: self._show_update_restart_dialog(previous_version),
        )
        timer.daemon = True
        timer.start()

    def _schedule_continue_plugin_load(self):
        threading.Thread(
            target=self._continue_plugin_load,
            name="tg-streaks-continue-plugin-load",
            daemon=True,
        ).start()

    def _reset_load_log_buffer(self):
        self._load_log_buffer: list[str] = []
        self._load_logging_active = True

    def _stop_load_logging(self):
        self._load_logging_active = False
        self._load_log_buffer = []

    def _handle_load_failure(self, stage: str, exception: BaseException):
        self.log_exception(f"Plugin load failed ({stage})", exception)

        logs = "\n".join(getattr(self, "_load_log_buffer", []))
        report = (
            f"#load_crash\n\n"
            f"Stage: `{stage}`\n"
            f"Plugin version: `{__version__}`\n\n"
            f"Error:\n```\n{exception}\n```\n\n"
            f"Log:\n```\n{logs}\n```"
        )

        try:
            copy_to_clipboard(report)
        except Exception as e:
            self.log_exception("Failed to copy load-crash report to clipboard", e)

        self._show_load_crash_dialog(stage)

    def _show_load_crash_dialog(self, stage: str):
        def show():
            try:
                fragment = get_last_fragment()
            except Exception:
                fragment = None

            message = self._t("dialog.load_crash.message", stage=stage)

            if fragment is None:
                self._show_error(message)
                return

            self_outer = self

            class OpenChatClickListener(
                dynamic_proxy(AlertDialog.OnButtonClickListener)
            ):
                def onClick(self, _dialog: AlertDialog, _which: int) -> None:  # ty: ignore[invalid-method-override]
                    self_outer._open_telegram_url(PLUGIN_CHAT_TG_URL)

            try:
                fragment.showDialog(
                    AlertDialog.Builder(fragment.getContext())
                    .setTitle(String(self._t("dialog.load_crash.title")))
                    .setMessage(String(message))
                    .setPositiveButton(
                        String(self._t("dialog.load_crash.open_chat")),
                        OpenChatClickListener(),
                    )
                    .setNegativeButton(
                        String(self._t("dialog.load_crash.ok")),
                        None,
                    )
                    .create()
                )
            except Exception as e:
                self.log_exception("Failed to show load crash dialog", e)
                self._show_error(message)

        run_on_ui_thread(show)

    def _restart_client(self):
        def restart():
            try:
                Process.killProcess(Process.myPid())
            except Exception as e:
                self.log_exception("Failed to restart client", e)
                self._show_error(self._t("status.error.update.restart_failed"))
                self._schedule_continue_plugin_load()

        run_on_ui_thread(restart)

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
        label: str,
        started_key: str,
        completed_key: str,
        show_bulletins: bool,
    ) -> bytes:
        if show_bulletins:
            self._show_info(self._t(started_key))

        try:
            response = requests.get(url, timeout=10, stream=True)
            if response.status_code != 200:
                raise DownloadFailedError(f"HTTP {response.status_code} for {url}")

            total_bytes = int(response.headers.get("content-length", "0") or "0")
            downloaded = 0
            chunks: list[bytes] = []
            started_at = time.monotonic()
            last_progress_at = 0.0

            for chunk in response.iter_content(chunk_size=64 * 1024):
                if not chunk:
                    continue

                chunks.append(chunk)
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
        except DownloadFailedError as e:
            self.log(f"Failed to download {url}: {e}")
            self._show_download_failed_dialog(label)
            raise
        except Exception as e:
            self.log_exception(f"Failed to download {url}", e)
            self._show_download_failed_dialog(label)
            raise DownloadFailedError(str(e)) from e

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
                None,  # ty:ignore[invalid-argument-type]
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
        def safe_call(fn):
            try:
                return fn()
            except Exception:
                return None

        lang_code = None

        lc = safe_call(LocaleController.getInstance)
        if lc is not None:
            info = safe_call(lc.getCurrentLocaleInfo)
            if info is not None:
                has_base_lang = safe_call(info.hasBaseLang)
                if has_base_lang:
                    raw = safe_call(lambda: info.baseLangCode)
                else:
                    raw = safe_call(info.getLangCode) or safe_call(
                        lambda: info.shortName
                    )
                if raw:
                    lang_code = str(raw)

            if not lang_code:
                locale = safe_call(lc.getCurrentLocale)
                if locale is not None:
                    raw = safe_call(locale.getLanguage)
                    if raw:
                        lang_code = str(raw)

        if not lang_code:
            return "en"

        normalized = lang_code.strip().lower().replace("-", "_")
        code = normalized.split("_", 1)[0]
        return code if code in VALID_ISO_LANGUAGES else "en"

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

    def _open_browser_url(self, url: str):
        def open_url():
            try:
                context = self._resolve_popup_context()
                if context is None:
                    raise RuntimeError("no context")

                intent = Intent()
                intent.setAction(String(Intent.ACTION_VIEW))
                intent.setData(Uri.parse(String(url)))
                intent.addFlags(int(Intent.FLAG_ACTIVITY_NEW_TASK))
                context.startActivity(intent)
            except Exception as e:
                self.log_exception(f"Failed to open Telegram url {url}", e)

        run_on_ui_thread(open_url)

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
                String.getClass(),
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
                String.getClass(),
                String.getClass(),
                String.getClass(),
                String.getClass(),
                String.getClass(),
                String.getClass(),
                String.getClass(),
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
        try:
            self.jvm_plugin.klass.getDeclaredMethod(  # ty:ignore[possibly-missing-attribute]
                String("finalizeInject")
            ).invoke(
                None  # ty:ignore[invalid-argument-type]
            )
            self.log("JVM plugin finalizeInject completed")
        except Exception as e:
            self._handle_load_failure("finalizeInject", e)

    def _prepare_jvm_plugin(self) -> bool:
        """Verifies/downloads the DEX and resources. Any download failure is
        reported (with a retry dialog) by JvmPluginBridge.load() /
        ZipResourcesBridge.load() themselves, at the point it happens."""
        self.jvm_plugin = JvmPluginBridge(self)
        self.jvm_plugin.load()

        if self.jvm_plugin.klass is None:
            return False

        self.resources_root = self.resources_bridge.load()
        return self.resources_root is not None

    def _inject_jvm_plugin(self):
        try:
            build_date = self.jvm_plugin.klass.getDeclaredMethod(  # ty:ignore[possibly-missing-attribute]
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

            self.jvm_plugin.klass.getDeclaredMethod(  # ty:ignore[possibly-missing-attribute]
                String("inject"),
                String.getClass(),
                ValueCallback.getClass(),  # ty:ignore[unresolved-attribute]
                String.getClass(),
            ).invoke(
                None,  # ty:ignore[invalid-argument-type]
                String(__version__),
                Logger(),  # ty:ignore[invalid-argument-type]
                String(self.resources_root),
            )

            self.log("JVM plugin injected successfully")
            self._apply_pet_fab_size_dp(self._get_pet_fab_size_dp())
        except Exception as e:
            self._handle_load_failure("inject", e)

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

        def show():
            try:
                fragment = get_last_fragment()
            except Exception:
                fragment = None

            if fragment is None:
                self._show_error(
                    self._t("status.error.backup.apply_failed", reason="No UI context")
                )
                return

            names = [os.path.basename(path) for path in backup_files]
            names.append(self._t("dialog.backup_restore.browse"))

            try:
                self_outer = self

                class BackupClickListener(
                    dynamic_proxy(DialogInterface.OnClickListener)
                ):
                    def onClick(self, _dialog, which):  # ty:ignore[invalid-method-override]
                        idx = int(which)
                        if idx < len(backup_files):
                            self_outer._schedule_restore_backup_reinitialize(
                                backup_files[idx]
                            )
                        else:
                            self_outer._open_backup_file_picker(fragment)

                fragment.showDialog(
                    AlertDialog.Builder(fragment.getContext())
                    .setTitle(cast("String", self._t("dialog.backup_restore.title")))
                    .setItems(
                        jarray(String)([String(name) for name in names]),
                        BackupClickListener(),
                    )
                    .create()
                )
            except Exception as e:
                self.log_exception("Failed to show restore backup dialog", e)
                self._show_error(
                    self._t("status.error.backup.apply_failed", reason=str(e))
                )

        run_on_ui_thread(show)

    _BACKUP_FILE_PICKER_REQUEST_CODE = 0x5A7B
    _file_picker_hook_unhooks: "list[Any] | None" = None

    def _cleanup_file_picker_hooks(self):
        unhooks, self._file_picker_hook_unhooks = self._file_picker_hook_unhooks, None
        for u in unhooks or []:
            try:
                u.unhook()
            except Exception:
                pass

    def _open_backup_file_picker(self, fragment):
        if self._file_picker_hook_unhooks is not None:
            return

        self_outer = self

        class ActivityResultHook(MethodHook):
            def after_hooked_method(self, param):
                request_code = int(param.args[0])
                if request_code != self_outer._BACKUP_FILE_PICKER_REQUEST_CODE:
                    return

                self_outer._cleanup_file_picker_hooks()

                result_code = int(param.args[1])
                if result_code != -1:  # Activity.RESULT_OK
                    return

                data_intent = param.args[2]
                if data_intent is None:
                    return

                uri = data_intent.getData()
                if uri is None:
                    return

                self_outer._restore_from_uri(uri)

        try:
            from hook_utils import find_class

            base_fragment_class = find_class("org.telegram.ui.ActionBar.BaseFragment")
            if base_fragment_class is None:
                raise RuntimeError("BaseFragment class not found")

            self._file_picker_hook_unhooks = self.hook_all_methods(
                base_fragment_class,
                "onActivityResultFragment",
                ActivityResultHook(),
            )

            intent = Intent()
            intent.setAction(String(Intent.ACTION_GET_CONTENT))
            intent.setType(String("*/*"))
            intent.addCategory(String(Intent.CATEGORY_OPENABLE))

            fragment.startActivityForResult(
                intent, self._BACKUP_FILE_PICKER_REQUEST_CODE
            )
        except Exception as e:
            self._cleanup_file_picker_hooks()
            self.log_exception("Failed to open backup file picker", e)
            self._show_error(self._t("status.error.backup.apply_failed", reason=str(e)))

    def _restore_from_uri(self, uri):
        def worker():
            try:
                context = ApplicationLoader.applicationContext
                cr = context.getContentResolver()

                display_name = "backup"
                try:
                    cursor = cr.query(uri, None, None, None, None)  # ty:ignore[invalid-argument-type]
                    if cursor is not None and cursor.moveToFirst():
                        col = cursor.getColumnIndex(String("_display_name"))
                        if col >= 0:
                            display_name = str(cursor.getString(col))
                        cursor.close()
                except Exception:
                    pass

                temp_path = os.path.join(
                    str(context.getCacheDir().getAbsolutePath()),
                    "tg-streaks-restore-import.sqlite3",
                )

                input_stream = cr.openInputStream(uri)
                try:
                    buf = jarray(jbyte)(65536)
                    with open(temp_path, "wb") as f:
                        while True:
                            n = input_stream.read(buf)
                            if n < 0:
                                break
                            f.write(bytes(buf[:n]))
                finally:
                    input_stream.close()

                self._schedule_restore_backup_reinitialize(temp_path, display_name)
            except Exception as e:
                self.log_exception("Failed to read backup from URI", e)
                self._show_error(
                    self._t("status.error.backup.apply_failed", reason=str(e))
                )

        threading.Thread(
            target=worker,
            name="tg-streaks-backup-restore-from-uri",
            daemon=True,
        ).start()

    def _schedule_restore_backup_reinitialize(
        self, backup_path: str, display_name: str | None = None
    ):
        name = display_name or os.path.basename(backup_path)
        reason = f"Backup restore requested from settings: {name}"

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

                    shutil.copy(backup_path, target_path)
                    self.log(f"Database restored from backup: {backup_path}")
                except BaseException as e:
                    self.log_exception("Failed to apply backup file", e)
                    self._show_error(
                        self._t("status.error.backup.apply_failed", reason=str(e))
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
                        name=name,
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
                        self._t("status.error.database.delete_failed", reason=str(e))
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

    def _continue_plugin_load(self):
        with self._full_load_lock:
            if getattr(self, "_full_load_started", False):
                return
            self._full_load_started = True

        try:
            if not self._prepare_jvm_plugin():
                with self._full_load_lock:
                    self._full_load_started = False
                return

            self._inject_jvm_plugin()

            self._register_streak_levels()
            self._register_streak_pet_levels()

            self.settings_actions = SettingsActions(self)
            self.chat_context_menu = ChatContextMenu(self)
            self.chat_context_menu.register()

            self._finalize_jvm_plugin_inject()

            self.update_checker = PluginUpdateChecker(self)
            self.update_checker.start()
        except BaseException as e:
            self._handle_load_failure("plugin load", e)
            return

        self._stop_load_logging()

    def on_plugin_load(self):
        self._reset_load_log_buffer()

        try:
            self.resources_bridge = ZipResourcesBridge(self)
            self._full_load_started = False

            if self._should_pause_full_load_for_update():
                return

            self._continue_plugin_load()
        except BaseException as e:
            self._handle_load_failure("plugin load", e)

    def on_plugin_unload(self):
        try:
            self.update_checker.stop()
        except Exception:
            pass

        try:
            self.chat_context_menu.unregister()
        except Exception as e:
            self.log_exception("Failed to unregister chat context menu", e)

        jvm_plugin = getattr(self, "jvm_plugin", None)
        if jvm_plugin is None or jvm_plugin.klass is None:
            return

        try:
            jvm_plugin.klass.getDeclaredMethod(String("eject")).invoke(None)
            self.log("JVM plugin ejected successfully")
        except Exception as e:
            self.log_exception("Failed to eject JVM plugin", e)

        jvm_plugin.klass = None
