"""
by @mihailkotovski
Перед копированием/изменением кода уведомите @mihailkotovski
"""

import json
import os
import shutil
import threading
import time
import weakref
from typing import Any, Dict, Optional

from dalvik.system import InMemoryDexClassLoader
from java import jclass
from java.chaquopy import dynamic_proxy
from java.io import File
from java.lang import Class as JClass
from java.nio import ByteBuffer
from org.telegram.messenger import ApplicationLoader, AndroidUtilities, LocaleController, NotificationCenter, R as TelegramR
from org.telegram.ui.ActionBar import BottomSheet, Theme
from org.telegram.ui.Cells import HeaderCell, TextCheckCell, TextSettingsCell
from org.telegram.ui.Components import EditTextBoldCursor, LineProgressView, SlideChooseView
from org.telegram.ui.Stories.recorder import ButtonWithCounterView

from android.app import Notification, NotificationChannel, NotificationManager, PendingIntent
from android.content import ClipData, Intent
from android.content.res import ColorStateList
from android.graphics import Color, Typeface
from android.graphics.drawable import ColorDrawable, GradientDrawable, RippleDrawable
from android.os import Build, Environment, Looper
from android.text import InputType
from android.util import TypedValue
from android.view import Gravity, View, ViewGroup
from android.widget import FrameLayout, ImageView, LinearLayout, ScrollView, TextView
from androidx.core.content import FileProvider

from android_utils import OnClickListener, OnLongClickListener, R as Runnable, log, run_on_ui_thread
from base_plugin import BasePlugin, MenuItemData, MenuItemType, MethodHook
from client_utils import get_last_fragment
from ui.alert import AlertDialogBuilder
from ui.bulletin import BulletinHelper

__id__ = "chatexport"
__name__ = "Chat Export"
__description__ = "Export chats to HTML/JSON/TXT with Telegram Desktop style and media."
__author__ = "@mihailkotovski & @mishabotov"
__version__ = "2.0 [public beta]"
__icon__ = "nibbler_vk_Nyasticks_by_MoiStikiBot/35"
__min_version__ = "11.12.1"

DEFAULT_CHUNK_SIZE = 2500
POLL_INTERVAL_SECONDS = 0.35
RESULT_POLL_INTERVAL = 2
EXPORT_NOTIFICATION_CHANNEL_ID = "chatexport.export.progress"
EXPORT_NOTIFICATION_ID = 0x43484558
EXPORT_NOTIFICATION_REQUEST_CODE = 0x43484559


def _android_color_int(color: int) -> int:
    color = int(color) & 0xFFFFFFFF
    if color >= 0x80000000:
        color -= 0x100000000
    return color

DEX_CLASS_NAME = "com.chatexport.ChatExportCore"
DEX_CACHE_DIR = "chatexport_cache"
DEX_FILE_NAME = "chatexport.dex"
DEX_START_BRIDGE = "startBridge"
DEX_SETTINGS_BRIDGE = "updateSettingsBridge"
DEX_EXPORT_BRIDGE = "startExportBridge"
DEX_STATE_BRIDGE = "getExportStateBridge"
DEX_RESULT_BRIDGE = "consumeResultBridge"
DEX_CANCEL_BRIDGE = "cancelExportBridge"
DEX_UNLOAD_BRIDGE = "onUnloadBridge"

CONFIG_DEFAULTS = {
    "export_format": 0,
    "media_handling": 0,
    "max_messages": "",
    "max_size_mb": "100",
    "remote_download_batch_size": 48,
    "allow_large_chat_remote_downloads": False,
    "use_api_mode": True,
    "include_timestamps": True,
    "include_avatars": True,
    "group_messages": True,
    "use_chunked_mode": False,
    "chunk_size": DEFAULT_CHUNK_SIZE,
    "enable_overlay_stroke": True,
    "enable_stroke_animation": True,
    "overlay_opacity": "100",
    "overlay_position": 0,
}

TRANSLATIONS = {
    "ru": {
        "remote_download_batch_size": "Сколько медиа догружать за раз",
        "remote_download_batch_size_desc": "Размер одной порции догрузки. Большее значение ускоряет экспорт, но сильнее нагружает Telegram и память.",
        "allow_large_chat_remote_downloads": "Safe-mode",
        "allow_large_chat_remote_downloads_desc": "Safe-mode: осторожный режим для больших чатов. Если включить, экспорт попытается скачать файлы, которых нет на устройстве, но будет работать медленнее и с большей нагрузкой.",
        "performance_profile": "Готовый режим",
        "performance_profile_desc": "Набор настроек скорости и нагрузки",
        "profile_picker_title": "Готовый режим экспорта",
        "profile_picker_desc": "Выберите режим: бережный, обычный или быстрый.",
        "profile_apply": "Применить",
        "profile_custom": "Свои",
        "profile_stable": "Бережный",
        "profile_stable_desc": "Минимальная нагрузка. Подходит для больших чатов, если важнее стабильность.",
        "profile_balanced": "Обычный",
        "profile_balanced_desc": "Рекомендуемый режим по умолчанию: нормальная скорость и умеренная нагрузка.",
        "profile_aggressive": "Быстрый",
        "profile_aggressive_desc": "Быстрее и полнее догружает медиа, но сильнее нагружает Telegram и память.",
        "profile_chip_chunk": "Порция",
        "profile_chip_media": "Файлов",
        "profile_chip_safe": "Режим",
        "profile_safe_on": "полнее",
        "profile_safe_off": "бережно",
        "export_chat": "Экспорт чата",
        "exporting": "Экспорт чата...",
        "cancel": "Отмена",
        "export_complete": "Экспорт завершён",
        "export_failed": "Ошибка экспорта",
        "export_in_progress": "Экспорт уже выполняется",
        "run_in_background": "В фоне",
        "background_export": "Экспорт продолжится в фоне",
        "cannot_determine_chat": "Не удалось определить ID чата",
        "export_settings": "Настройки экспорта",
        "export_format": "Формат экспорта",
        "media_handling": "Обработка медиа",
        "message_filters": "Фильтры сообщений",
        "max_messages": "Макс. сообщений",
        "empty_all": "Пусто = все",
        "advanced": "Дополнительно",
        "max_size_mb": "Макс. размер (МБ)",
        "include_timestamps": "Временные метки",
        "include_avatars": "Аватары",
        "group_messages": "Группировать сообщения",
        "by_sender_time": "По отправителю и времени",
        "chunk_mode": "Режим оптимизации",
        "chunk_mode_desc": "Для больших чатов",
        "chunk_size": "Размер порции",
        "progress_overlay_settings": "Настройки фонового процесса",
        "enable_overlay_stroke": "Обводка оверлея",
        "enable_overlay_stroke_desc": "Показывать обводку у фонового индикатора",
        "enable_stroke_animation": "Анимация обводки",
        "enable_stroke_animation_desc": "Переливание цветов обводки",
        "overlay_opacity": "Прозрачность оверлея",
        "overlay_opacity_desc": "Уровень прозрачности (0-100%)",
        "overlay_position": "Положение оверлея",
        "position_top_center": "Сверху по центру",
        "position_top_left": "Сверху слева",
        "position_top_right": "Сверху справа",
        "position_bottom_center": "Снизу по центру",
        "position_bottom_left": "Снизу слева",
        "position_bottom_right": "Снизу справа",
        "settings_saved": "Настройки сохранены",
        "dex_missing": "Файл chatexport.dex не найден рядом с плагином и в кеше",
        "dex_load_failed": "Не удалось загрузить DEX-ядро",
        "dex_not_loaded": "DEX-ядро не загружено",
        "export_cancelled": "Экспорт отменён",
        "share_error": "Ошибка отправки",
        "preparing_share": "Подготовка к отправке...",
        "share_pending": "Файл готов. Диалог отправки откроется после возврата в приложение",
        "use_api_mode": "Режим API",
        "use_api_mode_desc": "Загружать историю напрямую с сервера",
        "progress_preparing_export": "Подготовка экспорта",
        "progress_fetching_history": "Загрузка истории с сервера",
        "progress_opening_database": "Открытие локальной базы сообщений",
        "progress_exporting_messages": "Экспорт сообщений",
        "progress_cancellation_requested": "Остановка экспорта",
        "progress_source_local": "Локальная БД",
        "progress_source_local_auto": "Локальная БД (авто)",
        "progress_count_total": "{current} из {total}",
        "progress_count_current": "{current} сообщений",
        "progress_media_skip": "без медиа",
        "progress_media_refs": "ссылки",
        "progress_media_download": "файлы",
        "sheet_subtitle": "Соберите экспорт с нужным форматом, содержимым и фоновым прогрессом.",
        "section_main": "Основное",
        "section_content": "Содержимое и производительность",
        "section_background": "Фоновый прогресс",
        "section_main_desc": "Выберите формат, медиа и лимит сообщений для этой выгрузки.",
        "section_content_desc": "Настройте содержание файла и режим чтения больших чатов.",
        "section_background_desc": "Определите, как выглядит фоновый индикатор во время экспорта.",
        "summary_format": "Формат",
        "summary_media": "Медиа",
        "summary_source": "Источник",
        "summary_local": "Локально",
        "summary_chunked": "Порциями",
        "summary_standard": "Стандартно",
        "html_only_hint": "Эти параметры применяются только к HTML-экспорту.",
        "all_messages": "Все сообщения",
        "default_and_export": "Сделать стандартом",
        "default_and_export_subtitle": "и экспортировать",
        "close_sheet": "Закрыть",
        "progress_subtitle": "Экспорт можно свернуть в фон без остановки.",
        "progress_counter_idle": "Ожидание данных",
        "progress_chip_format": "Формат",
        "progress_chip_media": "Медиа",
        "progress_chip_source": "Источник",
        "progress_chip_none": "Не используется",
        "export_auto_mode_note": "авто safe-mode",
        "export_media_stats_note": "медиа: local {local}, requested {requested}, copied {copied}, missing {missing}",
        "export_avatar_stats_note": "аватары: {copied}",
    },
    "en": {
        "remote_download_batch_size": "How many media files to fetch at once",
        "remote_download_batch_size_desc": "Size of one fetch batch. Higher values speed up export, but increase Telegram and memory load.",
        "allow_large_chat_remote_downloads": "Fetch missing media even in safe mode",
        "allow_large_chat_remote_downloads_desc": "Safe mode is a cautious mode for large chats. If enabled, export will still try to fetch files missing on the device, but it will run slower and with higher load.",
        "performance_profile": "Preset",
        "performance_profile_desc": "Preconfigured speed and load presets",
        "profile_picker_title": "Export Preset",
        "profile_picker_desc": "Choose a preset: gentle, normal, or fast.",
        "profile_apply": "Apply",
        "profile_custom": "Custom",
        "profile_stable": "Gentle",
        "profile_stable_desc": "Minimum load. Best for large chats when stability matters most.",
        "profile_balanced": "Normal",
        "profile_balanced_desc": "Recommended default: normal speed with moderate load.",
        "profile_aggressive": "Fast",
        "profile_aggressive_desc": "Fetches media faster and more completely, but puts more load on Telegram and memory.",
        "profile_chip_chunk": "Batch",
        "profile_chip_media": "Files",
        "profile_chip_safe": "Mode",
        "profile_safe_on": "full",
        "profile_safe_off": "gentle",
        "export_chat": "Export Chat",
        "exporting": "Exporting chat...",
        "cancel": "Cancel",
        "export_complete": "Export complete",
        "export_failed": "Export failed",
        "export_in_progress": "Export already in progress",
        "run_in_background": "Background",
        "background_export": "Export will continue in background",
        "cannot_determine_chat": "Cannot determine chat ID",
        "export_settings": "Export Settings",
        "export_format": "Export Format",
        "media_handling": "Media Handling",
        "message_filters": "Message Filters",
        "max_messages": "Max Messages",
        "empty_all": "Empty = all",
        "advanced": "Advanced",
        "max_size_mb": "Max Size (MB)",
        "include_timestamps": "Include Timestamps",
        "include_avatars": "Include Avatars",
        "group_messages": "Group Messages",
        "by_sender_time": "By sender and time",
        "chunk_mode": "Optimization Mode",
        "chunk_mode_desc": "For large chats",
        "chunk_size": "Chunk Size",
        "progress_overlay_settings": "Background Overlay Settings",
        "enable_overlay_stroke": "Overlay Stroke",
        "enable_overlay_stroke_desc": "Show border around progress overlay",
        "enable_stroke_animation": "Stroke Animation",
        "enable_stroke_animation_desc": "Animate overlay border colors",
        "overlay_opacity": "Overlay Opacity",
        "overlay_opacity_desc": "Opacity level (0-100%)",
        "overlay_position": "Overlay Position",
        "position_top_center": "Top Center",
        "position_top_left": "Top Left",
        "position_top_right": "Top Right",
        "position_bottom_center": "Bottom Center",
        "position_bottom_left": "Bottom Left",
        "position_bottom_right": "Bottom Right",
        "settings_saved": "Settings saved",
        "dex_missing": "chatexport.dex was not found near the plugin or in cache",
        "dex_load_failed": "Failed to load DEX core",
        "dex_not_loaded": "DEX core is not loaded",
        "export_cancelled": "Export cancelled",
        "share_error": "Share error",
        "preparing_share": "Preparing to share...",
        "share_pending": "File is ready. Share dialog will open after returning to the app",
        "use_api_mode": "API Mode",
        "use_api_mode_desc": "Fetch history directly from the server",
        "progress_preparing_export": "Preparing export",
        "progress_fetching_history": "Fetching history from server",
        "progress_opening_database": "Opening local message database",
        "progress_exporting_messages": "Exporting messages",
        "progress_cancellation_requested": "Cancelling export",
        "progress_source_local": "Local DB",
        "progress_source_local_auto": "Local DB (auto)",
        "progress_count_total": "{current} of {total}",
        "progress_count_current": "{current} messages",
        "progress_media_skip": "no media",
        "progress_media_refs": "links",
        "progress_media_download": "files",
        "sheet_subtitle": "Prepare the export with the right format, content, and background progress UI.",
        "section_main": "Main",
        "section_content": "Content and performance",
        "section_background": "Background progress",
        "section_main_desc": "Choose the format, media mode, and message limit for this export.",
        "section_content_desc": "Control file contents and optimize large-chat processing.",
        "section_background_desc": "Choose how the floating background progress looks during export.",
        "summary_format": "Format",
        "summary_media": "Media",
        "summary_source": "Source",
        "summary_local": "Local",
        "summary_chunked": "Chunked",
        "summary_standard": "Standard",
        "html_only_hint": "These options are available only for HTML export.",
        "all_messages": "All messages",
        "default_and_export": "Make default",
        "default_and_export_subtitle": "and export",
        "close_sheet": "Close",
        "progress_subtitle": "You can move this export to the background without stopping it.",
        "progress_counter_idle": "Waiting for data",
        "progress_chip_format": "Format",
        "progress_chip_media": "Media",
        "progress_chip_source": "Source",
        "progress_chip_none": "Not used",
        "export_auto_mode_note": "auto safe mode",
        "export_media_stats_note": "media: local {local}, requested {requested}, copied {copied}, missing {missing}",
        "export_avatar_stats_note": "avatars: {copied}",
    },
}


class DexDownloadDelegate(dynamic_proxy(NotificationCenter.NotificationCenterDelegate)):
    def __init__(self, plugin):
        super().__init__()
        self._plugin_ref = weakref.ref(plugin)

    def didReceivedNotification(self, notification_id, account, *args):
        plugin = self._plugin_ref()
        if plugin is not None:
            plugin._on_download_notification(notification_id, account, *args)


class DexOpenHook(MethodHook):
    def __init__(self, plugin):
        super().__init__()
        self._plugin_ref = weakref.ref(plugin)

    def before_hooked_method(self, param):
        plugin = self._plugin_ref()
        if plugin is None:
            return

        try:
            if not param.args:
                return

            message = param.args[0]
            if not plugin._is_supported_dex_message(message):
                return

            if "openForView" in param.method.getName():
                param.setResult(True)
            else:
                param.setResult(None)

            path = plugin._resolve_message_file_path(message)
            if path and os.path.exists(path):
                run_on_ui_thread(lambda: plugin.install_dex_from_path(path, announce=True))
            else:
                run_on_ui_thread(lambda: plugin.download_dex_from_message(message))
        except Exception as error:
            plugin._log(f"Ошибка перехвата открытия DEX: {error}")


class DexLocalFileOpenHook(MethodHook):
    def __init__(self, plugin):
        super().__init__()
        self._plugin_ref = weakref.ref(plugin)

    def before_hooked_method(self, param):
        plugin = self._plugin_ref()
        if plugin is None:
            return

        try:
            if not param.args or len(param.args) < 2:
                return

            file_obj = param.args[0]
            file_name = str(param.args[1] or "")
            if not plugin._is_dex_file_name(file_name):
                return

            path = None
            try:
                if file_obj is not None and hasattr(file_obj, "getAbsolutePath"):
                    path = str(file_obj.getAbsolutePath())
            except Exception:
                path = None

            if not path or not os.path.exists(path):
                return

            param.setResult(True)
            run_on_ui_thread(lambda: plugin.install_dex_from_path(path, announce=True))
        except Exception as error:
            plugin._log(f"Ошибка перехвата локального DEX-файла: {error}")


class DexLoader:
    def __init__(self, plugin: BasePlugin):
        self.plugin = plugin
        self.dex_class = None
        self.dex_loader = None
        self._lock = threading.RLock()
        self.cache_dir = os.path.join(
            ApplicationLoader.applicationContext.getFilesDir().getAbsolutePath(),
            DEX_CACHE_DIR,
        )
        self.cache_file = os.path.join(self.cache_dir, DEX_FILE_NAME)
        os.makedirs(self.cache_dir, exist_ok=True)

    def _plugin_dir(self) -> str:
        source = globals().get("__file__")
        if source:
            return os.path.dirname(os.path.abspath(str(source)))
        return os.getcwd()

    def find_local_dex_path(self) -> Optional[str]:
        candidate = os.path.join(self._plugin_dir(), DEX_FILE_NAME)
        if os.path.exists(candidate):
            return candidate
        return None

    def load_from_cache(self) -> Optional[bytes]:
        try:
            if os.path.exists(self.cache_file):
                with open(self.cache_file, "rb") as file_obj:
                    return file_obj.read()
        except Exception as error:
            self.plugin._log(f"Ошибка чтения кеша DEX: {error}")
        return None

    def clear_cache(self) -> None:
        try:
            if os.path.exists(self.cache_file):
                os.remove(self.cache_file)
        except Exception as error:
            self.plugin._log(f"Ошибка очистки кеша DEX: {error}")

    def is_loaded(self) -> bool:
        return self.dex_class is not None

    def install_from_local(self) -> str:
        path = self.find_local_dex_path()
        if not path:
            raise FileNotFoundError(DEX_FILE_NAME)
        self.install_from_path(path)
        return path

    def install_from_path(self, path: str) -> None:
        with open(path, "rb") as file_obj:
            self.install_from_bytes(file_obj.read())

    def install_from_bytes(self, dex_bytes: bytes) -> None:
        with self._lock:
            self.unload()
            with open(self.cache_file, "wb") as file_obj:
                file_obj.write(dex_bytes)
            try:
                self._start_from_bytes_locked(dex_bytes)
            except Exception:
                self.clear_cache()
                raise

    def start_from_bytes(self, dex_bytes: bytes) -> None:
        with self._lock:
            self.unload()
            self._start_from_bytes_locked(dex_bytes)

    def _start_from_bytes_locked(self, dex_bytes: bytes) -> None:
        buffer = ByteBuffer.wrap(dex_bytes)
        self.dex_loader = InMemoryDexClassLoader(
            buffer,
            ApplicationLoader.applicationContext.getClassLoader(),
        )
        self.dex_class = self._load_supported_class()
        self._invoke_void_bridge(DEX_START_BRIDGE)

    def _load_supported_class(self):
        try:
            return self.dex_loader.loadClass(DEX_CLASS_NAME)
        except Exception as error:
            raise RuntimeError(f"В DEX не найден класс {DEX_CLASS_NAME}") from error

    def _invoke_bridge(self, method_name: str, expect_string: bool, *args):
        if self.dex_class is None:
            raise RuntimeError("DEX-класс не загружен")

        try:
            if args:
                if len(args) != 1 or not isinstance(args[0], str):
                    raise TypeError(f"Неподдерживаемые аргументы мостового метода {method_name}")
                string_class = jclass("java.lang.String")
                method = self.dex_class.getMethod(method_name, string_class)
                result = method.invoke(None, args[0])
            else:
                method = self.dex_class.getMethod(method_name)
                result = method.invoke(None)
        except Exception as error:
            if "NoSuchMethodException" in str(error):
                raise RuntimeError(f"В DEX отсутствует мостовой метод {method_name}") from error
            raise

        if expect_string:
            return None if result is None else str(result)
        return None

    def _invoke_void_bridge(self, method_name: str, *args) -> None:
        self._invoke_bridge(method_name, False, *args)

    def _invoke_string_bridge(self, method_name: str, *args) -> Optional[str]:
        return self._invoke_bridge(method_name, True, *args)

    def update_settings(self, settings: Dict[str, Any]) -> None:
        with self._lock:
            if self.dex_class is None:
                return
            self._invoke_void_bridge(
                DEX_SETTINGS_BRIDGE,
                json.dumps(settings, ensure_ascii=False),
            )

    def start_export(self, request_json: str) -> None:
        with self._lock:
            self._invoke_void_bridge(DEX_EXPORT_BRIDGE, request_json)

    def get_export_state(self) -> Optional[str]:
        with self._lock:
            return self._invoke_string_bridge(DEX_STATE_BRIDGE)

    def consume_result(self) -> Optional[str]:
        with self._lock:
            return self._invoke_string_bridge(DEX_RESULT_BRIDGE)

    def cancel_export(self) -> None:
        with self._lock:
            if self.dex_class is None:
                return
            self._invoke_void_bridge(DEX_CANCEL_BRIDGE)

    def unload(self) -> None:
        with self._lock:
            if self.dex_class is None:
                return
            try:
                self._invoke_void_bridge(DEX_UNLOAD_BRIDGE)
            finally:
                self.dex_class = None
                self.dex_loader = None


class ExportSettingsBottomSheet:
    def __init__(self, context, plugin, dialog_id, account_id: Optional[int] = None):
        self.plugin = plugin
        self.dialog_id = dialog_id
        self.account_id = plugin._coerce_account_id(account_id)
        self.context = context
        self.resources_provider = plugin._get_resources_provider()
        self.media_handling_row = None
        self.profile_row = None
        self.include_avatars_cell = None
        self.chunk_size_row = None
        self.media_download_settings_container = None
        self.remote_download_batch_size_row = None
        self.allow_large_chat_remote_downloads_cell = None
        self.enable_overlay_stroke_cell = None
        self.enable_stroke_animation_cell = None
        self.media_hint_view = None
        self.value_labels = {}
        self.summary_chips = {}
        self.bottom_sheet = self._create_bottom_sheet(context)

        self.temp_settings = {
            "export_format": plugin.get_setting("export_format", CONFIG_DEFAULTS["export_format"]),
            "media_handling": plugin.get_setting("media_handling", CONFIG_DEFAULTS["media_handling"]),
            "max_messages": plugin.get_setting("max_messages", CONFIG_DEFAULTS["max_messages"]),
            "max_size_mb": plugin.get_setting("max_size_mb", CONFIG_DEFAULTS["max_size_mb"]),
            "remote_download_batch_size": plugin.get_setting("remote_download_batch_size", CONFIG_DEFAULTS["remote_download_batch_size"]),
            "allow_large_chat_remote_downloads": plugin.get_setting("allow_large_chat_remote_downloads", CONFIG_DEFAULTS["allow_large_chat_remote_downloads"]),
            "use_api_mode": plugin.get_setting("use_api_mode", CONFIG_DEFAULTS["use_api_mode"]),
            "include_timestamps": plugin.get_setting("include_timestamps", CONFIG_DEFAULTS["include_timestamps"]),
            "include_avatars": plugin.get_setting("include_avatars", CONFIG_DEFAULTS["include_avatars"]),
            "group_messages": plugin.get_setting("group_messages", CONFIG_DEFAULTS["group_messages"]),
            "use_chunked_mode": plugin.get_setting("use_chunked_mode", CONFIG_DEFAULTS["use_chunked_mode"]),
            "chunk_size": plugin.get_setting("chunk_size", CONFIG_DEFAULTS["chunk_size"]),
            "enable_overlay_stroke": plugin.get_setting("enable_overlay_stroke", CONFIG_DEFAULTS["enable_overlay_stroke"]),
            "enable_stroke_animation": plugin.get_setting("enable_stroke_animation", CONFIG_DEFAULTS["enable_stroke_animation"]),
            "overlay_opacity": plugin.get_setting("overlay_opacity", CONFIG_DEFAULTS["overlay_opacity"]),
            "overlay_position": plugin.get_setting("overlay_position", CONFIG_DEFAULTS["overlay_position"]),
        }

        container_layout = LinearLayout(context)
        container_layout.setOrientation(LinearLayout.VERTICAL)
        container_layout.setBackgroundColor(self._color(Theme.key_dialogBackground))

        scroll_view = ScrollView(context)
        scroll_view.setFillViewport(True)
        scroll_view.setVerticalScrollBarEnabled(False)

        self.content_layout = LinearLayout(context)
        self.content_layout.setOrientation(LinearLayout.VERTICAL)
        self.content_layout.setBackgroundColor(self._color(Theme.key_dialogBackground))
        self.content_layout.setPadding(
            AndroidUtilities.dp(16),
            AndroidUtilities.dp(14),
            AndroidUtilities.dp(16),
            AndroidUtilities.dp(12),
        )

        self._build_content()
        scroll_view.addView(self.content_layout)

        scroll_params = LinearLayout.LayoutParams(
            LinearLayout.LayoutParams.MATCH_PARENT,
            0,
            1.0,
        )
        container_layout.addView(scroll_view, scroll_params)

        button_container = self._create_action_bar()
        container_layout.addView(
            button_container,
            LinearLayout.LayoutParams(
                LinearLayout.LayoutParams.MATCH_PARENT,
                LinearLayout.LayoutParams.WRAP_CONTENT,
            ),
        )
        self.bottom_sheet.setCustomView(container_layout)

    def show(self):
        self.bottom_sheet.show()

    def dismiss(self):
        self.bottom_sheet.dismiss()

    def _create_bottom_sheet(self, context):
        try:
            if self.resources_provider is not None:
                sheet = BottomSheet(context, False, self.resources_provider)
            else:
                sheet = BottomSheet(context, False)
        except Exception:
            sheet = BottomSheet(context, False)
        try:
            sheet.fixNavigationBar(self._color(Theme.key_dialogBackground))
        except Exception:
            pass
        return sheet

    def _color(self, key):
        try:
            if self.resources_provider is not None:
                return _android_color_int(Theme.getColor(key, self.resources_provider))
        except Exception:
            pass
        return _android_color_int(Theme.getColor(key))

    def _alpha(self, color: int, alpha: float) -> int:
        alpha_value = max(0, min(255, int(float(alpha) * 255)))
        return _android_color_int((alpha_value << 24) | (int(color) & 0x00FFFFFF))

    def _surface_background(self, radius_dp: int = 22, stroke_alpha: float = 0.06):
        background = GradientDrawable()
        background.setCornerRadius(AndroidUtilities.dp(radius_dp))
        background.setColor(self._color(Theme.key_windowBackgroundWhite))
        background.setStroke(AndroidUtilities.dp(1), self._alpha(self._color(Theme.key_dialogTextBlack), stroke_alpha))
        return background

    def _selector_background(self, fill_color: Optional[int] = None, radius_dp: int = 18):
        background = GradientDrawable()
        background.setCornerRadius(AndroidUtilities.dp(radius_dp))
        background.setColor(fill_color if fill_color is not None else Color.TRANSPARENT)
        try:
            return RippleDrawable(
                ColorStateList.valueOf(self._alpha(self._color(Theme.key_listSelector), 0.22)),
                background,
                None,
            )
        except Exception:
            return background

    def _apply_clickable_background(self, view, radius_dp: int = 18):
        try:
            view.setBackground(self._selector_background(radius_dp=radius_dp))
        except Exception:
            pass

    def _section_params(self):
        params = LinearLayout.LayoutParams(
            LinearLayout.LayoutParams.MATCH_PARENT,
            LinearLayout.LayoutParams.WRAP_CONTENT,
        )
        params.topMargin = AndroidUtilities.dp(14)
        return params

    def _build_content(self):
        self.content_layout.addView(self._create_hero_card())
        self.content_layout.addView(self._create_main_section(), self._section_params())
        self.content_layout.addView(self._create_content_section(), self._section_params())
        self.content_layout.addView(self._create_background_section(), self._section_params())
        self._update_dynamic_state()

    def _create_hero_card(self):
        hero = LinearLayout(self.context)
        hero.setOrientation(LinearLayout.VERTICAL)
        hero.setPadding(
            AndroidUtilities.dp(18),
            AndroidUtilities.dp(18),
            AndroidUtilities.dp(18),
            AndroidUtilities.dp(16),
        )
        hero.setBackground(self._surface_background(radius_dp=26, stroke_alpha=0.08))

        header_row = LinearLayout(self.context)
        header_row.setOrientation(LinearLayout.HORIZONTAL)
        header_row.setGravity(Gravity.CENTER_VERTICAL)

        icon_wrap = FrameLayout(self.context)
        icon_bg = GradientDrawable()
        icon_bg.setShape(GradientDrawable.OVAL)
        icon_bg.setColor(self._alpha(self._color(Theme.key_featuredStickers_addButton), 0.14))
        icon_wrap.setBackground(icon_bg)

        icon_view = ImageView(self.context)
        icon_view.setImageResource(TelegramR.drawable.msg_download)
        try:
            icon_view.setColorFilter(self._color(Theme.key_featuredStickers_addButton))
        except Exception:
            pass
        icon_wrap.addView(
            icon_view,
            FrameLayout.LayoutParams(
                AndroidUtilities.dp(24),
                AndroidUtilities.dp(24),
                Gravity.CENTER,
            ),
        )
        header_row.addView(
            icon_wrap,
            LinearLayout.LayoutParams(AndroidUtilities.dp(54), AndroidUtilities.dp(54)),
        )

        title_block = LinearLayout(self.context)
        title_block.setOrientation(LinearLayout.VERTICAL)
        title_params = LinearLayout.LayoutParams(0, LinearLayout.LayoutParams.WRAP_CONTENT, 1.0)
        title_params.leftMargin = AndroidUtilities.dp(14)

        title_view = TextView(self.context)
        title_view.setText(self.plugin._t("export_settings"))
        title_view.setTextColor(self._color(Theme.key_dialogTextBlack))
        title_view.setTextSize(TypedValue.COMPLEX_UNIT_DIP, 21)
        title_view.setTypeface(Typeface.DEFAULT_BOLD)
        title_block.addView(title_view)

        subtitle_view = TextView(self.context)
        subtitle_view.setText(self.plugin._t("sheet_subtitle"))
        subtitle_view.setTextColor(self._color(Theme.key_dialogTextGray3))
        subtitle_view.setTextSize(TypedValue.COMPLEX_UNIT_DIP, 13.5)
        subtitle_view.setLineSpacing(AndroidUtilities.dp(2), 1.0)
        title_block.addView(subtitle_view)
        header_row.addView(title_block, title_params)

        close_button = FrameLayout(self.context)
        close_button.setClickable(True)
        close_button.setFocusable(True)
        close_button.setBackground(self._selector_background(self._alpha(self._color(Theme.key_groupcreate_spanBackground), 0.55), 18))
        close_button.setOnClickListener(OnClickListener(lambda *_: self.dismiss()))

        close_icon = ImageView(self.context)
        close_icon.setImageResource(TelegramR.drawable.msg_close)
        try:
            close_icon.setColorFilter(self._color(Theme.key_dialogIcon))
        except Exception:
            pass
        close_button.addView(
            close_icon,
            FrameLayout.LayoutParams(
                AndroidUtilities.dp(18),
                AndroidUtilities.dp(18),
                Gravity.CENTER,
            ),
        )
        header_row.addView(
            close_button,
            LinearLayout.LayoutParams(AndroidUtilities.dp(36), AndroidUtilities.dp(36)),
        )

        hero.addView(header_row)

        chips_row = LinearLayout(self.context)
        chips_row.setOrientation(LinearLayout.HORIZONTAL)
        chips_row.setGravity(Gravity.LEFT | Gravity.CENTER_VERTICAL)
        chips_params = LinearLayout.LayoutParams(
            LinearLayout.LayoutParams.MATCH_PARENT,
            LinearLayout.LayoutParams.WRAP_CONTENT,
        )
        chips_params.topMargin = AndroidUtilities.dp(14)

        self.summary_chips["format"] = self._create_summary_chip()
        self.summary_chips["media"] = self._create_summary_chip()
        self.summary_chips["source"] = self._create_summary_chip()
        for index, key in enumerate(("format", "media", "source")):
            params = LinearLayout.LayoutParams(0, LinearLayout.LayoutParams.WRAP_CONTENT, 1.0)
            if index > 0:
                params.leftMargin = AndroidUtilities.dp(8)
            chips_row.addView(self.summary_chips[key], params)

        hero.addView(chips_row, chips_params)
        return hero

    def _create_summary_chip(self):
        chip = TextView(self.context)
        chip.setGravity(Gravity.CENTER)
        chip.setMaxLines(1)
        chip.setSingleLine(True)
        chip.setEllipsize(jclass("android.text.TextUtils$TruncateAt").END)
        chip.setTypeface(Typeface.DEFAULT_BOLD)
        chip.setTextSize(TypedValue.COMPLEX_UNIT_DIP, 12.5)
        chip.setPadding(
            AndroidUtilities.dp(10),
            AndroidUtilities.dp(10),
            AndroidUtilities.dp(10),
            AndroidUtilities.dp(10),
        )
        chip_bg = GradientDrawable()
        chip_bg.setCornerRadius(AndroidUtilities.dp(16))
        chip_bg.setColor(self._alpha(self._color(Theme.key_groupcreate_spanBackground), 0.92))
        chip.setBackground(chip_bg)
        chip.setTextColor(self._color(Theme.key_windowBackgroundWhiteBlueText))
        return chip

    def _create_section_shell(self, title: str, description: str):
        section = LinearLayout(self.context)
        section.setOrientation(LinearLayout.VERTICAL)

        title_view = TextView(self.context)
        title_view.setText(title)
        title_view.setTextColor(self._color(Theme.key_dialogTextBlack))
        title_view.setTypeface(Typeface.DEFAULT_BOLD)
        title_view.setTextSize(TypedValue.COMPLEX_UNIT_DIP, 16)
        section.addView(title_view)

        description_view = TextView(self.context)
        description_view.setText(description)
        description_view.setTextColor(self._color(Theme.key_dialogTextGray3))
        description_view.setTextSize(TypedValue.COMPLEX_UNIT_DIP, 13)
        description_view.setLineSpacing(AndroidUtilities.dp(2), 1.0)
        description_params = LinearLayout.LayoutParams(
            LinearLayout.LayoutParams.MATCH_PARENT,
            LinearLayout.LayoutParams.WRAP_CONTENT,
        )
        description_params.topMargin = AndroidUtilities.dp(4)
        section.addView(description_view, description_params)

        card = LinearLayout(self.context)
        card.setOrientation(LinearLayout.VERTICAL)
        card.setBackground(self._surface_background())
        card_params = LinearLayout.LayoutParams(
            LinearLayout.LayoutParams.MATCH_PARENT,
            LinearLayout.LayoutParams.WRAP_CONTENT,
        )
        card_params.topMargin = AndroidUtilities.dp(10)
        section.addView(card, card_params)
        return section, card

    def _add_group_divider(self, parent):
        divider = View(self.context)
        divider.setBackgroundColor(self._alpha(self._color(Theme.key_divider), 0.9))
        params = LinearLayout.LayoutParams(
            LinearLayout.LayoutParams.MATCH_PARENT,
            1,
        )
        params.leftMargin = AndroidUtilities.dp(16)
        params.rightMargin = AndroidUtilities.dp(16)
        parent.addView(divider, params)

    def _create_value_row(self, key: str, title: str, subtitle: str = "", on_click=None):
        row = LinearLayout(self.context)
        row.setOrientation(LinearLayout.HORIZONTAL)
        row.setGravity(Gravity.CENTER_VERTICAL)
        row.setMinimumHeight(AndroidUtilities.dp(58))
        row.setPadding(
            AndroidUtilities.dp(16),
            AndroidUtilities.dp(12),
            AndroidUtilities.dp(16),
            AndroidUtilities.dp(12),
        )
        self._apply_clickable_background(row, radius_dp=18)

        text_column = LinearLayout(self.context)
        text_column.setOrientation(LinearLayout.VERTICAL)
        text_params = LinearLayout.LayoutParams(0, LinearLayout.LayoutParams.WRAP_CONTENT, 1.0)

        title_view = TextView(self.context)
        title_view.setText(title)
        title_view.setTextColor(self._color(Theme.key_dialogTextBlack))
        title_view.setTextSize(TypedValue.COMPLEX_UNIT_DIP, 15)
        text_column.addView(title_view)

        subtitle_view = None
        if subtitle:
            subtitle_view = TextView(self.context)
            subtitle_view.setText(subtitle)
            subtitle_view.setTextColor(self._color(Theme.key_dialogTextGray3))
            subtitle_view.setTextSize(TypedValue.COMPLEX_UNIT_DIP, 12.5)
            subtitle_params = LinearLayout.LayoutParams(
                LinearLayout.LayoutParams.WRAP_CONTENT,
                LinearLayout.LayoutParams.WRAP_CONTENT,
            )
            subtitle_params.topMargin = AndroidUtilities.dp(2)
            text_column.addView(subtitle_view, subtitle_params)

        row.addView(text_column, text_params)

        value_view = TextView(self.context)
        value_view.setTextColor(self._color(Theme.key_windowBackgroundWhiteBlueText))
        value_view.setTextSize(TypedValue.COMPLEX_UNIT_DIP, 14)
        value_view.setTypeface(Typeface.DEFAULT_BOLD)
        value_view.setGravity(Gravity.RIGHT | Gravity.CENTER_VERTICAL)
        value_view.setSingleLine(True)
        row.addView(value_view)

        if on_click is not None:
            row.setOnClickListener(OnClickListener(lambda *_: on_click()))

        self.value_labels[key] = {
            "row": row,
            "value": value_view,
            "subtitle": subtitle_view,
        }
        return row

    def _set_value_row_enabled(self, key: str, enabled: bool):
        entry = self.value_labels.get(key)
        if not entry:
            return
        row = entry["row"]
        row.setEnabled(enabled)
        row.setClickable(enabled)
        row.setAlpha(1.0 if enabled else 0.48)

    def _set_value_row_text(self, key: str, value: str, subtitle: Optional[str] = None):
        entry = self.value_labels.get(key)
        if not entry:
            return
        entry["value"].setText(str(value or ""))
        if subtitle is not None and entry.get("subtitle") is not None:
            entry["subtitle"].setText(subtitle)

    def _create_switch_cell(self, key, text, subtext):
        cell = TextCheckCell(self.context)
        cell.setBackground(self._selector_background(radius_dp=18))
        value = bool(self.temp_settings.get(key))
        if subtext:
            cell.setTextAndValueAndCheck(text, subtext, value, True, False)
        else:
            cell.setTextAndCheck(text, value, False)

        def on_click(*_):
            self.temp_settings[key] = not bool(self.temp_settings.get(key))
            new_value = self.temp_settings[key]
            try:
                checkbox = cell.getCheckBox()
                if checkbox:
                    checkbox.setChecked(new_value, True)
                elif subtext:
                    cell.setTextAndValueAndCheck(text, subtext, new_value, True, True)
                else:
                    cell.setTextAndCheck(text, new_value, True)
            except Exception:
                if subtext:
                    cell.setTextAndValueAndCheck(text, subtext, new_value, True, True)
                else:
                    cell.setTextAndCheck(text, new_value, True)
            self._update_dynamic_state()

        cell.setOnClickListener(OnClickListener(on_click))
        return cell

    def _create_format_selector(self):
        container = LinearLayout(self.context)
        container.setOrientation(LinearLayout.VERTICAL)
        container.setPadding(
            AndroidUtilities.dp(16),
            AndroidUtilities.dp(14),
            AndroidUtilities.dp(16),
            AndroidUtilities.dp(10),
        )

        label = TextView(self.context)
        label.setText(self.plugin._t("export_format"))
        label.setTextColor(self._color(Theme.key_dialogTextBlack))
        label.setTextSize(TypedValue.COMPLEX_UNIT_DIP, 14.5)
        label.setTypeface(Typeface.DEFAULT_BOLD)
        container.addView(label)

        selector = SlideChooseView(self.context, self.resources_provider)
        selector.setOptions(
            self.plugin._coerce_index(self.temp_settings.get("export_format", 0), 0, 2, 0),
            "HTML",
            "JSON",
            "TXT",
        )

        callback_cls = jclass("org.telegram.ui.Components.SlideChooseView$Callback")
        owner = self

        class _FormatCallback(dynamic_proxy(callback_cls)):
            def onOptionSelected(self, index):
                owner.temp_settings["export_format"] = int(index)
                owner._update_dynamic_state()

            def onTouchEnd(self):
                return None

        selector.setCallback(_FormatCallback())
        selector_params = LinearLayout.LayoutParams(
            LinearLayout.LayoutParams.MATCH_PARENT,
            LinearLayout.LayoutParams.WRAP_CONTENT,
        )
        selector_params.topMargin = AndroidUtilities.dp(10)
        container.addView(selector, selector_params)
        return container

    def _create_main_section(self):
        section, card = self._create_section_shell(
            self.plugin._t("section_main"),
            self.plugin._t("section_main_desc"),
        )
        card.addView(self._create_format_selector())
        self._add_group_divider(card)

        self.media_handling_row = self._create_value_row(
            "media_handling",
            self.plugin._t("media_handling"),
            on_click=lambda: self._show_selector_dialog(
                "media_handling",
                self.plugin._t("media_handling"),
                self._media_options(),
            ),
        )
        card.addView(self.media_handling_row)
        self._add_group_divider(card)

        card.addView(
            self._create_value_row(
                "max_messages",
                self.plugin._t("max_messages"),
                subtitle=self.plugin._t("empty_all"),
                on_click=lambda: self._show_input_dialog(
                    "max_messages",
                    self.plugin._t("max_messages"),
                    self.plugin._t("empty_all"),
                    InputType.TYPE_CLASS_NUMBER,
                ),
            )
        )

        self.media_hint_view = TextView(self.context)
        self.media_hint_view.setText(self.plugin._t("html_only_hint"))
        self.media_hint_view.setTextColor(self._color(Theme.key_dialogTextGray3))
        self.media_hint_view.setTextSize(TypedValue.COMPLEX_UNIT_DIP, 12.5)
        self.media_hint_view.setVisibility(View.GONE)
        hint_params = LinearLayout.LayoutParams(
            LinearLayout.LayoutParams.MATCH_PARENT,
            LinearLayout.LayoutParams.WRAP_CONTENT,
        )
        hint_params.leftMargin = AndroidUtilities.dp(16)
        hint_params.rightMargin = AndroidUtilities.dp(16)
        hint_params.bottomMargin = AndroidUtilities.dp(14)
        card.addView(self.media_hint_view, hint_params)
        return section

    def _create_content_section(self):
        section, card = self._create_section_shell(
            self.plugin._t("section_content"),
            self.plugin._t("section_content_desc"),
        )

        self.profile_row = self._create_value_row(
            "performance_profile",
            self.plugin._t("performance_profile"),
            subtitle=self.plugin._t("performance_profile_desc"),
            on_click=self._open_profile_picker_dialog,
        )
        card.addView(self.profile_row)
        self._add_group_divider(card)

        self.include_timestamps_cell = self._create_switch_cell("include_timestamps", self.plugin._t("include_timestamps"), "")
        card.addView(self.include_timestamps_cell)
        self._add_group_divider(card)

        self.include_avatars_cell = self._create_switch_cell("include_avatars", self.plugin._t("include_avatars"), "")
        card.addView(self.include_avatars_cell)
        self._add_group_divider(card)

        self.group_messages_cell = self._create_switch_cell("group_messages", self.plugin._t("group_messages"), self.plugin._t("by_sender_time"))
        card.addView(self.group_messages_cell)
        self._add_group_divider(card)

        self.use_api_mode_cell = self._create_switch_cell("use_api_mode", self.plugin._t("use_api_mode"), self.plugin._t("use_api_mode_desc"))
        card.addView(self.use_api_mode_cell)
        self._add_group_divider(card)

        self.use_chunked_mode_cell = self._create_switch_cell("use_chunked_mode", self.plugin._t("chunk_mode"), self.plugin._t("chunk_mode_desc"))
        card.addView(self.use_chunked_mode_cell)
        self._add_group_divider(card)

        self.chunk_size_row = self._create_value_row(
            "chunk_size",
            self.plugin._t("chunk_size"),
            subtitle=self.plugin._t("chunk_mode_desc"),
            on_click=lambda: self._show_input_dialog(
                "chunk_size",
                self.plugin._t("chunk_size"),
                str(DEFAULT_CHUNK_SIZE),
                InputType.TYPE_CLASS_NUMBER,
            ),
        )
        card.addView(self.chunk_size_row)
        self._add_group_divider(card)

        card.addView(
            self._create_value_row(
                "max_size_mb",
                self.plugin._t("max_size_mb"),
                on_click=lambda: self._show_input_dialog(
                    "max_size_mb",
                    self.plugin._t("max_size_mb"),
                    CONFIG_DEFAULTS["max_size_mb"],
                    InputType.TYPE_CLASS_NUMBER,
                ),
            )
        )

        self.media_download_settings_container = LinearLayout(self.context)
        self.media_download_settings_container.setOrientation(LinearLayout.VERTICAL)

        self._add_group_divider(self.media_download_settings_container)
        self.remote_download_batch_size_row = self._create_value_row(
            "remote_download_batch_size",
            self.plugin._t("remote_download_batch_size"),
            subtitle=self.plugin._t("remote_download_batch_size_desc"),
            on_click=lambda: self._show_input_dialog(
                "remote_download_batch_size",
                self.plugin._t("remote_download_batch_size"),
                str(CONFIG_DEFAULTS["remote_download_batch_size"]),
                InputType.TYPE_CLASS_NUMBER,
            ),
        )
        self.media_download_settings_container.addView(self.remote_download_batch_size_row)
        self._add_group_divider(self.media_download_settings_container)

        self.allow_large_chat_remote_downloads_cell = self._create_switch_cell(
            "allow_large_chat_remote_downloads",
            self.plugin._t("allow_large_chat_remote_downloads"),
            self.plugin._t("allow_large_chat_remote_downloads_desc"),
        )
        self.media_download_settings_container.addView(self.allow_large_chat_remote_downloads_cell)
        card.addView(self.media_download_settings_container)
        return section

    def _create_background_section(self):
        section, card = self._create_section_shell(
            self.plugin._t("section_background"),
            self.plugin._t("section_background_desc"),
        )

        self.enable_overlay_stroke_cell = self._create_switch_cell(
            "enable_overlay_stroke",
            self.plugin._t("enable_overlay_stroke"),
            self.plugin._t("enable_overlay_stroke_desc"),
        )
        card.addView(self.enable_overlay_stroke_cell)
        self._add_group_divider(card)

        self.enable_stroke_animation_cell = self._create_switch_cell(
            "enable_stroke_animation",
            self.plugin._t("enable_stroke_animation"),
            self.plugin._t("enable_stroke_animation_desc"),
        )
        card.addView(self.enable_stroke_animation_cell)
        self._add_group_divider(card)

        card.addView(
            self._create_value_row(
                "overlay_opacity",
                self.plugin._t("overlay_opacity"),
                subtitle=self.plugin._t("overlay_opacity_desc"),
                on_click=lambda: self._show_input_dialog(
                    "overlay_opacity",
                    self.plugin._t("overlay_opacity"),
                    self.plugin._t("overlay_opacity_desc"),
                    InputType.TYPE_CLASS_NUMBER,
                ),
            )
        )
        self._add_group_divider(card)

        card.addView(
            self._create_value_row(
                "overlay_position",
                self.plugin._t("overlay_position"),
                on_click=lambda: self._show_selector_dialog(
                    "overlay_position",
                    self.plugin._t("overlay_position"),
                    self._overlay_position_options(),
                ),
            )
        )
        return section

    def _media_options(self):
        return ["Пропустить", "Ссылки", "Скачать"] if self.plugin._is_russian() else ["Skip", "References", "Download"]

    def _overlay_position_options(self):
        return [
            self.plugin._t("position_top_center"),
            self.plugin._t("position_top_left"),
            self.plugin._t("position_top_right"),
            self.plugin._t("position_bottom_center"),
            self.plugin._t("position_bottom_left"),
            self.plugin._t("position_bottom_right"),
        ]

    def _progress_media_label(self, settings: Dict[str, Any]) -> str:
        export_format = self.plugin._coerce_index(
            settings.get("export_format", CONFIG_DEFAULTS["export_format"]),
            0,
            2,
            CONFIG_DEFAULTS["export_format"],
        )
        if export_format != 0:
            return self.plugin._t("progress_chip_none")
        media_keys = ("progress_media_skip", "progress_media_refs", "progress_media_download")
        media_index = self.plugin._coerce_index(
            settings.get("media_handling", CONFIG_DEFAULTS["media_handling"]),
            0,
            2,
            CONFIG_DEFAULTS["media_handling"],
        )
        return self.plugin._t(media_keys[media_index])

    def _update_dynamic_state(self):
        export_format = int(self.temp_settings.get("export_format", 0) or 0)
        is_html = export_format == 0
        media_download_mode = is_html and self.plugin._coerce_index(
            self.temp_settings.get("media_handling", CONFIG_DEFAULTS["media_handling"]),
            0,
            2,
            CONFIG_DEFAULTS["media_handling"],
        ) == 2
        if not is_html:
            self.temp_settings["media_handling"] = 0
            self.temp_settings["include_avatars"] = False

        if self.media_handling_row:
            self._set_value_row_enabled("media_handling", is_html)

        if self.include_avatars_cell:
            self.include_avatars_cell.setEnabled(is_html)
            self.include_avatars_cell.setAlpha(1.0 if is_html else 0.48)
            if not is_html:
                try:
                    checkbox = self.include_avatars_cell.getCheckBox()
                    if checkbox:
                        checkbox.setChecked(False, True)
                except Exception:
                    self.include_avatars_cell.setTextAndCheck(self.plugin._t("include_avatars"), False, True)

        if self.media_hint_view:
            self.media_hint_view.setVisibility(View.VISIBLE if not is_html else View.GONE)

        if self.chunk_size_row:
            self.chunk_size_row.setVisibility(View.VISIBLE if bool(self.temp_settings.get("use_chunked_mode")) else View.GONE)

        if self.media_download_settings_container:
            self.media_download_settings_container.setVisibility(View.VISIBLE if media_download_mode else View.GONE)

        if self.enable_stroke_animation_cell:
            stroke_enabled = bool(self.temp_settings.get("enable_overlay_stroke"))
            self.enable_stroke_animation_cell.setEnabled(stroke_enabled)
            self.enable_stroke_animation_cell.setAlpha(1.0 if stroke_enabled else 0.5)

        self._refresh_summary_chips()
        self._refresh_value_rows()

    def _refresh_summary_chips(self):
        export_format = self.plugin._coerce_index(self.temp_settings.get("export_format", 0), 0, 2, 0)
        format_label = ("HTML", "JSON", "TXT")[export_format]
        media_label = self._progress_media_label(self.temp_settings)
        source_label = self.plugin._t("summary_chunked") if bool(self.temp_settings.get("use_chunked_mode")) else (
            "API" if bool(self.temp_settings.get("use_api_mode", CONFIG_DEFAULTS["use_api_mode"])) else self.plugin._t("summary_local")
        )
        if self.summary_chips.get("format"):
            self.summary_chips["format"].setText(format_label)
        if self.summary_chips.get("media"):
            self.summary_chips["media"].setText(media_label)
        if self.summary_chips.get("source"):
            self.summary_chips["source"].setText(source_label)

    def _refresh_value_rows(self):
        self._set_value_row_text("performance_profile", self._profile_label_for_settings(self.temp_settings))

        max_messages = str(self.temp_settings.get("max_messages", "") or "").strip()
        self._set_value_row_text("max_messages", max_messages if max_messages else self.plugin._t("all_messages"))

        media_index = self.plugin._coerce_index(self.temp_settings.get("media_handling", 0), 0, 2, 0)
        self._set_value_row_text("media_handling", self._media_options()[media_index])

        max_size = str(self.temp_settings.get("max_size_mb", CONFIG_DEFAULTS["max_size_mb"]) or CONFIG_DEFAULTS["max_size_mb"])
        self._set_value_row_text("max_size_mb", max_size)

        chunk_size = str(self.temp_settings.get("chunk_size", DEFAULT_CHUNK_SIZE) or DEFAULT_CHUNK_SIZE)
        self._set_value_row_text("chunk_size", chunk_size)

        remote_download_batch_size = str(
            self.temp_settings.get("remote_download_batch_size", CONFIG_DEFAULTS["remote_download_batch_size"])
            or CONFIG_DEFAULTS["remote_download_batch_size"]
        )
        self._set_value_row_text("remote_download_batch_size", remote_download_batch_size)

        overlay_opacity = str(self.temp_settings.get("overlay_opacity", CONFIG_DEFAULTS["overlay_opacity"]) or CONFIG_DEFAULTS["overlay_opacity"])
        self._set_value_row_text("overlay_opacity", f"{overlay_opacity}%")

        overlay_position = self.plugin._coerce_index(self.temp_settings.get("overlay_position", 0), 0, 5, 0)
        self._set_value_row_text("overlay_position", self._overlay_position_options()[overlay_position])

    def _performance_profiles(self):
        return [
            {
                "id": "stable",
                "label": self.plugin._t("profile_stable"),
                "description": self.plugin._t("profile_stable_desc"),
                "values": {
                    "use_chunked_mode": True,
                    "chunk_size": 500,
                    "remote_download_batch_size": 32,
                    "allow_large_chat_remote_downloads": False,
                },
            },
            {
                "id": "balanced",
                "label": self.plugin._t("profile_balanced"),
                "description": self.plugin._t("profile_balanced_desc"),
                "values": {
                    "use_chunked_mode": True,
                    "chunk_size": 750,
                    "remote_download_batch_size": 48,
                    "allow_large_chat_remote_downloads": False,
                },
            },
            {
                "id": "aggressive",
                "label": self.plugin._t("profile_aggressive"),
                "description": self.plugin._t("profile_aggressive_desc"),
                "values": {
                    "use_chunked_mode": True,
                    "chunk_size": 1000,
                    "remote_download_batch_size": 96,
                    "allow_large_chat_remote_downloads": True,
                },
            },
        ]

    def _profile_signature(self, values: Dict[str, Any]):
        return (
            bool(values.get("use_chunked_mode")),
            max(50, self.plugin._parse_positive_int(values.get("chunk_size"), DEFAULT_CHUNK_SIZE)),
            max(
                25,
                min(
                    200,
                    self.plugin._parse_positive_int(
                        values.get("remote_download_batch_size"),
                        CONFIG_DEFAULTS["remote_download_batch_size"],
                    ),
                ),
            ),
            bool(values.get("allow_large_chat_remote_downloads")),
        )

    def _profile_index_for_settings(self, settings: Dict[str, Any]) -> int:
        current_signature = self._profile_signature(settings)
        for index, profile in enumerate(self._performance_profiles()):
            if current_signature == self._profile_signature(profile["values"]):
                return index
        return -1

    def _profile_label_for_settings(self, settings: Dict[str, Any]) -> str:
        index = self._profile_index_for_settings(settings)
        if index < 0:
            return self.plugin._t("profile_custom")
        return self._performance_profiles()[index]["label"]

    def _apply_profile(self, profile: Dict[str, Any]):
        values = profile.get("values") if isinstance(profile, dict) else {}
        if not isinstance(values, dict):
            values = {}
        for key, value in values.items():
            self.temp_settings[key] = value
        self._update_dynamic_state()

    def _open_profile_picker_dialog(self):
        profiles = self._performance_profiles()
        if not profiles:
            return

        selected_index = self._profile_index_for_settings(self.temp_settings)
        if selected_index < 0:
            selected_index = 1 if len(profiles) > 1 else 0
        selected_state = {"index": selected_index}

        container = LinearLayout(self.context)
        container.setOrientation(LinearLayout.VERTICAL)
        container.setPadding(
            AndroidUtilities.dp(10),
            AndroidUtilities.dp(6),
            AndroidUtilities.dp(10),
            AndroidUtilities.dp(6),
        )

        intro = TextView(self.context)
        intro.setText(self.plugin._t("profile_picker_desc"))
        intro.setTextColor(self._color(Theme.key_dialogTextGray3))
        intro.setTextSize(TypedValue.COMPLEX_UNIT_DIP, 13)
        intro.setLineSpacing(AndroidUtilities.dp(2), 1.0)
        intro_params = LinearLayout.LayoutParams(
            LinearLayout.LayoutParams.MATCH_PARENT,
            LinearLayout.LayoutParams.WRAP_CONTENT,
        )
        intro_params.bottomMargin = AndroidUtilities.dp(12)
        container.addView(intro, intro_params)

        preview_card = LinearLayout(self.context)
        preview_card.setOrientation(LinearLayout.VERTICAL)
        preview_card.setPadding(
            AndroidUtilities.dp(16),
            AndroidUtilities.dp(16),
            AndroidUtilities.dp(16),
            AndroidUtilities.dp(14),
        )
        preview_card.setBackground(self._surface_background(radius_dp=22, stroke_alpha=0.08))
        container.addView(preview_card)

        profile_title = TextView(self.context)
        profile_title.setTextColor(self._color(Theme.key_dialogTextBlack))
        profile_title.setTextSize(TypedValue.COMPLEX_UNIT_DIP, 17)
        profile_title.setTypeface(Typeface.DEFAULT_BOLD)
        preview_card.addView(profile_title)

        profile_desc = TextView(self.context)
        profile_desc.setTextColor(self._color(Theme.key_dialogTextGray3))
        profile_desc.setTextSize(TypedValue.COMPLEX_UNIT_DIP, 13)
        profile_desc.setLineSpacing(AndroidUtilities.dp(2), 1.0)
        desc_params = LinearLayout.LayoutParams(
            LinearLayout.LayoutParams.MATCH_PARENT,
            LinearLayout.LayoutParams.WRAP_CONTENT,
        )
        desc_params.topMargin = AndroidUtilities.dp(4)
        preview_card.addView(profile_desc, desc_params)

        chips_row = LinearLayout(self.context)
        chips_row.setOrientation(LinearLayout.HORIZONTAL)
        chips_row.setGravity(Gravity.LEFT | Gravity.CENTER_VERTICAL)
        chips_params = LinearLayout.LayoutParams(
            LinearLayout.LayoutParams.MATCH_PARENT,
            LinearLayout.LayoutParams.WRAP_CONTENT,
        )
        chips_params.topMargin = AndroidUtilities.dp(14)
        preview_card.addView(chips_row, chips_params)

        chunk_chip = self._create_summary_chip()
        media_chip = self._create_summary_chip()
        safe_chip = self._create_summary_chip()
        for index, chip in enumerate((chunk_chip, media_chip, safe_chip)):
            params = LinearLayout.LayoutParams(0, LinearLayout.LayoutParams.WRAP_CONTENT, 1.0)
            if index > 0:
                params.leftMargin = AndroidUtilities.dp(8)
            chips_row.addView(chip, params)

        selector = SlideChooseView(self.context, self.resources_provider)
        selector.setOptions(
            int(selected_index),
            profiles[0]["label"],
            profiles[1]["label"],
            profiles[2]["label"],
        )
        selector_params = LinearLayout.LayoutParams(
            LinearLayout.LayoutParams.MATCH_PARENT,
            LinearLayout.LayoutParams.WRAP_CONTENT,
        )
        selector_params.topMargin = AndroidUtilities.dp(14)
        container.addView(selector, selector_params)

        def refresh_preview():
            profile = profiles[int(selected_state["index"])]
            values = profile["values"]
            profile_title.setText(profile["label"])
            profile_desc.setText(profile["description"])
            chunk_chip.setText(f"{self.plugin._t('profile_chip_chunk')}: {values['chunk_size']}")
            media_chip.setText(f"{self.plugin._t('profile_chip_media')}: {values['remote_download_batch_size']}")
            safe_label = self.plugin._t("profile_safe_on") if values.get("allow_large_chat_remote_downloads") else self.plugin._t("profile_safe_off")
            safe_chip.setText(f"{self.plugin._t('profile_chip_safe')}: {safe_label}")

        callback_cls = jclass("org.telegram.ui.Components.SlideChooseView$Callback")
        owner = self

        class _ProfileCallback(dynamic_proxy(callback_cls)):
            def onOptionSelected(self, index):
                selected_state["index"] = max(0, min(int(index), len(profiles) - 1))
                refresh_preview()

            def onTouchEnd(self):
                return None

        selector.setCallback(_ProfileCallback())
        refresh_preview()

        builder = AlertDialogBuilder(self.context, AlertDialogBuilder.ALERT_TYPE_MESSAGE, self.resources_provider)
        builder.set_title(self.plugin._t("profile_picker_title"))
        builder.set_view(container)
        try:
            builder.set_blurred_background(True)
        except Exception:
            pass

        def on_apply(builder_obj, _which):
            self._apply_profile(profiles[int(selected_state["index"])])
            try:
                builder_obj.dismiss()
            except Exception:
                pass

        builder.set_positive_button(self.plugin._t("profile_apply"), on_apply)
        builder.set_negative_button(self.plugin._t("cancel"), lambda b, _w: b.dismiss())
        builder.show()

    def _show_selector_dialog(self, key, text, options):
        def on_item_selected(builder, which):
            self.temp_settings[key] = int(which)
            builder.dismiss()
            self._update_dynamic_state()

        builder = AlertDialogBuilder(self.context, AlertDialogBuilder.ALERT_TYPE_MESSAGE, self.resources_provider)
        builder.set_title(text)
        builder.set_items(options, on_item_selected)
        builder.set_negative_button(self.plugin._t("cancel"), lambda b, w: b.dismiss())
        builder.show()

    def _show_input_dialog(self, key, text, hint, input_type):
        edit_text = EditTextBoldCursor(self.context)
        edit_text.setText(str(self.temp_settings.get(key, "")))
        edit_text.setInputType(input_type)
        edit_text.setHint(hint)
        edit_text.setSingleLine(True)
        try:
            edit_text.setTextColor(self._color(Theme.key_dialogTextBlack))
            edit_text.setHintTextColor(self._color(Theme.key_dialogTextGray3))
            edit_text.setLineColors(
                self._color(Theme.key_dialogInputField),
                self._color(Theme.key_dialogInputFieldActivated),
                self._color(Theme.key_text_RedBold),
            )
            edit_text.setBackground(ColorDrawable(0))
            edit_text.setBackgroundTintList(ColorStateList.valueOf(Color.TRANSPARENT))
        except Exception:
            pass

        edit_text.setPadding(
            AndroidUtilities.dp(20),
            AndroidUtilities.dp(18),
            AndroidUtilities.dp(20),
            AndroidUtilities.dp(18),
        )

        container = LinearLayout(self.context)
        container.setOrientation(LinearLayout.VERTICAL)
        container.setPadding(
            AndroidUtilities.dp(10),
            AndroidUtilities.dp(6),
            AndroidUtilities.dp(10),
            AndroidUtilities.dp(6),
        )
        container.addView(edit_text)

        def on_ok_click(builder, which):
            self.temp_settings[key] = str(edit_text.getText())
            builder.dismiss()
            self._update_dynamic_state()

        builder = AlertDialogBuilder(self.context, AlertDialogBuilder.ALERT_TYPE_MESSAGE, self.resources_provider)
        builder.set_title(text)
        builder.set_view(container)
        builder.set_positive_button("OK", on_ok_click)
        builder.set_negative_button(self.plugin._t("cancel"), lambda b, w: b.dismiss())
        builder.show()

    def _create_action_bar(self):
        container = LinearLayout(self.context)
        container.setOrientation(LinearLayout.VERTICAL)
        container.setBackgroundColor(self._color(Theme.key_dialogBackground))

        divider = View(self.context)
        divider.setBackgroundColor(self._alpha(self._color(Theme.key_divider), 0.95))
        container.addView(
            divider,
            LinearLayout.LayoutParams(
                LinearLayout.LayoutParams.MATCH_PARENT,
                1,
            ),
        )

        button_stack = LinearLayout(self.context)
        button_stack.setOrientation(LinearLayout.VERTICAL)
        button_stack.setPadding(
            AndroidUtilities.dp(16),
            AndroidUtilities.dp(14),
            AndroidUtilities.dp(16),
            AndroidUtilities.dp(16),
        )

        export_btn = ButtonWithCounterView(self.context, self.resources_provider).setRound()
        export_btn.setText(self.plugin._t("export_chat"), False)
        export_btn.setOnClickListener(OnClickListener(lambda *_: self._on_export_click()))
        button_stack.addView(
            export_btn,
            LinearLayout.LayoutParams(
                LinearLayout.LayoutParams.MATCH_PARENT,
                AndroidUtilities.dp(50),
            ),
        )

        save_btn = ButtonWithCounterView(self.context, self.resources_provider).setRound().setNeutral()
        save_btn.setText(self.plugin._t("default_and_export"), False)
        save_btn.setSubText(self.plugin._t("default_and_export_subtitle"), False)
        save_btn.setOnClickListener(OnClickListener(lambda *_: self._on_save_default_click()))
        save_params = LinearLayout.LayoutParams(
            LinearLayout.LayoutParams.MATCH_PARENT,
            AndroidUtilities.dp(58),
        )
        save_params.topMargin = AndroidUtilities.dp(10)
        button_stack.addView(save_btn, save_params)

        container.addView(button_stack)
        return container

    def _on_export_click(self):
        self.dismiss()
        self.plugin._start_export_with_settings(self.dialog_id, self.temp_settings, self.account_id)

    def _on_save_default_click(self):
        for key, value in self.temp_settings.items():
            self.plugin.set_setting(key, value)
        self.dismiss()
        self.plugin._start_export_with_settings(self.dialog_id, self.temp_settings, self.account_id)
        run_on_ui_thread(lambda: BulletinHelper.show_success(self.plugin._t("settings_saved")))


class FloatingProgressOverlay:
    def __init__(self, context, enable_stroke=True, enable_animation=True, opacity=1.0):
        self.context = context
        self.progress = 0.0
        self.progress_title = ""
        self.progress_details = ""
        self.is_minimized = False
        self.stroke_animator = None
        self.stroke_update_runnable = None
        self.stroke_restart_runnable = None
        self.current_stroke_color = int(Theme.getColor(Theme.key_featuredStickers_addButton))
        self.enable_stroke = enable_stroke
        self.enable_animation = enable_animation
        self.opacity = max(0.0, min(1.0, opacity))
        self.expanded_corner_dp = 24
        self.expanded_padding_dp = (8, 8, 12, 8)
        self.minimized_padding_dp = 6
        self.progress_badge_size_dp = 44
        self.minimized_size_dp = 60
        self.anchor_gravity = Gravity.TOP | Gravity.CENTER_HORIZONTAL
        self.dismissing = False

        from android.util import DisplayMetrics
        from android.text import TextUtils

        display_metrics = DisplayMetrics()
        context.getWindowManager().getDefaultDisplay().getMetrics(display_metrics)
        self.screen_width = display_metrics.widthPixels

        self.container = FrameLayout(context)
        self._apply_background(False)
        self.container.setElevation(AndroidUtilities.dp(10))
        self.container.setAlpha(self.opacity)
        self._set_container_padding(False)

        self.progress_view = self._create_progress_badge()
        self.progress_view_params = FrameLayout.LayoutParams(
            AndroidUtilities.dp(self.progress_badge_size_dp),
            AndroidUtilities.dp(self.progress_badge_size_dp),
            Gravity.START | Gravity.CENTER_VERTICAL,
        )
        self.container.addView(
            self.progress_view,
            self.progress_view_params,
        )

        self.text_container = LinearLayout(context)
        self.text_container.setOrientation(LinearLayout.VERTICAL)
        self.text_container.setPadding(AndroidUtilities.dp(12), 0, 0, 0)

        self.title_view = TextView(context)
        self.title_view.setTextColor(Theme.getColor(Theme.key_undo_infoColor))
        self.title_view.setTextSize(TypedValue.COMPLEX_UNIT_DIP, 14.5)
        self.title_view.setTypeface(Typeface.DEFAULT_BOLD)
        self.title_view.setSingleLine(True)
        self.title_view.setText("Экспорт...")
        self.title_view.setEllipsize(TextUtils.TruncateAt.END)
        self.title_view.setMaxWidth(self.screen_width - AndroidUtilities.dp(152))

        self.details_view = TextView(context)
        self.details_view.setTextColor(Theme.getColor(Theme.key_undo_infoColor))
        self.details_view.setTextSize(TypedValue.COMPLEX_UNIT_DIP, 11.8)
        self.details_view.setSingleLine(True)
        self.details_view.setEllipsize(TextUtils.TruncateAt.END)
        self.details_view.setMaxWidth(self.screen_width - AndroidUtilities.dp(152))
        self.details_view.setAlpha(0.78)
        self.details_view.setVisibility(View.GONE)

        self.text_container.addView(self.title_view)
        self.text_container.addView(self.details_view)

        text_params = FrameLayout.LayoutParams(
            ViewGroup.LayoutParams.WRAP_CONTENT,
            ViewGroup.LayoutParams.WRAP_CONTENT,
        )
        text_params.gravity = Gravity.START | Gravity.CENTER_VERTICAL
        text_params.leftMargin = AndroidUtilities.dp(44)
        text_params.rightMargin = AndroidUtilities.dp(8)
        self.text_container_params = text_params
        self.container.addView(self.text_container, text_params)

    def _set_interpolator(self, animator, class_name: str, *args) -> bool:
        try:
            interpolator = jclass(f"android.view.animation.{class_name}")(*args)
            animator.setInterpolator(interpolator)
            return True
        except Exception:
            return False

    def _resolve_motion_offsets(self, gravity=None):
        try:
            gravity_value = int(self.anchor_gravity if gravity is None else gravity)
        except Exception:
            gravity_value = int(Gravity.TOP | Gravity.CENTER_HORIZONTAL)

        offset_x = 0
        offset_y = -AndroidUtilities.dp(18)

        if gravity_value & Gravity.BOTTOM:
            offset_y = AndroidUtilities.dp(18)
        elif gravity_value & Gravity.TOP:
            offset_y = -AndroidUtilities.dp(18)

        return offset_x, offset_y

    def _animate_progress_bump(self, progress_delta: float):
        if self.dismissing or self.container.getParent() is None:
            return

        self.progress_view.animate().cancel()
        self.progress_percent_view.animate().cancel()
        self.progress_view.setTranslationY(0)
        target_scale = 1.045 if progress_delta < 0.06 else 1.07
        container_scale = 1.01 if self.is_minimized else 1.015
        lift_anim = self.progress_view.animate().scaleX(target_scale).scaleY(target_scale).translationY(
            -AndroidUtilities.dp(2 if not self.is_minimized else 1)
        ).setDuration(120)
        self._set_interpolator(lift_anim, "DecelerateInterpolator", 1.3)

        shell_anim = self.container.animate().scaleX(container_scale).scaleY(container_scale).translationY(-AndroidUtilities.dp(1)).setDuration(120)
        self._set_interpolator(shell_anim, "DecelerateInterpolator", 1.15)
        shell_anim.start()

        percent_anim = self.progress_percent_view.animate().scaleX(1.06).scaleY(1.06).setDuration(120)
        self._set_interpolator(percent_anim, "DecelerateInterpolator", 1.2)
        percent_anim.start()

        def settle():
            settle_anim = self.progress_view.animate().scaleX(1.0).scaleY(1.0).translationY(0).setDuration(260)
            if not self._set_interpolator(settle_anim, "OvershootInterpolator", 0.85):
                try:
                    settle_anim.setInterpolator(AndroidUtilities.decelerateInterpolator)
                except Exception:
                    pass
            settle_anim.start()

            shell_settle = self.container.animate().scaleX(1.0).scaleY(1.0).translationY(0).setDuration(280)
            if not self._set_interpolator(shell_settle, "OvershootInterpolator", 0.8):
                try:
                    shell_settle.setInterpolator(AndroidUtilities.decelerateInterpolator)
                except Exception:
                    pass
            shell_settle.start()

            percent_settle = self.progress_percent_view.animate().scaleX(1.0).scaleY(1.0).setDuration(240)
            if not self._set_interpolator(percent_settle, "OvershootInterpolator", 0.75):
                try:
                    percent_settle.setInterpolator(AndroidUtilities.decelerateInterpolator)
                except Exception:
                    pass
            percent_settle.start()

        lift_anim.withEndAction(Runnable(settle)).start()

    def _set_container_padding(self, minimized: bool):
        if minimized:
            padding = (self.minimized_padding_dp,) * 4
        else:
            padding = self.expanded_padding_dp
        self.container.setPadding(
            AndroidUtilities.dp(padding[0]),
            AndroidUtilities.dp(padding[1]),
            AndroidUtilities.dp(padding[2]),
            AndroidUtilities.dp(padding[3]),
        )

    def _apply_layout_mode(self, minimized: bool):
        self._apply_background(minimized)
        self._set_container_padding(minimized)

        progress_params = self.progress_view.getLayoutParams()
        if progress_params:
            progress_params.width = AndroidUtilities.dp(self.progress_badge_size_dp)
            progress_params.height = AndroidUtilities.dp(self.progress_badge_size_dp)
            progress_params.gravity = Gravity.CENTER if minimized else (Gravity.START | Gravity.CENTER_VERTICAL)
            self.progress_view.setLayoutParams(progress_params)

        params = self.container.getLayoutParams()
        if params:
            if minimized:
                params.width = AndroidUtilities.dp(self.minimized_size_dp)
                params.height = AndroidUtilities.dp(self.minimized_size_dp)
            else:
                params.width = ViewGroup.LayoutParams.WRAP_CONTENT
                params.height = ViewGroup.LayoutParams.WRAP_CONTENT
            self.container.setLayoutParams(params)

    def _apply_background(self, minimized: bool):
        background = GradientDrawable()
        if minimized:
            background.setShape(GradientDrawable.OVAL)
        else:
            background.setCornerRadius(AndroidUtilities.dp(self.expanded_corner_dp))
        background.setColor(Theme.getColor(Theme.key_undo_background))
        if self.enable_stroke:
            background.setStroke(AndroidUtilities.dp(2), self.current_stroke_color)
        self.container.setBackground(background)

    def _create_progress_badge(self):
        from org.telegram.ui.Components import RadialProgressView

        progress_indicator = RadialProgressView(self.context)
        progress_indicator.setProgressColor(Theme.getColor(Theme.key_featuredStickers_addButton))
        progress_indicator.setSize(AndroidUtilities.dp(34))
        self.progress_indicator = progress_indicator

        progress_container = FrameLayout(self.context)
        badge_bg = GradientDrawable()
        badge_bg.setShape(GradientDrawable.OVAL)
        badge_bg.setColor(_android_color_int((42 << 24) | (Theme.getColor(Theme.key_featuredStickers_addButton) & 0x00FFFFFF)))
        progress_container.setBackground(badge_bg)
        progress_container.addView(
            progress_indicator,
            FrameLayout.LayoutParams(
                ViewGroup.LayoutParams.MATCH_PARENT,
                ViewGroup.LayoutParams.MATCH_PARENT,
                Gravity.CENTER,
            ),
        )

        self.progress_percent_view = TextView(self.context)
        self.progress_percent_view.setTextColor(Theme.getColor(Theme.key_undo_infoColor))
        self.progress_percent_view.setTextSize(TypedValue.COMPLEX_UNIT_DIP, 10.5)
        self.progress_percent_view.setTypeface(Typeface.DEFAULT_BOLD)
        self.progress_percent_view.setGravity(Gravity.CENTER)
        self.progress_percent_view.setSingleLine(True)
        self.progress_percent_view.setIncludeFontPadding(False)
        progress_container.addView(
            self.progress_percent_view,
            FrameLayout.LayoutParams(
                ViewGroup.LayoutParams.MATCH_PARENT,
                ViewGroup.LayoutParams.MATCH_PARENT,
                Gravity.CENTER,
            ),
        )
        return progress_container

    def set_progress(self, progress, title, details=""):
        previous_progress = self.progress
        self.progress = max(0.0, min(1.0, float(progress or 0.0)))
        self.progress_title = str(title or "")
        self.progress_details = str(details or "")
        self.title_view.setText(self.progress_title)
        self.details_view.setText(self.progress_details)
        self.details_view.setVisibility(View.VISIBLE if self.progress_details else View.GONE)
        percent_value = int(round(self.progress * 100.0))
        self.progress_percent_view.setTextSize(
            TypedValue.COMPLEX_UNIT_DIP,
            9.2 if percent_value >= 100 else 10.5,
        )
        self.progress_percent_view.setText(f"{percent_value}%")
        try:
            if hasattr(self.progress_indicator, "setProgress"):
                self.progress_indicator.setProgress(self.progress)
        except Exception:
            pass

        progress_delta = self.progress - previous_progress
        if progress_delta >= 0.02:
            self._animate_progress_bump(progress_delta)

    def get_view(self):
        return self.container

    def _cancel_toggle_animations(self):
        for view in (self.container, self.text_container, self.progress_view, self.progress_percent_view):
            try:
                view.animate().cancel()
            except Exception:
                pass

    def _restart_stroke_animation(self, delay_ms=0):
        if not (self.enable_stroke and self.enable_animation):
            return
        if self.stroke_animator or self.stroke_restart_runnable:
            return
        if delay_ms <= 0:
            self._start_stroke_animation()
            return

        def delayed_start():
            if self.stroke_restart_runnable is runnable:
                self.stroke_restart_runnable = None
            if not self.stroke_animator:
                self._start_stroke_animation()

        runnable = Runnable(delayed_start)
        self.stroke_restart_runnable = runnable
        self.container.postDelayed(runnable, int(delay_ms))

    def toggle_minimize(self):
        if self.dismissing:
            return
        self._cancel_toggle_animations()
        self.is_minimized = not self.is_minimized
        if self.is_minimized:
            anticipate_anim = self.container.animate().scaleX(1.025).scaleY(1.025).translationY(-AndroidUtilities.dp(2)).setDuration(90)
            self._set_interpolator(anticipate_anim, "DecelerateInterpolator", 1.0)
            anticipate_anim.start()

            def hide_text():
                self.text_container.setVisibility(View.GONE)
                self.text_container.setTranslationX(0)
                self.text_container.setTranslationY(0)
                self._apply_layout_mode(True)
                self.container.setScaleX(0.8)
                self.container.setScaleY(0.8)
                self.container.setTranslationY(-AndroidUtilities.dp(8))
                self.progress_view.setScaleX(0.84)
                self.progress_view.setScaleY(0.84)
                self.progress_percent_view.setScaleX(0.9)
                self.progress_percent_view.setScaleY(0.9)

                settle_anim = self.container.animate().scaleX(1.0).scaleY(1.0).translationY(0).setDuration(340)
                if not self._set_interpolator(settle_anim, "OvershootInterpolator", 1.1):
                    try:
                        settle_anim.setInterpolator(AndroidUtilities.decelerateInterpolator)
                    except Exception:
                        pass
                settle_anim.start()

                badge_settle = self.progress_view.animate().scaleX(1.0).scaleY(1.0).setDuration(320)
                if not self._set_interpolator(badge_settle, "OvershootInterpolator", 1.05):
                    try:
                        badge_settle.setInterpolator(AndroidUtilities.decelerateInterpolator)
                    except Exception:
                        pass
                badge_settle.start()

                percent_settle = self.progress_percent_view.animate().scaleX(1.0).scaleY(1.0).setDuration(280)
                if not self._set_interpolator(percent_settle, "OvershootInterpolator", 0.95):
                    try:
                        percent_settle.setInterpolator(AndroidUtilities.decelerateInterpolator)
                    except Exception:
                        pass
                percent_settle.start()

            text_anim = self.text_container.animate().alpha(0.0).translationX(-AndroidUtilities.dp(10)).translationY(-AndroidUtilities.dp(4)).setDuration(135)
            self._set_interpolator(text_anim, "AccelerateInterpolator", 0.95)
            text_anim.withEndAction(Runnable(hide_text)).start()
            self._restart_stroke_animation(190)
            return

        self._apply_layout_mode(False)
        self.container.setScaleX(0.84)
        self.container.setScaleY(0.84)
        self.container.setTranslationY(AndroidUtilities.dp(10))
        self.progress_view.setScaleX(0.86)
        self.progress_view.setScaleY(0.86)
        self.progress_percent_view.setScaleX(0.92)
        self.progress_percent_view.setScaleY(0.92)
        self.text_container.setVisibility(View.VISIBLE)
        self.text_container.setAlpha(0.0)
        self.text_container.setTranslationX(-AndroidUtilities.dp(6))
        self.text_container.setTranslationY(AndroidUtilities.dp(8))

        text_anim = self.text_container.animate().alpha(1.0).translationX(0).translationY(0).setDuration(240)
        self._set_interpolator(text_anim, "DecelerateInterpolator", 1.15)
        text_anim.start()

        settle_anim = self.container.animate().scaleX(1.0).scaleY(1.0).translationY(0).setDuration(360)
        if not self._set_interpolator(settle_anim, "OvershootInterpolator", 1.05):
            try:
                settle_anim.setInterpolator(AndroidUtilities.decelerateInterpolator)
            except Exception:
                pass
        settle_anim.start()

        badge_settle = self.progress_view.animate().scaleX(1.0).scaleY(1.0).setDuration(320)
        if not self._set_interpolator(badge_settle, "OvershootInterpolator", 1.0):
            try:
                badge_settle.setInterpolator(AndroidUtilities.decelerateInterpolator)
            except Exception:
                pass
        badge_settle.start()

        percent_settle = self.progress_percent_view.animate().scaleX(1.0).scaleY(1.0).setDuration(280)
        if not self._set_interpolator(percent_settle, "OvershootInterpolator", 0.9):
            try:
                percent_settle.setInterpolator(AndroidUtilities.decelerateInterpolator)
            except Exception:
                pass
        percent_settle.start()
        self._restart_stroke_animation(220)

    def show_with_animation(self, gravity=None):
        self.dismissing = False
        if gravity is not None:
            self.anchor_gravity = int(gravity)

        self.container.animate().cancel()
        self.text_container.animate().cancel()
        self.progress_view.animate().cancel()
        self.progress_percent_view.animate().cancel()

        offset_x, offset_y = self._resolve_motion_offsets(self.anchor_gravity)
        self.container.setAlpha(0.0)
        self.container.setScaleX(0.92)
        self.container.setScaleY(0.92)
        self.container.setTranslationX(offset_x)
        self.container.setTranslationY(offset_y)

        if not self.is_minimized:
            self.text_container.setAlpha(0.0)
            self.text_container.setTranslationX(0)
            self.text_container.setTranslationY(AndroidUtilities.dp(8))

        self.progress_view.setScaleX(0.88)
        self.progress_view.setScaleY(0.88)
        self.progress_percent_view.setScaleX(0.94)
        self.progress_percent_view.setScaleY(0.94)

        show_anim = self.container.animate().alpha(self.opacity).scaleX(1.0).scaleY(1.0).translationX(0).translationY(0).setDuration(420)
        if not self._set_interpolator(show_anim, "OvershootInterpolator", 0.9):
            try:
                show_anim.setInterpolator(AndroidUtilities.decelerateInterpolator)
            except Exception:
                pass
        show_anim.start()

        badge_anim = self.progress_view.animate().scaleX(1.0).scaleY(1.0).setDuration(320)
        if not self._set_interpolator(badge_anim, "OvershootInterpolator", 0.95):
            try:
                badge_anim.setInterpolator(AndroidUtilities.decelerateInterpolator)
            except Exception:
                pass
        badge_anim.start()

        percent_anim = self.progress_percent_view.animate().scaleX(1.0).scaleY(1.0).setDuration(260)
        if not self._set_interpolator(percent_anim, "OvershootInterpolator", 0.85):
            try:
                percent_anim.setInterpolator(AndroidUtilities.decelerateInterpolator)
            except Exception:
                pass
        percent_anim.start()

        if not self.is_minimized:
            text_anim = self.text_container.animate().alpha(1.0).translationY(0).setDuration(260)
            self._set_interpolator(text_anim, "DecelerateInterpolator", 1.15)
            text_anim.start()

        self._restart_stroke_animation(110)

    def dismiss_with_animation(self, on_complete=None):
        if self.dismissing:
            return
        self.dismissing = True
        self._cancel_toggle_animations()
        self._stop_stroke_animation()

        if self.container.getParent() is None:
            if callable(on_complete):
                on_complete()
            return

        offset_x, offset_y = self._resolve_motion_offsets(self.anchor_gravity)

        if self.text_container.getVisibility() == View.VISIBLE:
            text_anim = self.text_container.animate().alpha(0.0).translationY(-AndroidUtilities.dp(4)).setDuration(110)
            self._set_interpolator(text_anim, "AccelerateInterpolator", 0.95)
            text_anim.start()

        self.progress_view.animate().scaleX(0.96).scaleY(0.96).setDuration(120).start()
        dismiss_anim = self.container.animate().alpha(0.0).scaleX(0.96).scaleY(0.96).translationX(int(offset_x * 0.2)).translationY(
            int(offset_y * 0.4)
        ).setDuration(180)
        self._set_interpolator(dismiss_anim, "AccelerateInterpolator", 1.05)

        def finish():
            self.text_container.setTranslationY(0)
            if callable(on_complete):
                on_complete()

        dismiss_anim.withEndAction(Runnable(finish)).start()

    def _start_stroke_animation(self):
        if self.stroke_animator:
            return
        try:
            from android.animation import ArgbEvaluator, ValueAnimator
            from android.view.animation import LinearInterpolator
            from java.lang import Integer

            colors = [
                Integer(int(Theme.getColor(Theme.key_featuredStickers_addButton))),
                Integer(int(Theme.getColor(Theme.key_chat_messagePanelSend))),
                Integer(int(Theme.getColor(Theme.key_avatar_nameInMessageBlue))),
                Integer(int(Theme.getColor(Theme.key_chat_messagePanelSend))),
                Integer(int(Theme.getColor(Theme.key_featuredStickers_addButton))),
            ]

            self.stroke_animator = ValueAnimator.ofObject(ArgbEvaluator(), colors)
            self.stroke_animator.setDuration(4200)
            self.stroke_animator.setRepeatCount(ValueAnimator.INFINITE)
            self.stroke_animator.setRepeatMode(ValueAnimator.RESTART)
            self.stroke_animator.setInterpolator(LinearInterpolator())

            overlay_ref = self

            def update_stroke_color():
                try:
                    if not overlay_ref.stroke_animator:
                        return
                    animated_value = overlay_ref.stroke_animator.getAnimatedValue()
                    if animated_value is not None:
                        color_int = int(animated_value.intValue()) if hasattr(animated_value, "intValue") else int(animated_value)
                        overlay_ref.current_stroke_color = color_int
                        background = overlay_ref.container.getBackground()
                        if isinstance(background, GradientDrawable):
                            background.setStroke(AndroidUtilities.dp(2), color_int)
                        overlay_ref.container.invalidate()
                    if overlay_ref.stroke_animator:
                        overlay_ref.stroke_update_runnable = Runnable(update_stroke_color)
                        overlay_ref.container.postDelayed(overlay_ref.stroke_update_runnable, 20)
                except Exception as error:
                    log(f"[ChatExport] Ошибка обновления анимации оверлея: {error}")

            self.stroke_animator.start()
            self.stroke_update_runnable = Runnable(update_stroke_color)
            self.container.postDelayed(self.stroke_update_runnable, 20)
        except Exception as error:
            log(f"[ChatExport] Ошибка запуска анимации оверлея: {error}")

    def _stop_stroke_animation(self):
        try:
            if self.stroke_restart_runnable:
                try:
                    self.container.removeCallbacks(self.stroke_restart_runnable)
                except Exception:
                    pass
                self.stroke_restart_runnable = None
            if self.stroke_update_runnable:
                try:
                    self.container.removeCallbacks(self.stroke_update_runnable)
                except Exception:
                    pass
                self.stroke_update_runnable = None
            if self.stroke_animator:
                self.stroke_animator.cancel()
                self.stroke_animator = None
        except Exception as error:
            log(f"[ChatExport] Ошибка остановки анимации оверлея: {error}")

    def setOnClickListener(self, listener):
        self.container.setOnClickListener(listener)

    def setOnLongClickListener(self, listener):
        self.container.setOnLongClickListener(listener)

    def setClickable(self, clickable):
        self.container.setClickable(clickable)

    def setFocusable(self, focusable):
        self.container.setFocusable(focusable)

    def setBackground(self, background):
        self.container.setBackground(background)

    def getBackground(self):
        return self.container.getBackground()


class ChatExportPlugin(BasePlugin):
    def __init__(self):
        super().__init__()
        self.menu_item_id = None
        self.dex_loader = DexLoader(self)
        self._hooks = []
        self._download_delegate = DexDownloadDelegate(self)
        self._observer_account = None
        self._pending_file_name = None
        self._pending_install_after_download = False
        self.export_in_progress = False
        self.cancel_requested = False
        self.progress_dialog = None
        self.progress_dialog_views = None
        self.progress_overlay = None
        self.progress_overlay_container = None
        self.current_progress_phase = "idle"
        self.current_progress_current = 0
        self.current_progress_total = 0
        self.current_progress_text = ""
        self.current_progress_details = ""
        self.current_progress_value = 0.0
        self.current_source_mode = ""
        self.current_auto_large_chat_mode = False
        self.temp_export_settings = None
        self.background_notification_enabled = False
        self._job_lock = threading.RLock()
        self._job_token = 0
        self._shutting_down = False
        self._pending_share_result = None

    def _log(self, message: str) -> None:
        log(f"[ChatExport] {message}")

    def _t(self, key: str, **kwargs) -> str:
        lang = "ru" if self._is_russian() else "en"
        text = TRANSLATIONS.get(lang, {}).get(key, TRANSLATIONS["en"].get(key, key))
        return text.format(**kwargs) if kwargs else text

    def _is_russian(self) -> bool:
        try:
            locale = LocaleController.getInstance().getCurrentLocale()
            lang = locale.getLanguage() if locale else ""
            return bool(lang and str(lang).lower().startswith("ru"))
        except Exception:
            return False

    def on_plugin_load(self):
        try:
            self.menu_item_id = self.add_menu_item(
                MenuItemData(
                    menu_type=MenuItemType.CHAT_ACTION_MENU,
                    item_id="export_chat_button",
                    text=self._t("export_chat"),
                    icon="msg_download",
                    on_click=self._handle_export_click,
                )
            )
            self._hook_android_utilities()
            self._load_cached_or_local_dex()
        except Exception as error:
            self._log(f"Ошибка загрузки плагина: {error}")

    def on_plugin_unload(self):
        self._shutting_down = True
        with self._job_lock:
            self._job_token += 1
        try:
            if self.export_in_progress:
                try:
                    self.dex_loader.cancel_export()
                except Exception:
                    pass
            self._reset_pending_download_state()
            self._detach_download_observer()
            if self.menu_item_id:
                self.remove_menu_item(self.menu_item_id)
            for hook in self._hooks:
                try:
                    self.unhook_method(hook)
                except Exception:
                    pass
            self._hooks.clear()
        finally:
            self._set_background_notification_enabled(False)
            self._dismiss_progress_dialog()
            self._dismiss_progress_overlay()
            try:
                self.dex_loader.unload()
            except Exception as error:
                self._log(f"Ошибка выгрузки DEX: {error}")

    def on_app_event(self, event_type):
        try:
            from base_plugin import AppEvent

            if event_type == AppEvent.RESUME:
                if self.export_in_progress and not self.progress_dialog and not self.progress_overlay:
                    run_on_ui_thread(self._show_progress_dialog)
                elif self._pending_share_result:
                    run_on_ui_thread(self._flush_pending_share)
        except Exception as error:
            self._log(f"Ошибка обработки события приложения: {error}")

    def _load_cached_or_local_dex(self):
        if self._try_load_local_core():
            return True
        return self._try_load_cached_core()

    def _try_load_local_core(self) -> bool:
        try:
            path = self.dex_loader.install_from_local()
            self._sync_settings_to_dex()
            self._log(f"DEX загружен из локального файла: {path}")
            return True
        except FileNotFoundError:
            return False
        except Exception as error:
            self._log(f"{self._t('dex_load_failed')}: {error}")
            return False

    def _try_load_cached_core(self) -> bool:
        cached = self.dex_loader.load_from_cache()
        if not cached:
            return False
        try:
            self.dex_loader.start_from_bytes(cached)
            self._sync_settings_to_dex()
            self._log("DEX загружен из кеша")
            return True
        except Exception as error:
            self.dex_loader.clear_cache()
            self._log(f"Ошибка загрузки кешированного DEX: {error}")
            return False

    def _ensure_dex_ready(self) -> None:
        if self.dex_loader.is_loaded():
            return
        if self._try_load_local_core() or self._try_load_cached_core():
            return
        raise FileNotFoundError(self._t("dex_missing"))

    def _sync_settings_to_dex(self, override_settings: Optional[Dict[str, Any]] = None) -> None:
        if not self.dex_loader.is_loaded():
            return
        settings = self._collect_effective_settings(override_settings)
        try:
            self.dex_loader.update_settings(settings)
        except Exception as error:
            self._log(f"Ошибка синхронизации настроек с DEX: {error}")

    def _handle_export_click(self, context):
        try:
            self._perform_export_click_haptic(context)
            dialog_id = self._resolve_dialog_id(context)
            if not dialog_id:
                BulletinHelper.show_error(self._t("cannot_determine_chat"))
                return
            fragment = get_last_fragment()
            account_id = self._resolve_account_id(context=context, fragment=fragment)
            run_on_ui_thread(lambda: self._show_export_settings_dialog(dialog_id, account_id))
        except Exception as error:
            self._log(f"Ошибка открытия настроек экспорта: {error}")
            BulletinHelper.show_error(f"{self._t('export_failed')}: {error}")

    def _resolve_export_click_view(self, context):
        if context is not None and hasattr(context, "get"):
            for key in ("view", "menu_view", "anchor_view", "item_view", "clicked_view"):
                try:
                    view = context.get(key)
                    if view is not None:
                        return view
                except Exception:
                    pass
        return None

    def _resolve_feedback_context(self, view=None):
        context = None
        if view is not None:
            try:
                context = view.getContext()
            except Exception:
                context = None
        if context is None:
            try:
                fragment = get_last_fragment()
                if fragment is not None and hasattr(fragment, "getParentActivity"):
                    context = fragment.getParentActivity()
            except Exception:
                context = None
        if context is None:
            context = ApplicationLoader.applicationContext
        try:
            while context and not hasattr(context, "getSystemService"):
                if hasattr(context, "getBaseContext"):
                    context = context.getBaseContext()
                else:
                    break
        except Exception:
            pass
        return context

    def _perform_export_click_haptic(self, context=None):
        def do_feedback():
            view = self._resolve_export_click_view(context)
            try:
                if view is not None and hasattr(view, "performHapticFeedback"):
                    haptic = JClass.forName("android.view.HapticFeedbackConstants")
                    feedback_constant = getattr(haptic, "KEYBOARD_TAP", getattr(haptic, "VIRTUAL_KEY", 1))
                    if view.performHapticFeedback(feedback_constant):
                        return
            except Exception:
                pass

            try:
                feedback_context = self._resolve_feedback_context(view)
                if feedback_context is None:
                    return
                vibrator = feedback_context.getSystemService("vibrator")
                if not vibrator:
                    return
                try:
                    vibration_effect = JClass.forName("android.os.VibrationEffect")
                    vibrator.vibrate(vibration_effect.createOneShot(20, 60))
                except Exception:
                    vibrator.vibrate(20)
            except Exception:
                pass

        run_on_ui_thread(do_feedback)

    def _resolve_dialog_id(self, context) -> Optional[int]:
        dialog_id = context.get("dialog_id")
        if dialog_id:
            return int(dialog_id)
        chat = context.get("chat")
        if chat is not None and getattr(chat, "id", None):
            return -int(chat.id)
        user = context.get("user")
        if user is not None and getattr(user, "id", None):
            return int(user.id)
        return None

    def _show_export_settings_dialog(self, dialog_id, account_id: Optional[int] = None):
        fragment = get_last_fragment()
        if not fragment:
            return
        context = fragment.getParentActivity()
        if not context:
            return
        ExportSettingsBottomSheet(context, self, dialog_id, account_id).show()

    def _start_export_with_settings(self, dialog_id, temp_settings, account_id: Optional[int] = None):
        with self._job_lock:
            if self.export_in_progress:
                BulletinHelper.show_error(self._t("export_in_progress"))
                return
            self.export_in_progress = True
            self.cancel_requested = False
            self.temp_export_settings = self._collect_effective_settings(temp_settings, account_id=account_id)
            self.current_progress_phase = "loading"
            self.current_progress_current = 0
            self.current_progress_total = 0
            self.current_progress_text = self._t("progress_preparing_export")
            self.current_progress_details = self._build_progress_details("loading", 0, 0, 0)
            self.current_progress_value = 0.0
            self.current_source_mode = "api" if self.temp_export_settings.get("use_api_mode") else "local_manual"
            self.current_auto_large_chat_mode = False
            self._job_token += 1
            token = self._job_token

        self._set_background_notification_enabled(False)
        run_on_ui_thread(self._show_progress_dialog)
        threading.Thread(target=self._run_export_job, args=(token, int(dialog_id)), daemon=True).start()

    def _run_export_job(self, token: int, dialog_id: int):
        try:
            self._ensure_dex_ready()
            settings = self._collect_effective_settings(self.temp_export_settings)
            self._sync_settings_to_dex(settings)
            request_json = self._build_export_request(dialog_id, settings)
            self.dex_loader.start_export(request_json)
            self._poll_export_state(token)
        except Exception as error:
            self._finish_job_with_error(token, str(error))

    def _poll_export_state(self, token: int):
        bridge_errors = 0
        terminal_phase = ""
        terminal_seen = 0
        cancel_started_at = None
        poll_index = 0

        while self._is_job_active(token):
            if self.cancel_requested and cancel_started_at is None:
                cancel_started_at = time.time()

            try:
                state_payload = self._try_get_state_payload()
                if state_payload is not None:
                    self._apply_state_payload(state_payload)
                    phase = str(state_payload.get("phase") or "").strip().lower()
                    if self._is_terminal_phase(phase):
                        terminal_phase = phase
                        terminal_seen += 1
                    else:
                        terminal_phase = ""
                        terminal_seen = 0

                should_poll_result = (poll_index % RESULT_POLL_INTERVAL == 0) or terminal_seen > 0 or self.cancel_requested
                if should_poll_result:
                    result_payload = self._try_consume_result_payload()
                    if result_payload is not None:
                        self._finish_job_with_result(token, result_payload)
                        return

                if self.cancel_requested and cancel_started_at and time.time() - cancel_started_at > 15:
                    self._finish_job_cancelled(token)
                    return

                if terminal_seen >= 3:
                    if terminal_phase in ("cancelled", "canceled"):
                        self._finish_job_cancelled(token)
                    elif terminal_phase in ("error", "failed", "failure"):
                        self._finish_job_with_error(token, self.current_progress_text or self._t("export_failed"))
                    else:
                        self._finish_job_with_error(token, "DEX не вернул итоговый результат экспорта")
                    return

                bridge_errors = 0
            except Exception as error:
                bridge_errors += 1
                if self.cancel_requested and self._looks_like_cancel(str(error)):
                    self._finish_job_cancelled(token)
                    return
                if bridge_errors >= 3:
                    self._finish_job_with_error(token, str(error))
                    return

            poll_index += 1
            time.sleep(POLL_INTERVAL_SECONDS)

    def _is_job_active(self, token: int) -> bool:
        with self._job_lock:
            return (
                not self._shutting_down
                and self.export_in_progress
                and token == self._job_token
            )

    def _try_get_state_payload(self) -> Optional[Dict[str, Any]]:
        raw = self.dex_loader.get_export_state()
        payload = self._parse_bridge_payload(raw, "state")
        if payload == {}:
            return None
        return payload

    def _try_consume_result_payload(self) -> Optional[Dict[str, Any]]:
        raw = self.dex_loader.consume_result()
        payload = self._parse_bridge_payload(raw, "result")
        if not payload:
            return None
        return payload

    def _parse_bridge_payload(self, raw: Optional[str], kind: str) -> Optional[Dict[str, Any]]:
        if raw is None:
            return None
        text = str(raw).strip()
        if not text or text in ("null", "None"):
            return None
        try:
            value = json.loads(text)
        except Exception:
            if kind == "result":
                return {"status": "error", "message": text}
            return {"message": text}
        if value is None:
            return None
        if isinstance(value, dict):
            if kind == "result" and not value:
                return None
            return value
        if kind == "result":
            return {"status": "ok" if value else "error", "value": value}
        return {"value": value}

    def _apply_state_payload(self, state: Dict[str, Any]) -> None:
        phase = str(state.get("phase") or "").strip().lower()
        message = str(state.get("message") or "").strip()
        current = state.get("current")
        total = state.get("total")
        progress = self._state_progress_to_percent(state.get("progress"), current, total)
        source_mode = str(state.get("source_mode") or "").strip().lower()
        if source_mode:
            self.current_source_mode = source_mode
        if "auto_large_chat_mode" in state:
            self.current_auto_large_chat_mode = self._coerce_bool(state.get("auto_large_chat_mode"))
        if isinstance(self.temp_export_settings, dict):
            if source_mode == "api":
                self.temp_export_settings["use_api_mode"] = True
            elif source_mode in ("local_manual", "local_auto"):
                self.temp_export_settings["use_api_mode"] = False
            if "auto_large_chat_mode" in state:
                self.temp_export_settings["auto_large_chat_mode"] = self.current_auto_large_chat_mode

        self.current_progress_phase = phase or self.current_progress_phase
        self.current_progress_current = self._coerce_progress_count(current)
        self.current_progress_total = self._coerce_progress_count(total)
        title = self._resolve_progress_title(phase, message)
        details = self._build_progress_details(phase, current, total, progress)
        self._update_progress(title, progress, details)

    def _state_progress_to_percent(self, progress_value, current, total) -> int:
        try:
            if progress_value is not None:
                value = float(progress_value)
                return max(0, min(100, int(round(value))))
        except Exception:
            pass

        try:
            current_value = float(current)
            total_value = float(total)
            if total_value > 0:
                return max(0, min(100, int((current_value / total_value) * 100.0)))
        except Exception:
            pass
        return int(self.current_progress_value * 100)

    def _resolve_progress_title(self, phase: str, message: str) -> str:
        if phase in ("sharing", "finalizing", "finishing"):
            return self._t("preparing_share")
        if phase in ("cancelled", "canceled", "cancelling"):
            return self._t("export_cancelled")

        normalized_message = " ".join(str(message or "").strip().lower().split())
        message_map = {
            "preparing export": "progress_preparing_export",
            "fetching history from server": "progress_fetching_history",
            "opening local message database": "progress_opening_database",
            "exporting messages": "progress_exporting_messages",
            "export completed": "export_complete",
            "export cancelled": "export_cancelled",
            "cancellation requested": "progress_cancellation_requested",
        }
        mapped_key = message_map.get(normalized_message)
        if mapped_key:
            return self._t(mapped_key)

        if message:
            return message
        if phase == "loading":
            return self._t("progress_preparing_export")
        if phase == "rendering":
            return self._t("progress_exporting_messages")
        return self._t("exporting")

    def _build_progress_details(self, phase: str, current, total, progress: int = 0) -> str:
        phase = str(phase or "").strip().lower()
        parts = []

        counter_text = self._format_progress_counter(current, total)
        if counter_text:
            parts.append(counter_text)

        progress_value = int(progress or 0)
        if progress_value > 0:
            parts.append(f"{progress_value}%")

        settings = self.temp_export_settings if isinstance(self.temp_export_settings, dict) else {}
        if phase == "loading":
            source_label = self._progress_source_label(settings)
            if source_label:
                parts.append(source_label)
        elif phase == "rendering":
            format_label = self._progress_format_label(settings)
            if format_label:
                parts.append(format_label)
            media_label = self._progress_media_label(settings)
            if media_label:
                parts.append(media_label)

        return " • ".join(part for part in parts if part)

    def _format_progress_counter(self, current, total) -> str:
        current_value = self._coerce_progress_count(current)
        total_value = self._coerce_progress_count(total)

        if total_value > 0:
            return self._t(
                "progress_count_total",
                current=self._format_progress_number(current_value),
                total=self._format_progress_number(total_value),
            )
        if current_value > 0:
            return self._t(
                "progress_count_current",
                current=self._format_progress_number(current_value),
            )
        return ""

    def _coerce_progress_count(self, value) -> int:
        try:
            return max(0, int(float(value)))
        except Exception:
            return 0

    def _format_progress_number(self, value: int) -> str:
        try:
            return f"{int(value):,}".replace(",", " ")
        except Exception:
            return str(value)

    def _progress_source_label(self, settings: Dict[str, Any], source_mode: Optional[str] = None) -> str:
        normalized_mode = str(source_mode or self.current_source_mode or "").strip().lower()
        if normalized_mode == "api":
            return "API"
        if normalized_mode == "local_auto":
            return self._t("progress_source_local_auto")
        if normalized_mode == "local_manual":
            return self._t("progress_source_local")
        if self._coerce_bool(settings.get("auto_large_chat_mode", False)):
            return self._t("progress_source_local_auto")
        if bool(settings.get("use_api_mode", CONFIG_DEFAULTS["use_api_mode"])):
            return "API"
        return self._t("progress_source_local")

    def _progress_format_label(self, settings: Dict[str, Any]) -> str:
        export_format = self._coerce_index(
            settings.get("export_format", CONFIG_DEFAULTS["export_format"]),
            0,
            2,
            CONFIG_DEFAULTS["export_format"],
        )
        return ("HTML", "JSON", "TXT")[export_format]

    def _progress_media_label(self, settings: Dict[str, Any]) -> str:
        export_format = self._coerce_index(
            settings.get("export_format", CONFIG_DEFAULTS["export_format"]),
            0,
            2,
            CONFIG_DEFAULTS["export_format"],
        )
        if export_format != 0:
            return ""

        media_handling = self._coerce_index(
            settings.get("media_handling", CONFIG_DEFAULTS["media_handling"]),
            0,
            2,
            CONFIG_DEFAULTS["media_handling"],
        )
        media_keys = (
            "progress_media_skip",
            "progress_media_refs",
            "progress_media_download",
        )
        return self._t(media_keys[media_handling])

    def _is_terminal_phase(self, phase: str) -> bool:
        return phase in ("done", "completed", "success", "error", "failed", "failure", "cancelled", "canceled", "idle")

    def _finish_job_with_result(self, token: int, payload: Dict[str, Any]):
        status = str(payload.get("status") or payload.get("result") or "").strip().lower()
        success = bool(payload.get("success")) or status in ("ok", "success", "done", "completed")
        cancelled = status in ("cancelled", "canceled", "cancel", "aborted")

        if cancelled:
            self._finalize_job(token, cancelled=True)
            return

        if success or payload.get("filePath") or payload.get("file_path"):
            file_path = payload.get("filePath") or payload.get("file_path")
            mime_type = payload.get("mimeType") or payload.get("mime_type") or self._guess_mime_type(file_path)
            assets_dir_path = payload.get("assetsDirPath") or payload.get("assets_dir_path")
            effective_settings = payload.get("effective_settings") if isinstance(payload.get("effective_settings"), dict) else {}
            stats = payload.get("stats") if isinstance(payload.get("stats"), dict) else {}
            if effective_settings:
                self.current_auto_large_chat_mode = self._coerce_bool(effective_settings.get("auto_large_chat_mode"))
                self.current_source_mode = "api" if self._coerce_bool(effective_settings.get("use_api_mode")) else (
                    "local_auto" if self.current_auto_large_chat_mode else "local_manual"
                )
            if not file_path:
                self._finalize_job(token, error_message="DEX завершил экспорт без пути к файлу")
                return
            self._finalize_job(
                token,
                success_payload={
                    "file_path": str(file_path),
                    "mime_type": str(mime_type or self._guess_mime_type(file_path)),
                    "assets_dir_path": str(assets_dir_path or ""),
                    "effective_settings": effective_settings,
                    "stats": stats,
                    "source_mode": self.current_source_mode,
                },
            )
            return

        message = payload.get("message") or payload.get("error") or self._t("export_failed")
        if self.cancel_requested and self._looks_like_cancel(str(message)):
            self._finalize_job(token, cancelled=True)
            return
        self._finalize_job(token, error_message=str(message))

    def _finish_job_with_error(self, token: int, message: str):
        if self.cancel_requested and self._looks_like_cancel(str(message)):
            self._finalize_job(token, cancelled=True)
            return
        self._finalize_job(token, error_message=message)

    def _finish_job_cancelled(self, token: int):
        self._finalize_job(token, cancelled=True)

    def _finalize_job(
        self,
        token: int,
        success_payload: Optional[Dict[str, Any]] = None,
        error_message: Optional[str] = None,
        cancelled: bool = False,
    ):
        with self._job_lock:
            if token != self._job_token:
                return
            self.export_in_progress = False
            self.cancel_requested = False
            self.current_progress_phase = "idle"
            self.current_progress_current = 0
            self.current_progress_total = 0
            self.current_progress_text = ""
            self.current_progress_details = ""
            self.current_progress_value = 0.0
            self.current_source_mode = ""
            self.current_auto_large_chat_mode = False
            self.temp_export_settings = None

        self._set_background_notification_enabled(False)
        self._dismiss_progress_dialog()
        self._dismiss_progress_overlay()

        if success_payload:
            run_on_ui_thread(lambda payload=success_payload: self._share_or_queue_result(payload))
            return

        if cancelled:
            run_on_ui_thread(lambda: BulletinHelper.show_info(self._t("export_cancelled")))
            return

        message = str(error_message or self._t("export_failed"))
        run_on_ui_thread(lambda msg=message: BulletinHelper.show_error(f"{self._t('export_failed')}: {msg}"))

    def _share_or_queue_result(self, payload: Dict[str, Any]):
        self._update_progress(self._t("preparing_share"), 95)
        context = self._get_active_context()
        if not context:
            self._pending_share_result = payload
            BulletinHelper.show_info(self._t("share_pending"))
            return
        self._pending_share_result = None
        self._share_export_file(
            payload["file_path"],
            payload["mime_type"],
            payload.get("assets_dir_path"),
            payload,
        )

    def _flush_pending_share(self):
        payload = self._pending_share_result
        if not payload:
            return
        context = self._get_active_context()
        if not context:
            return
        self._pending_share_result = None
        self._share_export_file(
            payload["file_path"],
            payload["mime_type"],
            payload.get("assets_dir_path"),
            payload,
        )

    def _coerce_account_id(self, value) -> Optional[int]:
        try:
            if value is None:
                return None
            account_id = int(value)
            return account_id if account_id >= 0 else None
        except Exception:
            return None

    def _resolve_account_id_from_fragment(self, fragment) -> Optional[int]:
        if fragment is None:
            return None

        account_id = self._coerce_account_id(getattr(fragment, "currentAccount", None))
        if account_id is not None:
            return account_id

        try:
            if hasattr(fragment, "getCurrentAccount"):
                return self._coerce_account_id(fragment.getCurrentAccount())
        except Exception:
            pass
        return None

    def _resolve_account_id(self, context=None, fragment=None) -> Optional[int]:
        candidate_keys = ("account_id", "accountId", "account", "current_account", "currentAccount")
        if context is not None and hasattr(context, "get"):
            for key in candidate_keys:
                account_id = self._coerce_account_id(context.get(key))
                if account_id is not None:
                    return account_id

            fragment = context.get("fragment") or fragment

        account_id = self._resolve_account_id_from_fragment(fragment or get_last_fragment())
        if account_id is not None:
            return account_id

        try:
            UserConfig = jclass("org.telegram.messenger.UserConfig")
            return self._coerce_account_id(getattr(UserConfig, "selectedAccount", None))
        except Exception:
            return None

    def _collect_effective_settings(
        self,
        override_settings: Optional[Dict[str, Any]] = None,
        account_id: Optional[int] = None,
        context=None,
        fragment=None,
    ) -> Dict[str, Any]:
        override_settings = override_settings or {}
        settings = {}

        effective_account_id = self._coerce_account_id(override_settings.get("account_id", account_id))
        if effective_account_id is None:
            effective_account_id = self._resolve_account_id(context=context, fragment=fragment)
        if effective_account_id is not None:
            settings["account_id"] = effective_account_id

        settings["export_format"] = self._coerce_index(
            override_settings.get("export_format", self.get_setting("export_format", CONFIG_DEFAULTS["export_format"])),
            0,
            2,
            CONFIG_DEFAULTS["export_format"],
        )
        settings["media_handling"] = self._coerce_index(
            override_settings.get("media_handling", self.get_setting("media_handling", CONFIG_DEFAULTS["media_handling"])),
            0,
            2,
            CONFIG_DEFAULTS["media_handling"],
        )
        settings["max_messages"] = str(
            override_settings.get("max_messages", self.get_setting("max_messages", CONFIG_DEFAULTS["max_messages"])) or ""
        ).strip()
        settings["max_size_mb"] = str(
            override_settings.get("max_size_mb", self.get_setting("max_size_mb", CONFIG_DEFAULTS["max_size_mb"])) or ""
        ).strip() or CONFIG_DEFAULTS["max_size_mb"]
        settings["remote_download_batch_size"] = max(
            25,
            min(
                200,
                self._parse_positive_int(
                    override_settings.get(
                        "remote_download_batch_size",
                        self.get_setting("remote_download_batch_size", CONFIG_DEFAULTS["remote_download_batch_size"]),
                    ),
                    CONFIG_DEFAULTS["remote_download_batch_size"],
                ),
            ),
        )
        settings["allow_large_chat_remote_downloads"] = bool(
            override_settings.get(
                "allow_large_chat_remote_downloads",
                self.get_setting(
                    "allow_large_chat_remote_downloads",
                    CONFIG_DEFAULTS["allow_large_chat_remote_downloads"],
                ),
            )
        )
        settings["use_api_mode"] = bool(
            override_settings.get("use_api_mode", self.get_setting("use_api_mode", CONFIG_DEFAULTS["use_api_mode"]))
        )
        settings["include_timestamps"] = bool(
            override_settings.get(
                "include_timestamps",
                self.get_setting("include_timestamps", CONFIG_DEFAULTS["include_timestamps"]),
            )
        )
        settings["include_avatars"] = bool(
            override_settings.get("include_avatars", self.get_setting("include_avatars", CONFIG_DEFAULTS["include_avatars"]))
        )
        settings["group_messages"] = bool(
            override_settings.get("group_messages", self.get_setting("group_messages", CONFIG_DEFAULTS["group_messages"]))
        )
        settings["use_chunked_mode"] = bool(
            override_settings.get(
                "use_chunked_mode",
                self.get_setting("use_chunked_mode", CONFIG_DEFAULTS["use_chunked_mode"]),
            )
        )
        settings["chunk_size"] = self._parse_positive_int(
            override_settings.get("chunk_size", self.get_setting("chunk_size", CONFIG_DEFAULTS["chunk_size"])),
            DEFAULT_CHUNK_SIZE,
        )
        settings["enable_overlay_stroke"] = bool(
            override_settings.get(
                "enable_overlay_stroke",
                self.get_setting("enable_overlay_stroke", CONFIG_DEFAULTS["enable_overlay_stroke"]),
            )
        )
        settings["enable_stroke_animation"] = bool(
            override_settings.get(
                "enable_stroke_animation",
                self.get_setting("enable_stroke_animation", CONFIG_DEFAULTS["enable_stroke_animation"]),
            )
        )
        settings["overlay_opacity"] = str(
            override_settings.get(
                "overlay_opacity",
                self.get_setting("overlay_opacity", CONFIG_DEFAULTS["overlay_opacity"]),
            )
            or ""
        ).strip() or CONFIG_DEFAULTS["overlay_opacity"]
        settings["overlay_position"] = self._coerce_index(
            override_settings.get(
                "overlay_position",
                self.get_setting("overlay_position", CONFIG_DEFAULTS["overlay_position"]),
            ),
            0,
            5,
            CONFIG_DEFAULTS["overlay_position"],
        )

        if settings["export_format"] != 0:
            settings["media_handling"] = 0
            settings["include_avatars"] = False

        return settings

    def _hook_android_utilities(self) -> None:
        try:
            android_utilities = JClass.forName("org.telegram.messenger.AndroidUtilities")
            for method in android_utilities.getDeclaredMethods():
                method_name = method.getName()
                parameter_types = method.getParameterTypes()
                if method_name == "openDocument" and len(parameter_types) == 3:
                    if "MessageObject" in parameter_types[0].getName():
                        method.setAccessible(True)
                        self._hooks.append(self.hook_method(method, DexOpenHook(self)))
                elif method_name == "openForView" and len(parameter_types) == 4:
                    if "MessageObject" in parameter_types[0].getName():
                        method.setAccessible(True)
                        self._hooks.append(self.hook_method(method, DexOpenHook(self)))
                elif method_name == "openForView" and len(parameter_types) == 6:
                    first_param = parameter_types[0].getName()
                    second_param = parameter_types[1].getName()
                    if first_param == "java.io.File" and second_param == "java.lang.String":
                        method.setAccessible(True)
                        self._hooks.append(self.hook_method(method, DexLocalFileOpenHook(self)))
        except Exception as error:
            self._log(f"Не удалось перехватить открытие DEX через AndroidUtilities: {error}")

    def _run_on_ui_thread_sync(self, fn, timeout_ms: int = 3000) -> None:
        try:
            if Looper.getMainLooper().isCurrentThread():
                fn()
                return
        except Exception:
            pass

        finished = threading.Event()
        error_holder = {}

        def wrapped():
            try:
                fn()
            except Exception as error:
                error_holder["error"] = error
            finally:
                finished.set()

        run_on_ui_thread(wrapped)
        finished.wait(timeout_ms / 1000.0)

        error = error_holder.get("error")
        if error is not None:
            raise error

    def _reset_pending_download_state(self) -> None:
        self._pending_file_name = None
        self._pending_install_after_download = False

    def _ensure_download_delegate(self) -> None:
        if self._download_delegate is None:
            self._download_delegate = DexDownloadDelegate(self)

    def _is_dex_file_name(self, file_name: str) -> bool:
        normalized = str(file_name or "").strip().lower()
        return bool(normalized and normalized.endswith(".dex"))

    def _is_supported_dex_message(self, message) -> bool:
        return self._is_dex_file_name(self._get_message_file_name(message))

    def _get_message_file_name(self, message) -> str:
        if message is None:
            return ""

        for getter_name in ("getFileName", "getDocumentName"):
            try:
                value = getattr(message, getter_name)()
                if value:
                    return str(value)
            except Exception:
                pass

        try:
            document = message.getDocument()
            if document is not None:
                for attr_name in ("file_name_fixed", "file_name"):
                    value = getattr(document, attr_name, None)
                    if value:
                        return str(value)
        except Exception:
            pass

        return ""

    def _resolve_message_file_path(self, message) -> Optional[str]:
        if message is None:
            return None

        try:
            attach_path = getattr(message.messageOwner, "attachPath", None)
            if attach_path:
                attach_path = str(attach_path)
                if attach_path and os.path.exists(attach_path):
                    return attach_path
        except Exception:
            pass

        try:
            file_loader = jclass("org.telegram.messenger.FileLoader")
            file_obj = file_loader.getInstance(message.currentAccount).getPathToMessage(message.messageOwner)
            if file_obj is not None:
                path = str(file_obj.getAbsolutePath())
                if os.path.exists(path):
                    return path
        except Exception as error:
            self._log(f"Ошибка определения пути DEX из сообщения: {error}")

        return None

    def install_dex_from_path(self, path: str, announce: bool = True) -> None:
        try:
            self.dex_loader.install_from_path(path)
            self._sync_settings_to_dex()
            self._log(f"DEX установлен из файла: {path}")
            if announce:
                BulletinHelper.show_success("DEX-ядро Chat Export загружено.", get_last_fragment())
        except Exception as error:
            self._log(f"Ошибка установки DEX: {error}")
            BulletinHelper.show_error(f"Не удалось загрузить DEX: {error}", get_last_fragment())

    def download_dex_from_message(self, message) -> None:
        try:
            document = message.getDocument()
            if document is None:
                BulletinHelper.show_error("В этом сообщении отсутствует DEX-файл.", get_last_fragment())
                return

            file_name = self._get_message_file_name(message)
            if not file_name:
                BulletinHelper.show_error("Не удалось определить имя DEX-файла.", get_last_fragment())
                return

            account = int(message.currentAccount)
            self._attach_download_observer(account)
            self._pending_file_name = file_name
            self._pending_install_after_download = True

            file_loader = jclass("org.telegram.messenger.FileLoader")
            file_loader.getInstance(account).loadFile(document, message, file_loader.PRIORITY_HIGH, 1)

            BulletinHelper.show_info(f"Загрузка {file_name}...", get_last_fragment())
        except Exception as error:
            self._reset_pending_download_state()
            self._detach_download_observer()
            self._log(f"Ошибка запуска загрузки DEX: {error}")
            BulletinHelper.show_error(f"Не удалось начать загрузку DEX: {error}", get_last_fragment())

    def _attach_download_observer(self, account: int) -> None:
        def attach():
            if self._observer_account == account:
                return

            self._ensure_download_delegate()
            self._detach_download_observer_locked()
            notification_center = NotificationCenter.getInstance(account)
            notification_center.addObserver(self._download_delegate, NotificationCenter.fileLoaded)
            notification_center.addObserver(self._download_delegate, NotificationCenter.fileLoadFailed)
            self._observer_account = account

        self._run_on_ui_thread_sync(attach)

    def _detach_download_observer(self) -> None:
        self._run_on_ui_thread_sync(self._detach_download_observer_locked)

    def _detach_download_observer_locked(self) -> None:
        if self._observer_account is None:
            return

        try:
            if self._download_delegate is None:
                return
            notification_center = NotificationCenter.getInstance(self._observer_account)
            notification_center.removeObserver(self._download_delegate, NotificationCenter.fileLoaded)
            notification_center.removeObserver(self._download_delegate, NotificationCenter.fileLoadFailed)
        except Exception:
            pass
        finally:
            self._observer_account = None

    def _matches_pending_file(self, location) -> bool:
        if not self._pending_file_name or location is None:
            return False
        return str(location).lower() == str(self._pending_file_name).lower()

    def _on_download_notification(self, notification_id, account, *args) -> None:
        try:
            if self._observer_account is None or int(account) != int(self._observer_account):
                return

            if notification_id == NotificationCenter.fileLoaded:
                location = args[0] if len(args) > 0 else None
                final_file = args[1] if len(args) > 1 else None
                if not self._matches_pending_file(location):
                    return

                path = None
                if final_file is not None and hasattr(final_file, "getAbsolutePath"):
                    path = str(final_file.getAbsolutePath())

                install_after_download = self._pending_install_after_download
                self._reset_pending_download_state()
                self._detach_download_observer()

                if path and os.path.exists(path) and install_after_download:
                    run_on_ui_thread(lambda: self.install_dex_from_path(path, announce=True))
                else:
                    run_on_ui_thread(
                        lambda: BulletinHelper.show_error(
                            "DEX был загружен, но локальный файл не удалось найти.",
                            get_last_fragment(),
                        )
                    )

            elif notification_id == NotificationCenter.fileLoadFailed:
                location = args[0] if len(args) > 0 else None
                if not self._matches_pending_file(location):
                    return

                failed_name = self._pending_file_name or DEX_FILE_NAME
                self._reset_pending_download_state()
                self._detach_download_observer()
                run_on_ui_thread(
                    lambda: BulletinHelper.show_error(
                        f"Не удалось загрузить {failed_name}",
                        get_last_fragment(),
                    )
                )
        except Exception as error:
            self._log(f"Ошибка уведомления о загрузке DEX: {error}")

    def _build_export_request(self, dialog_id: int, settings: Dict[str, Any]) -> str:
        request = dict(settings)
        request["dialog_id"] = int(dialog_id)
        request["dialogId"] = int(dialog_id)
        account_id = self._coerce_account_id(settings.get("account_id"))
        if account_id is not None:
            request["account_id"] = account_id
            request["accountId"] = account_id
        request["settings"] = dict(settings)
        request["maxMessages"] = self._parse_positive_int(settings.get("max_messages"), None)
        request["maxSizeMb"] = self._parse_positive_int(settings.get("max_size_mb"), 100)
        request["chunkSize"] = self._parse_positive_int(settings.get("chunk_size"), DEFAULT_CHUNK_SIZE)
        request["remoteDownloadBatchSize"] = self._parse_positive_int(
            settings.get("remote_download_batch_size"),
            CONFIG_DEFAULTS["remote_download_batch_size"],
        )
        request["allowLargeChatRemoteDownloads"] = bool(settings.get("allow_large_chat_remote_downloads"))
        request["wrapperVersion"] = __version__
        request["pluginId"] = __id__
        return json.dumps(request, ensure_ascii=False)

    def _parse_positive_int(self, value, default=None):
        try:
            if value is None:
                return default
            if isinstance(value, int):
                return value if value > 0 else default
            text = str(value).strip()
            if not text:
                return default
            parsed = int(text)
            return parsed if parsed > 0 else default
        except (TypeError, ValueError):
            return default

    def _coerce_bool(self, value) -> bool:
        if isinstance(value, bool):
            return value
        if isinstance(value, (int, float)):
            return bool(value)
        text = str(value or "").strip().lower()
        if text in ("true", "1", "yes", "y", "on"):
            return True
        if text in ("false", "0", "no", "n", "off", ""):
            return False
        return bool(value)

    def _coerce_index(self, value, min_value: int, max_value: int, default: int) -> int:
        try:
            parsed = int(value)
            if min_value <= parsed <= max_value:
                return parsed
        except Exception:
            pass
        return default

    def _show_progress_dialog(self):
        try:
            fragment = get_last_fragment()
            if not fragment:
                return
            context = fragment.getParentActivity()
            if not context:
                return

            self._set_background_notification_enabled(False)
            resources_provider = self._get_resources_provider(fragment)
            self.progress_dialog = AlertDialogBuilder(context, AlertDialogBuilder.ALERT_TYPE_MESSAGE, resources_provider)
            self.progress_dialog.set_view(self._build_progress_dialog_view(context, resources_provider))
            self.progress_dialog.set_neutral_button(
                self._t("run_in_background"),
                lambda b, w: self._run_export_in_background(),
            )
            self.progress_dialog.set_negative_button(
                self._t("cancel"),
                lambda b, w: self._cancel_export(),
            )
            self.progress_dialog.show()
            self.progress_dialog.set_cancelable(False)
            self._refresh_progress_dialog_content(animated=False)
        except Exception as error:
            self._log(f"Ошибка показа диалога прогресса: {error}")

    def _cancel_export(self):
        self.cancel_requested = True
        self._set_background_notification_enabled(False)
        try:
            self.dex_loader.cancel_export()
        except Exception as error:
            self._log(f"Ошибка отправки отмены в DEX: {error}")
        self._dismiss_progress_dialog()
        self._dismiss_progress_overlay()

    def _run_export_in_background(self):
        self._dismiss_progress_dialog()
        self._set_background_notification_enabled(True)
        run_on_ui_thread(self._create_progress_overlay)

    def _create_progress_overlay(self):
        try:
            fragment = get_last_fragment()
            if not fragment:
                BulletinHelper.show_info(self._t("background_export"))
                return
            context = fragment.getParentActivity()
            if not context:
                BulletinHelper.show_info(self._t("background_export"))
                return

            root_view = context.getWindow().getDecorView()
            container = root_view.findViewById(0x01020002) or root_view
            if not container:
                BulletinHelper.show_info(self._t("background_export"))
                return

            if self.progress_overlay_container:
                try:
                    container.removeView(self.progress_overlay_container)
                except Exception:
                    pass

            self.progress_overlay_container = FrameLayout(context)
            self.progress_overlay_container.setClickable(False)
            self.progress_overlay_container.setFocusable(False)

            settings = self._collect_effective_settings(self.temp_export_settings)
            try:
                overlay_opacity_percent = int(settings.get("overlay_opacity", "100"))
            except Exception:
                overlay_opacity_percent = 100
            overlay_opacity = max(0.0, min(1.0, overlay_opacity_percent / 100.0))

            self.progress_overlay = FloatingProgressOverlay(
                context,
                settings.get("enable_overlay_stroke", True),
                settings.get("enable_stroke_animation", True),
                overlay_opacity,
            )

            self.progress_overlay.set_progress(
                self.current_progress_value,
                self.current_progress_text or self._t("exporting"),
                self.current_progress_details,
            )

            self.progress_overlay.setOnClickListener(OnClickListener(self._on_progress_overlay_click))
            self.progress_overlay.setOnLongClickListener(OnLongClickListener(self._on_progress_overlay_long_click))
            self.progress_overlay.setClickable(True)
            self.progress_overlay.setFocusable(True)

            overlay_params = FrameLayout.LayoutParams(
                ViewGroup.LayoutParams.WRAP_CONTENT,
                ViewGroup.LayoutParams.WRAP_CONTENT,
            )
            self._apply_overlay_position(overlay_params, settings.get("overlay_position", 0))
            self.progress_overlay_container.addView(self.progress_overlay.get_view(), overlay_params)

            container.addView(
                self.progress_overlay_container,
                FrameLayout.LayoutParams(
                    ViewGroup.LayoutParams.MATCH_PARENT,
                    ViewGroup.LayoutParams.MATCH_PARENT,
                ),
            )
            self.progress_overlay.show_with_animation(overlay_params.gravity)
        except Exception as error:
            self._log(f"Ошибка показа фонового оверлея: {error}")
            BulletinHelper.show_info(self._t("background_export"))

    def _apply_overlay_position(self, params, overlay_position):
        try:
            position = int(overlay_position)
        except Exception:
            position = 0

        if position == 1:
            params.gravity = Gravity.TOP | Gravity.LEFT
            params.topMargin = AndroidUtilities.dp(80)
            params.leftMargin = AndroidUtilities.dp(16)
        elif position == 2:
            params.gravity = Gravity.TOP | Gravity.RIGHT
            params.topMargin = AndroidUtilities.dp(80)
            params.rightMargin = AndroidUtilities.dp(16)
        elif position == 3:
            params.gravity = Gravity.BOTTOM | Gravity.CENTER_HORIZONTAL
            params.bottomMargin = AndroidUtilities.dp(80)
            params.leftMargin = AndroidUtilities.dp(16)
            params.rightMargin = AndroidUtilities.dp(16)
        elif position == 4:
            params.gravity = Gravity.BOTTOM | Gravity.LEFT
            params.bottomMargin = AndroidUtilities.dp(80)
            params.leftMargin = AndroidUtilities.dp(16)
        elif position == 5:
            params.gravity = Gravity.BOTTOM | Gravity.RIGHT
            params.bottomMargin = AndroidUtilities.dp(80)
            params.rightMargin = AndroidUtilities.dp(16)
        else:
            params.gravity = Gravity.TOP | Gravity.CENTER_HORIZONTAL
            params.topMargin = AndroidUtilities.dp(80)
            params.leftMargin = AndroidUtilities.dp(16)
            params.rightMargin = AndroidUtilities.dp(16)

    def _on_progress_overlay_click(self, view=None):
        try:
            if self.progress_overlay:
                self.progress_overlay.toggle_minimize()
        except Exception as error:
            self._log(f"Ошибка клика по оверлею прогресса: {error}")

    def _on_progress_overlay_long_click(self, view=None):
        try:
            self._dismiss_progress_overlay()
            run_on_ui_thread(self._show_progress_dialog)
            return True
        except Exception as error:
            self._log(f"Ошибка long-click по оверлею прогресса: {error}")
            return False

    def _dismiss_progress_overlay(self):
        def dismiss():
            try:
                overlay = self.progress_overlay
                overlay_container = self.progress_overlay_container
                self.progress_overlay = None
                self.progress_overlay_container = None

                def finalize():
                    try:
                        if overlay_container:
                            parent = overlay_container.getParent()
                            if parent:
                                parent.removeView(overlay_container)
                    except Exception as remove_error:
                        self._log(f"Progress overlay finalize error: {remove_error}")

                if overlay:
                    overlay.dismiss_with_animation(finalize)
                else:
                    finalize()
            except Exception as error:
                self._log(f"Ошибка скрытия оверлея прогресса: {error}")

        run_on_ui_thread(dismiss)

    def _dismiss_progress_dialog(self):
        def dismiss():
            try:
                if self.progress_dialog:
                    self.progress_dialog.dismiss()
                    self.progress_dialog = None
                self.progress_dialog_views = None
            except Exception:
                pass

        run_on_ui_thread(dismiss)

    def _share_export_file(
        self,
        file_path: str,
        mime_type: str,
        assets_dir_path: Optional[str] = None,
        result_payload: Optional[Dict[str, Any]] = None,
    ):
        try:
            context = self._get_active_context()
            if not context:
                self._pending_share_result = {
                    "file_path": file_path,
                    "mime_type": mime_type,
                    "assets_dir_path": assets_dir_path or "",
                    "effective_settings": (result_payload or {}).get("effective_settings", {}),
                    "stats": (result_payload or {}).get("stats", {}),
                    "source_mode": (result_payload or {}).get("source_mode", ""),
                }
                BulletinHelper.show_info(self._t("share_pending"))
                return

            source_file = File(file_path)
            if not source_file.exists():
                BulletinHelper.show_error(f"{self._t('share_error')}: {file_path}")
                return

            filename = source_file.getName()
            download_dir = Environment.getExternalStoragePublicDirectory(Environment.DIRECTORY_DOWNLOADS)
            exteragram_dir = File(download_dir, "exteraGram")
            if not exteragram_dir.exists():
                exteragram_dir.mkdirs()

            dest_file = File(exteragram_dir, filename)
            source_path = source_file.getAbsolutePath()
            dest_path = dest_file.getAbsolutePath()
            source_assets_dir = self._resolve_assets_dir_path(source_path, assets_dir_path)
            dest_assets_dir = None

            if os.path.abspath(source_path) != os.path.abspath(dest_path):
                shutil.copy2(source_path, dest_path)
            if source_assets_dir:
                dest_assets_dir = os.path.join(exteragram_dir.getAbsolutePath(), os.path.basename(source_assets_dir))
                if not self._copy_assets_dir_safe(source_assets_dir, dest_assets_dir):
                    dest_assets_dir = None

            authority = ApplicationLoader.getApplicationId() + ".provider"
            uri = FileProvider.getUriForFile(context, authority, dest_file)
            share_mime_type = mime_type or self._guess_mime_type(dest_path)

            is_html_export = self._is_html_export(dest_path, share_mime_type)
            if is_html_export:
                intent = Intent(Intent.ACTION_VIEW)
                intent.setDataAndType(uri, share_mime_type)
                self._attach_html_export_access(
                    intent,
                    context,
                    authority,
                    dest_file,
                    exteragram_dir,
                    dest_assets_dir,
                )
            else:
                intent = Intent(Intent.ACTION_SEND)
                intent.setType(share_mime_type)
                intent.putExtra(Intent.EXTRA_STREAM, uri)
                intent.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION)

            chooser = Intent.createChooser(intent, self._t("export_chat"))
            if is_html_export:
                self._mirror_uri_access(intent, chooser)
            context.startActivity(chooser)
            BulletinHelper.show_success(self._build_export_complete_message(filename, result_payload))

            if os.path.abspath(source_path) != os.path.abspath(dest_path):
                threading.Thread(target=lambda: self._cleanup_temp_file(source_path, 60), daemon=True).start()
            threading.Thread(target=lambda: self._cleanup_old_exports(120), daemon=True).start()
        except Exception as error:
            self._log(f"Ошибка шаринга экспорта: {error}")
            BulletinHelper.show_error(f"{self._t('share_error')}: {error}")

    def _cleanup_temp_file(self, file_path: str, delay_seconds: int):
        try:
            time.sleep(delay_seconds)
            if os.path.exists(file_path):
                os.remove(file_path)
        except Exception as error:
            self._log(f"Ошибка очистки временного файла: {error}")

    def _resolve_assets_dir_path(self, file_path: str, assets_dir_path: Optional[str]) -> Optional[str]:
        candidate = str(assets_dir_path or "").strip()
        if candidate and os.path.isdir(candidate):
            return candidate

        inferred = os.path.splitext(str(file_path or ""))[0] + "_assets"
        if inferred and os.path.isdir(inferred):
            return inferred
        return None

    def _copy_assets_dir(self, source_dir: str, dest_dir: str) -> bool:
        try:
            source_abs = os.path.abspath(str(source_dir))
            dest_abs = os.path.abspath(str(dest_dir))
            if source_abs == dest_abs:
                return True
            if not os.path.isdir(source_abs):
                return False

            dest_root = File(dest_abs)
            if not dest_root.exists() and not dest_root.mkdirs():
                raise OSError(f"Cannot create assets directory: {dest_abs}")
        except Exception as error:
            self._log(f"РћС€РёР±РєР° РєРѕРїРёСЂРѕРІР°РЅРёСЏ РїР°РїРєРё Р°СЃСЃРµС‚РѕРІ: {error}")

    def _copy_assets_dir_safe(self, source_dir: str, dest_dir: str) -> bool:
        try:
            source_abs = os.path.abspath(str(source_dir))
            dest_abs = os.path.abspath(str(dest_dir))
            if source_abs == dest_abs:
                return True
            if not os.path.isdir(source_abs):
                return False

            dest_root = File(dest_abs)
            if not dest_root.exists() and not dest_root.mkdirs():
                raise OSError(f"Cannot create assets directory: {dest_abs}")

            for current_root, dir_names, file_names in os.walk(source_abs):
                relative_root = os.path.relpath(current_root, source_abs)
                target_root = dest_abs if relative_root in (".", "") else os.path.join(dest_abs, relative_root)

                target_root_file = File(target_root)
                if not target_root_file.exists() and not target_root_file.mkdirs():
                    raise OSError(f"Cannot create assets subdirectory: {target_root}")

                for dir_name in dir_names:
                    target_dir = File(os.path.join(target_root, dir_name))
                    if not target_dir.exists() and not target_dir.mkdirs():
                        raise OSError(f"Cannot create nested assets directory: {target_dir.getAbsolutePath()}")

                for file_name in file_names:
                    source_file = os.path.join(current_root, file_name)
                    target_file = os.path.join(target_root, file_name)
                    shutil.copyfile(source_file, target_file)
            return True
        except Exception as error:
            self._log(f"Ошибка копирования папки ассетов: {error}")
            return False

    def _is_html_export(self, file_path: Optional[str], mime_type: Optional[str]) -> bool:
        guessed_mime_type = str(mime_type or self._guess_mime_type(file_path)).lower()
        if guessed_mime_type == "text/html":
            return True
        return os.path.splitext(str(file_path or ""))[1].lower() in (".html", ".htm")

    def _attach_html_export_access_legacy(self, intent, context, authority: str, export_dir: File) -> None:
        try:
            export_root_uri = FileProvider.getUriForFile(context, authority, export_dir)
            clip_data = ClipData.newUri(context.getContentResolver(), export_dir.getName(), export_root_uri)
            intent.setClipData(clip_data)
            try:
                intent.addFlags(Intent.FLAG_GRANT_PREFIX_URI_PERMISSION)
            except Exception as error:
                self._log(f"Не удалось включить prefix-доступ для HTML-экспорта: {error}")
        except Exception as error:
            self._log(f"Не удалось выдать доступ к папке HTML-экспорта: {error}")

    def _attach_html_export_access(
        self,
        intent,
        context,
        authority: str,
        export_file: File,
        export_dir: File,
        assets_dir_path: Optional[str] = None,
    ) -> None:
        granted_uris = []
        export_root_uri = None

        try:
            granted_uris.append(FileProvider.getUriForFile(context, authority, export_file))
        except Exception as error:
            self._log(f"Не удалось получить URI HTML-экспорта: {error}")

        try:
            export_root_uri = FileProvider.getUriForFile(context, authority, export_dir)
            granted_uris.append(export_root_uri)
        except Exception as error:
            self._log(f"Не удалось получить URI каталога HTML-экспорта: {error}")

        assets_uri = self._resolve_html_assets_uri(context, authority, assets_dir_path)
        if assets_uri:
            granted_uris.append(assets_uri)

        clip_uri = export_root_uri or (granted_uris[0] if granted_uris else None)
        if clip_uri:
            try:
                clip_label = export_dir.getName() if export_root_uri else export_file.getName()
                clip_data = ClipData.newUri(context.getContentResolver(), clip_label, clip_uri)
                intent.setClipData(clip_data)
            except Exception as error:
                self._log(f"Не удалось прикрепить ClipData для HTML-экспорта: {error}")

        intent.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION)
        try:
            intent.addFlags(Intent.FLAG_GRANT_PREFIX_URI_PERMISSION)
        except Exception as error:
            self._log(f"Не удалось включить prefix-доступ для HTML-экспорта: {error}")

        self._grant_uri_access_to_handlers(context, intent, granted_uris)

    def _resolve_html_assets_uri(self, context, authority: str, assets_dir_path: Optional[str]):
        assets_path = str(assets_dir_path or "").strip()
        if not assets_path:
            return None
        try:
            assets_dir = File(assets_path)
            if not assets_dir.exists():
                return None
            return FileProvider.getUriForFile(context, authority, assets_dir)
        except Exception as error:
            self._log(f"Не удалось получить URI каталога ассетов HTML-экспорта: {error}")
            return None

    def _grant_uri_access_to_handlers(self, context, intent, uris) -> None:
        unique_uris = []
        seen = set()
        for uri in uris:
            if uri is None:
                continue
            uri_key = str(uri)
            if uri_key in seen:
                continue
            seen.add(uri_key)
            unique_uris.append(uri)

        if not unique_uris:
            return

        try:
            handlers = self._coerce_java_list(context.getPackageManager().queryIntentActivities(intent, 0))
        except Exception as error:
            self._log(f"Не удалось определить обработчики HTML-экспорта: {error}")
            return

        if not handlers:
            return

        grant_flags = Intent.FLAG_GRANT_READ_URI_PERMISSION
        try:
            grant_flags |= Intent.FLAG_GRANT_PREFIX_URI_PERMISSION
        except Exception:
            pass

        for resolve_info in handlers:
            try:
                activity_info = resolve_info.activityInfo
                package_name = activity_info.packageName if activity_info else None
                if not package_name:
                    continue
                for uri in unique_uris:
                    try:
                        context.grantUriPermission(package_name, uri, grant_flags)
                    except Exception:
                        context.grantUriPermission(package_name, uri, Intent.FLAG_GRANT_READ_URI_PERMISSION)
            except Exception as error:
                self._log(f"Не удалось выдать URI-доступ обработчику HTML-экспорта: {error}")

    def _coerce_java_list(self, value):
        if value is None:
            return []
        try:
            return list(value)
        except Exception:
            pass

        try:
            size = int(value.size())
            return [value.get(index) for index in range(size)]
        except Exception:
            pass

        try:
            array = value.toArray()
            return list(array) if array is not None else []
        except Exception:
            return []

    def _mirror_uri_access(self, source_intent, target_intent) -> None:
        try:
            clip_data = source_intent.getClipData()
            if clip_data:
                target_intent.setClipData(clip_data)
        except Exception as error:
            self._log(f"Не удалось перенести ClipData в chooser HTML-экспорта: {error}")

        try:
            target_intent.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION)
            target_intent.addFlags(Intent.FLAG_GRANT_PREFIX_URI_PERMISSION)
        except Exception as error:
            self._log(f"Не удалось перенести URI-флаги в chooser HTML-экспорта: {error}")

    def _cleanup_old_exports(self, delay_seconds: int):
        try:
            time.sleep(delay_seconds)
            export_dir = File(ApplicationLoader.getFilesDirFixed(), "ChatExports")
            if not export_dir.exists():
                return

            now = time.time()
            max_age = 3600
            files = export_dir.listFiles()
            if not files:
                return

            for file_obj in files:
                age = now - (file_obj.lastModified() / 1000.0)
                if age <= max_age:
                    continue
                if file_obj.isDirectory():
                    shutil.rmtree(file_obj.getAbsolutePath(), ignore_errors=True)
                elif file_obj.isFile():
                    file_obj.delete()
        except Exception as error:
            self._log(f"Ошибка очистки старых экспортов: {error}")

    def _get_active_context(self):
        fragment = get_last_fragment()
        if not fragment:
            return None
        return fragment.getParentActivity()

    def _get_resources_provider(self, fragment=None):
        fragment = fragment or get_last_fragment()
        if fragment is None:
            return None
        try:
            if hasattr(fragment, "getResourceProvider"):
                return fragment.getResourceProvider()
        except Exception:
            pass
        return None

    def _set_background_notification_enabled(self, enabled: bool) -> None:
        self.background_notification_enabled = bool(enabled)
        if self.background_notification_enabled and self.export_in_progress:
            self._update_background_notification()
        else:
            self._dismiss_background_notification()

    def _get_notification_context(self):
        context = getattr(ApplicationLoader, "applicationContext", None)
        if context is not None:
            return context
        return self._get_active_context()

    def _get_notification_manager(self):
        context = self._get_notification_context()
        if context is None:
            return None, None
        try:
            return context, context.getSystemService("notification")
        except Exception:
            return context, None

    def _ensure_export_notification_channel(self, manager) -> None:
        if manager is None or Build.VERSION.SDK_INT < 26:
            return
        channel = NotificationChannel(
            EXPORT_NOTIFICATION_CHANNEL_ID,
            self._t("export_chat"),
            NotificationManager.IMPORTANCE_LOW,
        )
        channel.setDescription(self._t("background_export"))
        channel.enableVibration(False)
        manager.createNotificationChannel(channel)

    def _create_export_notification_intent(self, context):
        try:
            intent = context.getPackageManager().getLaunchIntentForPackage(context.getPackageName())
            if intent is None:
                return None
            intent.addFlags(
                Intent.FLAG_ACTIVITY_NEW_TASK
                | Intent.FLAG_ACTIVITY_CLEAR_TOP
                | Intent.FLAG_ACTIVITY_SINGLE_TOP
            )
            flags = PendingIntent.FLAG_UPDATE_CURRENT
            if Build.VERSION.SDK_INT >= 23:
                flags |= PendingIntent.FLAG_IMMUTABLE
            return PendingIntent.getActivity(context, EXPORT_NOTIFICATION_REQUEST_CODE, intent, flags)
        except Exception as error:
            self._log(f"Ошибка создания PendingIntent уведомления экспорта: {error}")
            return None

    def _build_export_notification(self, context):
        builder_class = jclass("android.app.Notification$Builder")
        if Build.VERSION.SDK_INT >= 26:
            builder = builder_class(context, EXPORT_NOTIFICATION_CHANNEL_ID)
        else:
            builder = builder_class(context)
            builder.setPriority(Notification.PRIORITY_LOW)

        title = self.current_progress_text or self._t("exporting")
        details = self.current_progress_details or self._t("background_export")
        progress_percent = max(0, min(100, int(round(self.current_progress_value * 100.0))))
        indeterminate = progress_percent <= 0

        builder.setContentTitle(title)
        builder.setContentText(details)
        builder.setSmallIcon(TelegramR.drawable.msg_download)
        builder.setOngoing(True)
        builder.setShowWhen(False)
        builder.setProgress(100, progress_percent, indeterminate)

        content_intent = self._create_export_notification_intent(context)
        if content_intent is not None:
            builder.setContentIntent(content_intent)

        notification = builder.build()
        notification.flags |= Notification.FLAG_ONLY_ALERT_ONCE
        return notification

    def _update_background_notification(self) -> None:
        if not self.background_notification_enabled or not self.export_in_progress:
            return
        try:
            context, manager = self._get_notification_manager()
            if context is None or manager is None:
                return
            self._ensure_export_notification_channel(manager)
            manager.notify(EXPORT_NOTIFICATION_ID, self._build_export_notification(context))
        except Exception as error:
            self._log(f"Ошибка обновления системного уведомления экспорта: {error}")

    def _dismiss_background_notification(self) -> None:
        try:
            _, manager = self._get_notification_manager()
            if manager is None:
                return
            manager.cancel(EXPORT_NOTIFICATION_ID)
        except Exception as error:
            self._log(f"Ошибка скрытия системного уведомления экспорта: {error}")

    def _theme_color(self, key, resources_provider=None):
        resources_provider = resources_provider if resources_provider is not None else self._get_resources_provider()
        try:
            if resources_provider is not None:
                return _android_color_int(Theme.getColor(key, resources_provider))
        except Exception:
            pass
        return _android_color_int(Theme.getColor(key))

    def _theme_alpha(self, color: int, alpha: float) -> int:
        alpha_value = max(0, min(255, int(float(alpha) * 255)))
        return _android_color_int((alpha_value << 24) | (int(color) & 0x00FFFFFF))

    def _build_progress_chip(self, context, text: str, resources_provider=None):
        chip = TextView(context)
        chip.setText(str(text or ""))
        chip.setSingleLine(True)
        chip.setGravity(Gravity.CENTER)
        chip.setTypeface(Typeface.DEFAULT_BOLD)
        chip.setTextSize(TypedValue.COMPLEX_UNIT_DIP, 12)
        chip.setTextColor(self._theme_color(Theme.key_windowBackgroundWhiteBlueText, resources_provider))
        chip.setPadding(
            AndroidUtilities.dp(10),
            AndroidUtilities.dp(8),
            AndroidUtilities.dp(10),
            AndroidUtilities.dp(8),
        )
        background = GradientDrawable()
        background.setCornerRadius(AndroidUtilities.dp(15))
        background.setColor(self._theme_alpha(self._theme_color(Theme.key_groupcreate_spanBackground, resources_provider), 0.95))
        chip.setBackground(background)
        return chip

    def _build_progress_dialog_view(self, context, resources_provider=None):
        container = LinearLayout(context)
        container.setOrientation(LinearLayout.VERTICAL)
        container.setPadding(
            AndroidUtilities.dp(8),
            AndroidUtilities.dp(8),
            AndroidUtilities.dp(8),
            AndroidUtilities.dp(2),
        )

        hero_row = LinearLayout(context)
        hero_row.setOrientation(LinearLayout.HORIZONTAL)
        hero_row.setGravity(Gravity.CENTER_VERTICAL)

        icon_wrap = FrameLayout(context)
        icon_bg = GradientDrawable()
        icon_bg.setShape(GradientDrawable.OVAL)
        icon_bg.setColor(self._theme_alpha(self._theme_color(Theme.key_featuredStickers_addButton, resources_provider), 0.14))
        icon_wrap.setBackground(icon_bg)
        icon_view = ImageView(context)
        icon_view.setImageResource(TelegramR.drawable.msg_download)
        try:
            icon_view.setColorFilter(self._theme_color(Theme.key_featuredStickers_addButton, resources_provider))
        except Exception:
            pass
        icon_wrap.addView(
            icon_view,
            FrameLayout.LayoutParams(
                AndroidUtilities.dp(22),
                AndroidUtilities.dp(22),
                Gravity.CENTER,
            ),
        )
        hero_row.addView(
            icon_wrap,
            LinearLayout.LayoutParams(AndroidUtilities.dp(48), AndroidUtilities.dp(48)),
        )

        text_column = LinearLayout(context)
        text_column.setOrientation(LinearLayout.VERTICAL)
        text_params = LinearLayout.LayoutParams(0, LinearLayout.LayoutParams.WRAP_CONTENT, 1.0)
        text_params.leftMargin = AndroidUtilities.dp(12)

        title_view = TextView(context)
        title_view.setTextColor(self._theme_color(Theme.key_dialogTextBlack, resources_provider))
        title_view.setTextSize(TypedValue.COMPLEX_UNIT_DIP, 18)
        title_view.setTypeface(Typeface.DEFAULT_BOLD)
        text_column.addView(title_view)

        subtitle_view = TextView(context)
        subtitle_view.setTextColor(self._theme_color(Theme.key_dialogTextGray3, resources_provider))
        subtitle_view.setTextSize(TypedValue.COMPLEX_UNIT_DIP, 13)
        subtitle_view.setLineSpacing(AndroidUtilities.dp(2), 1.0)
        subtitle_params = LinearLayout.LayoutParams(
            LinearLayout.LayoutParams.WRAP_CONTENT,
            LinearLayout.LayoutParams.WRAP_CONTENT,
        )
        subtitle_params.topMargin = AndroidUtilities.dp(2)
        text_column.addView(subtitle_view, subtitle_params)
        hero_row.addView(text_column, text_params)
        container.addView(hero_row)

        progress_bar = LineProgressView(context)
        progress_bar.setProgressColor(self._theme_color(Theme.key_featuredStickers_addButton, resources_provider))
        progress_bar.setBackColor(self._theme_color(Theme.key_dialogLineProgressBackground, resources_provider))
        progress_params = LinearLayout.LayoutParams(
            LinearLayout.LayoutParams.MATCH_PARENT,
            AndroidUtilities.dp(6),
        )
        progress_params.topMargin = AndroidUtilities.dp(16)
        container.addView(progress_bar, progress_params)

        chips_row = LinearLayout(context)
        chips_row.setOrientation(LinearLayout.HORIZONTAL)
        chips_row.setGravity(Gravity.LEFT | Gravity.CENTER_VERTICAL)
        chips_params = LinearLayout.LayoutParams(
            LinearLayout.LayoutParams.MATCH_PARENT,
            LinearLayout.LayoutParams.WRAP_CONTENT,
        )
        chips_params.topMargin = AndroidUtilities.dp(12)

        format_chip = self._build_progress_chip(context, "", resources_provider)
        media_chip = self._build_progress_chip(context, "", resources_provider)
        source_chip = self._build_progress_chip(context, "", resources_provider)

        for index, chip in enumerate((format_chip, media_chip, source_chip)):
            params = LinearLayout.LayoutParams(0, LinearLayout.LayoutParams.WRAP_CONTENT, 1.0)
            if index > 0:
                params.leftMargin = AndroidUtilities.dp(8)
            chips_row.addView(chip, params)
        container.addView(chips_row, chips_params)

        self.progress_dialog_views = {
            "title": title_view,
            "subtitle": subtitle_view,
            "bar": progress_bar,
            "format_chip": format_chip,
            "media_chip": media_chip,
            "source_chip": source_chip,
        }
        return container

    def _refresh_progress_dialog_content(self, animated: bool = True):
        views = self.progress_dialog_views or {}
        if not views:
            return
        settings = self.temp_export_settings if isinstance(self.temp_export_settings, dict) else {}
        try:
            views["title"].setText(self.current_progress_text or self._t("exporting"))
            subtitle = self.current_progress_details or self._t("progress_subtitle")
            views["subtitle"].setText(subtitle)
            views["bar"].setProgress(self.current_progress_value, bool(animated))
            views["format_chip"].setText(f"{self._t('progress_chip_format')}: {self._progress_format_label(settings)}")
            views["media_chip"].setText(f"{self._t('progress_chip_media')}: {self._progress_media_label(settings)}")
            views["source_chip"].setText(
                f"{self._t('progress_chip_source')}: {self._progress_source_label(settings, self.current_source_mode)}"
            )
        except Exception as error:
            self._log(f"Ошибка обновления кастомного прогресс-диалога: {error}")

    def _update_progress(self, message: str, progress: int = 0, details: str = ""):
        self.current_progress_text = str(message or self._t("exporting"))
        self.current_progress_details = str(details or "")
        self.current_progress_value = max(0.0, min(1.0, float(progress) / 100.0))

        def update():
            try:
                if self.progress_dialog:
                    self._refresh_progress_dialog_content(animated=True)
                if self.progress_overlay:
                    self.progress_overlay.set_progress(
                        self.current_progress_value,
                        self.current_progress_text,
                        self.current_progress_details,
                    )
            except Exception as error:
                self._log(f"Ошибка обновления прогресса: {error}")

        run_on_ui_thread(update)
        if self.background_notification_enabled:
            self._update_background_notification()

    def _guess_mime_type(self, file_path: Optional[str]) -> str:
        ext = os.path.splitext(str(file_path or ""))[1].lower()
        return {
            ".html": "text/html",
            ".json": "application/json",
            ".txt": "text/plain",
        }.get(ext, "application/octet-stream")

    def _build_export_complete_message(self, filename: str, result_payload: Optional[Dict[str, Any]] = None) -> str:
        base = f"{self._t('export_complete')}: Download/exteraGram/{filename}"
        payload = result_payload if isinstance(result_payload, dict) else {}
        effective_settings = payload.get("effective_settings") if isinstance(payload.get("effective_settings"), dict) else {}
        stats = payload.get("stats") if isinstance(payload.get("stats"), dict) else {}

        parts = []
        if self._coerce_bool(effective_settings.get("auto_large_chat_mode")):
            parts.append(self._t("export_auto_mode_note"))

        media_found_local = self._coerce_progress_count(stats.get("media_found_local"))
        media_requested = self._coerce_progress_count(stats.get("media_requested"))
        media_copied = self._coerce_progress_count(stats.get("media_copied"))
        media_missing = self._coerce_progress_count(stats.get("media_missing"))
        if any(value > 0 for value in (media_found_local, media_requested, media_copied, media_missing)):
            parts.append(
                self._t(
                    "export_media_stats_note",
                    local=self._format_progress_number(media_found_local),
                    requested=self._format_progress_number(media_requested),
                    copied=self._format_progress_number(media_copied),
                    missing=self._format_progress_number(media_missing),
                )
            )

        avatars_copied = self._coerce_progress_count(stats.get("avatars_copied"))
        if avatars_copied > 0:
            parts.append(
                self._t(
                    "export_avatar_stats_note",
                    copied=self._format_progress_number(avatars_copied),
                )
            )

        if not parts:
            return base
        return base + " • " + " • ".join(parts)

    def _looks_like_cancel(self, message: str) -> bool:
        lowered = str(message or "").lower()
        return any(marker in lowered for marker in ("cancel", "отмен", "aborted", "stopped"))


plugin = ChatExportPlugin()
