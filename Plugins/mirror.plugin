import base64
import json
import math
import time
import zlib
from datetime import datetime

from android_utils import run_on_ui_thread
from base_plugin import BasePlugin, MenuItemData, MenuItemType
from client_utils import PLUGINS_QUEUE, get_last_fragment, run_on_queue
from ui.alert import AlertDialogBuilder
from ui.bulletin import BulletinHelper
from ui.settings import Divider, Header, Input, Selector, Switch, Text

from java import dynamic_proxy, jclass
from org.telegram.messenger import ApplicationLoader

__id__ = "mirror"
__name__ = "Mirror"
__description__ = "Automatically copy messages between Telegram conversations."
__author__ = "@kvucoPlugins"
__version__ = "1.0.5"
__icon__ = "zelenkaguru/7"
__app_version__ = ">=12.5.1"
__sdk_version__ = ">=1.4.3.3"

ENTRY_CLASS = "com.extera.plugins.mirror.MirrorCore"
DEX_BEGIN = "__DEX_BEGIN__"
DEX_END = "__DEX_END__"
BuildVersion = jclass("android.os.Build$VERSION")

PluginsController = jclass("com.exteragram.messenger.plugins.PluginsController")
Bundle = jclass("android.os.Bundle")
Context = jclass("android.content.Context")
DialogsActivity = jclass("org.telegram.ui.DialogsActivity")
DialogsActivityDelegate = jclass("org.telegram.ui.DialogsActivity$DialogsActivityDelegate")
Intent = jclass("android.content.Intent")
Settings = jclass("android.provider.Settings")
AlertsCreator = jclass("org.telegram.ui.Components.AlertsCreator")
DatePickerDelegate = jclass("org.telegram.ui.Components.AlertsCreator$DatePickerDelegate")
Theme = jclass("org.telegram.ui.ActionBar.Theme")
Uri = jclass("android.net.Uri")


class DialogPickerDelegate(dynamic_proxy(DialogsActivityDelegate)):
    def __init__(self, plugin, key, label):
        super().__init__()
        self.plugin = plugin
        self.key = key
        self.label = label

    def didSelectDialogs(self, fragment, dids, message, param, notify, scheduleDate, scheduleRepeatPeriod, topicsFragment):
        try:
            if dids is None or dids.size() <= 0:
                show_error_ui("Select a conversation first")
                return False
            dialog_id = int(dids.get(0).dialogId)
            error = self.plugin._invoke_core("checkPeer", str(dialog_id), self.key == "target_peer")
            if error is None:
                return False
            error = str(error or "")
            if error:
                show_error_ui(error)
                return False
            self.plugin._clear_comments_cache()
            self.plugin.set_setting(self.key, str(dialog_id), reload_settings=True)
            try:
                fragment.finishFragment()
            except Exception:
                pass
            refresh_plugin_settings(__id__)
            self.plugin._schedule_comments_settings_refresh()
            return True
        except Exception as exc:
            self.plugin.log(f"Dialog picker failed: {exc}")
            show_error_ui(f"Could not select this conversation: {exc}")
            return False

    def canSelectStories(self):
        return False

    def didSelectStories(self, fragment):
        return False


class MirrorDatePickerDelegate(dynamic_proxy(DatePickerDelegate)):
    def __init__(self, plugin, key):
        super().__init__()
        self.plugin = plugin
        self.key = key

    def didSelectDate(self, year, month, dayOfMonth):
        try:
            value = f"{int(year):04d}-{int(month) + 1:02d}-{int(dayOfMonth):02d}"
            self.plugin.set_setting(self.key, value, reload_settings=True)
            refresh_plugin_settings(__id__)
        except Exception as exc:
            self.plugin.log(f"date picker failed: {exc}")
            show_error_ui(f"Could not save this date: {exc}")


def json_compact(value):
    return json.dumps(value, ensure_ascii=False, separators=(",", ":"))


def show_error_ui(message):
    run_on_ui_thread(lambda: BulletinHelper.show_error(str(message)))


def show_success_ui(message):
    run_on_ui_thread(lambda: BulletinHelper.show_success(str(message)))


def refresh_plugin_settings(plugin_id):
    try:
        PluginsController.getInstance().loadPluginSettings(str(plugin_id))
    except Exception:
        pass


def get_private_field_silent(obj, field_name, default=None):
    if obj is None:
        return default
    try:
        cls = obj.getClass()
    except Exception:
        cls = None
    while cls is not None:
        try:
            field = cls.getDeclaredField(field_name)
            field.setAccessible(True)
            return field.get(obj)
        except Exception:
            try:
                cls = cls.getSuperclass()
            except Exception:
                cls = None
    return default


class Dex:
    def __init__(self, entry_class):
        self.entry_class = entry_class
        self.clazz = None
        self.loaded = False

    def load(self):
        if self.loaded:
            return
        if self.clazz is None:
            ctx = ApplicationLoader.applicationContext
            if ctx is None:
                raise RuntimeError("application context is not ready")
            if BuildVersion.SDK_INT < 26:
                raise RuntimeError("embedded DEX requires Android API 26+")
            ByteBuffer = jclass("java.nio.ByteBuffer")
            InMemoryDexClassLoader = jclass("dalvik.system.InMemoryDexClassLoader")
            loader = InMemoryDexClassLoader(ByteBuffer.wrap(self._read_payload()), ctx.getClassLoader())
            self.clazz = loader.loadClass(self.entry_class)
        try:
            self._invoke("load")
            self.loaded = True
        except Exception:
            try:
                self._invoke("unload")
            except Exception:
                pass
            self.loaded = False
            raise

    def unload(self):
        try:
            if self.clazz is not None and self.loaded:
                self._invoke("unload")
        except Exception:
            pass
        # Keep the loaded Class reachable after Java unload. ART/JIT can still
        # retain trampolines into anon DEX code after hook cleanup.
        self.loaded = False

    def call(self, method_name, *args):
        if self.clazz is None or (method_name not in ("load", "unload") and not self.loaded):
            self.load()
        return self._invoke(method_name, *args)

    def _invoke(self, method_name, *args):
        if self.clazz is None:
            raise RuntimeError("embedded dex class is not loaded")
        matches = []
        for method in self.clazz.getDeclaredMethods():
            if str(method.getName()) != method_name or len(method.getParameterTypes()) != len(args):
                continue
            method.setAccessible(True)
            matches.append(method)
        if len(matches) > 1:
            raise RuntimeError(f"ambiguous core method: {method_name}/{len(args)}")
        if matches:
            try:
                return matches[0].invoke(None, *args)
            except Exception as exc:
                cause = None
                try:
                    cause = exc.getCause()
                except Exception:
                    pass
                if cause is not None:
                    try:
                        message = cause.getMessage()
                    except Exception:
                        message = None
                    raise RuntimeError(str(message or cause))
                raise
        raise RuntimeError(f"core method not found: {method_name}/{len(args)}")

    def _read_payload(self):
        try:
            with open(__file__, "r", encoding="utf-8") as source:
                text = source.read()
            block = text.split(f"# {DEX_BEGIN}", 1)[1].split(f"# {DEX_END}", 1)[0]
            payload = "".join(line.strip()[1:].strip() for line in block.splitlines() if line.strip().startswith("#"))
        except Exception as exc:
            raise RuntimeError(f"embedded dex read failed: {exc}")

        if not payload:
            raise RuntimeError("embedded dex payload not found")

        try:
            dex_bytes = zlib.decompress(base64.b64decode(payload))
        except Exception as exc:
            raise RuntimeError(f"embedded dex decode failed: {exc}")

        return dex_bytes


class MirrorPlugin(BasePlugin):
    def __init__(self):
        super().__init__()
        self.dex = None
        self.last_error = ""
        self._dialog_refresh_scheduled = False
        self._settings_refresh_scheduled = False
        self._battery_refresh_scheduled = False
        self._comments_available_key = None
        self._comments_available_value = False
        self._comments_available_at = 0
        self._comments_refresh_scheduled = False
        self._peer_settings_key = None
        self._progress_dialog_builder = None
        self._dialog_picker_delegate = None
        self._date_picker_delegate = None

    def on_plugin_load(self):
        self.dex = Dex(ENTRY_CLASS)
        self._load_core(show_error=True)
        self.add_menu_item(
            MenuItemData(
                menu_type=MenuItemType.CHAT_ACTION_MENU,
                text="Mirror",
                on_click=self.open_mirror_menu,
                icon="msg_forward",
            )
        )

    def on_plugin_unload(self):
        if self.dex:
            self.dex.unload()

    def settings_payload(self):
        source_peer = str(self.get_setting("source_peer", "") or "").strip()
        target_peer = str(self.get_setting("target_peer", "") or "").strip()
        mirror_comments = self._safe_bool(self.get_setting("mirror_comments", False))
        date_mode_index = max(0, min(2, self._safe_int(self.get_setting("copy_date_mode", 0), 0)))
        date_mode = ("all", "day", "range")[date_mode_index]
        date_start = ""
        date_end = ""
        if date_mode == "day":
            date_start = str(self.get_setting("copy_date_day", "") or "").strip()
        elif date_mode == "range":
            date_start = str(self.get_setting("copy_date_from", "") or "").strip()
            date_end = str(self.get_setting("copy_date_to", "") or "").strip()
        if mirror_comments:
            comments_available = self._comments_cached_value(source_peer, target_peer)
            if comments_available is False:
                mirror_comments = False
            elif comments_available is None:
                self._schedule_comments_settings_refresh(source_peer, target_peer)
        return {
            "source_peer": source_peer,
            "target_peer": target_peer,
            "batch_size": max(10, min(100, self._safe_int(self.get_setting("batch_size", "50"), 50))),
            "send_delay_ms": round(max(0.0, min(60.0, self._safe_float(self.get_setting("send_delay_seconds", "1.5"), 1.5))) * 1000),
            "notify": False,
            "show_sender": self._safe_bool(self.get_setting("show_sender", False)),
            "quiz_auto_answer_unknown": self._safe_bool(self.get_setting("quiz_auto_answer_unknown", False)),
            "quiz_send_explanation_spoiler": self._safe_bool(self.get_setting("quiz_send_explanation_spoiler", False)),
            "mirror_comments": mirror_comments,
            "date_mode": date_mode,
            "date_start": date_start,
            "date_end": date_end,
            "types": {
                "text": self._safe_bool(self.get_setting("type_text", True), True),
                "photo": self._safe_bool(self.get_setting("type_photo", True), True),
                "video": self._safe_bool(self.get_setting("type_video", True), True),
                "gif": self._safe_bool(self.get_setting("type_gif", True), True),
                "round_video": self._safe_bool(self.get_setting("type_round_video", True), True),
                "file": self._safe_bool(self.get_setting("type_file", True), True),
                "voice": self._safe_bool(self.get_setting("type_voice", True), True),
                "music": self._safe_bool(self.get_setting("type_music", True), True),
                "sticker": self._safe_bool(self.get_setting("type_sticker", True), True),
                "animated_sticker": self._safe_bool(self.get_setting("type_animated_sticker", True), True),
                "animated_emoji": self._safe_bool(self.get_setting("type_animated_emoji", True), True),
                "location": self._safe_bool(self.get_setting("type_location", True), True),
                "contact": self._safe_bool(self.get_setting("type_contact", True), True),
                "poll": self._safe_bool(self.get_setting("type_poll", True), True),
                "todo": self._safe_bool(self.get_setting("type_todo", True), True),
            },
        }

    def _load_core(self, show_error=False):
        if self.dex is None:
            self.dex = Dex(ENTRY_CLASS)
        try:
            self.dex.load()
            self.last_error = ""
            return True
        except Exception as exc:
            self.last_error = str(exc)
            self.log(f"Embedded DEX load failed: {exc}")
            if show_error:
                show_error_ui("Mirror could not start. Try reinstall the plugin.")
            return False

    def _invoke_core(self, method_name, *args, show_error=True):
        if not self._load_core(show_error=show_error):
            return None
        try:
            return self.dex.call(method_name, *args)
        except Exception as exc:
            self.last_error = str(exc)
            self.log(f"Core call failed: {method_name}: {exc}")
            if show_error:
                show_error_ui(f"Could not complete this action: {exc}")
            return None

    def _call_core_void(self, method_name, *args):
        if not self._load_core(show_error=True):
            return False
        try:
            self.dex.call(method_name, *args)
            self.last_error = ""
            return True
        except Exception as exc:
            self.last_error = str(exc)
            self.log(f"Core call failed: {method_name}: {exc}")
            show_error_ui(f"Could not complete this action: {exc}")
            return False

    def _context_dialog_id(self, context):
        try:
            value = context.get("dialog_id") if isinstance(context, dict) else None
            if value:
                return int(value)
        except Exception:
            pass
        try:
            fragment = context.get("fragment") if isinstance(context, dict) else None
            if fragment is None:
                fragment = get_last_fragment()
            if fragment is not None and str(fragment.getClass().getName()) == "org.telegram.ui.ChatActivity":
                value = get_private_field_silent(fragment, "dialog_id", 0)
                if value:
                    return int(value)
        except Exception:
            pass
        return 0

    def _current_activity(self, context=None):
        try:
            fragment = context.get("fragment") if isinstance(context, dict) else None
        except Exception:
            fragment = None
        if fragment is None:
            try:
                fragment = get_last_fragment()
            except Exception:
                fragment = None
        try:
            activity = fragment.getParentActivity() if fragment is not None else None
            if activity is not None:
                return activity
        except Exception:
            pass
        return ApplicationLoader.applicationContext

    def _parse_date_setting(self, value):
        try:
            return datetime.strptime(str(value or "").strip(), "%Y-%m-%d")
        except Exception:
            return None

    def _date_setting_label(self, key):
        value = str(self.get_setting(key, "") or "").strip()
        if self._parse_date_setting(value) is not None:
            return value
        return "Not selected"

    def _open_date_picker(self, key, title):
        def open_picker_ui():
            try:
                context = self._current_activity()
                current = datetime.now()
                selected = self._parse_date_setting(self.get_setting(key, ""))
                if selected is not None:
                    current = selected
                self._date_picker_delegate = MirrorDatePickerDelegate(self, key)
                builder = AlertsCreator.createDatePickerDialog(
                    context,
                    -120,
                    0,
                    int(current.year) - int(datetime.now().year),
                    int(current.day),
                    int(current.month) - 1,
                    int(current.year),
                    str(title),
                    False,
                    self._date_picker_delegate,
                )
                if builder is None:
                    show_error_ui("Could not open the date picker.")
                    return
                parent = get_last_fragment()
                if parent is not None:
                    parent.showDialog(builder.create())
                else:
                    builder.show()
            except Exception as exc:
                self.log(f"open date picker failed: {exc}")
                show_error_ui(f"Could not open the date picker: {exc}")

        run_on_ui_thread(open_picker_ui)

    def _set_peer_from_context(self, key, label, context):
        dialog_id = self._context_dialog_id(context)
        if not dialog_id:
            show_error_ui("Open a conversation first")
            return
        error = self._invoke_core("checkPeer", str(dialog_id), key == "target_peer")
        if error is None:
            return
        error = str(error or "")
        if error:
            show_error_ui(error)
            return
        peer = self._peer_text(str(dialog_id))
        self._clear_comments_cache()
        self.set_setting(key, str(dialog_id), reload_settings=True)
        show_success_ui(f"{label}: {peer}")
        self._schedule_comments_settings_refresh()

    def set_current_chat_as_source(self, context):
        self._set_peer_from_context("source_peer", "Copy from", context)

    def set_current_chat_as_target(self, context):
        self._set_peer_from_context("target_peer", "Copy to", context)

    def open_mirror_menu(self, context):
        dialog_id = self._context_dialog_id(context)
        if not dialog_id:
            show_error_ui("Open a conversation first")
            return

        def handle_click(builder, which):
            try:
                builder.dismiss()
            except Exception:
                pass
            if which == 0:
                self.set_current_chat_as_source(context)
            elif which == 1:
                self.set_current_chat_as_target(context)
            else:
                self.open_mirror_panel()

        try:
            builder = AlertDialogBuilder(self._current_activity(context))
            builder.set_title("Mirror")
            builder.set_items(
                ["Copy messages from here", "Copy messages here", "Open Mirror"],
                handle_click,
            )
            builder.set_negative_button("Close", lambda b, w: b.dismiss())
            builder.show()
        except Exception as exc:
            self.log(f"open_mirror_menu failed: {exc}")
            show_error_ui(f"Could not open Mirror: {exc}")

    def open_mirror_panel(self, _view=None):
        try:
            PluginsController.openPluginSettings(__id__, "mirror_panel")
        except Exception:
            try:
                PluginsController.openPluginSettings(__id__)
            except Exception as exc:
                self.log(f"open_mirror_panel failed: {exc}")
                show_error_ui(f"Could not open Mirror settings: {exc}")

    def _is_ignoring_battery_optimizations(self):
        try:
            app_context = ApplicationLoader.applicationContext
            package_name = str(app_context.getPackageName())
            power_manager = app_context.getSystemService(Context.POWER_SERVICE)
            return bool(
                power_manager is not None
                and power_manager.isIgnoringBatteryOptimizations(package_name)
            )
        except Exception:
            return False

    def _has_android_permission(self, permission):
        try:
            app_context = ApplicationLoader.applicationContext
            package_name = str(app_context.getPackageName())
            return int(app_context.getPackageManager().checkPermission(permission, package_name)) == 0
        except Exception:
            return False

    def open_battery_settings(self, _view=None):
        app_context = ApplicationLoader.applicationContext
        package_name = str(app_context.getPackageName())
        if self._is_ignoring_battery_optimizations():
            return

        intents = []
        if self._has_android_permission("android.permission.REQUEST_IGNORE_BATTERY_OPTIMIZATIONS"):
            try:
                request = Intent(Settings.ACTION_REQUEST_IGNORE_BATTERY_OPTIMIZATIONS)
                request.setData(Uri.parse("package:" + package_name))
                intents.append(request)
            except Exception:
                pass
        try:
            intents.append(Intent(Settings.ACTION_IGNORE_BATTERY_OPTIMIZATION_SETTINGS))
        except Exception:
            pass
        try:
            details = Intent(Settings.ACTION_APPLICATION_DETAILS_SETTINGS)
            details.setData(Uri.parse("package:" + package_name))
            intents.append(details)
        except Exception:
            pass

        for intent in intents:
            try:
                intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK)
                self._current_activity().startActivity(intent)
                self._schedule_battery_settings_refresh()
                return
            except Exception:
                pass
        show_error_ui("Could not open Android battery settings.")

    def choose_source_dialog(self, _view=None):
        self._open_dialog_picker("source_peer", "Copy from", check_can_write=False)

    def choose_target_dialog(self, _view=None):
        self._open_dialog_picker("target_peer", "Copy to", check_can_write=True)

    def _open_dialog_picker(self, key, label, check_can_write):
        def open_picker_ui():
            try:
                parent = get_last_fragment()
                if parent is None:
                    show_error_ui("Open a screen before choosing a conversation")
                    return
                args = Bundle()
                args.putBoolean("onlySelect", True)
                args.putBoolean("checkCanWrite", bool(check_can_write))
                args.putBoolean("allowGlobalSearch", True)
                args.putBoolean("allowUsers", True)
                args.putBoolean("allowBots", True)
                args.putBoolean("allowGroups", True)
                args.putBoolean("allowMegagroups", True)
                args.putBoolean("allowLegacyGroups", True)
                args.putBoolean("allowChannels", True)
                args.putBoolean("resetDelegate", True)
                picker = DialogsActivity(args)
                self._dialog_picker_delegate = DialogPickerDelegate(self, key, label)
                picker.setDelegate(self._dialog_picker_delegate)
                parent.presentFragment(picker)
            except Exception as exc:
                self.log(f"open dialog picker failed: {exc}")
                show_error_ui(f"Could not open the conversation list: {exc}")

        run_on_ui_thread(open_picker_ui)

    def start_mirror(self, _view=None):
        payload = self.settings_payload()
        if not payload.get("source_peer"):
            show_error_ui("Choose where to copy from first")
            return
        if not payload.get("target_peer"):
            show_error_ui("Choose where to copy to first")
            return

        status = self._read_status()
        if bool(status.get("running")):
            self.show_progress_dialog()
            return

        if self._call_core_void("startMirror", json_compact(payload)):
            refresh_plugin_settings(__id__)
            self.show_progress_dialog()

    def cancel_mirror(self, _view=None):
        if self._call_core_void("cancelMirror"):
            refresh_plugin_settings(__id__)
            self._schedule_settings_refresh()

    def reset_progress(self, _view=None):
        if not self.get_setting("source_peer", "") or not self.get_setting("target_peer", ""):
            show_error_ui("Choose source and target before clearing copy history")
            return
        run_on_ui_thread(self._confirm_reset_database_ui)

    def _confirm_reset_database_ui(self):
        try:
            builder = AlertDialogBuilder(self._current_activity())
            builder.set_title("Clear copy history?")
            builder.set_message(
                "Mirror remembers messages it has already copied. Clear the history if you want "
                "to copy messages from this source to this target again."
            )
            builder.set_positive_button("Clear", lambda b, w: self._reset_database_confirmed(b))
            builder.set_negative_button("Cancel", lambda b, w: b.dismiss())
            builder.make_button_red(AlertDialogBuilder.BUTTON_POSITIVE)
            builder.show()
            self._mark_dialog_title_red(builder)
        except Exception as exc:
            self.log(f"reset confirmation failed: {exc}")

    def _mark_dialog_title_red(self, builder):
        try:
            dialog = builder.get_dialog()
            title_view = get_private_field_silent(dialog, "titleTextView")
            if title_view is not None:
                title_view.setTextColor(Theme.getColor(Theme.key_text_RedBold))
        except Exception:
            pass

    def _reset_database_confirmed(self, builder):
        try:
            builder.dismiss()
        except Exception:
            pass
        if self._call_core_void("resetProgress", json_compact(self.settings_payload())):
            refresh_plugin_settings(__id__)

    def _short_text(self, value, limit=140):
        text = " ".join(str(value).split())
        if len(text) <= limit:
            return text
        return text[:max(0, limit - 3)] + "..."

    def show_progress_dialog(self, _view=None):
        run_on_ui_thread(self._show_progress_dialog_ui)

    def _show_progress_dialog_ui(self):
        try:
            status = self._read_status()
            builder = AlertDialogBuilder(self._current_activity(), AlertDialogBuilder.ALERT_TYPE_LOADING)
            builder.set_title("Mirror")
            builder.set_message(self._progress_dialog_text(status))
            builder.set_positive_button("Run in background", lambda b, w: self._background_progress_dialog(b))
            builder.set_on_dismiss_listener(lambda b: self._clear_progress_dialog())
            builder.show()
            self._progress_dialog_builder = builder
            self._update_progress_dialog(builder, status)
            self._schedule_dialog_refresh()
        except Exception as exc:
            self.log(f"progress dialog failed: {exc}")
            show_error_ui("Could not open Mirror progress.")

    def _background_progress_dialog(self, builder):
        try:
            builder.dismiss()
        except Exception:
            pass
        self._clear_progress_dialog()

    def _clear_progress_dialog(self):
        self._progress_dialog_builder = None

    def _set_dialog_message(self, builder, text):
        try:
            dialog = builder.get_dialog()
            if dialog is not None:
                dialog.setMessage(str(text))
                return
        except Exception:
            pass
        builder.set_message(str(text))

    def _set_dialog_progress(self, builder, progress):
        progress = max(0, min(100, self._safe_int(progress)))
        try:
            dialog = builder.get_dialog()
            if dialog is not None:
                dialog.setProgress(progress)
        except Exception:
            pass

    def _update_progress_dialog(self, builder, status):
        self._set_dialog_message(builder, self._progress_dialog_text(status))
        self._set_dialog_progress(builder, self._dialog_progress(status))

    def _dialog_progress(self, status):
        copied = self._safe_int(status.get("copied", 0))
        skipped = self._safe_int(status.get("skipped", 0))
        deferred = self._safe_int(status.get("deferred", 0))
        scanned = self._safe_int(status.get("scanned", 0))
        current_unit = self._safe_int(status.get("current_unit", 0))
        total_units = self._safe_int(status.get("total_units", 0))
        sent_messages = self._safe_int(status.get("sent_messages", 0))
        total_messages = self._safe_int(status.get("total_messages", 0))
        if scanned > 0:
            unit_size = max(1, self._safe_int(status.get("current_unit_size", 1)))
            in_flight = 0
            if total_messages > 0:
                in_flight = min(unit_size, int((sent_messages * unit_size) / max(1, total_messages)))
            processed = copied + skipped + deferred + in_flight
            return max(0, min(100, int((processed * 100) / scanned)))
        if total_units > 0:
            return max(0, min(100, int((max(0, current_unit - 1) * 100) / total_units)))
        return 0

    def _progress_dialog_text(self, status):
        running = bool(status.get("running"))
        phase = str(status.get("phase", "idle") or "idle")
        summary = str(status.get("summary", "Ready") or "Ready")
        if phase == "cooldown":
            phase = "copying"
            summary = "Copying messages"
        phase_title = self._phase_title(phase, running)
        copied = self._safe_int(status.get("copied", 0))
        skipped = self._safe_int(status.get("skipped", 0))
        deferred = self._safe_int(status.get("deferred", 0))
        failed = self._safe_int(status.get("failed", 0))
        lines = [
            f"{phase_title}: {summary}",
            self._progress_counts_text(copied, skipped, deferred, failed),
        ]
        last_error = str(status.get("last_error", "") or self.last_error or "")
        if last_error:
            lines.extend(["", f"Error: {last_error}"])
        return "\n".join(lines)

    def _progress_counts_text(self, copied, skipped, deferred, failed):
        parts = [f"Copied: {copied}"]
        if skipped:
            parts.append(f"Skipped: {skipped}")
        if deferred:
            parts.append(f"Deferred: {deferred}")
        if failed:
            parts.append(f"Errors: {failed}")
        return "  ".join(parts)

    def _schedule_dialog_refresh(self):
        if self._dialog_refresh_scheduled:
            return
        self._dialog_refresh_scheduled = True

        def refresh_dialog_once():
            self._dialog_refresh_scheduled = False
            builder = self._progress_dialog_builder
            if builder is None:
                return
            status = self._read_status()
            try:
                self._update_progress_dialog(builder, status)
            except Exception:
                self._clear_progress_dialog()
                return
            if bool(status.get("running")):
                self._schedule_dialog_refresh()
            else:
                phase = str(status.get("phase", "") or "")
                refresh_plugin_settings(__id__)
                if phase in ("finished", "canceled"):
                    try:
                        builder.dismiss()
                    except Exception:
                        pass
                    self._clear_progress_dialog()
                    refresh_plugin_settings(__id__)

        run_on_ui_thread(refresh_dialog_once, 1500)

    def _schedule_settings_refresh(self):
        if self._settings_refresh_scheduled:
            return
        self._settings_refresh_scheduled = True

        def refresh_settings_once():
            self._settings_refresh_scheduled = False
            status = self._read_status()
            refresh_plugin_settings(__id__)
            phase = str(status.get("phase", "") or "")
            if bool(status.get("running")) or phase == "canceling":
                self._schedule_settings_refresh()

        run_on_ui_thread(refresh_settings_once, 1000)

    def _schedule_battery_settings_refresh(self, remaining=180):
        if self._battery_refresh_scheduled:
            return
        self._battery_refresh_scheduled = True

        def refresh_battery_settings_once():
            self._battery_refresh_scheduled = False
            ignored = self._is_ignoring_battery_optimizations()
            refresh_plugin_settings(__id__)
            if not ignored and remaining > 1:
                self._schedule_battery_settings_refresh(remaining - 1)

        run_on_ui_thread(refresh_battery_settings_once, 1000)

    def _read_status(self, show_error=False):
        raw = self._invoke_core("getStatusJson", show_error=show_error)
        if raw is None:
            return {}
        try:
            return json.loads(str(raw))
        except Exception:
            return {"summary": str(raw)}

    def _read_history(self):
        raw = self._invoke_core("getHistoryJson", json_compact(self.settings_payload()), show_error=False)
        if raw is None:
            return {}
        try:
            return json.loads(str(raw))
        except Exception:
            return {}

    def _last_run_subtext(self, history):
        if not bool(history.get("has_history")):
            return ""
        copied = self._safe_int(history.get("copied", 0))
        skipped = self._safe_int(history.get("skipped", 0))
        pending = self._safe_int(history.get("pending", 0))
        parts = [f"Copied: {copied}"]
        if skipped:
            parts.append(f"Skipped: {skipped}")
        if pending:
            parts.append(f"Pending: {pending}")
        return "Last run: " + "  ".join(parts)

    def refresh_progress(self, _view=None):
        refresh_plugin_settings(__id__)

    def _safe_int(self, value, default=0):
        try:
            return int(value)
        except Exception:
            return default

    def _safe_float(self, value, default=0.0):
        try:
            number = float(str(value).strip().replace(",", "."))
            return number if math.isfinite(number) else default
        except Exception:
            return default

    def _safe_bool(self, value, default=False):
        if isinstance(value, bool):
            return value
        if value is None:
            return default
        text = str(value).strip().lower()
        if text in ("1", "true", "yes", "on"):
            return True
        if text in ("0", "false", "no", "off"):
            return False
        return default

    def _phase_title(self, phase, running):
        names = {
            "idle": "Ready",
            "scanning": "Looking for messages",
            "copying": "Copying messages",
            "copying_comments": "Copying comments",
            "preparing": "Getting ready",
            "downloading": "Downloading files",
            "sending": "Sending messages",
            "cooldown": "Waiting",
            "canceling": "Stopping",
            "finished": "Done",
            "failed": "Could not finish",
            "canceled": "Stopped",
        }
        if running and phase == "idle":
            return "Starting"
        return names.get(str(phase or "idle"), str(phase or "idle").replace("_", " ").title())

    def _peer_text(self, value):
        raw = str(value or "").strip()
        if not raw:
            return "Not selected"
        try:
            if self._load_core(show_error=False):
                label = str(self.dex.call("getPeerLabel", raw) or "").strip()
                if label:
                    return self._short_text(label, 90)
        except Exception:
            pass
        return raw

    def _comments_available(self, source_raw, target_raw):
        cached = self._comments_cached_value(source_raw, target_raw)
        if cached is None:
            self._schedule_comments_settings_refresh(source_raw, target_raw)
            return False
        return cached

    def _comments_cached_value(self, source_raw, target_raw):
        source_raw = str(source_raw or "").strip()
        target_raw = str(target_raw or "").strip()
        if not source_raw or not target_raw:
            return False
        key = (source_raw, target_raw)
        if self._comments_available_key != key:
            return None
        now = time.monotonic()
        ttl = 120 if self._comments_available_value else 10
        if now - self._comments_available_at >= ttl:
            return None
        return self._comments_available_value

    def _clear_comments_cache(self):
        self._comments_available_key = None
        self._comments_available_value = False
        self._comments_available_at = 0

    def _sync_peer_settings_cache(self, source_raw, target_raw):
        key = (str(source_raw or ""), str(target_raw or ""))
        if self._peer_settings_key == key:
            return
        self._peer_settings_key = key
        self._clear_comments_cache()
        self._schedule_comments_settings_refresh(key[0], key[1])

    def _schedule_comments_settings_refresh(self, source_raw=None, target_raw=None, remaining=12):
        if self._comments_refresh_scheduled:
            return
        if source_raw is None:
            source_raw = str(self.get_setting("source_peer", "") or "").strip()
        if target_raw is None:
            target_raw = str(self.get_setting("target_peer", "") or "").strip()
        source_raw = str(source_raw or "").strip()
        target_raw = str(target_raw or "").strip()
        if not source_raw or not target_raw:
            return
        self._comments_refresh_scheduled = True

        def check_comments_available():
            available = False
            try:
                if self._load_core(show_error=False):
                    error = str(self.dex.call("checkComments", source_raw, target_raw) or "")
                    available = not error
            except Exception as exc:
                self.log(f"comments availability check failed: {exc}")

            def apply_comments_result():
                self._comments_refresh_scheduled = False
                current_source = str(self.get_setting("source_peer", "") or "").strip()
                current_target = str(self.get_setting("target_peer", "") or "").strip()
                if current_source != source_raw or current_target != target_raw:
                    self._clear_comments_cache()
                    self._schedule_comments_settings_refresh(current_source, current_target, remaining)
                    return
                self._comments_available_key = (source_raw, target_raw)
                self._comments_available_value = available
                self._comments_available_at = time.monotonic()
                refresh_plugin_settings(__id__)
                if remaining > 1 and not available:
                    run_on_ui_thread(
                        lambda: self._schedule_comments_settings_refresh(source_raw, target_raw, remaining - 1),
                        1000,
                    )

            run_on_ui_thread(apply_comments_result)

        run_on_queue(check_comments_available, PLUGINS_QUEUE)

    def create_forwarded_content_settings(self):
        return [
            Header(text="Message types"),
            Switch(key="type_text", text="Text", default=True),
            Switch(key="type_photo", text="Photos", default=True),
            Switch(key="type_video", text="Videos", default=True),
            Switch(key="type_gif", text="GIFs", default=True),
            Switch(key="type_round_video", text="Video messages", default=True),
            Switch(key="type_file", text="Files", default=True),
            Switch(key="type_voice", text="Voice messages", default=True),
            Switch(key="type_music", text="Music", default=True),
            Switch(key="type_sticker", text="Stickers", default=True),
            Switch(key="type_animated_sticker", text="Animated stickers", default=True),
            Switch(key="type_animated_emoji", text="Animated emoji", default=True),
            Switch(key="type_location", text="Locations", default=True),
            Switch(key="type_contact", text="Contacts", default=True),
            Switch(key="type_poll", text="Polls", default=True),
            Switch(key="type_todo", text="Task lists", default=True),
        ]

    def create_copy_settings(self):
        source_raw = str(self.get_setting("source_peer", "") or "").strip()
        target_raw = str(self.get_setting("target_peer", "") or "").strip()
        comments_available = self._comments_available(source_raw, target_raw)
        date_mode = max(0, min(2, self._safe_int(self.get_setting("copy_date_mode", 0), 0)))
        items = [
            Header(text="Copy settings"),
            Selector(
                key="copy_date_mode",
                text="Date filter",
                items=["All dates", "Single day", "Date range"],
                default=0,
                icon="msg_calendar2",
                on_change=lambda _value: refresh_plugin_settings(__id__),
            ),
        ]
        if date_mode == 1:
            items.append(
                Text(
                    text="Day",
                    subtext=self._date_setting_label("copy_date_day"),
                    icon="msg_calendar",
                    on_click=lambda _view: self._open_date_picker("copy_date_day", "Select day"),
                )
            )
        elif date_mode == 2:
            items.extend([
                Text(
                    text="From date",
                    subtext=self._date_setting_label("copy_date_from"),
                    icon="msg_calendar",
                    on_click=lambda _view: self._open_date_picker("copy_date_from", "Select start date"),
                ),
                Text(
                    text="To date",
                    subtext=self._date_setting_label("copy_date_to"),
                    icon="msg_calendar",
                    on_click=lambda _view: self._open_date_picker("copy_date_to", "Select end date"),
                ),
            ])
        items.extend([
            Divider(),
            Text(
                text="Message types",
                icon="msg_forward",
                create_sub_fragment=self.create_forwarded_content_settings,
            ),
            Switch(
                key="show_sender",
                text="Include sender",
                default=False,
                subtext="Keep the sender name on copied messages.",
                icon="msg_contacts_name",
            ),
            Input(
                key="send_delay_seconds",
                text="Send interval",
                default="1.5",
                subtext="Interval (sec). Increase if catching floodwait.",
                icon="msg_contacts_time_solar",
            ),
            Switch(
                key="quiz_auto_answer_unknown",
                text="Auto-answer unknown quizzes",
                default=False,
                subtext="Temporarily vote in source quizzes to reveal the correct answer and copy them correctly.",
                icon="msg_retry",
            ),
            Switch(
                key="quiz_send_explanation_spoiler",
                text="Quiz explanation separately",
                default=False,
                subtext="Send quiz explanation text and media as a separate spoiler message.",
                icon="msg_discussion",
            ),
        ])
        if comments_available:
            items.append(
                Switch(
                    key="mirror_comments",
                    text="Mirror comments",
                    default=False,
                    subtext="Copy channel discussion comments.",
                    icon="msg_discussion",
                )
            )
        return items

    def create_settings(self):
        source_raw = str(self.get_setting("source_peer", "") or "").strip()
        target_raw = str(self.get_setting("target_peer", "") or "").strip()
        self._sync_peer_settings_cache(source_raw, target_raw)
        source = self._peer_text(source_raw)
        target = self._peer_text(target_raw)
        has_source_and_target = bool(source_raw and target_raw)
        status = self._read_status() if self.dex else {}
        running = bool(status.get("running"))
        canceling = str(status.get("phase", "") or "") == "canceling"
        history = self._read_history() if has_source_and_target and self.dex and not running else {}
        battery_ignored = self._is_ignoring_battery_optimizations()
        start_text = "View progress" if running else "Start copying"
        start_icon = "msg_stats" if running else "msg_retry"
        start_item = {
            "text": start_text,
            "icon": start_icon,
            "accent": True,
            "on_click": self.start_mirror,
            "link_alias": "mirror_panel",
        }
        if not running:
            if not has_source_and_target:
                start_item["subtext"] = "Choose source and target first."
            else:
                last_run = self._last_run_subtext(history)
                if last_run:
                    start_item["subtext"] = last_run
        items = [
            Text(
                text="Copy from",
                subtext=source,
                icon="msg_forward",
                accent=True,
                on_click=self.choose_source_dialog,
            ),
            Text(
                text="Copy to",
                subtext=target,
                icon="msg_forward",
                accent=True,
                on_click=self.choose_target_dialog,
            ),
            Text(**start_item),
            Divider(),
            Text(
                text="Copy settings",
                icon="msg_settings",
                create_sub_fragment=self.create_copy_settings,
            ),
            Divider(),
        ]
        if not battery_ignored:
            items.insert(
                4,
                Text(
                    text="Disable battery optimization",
                    subtext="Recommended for long media copies.",
                    icon="msg2_battery",
                    accent=True,
                    on_click=self.open_battery_settings,
                ),
            )
        if running:
            if canceling:
                items.append(
                    Text(
                        text="Stopping...",
                        subtext="Finishing the current message before stopping.",
                        icon="msg_cancel",
                    )
                )
            else:
                items.append(
                    Text(
                        text="Stop copying",
                        icon="msg_cancel",
                        red=True,
                        on_click=self.cancel_mirror,
                    )
                )
        elif has_source_and_target:
            items.append(
                Text(
                    text="Clear copy history",
                    subtext="Allows copying the same messages again.",
                    icon="msg_delete",
                    red=True,
                    on_click=self.reset_progress,
                )
            )
        return items


# __DEX_BEGIN__
# eNqsnQmcjeX7/6/7Wc45s585Yx3LjLENWWasZ8pOoeyOsrRIJRXDIEtahGxRiJK0E5W+0UpJKq1oValEUVGpJKXd/3Nd93XMU1/+v9/r//rX6+1z3ctzP/d9P/f+nHPm4ksmpRY1b0mbZ737V+svElU31l938NlLe/Z9Y/7aDVuiz5VRF4fGENGks1vESP8r6uzQ/BCJf8wQrcglWgU9WJPoL+jmZkSTXaLVnYg6I86Y83HNp4beHk509UJD14LrwQ1gLpgPFoI7wArwKHgevAl2gi/BT+BPEFlkqBqoCxqDduBM0A8MBCPAVWAGWAmeAbvAIeDdYigX9AKXgzngPrAJfAyOgJTFhvJBd3AFmAkeBK+An0FsCe4LzgTngOvALHAHuB+sAevBa+Bd8BU4DHJvNdQJnA66g56gHzgHnAcuBpeDMWACuApcA64HN4DZYB5YBJaCu8FK8DBYBzaBl8Ar4HWwHXwI9oB94AD4HhwBvwO6zVAYZIAKoAYoAIWgKWgNOoDTQTcwEIwEU8FisAJsAG+CL8DPnNZSQzVBCUiA2eBRcBCk3G7oDHApmAeWgxXgcfA++BY0XWaoIxgKrge3g4fB6+Bv0OgOQ2eBy8BMsAq8Dr4D4eWG6oPTwUhwC3gOfAWy7jTUEgwAo8AYMAPMBvPBCvAseAvsBEdA5l2GGoLO4DxQCq4FK8Bz4CCI3Y22ApqANuBcMAbMA3eAu8D9YBVYA54Em8CL4FXwJvgY7AH7wNfgB3AY/AL+AHQP2i1IAZkgD7QArcFpoAPoCnqCfmAwuBRMBNeAGWAeWAzuBCvBI+Bx8Bx4AbwCtoK3wQ7wEdgDvgTfgkPgF+DcaygHVAU1QR1wCmgB2oBOoDs4FwwDl4M54GawGCwD94AV4HnwLvgDVLvPUDE4A1wObgB3g+fBHvAHqHu/od5gMrgfbAdHQa0VaBNgJngB/AmarzQ0GjwMDoD8BwxdCO4HX4C6GKyGg1XgAGiwGvcFD4OD4JQHDV0EloHXgfuQobZgIlgHvgV1H0a7AIvBB6DSGuQDTAVPgu9A7UfQv8FtYDtw/4PnBsaAB8AHIP1R1BuYANaAfSC2FmMUmA7Wg+9A/jpDg8D14HHwITgKch9DfwMDwEgwE9wK1oKtYD8wjxuqAlqCruBcMAHMAfeBJ8E2cBhUfALtHvQA54MxYAZYBtaDbWA/+ANkPWmoHugIzgOTwV1gI/gY/ARSn8IzAi3BWeAicDVYDB4Az4K3wZfgdxB9GuMQaAW6goHgMrAArAOfgV9A5nq0P3AauAQsABvB5yCywVARGACuASvARyD7GeQBjAMrwbsg91ncB8wEW8CPoNJGpAsuAHPAa+BHUO05tFEwEiwEa8E74BdQbRPaPhgLbgObwX6Q8TzaNhgC5oEnwRfA34zxCvQEpWAJeA58CXJewLgDJoO14AuQ9yL8wDhwD9gJMl5CvYJhYBZ4GLwHaAvGJDAAzATPgO9Bg5fx7MF8sBH8AApeQRsFs8DT4HuQ9yrqHcwEG8F3oP5rKANYCraBY6D56+g3YCXYCVLewH3BQHA1eAC8A8xW1AE4G0wBd4EXwT4Q2YZxHQwGE8AKsAuEt2NMAVeCVeBr0ORNQ+PB0+BPkPYWxiHQCLQCncENYCtw3sbzAMPAODADLAR3g3VgK9gLfgXZ76B/gQYgDrqD88DFYByYCm4GS8Ea8CJ4H3wJfgH+u+g3IA8Ug4vAEvAS+A3Ufw/+oARcDEaBxeBB8CTYAt4Cv4DGO3BvMB+sBYdA0/cxXoHl4F0Q+QB9DkwAT4AfQdMPMeaBLeAD8DvI22moCxgBbgZPgE9A+CPM86A7uB68BD4HDT821AdMAKvAW+Ao+AM4nxiqDvJAbVAImoBTQQfQDfQC/cFgcAEYASaBaWA2WAzuBCvAavAIeAxsAM+BF8Fr4C2wA3wC9oGDwOzCuAiqgHxQCJqC1qAzOBP0BgPAEHABuBhcBZaAp8EzYBN4CbwO3gSfgr3gR2CwRs0AuaAxKAHtQT8wHFwJpoHbwJ1gBVgNHgHrwFPgOfAieB28Cd4Hn4A94BDwdmP+BNVBbdAQNAPtQQ8wCAwH48BUMB8sA6vAE+AF8Cb4BOwHP4FjIG2PocqgNigCHcGZ4AIwHlwN5oH7wWqwFmwC74HPwSHwF0j7DHkDp4AWoAR0Av3AMDAeXANuAHeAR8FL4C2wB3wLDgPzOcZoUAMUghagM+gOzgHDQCmYB9aA9WAL2AMOg79AdC/mDtAUtAXdwEAwFIwFU8FCcA94CDwJdoBPwS8gvA/9EtQEDUERKAEdQG8wGFwIrgDjwNVgOrgFLAMPgU3gbbAbfAV+Bd4XmCNAQ9AVjAMLwTLwEHgavAI+AJ+Dn8HfoMmXaJ/gAjASTADXgqXgfrAGbAJbwUdgP/gTRL7C/UAdUATagt7gHHAhGAuuBnPBEvAQeBy8DN4Fn4NvwO/A34/2DRqAdmAAGAauBNPBLWA5WAOeAVvBx+A78DtIO4D6BI1Ae3A66AUuBZPANDAf3A0eBE+AzeBlsA18BPaBX0Dq1+jTIA+0AGeAPmAIGA+mgpXgMbARvAl2g+/A7yDnG7Qv0BicCrqABLgMTAKzwSKwDNwLHgVPg7fBt+APEPsW/QZUB7VAPXAKKAItwamgPTgdDAGXgmvBTLAEPAA2gW1gJ9gN9oOjwD2IPgDyQFNwGugEBoOJYBZYBJaDlWAdeAm8Ad4BH4O94DeQ+h3SAa1AJ9AXDAWXg3HgWjATLAN3gxXgSbAN7AHfg19B7HusVUAhaAY6gQvABDAH3A4eBpvAVvAR+AE4P2DcBY1BO9AbnA1GgCvBLHAreBA8CV4A74Cd4GtwFHiHMPaBOqAJaAfOBEPBGDAVLAIPgsfAC+Bd8AX4Gbg/Iv+gLmgK2oIeYAC4CIwF14NbwF1gFVgPXgBvgk/AZ+A7cAT8BSKHsf8FlUEx6A4GgilgFrgNPAgeA5vAB+Ar8BP4C6T9hPUDOAV0AsPAFLAYPAw2gs3gFbAVfAy+BD+A30DGEcxpoB5oAlqDbmAImADmgjvBOvAyeBt8A34FqT+jL4Ji0BYMBpeAseBaMA8sAf8BT4ItYBvYA74H5hfcGzQGp4JuoC8YBkaCyWAmWAiWgfvAI2AL+Ah8D/4EsaPoO6ARaAe6g75gGBgHrgfzwa1gFVgLXgA7wV5wABwCvX9FWcEh0PY3rFXBFHArWAdeBV8B/3e0IdARDAATwFLwDPgIHAUZf6A+QQ9wOZgKtoC9oPBPjHXgJrAcrAaPg/XgefAqeBPsAz+B34D3F8YbcBqYBO4Fq8ArYAfYD34EvwHvb7QpUBs0Bq3BCLAA3AruAA+ANeBR8DzYBj4AX4O/QOwY1tegMzgbXArGgOlgAVgCVoHHwBbwDvgM7Ae/ggbkUE8wCiwCd4LVYC14FrwP9oGDIGIcqgSago7gLDAIDAdlYAK4FjwIXgDbwR7wCwg5DlUGeaA+iIP24HTQA5wPrgBl4CowAywAq8BmsBXsAj+Av0HIdSgT1ABNQXNwGugNBoMRYDy4GswFt4P7wCPgSbAJfA2OgjTPoZZgGJgPNoI94BtwGKT4DkVBLmgJOoAzQA9wLhgJJoKpYA5YBlaD58BWsAd8D34GqAgKg4qgOmgATgVngD7gbDAR3AGeA++AP0CNsEOdwShwI1gPdgE3gnoEl4LrwK1gHXgVfAi+AT+Dv0F6ikPVQCPQHJwJzgejwTXgLrARvAo+APvBH8BLdagCyAOtQUfQBwwCQ8ElYBS4CswAN4F7wENgLXgNHATRNJQX9AfXgiVgHdgKvgGHwW/gGEhJxz1BLdAYlIDOoCe4AIwH08BccCd4CGwEH4CD4GeQmoE8gzjoAwaCi8CV4AZwK1gNHgM7wX7wEzgGsjNRV6ARaAt6gGFgNLgOLAErwCPgGfAq2AG+BD8ANwv1DWqDVqA7OBsMAcPBlWAOWATuBg+DTWALeBccAD+BP0EoirYO8kBr0AWcByaA6eAO8Bh4DewFf4OcbNQdaALOAP3BFeAasAisAuvAZrANvA++AkeAG0M/B9VAQ9ASdARng6GgDEwDN4E7wBrwGNgAXgQfga+Bk4N8gFzQCHQCQ8DlYDyYBW4Dd4I1YBP4EHwOjoG0ChgrQAvQFZwPLgVXgqvBTHALWA4eARvAZrATfAecimgDIAaqgVqgCJwGzgC9wQBwOZgKFoLbwYPgKfAy+AAcAKYS2iPIA83AmeBscDGYCuaDteAp8AJ4HXwMfgBHgamM8RNUBAWgCJwK+oIrQBm4C7wI3gK7wTfgV76uCq4D1UAD0BkMBqVgHJgMZoL5YBV4H/wBvKpog6AOKAbtwJlgILgUjAHXgJvB3WAleAw8C14G28CHYD/4A1TOdaguaAM6gx5gADgPXAxKwVVgOpgLloCV4HHwIngd7AAfgq8BVcO4BopBT3AhuApMBYvBWrAFfApC1VH/oBqoB5qBrmAkGAOiNTDmENEz4DVgDJEDXOABH4RAGERACkgF6SADZIIsEAXZxr6vygEVQEVQCVQGVUBVkAuqgeqgBqgJ8kA+qAUKQG1QB9QF9UB9UAgagIbgFNAINAZNQFNQBIpBM9ActAAtQSvQGsRBCTgVnAbagLagHWgPOoCOoBPoDLqA08EZoCvoBrqDM8FZoAfoCXqB3qAP6Av6gf4gAQaAc8BAMAgMBkPAueA8cD64AAwFF4Jh4CJwMbgEDAeXghHgMnA5uAKMBKNAKRgNxoAyMBaMA+PBlWACmAgmgcngKjAFXAOuBdeBqeB6MA1MBzeAWWA2mAPmghvBfHATuBksAreAxWAJ0NddtBTcDpaBO8BycCe4C+irFroX3Afu5/eaYCV4wNj3m6vBg+Ah8DBYAx4B/wGPgrVgHXgMPA6eAE+Cp8DTYD3YAJ4Bz4KN4DmwCTzP703BC+BFoMe49DJ4BbwKXgOvgzfAVrANbAdvgrfA2+Ad8C54D+wA74MPwIdgJ/gIfAw+AbvAp2A32AM+A5+DvWAf+AJ8Cb4C2PoTtuyEbTdh20zf8vtegC0lYXtI2N4RtmiELRVhW0TY2hC2KIRtBtaJRFjeE5b1hOU7YXlOWIYTltXyvhjLXcKSFR0b/RtgGUhYvhGWaKRLK8KSiLDUISxbCEsOwlKCsBQgTOmEaZkwnRKmR8I0R6PBXeA5sAk8DzaD+/T99cEZhu5X+whsHluMujeq/Sf8n1PbQ6PbpHY67OfVrg57i9qNAnaLgN0G9ma1u8B+Qe0esF9UOxGIMxT2S2qPCPhPDVw7O5D+goC9NBD/noC9OhBnbcB/PeyX1d4c8H8N9mv2cch/b6j9Nvy3qr0T9ja1uVzb1f4s4D81YK8O2AcC9vqAfQj2O1BX78u2r/YHav+qcSKBOCmBOCkaZxe0EnvOxJ4McgqIwt7Pz0rzfJifFT9HjdMaFMLeCS3hMNgfqd0D9u9qj9R02oCrYX8B7aB1cpDsZx5mwP9HtecF7MWw9/L9wXJN5wytq1+gXfkZwf+o2htn2vJ21zJy/N4a/ze1X0Ocv9TeofH7BupnoNrvqs1t+wO1dyH+h2p/oWUZqGXZr/YR+P+ZvFbvxXZkVnmc1fpcBiU/JMLzhrYrLvuFXP+zbJxhGuVvtauo/0WBfI7QfO5Xuw7i7INezs9P448MxB+p8fepzXG+VLuF5pPtNrNsfkZq30zaXQL+PQLxu2i5RgfuNVrv9Z7aPD58oDa3vY/VTswq9+d6/lztoVqW0ZrP/WqPDNjjZ9l6ZvvqgH+bgM15O6T2DM0/2wsC1y4NxL8nYE+9odzm53tA7dWBOGsDaa4P+G+GzfUwJlDnYwJlGRPI5xjNJ7eHMdoeuD7HBtrzeI3D97qS27M+3ymBOp8SqPMpWuc71H4b8d9Xe9csW/9sH5hl2/MUbc9H1D6ieZui4+2BZJpYVHytdvpsmx+2qwTsfNg/q1042/ZBtuOzy9PsMtvm/5pAH7w2UN7rA+1qmvp/pXYPTWea5i1pc/s/rHYiGEfvNSNwr9mB/sX2CI0/W8crjjMvkJ95gec1T/OWtDk+1//8QH+cr3EOqN1Dx735Ov7/qPb42bYdzg+0w/laFk7zJk2T7cVq87VL+L6zrf8y9f9E7Rnw/1TtBbA/U/ue2bZ+lul4zmVcHmg/ywP3ukvtH6B3czvHtX+o/Tbs79VOn1Nu14H9q9ptAv5D59g2wHuUSXNs+vcG7nWf2pyfVYFntCZQ/2t0/GT7ES7jnHJ7Huzv1F48x/a1/2hf+1Hte+bYuYPt1XPsWM32E3PsfR/Ve32r9mZN/1FdJyTtZB7Wqj/3o8fANo3P9k61H+dnHbAPaN6eDIwDTwbiPKnzPufhqUAentL7sv/TAX+2D82x4+cGHT+/VZvjfKP2rxp/QyD/G3TsOqC2N7fcTp9r42zU+Oz/nPb9/Wqv13Q26ZqE77s5kDe2K8y19c929bnl/hx/D5TXQ3U0Dq9FWmic3Tr2ct1yvDbw3612N42zR8eQH9UeONf2oz06v3OczwLpcLyhei+ZwzQdbhPj59r8/xbIP9tXa5w/tb19p/biOeX+nD4/07/0mfJ1xzg+rv1J7QWweb2drgvoMWpzmy9Tm8fqsWovR/xxaq+GPV7tzQF7G+wr1d4Je4LaPLYn4xyA/0S1j8CelLwv7MlqR24st3leTtpR+E9Ruwrsq9WOw75G7TaBdHrA/1q1E7CvU3sk7KlqXx2412zYV6m9APb1anMdTkuWC/mZofY9iDNdbV6r3KD2mhvL7ScC9saAvQX2rcl6g32j2jtgL1L7i4B9CPbNybqCvSRZV/PK06wAe47a1QN2nXnl6TQK+Mdh36Z2h0A6PQJxEgF/Xtsk/XmvlLTPDcS/eF55ua4OxB8ZTCdgtwnU20a0vdnJawNpzp5XXvYFAXt5IJ0ugfysDly7NmCvD9ibYS9V+7WA/45AmkMDae4KlOuLQJyDAbtOoCxHAmn+GYjjzS+30+eXl6XC/PL4+QG7cH75sysK+LcJ2F0CafYI+CdgL0w+I9i3JJ9F4L5XB+zZAXtxIJ17AvZq2Dcl6zYQ/+2AvQv2gmR/D+RtaqA+ef+bjM/z78xkfw/E54OhpB25qTx+NOBf/aZAOw/4Nwr4t4A9N1lvsOer3S0Qn+edxWqPuan8Wc8O2Atgz1N7Kezbk20JZVmWrJ9Amrwfn5VMH/58tpGh4+3davN4e4/aPN7eq/bGm8rt12DfpzaPmfervQP+q9Q+BHuF2pGbDa1UOwr7AbUb3Vweh8fS5LU94P+Q2hfDflht3gcl4/M8vlrtqxHnQbUXw35U7fWw16r9Nuz/qP3rzeX3qrDA0JpkfmA/onafBeVxJgXiLAjEeSIQZxfsdck6gf1Ysm4XGno8mT7y/4Ta6fB/MpmHeeU2792O+wfi8Fj3lNrnBuJXD8RpE7iWx66kzWPR02rXWVieDu/v1qu9NJB+USAO71mS6fDYkvSPB+7L/Sjp32FheZrdYG9I1mcgPq/lkjavD5P26oC9NlCWcwPXdgnkZ3MgzxcvtGvUTFO+Hs405fsRtkcutOs0tscvtP58pr8LdRWlRrTCJYrRjapzaKXLZz+e+dBjzTCDQkRVsVt6B/41sIOt5bAOpV4+UU1sTPm6mti5rRR16AHRDLNKdbXqg6LL6CFonl6Xh10AX5ev7nyspK17tKSTj/RWiS6j1aoPimaYh/S6h9W9BlpL06ml6Raou0DdtdVdW9111F1H3XXVXZdCxrpteerivqtUV4va8tTT+PX0+vrYia0QfVTdy+T6QvUvxO5spaj1P4U60x++1T9V80NWC0J81ob9iMM6msa7rA5d6Vr3RNVJ6j9ZNMP85FkN+1YnqU72bbxX1H3At9dTyGqh6DTqAW1MzSW/jWmU5Lcxnu8DojdKPTTR8CYa3gT5fEDUPvcmiMf11FTjNYV7pegoiddU0ymCe4WoDS9C++PwYq1X1pWiy8W/mabXTOM3Q37ZvzlNp0aihi4TdWiWaIaZp7pO4z0mOo2eEJ1DqzzrfkZ0JD0vOoVCvvVvonqeb68fqjrbt/d5xbfpp4VYSygjZOPXDPH5aGea67Iuo31ItyW1kvy3RH2vFM0wD6hyfbTCdRVEbTlaab5bIT8HPNaR9LVn3Zw/Dm8e4rPXNnTQYZ1C3znWPdW1mkB4HPHO8lhtOiVIv9ix2ky1uWoL0dl0mmvdl7vWPVV1nehouk/T+dS3yuVG6U1F0WlUN2T926m2h56G/L9FrBnmU4d1LDUN8VmwQ7U91inUDtoO6XM9taOBUk8dkF6x6GzqLDqQnhY19J7HOoVSfKvFvo0/wLfxL1b9Tv2/V/1B9S/V2shHR1xf12U19Df8O6Efz3dZ7f06YRfM42InPM+IhG+kuM9n049QTZfVtrvO2LF38Fj/pHNDfF49mqq4rFMoVzTDNFT3+aLL6B7R3nSvhhd5rL75VHQPtfVZDQ0O8Zn3MrrTt3qXz2fdb9GdLusa+g/c3bQ+uyF+G4/Pvw3Vdq12Vr1CtVR1tOpU1XWi9jmztvBZM8wG34a/oPqp6ne+jVclZN3tVNurdoCeSTdRR4+1g7SXM1FDNUSnUR70LOolz/0snYd6qLvncbX+vbididp+0kv7SS/tz720f/TS/tEb7jtc1tHUGO4+aA+jiNWhSaIYb9S9Ut0PqHu16oOqD2n4w6prVJ/W8PWqb4uOlH0+u2OO1RzVAtXaqnUcm06huhuoNlX/ItEMk+Nanan6tMfanWJSrrE0AzqRJtAoxJ+J/0dC5+L/CqpNoAvIpZ9Ew3QEehvSaeLx+R/W3y5rhikT91P0pfqfGeJzQEPvenwGiDWOa/UZ1VPV/zTVr1T3e3zWl2FOF/2MTgvxeV+GaeRZvVb0Nsr1WUdTfZ/P/Qxtga6gWyjfY51EV/t8jvc6/eGwZpgRrtVmHqsj7Zzdn4k+Tt+om+Q6tN8Qn+s59LDH53VT6KjDauhal3U0Pau6UTTTPOfa8NfUf4Nn9VtRh771bTrNQtZ9fsjGLw1Z/7EhexZ4i8tqqK5ndQL810Gf9Vgxn4fs2eCLDquhjxzrfsDjs0H+LB+f/5Vrnirf7ynk52eH1dA4l9Wh2aK2vT9Fz9IO9a/j2XhtRO3zeErHM05nn/pzfT2F/ljLt9c3Es2g3qLP0D2+Tad7iM8ZsR/xrKbDf73mZwP0G1E7frL7FdHHaKe6V3pWW/usdhxl93DRDOOHrLuFqC0vu69QHak6KsSfb8E+zbd6n8/nk3a83ajPb6PedyOPBx7rFDro83mlfS7P4Uls9Kz7W/Xn57QJ8Q7xe3C4G7us6CeqI1zr/5qoQy0869/Ss+426j6qmuVbvVT0Fxqj7jLVx1S/gT6vz+V5tMNK4v5Fxq3Ner/NyNdan3U0HfWt+1fRDfQb9EX4t1Gdo8rXb4G+5LDafrJF6/dltLTqSPeV4zpF8v8q3Ma1Wgvu13HdOMTfiusugG5DvHyX9Vkq9K17aIjPb23/2q7tcrv2s+3az7ZrP9uu/Wy7lov913tWN6h+K2r723btb9u1PWzX/rZd+9tbWs53tD+9o/3pHU3/He1X76GmeT7doe11h+Zzh/afHdpvdmi/2aH9Zoc+lx3aX3Zo/3hf+8H72g8+UPcH6v5Qr/tE2+Eu1d3I932iDoVCfEa9gX5xWG3+9uCJ8nP+DPHiLqstx2f0m4yje7Wf7dX09mr/2avPda/2k73aT77S5/UVRsL3Xevm57Vf/ffr+nC/+h/Q+x3Q8h+gJ6QcBzAC8fP+Gv6/eKzW/Y2OT9/C/0HPaqbPasMParv6Tq/7Tv2/w3X9oN9rP/per/9ew39Q9w963Q+a7g8afkj9DyH9yuK2/j/qOPCjPt8ftTw/an39qOPBjzoeHNH+dkT72y86jv6i7eqojmNHtX8f1XGMtZXPOoUehf6q9fUrruTn8Bv813is1v2H1jfrQHVzff+p6f6p6f2F/BlVx+d3Bva6Y/qcjul1jrHtxTEO9Re17cY3GbTH48+E1KZPXNZBVMXjz4Icpo/580E03Fyg7ps5Pg0zX/i8JaxLlyJ+GDtb8vjzIhOoqsefD0k1fF2KXpei16XodanY2f4J/zS60QyBpiN8CL83ov70kcuaUM0wl/usqWYxNEP7dwb1oxEoTyZu+pLH2tywRinbcDpRpPck4mVrutm4cys5t2gv4TEaaDg8R8NzNLyCuiuou6K6K6q7Eg2V6yurf2UaSj+6rBfT6SHWS+lCaBUNr4KWebHLOpye9fl85DANJqtDRNvQNof1TdoueqsJu6wuVRNNpzquDe8h2oR6i55PL4heT9tEjdkuOs68JTrRvC060tTzWJtSZ8+mf5boTeZs0TJzrujt5iLR8eZx0cW0RXQ3vSy6n17xbH4v0HI8IDrWvCE62WwVnWC2i44yb4pOMu+LXkEf+/Z8aL/odPpF9Gpy5dxoKnkhW+4mohfQxdBcradcjMg9XdaXqZfoDjpH9C2p31weAUWXm2Ee6xaa57Paeq+m6VTDDPeDy/ouGY/1aXJEa5Iv+gFli9aimOhdpqnoFaaf6FJzseit5ipodU23OmaKvi5rR7pL9AF6XPQKelf0VnOFZ3Ws6J1momgp7Rb9jPJ91k40WPRaGuLz+ZlNvwZ63BdynhaiL0VXkSvnbG3JE73CpIvuo0LRndRU9HrqqtpN9AXqru4zVc8S3UZ9RG05atCX1F/0E0qIfk0DRBvQMNFWUu81MD5eIppDN6nerLpA1NZHDeyj7hddKfVSg86hp0Tfp4oe6wz6XHQX7RWtRCW+1dNE55keovebnqLDTS9RW1816AuaL3qL2Si6l94W/ZzOCFntKuqabqIf0QDRAjpb9FM6R/QlKgvJ+aT085o6juSRJ+48dedjxGR3vrpraXgtdReou0DdtdVdW9111F1H3XUpRdx11V1Pw+upuz71End9dRdST3EXqruBtpMGNJ+Gu6zjzJuijpnrsfYyN4v2NAtEPbNYdYnqraq3iaaYpaK+Wab+d6guFx1vnhQdTafK/a80K0SvNy+LXm1eFb1GdYp5TfQ6s010quoo85boVWaH6GjzgegY1WvNh9CGWr6GaOcZLuu3dLboV9IOG9Jp9KT6+3I+Wk3qp5Fe1wgjwT45Hx1ME+T88xR6T3S4KZRz0OWmp+pbotWprpx73mTOCtl0LpLzzgqSbmOt9yaUI+4m6m5KFcXdVN1F1EjcReouxkzP7mKdf5pRJQlvpuHNMYOzuznNo90O69nmsOhw85vq76K/0zHRwxR1rTYQ/YlOEW1hTlX/oarTRP+gOaKTaZHoInOb6GVmqWipud2ee6pOVz1G/1Fdq+m9KLrc9PCsjhE9xcwSbWtmizYwc0R/o4Wijcwi0XbmXtGr6HU9R7V6Db0hejP9odpX6qU3Xejbct8ierlZJVpo1vs2Pz+LVqBj6q4k56su1QlZd7Fqf9EFNFnOXatL/bekyqr2ObQiEncramn2OqyXSr230vprhXF8sJyvDpf1SytqZmaK3mbeFF1g3hZ1qKlvw4eJ9pF1DF93q2/TWSdq89tK89ca6x2+f5wuEi3Bc7XqUGvXuktEj6jeQm1Ej9F16r5etLF5RP0fFT1M7eU89lIzQHQItfSt/wjVlb5N/0vReVQP+TlV+9GpWj+nQduS1cGqQ1Rf9KxepvHCIasV5PzWlovPcV92WYvpJcRrq9e3paiEt6MmqnPNFtR7ew1vr+uwDlRbwjugnR9wWM+lIvGfTF1E08wM0cN0g2hnWi96qakv58AjTAPR4aahuvt7Nnyk6EwzRfRmM080QvNFm9Fjoj/Tq6JH6TXRKmar6jbRqma7Z+/vyrnxdZQq6lCRb/PZTLSjuV60i5km2slMF+1sZqj/TNVZonEz17f3X+Tb8t/u23I859v7bRL1qFaI1aUGordTK9F7qLVqieqpcp5t23sn6ifaWeu7sz6PLnS+6Olor6d4rK3pQTlX7iv+XTV+V43fDTuTIfJ56sOq1r87WlCqnON2FfeZmt6Zmt6ZtILqy/lunoSfpe2tB9UUdw9199TwnuruhfB2xGr7SS/473dY7XjZS/tvLx0ve1Ebc5dn9R7P+i/1rXI/7KXjRS/tj72pSO7XR9t7Hy1XH6T/spy3ZtErqq+KrqbXRC+n10WvoDdES2iraE3zpupbounmbcem+45oY/OuXv+eaHvaIXoRvS9aYD4QPZU+FK1Pn4jWoV2ik2mPpve36E3kyDnwQtXryXetf0j9rc4yKaLLTKprr4+JjqBaopdRgWvLM9CeK9OFohvoIo2/xLX5XSbamDZp/HdFb6CDoo3oO9XvRX+k30SH0++urT9PzqsPUdiz/pmig6mSZ/PXS3S5SXg238NFZ5tJ6r7GXmemiWbTjaIV6SbRMN0i2sHcbsPNnep/t2g385Bny/OY6EZ6QnSTamOz2bP17/msUSoQxYwn2pIaig6jU0Qb0kDR7ma06JmqZ6leRNeJzqIbfdsubhLNMgtEM81C66blqnf7Nn/32/sYq43NatFn6CHRFvSw6He0Rt2PiOabx31b7qd8286eEZ1Jm0Xj9I5v2+u7okW0U/Qn+knve0TvlyLvE043uaJnmGohG16k2jdky9MvZO9zXsjed5hoTLWGuUQ01wwP2f4wUfQ5mhSy7X1miL9v0Y7+dlnPpBDq/xyqJ/3yHF3/nIP2cYPP35UoEP+ByJ/V4Wa6a/Vu0cH0sWqKx9qeUj0bPlC0Kt2vulr0AfOIhv8sOscs8+318RB/H+MwVYUORr3y/YYc1zqqNn+sr4t2o8OiF5sbQqyXiJ5PdSX+UP7Wn8vanS4L8fc67H2GYQR8w2XtSftEJ9IB0Un0tehgyvFYz6LK0OFa/kuPa1PREdhPLXdZHerksaZSHZ+1l9zvcpT7BY+1OtXzWfuI/xVk6B2P1VOtdtx9UN2sI3V8HIl87MV9Ruk4Xar+o+lG08FlvcncKmrzM5pWmN2idl01mt+riXqU4dv46SHWIVRZ9FzVwdRF9ELJ52i6RHSM3m+Mln8MldGPDutY1XGq41WvVLX3HaP1XsYnGi5/n6KhpDOWJtBIcR+mLh7rcNNV9ULRVDNDdJjZ61v3Yd/G5/XQOKTrelY5/fE0QM67xiP+5z5/P2MgnYHwScjH36KualU6Bp1MxZKPySjnVpf1Rqrtsw4z+3z+foQt9xSdD6dQa2P1b/rMYZ1nrC42n6ubn8cU6if5mKL1P4UW0XTP+i/2bTyri80SUXsedzWtNP1c1nvlOV6DdSmnfw3Nkf56DfJdzWNNNV/5/D0Nm+51tETufx2VmN991lNVl1D1EH+HoyrlevwdDFueaXSasXqY6rus80xHUVt/0zTf07D+vUH0bOLzQ77uCt/GWyxq8z2N5tLluM90TX+61tN0rZ/pWj/T9X7TtZ6m632mazmma73MgH9jj7Uv/eXxd0FsOrM1ndl6/Wy9frZeP1uvn631OlvzN0ef3xy6wfzqsB6mC3D9jZrfG6m+sdpXys/uz3z+TokNn6fteB7aT7b4L6Mc0fOoouhtUj/zNf587Sfz6Vb63WNdaKISnibXz6elFBO9Q9KZTzWogugISW8+Ldf07lS9S/VuVdveb6In6UuX1aGvRO+VdnITPUgR3Pdm5OFVl7WynMPdjH3e0z6/h25PP8F/Ieb/iGqaarpqlmpvz2of1UtUL4cu4k+Cic4wfVWvht5C803C5+/JjKRdLuuH9IVoB/pW9D7J52Lk45hoF0rzWE9XPUO1K6WL9jZPiQ4RXUL30xFcdyvq+yXVSIjfq/8i9b+MQsZqWPVukynv2avSN6J5qs/LOeYyPc9chno9pPqzxvvFte/lj+r1v4o69IeoPf/k8CzRkRQVteefHL+C6HB5/87aTbW76B3mTHUPUj1P/c9X91DVS9V/hLovE11uRqm7VHWcxhuv7itVJ6hOVr1W412n7qnqvt7jzx04tMdlXU+fiT5Ln4u+Qn+Kvqr6Gv0l+rrqG6oPyfNdTgepuqT3A9UQvcBc5LMOMiNFW5lRqqU+f36hv/mKvw9ND8v17L7N588xrKE00UdU/6P6qOpa1XWi92g/vJcO0Icu6+OS3r3UwwzyWTPpddGEsXqueU8+B/GExLsP9TEa+b0fM1dN0XvMOdAHqJa0pwd0HfIAVaFrff6ulEP7XdaYrPce1vs/jHWJ1cNUz2WtTFMl/D7TUj63sJl/shx6jtkj7/m3ivspdT9N28X9tLo30Nvi3qDuzfSxuDer+0W97xbVl1VfUX1VdavqdtW3VN9RfU91h+r7qh+ofqK6R/Uz1b2qB1S/Vv1G9VvVg6rfqX6v+oPqIdWfVI+o/qz6i+pR1d9wB66H37QefGP9fTOKPnVZIzKeh8yF8jzCxo7PETNANMWcJ5pq+oimmcGimfzOK1r+3d17VbcU2m+bZ5G8JDseXp33iG2S30SXoOPX82elk9fv0OuzA+FbAuH7NTwWCN8WCD+q4TknuT6jgQ2vcJLwGhpe8SThjTW8kpYhWL6hKF9bDa98gvAxCO+h4VVOED4V4cM1vOoJwhcgfIqG554gvENbh05vaMOrnSC8D8InaXj1E+Uf4U9qeI0ThE9C+FGEp7d2qOZJnt/BU+z1eScJz2lkw/NPEn6Khtc6SfhADS84SfgYDa99kvDZGl7nJOH3aHjdk4Sv1/B6JwnfpeH1TxJ+RMMLTxKe1diGNzhB/a9H/TfU8IYnCI+2c+g1DT/lBOH5CP9OwxudILwI4aaJDW98kvwN0vAmJwmfqOFNTxK+VMOLThL+lIYXn6R/v6PhzU7W/jS8F//j2fAdCOfydUP5wk1teHMtf69g+0f4EA1vcYL6qdLeoQFFNrzlSe5/iYa3OkH6hbh+uoa3PkH6AxG+Q8PjJxo/EH5Aw0tOND4g/FeER9E/Tz1J/qYX2+tPO8H1O3H9Eg1vc4LwAwh/XcPbanhpIDzSwaEvNbxd4P67Avf/VcPbn2R8zW5mwzucqP6Rfm0N73iCcOroUG5zG97pRO0f4adoeOcTtX+Ed9LwLicaXxF+voafzv+E/hk+AuHXafgZJxo/ET5Pw7ueIHw2wu/U8G4nCF+K8Cc1vPsJwlcj/BUNPzMQ/oW2//UINy1s+FknaR/1NLzHScLba3jPk4QP0PDeJ8jf27h/qYb30Tk+GH6I6w/hVdF+Hf3Vmu8uN5TK/RWbhxhWFS1ZD2VRLJpFEcRx+fPeiFNF5rRxlGu+dmKx2rWjpro3DjvJnp7rlxa1dqJuOtWulW16hg3cZc5Qk6nztCFjLqnM9+OvLv2EtEI8j0S7O/ZXtDIlhMN+1bAEClYW3SNhLv53NJ/8mzV5BjsjZxmV5X9iamHVlCU/mVF+vXc8bXut0WsjgXsaGuI4CPcR7gXCE/mGErUcKuuw20Q7n+je3XDvbrj3p/+re//z2qdw7VO4dtf/w7XJ+372vy7zn/8qs0vnOh7CM7XM/hVGvhtWFv3CSaN06AX4N80kXvJMonncxCqUdSiknE7pblmUr0n1435t4tDS/K+dIkp3S6P1HBeq8XyO51NqKB7KpAJ/II3Jz3VaYi+XYcry62M8T3cLnXD/xMueKYuei5j2CqQcioVjkXgoSnE/k0o77Hf6zPqv1JO5CP0jXtf/jpd4gdMfyp/GNHqNUwmVGHdwnbHXRWelO3wdVloOX7do7vH0nX/E65puOJ6khXgL5ybTP/8f6f9PacczqpDGzYhnxKggYwqey2Qng0rzr3G2U1nRy/RJuKyoHm2NpKdlp2nctHhaRUpstTW+ltLhxhXRfDyv1PSS9DDxffDs/nFFBmonLVmLaSeJYdKSeT1hDJ9y0zB6pGW7/Hk/B+0lTE2vsL/DFaOSsIuRpcAMpFinsug+KWmBOwYlGownURqt6YQp3SvNv8DpTLleb7JWaXSYPKcxRVWdWnhq1VGrXHPLpETW9rAqW0TxCGoyWgC/NIdPwKdSccTx2kbkrpGBVCafn/r3PU3gnv1OcM9CuWdE7jP0ePr8Jqo4hNRDZ4tfBH7sbhNyqbBHaTSPy4IrUF+42qFUt8T1Yddx+FsdD9CYotPcWKQgMgStvYEziX8xsUPt7Ew8sbZuDLFjR5p1DlPMK+3Q1KnSovDn0ugpuEdptI3rSIphSo3EI/apsl0S4adag+8azg5nU0bYXlE/zBZfVR++N8hvfBmaK+rRQt538Xsv+U0wT35LJQfh62WJ5tnf7SB+HfGC4R/T49/Yi8CHx5gBeK68FuWn72EmiJk4nzSaArI9uIPkk6Teqkk+rY36yYkjL5XgLg77TttwmtQg/+Jccdjx49xKTLaXHRirhl3xz7HK6K/JXXqFfBUBay5P9l3ZWj7O3yiE3STtLu5lUqKjZ0owlhVG2CqL/iV9E+OKyY99RLlue7fEyUPbRMvIH+IUU4mH2cwr9Eujf6J1paKFnOt0RAspkBbSUVoIt5yy6Drj4b6JWB/p1Y6Wtzylf6dQeJIUCryjKN8o16P+sQFmcKf9JhHFzr7D19CeSPsU+YzskA5fwd074D4A99lyb/5sbQn2vwXob2Oi/FlR/qnQIc4UM8S52iSyzzZDzDUm0fEbw6OHoWwvWb+z/lW/yXl1Hvx5P10adV0jqUdhx9ROhe2xjZpFbWVnmRhmmErw53XBIlzbUtpHK7Sx9IzS6DSHdWx0NTQV6+j6NDb6IPtR2MlzplAkpQINWDCXYmlxzOal0Q4mhbLSzAvePsePkhPKJicco0gkhyKpFWli6gay+dh5LCstVj0rnfd+2XJ+sAr35rNWhypSBDnyw5Up4tRAXh7GHJ6eEksty78YK6NUjGJ/cH92y/I7o9+n+5FQHo3NX+OcjXmrElKM++gH7Qod702OsQtjQsSriTEwm75oDzvqUl70cspBYy12M9Pjbm0qLYrS3qx0N8+9nErwNL2XCiOl0Y5o92luJR/dqH/yXqUdYrT3gmLHTS9BC4nFCp2YYxrh3+oFGRitMKpHU9Md7kuNURuZXNNOrHbcmY2wiTQmwi2tBzlOVYqEapHjFpDj1SbHr0Oxhjkx7PjC3N/TvbjXAm2uo3mAxps/HTyfFKkXvzqFO3pjI1411GldCl/qhHJRq29yO0opSXEpKxaM7zuViWNIfFwbCdeS6yamdpeWlyW17Y0NX1pYyxtrc8BxCrwxlOt387k3JJI58Lk3JODfUHpDQnqDd7w32Gu6nvCaf8YsDAfvNd7kuIiJEt+MOtpB+S7b+Xj2lfDsP0AO003c2NpYJT0T/TXlv0ompXpXS8XXFhDyY9qGOD/R5HWG8xOFfxMpQ1RyRrIi4t8otde0OeE1/4xZFv2Qc4YeUUwJs1tGVNurGlC8IFX8EvQpStQFo1m6dUc+TaaZHsM66PgTSgmUIxVPKA1lyXg7WRbUUaHLJUrk8yg4UPzj6KcT0eIrSu3k/qt2TvjspYa2aqq2pM95/3NJ++X7JsekY88Qh09DfR7/+zu+FmhpPMLmuu3c8lGWZ1m+a3FybDV8V4MSc/zSqONyvuKGx60UsXncyokZzF2J2p6Ju2nEOqB5e9g+5eTwtTk5/ISu4JHdjbsXwR6pdis8kTLKD/Na8nr0b06vlpbF95H/bPTMCPJeoS52MuHFZrv3TSQ9ap9RZj450QIKXxcJV6OJFbrKWJZF4XwuCZcy11zglBbFTdhkZWMEzMwhP8dFeHiR2SKpYJ1Q4KMOQtWkn1xCemWI6+AS+OdKu7xEasOumtcZnwozTpaazl9Yr2NUTYtgRr3MKc0/1YSx3k/45avvRMiulEOUHuKZgWT9USgzg7UjMjOQrFGqUyxFc5Yec2IRebouxuNQdWmd4b7hq5wwasRHPWS8pU+Ye7RjKlI87S9cH6+KleRthX4ijD3GxNNMvCpmcifcIO6XyiojQ2bdERRzyvp1IvN+uh8vSKeSqqnSZxaE051EA/SZ+p8m6wgr7F5Ua3K6E0NtTXWGoSR5od7kxFxZRWFOchMjcMVlx6/IdlKRxxBaYxj5zkLPiiLvWx2syiMxzJc9wv18rzJNzH6dn6RjXopjPaHXZkiZQ2gRnEYEaaQgDR/jUhrSSa+LVh4j9AyamHmqlj/LCW81PeKhsYHyjURpOtOFGShNnV1G110er1QSDXaZf5XYS3RH/rsez7/HJe44Od2zJb4YzzHP750sbSgxELEH/au0gZKatpEYnsFFXEauhYnZ2/4/lZPHpPI2f97xNm9ywsYUcUtJtg5pMRHMdCkFGNswHqSdqanUxJqF1xzdRhr5nUYf64CIU5Eq8e9CYLNe4lfFaBJFiSqQH6mEq6sgT/BLR8oZSDmzOjlZNTDCnIFWzSmONzOdCrwOr9C6c02Z0x3Dc/oUzOmpVODafRhaC+Zwp7jA7yhpcxpjiq4zP4d5hdWfStDHsrzg1aVFObDjbnPSMYN41Vx4Cq+OsOMgXh2RjFIFsjri3yTKMx0pUfk+6pd3v5TiZCXQ3GMU2n2MT2SyvOQ67hLUC7+XiRsekXN0/RZBj79EZpp4YP3m6Pr5ClxTUd5n4Jnk98ZYztdkwH1h0o3rkHeXr+Kr+djNkd8WHotrc2XdeD1GHt4blGBdxu2uorQ73rv9hJA0p6zoIqojszRyUCHLy8JoY/cXk5EGr3Fj0RKvAXLahveYiIc71oqHUrBKm2FaqU8W8TpOVlj8K/Ah18RDGaTrLJfXcy1lxOaYMTfLzZJ240peb8B95sl7C/tMOdfTsHNKN/wsZM3lxrHG4qeBOcWRMyQ7B9jcfMdryWaS/oFj/NzzqXAfl5yfYyxWYjrLnNAAtcD79bDhJ1Qid+GZMWb0mSO1asln7uV5fFLVlz9LgHG7rZ6PHDymuwnDvaSqH8su3FuC/pS4bZaM3tgJhuNh7MLD52Lfyr48fk+mLLesqCXNSkkPayyMy6gLpyCC+YNGODx/RGS2mCSzBUofTl5ZHTXFe4v7UU+d5XnEsrl35KBmWtZqg9ZXlQa8OxdrYD4rQarZEYTxs8ZaBW2+NyVa7E7uEm2N1WSfxGmfUk4U6WE1zrOP71dAH0LLjqDHp6N1Z+aiP2GkyXxHnnwJP+fsncaYwhey5D1EBWnTTyJfw+RvFtRAi66JEesU4jk73tgnzglmDQezRgFmDfZxc3gF5OZTvBHGy8Z2vCxyMKbWRp4K/jVe1sYM4XJZmskz6U2VsL2IVSwO1asQD6E/FKI0Fcebjk4hpYUKM7he8vCkWp+dRsNCFU1pdLwTQohfKYS7+rg4FtI44ZYTvqEBR+cS73ez3Eg4T+rsUkqP5EU6UUnIl5UJ1h8xbpEzeZefsm5Q81tTqTgFd0/xKN4AfaY69gvp8ZhPPN6jZCk5JK4zP2VXag6VNKlIiRsfwEyaJTrg2f4mnpYiayCUM60kTfpwBtKpkbhxBeJ1RryVMu+kyanOaXCvosRNKyhxw0oq69+J1p2PVcDslTJ7oK1EJjqdsabkMyE7e0TCdcjJbIqZo0j+BkeE5+9+Tm6IwtPDS8MP5lQzlFMd47SHeSGEPQuP5rE3ZEbh51fYMXkVX5OMnZi2Gi0MuZu2MnmqglnxNPFPTF8ps2MtF09yus0Zn+D8O2fJO3IO+U7JXHKubC62Sy5KuCVX3xnKNTE3VlHaloM26VWlBpR/jK8sn7OGypyVZbKqca7L58IamAt5j4S7pdSRMvqmHlZaDTCbNcR8iLv+EfGqBGbDmpQmbfpPtGn+nV0HfTuCds5711hOnBZRwvMMz/gNMGYVeAMp5vM+0fjpId4nXiz14slYMV7OXKzN8Uei7fE+Mzyi0EvU303hSVxGjBL1PqXwZdxHBjRpb1h5ncdnc8F1XiVMr7Ea8VAduwbx+KyyBu/DZOf8DGo4Vp1bqu9hZ4kVQCRcBSXG+jcL81O0GtbC1ZN1i3LGHaQzACu1Tv9v6byh6WS5uXSRw2E8WsHtZ6Fn2/Gq9igjf8skQrwryqNYzRKMI1mZCfMYxbL41JWtRHZwv8WzIPoBamEdxY/vyUuwapb2kImZAE8k7mQfvzbhQbt8il03dpgYMZZ0TDeFjdkVN3ze0IdH7th/xUYRnQqgIlaPUUORbI8mYv0Z5TOpzAJjZyKDJ9wb64n7TQuX55WOFK7GIwF6pWn9uT2tyvU6y+q/I/GqfyHcacfPlySmnjv9c0cUlxOF6yWluIkgJXufYrcwzKPQH7Lq6E3h3IRZh70Yt3yMxrz+CqMnRGqS6SPr9bRu8gkUPlnLyopFUVvJ+5qW0S4yoqVpWnzlgLFz7RyRlYjtpvLzsjCeK55UNI0K63BIosqnFMvgfS3fl+/k8EoytQZmhjyJG67FsR2qZHOXVlPmifK87KRsE0mLUuGGrEz+zEUaVi5EPUfZM8sIn9gZzArZp5kI8hzXdXO+STdy/2z0iWrhOrFU7c1ZsRTHxcrLR38Mo2ViBeako3Vm2HmKd9iutEh+15WO9heii0bZv4FR4JdgzhyT3xytKNYunr0Ua8vrUQRe+XG/XEDDsnyqkcUjFe/uS4uWOVl+zayC9ObkYJXl89yIvVz4hQlh/g2DGJWgtcRMoc8zCdIxg/rOp0F9ZqPmq2JvHfNiGHuw1jXZqBu0r1BFmVedFF6Xozdh1Rge6GC2nZjJ77rDN5RG2xg+m/ZxNdZl/I17rPQwW/Sxqwp7RhDGSqGb7Ad9GtBnrqwFS6MtDK/v+iEHzaO+rHO+hO9RpBEoUf5yB2NbVrafgX/ry3UcYxiurWE0lsOxalFNw+H6L+LaFPnzpxXkudnn1wM1x7/9noZcLRD/VBkveVS4k+y5scd/fYZ/r9TIn6ShhihRJmIuwXPpJOMCWlRmfSrvbbH2cWpidOWmbxXqIX/t0H6wYyzqSPmZ6W6BZ+N7eLJxsrupoVgv5YWxYuvQnurvSw2PKWpmrvR0hRfm9Z1dc9WS9V1Y5ngbu3BfasqAwddQswez5J1OX/pn+pzSEo9DZlDhLo2RUiK9jOcpblcNiPf3NRyuyRr8fg0tqFlKTaegKlpQdjXyY5iJKhZSpEIK+lhV4nYUlRS4nIMWofXcOhutsqqpKqtSjBO5k8xqJ1dGh7NpknkQNvfiOA1YxD04eTpb2HVA7BriHWAidi3Fwo7B3Vz0Uuz5HJ37nFTMfdjnO+n1ZK3IrQ7rmjoZmN8zqHBorMCujhPZ15ITtbl1KqCXV0QalZBGZcyf2Rgjq6RghxMn3ndEUqqQnObzuw3kg59j4arxxjf8KwY/UVUZB9CCF846PrpjV4m52rZgrFgXogWbLGnB/IsVWhvEvZTz8xvSOYp0yttoDW3JBq0zOyXZRk8Q60Qt2UE7Q4+JZfPzisodDmEMukzb9FT5XJEnv5nuok2/LJ8D8ORvZ2Eslb97FUE6zeWdSIb8TStu4z7/4ud9Dv9do7sdesxxdU/z2Sj7XoTHtKiTLu9FIhp2AGEj5Vz+rOPvUn3p4ZgHfDsPXBLO9u2pXitHT/Uwp/QJnOZlyk6T3y/w3jFcq9DNNbz26ce16LKizr21ncvye1Br9IEC931+54YdWI4XoUEtHjIcB/f1B7V80JSEw1QQtu0+jDVNadHVzjku5ybwrg45sP9iTRtCLdVj+6Lku7pQMt92z+vQnyhnDSnnW9yiEe9tyXtplH/tJZXa8moAofkuz4vsy62kLc8eJgfDR5b+Xmo6UsvS+nNg5ZJ9B8Z16ZYa+bxPBTp27Nh1sezm4bDUAVZd4cJMtbCPO90pg8/ac8ryuzpjYLHPhfz+mkw/jeW2bJZFLYdibCzq79QxqW7rcWjf/k7+g2zNCr9v3i8sIQmT6mGXzH/DScuZjTxUPUE5+1HI/LOM/bJD5eWLJsuXKi2K9+5cvgx53xPStlIFafcluyPmM3ceyTxpK2fISAY/yqNlyTUvv700FZPnAt/w/tXVt912Tf35Md7L2ne2qcdX0Lx69k3hu/ysc72K0uI6H9+H8fqF380OSr6blRaQfEOb7fJvJfL7Zvvez6UGpfa9kz1/+JXzGMgBv1ssjf6uvsccEwjNdVdhnYedUX5v2d0mY/+t7xuOiXJo52QZnXCHbP3bShGtx3TkoYa8R7X12ULC7LlIC+RttjyrhzEa8d6fzw4q416JcfY8HqsIl7/1wmcV2R6f1YT15CLb6zfONzl+OjXvzy4+k7ZhOXIeEZaVvg2RORsun3JDFzrZoeN+Mo+XOOx/GfwTY+xq3/YLV+5V4laQk1xr25Nce58I2jpfeYmkKLXjJcbaPYJ9l2/fWk0wdjdykbyXjTtZsiuxn8FIkV0J2yWS19zQeUgthez7x56onwH/OIuJR3rLSjWSrPGuBViljIlm8a962XOaCLdK+36+RFplRE9p+PSPdzxeS36P+wf2i2l+cdh3S8KZwRM8zKm15d8iKvyEz+r6hpInfRn/Ounr28rGs9cUbrHvm+0noi7Udhd3Wxw/2yuhJtSGTzwacol4juG3BGPyr5I3I3Li53LuXSldZcm9KzPt03imzeWzInFz6Fi/WHezttMg/Mt3xqr2q2xp8xH9vM8o3Jv/FlTcv+D4mZucK57zz1MtHu/5fr7cr4Pcz9f7xc0R3KebacO7wu9jTr+c7obHqWaoWbb7VehmeDwXH5d1GK+KMKKNg7tVbVz1XjLPMjdvW+g5oUEVupt1tQfldDOcdlvnxWNagud0neXzG9aYn+2jRH4qWgF/jmsGytNW+kkxzzMO76OgrqljWoanhhfyXonk7cNMrEVSI7GqrZdihuXTrJSq8kZoqeyHJlFuOCV5rgV3Idxpx8+55NqwjZc89eL9UZiyQzzvhqRNLkFeuv6jTeqMiRpMpwLkJBbmuedsOfMdirk2lp0c8VkXUmpKzNHZDqN8baz7mlHd21LDBSlo20VnO3XD2P0W9aR64XSHUypxOKUFMm9cj6tZ7+Wz9jpmrLnNdC/8OFv+Dpt99g/rHFRA05E//rV6PlEqLObnzTt3ft4hORcslOcd0rUq94gwpRl7Rm7cuH/wGJ+y2n6P9hP54hifsU6Sa3cf4/15LV7N5j+EOazw/Wwq/3zG+lJdb0Sby1iS/BwG53ATwprZz/949vM/DyOfHTAjleafibGmwHsYeenCMxRG/o5uHUJt5XeGopUVdXdbODw2deL4+NeTtPl3jsKe9Y17/vGQFJ2vKx2fr+vL58UjkhuiraV27VNA25EH/lWjfn3rmEFmoeFT00Tf2qbQixn+HM4Qs0D8+p2TbwrdimGaOig83/Tv28AMxiPod05DMyi81CTOqWeGhBdDa0FvghZAbzY8F7vySbftqu1cHtkHufMMn+z171vf8JhUwKeDSHGJSfQvNEPcW6GnQG+H1oXeAs2D3mjioYicaeHpuUPcRZiVwq4no3KIct0xoSHuYuz3kvP2VyhnU3kezVxXxvYa0vdjJhzLw4hYWlTs9jHpfr8O55pBfXabAR3ON+f2+ezk+e7/qUl0Ps8M6b8HOgS6y/CezOjfMzxSaj8zWEDTUK/81ysuQ41nHm8fv5X+8/MlhQD9nf4utX9PJYLnFMsuKUihAozhvEaOUbhpoRvJKgjsbaon9zZZNZ1Yc0dPBsqKOlAii+eTAup3aA41KwrRgEPYpVBzp9+hGbQ22u+nedKXitFnW0QLneOpyXreweo8M7lGd2wfd/49WmLmqpQcLUN5IbvisaNHW3MdtCR8UEa2YVS4n0c2+x61zr9y9I3RHB22OaolOdpjyvcP1QO7jEzdPxSk2RkvjQrSrYVxJ9NamZJTJ5DTSmEjp9TxUP1/5fgPbm0pxaGUCvHQN8d4prsjlcMul/U076U6cozoHeFomHv7LB5hMevdcTnWZhl2dMn4r/vFQ+1PeJ9g+sVR3DN64Fg89NWx5F2KQ9Fw25AdVx7kcSp/jTNK1gU7An4PwS9wkh7lOopqvpZfXrgpVpf3Uhdg52Lz5yfz53D+HMnfHsmfo/mzZ+71ZQeb9a/nk3vC55P1PzyfMvn7s+lhbRORQZOeT9ZCpXjtquiFRWRH4BwqDleqEA+nk55Lh7nUV0r78Sknin2xFwvzW5E41aOx+TcYPlmIh/MpMQkzdKS6aGLiegpfFZ7jZGNnHatLkZx6ZDpHqvgUvjhWnFMVY7FfnZxwTXIieVRYJZKSL58FSfr836+bmF2i71vq2dqJJmvnRuLa4R1Jvx/nUSzGNRSTGrqW/u81FI9GqdhFyd3jJXf55Jufbwt+TwE7fHos2/bqBlT7WAYVNgCJWPE/89Bd8hCLrc3nPJTnoO3/kIMMihWXRfnvCqfrWfE/061xknSzT5JuoWfTzZSUOf1+h5FeDOn9OFfOfvv9KOnVSo4+vI5qUetf6R0/K0imZ08LCjEWFR4j3bcb+U4Xfx4zTvY8oIt8j8nDjM/udPnO9H/g7i/nBVnyN6VPhZv/rjF/Ov8L8a8ifyc4Be66/GlLqia/5x9G6n3FXYuG8t8HhnuyuOvI38ThETyNZvrXtd3trfOoI80ztMu5ru2N3hyPf8jlPx7/fN/HHk3zZe7h+FNHG/ke1zDMAjXQF+6S99hjo/Nlv5V8rxrHuFv+VrUEO4Ujx7hXOlT4A7/RzzA1scPCTMlft5a/I2Kw20y+05472u7v5dQes2nyM5350U+wHsqm/OguOaPMj+6FplB+9j769/vq5OdIF47+78+Revoe/LbR9swx7p5C+UXv2nV13QJ6B+MN/w5l/+LmWCe/g6fIM/ldpn9xccB9h+lX3Ay7buzHmzczg9zlpp97p9nturDvxIwdc7Nd+y6aVyj3j7Z/hyUXpc81a71c51Ev113nxbx4ZDeVpPCJ5Mey0w/LePM2awq/S49gb1NaVMMdFEpPKUmpRgVh+/nT1jJTYXeUwuvc27A+riXr39tkxRsuX/F6Be73KNFwzPdDSnaYnJSLaVDJe/JLt4NbvmtyU/o7lf2h32BNMWiNaX0Jj0o9KDF5A8VTUB9nJCY/Q/m3H6Pgvev/172bnuTeiYnPUF74GPrJOTRLRrxUpP2o4U8hFiL0UZMdTqZa+V+p/lc5Umy8Bsl48km0qnLnFOKxN1G2wZ4ehhITN2A0XYYVXm64T4hjTJb0UgLpZYT/K8UIpzhHUyyLPmbkaZzk6vrhbC+L7O+hm+vafmleMXJ6xG3rAzzvJrJOS+i+qcC+1aFN8mk6I7uWWpSr+bfPrhjuykn3P97rxEy2KXA+wXM813VoSK3/mGzH0/3pPtwrYdfh/BYQZU7U6mXK71wVqx8+8bN3jsmdm2FtWSx3aiZ3ku8m4E4Or+PdvbjPBdxeijeYRH4CaQ22n0kuehbuswLu9XD3C7ifgfucgHujSRQMFDffYUiz50y26+uZyZHRds3+P3+OG/3c6Uv5sX3yjqnwrPzYLliYWaJ5tAvrxEHEnwbhz+QmcnqZ5OeyczGClRX1l7BsNz/nE/T0rH/F2OmWx5BTKq2lnBN+TpxPq4r+9U0CrjXe/Wg87J8HULGkF9HvN6WMMfJ3mFHO0HBKXHgU+8th8p2JwvPZpSfzgdGzZmD0fEreUBYe5hpBHbr9CtpJ/rGb85p32n8svxnXxb5jee4uynYTE7nufivf/4WNa7+dkRhu7/TPnWPNwM7xKZsnuRPavZvfAil73x7T9LziMJ5QOHmn7ON7gJpj/rkHCKt/7TG2D3C5G1H/C9cd35GW+LhPQeLCtfJOxsinaXgEz3ISw9f+O5eorYxkLk2eWY/xfifG+8PHspys4/uiJrjXeOlv+a6r75wTMqtXlX1SCWa6azqwbzTgi50JemRZ/h7ETcXuVN4jO6VFBW5nP93lT40OaHHElHiY+5v/bORTRibLyfdeQutoT/neG9DTKOFVMnEvRXRAl5lGPgnTNOFVw5U+DXCrmyyXXf27tDCaL49zYNtsMkZ+C061AuW34FSjlGihqbYIpOoNaM5xszDr2LZ1Osq90tZx6v2UuNeTd90N6E6xy6Ln6UnTEvSej9GSFkI/J/sG7EZK3Bf4TMD99ryPay+/zpvIwxTk5TDF5TMJV1J+hd0Uqxj3YpRndlP/Snia2BgWe6FI3OPPmw3jk9bUpwdnpZZGs91Uff9cGq2kNn92Nyx2ieHPWKSrXRd2ltr8ecMKalfVEmTo9xCy5ezShqXJ2WVq4HNqhQ6eZVo1uw6oHHf4U8EhjpFWksY7qVS1j+coNZ7q8qcKiN8e8bgpT9fPkjNy237PR90W27r1m1Kir45PDRIDbM3y+Vp+vy9wdU3K7/clNJcG/B/i3gQ+rqp6AD73vvdmzfLmJU3aoU1mJl0GaCFLQyelKWnTlh3ayUslnQpturGmmS7pXtoiq8sfkKWURXBDQKQIgiwCKqCCqIggiiwuuLErKsrff75zzr3vzZu0BfXz9339/dJ597777n7PPfvJzxYdiAXne48WHZYDqVEv4deVas4SdwZ4MsRjrsYZLbVJZ2gVtkl27O78Ev+WZEU5xHN6jEmi2DVPprvjYOsSS7gE7e4BW+BeihulEkV7KWORdgDnGcT6U9qXQs4YB27raYLhS73bdhqewqL9Y9a+KaZ+JZt4fxr4vYM4Tj/zDJsRwu1E+s/Q9W3E+ubpu4egTNH+q1S638RhL2LBYIohmEylcU7EFGiUL4GbLsGADtEEbmavcITWMUPoh6c9/VMsXYelf4pvbxdkWeemT8A6LUyfgPegm16gUwswlSjhkYOKb+/Z5qXsO3AeLVyLO7A3dKOW+FyXDXp8rn9Kj8/l1XP1PvU8qOt5cJ96bhjc156J8j83qGxqi/ZqXjHSECB7WtLmNHWZWwYVv4/K0CnEfWe0gduyF0d3OP96M8W7fILbpmBlOa95dIDXjLPn/FRpuGBvcf4St+MqnKN1Yn43jO3XKJ1JqfHle7APVaxPitiUvUB04M50EyeU6ZF6uqAPDmo9TpYNEFQrWYiR7SjuRlNbijEPXFuK8VlmbU6DNPNjno2YaDGESfsNcWczYSreHrXzOLazSLWTWs4WaUlxBEPrJCjJUoOlJUt4R/d4ciR8XjbylsZ9fTrd9ozrJxBnDOEqEJ8hKKUkbs/RbIG5VhL97rUy7t9tBXEA79z9YtCzmeK1ZZu6w1m6YzBXFtdzkmvss57Yz7HeeiKF9HX8w1UEWsUBdfKxvbBIytUhkl5J/uLXw3gKBPAZpVX9A7bdFTij3nlJypyFpwfPkHd2bUPVQO/uxndFuz6Qsy5EOfFAznsm5UQCOY2cM8pQ2j+Usxq/Ssp7jYSMMy0E8L/Yn7tH8N1XI3wjOHErz0mUZXR073wW19mt2Su0hMVob90InEYI0cRaqF+H8NRGE2dlzu3CbTtOFPNnwO6pFUbSOgKSoUarw5jI8gNau9MZS4tqCmM3r+EpnvwgOpbTuz2JQYRWM4Lfql5GIBHRX4eLiwYhN6kC98a/Wnfjh9adsWbDYNcO0b6O9t98eF5KkT2PJR+xsbzzvjSihWR0ml8r6zh4LZe1kUB8zOJ5rysK6GG4vwDcI/DshU5kzn322EzoJ9iPPiPELdC+61vyBUHUp4H3x6IlnxVusylcmCMKS76kn2fg8+dwxk2Rb+sQfctuwj25Ckjfr7Dsi8LFP8LHC8u/KEiOW1h2s9DSFdYUDrcXVtzIt+GiZZ8XGWMPaTeNGux9RzbNSxiOTMiSnGFK8cByhhM+VM5w/H9FzuCdpelF5fcAYbNEmiXt0ywZ07/fkGbRd5zFdxzdSd490F0svx88evGYotKrL6ZeIn4S8ysI3noS/m0Zwhxbma80xkgzXfhTvgtKev0LiqrfSfgo4r+f1HfVEjyZCDsEWTi3yJE2AYYe2ylFxXPpx91CkmQqqzRXHYS2oy2E4rgf2hWd2DxVCunYRbuVuT9nkB0Ry6AJ2xA8eyFNmy4pqjuNaLxpiAm18141tV4O7gbZmiacyhE6X7an06xdWMv3/8OebQGe9IdxzulOxPs/sw533jrESC5i7htjMCZrkL9YrawN9J1GfTi7qPQ8klBnlealSqg7vEp4d3hEn5O1WP5KxsFOwjeOzEEtQqUdhO8xpE7IjFkHGSsqBlNDRjsMNo8HVyRMdz7ekwih6bdoW4aSyteD20PprXynM42K6YyFNYTiYrB5o3GESTWsEAmr2PUJ6F5B+By14lGexY9cBcZyr+0B+y9sR+DU5UzSD4iTtAgpOZJoY50mSaE3YB7V2R1CijBsMh5GnAy6y4imIAjt6+zbajRh4WlWD6bWeaMi/YLmC2C+UT7yCLaxPlCG9DDDvP6fxLn7NOnsk26zGAUyhHi8UzobqaaX8SY9xnLCHcZBbF9IkK2L4Y4IWF6O36/lZSLcKF8Gt+kuxBuOE/Trti3H5zn+s5bPI+1Gt0eO83vbVuJ8K0iLWGIWcy4aFM6YXOTjUOSVjlkd1gf1JnWA3hTtjxPtZBWbryVtrIgTy0UOAffj2L94jH97H1iJz3jeDkIKJX7PXfRFhLXWWUe3IleBu/kTd4tcRRX/ut/fyHCygv05mBDeFf6alvtHcpE2yI1uQcxkD7eWixyq24oG2jLASd7zOd1OjL6Nsa4/tvNxbCee4F/3AdVOnKmiMEiyGdwVviNr8tNpat4GcW7TrHdQzXP6QXN08IHmKLUTLscRE3V24K8POsDXHXWIh9YglDWYq1dDnkhmEVdobUcdjqiWdObxTYye3DlH4pi20R1kYol1ufpq6KirxPk6DOnUCoNtU9pehPDM8IlZI9wZPiljWGLQHsS6nVGkj7whRDyEmONO/V/cSbW+FV54K9Y3o5gaQvhYYVPJorse7P6KRCY2B4Zi54kYJGIHHt24A4xO6ZgQDP4+nhvyW5UzT0AIdxxS/biu5jz+7Z2D62B24TOunzmDf905G0WZ5U2z24a5bRtFQMsUnsOnAlZfgxRzn/gya+YRldAnbmN9EypzsuJLyk+wPbPHh9TcR4O0FppNxc0OI1Qn+PiihunF1KcYy8V7me2vigs/BuluwuKIy2ZwbjbqkuVl8xpjruxFQFEtlHXvPKvMuld63NAjfG7ovrrt1Xw3h1iP5A3sA+lfOb8WS8JF3LlXhL+wIbRWaUchle88o1PhvUPupLuEE8mFHaAnd8NywXyumJfSNjhhPKFhx2hfGwP9bWTvBc8bhsj+rTpkafr8n0Vlb1e0b+R9priE1Sbduz2zJyN9PkfarXG4BPO8O99cQ3IW9Y2i1pVfpTxMEdUBXYv4AcsdVlbO8cttGlFuSVm5MQcst0wonoPae6k1/9/QcYes+TA6bu6/QWFd8h/TcXP+3VZw7yt8BukoHEOTxmfSAX0h3E9jBuzjWPswJ8gi8gi5AO87p2YgNUg89tRqORtnljT/OhRn+42E9tdVwes1zuepnIBtkC/LnKwDrXGgZFNVnq6ZLOmapbbLFGlcsLxPfe+uUTyZEn1FWIzWikIKajrjVBM1TsWndErGfBfLnknYSft6PG/nsQZxYe6vhdu+TnRYxCloaqvytKmt5y2r+ZO9J1t4OmJUgrCxdFsxdaEkr06F7l8xfOlSVMy3Ez7/qR/7doSav5I2nZq/yaSvr+k8SaNLj9Skk4T/aWoYcUPSpDuceZc5+TbraKQh+5rmkSjNSvI345+Hc9aQFi2dh236PIQDc5Tw528dlltWrqsY+ii4SxEiiz7+dROIdYgefEaILE7mXzexQdECRzvC7ccyGQVZJOO0neDn+bythxVnf7K7jCD3hpG0P8L1Rm+0ZtH+NJ0YK2f9adidgJivfQXDH8KA53LZ3w4X7S6WGShLVcaGH0xoHVUa1/k4ruN5/FtA6ah6lgLt6Vme1hw+H8n7oy6Af9frvULPNttLPgzu3LW4L2xcR8Rc2tcK8i3gtq0VhCkbvAfWChf/OhBfZ1qpaS1DCTejyrA8aPZaUeqnt/+vwH7WMr7ew/g6edBIiTMRm4sidn0mkJ6J4mv2pP4hbB9vvxa/O0XLG5TPK3chrk8oDG4vznDvZsIbLJI9IGR6TuPaVlLMVtifoW7vGhgp8Rn7wRIfSfh5Tz9hyB9US+MBaklYpHNO/b9d919xds8FZUPcgbA5A4o+B66VqKY+OVN4ltaL5HTh5hETIumfk7Xo2c0grS7n7Ccf6XY5g1cgRCsgu4SLf1SmEOrieB+F0CzBPO5U0VhPUMDKiYKYqyTGVifi0kith5z6wY+8I+cuSVijmKMF8G3sv7LZcRD3r4GMvBF73WxIWCfGET1nFJpozzH9Yji1jkV7x0sTTeJh7k4N3sXG4iabbTt6Ez1aJ9sxF4+vFr1pyo97X+JYLmaOZjZOll9E6yic4XLOFQ37lr9ov+UvU+UbVfkKrzyelwuZRhCLySI3E7oasUDSW1pDZydShGW0G6N6jPHeKbZYJzL4HLed8FDFW3iqelPVYvESIQoTER5Jj6/xizUH5mvM+1C+xtz/kv6k5hGu8WV14cngFhsVLJvobmhkuBQuWTvh3BzkWTuJRrFX6xcfJmbgKc87U/A3Bj3OZEGe1A5nT2q/GSY47I37zx8w7mM+dNxH/1fG7fUF1v7/z1vy5PiV2JejAnxa55dtNSTV+CruGefpthqkhJruFgqzcjMI0TKbWfpB99IEoF91LzXCdJLs6LS2JxNTESPPJtxM6RZqDdxCCYbLxKkq8U2Sa5WfCLKMde3TBHElspVu4jRBvJ8fM3+I5ER0ejweigfD0/itzTC8MwDDH1I8MXiI2/t74CxkP2AdTvrQdTjxv8TjU1IQvMxgNd+Tu/iW6sATTM+Cn03WxciJJeCmNogczgH9ap135te7NuWP5l8vn/yrUbpob1aaEfhNPjUkXsLp7sNfN7VRlCw5qI6NAc9vtZwuYBnX3oT523Udm7w6bFXHZqHKbhZkTecmNjOcp/UppDeLfBrLCpHqw1/CF721Os5fq72BtXpEr9UjvFbv4xeePGzBWk8etkPz0l7W8rCXfXmYhy/3YdkfMy71Mq7nR3FFMuLP+HQGrklB4F7EP/IXuMg5i58XOWfwryMXOyvVk7PYWcVP7mykhTMDYrEzEEifjemzA+lzMH1OIF3EdDGQXoPpNYH0mZg+M5AexPSgn3YzqxkbZmzFWR3IXy4Ij1L5y4U76y6RT6wQnQKpTF0mn1kh+pwV/E3eOV28JKXdh78Z8RecgbN4BhA/sRDezqXfLP+6vRs9TjniJQ3gWpg3i/BL3AELcAdQeg79qZ1AWGDB3SRc826Rn4srbFl2H/4moNJkL1Iwi+kdi/WZTO0Tc67nO0p7W1OUGsVyoDW+X8Q0TNqA6/dddS9EHwf3srdx538LcpFHwN15l9Y1uB/ca97mWyIKFVG6JaJ8S3yFb4ko3xL/gJQjlaT57UYhsec3kPahLNrX0x4y3fPvEj29ZwnlJ7Iaz+uLwz2zcU+U5ePXTxftjcTZYj92SPmZZ4N7Ka7b7JViscQdw8+4R+SAfsb9Ic/Wz7g35Dn6GfeFLOpn3BNyjX7G/SDP1M+r8HmVfsZ9IXFfdK0WxE+nPHd2YG9I3BtdCH1hPNBv6dyPVWUB98hs3DOp3ZDiM26r/MvVXnK76d3/QCvXRXvrMzQvwPDlorvx3aU06khh990iEU1ANfNiBFyL65MTnl6hDeRf1zFycaRB7iYOUhX/9sw5S1AExJaIGeqImBxBmHQNo6xj+CP6NQmbjGldQzdEnI00sH6cvUgqS5s/8P9KV454ioTZ3gfJuOIZ3edpyEVpR0UhYWTk2/j16aSvNv4Fob8y+8b/TOQqcAbH/1ykHyOKcTFUVBDN8ijUm6yH+NEvi4PdiopcxSJwn7oPcpW47q771P2QevF96Kjcf7/G7NOvww/QL/f790NjxfugRx51Kgeam4zvVRZTV8OTvLPGQSauWlA2Ra9BsqKNn570WqmgVp7kuslXLtUdh0T0X+mZ1lekOXjyPmAvLSF6ch9gfcW40lfct9+V0ZG9ov8P8mqPUe23cV6czzqvLqau9XoZqGlS9APXluu6BZKxJs6/ZZ++kF3q80rv8Wr5J7oPDObL/bAELyoQXnzzLoYX9NubGMTnB/lZ4Sn3+M8lvQbam7cGLPdLdhuO4T6Kpecs9yRFZns3njUvD/GZqZoyzZmjEJKuQ4ia4N90b6dVAcrPBtHIBsPAbXI2MHfv2RLHANc9hhCzYh1DzCd4NuJ63e7HHTWRZ+N+no0YVMRoNmIIhRQ9wHiWMZBaS1oyTCsLTTcrr5drtddLXavvKTPie8q8wOca1+JpjnG+8q4lWWvtVLLwUV4yNQ/ZnbZOJCMLZDK6xhKfD38t/PiG0DV0J8RyMYT4zdeBG0vEEuzjm+DF27g+7/p6yMdZSXmslTSOtxwzV/lHcJ/F9bJ+y78EdevCCDOiVjionxxn6cWP6NcimFHBFiHjjCtjFYqOjisK+EgeZwXEKmk8L0CyUsGJF0aeGTNjvIFrvIL0Tdt/JPRXVl/7D8WiaU+LpOXK+hDDhd4viyOHSC/vBHDbES5YrJ/cjnChF+FCoO1D92n78AO07bYhLDAULKB8slBvMo60CBZ0stRiFGQqVa3jvFp5hcZxXeT3luqqhET8QK2rE0kWEm4nnvdKPu/45L7I571Snfd9+1YZ91qe4tXIUOcprrGSzzivBqa+4fUm8PWkeMI8qKSf/I74sfDl2BPXKT3+DPwV556iHpLdtpt+VdF9Jj2R5T/5icc8GQmcxGzYzfyGcQ918/0mwGM7bF25fDuq4UIb5s9lnPFPpGeN9yKetRCeisgGPhUXgNZpZavNIUiSHQ+Oa4jHRdqgNK4Q7VqzVyat9ZZjKM4/7XeyGO0I1asaw0Nc45AnhdE1jqyH7CWi3OeudYqnr/rcJyJQQFi3yIjBYisOi0MVsDhcCYVIFY9H2cORj336Jr8jgphQFHp2xaD3Y3HoPb8Cei+oBPfCKtgQuR8OHHOgT1QD+V+o8vHw9/T7fKoaV8JG2uphoPdxlqoDHL9O0Sze9wkgbe6CUQOLzFoohEZBIVwHiyP10BcdDYtjY/hbpXdGddO3+RsRJt6EOOpna6Dnc0grfAFh5RfroPfmesh/aTT03jIGNsS+9YH9TmK7B+2330ns90HY70fA/oDvxwKdpf19j7hSehx+/+gHft+A3zfu93vEltON+P23P/D7FJCca3/fp/D7NH7/EH8fYe4IwCnrFH3kfZ/B75twf4yHgjUB530i7o9JXF5AWNdH5fODGXCLTdCzBu+pdUihr58IvUOTYEP4mx84v1kgi4T99Q8phPTB2L9vcP9M5qoD9K9Ta+zFdTgECAYVjMlchkPOch1Uxm0+BNyWQ8FtnYy3wwNgywPXMwXrOQzrOXy/9UzBeg7Deg7Heh7kesg7ONHWZ2A9RH+HN4pVYqGYazaFHYGz1gBDUYpZMtD8f3ibKJ9Kca2/vQG/edLjuae2yEsRBzgCBpvTYiee3MOF8gkQoC8QM17h0RfsqcKRJFVKsTZdCGHHZPIP/EeSaEWk8mBAEq0mlmiBlmgpmULS81yA1H0tRwpvkWZI+bHIkAYU+//vCPm1Psz2FMbP2VKg1fNnYXh2FVM9u4rmj8rZVrl1RTH1HfJphC1dP6xr2z1gT8RfsnXPRUgDq5lkndGayCcQC2vC8ZPfXfKuMIVKKUmQFBd3RNPKLh5+YWovj/YhXIL6QT5+E9yPCPTa6/BpCZcIaoZ1RGshF3Ogd/Q6pOii+LsROiIWwmTyMZmM3MH2dJdgX0i2JBmuX4frNB1UjBD2GLykpKfsLi3F2XCcHFm1ikkw2HUXLFhStJ9BWFyN8Iq9eIFbc5rQusYm8ZC6tW5NNc7Q67KG7GDxHAx2fS3wba32P3/LOsUrIj9YufgkcL/654CHfpydcU7EiTqWEwrvNb8Z/nX4XRm2EaodbgDXOxH31Z1wE/OpJXHqm0GkBHuvI09C0MeehATLSscTHw7bemid0ssmz43suw97EzFGgWXWQSQ0GiJh8gidhEj0IJAx8n9MHs0aQFY2gqxKaW+7GZCJpoCXO4p55bDe1bNYfz/z3qawTLVjMp7KGvIENktOJq3u5q+TJHJMTSP2IYX4tUURUsjDplPhVDpVNbWYPwr/EMjXjMY/vPjJs2INVl7TgHMVJ19L48GqngARZxJEarLY74MhUn8IyOShINOHwYZxhxq0pjJ0OCxeWCmUt9Ww2nGGQ/x3OyPG45xlcAZvh7QVkS0gjVaQJlJHC6RoNWlU3jq9s07xlNU6NeE6naQ4AAdYn8l6fRrJU9EHrE9hP+tjrv9vrs9T/vocjLUTLDsI61/M65NR65MurU8a12eDfBxGsyYQr1N1rhp7GhpjkVX7astbHVynBLAHbPJj6vXLon7pPkWwTxHqk00+srAvicMQmtGajP+ANUnjPNXjrN3Ma0KeKCfhmmQDazIOx0N3SIceB/uexJs8gje5NMZifceDa35N6YPO7RV3sbStXyj7bIHwMmsQP988NhMKCTd8N7iRr4Ebvx3ciq+AW7kXhiJrsacZ9gdWA1bVKJzPetxzo0E6YyB8zhDCE8SPxB3MV1wn34ODwGsnjzjqQNcwtO6lvtJ6t8ZL690fWO/RSGmSToOL4ziYxzEKOiriyotdfAWX3AtOvDqewbtA3Bn+VvgZ8zcR9hCdABnBHRWtwR1QC0NxspuJxJW/P2rzSaT0quOky1bJ+OFKbINiNjrjxCvhdyzE1cjLdYQ8BYZqQXreAqP1ZT6btbfCquoqwnFoD23Ceih2IJ1T3vlxpyKCN7iF54K81kYQe4tYpCuHezWsfPltqD5En4YmXNfb4B32kymxb1KOxbkSotqJVVdXezE5PrVe4dcRqIFc5XRwn74Xqekj+NdNvIHPzf5zmRXBeEeGU43wMiAkiRNMMX8dsXCkFJMj6uCpnEIWHFCorBKZSjr3t8BTuEtr2VM/zVpFZayiuoLnv2IVz/8j4FRQzI6wjhX2RewbxYZze385QnJA0CZnYS/nLxU5hOmkd+LMoghCdIfQLUscCgluz1LR09Ys6EZtNZPGN6W6Yd3epSLfezjTY3mkpOfzjZf3LBotivJjWxTPCOeSPNGEVJw7olVm+bZjX11fsolpQfjEPH6yB/R9XiRFBm+pU/iWohNtGzkrijf9w0bS/Aj7dSQdKY8GemB9OQ1US5w7TD+yXuHg5LujgmS7f2ffq+Hw0+avpGmDtHDOQyoOyrqKqQboeGQkG/7hehUHzUk5YmDBazgLJdrqp+sVTuaMR5hqhJkqWhf6o473ovi4L69XOCS1rSzl2w078P63H/L+jQ95/+4B3o/GXUDw8/31Sr94g/ihTJBNEx4w9hFNkb7oFotTTJaIcMCSdCIoigmerQj53W8zVOxb4puyD7MhwbFAN4hWo47qqtHQVd2IFXjKqqnuOvI8KurZv69ljPE9zUvyNhpXsH9d3RFGcJ7qhz54nA0HeG9pPe4JQ17spGl8bgbsHOsBOyw9qxak393t6Xc71ZL0uwk73Nbm+TQn/e5Wxhd/LqXn25y1vQm/qfb14gGasa2t+7OfnaP0bbUepJma9QpjXQloNF8Bd869wpOxqfMlwJ39pmDfYRQx06KUm3nVw6lYIp+Uc6XSzHNnIzShszfnDTHCexxiXxM973GyESE82f27Ta+LnPE2/7ptr3oR0nhc2d8kDUNrHLqZ17lV1Ur2pwP2UvY0qPtKUcO0NiDpJNZCwvTiQXUfaB72sSN+Bc9sZ+hfiwc14QDRnBoFzmHiXqz5WW1biVSZQ3Mynn+1xpbCRJMZnPFB+1ScwaL9PHP0C133ipJmUaHrayLfdY94EeD/+vA3YR44vlTDAXrkyVBPGTqwDHX1h8pQB/4rMtSQXpMV2Jdjy3xXuQvoFprFv57Fs+Nom2eDtMG0zbPRaLyntNL+RNxaydzaz8su1nKylEbZu/mpr4lOXG36pVJmWSkT+lpfE/k29Y79HaVuIi9w31O+jRScXjtUDqct7Utq45DSE/Po4GYgKq9gtEAB8dwqHiP9O39I8YB65neomGcmyE7EtnryR+CzhE6zZOMX0rLlS4aUT8cB+yVQ/hbxvka4n5/fITrJ10hvh9g5v2d2B0W3kEeSB7VR1ZZjVVtKj4vquHRI8e6oDopp4SRySFvnwmMhP9ghKJpCto6edsqe5VgPYoLTyWvkBsxZ07Mc+xaW5pHkm1444eow2YAo2Hb1kNK7cu3ZuGOulik+xUtAUcItelcruOUgJIqwTqyAG4aUjJfsVzJiMs7bz4E495+EjjBCjvAlocHUI3INPsnQQPN38Ik8XHyEdUoK9V2Yf+h+8nHHh/8pawyio78t1xixcI2xHLRXOnxW3jfpua9+Kpb9mVVn/MIa6HpcrumOG9qjRdcxckU3Reei53n8vKhmJpYO40050Pwtrndx/ZGY8werlD4K05sD6RmY3hJId2L6XbOUnk7vjaTxmuRxGMFxTMN3W419x5ejNvaT3475J4QGUo9xPvkz4vWwr2FdpDX2V/Tvl7Qv43ovsgSu1Wqkq2eDU0s9m2QiZW23qpOBGEqTmf1tQhB/6nu8/58WHn/pfn1n0X6SfDfhTQCv4T2dhG2Gm54GXpQQN50DLxagujtAa7YnxUn6yU23j9B2TwT8jn5rSOmUFOESqWybf29cbxiI2X0CcSinZ6D5YtmMsJh8h8WNh2psQ8EU6uv3hxTOn4QBmRGb8E66MNRv/C/ipOtl2iBYtpE1xzeQ5i3HOhqwLycNOVFn/N54qMUWGWslwtbpmDfJCj4rP59pnpcmn9f1U2yvqOGAwbu70NyFf7OgRrRCX8tU7P0vrEUtuJ/Ea7LQMg3f5fCvPeAtrdA8m7wG4joQvEvKKSFai5Qg20vvabWf12h6T3f7b+81vKd1frn3/HI1ft5b0nt603960C/X6dfS6dd8pP62Crw5BvgLjvkKjQsvQri2aD5h59N8b+E00g6E6ZRHu0L43s43qd2yoLRbCC4hvjA/ByVNloRO/5J4cHJRvgOKzS9Aq4hzS1U4cyuxja/yyth4p2CdlptXdSq5eeeIGqdguj2QnrBPC+q9Sgdb68F3CmYbfsvb/ZZV+s4R6W1+uoAjLd0ZzobyO6Mb74w5eGfMxTtjHpdTdw3xT6mc29WNmMocxKbmgts9Dym6iLBbq9i6gepLblB22F59R2N9x0CfcSwsNo+DgnW8vocsXWcV640ejbfrMZDPHwu9PceB6x4PA26M6z1QP0/Aek/Efp6E/Tx5v/08Aft5IvbzJOwn4mBdUa7vQDzj+VjfAqwvv1+e8XxwWxaA25rHVa8QB+Y994ke6JMu1tPL41Q0kBeXOt/cA/kWF+vphWJvXNizD9yfhdifj2A9p+y3PwuxPx/Bek7BO6aS++PhUk0blC7ugP0u39Md0IN3m8JlCF/uh3HS7V7OcFnhF58lHLZ7KfvEGplzJea43Svx6XP8tNzPW+XnrdB5DXLA/uIIfzCjAv5gyM54Jt/FSeNYOdhsivFG9jXl5bCa98FkUL52FZ7j+vPXvsGT+X2c/BMrmISY4D14sjJyCgzO/w20zh1M1YkmbONeHa1I2Y0qO6Mwjv99rE3Fc4qx3rKEWVjvbbreQftrkrAA9URYwOEwuOD30H8p1TvRqzfaGN3FOkWXgdt/JuL+F/oxI5Td9/lcUrDuEMLSrnHiqp0VkQF7N/uHykUjkIxcDzT6oUg1nskacQHW6i7FdRFu/wqW85MPm4x5Hq9RhLgBqRtJyk5+Js0GoXMF3ajB2XZqSCKwn/k2c1HSleiUB3vYvXSMwa6/wLxp2V8pn+qLLc+n+mKSEIRH8004yJL7SzFd46Xt62kcYVVO35PasxTNcAiqjcpwg5gUJv7RHB2790v+mt4sKnDmifZfsUHxljIWrmmI1jSE55bW1ML1OAzCRXGeuCL81aHQb+lU4BqEeWYtjlSg1jbCtoVRII0Baj0MzujqCNk8Kb30tRs078EecKVIG0SFmjpm+JbSuy4h0s3qncInd20gr27K1mMce9YgrxSpxMvKR1eN29ItOFpLW7dIivnWQOoq8s6tvCHIkm8Kr76LNyh/eb69WVmNDqj6KvepLxeor0JDisuwrrAAtm7usmbE6lnzg1Yvyau3B5LRQ3m19njaHlr3pRo6yJ4zrvzSkfw99dWXka6cAR1ynK+BcBDXci3XcCQozZxkdDLsTx+nGhrjL4P71bvEh9cw5QA1ZOTvce2WGBKpvGfw6RSDokwoTUsLCnM/I1z6u+cu0bPyLLFo4Vl+2l25WhQWrlbpBzD9wHJRKCwX7vGfEc6oxV9bKQrjHxf5jy9i30o03p4LCkAaH2eFK+KFQ78j+g55TOQPfVQ89wB09eGvi3kd8Q8ZiUj72ktBrYdqCLaU37nI834vyHMtcwXNFoQHr8MV9w80H2WkzWqrMq4sGGdangUjxcvRdoz4fBPOWxWnL/N0JqIqpSlpfxYBluPfNj5jvxKm1mF9ZINv82/U8S7j/WurffacQbRSDdOtq7H+o9j+mHZuBDwPMPXal7iA72BdzXz3IKTCus4CTb1KqiXN8O50UJ6TZjL/owGqTbflNChRuVfLWQyZTw/wLmYyhKaylfj/JArRyfcAYfZhfX5+vEHxYQnyhQK0hB+nDqHa1AA1cTpTgcpO7B5cyZBo96xNEzkDMbmpeN5Mfd7Mhw06b92e5ZVUFhQzVexWPZb27u8Ph9NIhfu0xysblC4xcUTc9CqW/dr8RPeikiMb2k7xtz6sGWyuxv1TIRPSoytex3cry2y2ZjAPxG1Zvt8ZZt+SVrZC90won6d5iFmXWcKirzqtKn/O6Zd41XR3zLISXr5FPj4Rn2k/1csRxIMYxXZaFZi/KrBqN8u52J/e9sWw77ub+F0pTZZzCT8WxT9xbBTPfYZFcTw8O/aE5Xape87idq9mz2aNiJvnYCPncxRXbHeJ9smfPTsTUl+TlPmL2nOnh2NMKrvzGG/BOpWPFdyRBkUonMl7u7WHOHcd5MmRcBoccRND7RfYBm8iZJ/zPLg5Ce3DTZK3Pu3DLVD/dfilquNm9tR/s2e39mpHmE7AF6SKdhAHd54/1jB9NwS0B1xBXP3BVJVwmb+v/FzR35iN6u4pUWL5lkXMtVV02C+swa4/QY1MykJIPSVwH2bEKoQje7DuGbhTJgng+8/WPi28eyizUfGyB+13QPF4r9F2e18Z4YupIuCL6XQerw3Zd1g3Qu/pyRuVjHQwlRArcMZK/dWndB/8xJtJsd8zGtZnlLxAI21nZF9NYP9Nvqs7sK0L6LcK91GVaqcKsmFuM3WFfAXCVY5N5+BtiFVruyaok83rH0gR5lWlffrUMZdYv5eEy1SRFF+lKQ4DzAgjdv4Y4nW9F3o2sngijsMxHUSe9S2FHSKGEmpFWP4mPP4sYSZne5iJ1WjtwlMyFqhVi3dvHY7zWnkq6Nas1rmV3rPULRvmdDHe/KxXYupf8PzqPj04oS78e4M0GaeT31Fr0K7HfmRvTlSFIcxz4+LcDPE67BY27oOzmYIgDf1+TKm5otTSstTystRZZanTAvRCQawsS60qS51a9t2Ksnd9SEUvFothkSwghXYm1AuPr5VeoTw8El+rO00eafFUiWWwRMaF5ptKZROrcitG5JK+joIvqzcqm9yxcBPbLe9A3JCsqQc5cuhquZG9Q1aCEyGfCOTnoz1CHjFGkUdXPGX3IMQZhev8Llt/9ywraB+PcciLmUJbpQraj2Pg+UhEZF+s0rBNwDZs+0SmM/k+nLXUO+VSQzDmW6SNXBhhwNErwYfECDM/Ap0hpP7nLvegXZjOwnrePzgXR/v5FKuVdEVDBMFoL+7N67lgiF5vuXNXlUHMFRx3uNJqYN4QzdGxDAOO83nHn9yo6OQBe6xQfnzU/X75RsVDnWEQT8FN8ajSfRDkQ7eo2g2s3aD7rJLrTPi+IPZsVL4NcqaKJWyWxRIuj9aTCUTrQSgN7zKUBo539Oaw23Im6LTUv6QFJQyOzXME3qHZF6p8P3df2Kj42b32YhUlzl7KJ9CLLuPaRC/czbxyxXtAum+jwsl75uOM589EPPgFK2lOk0X3DuYTVOIZXME3zYB9uvbyFwF3/pnspZ1qmsS8dPYbJDorFH5uwl1Y7wPEk7vfgc5YNcyLxSF10+nKBjimaDLSU0caLJa+03GmpbGMiMtUDd1VWEac7mFQ5BtSpO9yTGfCtJ44zIvHrdRduGcRL26M30ORfo53z8N1Mtzz1e6L8S64DneB3kkXrPTyw7Q7hggiWSY0GFWwJBwXbZsOA72zzMGuWnHMUr6xPtbHd6OrcSbHdj+GND7iSConHvjGFsecWRHLRnVOjG60L0FFLPz5BmNJDFu45WBcgTHeF9HBrlFi951EB+MqXYx9j7qX9PH9uQgaolmzMtYQnYS1/B2O0e8H7X9QDF3Eg9W7xbs+6teWjM2Tg82G2B3Dt9EGY1IUGB4cD4oXGMX1+DzfWWG4T9OgBnxdwLUCbhEeb9eAF3HNBsplPi1Ejw/a/yQLYdnJnodKOA95NFEcFJXzWc5RHJSROVdijjtVcVBM/Z3KW+XnrdB5DYJG3uThOIHTYpedFv9ENJvCsLJvuFOxt6bbrna5BQ0mwwBzkoU4Ne6iSWZ2mHVocPyn6TgtZ+r5kHAG458hrdX53kYV56bTnObj+qcGMNGbCC639AVybuacILZ6Hees2qfMihFlGiCrsX63Gc9V85FI0VaBwl1auHdTtS6pAaFNSqdFQSZHLE4vBlof8tLZADRvSEen1ar9A3MDfvbx/SRhgvIzoXAYxRVX4x/DPOOoxjVqsJ2vaRzJYlrjeISuhOVYZVhO6eYbYWWMmMZYz8qYYz5pTzGGD0m15xj/ViMtDJn9pcabwoQ3ldfXFqhvJN40mvAmo2QdchVLEjtMxHEXIlQLX8u6kENQFbAguY79o+je8C2VfULDS8RYvC+Plt6X1DPBfo2/UuaF2bFz4WTAD7PamWH2inM1YjdleWHiOE0IZ3+lva8hpPo6jkatt4qv3r7J02d/hyXOVELyChg4UkcoHxFfl+PZV7PF69W1Sfmn9HBar3RCSJ7r69kroJNotSMI46/jMzPbqNoP7sv4LtMMHr1APW51Nb/RoJaPACVzUH5hFmDbx7BcZwZQZBXlpWVmwEtLjR8jcDHSCUlRpb22uAv2iLKocdpTUBWk5r6PdUWg0XwflE1Ht6XqPeqA9Y6swwAlGV26Sd3pNCfAsEFydAgaiQim+UndZ4a2E1B1KP94hhfTa5PW98a1SOm1SPOdqvCxdZuUjm2n0aFijLXnrIPAnXCWt9v4FlH2odVl+T4VirvREu6E0w7whZ9v+PAYv2g3q1hOTfTTuZuUP6rerjPYS3YY7/G9COsXzzoD4fvF2PevSruMz9uMmPwfgXzBP4Oz0G/9H9IJiGV03SVTPdQunoe5eB5gN6hIs1Ueh9dSdlpnemmEzWshGNFDlf0rEEerE3f2YGq+IKndWI7nQHjDGXMUfBNw6SblO6cOSFZ8FttX0LfSw0MwtZLgDkJnR1BNNq+Y9Hw8ijNCnl7+7k1BvfwC0gQFec4HyHwG8P1qWIy7bDFSQ/uTpQwghrsaemcPQm93EedmEuNIUY333bRJyWHzg2uJP988EXYQtSSV5rpkzfUI0jk6QnokFxnNmuuSNcQdICl3S8SwchGluc5R8SKWQRafzNH1fcIKuHWTlhmlhhhDXQ+eVmADVAsF6w0dQ5K8SEU1n+VO/O4zoCKlRVivKcd6TcFodH9jGDvYnBEh9lYYZmz+lwylVa6KLci+oUe8yfevZd1yhujqW8OpzRnESf8teSxgbT6LdfuwPJ7s38tWkyDUROW7kHINFXMR2BP8f/btR0F9W+l9K6lUYAQbTOFuyAlOhUfEA0ZMo9q7a3A3r0ZoebAIRSj/7eFqjj9qaW8D392kaC+tFcV6Ys2gPbsprD87kDpR64wRb/NEOdV7M8bz09WMs97AXlRKnrt0jhfVkjiSgjnq4iA+H7SurdrnkQFPb1K+J9wlvB+WrvetER3nzrRjOKa7Yj1MJ+9fyh8nYsZ7sKUJYiHT9ojTL8dvUufJWnBqnxdiOPt77beQfQ2TVUOItFRMsmSYKEJ6DNWy2ihFg1G0VkjjF434e6Qv85ju80x+tUnhVjoqE94CaxieAcPSqzi2kyPZa0XpDXNpWzxOTIAyIH6T2+OXA02vMkQ0TKd5GsclyM9fy7YbxEurDpXO0Vv7OUcXfMA58u6Av25SPpn35TcMlXED1iBMWY8U/Aak4CP70PWUG95vbmy/udF9eACe/9nQZs8/udqHpANCZVoZR1ByoKrNilZfYkVEq2vjbwx/K/A3jL8R/I2KNpdkLVWQXoF0m4XUlhWC1ILVypeEtRqyRhKrWiLCuK9aa6P4hN/Usi67iatRmzNrIb10WjoB80IhmVpKFF8cGkNqzXC1Q20TiZbqD9WKDOJsg/MbRcvcgdQrso7XwdT4Z8NmrdvVdaiGsYrfMHGzsjNy5w8x/uvm10PJS9cdwGnfQxdxArNiQoh42tdDLtwCucjhvOsnMJ/7YBURx16EJz0Z+Rl7VrjQw1Q5lsyFmD8KdD7+v8qLaqPtEN2F6tTUwcEwiC0cii3gU+ow3UJTWQsv/ActZOdQtEOnZnHPukDM5aT5vKn8UVFEnhxZ/YRHGdlQMnynQTFz7uW3azE3B+pJt8kyS8of5eWXxWoeOaqsRefrYDhxmPMQIoR0/FqCCJbihIT4/Ifo/NOaPcpn5XFf/jJvs5K/BM5Y6mMkW29aDyV/r6pGGeCtGBy/meoaxatP6fmbla63Om0BPUx7M6Zm4HqfIFJGwd5UltpSltoasN1btDmIIywW2/C8buf3SmN42WZlCzhOuPb54J0+ig2Xwh3XIKplJf4/SQoNI6p8eFepYQxSllhHDdUh3dR5eK/cyVSrmz6fnwnW7EVYX60jHaq4jRUahqq6bN+//8BmpX/l2BQPPSkuMxLCSannFyx8blbPj2J+UpxqlXz6SNiwWclqxkmi39z0x5lrbGOribK2w6D8Tqq2Y/73WwPfn4ffnwckkf/Xv9854vvz/+Xv1egvwO8n+2vvQJ99sdLqsi8M8CwK9nllqfPLUhcEdbvsj0Of8wlMHaXpBcl/V2lYOk44msvGVDSvswzQBqp/YcYF6LvrNyuce5zsRMhCa62xdEHR9Gy+MXH/py4I5N9B0rLU+SNyBuypgjT5+lIXIWStNLJNncY46DSRikh93C9Ld6XNsL46mF9GEzSbNDoVSRNnmM/U0djrE/z+Z3V8cQm3bfbouN+Y44xOsxPclvN8XgXt+TTz//DktpwfyL+VuRoXBHLuwBxPD1qAp/88xtN/xv8XjNCCphGTz8S+qRfxehjMnas0G4xJJt1dM0H5xFR9nqshi4T7N6szNw7v2guZ134Jfw/gxambJIT2k1bhfx/X+JuEb2t4QmtWP2LNvqxnNx5cM4PG28x0zWwVn9Tw5nW0jgcoRvgDfUrDkHzqE5BPf4wlZKQHa/MKfwzUClX5PjafwfLdrGPJ0Svx94tC+W6NQq94nO/ipKgzsuF8y8XYr5mCPUSJxyDfgi20fox1rKkFhUl5MRP7xDe4RJ94En8vwt/v4u8n8ff7uH5K0qM4vCFOF8SjPLcef/nXm5V9tGMvti/xTw3otQD4w2Yl3+0TF+s1KMVPIRpqZFwVCOQXYa5QViKlfyr/GF2Pgg1vb/bi2Z6t6zoDPhVo5y+by9uRuv//2Kz88y+D/2Gf7vXw3TtAU5gq793hIhyLVWTfKqc0q3jMmrZ0DF2fsUXtmyK8SfZFcISp/HlUsRSStBmmmUqbgTQWFCV8GnMbeNeDVxp8H35RxtGondgWFbeoH0c2DpYJ7B2eENrPTuJMTKl+ZuQaxAem40i7BeED45CyWY700BnyUujHv4HUmbjq/QKf7LOxzaTImYozUjoZCl+fwu0fpmNEGjB6ixdL4FOMY1GbVI8jqe4GUPTASknatW/w6DtKo9f1mnpcCc03OcjnMyaZF+P5Lp60RdkC0f3cM+syvqNt6c66HPaNqlZbJjel86lwrLeHXeNyL/L7HxL+uWvdongJbupy8Pw5efQu5ylvqGW8ZKWvpHCDaVvUfbUvrnE5LHIuC9z1AEdt8fbdCb7dFuXP8fNPFp49mKl1IaPct0/jTXgFFLv6hd1NVB4cwK9AQXwacZMrfL5WqY5GcQo0ylNwNy8TaY4oAQc8d5aWgR+H/crzvJf0wBcj1Zk0T+Gd2uVptZlu85UH0l7nqG8JT+MEKTOiE/48PJByEcoTvHpjuFq6zTcc8Pt9v3438PWb+DX55vbiqS/aovQikTIJ98CMCFIdJ7tLb/T4sr6k3ZElq5S2gFXK6Ujj3sh846la/yagFWF1Wr/1n1vXvzKcM18a9vRy2j72/HBORjyOqNk+/kdeWZl9olNWwNQ1EbztFouJHMvBBKe+OlwdgJ1nbFE+1/P2btzp7eDaV2HPbteyuSmYvj6QnoDpPVDySYhYg31tQJY3CtNXBtKVmL7BT3do2+Jqtl4p8S3XbVG2K8XUR0j7RrjySuYrOqy1iXXIG7y0GBmP1ds3W7fQSSD9pz2guSaB1awLrOYMcBOXByQd5/JtlBN/HCbtU1pXTx/j/C3q3nTbLod94yc5gfhJuILp6/jkCx7nm8MOnhjiipiad/6pLUo+kpSJEO2kJg05BlIn6fqqmLOi+OonS8Oz5dN2PlGtf7Ib6/kTn49P4vl4mG1InzXzu3YjRvhdacuk6DbcXVeRL2ahpBdPcO4blrvr+hG5GaHqEEDvaJcGomAbpJOmo2DzHi3afSyfcM//DO/VdpZOPCFnGdnfJeU7Fj2nkHYmvPU+rnPPyMjaWGcuUOcMyJiqByYkrfFmzxGXwUDX96Q7lLFULdb+5h6pPF//JcQUfOohuQKy7ySNmSb1YqpFObMg+1hStul+JcVCnIFrcRRby+ZFhNxdN+6Tewjm3rRPbivO7JXk5bss922s9wbM3R7IdXdSlNnvK75P6mtkY25X+zewHfrwG9hhiK99ocHPRUTrlz6A638Dr38XwuaCcIcqRLHrLPrFe7aX7ihxNdZxt1C6jQv4juqQCTXXqUd4zxUFwVEEwfj/IooYAa64Zj8320GBm+0UrlXxTxfwanYglMoYqtYmrrUZsr9SNXmR9w7x29mgccW0brNkJZqUVSEdnaH5o3KWlTSnexg5ppeGR9olav+ToWqexxWe/0muY4Xnf9Ki0hZkrFVY8gLWvFphZuUkKzvMdyfP63aGHwQHn9+iaLsBO82xqejXhLjRgugA2cZ6sfRe9mBCAM4E9RydgJ7jDGWnmrnc5wTtTWdfqxYeb+x3WxTNTDCTYF+r1iVMaUyC7cVltfT08qslgGdT95b/7bVQ4r198Lee3t9f8dsJvIdwTeEu2oNCeZ5JhLTnGfYwkzSVPfxsTveHk+YC0GllB2+yHbzp6VJSv/6p+1V+f5Md1yme9RamU94tzj5dPXtbc6uSU5QwGnfWtfvcoqXdWVeGd/Fcd1/FULiJYc3rw8pjffZ3Rft/JGjZsuLjVW9VvF46QxQZSZ3Mb0qt6YvjbTFLe3JBKGksGBHBRM8c7sWZuLsWCLJOa6IYrEYpUgpxj4qpT+G9njGOwrwevL2KqUsxTV5Rw9puZNxWZV8bGPfSPfs5jZnAeBF2IvzsGa8w027LnXo58E5LbSd74dbLvQgPePZf4XnAs/lCCQc9eKuiOzKgIC2MhP++lqH0tQwf0lqGnp14y1bF9yjdRVrnYj9rldxHt3C8pwEq3xymtaL7xYOgKSP764RvOzZjv+1YH9rOam5ngn9/sR2I2WH+Edv7/XCpHW88x2wlrsO/Np66snY8i/YOg+JFqLHp+n+X8PH2nq2KD+Ph7SXuz3VlqavKUteXcYY+U4bv7yl7d21Z6say1E1lqSvLUjdAwbkaCjXXME2p6J/TtypdlBExkxFPWehpFCDeg7sQ8akeqXZh2nDh8pHYF87V4WW63yrqdL5nt2//QnwAkiT1YZ7bc51/gjuMl4d9Te/UEtak6DCeHXZrfKrqB65doqE64CvD7vw9/A536s2e7gKd901bFV1Bd2W6u4LPPcUUUmVJFjpgN/K5Jp+aBLlMDx8TyltmUrie30zlKVMQ9GsVCsboUQd2yLiyu1PLVpkqHLAPKlny/17pBv8P3qJHsm5wlQ8PL9qqdCfcrmv2g9MeEsBpVf2qXVW/uvX/OKzger3heRQrjcjxoLjWS8g+48k5aP0v0/ChU4yBTlnPeofZmiCGHtRQcQyFo1+HtVbxnFMd125V/OVOzOXvq4ga8G46sjFImsB2UylTrZW6Qz63Vdkeuc1XAVHca1K3ETey+Xqd2kuprsv3MydVgTlBnLz1M7pP2bcGUqeLbl+fEuCOrYqGdlPXgdvit0HYZmpItGo9EsXbv2ersg8u9YDKkqYeyZz23xO7rCdEP3JvaK6s7Bteb0qxdb6NbWxjOSn2Z+lVXn84db2ful6/4xlYctV+6NYRM9BfmoGA9hzfLkqaSBJV4hi6/XsC8imkkZou92LmGM8bxvB0IwXueMWb8KhUd4U6q3NZ/yMCxIFU2k1hXfY8LEtfZ3+psCDPg0VYt3cetscy3YerAvZYP8W5IL4MQ4rRtE/v5xup3cbn1AP83Cmi4GsA4wmsQUwZRPa1Kij5B3oF62mjekhHt3nA/ibdGYoKPVhFLrqf/eu1t2aAYokONH8DITbxqdnWxiT/AO0iZnSa1dBpVJZZbXg7f57BfiNeqGI+K0A944mH+2N5/d8cC81hcCwxrTfzF6wnzf64uxEWPsreCd2dSOelBsk7Nyy3SrnX+blvBHKv17nujusCFL1KU016j3HK23FJQLou9RjXoCIL0ywCR0Y6GBxDj8Zs787os/wN0nywtG5+qCNUA+4Fn2HrJYvi2yrryTB5J5hvKknAkMnekl9IwrM841ERQ3j4E34+WJA16nP83IErQTdFK0cRn8tW2307duvxjPReMKbszsmft1tjxMqmSeo6iArvQ9o5+5sktPlzlYNpoPeGbE+TrdeDxI9T0UAnEt6VNMebpBVDXiFmr4ibCgdLmkh/Yh2zKeo94mPF1ACdrDJcLKzoYpaAs0XK95Ow0J9jdydi86nVvHYiVMq90c89JJB7k5/bapRyiZZYw7lve/VG3R0Ed9fiPqM91efpN8GZ7PuDcdBttGtxn5I9z2Ra5ZB3ViaUn5WGsrNCK3OEII8QLN+3SDu/TB8XV3iayWfk5aqAfcrB2xSuxWdzQtnZbKBYPt7sj2bfQdSaLelkcisIYaJAe6hdKv3MI6Sy7qny+bdTtyldt5zV4Vl7KN2WFo077gd2ji+DnYTHjfJoC/HmsNbtMDoRd8zJ3/qYSafxCtId1zJsr5HZn5X07bq2qZhunUYCCIa4qRt9KU5wfmyTS5AG3fibyqRlnm3YVCt4Px6v507RoBxpiyDG8HSKEduyx/MQrnhdDsHh83FPEter5KEtrEueh2n6Nvurkl/Y3m0qJiLR7F6MOPJjgHiKuJrxFKXb5vGDC9vK/cx+DgrysxoPoPfLtyn7cK2J4tyMuOYXEXpw/KhmOpF/YT/kBfvzSA1U4jyZKiZMWEU/Zn0uo1DzJSjUfiEgUyluGylTUbT8kJ+/k3nbMYaiAFu3lWT0xLf+GvbzbigYt0PB/AoUrL1QCH0VCuE7SUsOocNOjod1D+IvBkvw7oLF0TsCMvortpXzwe/F+r4e6N/1B+jfZ/38C7WNh5qnL42YxwdhkXxAy4ro/Z3bPJvGY7HMJcQlwl49jqvSKQviCUgE+Lpf31byLUyY3CGyYD9BUXPMgv2Ylk4/GujTI36fLhHg6xkAPL5N6agW7ZtpL8hOirhm/5DjRwykNosU55V893i2Jk9tU/ITt/mH2kdZQlAdpH02g1ODqUuFoaEqvfsSvTM7zQi0mqTRfSliSGSVOw9piwvEeJEwvXv9+W3K16Q7/4fchyUsP1G1uvmnA5pZV/FtkZN4shb6+XyyQ8w73AOJkJv/AVIdP4Ji8y2El4rspEqpNHqvDWj0LoNkaIank4JpA9NVOk38J6n5T/tqrATaxZMcEgnR5MvWJvtn5HfblEzKow1L1NnTeGJ+gCfmR7j7f8jw+0BymB/j/ntmP/6d3dSPwU0/g729QdiBNf/zmUou6n3/EyjZ7Lyn37n2T7D/V4sD+5UuiGex3ef22+6z2O5z2O71ul31/s/blLy2BkaPGkhdKzyfqJ6O8HvbFN42YN9E1rSIc/8cduZJ85e0ftM9Mat31k+hQ4wiDWIrCotn/QL/foblP4vlc1YYeuf+jG+WakEerki719PtHbCvE0rf18tRMsGJSo56iPAlwKDPw2cCdlgA0e2Kv0Vjl0FaPPUC6y87CaXBXEj9HPqante6Y4q/UIvfpvlMvPAhMgrEWFqfx3nPGorfqrQKB5onGa0i+7uqQCytsdvV+STZ90/xi68I2j0n4Og+jXNOOtO0hj/37as8nZXx2xW/sxe/Ip28QfsaUdI5L8nOJ29Xa6Xmgd6LsjG1/b8e08H+mJReswVHblc+i9X6u10vBCROBltK3I75njS8X1AE+TAofjfrU4g2uxJ6u0ojM3VdVWVrlBEnImy5UswC5Qm6XwR0y2uotZD/nd43fAde49+BXm5Q57sSsrJSv5mkff0KrIvOfIjt5A2tMxLH9AKtnyXlwEeXF4wzFtBPIGZlfruSjfWmaDSVAW3z032Zmuqd3Ef33NIyoz6sYxrD8K986LzFeaVp1kt7pnzOjsc5u+wAcxb8CpiGD6tz1eLBjhXby2HHSwg7XuTzpcY8sF3xZUlDYBjrAlB21dsRutJOo50U0l4nk/JPptM20ByCFknS1YGURZrMNu3cpDwXIfCrOCsJUam527s0d7toD4v9cUFeFZ5uG54PhgMT/X5txn7N8PuVlB+xSv2J+f15x3RqBpor/f5UYIl6CVcfi/s8CmkjKTfv06tt/1KvqE9N3KcJPnw6f7s6g8vgr1o/QMGui7frGNa+P5OfkQ4TS8WRwmv5Muu+um1fxjfTOTYml6Nbsu0WIP+ghsYRi/bz2uuEfmJdn6L9nPQ8bdJX3p1xxXaFcxAXoteux1s/gmWf9SPwkMSWJLde+T2B8thH7fMrAr2JJEcAC5anfzduV7pSKbyTqFw+MRNp51I5LwbJF7cr+YaKd0gxL27BNpL8q9vB+m1OpxLfxdWj6LH4nHkClP+dau1D9WfCk5148HbvduWbkmIxu/Z9ioJO3Id/e7CF46T2UVY23oju/73bFU8N+xXmfhW/DLQmNItu8TZgHzMbboOetdM4JmpL2ODYZDTHnp7RQ3rO+uEfQPHC6HfAXqWhY0leXKl9M30byzdSPJ+9NPZnhSqHeNIjwXQU8d1fi/uZyicrfJrdHrNBtIRMSfEJis2/EfcLooxf9/Nzod8CffWEjlZujs8aPXO8b+aC+yi18E2hPDy1Q7/5d0han4HC3Ccw/zs45xVmv4V5oYWisBDzmn8pnjDdx24Bp27xwtGiZ+4YUexy5BMryJKnEnp6G0TP0WOEihtkQc+cMSKbyMQQIsc/Igki3w+TYpnYSkwnJXlTo3RPqEEUm38vzgxXAP0eZ1WYroX7wULczLoNsYRx3uiwlWroWYgjiIRkB2lxhKlN+mpRCClr/mo5f5WzloA7l9KIDc+ldIzTdDKUNdQErvUJUcE72ZyAczMKaxaWZB8vlq7VdI/DWmIt4B6HtcSmAKWpFrJBzsUmqLhR4k1TeRDqEbeyPCopk+JIxj4xb5aXV4oe9SrObiZ2KAy6P4P7KxJ6VraCNysq/akR6eWyPD3aT9N9YTAeAOcKOJdpkD8gDF+K4Me99TYoxL4n+uLfFcqLQDI2XxYXSRlNV8TcG+9jzlGKY6sl4ePMiY6CF0utSsdSc2+lP2V1F4wK9ipFBRPurfdDY+x34N72beHe+E1ck9fFfZwq2g+R3x3LvQ1z7QfoOeTe+i32BEtzQbgIRUjLxZUHYcyLh+9uNO+hWaa9iW/eGnaPm4Mn4A/iFvwiwzahP6BYIIZT74weCr9NmF50Kt57DeIWmYiSzNKzW0qeq2jbkleLi2oeSBUhiqli81NiKEIUzijvvdGCWOl04nnE+yXufeNaWWjCvQ8W+b54teR3ZhLW28fzPA16Wki+0iJsqe30A5G8j/UjeTeae3BMT5IkXLDXKrFV0/zK22/QC8wsyL48kPoVYrIxXXbLB5X9nis2Bd/LkTYgm3UUtOzdCbYBsvg+yp2reEoqbvMoyFQcBxGpfNMPdn1STG6meSYI22PcijfGUfAyArneuiOhd8xR0JucAb0HdULPuJk6KoV5lDBFyvxTBGHvhuojYaQP0G7dXga6GQ/ahTB+26xFdGbsHymuS6qbJLQm+cImX0cF8xZw8a/DrPTKGD3mreyFqtuqMLJh16Dz2MAWXwXjNm7P0jFLTsb2Dh3RXtF+kfQGRLDVgVSjbu02PsEu/ha7/i5qZqselPLepzxs58vgxVenO30RtrOR9Z9PRFhwBd4eZFVBvlI8765JeTaQVtEkpnYryGtaqpffFFP/K4bwTCQQ75iNOXGZMbG3FvXWhUWzVS9DzGcaZ0wwKHb6beDiH3mA7enCnh2tehsqVoQLR98SyHuf8kKFeV+mWGuhlcB+9VMUO4pO8Rr+X3qYTJhgyRBMCv3rJRt9nGYZzzndZ4M4F8t5zp9E+OPishfMCz24Yy6URVdKI41wtes+xI2PAXcOQg7z+95aIC15CWFmzRMNOk1e3kWIn03AnOBKurMuFrSaYsRqunMu/g9XNGOswv4g3JqzR9B91Q2TDEvHmD//XOVzk/pvt1YEdBUEfPJcZR9Vjod7FrLkv4x2ZwZnc7B5LJBtwFFGt3SMfvOfjJfPn0ccUzx/5tE46xcJ0kb4IX69kCLjdWN/8C9pHScL7XtEwtTWskajcTRFdDSXWX9li+eMqWjmqTDJ1DSEQdq2BqaV7COkcP3pMW37/Vns9yLGd5yE03pEeiFouNgl5IoLSdPpecI/EI88FtrSFB9EYcSTPS0UvCMcTPd6cY/s45TnFjunfZ6oXC8OEuHVBFVL1khU4tO4LyxQNknKNhv3iZdWvBqDYbBtEi+U+Mi3JvbBsfvh71pnWOmw3n2ukidTPum+860e7xf/YE8ihJF5kTAoiuPrMA7pBiyZIC35VlDlYvzrle0QE7kMRSh+nJ+S8k6j4Dymnz8BBecJ/bxIFpxH+TnvfJu16vvwNynOCyl6gd40MI1BWs/Er5jBvT7Z501+19tv9rvC0y/e991f/Hcebv3UuQqH9TT7HCjpi9gBfZEZ+r64QNsRZd+o9mMdPHeu4mPq+FZePD2EznVg6/a8si+eq3SJ8zBTEL6/vzK/8cs8iDTBg7C/Mq99SBka25/OVTwRb046cRZabcLUHX8OqK6/n+vh8eQzy02bgqkB/HUT85QPvwriXLuZeSKf6RNaWiUJj/fqgB3/Th2LyuoI7k1eJ/ie8O5CQ/+Fd3h6WKTj9RPmwSTlG0pjS5K2APCeI4358Vo3at/1r/D9bO/77jFx4HfDH/DuLf9dTNs1VWFff0D2n1DD0YOV1uQ/BPmLot9LSYK1U9Er6lTdCe5lpbSKFn0zUyM7WM+a/FeRr0PGucOW0RFWNMwOSXEV/ieQT19eyGnCyOeHEL6P0tTBKKIOGrzvIrkIUgfne98xdXB+4KvR+FXkNHBH41dEO/RRb7DnEbKSN9sR/8fSyQhSdQK/J++k1yj6KMp+ZxFORJKRlTIRKe9HFfeD6YtRQfpiIvfrUklRhXX9VzYQbWR0RD36gmsZg7VE8TYbg7VE8a4bo2qhVrmkpHY/ya1/iv9fzjn1+H9lhHYAxXVLEMsG95GADWTryGv6XWFofvzMHcouY+T62RTFgqJ/wDyy/YS5RrG5GvFFiv5JsIlqF1oHEHzb8fqSPhbWO5PbrdV6lgPNo2QX0qFUA2lHh+2stcZu0/bVLC9H/KcJcibFE8EzJcdB72xDdMgxkJ9tiRrEPFtbQ/gsRSfxO7rwhMEsoesKUYQTd84sEe5SEouEz1Ppwb7cqn11fgncFlPvus/i+TEMQ1ujJ403ZQLvJzOQ8xbnOIGcmhDlVAdy7rYopy6Qs47LxAI575mUE+acDkFyhgr9FlexMGBXBcp2cn01gZwjsb6ifRaUODtnsieChEFjKdqngPDLPogt5VtwthCTbEtzfQbVVxuob3VIlXEcVSLHLYYCJe41VAm65/JtWJuogbYa3nH4dgp+nzQasSWl50fzvBnneBXjuZfgPfYicBzABV0B+69F4C6Y5ad55POXCLXraCe5+S7eeeTD0+2dxc/ks7+YsmUeetK3Qk/PrYh/LCfdL6uk/54kbU/7VOx3zBhoflISp67bcHuuYktYg3O/y7lvWG7P9fvkTkFM/Gz62uIci3S/hYoMpX2ALON9tFLb0gn41A4VV27kmWnmnfxH1snV48QdHYUe3H0UKcrmNaMT02FYLIlXVIBaS3WOktzWWJ93uRvbuojP0W+Fiv1VGvd9Vq/xOHhjHh1yjSeANBcp9U0rb3wHvDF2G3njISjNQ974np/age++5afuxNS3/dQgph72U1Mt13jMb6EJ333DfzcrlDee9FMvYQvf9VOz8d33/VQRv3uk1DN895SfWoPvvumnPm+68KhqD/SqEKcWLg4leK4+xnN1vs87vHOHoueTWmcnobmH6tdNjBOl3RhmrqGbGScS0k1/mTE/N0Nc1Q6WcSRkKYbOAzs83xJjpenFwsAFQsrJ15TOiI8gZXQq4q+LENPOIn7ab76POH2zbDUHU3U8gtGsCZ8zQ9oaKvu3AbteWw+QNnLCrDRVTtaqxB1Tj7tAeRgD9lsaBaVrrGxvWthHiMePfWqH4i0O2OfwPJF3ngF7QMNonSsSPv/2mR3KH1QH232qnajK6DlijmRSzJD0jWrn5zsU7tePtP44FY89tUWKALaq8JeotrGO67vglzuUbXDQH4zi8CiOpTlec3pCudB0j9MTagwdAz2rHoCWaFjmonXMtVVpM9wRxbu1/nLIr3ww4KNxm1wF2RdYq3X5A6zVOstya7HUigcDPiK3SZffa03Tbyeglr0xAbyN/XzL53nUBKJbZwylf2cg9a3ooEzoLcxZZSCuUX82c34Lq54Xbv05fszqvlXPib4VPxU14eXQt+pZUR9OPSwhGT1JFvukXHJhRdjdiHRu5Hb8Gunc8DDkwjeAu/FvIhfZw78kOVN+2Om2uhR3xJtiEEfyMO7eioqi/SD9VqqSu/iur40ODw+fWxuj/2lOI8wjO5XnFN/Hybdh/qGZwm1U/ayGWHXb/5ngNGSlU9VRVQH5h2eKzuoI/5K17z8gFpM27vmqVua6xZm3ORny9ysvnrSK5Lnuc5yfAtV2JroHaqvoibhEeLPHaqsp5TTUJvi3sdakX+LPRXklLoesqfJkzAY1juxnVE7rcQ6462iUFzCHK7/yWTF1eQTaFlFUcvGFg2HacAZP+6CdZEu0S3HFEpLkfW5RcfeVzA/PEtBZimt+6OidApaS/PPGyxEj+7HYTVxLnOPdPMdR1uN5kH7NTFjthDDiYceBe+nflI7LHPZQfc3fhI5OHK3FUinc/9SHSz2ZaiA68WQ/OnFYNsp7uOV8OuAPNXWutun/3TDNttLg+SWOTvE1IyQJGuXU0bwBz1sYsk8nwopvR3fGYTimF3kPJzjmqdQxT0t2DYpv/y2rsPAx/Xwp8/CdkNrZkTDO9kbif/xQ7KirwDN5G7in0668mX/Ld+X1vCuX6Dd6F/IeoIiWeg9W5ipdbw9WNlYq/m0lxKpyVWR58TI8g89OY64qBLTXnIOyh+ZfmimmHUbRzV7ht7mq0ZCpOgbyz84UVKbnZ51isOticeotNAu/4ZHk+74De//eh/9n41SGyh4xIQTO2Kzhpcn/DNcRtTluVB/5C72dUmJP9hZxS0d8KnRUtEKtxTuvbwq4K9TOIwyhfRSuehO11df3MLep4tVfJlS8+v4I9uKi7zCv4gLMVzyyC/w48zTP4qqD4QRc0Rpcjwa9XykS3DeI+6t5lApu5ncqu8iSRknOHAsj/L0iJuV4HiyRdr8depwxzCtt1f4PUyL7WikO5aKdik5m+0TIGi78lXnvMKLtJf9B2/+Lbf/pA9o+fZ+232MZQIn+VbzggZ2a3mx+WtjSozfVmV2/c+RdQmcnCp41AJ3jC5n2fFNcyOc4os9xhO7eyOHg7sC9DIeCe1Fpx+p9imet0dun+oTmUzNF26XvBE7ja/5pDJedxgQ44UTY0vF3LtpJFmrUz0exn/MNsvVsMApih6DbtiC2C9L/OtfjfhrHy2KPlGYaqTyBt4I4Fn/vh1Tifbz3jgQX8IaA95mPaTKVTN4LhfTSrvGOIL5oi5djuuIt5ovWStd8S+zrIzgR8Hp6u/JUYL+gNT/eHCaLw1WIR13O2hImTDIpreysPg3KzsoFhK2gpD4lC0hVXmEtUW0/e+NOZQvBstcI3uK72gXFwqPIffScjffsOiKQc4TISlO2RCRQjHOKjjnE8MaAbf0tIcwNkZ/+hEVjZWtBteNCJBlYoX2+e/EBbt/pxQcgmjvNOJAXP/VufHc2+1SuR8pwFLhTEVs0EzzLkq3Ub0HcfQzOY9IwRQV77E2YVIok0Uo2auNMPYYrvMAwtQT+ICVv6N4psraq8Ugs80cs06/L9DSPEYu6nhD1oNb+RF772UgTulNx7c0xWArX2yAZ7RuiFfv1sFC+gblfktIDqfEG6Sc/jPhrBvEetw1Xo03ddK0wSTqIwXq+vJ/aqXVIfDlXV6Mn53IkzcuQrOCIQj1NDaLFMiLsEXLDLeCMXrxxtNBShhC1ORFv1pHysIRU8rDrgvKwp5XfV+YD7lS2gWr947jC0wKrPW3Ear/srzbmT2yJhIB9FY5Y7yHuT9pfby8uze93KtosA6/hjC8jGYN8SvQZ3xd95pOiXjIOZuZZ1iBa8eQ4OOOk2Z3AGRfDFMWOZjNRwhscnE0Vz5Ljau1UflTcJbfh3mgXXozHHPHo4XFsM09y1ZV4f9pjDYb5oV3CDZ0nyLpvCG+mDLyOpZZTqdAPdG59iHeCOFkWF0hpDSHtvBD7FcJ+9WK/LOyXlQlhv3qxX72qXwthUgi0LxKc613KViHow+R11jYlmcjHhY3pvwds04r2ZUy3noG4fNDnRGLXvj5VPB4g+06x3xH7z4/78TI9mQjnuwlJvtYgwPdUPljuP0A9r4pg7N063Z9iqob9dnq+Usbq8Sr+/DLxV9ZEWSb+BsxZiGREJ+5tknmQXRb5OPnbPj5KlL3HaF9Hd4Kus4h3opYO4IqSdT89k3cc8oejnmex1m4SiDPzqva6wrxh8nqyS91zqm/k66S8bdOTt7BWmfbPu0vpL2XgaF6xi3DFCub9ULDu0756STfLKFur2d7c2P8ngnGhj9ml/EJk4Dtkt0o7LQiZjI8Jejb5+XxRb6i9dwLvPdmN8KXpPtY+qmKpF9nUJeEqhvoIvzK4AzMK6o/3rOlYg6MJJhmg51JC7y5lPzYW5tMc4Py/p3ndSVEwlKRj5JzERvgnUzakFizepWRRih/O/HKRlG9rfnjR/meZRXpGbkJYdFGo31RRIFu9KBRGKZrykMJQ2EY4SdaCylqYfUglLVHuTcrKiAI4zmDzIVLIfrqFU/M1LXkYlzqU9SorLfXk0e6kn6r2/WxQMXZI7+B4zdeZz78W+x5R+ph9cDLep5bGgc7cpXVD2XO9hvzqjtf+Mn+kLW4n4D1Q5d93a3cpP2PEd3JTjwg38xBrXAhZKrO5rMx7WOavfhlPV2rnLnVvEq2vPI15OsICLtylbOrYpqZ6wI6BwoxIGhQnbzxyq9aUIw9nKgralpCKgkbyBann3rckl15pdS4VVHD89i4d0Z4RaM/k9nbu096O/6A9T0/rml0Kh6O49H/TZ9vUnoZuwHdHl8427uu/4u5CqIM3pIpR5oh+gySE5B35PtqrMhsqcRIayCtVSQYXsHqqCFg9naS1QrPvkF5Zzoj7mp+thvKTZGiPhohpGZ6PxDGg7iZ1fmaDisUbhTt3qVhBSVhhEW+WdAvISqJK1kO/wHGIIvycpGayA/d3g1giFV+TcY21iitL52WJofIJirgbFId2Lesp2HJNqRajw5iKtfQ03Qo964lj2ye6V2QQo3UMuola4AzxdyULNc6HQs0TAa9uPRmlD5U2kjBd+9boWePl6fMYptULE3QKE456FEOfMEwKq+h4+8+tDBPcJqkFR9QNP8pfUx7xGL2ywDRGCBSfi3zZ1zIMU/G5yCsf4VHNmD6d4ZUFa/R8r9U0yjY/FsJ2olB8O/hn9L4hHqKyZ5rN/pgtvCnqGb8gLEJ59iY4RxbNlvbtQHBNsv2v5UvOk0YPlMvQy6WDSpJ9kJYUYq2xgdQEbpm9uOIbi/f1KKZX6uCX2L9PmdQPspZ+2ErKt8yiXUmYAtxLuzhUpOgONPtwHP4uiai9QNSTG1V7hPgnbkztC4pUQfsCsd1rcR/crjj3T75REe+tmA65ihhSaq+J3bGKCoKXB8MNhlNBsYaBuQWI71Vmjfw3diPNr/bGYRBe7cieK9VeODWUtNT+OBV6vlDKC+s8or9fQkhe6aXtX5DGkeWl+hHSkaXjq0gL5K2pUBNazpGjVyCNX5wPcvezVHYhwv0Be6MgzPsc1lTrD9EeepykBZV953wHKJ2svAcK5zyhn/uMwjmP8bPmGWC5p5hLQKOo83IF9bAOZ7qWe1in9AESinNQ5/ELtAax4/EfQu1LL5Ha13KoMTQDllUi3lNJfJjtsr+S2jyRvChX9b30MFCaNCL6nn6En90BJdOv4vff5DyeXdxdVRCxCY5ewr05jDUx8plHuDeHYX5dWf5TOl/Poyz8+AnMWSrOYjgWB7dfWWIfCggMsW+NSGO8LXaHyHKAbSlxhZtVXVO+V9avvW/0YU5Snor32hniLKlmv3QGkpUns6XOj72Rhqgn/SP0SLKPaZ8tlcG6W984nsecfwnbXK7yiYbbe1If5iTNI1Wbpm7T9ySQrDyJ2/yJ12YltfkTzK8Gle+uVLV5/VkOyZDi8S/3vcdQfglfUv08GIrDxeYnxESCs+O3gjvnci+2j9lhjmat2/w87O0UfPeiaqMS29/73P9D2rcAxlVUDZ97d/N+J32k76UEaGmTPqEt2UDbNECym7Y2abHbYtxmb5KbbHbD7qZJWv1EqGzLS0RIC6KiH4EiVVBaUITvA1MUQbS0RVHxgwJ+4i9+VkFFrfY/58zMvXc3SRukj7kzZ86ZOTNz5syZ524mKFJMzexirn2ZzRU70+hLJP33kH7nCPrvSclurhhIoVvuzuWWmeq+CbZUPE+td7q5Ygdcs96l7/fhnPj8AcRdRmtVWA9rtZ/Smc1KgtGean4hwX5Pq9IRkSbV/IpC+o3X1TI+of0O66YdpWVbIfL35n+xlDQHHSdOyjYjVPBHmoLirlmfrT83bY6LcV+63fFKUanoO2VYzv9xSNKbm8usMp4zkHKipbnVvh//zUXNx5yhVhf2KFc5NNcm1bvXruWuYnl2Z6VO53hEqtfAcxrdDxkGL45pG2tvxPG/EjbU7oHSmiXH5qSkwJbBTJHGTXwWSFCLeQv133tlDpsdp4YE5lz4VBo3tHe4Qp/jIqiQZNXmRaLNG4ehdCbps5mQW3ZbQUHZZoRQzJbaw/xt7hZUBdwXMb2Cz9Er3AWX6FsaVfmcPaBR9oDm2QPcC2yJnqqLN7AqRK9PkfapfA6M5wkYpnFlinqFA92l1lscQssJCVCtquo3G204HdsYa+GftKJPI/VCvqkj4i9gzbU8v4DrcgWPeh8lXa0JiaERfgN8D7w40gn/MNQgtRr57d8IF+/bof1WnHqSq1XHNuKTECv1OdnLae8PdlFt4Vya7mVSSqIdaSdtIp2W0A7zTFHUo8jpeTmzvQZ85Ba/y/v4f9KKpAVGdwGkjacv8bjBLP6783d26Nd8uKRZf5kzW3znog1SgKNiWTHAnHL5/ZiCZGcVw5wCh/8jBcD+zGLIak8tIZ1DKyvRrLNoQvrudJxZs8+yqXNFdAPVrsPRSpZ5xpLNWS5K5uAJmCf9+3KGvZJn2MI/0Zp5j12TmeOsyTHpdIvOtWSxoruA6cTOUaa8E1XA84EsOV8v4LVNN8526bwL3d5YR+uOUA7/IsnC+W0upnkTVMCd/AbYTLhHpzOXFUB3rSZi+AX89mO4ziXi17rIBq2AjbR6heEnXeI30d510b2sC2Gum8KVUMNnNy+EIIcXwifdtLq3BG50kw26BPI/VfPt7N9nfarmN1mvo/tq1s/Q/VXWJZ+qOZr1InqPZ72QBccz4bksOJkNh7Lh99mwJ8e+e/bV68Ud+E+vnKQfyfoiTu4n6cUX0O8MLsZ5nvollQI33WFajvMMOmUqfonFbZ8zlS/eXuAWtrW8U1ak9pQPXC/muZPgPU2leKZfZXH+JouYN2vw6PXidypobkNzE3Vmk2ZhZWnnNLGX59CqxlR6Txh1SRlc4KIVmr8ArdJcAzfz6gLNUGZrMbQ8iul3oIDs7Vn8Fu45LhPToxkevYDrcrwsL8q3jK1/L8ziX7jS4Bnk7RDb91MhW5uG+mw66K4ZkO2eCQl9hzYdLU5vBo0A9+lkNdLL/lOzd7O1mGS9uUy93Z+1pV1amVkB15b2Z6F50kK+p/BxF47B5YvR/2Pt2oz8jOWZ/6Gv4DR36M0dYqeXduzQStYWZ9Pa6mwQs+SlLvu3yRdieIV856052G2d7BRvwMmZs7U+50y3ebPw07m0yMpebfE+0nw6SsiPtI8D/UL7p+UvtBNPC/XZOWJPCSUmV8yLcyFtJyZveV6FY9da/NJXHuTm025PPuTn0+7SD/Nm51+pYgoixf/Ab14h7Qq++i/aDXoe5vx8dvZEzGE23Wbz3Ib2Le2z5TJPKfnlLs99S5P55c7K9crTzrPzvNB0QLyQ9ULB7HzBbT40f+12SLulguOodUulYFZBB9rl4ncXaZ8zsrBFW1xIu5j0ahTdYf852kKzCu+G5YUnTjf/t3gT6zV234E5P2t+QoyF4iYR0nyd7MQX3Pm5zQ+hLfhdkTv9GjblTvWyPF/sxCEsf1a+eHs4H+FTCzr1yMo2bfGrc37X/NBOSftposlbkYc2Yh7q/yfRdvp6mo346uYnhf2kQ/NDAyl0y/OoHUOnmx/aAThrw/nIC7KVmrd2o20hpIFSucbzG+2XSFFaUJM3GWdpA9rivNw8L9Jn/XJOVvPMJbzH93pePqaZBc1Xd0N2WR6Ordll+ZA9oQDk6YHC5od3QPO0JdD03a/BxsZq2LjyMsBJEH69qJToWwP6ZICmlZdCRjlqMFqHKi2AvvylKKv50lYgG5R6N5WHbFggOUJ7lbhvPiBsWJB27asY1/yUqOcChFE9ix1r8ZJfAbefeGnkBOQXzSrCFi8SL40UQW5xTfHb7CdbVMKKIisNbfG/5vyy+RmnpVMkxuAZ2AY/TpkJzbDq/ytp9usTwk/S8c2vND/vDLXmoi7ORQnNFRajkCBqrwTaqTim5KKdmuuBDblop3qXPD8tBXMFyVoJjbNbstRKN73Z/rylv9Dq+FeznlSv5unCMlpO1igkxZyuYhhecbu1zfil8BbPYZ7BTXUvYW2zmrUNadR8uYLR/Khtgau4xWp1Q9wjkJZjaeptE0sb2dYgWXNf0NQd4IczcuQvc1Xu0sCkNU/+1bxPy19kwtJ+PO03BhF6leN3mkb+fqqw9bgfS+g1H83SmyfNeb1A4/KfO0zvZmmbzxXlVydlt5xzWFpTt7FlJXZTL3Kp36f/GL/1UShXvZRtlf6+7K9ZjvvpN1z2F7gv4LfXzlfj6qdq2qrA3hO6fJe470rj89j3z1PH10weX93g36V+/5fW78WOCI2T7wPZScvpBVxpKy1ePA0muV7NwHmovthNI63Ov/5Ea69/pXcJGD4n2/7N1zZ+OX7riF0AtRY7ke0oF98DzgdxH1qsvi75lHneEmuNOrDLuUb9sO58p+Lju6w1XE+hfPdAvYvStku8UdcK7wPdbmwuXumYB+DoV7zKEc5m6z9S/Cb/EstsrY3up/PtS9pZEi8ATtHVC4Dqfdj4LvG7F8u1BZjWzzRxom4uNJc9iPPLMv5eU/yqbs0Oc0TKO62Ul+v06+9Pa9ZpWYmxw8IQ3HDfggG2GHRQq4ON6auDkka94XTdLnHuu2n916Bp29fkHibdlpiqibPeGVrpRDr9Tf5NVPIMzD3zWs7d4Lc38uQLKDdjWtfzfpr6Heri7XmZk0D8DjX5F+mZUKPHeS61Qo/C8uws2HjdjeLGdrvYPV7t2nLus2LtwfVFuZMsfk38u9vk3SMRzrwnU89c5M4Uv4OYRWc6Nu7CtNzzoDUT08+8jK2kjW3fh+WZKzCvTNh4A2q9zCUg1u/ol5/Ei+pTc3C+bT5rwXMVPBfn3p3PY6knuUNcomYsBdmn22HOE1KSM0mSM2F2JvafLOo/dO+NpCZs27mJTHlP4r5d4t0btbbrHrG263as7TaPurbL+1C63IfSxT7UeY59qPmsKSp5zypSfAFL7Tz+rZTz5Nt+ZSDXh9209y7Wh/mkR8aX2BpuBtIodApV7mRl0E4WQSv4zRcNmqzzpxtQT4rHIkTvKoRnJlLYWwlvzOGu7PMNmkM+87g5dDh5WDjJN8yA17vFTJpbk3uTe2HvXCRp0GYVacUlz2ozi7QpJaZ56ZaBqmTjpTX6Tq2s0Fvlhafmcnp7AtrEEnh9DmZ1dBY9wPM4utWBpKnNKE0iZcDnSyaTAV+lDw7NQqSnphFnX56UY+0+F2qTiwKYFTw3uRjS/tyA5FpJkTahZKhfKykZ7NfKirpqtJKCT2ilBVpxqTa1RJtebCY7a17bSf6JReb9u/3ahPwHEfTPrdrEYhO/O7Ty4pMnTdNnalOLurVJJdoFJckus1MrK9YmYDIl9/fv9lVW7U3u1ibnf1Irz99v+pJdBDF3Y1oEuUmbVgrfm5LvZE6bXri1f3eg5j0zsOM0fG4Glf7N6Vi8Ibh5BrfCl6fTFhBU79sxBA9N5/oKYHF2mI39vk70+MxvJ72V63w74M0pdoXMT+4z98E75aUw4s9LfyKutOLigFZabO44hm5//9DQ8FAl1tPMPK2kdDhJ3gl52sSyFzTk95GpuTb5nV1DWlmpWeMzDyHSxIJPsKtHO+AfjvxvGBzUSiZoZSU+bXJJYK958lsfw2rXpkwwk1uPw39OcSQ4H+ttq1ZWAD+c5IAWDt7o7Z+plRSblVp5kd/rPbFl+DLvzH7Di+B5ele1vnM+/HxiQUrRGpI3+XwkB6d9Pm9gi79mEJH/aFZ6T/sasWylXm1aWSemdOIyTEWmtLVa75hvwg8milHqe9OEiJudJ4YHff2Vuzu9cDfJ21/gt1O5JToCR83kz+EAAV+Bo6JJTj/mM8092rQS+AfBzU8m/5SEt6lLFgZ8j+3YYe64P1ljahOKPmEePQ5fn4Q4/TsC2qSi3QFvI/6vgZ9Rn52/b/B0ZwBO0R0CmI8IpUVd5qBv513eSl8jPDyFGSj0Jf39KPDfgmNTM2VtBczHfQHTh63qg88Klm48PZSkclFf+S0y1bDVO3Rjza4tMDgjQxEN9fvoz+lBdDo7TXh4hij9sM93wgfPTUTSH5Dzh0mpFb0ZK7ZTfwY6jpoz+/X/+tTmwOmhRn05Br39+hWbA4EAFnX34JuB075hLEP/n7zo02dWc3xw85A5pJWWmr5KbUGJXthxkqCJR7XySSfIt6NvgL/9Nyyk7xb98U/NhxeQDa82qbgRvjITy+KDb7Km+AUG5m8NJJM+H/x2JjOv7w3O138/8yXfG0J1VB76mIm++69ervdqEwpJFZQV6KWbtVn58N+CpFPzlATgefoNzc27SWArUd8k4UECoB66y7e36pDvmJ7QZhXC52cRcIc2sdQ3fOnL9PEl4ctk7Lw0B7XWvgewvPCtOdTyvkOP7THvx97iq3mjBj+QJHB1V/IB03wA8+gaRl0yjB0O7iPSrajf/nIBNtf9A0k4fQG12El4hb47d2IxkoO+Tl+ND14gFB/8gD5wRASeJQP5tvMJcge5/ceRi60++MX5VEkP0EGnGb6AD26/QDT9jEGtvOTBmoDXxC8MMclOJPHthBPnU/XWXG3WqL/w7fOF2jH3oRY2TW1mcef95ByCf54v0kNRNfmPr8p3sBIOXuAQt2PnszL3wa3nI3u/pOor3HMUfnEu9RMY9kChP/wD+PZsqvxnTRxY4DUR9fUKKadm/6Hh3VWP7Xy83zw0tPOdfvj1rCwpiB1mP7ZtpTap1B/ox2Yu9Q2c8J24Kdk57Dvxm2FtSlknKt2jgTd8lTuSvsc6/SirHXDQw4NTadEOJKo0vfDFCtWPbjSvDiyD98+V4Y4dW7UpxfDIeayRH6JPEu4Xn++LzzPic+N5qo7/XDFaHb91LotNV8C7BX4+G1tlxyDcReXsN+8KJAPDB+ExDFXvwP7fCN8UFXCY47FsQy8js8kheG42K5RhrOih3UNw02xG2y0z3Ozbj7p4Sun98HqFUBR7sVWGD1OjJH2VMHgeNWNDAKupKIlKFf+dTA5hw/zzPEdzHTqPRBG7L7zsBP/WGThKt1P7uu7HrnIUXjiPuiDcJpj59WwH3j3Mb+fXkHscX0/7kp3Yq+4UiNfTB3vt+6JM/aYvEEiKP8PDZnL34Ink8H746TlUbc+SNNOYNLHkblP98d1Pyb7ZCV87R7Uetjf2+lnYf3cfM7HXzyrBLLHbTif7QPb68/L1j82Af3moLqqxmHtQCw4Peh8wUXkPmTRITS6o+RMc9whlcqBlvn7ccyfaCUKfVJpSn4iU0/TJ17jFTsBPyUy55xxOghJ+UywiMpMADrsFINPyiQHo+Xl2dB/8XFhIgYAX8zEbUb5QZTRqM9Df32Xu184tOTgPrr2QtY2592cv++DUhTZ9g1Y2wa/v+BfcdeEIs0ibXQj3XVgywjy4Ya22OC88DO840/kzp3Jw9FT+e7RUcDzRZhd3DcMXHeW5r+XEH3F0Ri02oPf0wWsXpthCHTvhsXmp1tFkTH1e6tCjTSpshJvnOQs5aQLaMJX6wL/gJ1wTR33avCI4QiboDYFD8MJcrlarAUhB77qQR1j4LH874DPz2Nh6iOnhz+LzvPjcP89qozcvdNqcs4tgWNANis+TbLI9u+foDnhtBmu9rTsDBw+iLVX0p+Pw/AzLqq1MvhwwaaSDW2c6+stfEaNwiw97/PHTVfDEDEzyV4WiK3srKyvh2QJS+QWk5n9I7pvwboGwFekzMDg8jBaFb/vhQepFJ4aHd2MQjiORWbMH3qFDH4VJnxdeI+xhb3hXGP6WjzLeU43D/2Jf5YI9gwHnH9hfAPMrK7UJpQf3JBM3D2HTJYf79Z03fFbvvEEcTSK5zVCVU4l9PeAdFInMPP1NeKCSshzSp98wDMlKKck3Dz57v9nl82IzzERjG5uObJkafcdm+ON8keaQnve9N+D38/kRZHh9PqlW+H0VDa/w6kIan4bh/iqO9VFXxC5WWulD4y4Q6KeZQuWJSrSFppWaegH10IloKt2yCKmvhtuRutD7RufpfvgVpkdx5j6cFcCfF2DHhe8soNnzUwt0tp++fcWz/cMP+O6Hny1gaek88TjaOfuSjdgXA4E95l69czO8txCbzAwMwDuUNDU1vMg+MoFME761kGmP+vpRLEuLvKjkTK/p24Gz05/J4urR6p398k9yUSUNBp9exLXV7+03Ue8/xPkXev/4GDXwXvgr1jT8Zj6WyDsI/1hAownpQvga8jIML2LssQfa4JSocYzBfwfXeYcPV/kGB5ODe2d2DnkDW1d5vVjnlWh4wXtVVOAt3s4B1Xje07u9AX2gA760kMxSbFT8swVewaRxGjbT6/WG4EGRvs+Lmvu0NqGYyusb2tO/Gw1hble4UWF44V1ZVsTu78Jo+GGVlBufNrvU1wl3ijJedlVNEp5YyN1pD1dd3/CxYbgFC1kY8FYFArcN3WL+3QtvLoT5fm8//LZQzk0PLCbVi+PvI4swPAi/X8SGRqD/4FFz4OCh3fjvUP9puGUxgd98Ge5ZjGgP6TsaTPMus8Z7B7yxiNvvnGIyyB4TiXyJE9F3Vg/srDx48CA8ytQmPLVIVi3OCUmdfE+E4Qfii9NVHEo8RQdNraKkiqa/ZQU1DyZr4AHKA0WgBr5QhL4VO/r74eEyWYJ/lJHwnS5D574J2LZdXTVw/QQMvUugv5LzA0Qp1PsbqnwoKjsq4SiG53vhp5RE9fDg4LHhY174LYU2Ywj7Fc5ebsa0ZngrvfCGwNrrhRfLSE72TqDyYkvhJJKM2CTmVQm3I3phJfyNVM375HyxEFXLSt/lcAA93gH4FwK9vppK7wlSEbcUkkR4K2suvVTv74ObithkIWFMFiHxdaKY8EAeBh7II821u5T6HY6aU4p2ogkCp0qQrQ7kdehEEuesz5ZiBbMZAF8o5XENvlGiCa39jPI8VSSkKQnXlZHxcPAgTZXeCJAb6CfXH/Cj5sU/z8Jx2pQsPGaaSa+fbFw0Y72mV5tZRmI4zJFwpJR0QTHcy1mehtcop9PwuIjdy1B4q4Tm3uVFyUb4Lkeg8ik6NlADX+UInOf0J+HnVLzAaTjBX+wT98M7ymsm4a+WH77JaeiJatX3+3dWwm2ccT/OK2ji1JWE50ROqC/PKTp9FJ4o5hoYhv3886J9vkH4Jvs2w/NE2k+V3wlPUzb9cIgZH0QVXtwF9xVTNMcHhtDwO7EnCS8VkylsdsKvyFN9l9np2w3PKuC71HpYh8n+gb0DptmpTZiYfK2TFEpJ8W5z59GDyRupiuFPkthEKbqemrP6RjTQ4FgJa6YJRckTcJIvwjeQqn2c+BxKBpIwVEZM/tcE2QEOijm4t7I/eRR+goKFDq/0THSMujQQ4h/vYCU8pwj/QLh/YNz3lT54DmEmpomfmo9swC/FvjGBbc+36dO18xO7/XCKvPuHX76fBqFL4RcTmIUVO3C0/cZ8NBC+hVq20D+4s7Jyiz7Qd40XfjaPuuCO6v6b4QvzhTTeMR87jjcJv6MoXw3sIRovfEVGvzoPQ/tqTvtqGs1bDugDDd493v1eeACjT3rh0/jxnvZ2mTftgIOqQAfcmKJ5126vdwft+s/w3bXbRC8dCL81A8e//jcqzS0wlEOzu6+we1c2le86ivRp5aXw91yE4hSWAF3wGXou98fZQmkVlwyanV3Jzj2mz0RdtGeF3neDXnADNp45YN44ZCb7fUNhbUaxvrND336DbtygD9wAu3LF8O0LdAa0mSW+fpNsZdM8CF8UMTu85j4TXs4S49bpztOmVlx8Ev4zU6j5hkE4SWu9hf2VJ3wH4e/sx2kMDm58weTo4F1rzcqhxqFHdJxbF8IzDA0EOht9Q4NmcnhI73sUVkuT46Y8YVkHfvHZQP8OPXGDr7Mf3snl2M1e376T/1eDJjIM6ajJtvp2kKlB89PklgCspmr6cZZQQXA0lzv2vkz+/C2DP6+J0ONZ/PlvxK1+DL4vPveivXTQhMc1Zh9nJKVUmXtMqrq7kpXm3hqfqMCkHqnW+6r1gWr9ggb4OuMPJ/3wRWqnwSFtahn8mpLssufZZs1x/mxJciCA//W2P8Cf6ZeTTew4WAJvMjB48OAgvE2wIZ/5ONykk6nRn/5nC9yFLV+49cQWH3wvnxKoCVASfi8N6q8TBN5Cd34/vIyygcL7L8bHznri2X6cwh+gmmkIdHkDZhLbvNF80+vz04wRp104ZMwt0Yon6O13wmczRVN4fb7+rf1JXmjFqWOGaPUbtpgDgeSbl2Hnwqn7iaEbaTnwLp82szgJg8TEwGm4Hb8NsFuGdnFotRhOX5FC149WAxp0ZUVD8GKOxlJILPmSOMZNKPqEHr8Bfsoi0YCm1Z7j8GAOW4xfz2GY78/JwF5f4G4UJa9vL7YIfCZHiKzPNAN7UGrfksS734SHsoWMBSorMYur4AnRaYYCya7AHgLUYu+f1peE79DvEa1mEfmLi4yF3ZWwm/o96kk47aJREW7NE4mZqK/gOv4F7F+LrAPJwc4AbIOGSm8AHqSyH/RqVcXm4FVwiplBS/fP1AXmwx/E50U3kd+LbjWpgmHlec2FJf1fF2meyqRviCw3k2zpz2F287d4vV0zh4e9M7tm9idNOpZSuFffPn+vXlidhEP5PJqWoH1y3IT/0jXowFb0VvpIzo4NaeeiVe6r93u9qBjMpOYpNs2upDmkXYAms888ilPSKSXD9CmnOfPVvqGTA0N/xJkMdupkv/fklj3JozVorg6b+rRqvaPvkD6hr7Omy0SFkvRWJo/5hrw4qE0t0qYVXHqLvuPOT3o/7ttxeihgBvqPevXCPrQxvckqPaZVFgaGkIEBCei+D/vytTIw0CE8M3EWjNZF317fQD/clYO2YT/Qr737vEf3VlaaJwIwLBXRfTeSdWJ+fxhF9VtrT3p9e9B6PYgDWtJ32ttPI/NjuwON+8iQvx+L8+xg1bE9e+7WKkqxZw/RMoz5bPKYF8uFevSQntfn03u/R+XhPYZpBbqx2QsHMtWyGBI8fleS5nxF1CiPZKDOx+lmNRrRu9EIadwRgJe4YX/pJuPvNTcJVKNWWnrF9fCum2cOXUfvx5ninfA7jttnGvBYhtgSOPGYaW7xvtx8eljv/8PufniJjvAVYnOag95+lMMave8+kHdfuuX3Wvn9mvx+UX63ym+d/IJ17w2s+zO63K/N5O/d8BrdMdBWOMIUKuB3Ra7Ev/T9KP7Vee8szGlEZTpREHfcovJWC8XtBLE2shMxCjT6S6lq1j2+uyWtgt2Nf+l7AEtD8Q9LvMck3mP4l8JPSniBJr5oCli/9a473qOLyjfnvqvKq6kyR3mSS9x8B8QdPGqaW1EZ1Dr8mvR7sQTzeZeZaERpiFdRJp3PGWpppRS+Os4vQ6a3UqZD6e6WZbpV8norc0vf7/LSkp1GgWbzd0D8hqfljzL8bvzulOm9JtuL6FUZNVlGjetWpHUAaQpkfRyQNHfDPo6jmFaJf51DVsR/0e5hGQ5LvsNQorlkPayw3nC822qjWpnORyXdRzHGJdtI8RGVaVH+hBu32l3I03dk+I8sZQCnLNkr1txSlqkm1PuRBZrNxwoQS3MUryRUZ9kU9352yjwEbKdsi6VW24lfErFlwyvL85pVL6I81+Ff1b9OS/ydQJKgeA3z9ymUYrdDBm15FGXLsPqgaP9CWVckd/9C2E54BnOietLgLvxOl/X3HaydIayZF+EI0v6KZUTR3A1OKrpTfxfLWA7XrUjbjhd997VR4MWyDURMdESqKs1To9AKef0OywGFO+X3VqtudThk9Y1aq751R92rdt2f1savyrS+Jb9fl99D8vu4TONbLCcij+9IOXhNtv/TEv40/lVt8DBLq4CfxrDLahtw6MpCTcnva/APifOkJe9R+Rs0f5G8/M1Rnp1Wnz4k76TZd4dzrDM1US6tkluXJe+2/x5Lbsh/AH7iiCvR7Du/bvldIXWPJmWS+m6qLrube47QQkt4LMlK03V3W+PIlyWvD1uSQdxSvYk6egx24F+iccuw0J7kd2sCJ1NTlG5OdwXLtN0fVH2AfEsVuE/q/C3QCGcJjoyU602yTim8E26S/pswdJP4TRX8XidxLmaNI3SOV6bnlW1JI4JbfmulHNZKGSBJjEo+1kq6tbBH6reo/ApdcjVsZ7wWmafiUme/0OOCU5D+mzAlJTsfld+4/O6U3xssfa3kMy5xdsr8d0pd+h+Sz8/IND+DlgOFkxK+R+YrakjgUA2J7x7mmb4kVQS7WfJwM45kRH+LpLkFaYTOnIfjndCZdnhnig5d6fB/3fLvtMayW+F/5fc0679bLfnQNBe3V5TlWIcvMO0dcCfbFML/MPsp7zt5lLL1+xel/8uy7PdKmq8glhrvHgQ1Nv6Ew9+Q4YcxLbf8itvI4JB44usu2UsLtLCDP6FLM7Qol28dj8qqzf4hx183lon6QYb2Gvd3NQ6RthdTfVuHkq/b4X+Mz+CJfk34+VYfXIGyrsl+syYNqvput6RNHWEUVsGZ4jTVJ3eOSDtnhK5Ywb85lSt1Zir+Oh5XBFzpnnz8u5tfyVM6SNFGHdQEL5XwnY4YHUeXAs2mJrz8UXWUs46+M6Icgq8XU0a8Aq0TlHW2xHoP4cw4dl29PkYefxxHHn8cZx6nRuQhZKVA65Y6SGPKbmm/3Mg4V0j5D0qcKFv8SpcI3P+AEmkvRqXdLvQ1fXfjKEH0/yNlq1ArZP2z3QqzdZQLem4uuHIrPaB5QPfMgWxPsDXRGwzXQJYn2JYwYjWQ4wlG4n1GLF4DhZ7WYE/CjEb8ZreZqIECT2tHNG5E6iMho78GiVs7eiNdiJjnaY1G2sxYtxGqgXxPKJgwLjfDnFyWx4jFoujJ9Rj9PUZrglBKPG1BM2yEPPFob6zVqA9RYgJUA5metmhvJARFnnC0NRhuNOLxYLsRpxS6LX+mJxLsNmrA7Ym2eZDRnqgZSXi6zXjcjLRDsacnZnYHYwN+SkEkHzOC8WikBmYpH7KcMJDIjHsi0QRCtxtBZADKbISeAckVeGxgbzjEBK0ISRiKJyhVGEZ3T2IAGQqH02GJaCgKExSsDRO2OF6soFjBCSaWzeDZHsV6jHtWra8nTnsjwe3IUXBb2ICpisaM9PQikYHYKr0ZKk4CqIES2NbIbsgMwuT0aM7RDNlppkQIohFxVB4ZV67iIlHP9mDYDCkpsquT6iwYDkf7sDonKmBPRzRi18IFCnxNr7kDWY7FUGBURTiLbjVHIhhrN7AtghFKHmUzxBzHYUoaBnKkcjknLcouooUybVQUGVmZGrkg2IpCEUllIhGMd3nCZjwRt9PqjagegJB4TzQSN2w2eyPx3p6eaIxiRZVadNuNSK+RJuF5GNkdNCPor4HzMZDojUUMEkxPdzDR2kHN09tDHdHRxzI9cXMHdppST7zL7OlJ6X8YF44mqL/Sl3ubHVko/Rui0YToTaLolJTwrTHjrb3IGvWwXAkjzPMrYssrwsHubaFgxZK1mz+ywWxfv66hcsM1/QsvDzbWVl4UMPs2LG1Mwdu2OLF+2bodleHEmm3LQ4HQ2q6lV/b3DSTW18ZBmwP6nLn4vx5cc+rnklMPbnTmQga6gbkUaAig29BA/oYAIQdAw8/cTfgfvVWgV1WBu8qItKLb2dMOGVWdWNEY6DIG0I2GQ+gmunsgb8ESrNtreo04ltW1YMlc0BaBdgnol3gg29saxgZIXAqZXvHVL60G16XVmC869exuYjdALkZmkNtAAeIXXfxk0Yc8LhmF3GeQG4BM/gQIhb71hIL8r4TCVbXN9ZvqWq5aVd9ct6EJSlclsHN3GKFGkpw1MVTlCBvoFeBu1HLx9ShaDKsViqAWlRvDChF2eTTWF4yFOFyGYVKbpPctJAJKRbwxYYZFahMQuD5oikwt1CKCYnexAJMQ0ER1GEmYwbBNTojNqEEsxCm169ZvxjL56lr862p9Lc31jXXrNja3NDZhVEcURx5PkPTYdlQrzB4qIA8sHj2qr8OIGZ6BaC91S9En1ehRBeeNTqOwsUMjahVUW2hSDYQsKcehL5iQmah0mXabQZklqmCaRSz0IffFNh4VHSlHPMQZxyUoRTkYGcFY2ESdhzAsZQemiRwkGK8KZkli0ScxjZDiD4fgOGZ9rkRI9EWR5bY2ZJLUk6OoWLjC2mg3SQbJg9GfgMxaGsLboYDaA5G7e8JGAjVtStDTZyY6cKSJhEjBmAmjOw4TGKMDdV00hphh5B01/OpUqDVqYv2gWgtVeWoJ0dPqxMLmFOUMtqNqq4JiSkMMXsxqHGakQ9BWiKkG8GDNyGisxmCY8hnwmBE0BqLtMW7RSQpBkbTHor09Hjsn1ZYwXUGQNVaVtvYOD8CSWqtAQliwmftiZoIGpzGEpQrmjyDilhWtOAJ7ho3dOqKqqog/K5oi4tFuw66naWmxCaSzLJVZtek2jFWbImvPmRFQ9822MdpMkj9K3ylgiFOegmPGO5gVGrxm2jGoQbtGpO4oWzy43Ugr+XmOWJaWuBRHZyFHQesLmglCI4lpNsJGeyzYXQVzbLRELNjaZUVZySJ1ohdl54J0TGq9NqE5Dad2GRVRCdw2g9NkFVExGiKbGk4sR23JET21PibVUhuldAqqR9DWQOma5RUVTQMRTDdhttaGg/E46GvWgGsNOhPX1F2+aqO/uaV21frm+nVrW/z1jfXNNripbu2aljV1/lWbSQOXKnBz3UebJWoWxtY1162BqWvqm2o3NjWJVNamqO5yR9yGulVrNrc0Ndetpxgn1YZ165qdVJPXrLtqrX/dqjXI3draOj+SNm+or6PkVITCbqqrXbd2TRMUrTHCJgrgQDPVpRGDQtsoIcMFKtZE+yLhaFAKiyFtvkQ02kUOTjci7VWYc2939wBOXwyFfhUKDiZXWYeKMubZFkUNKNUUSr5S36gB4p7N+KeysbFyzRpsXIFO+YSCrNu42dKw8sU8yQhtRAsCQ0KcamleZYWaUAANmHmFkeBWFmpNpWcprNIr65ua123Y3LJ+1RV1soGmKtiGuo9srGtKqeGCK3Gm4TOMnlVUa6ChdVGPhopeTx50surr/X78Jzx+P0X5McpfDxnoIEZuvZ/NEn99A8H9iICGDtpgm9hFS6UBdLTCXA0N9eRQCFNo8CM6mTdFvjoUhFV+smRowIcyB2D9Rn9THfGpYbqYpwuzhAw/8QJu/AQI4odMP/Hihzw/MYNcBgINUOhXnKGDubm5HBnk1jOFoqRwFn/9ChAQ3wDngqFsP6cSIC6Qb38DO35ysEb85MFE3H5iz80Jk0vcccaEmksuJY0oRcIfEEFMn1AYmwmpcv2cVhZ/6olfP5eQYhs4FpMs8cs0ORnikOP9jE3fGX77j8jcwlXInCt7ubDUADqGp/hRrGNRM7RATpAXSAuhGnNVURGc9myMmdVwQQqoz2wzF1yFTmMwgkIZqyA/GpNd1TDzjIjVMMGKj8YXXIn+MEHLnFB/NNpDwNlO4HqcT8as7IJdhshuylg41VBuRVGpFjSjw0apk8deBGB+kfamnmAsbqyKxYIDZ4v3wtTU+BTaOX7UzgswLyMWXNAT7m03I/EF3SatjSxo5E9tNGZULBw35qJxYy4ZN+bScWNeNG7Mi8eNuWzcmMurYcX4MEdOjMZPOmL+9EFI06ZZ1XDRuEkds7FquGTcZOmTtg9Cmz63q4bqcdOOmAJWw8XjJ3bMFKvhsnHTjTKhpL6z8kMlsGjhh2Vh0YdNYPGHTWDJh01g6YdN4KIPm8DFHzaBZR82geUfNoEV1VDzYRL4IF3IuYYybjWTOvmvhsrxktEaQTUsHx96mkVOPfTfpFz0b1Mu/rcpl/zblEv/bcqLxt3yaZTjbvnUSdG4VeYYsyJq0ks/VArVsGR89M6Z0viJHBOqD0rE865xt0fKdIqqZem/QVcNi8dHpQZsrotxNoD0BiOtRtgI1fW3Gry5N25OhZdr5QOUz0m1aPzls6mqYdn4aNZhd2gLR/vIol8fJGU4ThYtC6YpTD2iapxUhhHbYLSNPxdp6kgrdJxUG4ye8MAGIx4Nb6e+svgDUDXzOuQHyIn3OTCv3vD4B5Im2vUi+5rlcdwDSRMvqo3bRG1KoBtKq4tx9uamgUirraw+ONGif4cIR51FH5Ro3IJOAn6VmeioQ6MhYRpYjeePi7AaFjrwaJFzAS0fGRGcF/OUNb6AVg7jC2rRlXPi2f5QMLzd7FrAm5s8wVhQF2kNR2kbklcUaV48EqcepSIm488ZJb7R6N4mEagAM0ZBaTLbIyglxPfUUaKbO2LRPiSd6DcjXSOXP8v9ncHtwQVmdMHqXt4CQfEJhqiWC60YGo2qHZg8Ohnx1pjZgwJHqwjOmHraZm9KxIxgN3HkjFrXm3DEpSZIBxXE2EW8qpiUxKaNBlbsFluRClImIOFgpH3B6mg0bAQjVqYMxPaLCauy1aBlG0eMaJHJ6SB/VCTtxK0jubF4FiB70KhwgOvDYaM9GF4Va++lFnBgnTMSi5W6A8VZmno0R3mRxpMGjMV6exIpg1axA4NWYlIhjcFERzWUOiDrtnUarQlaaLJhG3ojEdqyqYbpqdCE2e1k0ZkONo5JuU0ZAVvda4a5ElPQB+IJozsVhpKLLZlasyzNghdnyjGjLYxsoyAZ4ZAldSlRjUaiI2rHRVBQsAPE4kaC5SBOQ8C5I+OwGSIhNo04HLfqIG609sbMxMCCJvIYGxAt2m3F8ppZk0nbgWvomE801h1MWALF616rtsVpOyPRGOyxyigiaCnMb8YTY4C9qpZscNxqMIbVBrE/IdfVMMkJxZEVa4KbavKo8Hg6QXdPMBbkPl7sgK9hc6PMAbkyGO/gcowC9I4ANlFdjwL0phSinrTviLxFvTgrgA8vIT9FDhjz4gRwlmkAL5znALRGI9iOMbGSi6JN5refzohUw6zR0ZpR+oVlWTk6AnLfbbYuWMUftIFog5k0zQR/NNql9tisnYlJo0Ev8RB2rH0BnbxY0NC0bq1cKJ2YClXddg6DE3JzzjFurRKnb+oj8USQmZg7JqZYnKWBTQ6bF46J2tMTNsVymtKM546FS4Ol4vK8sZDW4Ow+2q7Qzh8bLd5DTfORXqPXOEOeNK4oxsZMrL4bK1qtCp6BN2v20RDdRrbOGGhyjS+SiFG3ilWs74gmomiCxAZojjQmETe3KHjFFbTbbhmLcZolj4uuCTv9qojZzSXBTho8Q6lTKMniOTNeU287ckLp4pQh2I08zTsLRdyug2q4fCzktdGE2SZFqNagEaxiJGgN0bHOmTf+dGhpaQxkqifF5ZVGuAczdYBUAb0fkBy7Lrd9faQtegZOR1KfQYA3xtH4k4tJZ0RS7Th7TCS7O4+pIzaZISNaF0IjOyQKMWYN4uAXMxK8MUxmbmyArI4Rxt8FqeSJdtrTwuJE5Ghj7TXNGQ1RJGXEUCWYO4yQkOe5o2FKszINtWI0VDl7syXq3NGw0pOaMRpSs19V+qzRozesr60grTdGJjbC5b3h8BmR1kRb2WKshvlnR1qVQDNrWy8V7oKxsYVqVErvDHlfYUTX05nKM6bGbYVFwaYNj1ChIxC5m4zRQA40WjkYIdFOLGs2PeesODwHRA284OyYOOuLhORW2BkLLQkkYuVZEXlefqWcnJxBZkS5PWdAoCHljPXHGE0owGfOKBo+c2sRwio+VjxiJElHE+sh8TPicQU0BmNdvT1nlGRMC0WYdcQGPttLi01jYjf7W1qF5KGVvc0MhQxeqxsHfrxFnNuyBPei8VG1G9xlLbLF4yazh/QV46OJO88Ms11wxh4hSBPjqK/Eal6Q2mC2dyRGjuaj4Dsq9+IzI4st3XhLTCxB0fBEk/sRNtNZ6EKiH5x/RqKQpRnnjQuvjm4YnFHfIbIhZvRnrud2LtKZuWu3dOe8ceFJ7uafEdm0VOhZ+y9idysduWA8aEJNro7SJHrx+AmaeqJmWKy7jocmVcOu+MA062PGdtPoo7XDcZBaBnmQxuoF4yaRjVE5bgIcKT8IS4ju572M8ZPUR7ZHzdaz9iQnibVofzadkELEgjV+/ObomujZtKcTf6N9i+ODtMgmuuFBy78fiKAlHBwwYssWna1fScIWYduebQhJxZZ5LB5naeJCo7LZt3x8BPbBZ8vqWTw+SqyIhEVTPT6alKsqHzRDe3hMjFO4meZKcVL3g2RjD6cXjZsmxVhZMW6yDUYrDh/KYo5/kIKR2cMTr3HKBp2o3hQdc3JiEfQIM3DO2ZGkKjvzgNbDnX7OWXGUQTj3rJhWNZ9ZkyRG7NOcGb9XTIpJn7fgSLX4zPUUDUXreWn5LEhiYfGcsZE28qnyM9aiQImf0egmc+iMzBCCUA1nyOkqY9t67pNn0GmWQNndZOqY2GOklAhjdAt1TGwY3l8cELV58djY4vh9heVT6yosX2lS02vywuAqNPa3oyEhpggf6WXpXzISNdrdE43Q5tUCkq8FPD9xHl0cMTiMh2h9kI7CLhwXIa3eqAWilWelCMpsRuYpttRqP0QKVwRpaQ179ZoPkYi9ArDqQ6TS2Bs3Wz9caZoSpjieU9C46qMt/nVXtKze3FzXBCUUTL1ikd9IJ9PVdYbsRnWrKc9x2ANWiu1ce7OTrjqpe7GeaCQ8wMvscp7F1wSktzXaY9L1lDXpKfRFY11xQdrXYUTkDYcRl8uCMUOlhalcJFOxbk8HQwNVng1GtMcQN9bELrRIIDagrnTVSbKIYYTinh4jxhdaxVU+Srzduj6l2HZcjIqLGxCY++QxzrJAkb3bXUVbFwpgX3TIcxwtwSrnwFVYA0YMitNPkED5+nV+f8uqtU1X1W1o2bSOLlu21PpXNTXBRI5Zv6Fu/aoNdS0f2Vhfx7cqJqSA1U2XySlQ5yWMlKMnkCXPlCDceVgEitT9DXULpnSUCx0ZtD08AAUppyQgz3EUhOIcZzwgS8laqUMQFc8TGJZ+yaakqXlVc12LuoRTv/YKKBSgy+vX1jddWbcGikRYlJYQCgSA0hP4KUdGIJtUjxG7xAPFcrXZ3jpa2GRdwhK38LYZKN2Gx6CTb0qCPX3BuJDuUNVZKNQtLCfF/Ca6Omdd4iQxjyfMcNi6MIaiuc2Q6LA0FduMiGtnwfC23m6mjYiLREbM7pYyjwTfyu5jjmNR8nbQEwCtaVcPq7BG6HoR3ZTLFOdkYC4OUD0WpiwSXwtUd5ms62BZTaLgcB57KJ5fdmAsuYFn3ZKrqoKyUc7WQK59KgUKmv0hI64WjiGnWV1th/Osu3L2/U5x+z3lJt5FFlrItK8/9iTS2oTUlrxOZ9JFrKoRZPKRCav1qUHsbMbGd7LDLWZ2I/45I/DFrfyQjXKphYIJxFExqCZTN+64Tp3HTubTVfwYtRLVbc0Y9GF5YuUs5BeNRS7vsPEdZDPSFk0lW3o2MpnjB6NS4jk+KhoGxs5r2RhUPbHoNqLi5x/Esw+phEvGIKThhwhlL0gl8o5J1IYtzp2SRgZ+boQGkVTq6rNTM7sxMScYb9bd0e0WKRXb4Ic+UqkvHZOauqktQPa121T6phR61mk9MaPHVhqp9349zdir4lL1iAGbukLQ02b0Yb/GfEKonIrTT5/B7GbBBb8vknLTW+RGSvZcgUNXkUl9sNWg9GSsNxLhHs9IqhXGSolzEzc3UzBC8lQx4Zxn47QJFTs2U5ZeiI+uujmtnqAZGi1TO60L09DOlO+sNNy0F2gUAl0EHiOFiwmBZcfo7wkHI6LGzp7zymb7xrkysDqCXPL0tw3UBVP7pn0s2l0F0zkFYdSKrEi3ildJquAKik0z4MZInwxDZYmKh1PkXXO0q6ugQiQkR5qxioNYDs0+KhKaFQtSsCQ76vUffnVBPtygLILzmSAx0GPQ60LKypBcjolnsTECT9+I1tnG5stblkM5ToJHvy+sbQJ9Uz24NtFF0k18vxbdBn99AF2G+CFzE9/F5VCAUBsgAx26mLoJo7PQaaA7nYWb+AKsn+/DIn62CAeYEsO55BImpkJEfCcUU6KPvolSa2DHj0C6PYpAP4bpousmvoeLboAhhMH3VTPpEyAexIVTKNukrqhiAZCJ+gZMrmiTdRdWADA9urW6yW8lQldoKU0FZnblTdb6AEcFqOx0dXUTX13dJNgmIAYnXOW45289iDBlNKg4KOq5yrbzgtuj2CsttRmmd6/ioGHy9PoKXeB107Vd8jZAAToBUY90cRdhfoLJC74MK0wJBiBXhDkuz/b7oSSgWswvW03gcvv5KVe/QFK3eTmiHkpHgALMg2xdlY/ICBMtowCHGvziunSAU0c3A1265hzApsdY9FA5qYWZLfIHyB9gPwkMXbaei9QU0LeshqItaSdrS7eMPHpatmWUQ5laC2S0kO6FjOC2aG8CMoWJCHnipYyKRQsXLrQDi5yBxc7AEmdgqTNwkTNwsTOwzBlY7gyscAQWOzlY7ORgMXGQKwNLHf6LHP6LHf5lDv9yh9+R3eXhYHscijDQGwu2DrTE0L7BeUBWsBW1fsyAMumhOaK69ZwKlDevqSYTNO8tFF+1ZAYTRVgY+fGmSLAn3oFTUFcwFEKaUGhVOAz5+MU5NtrHNFkuwxCfSVwTHIg3CXuAUewJbIEzFMfChELrtuHcgeYU+Y4AFS0UWi+marSuEWfSpt6eHnqoBWcvWRgkLzKEjGSph8MKpUcIThzDfCzMCLWgRdVpQrEVlmMUksqHyDKDPfRwDUwQ31Wp1VEatI/8qSdxnDB5JR3bild31gcTHZi5XOlpES92UZw4oRJHOabpI5aKPuquFlYIBeX0u2Abz4NbYrxVDTnb6OQfnW2AXPa20GtdiGW0mxErBTcayCHI3yaOfW8KhntxSrYthoZPaxCnZLnb6BAyrTzGYTr762wDQe5mcgPBzDFjxbRcpCTaRvrjVI1TcHyrDcZiA2Larg7l0OJLCUXxIg4xQCetIIdAKI7NUY6lo/r2/gsUIch5bAzyJWA9v+GmQrSKHeekNkbISoZC9F4VQ8FtjlJOkNnKC0FQLL7qyS6Ul3wBEYYG54ehDVZ0gQIkaBkYslvlghJnhj5+Sk8O6CGDX0Ayu+UrbfQujOGBLPlGIkyWHk8chSbheD2tSEZYBnO+81lFTEAYSpArPS1mCNw8NFFUgoIZfC8D2e0wWrtq1aCWw0FaMyIEejIkmz91KCGZ4m1GTILkpLcHMWi5AKsO1SwvWQsxyKCrHAaD48YqWiigGodSDvNqpgUrtmBNXDYqCEFaeoyYGaUsKaTw4gnrrA6WTLDsMwYwVvib5Rtx2PUkpEUYjljeOEyz1zcj1uNTPdE4Pyp3oRUZi/KjguJJFJpVW2/arZe40y3c+ECkVb07qVLKauWT4FR+mUmIz0hDngzTlhhWu1hIEMUmXEeQmkE9fklFkd4m2fzUAmJqBFVqjkT9TT1r1IYKbRu//RP1kNJxyM15o+GPfLkvm9FM5DNP+aieERwN06QI20UsXZWLr6iZOE3hRBtwCYgVNPCJW5qkRamdegbEltgmfoKSklcQanpanZSHT0TIku8yZ2hdjDfpSAYkcECgX2FEhQiqEJ8UEXiNKLLBsMhRKaRiR0AQUsEG1rVRUen6M44TRcond+hEFM1YRS5qP0yUmHfRBbc015NpM3aWequqWHpaLFMSmYqJZRFWIBTolTNPIuM3K0lG2KOqK6dVHb2n/KS3vu1yfhhLpINASw+WiblQyk4CTBTAtGNTyI8A80pxiQjIPRRWwfkCJDZ3kDEOWeZBFoejpBjE2qA8T49kIizeIiqSISXWyLAAiLssmK8Mmt1GoxnGSRvyJUC8bVHkCLBKUAQtQpBJyakcW3qJosQZEkOhm82QLHJJxU2Wnrr+1nBvHHuIskmK7Ado/cFtqFmzebcuGsKhkny8vArFllfRMVoLqTpGa+kmilz28rIIuELBAcgItRK3eSG69yBVQjYH6N27EuWzFUApjhzBAdaj6nw/ZIrRhFKhL5nL2OoigEZS1AYWh+TQXx+5PEyWAmbGlwnqiU32ETuT0UijrXxEDTnPjCP/9nh7rmMSzqrT8aaqpRPzQpJJEv9sdb4M+ZC+NWiRcsqFCnKlwXwVqPBVZgito7yQ/dwWshdDxcQ7FFJtkvRhfWWHrOTYglsXCQ9IFYwNYZk92RhglYVokWvodoS8pA1lMrxeTvhZDxRIYBPmHTaIJt4bM3ile40Zo9SkqspEUyCIpkYGH42DfKMftQMSUd1jZD+/4DrZGMOGmmKMaT4VGvJgl7QNJ2I4xg/CpOgFV1vFQnIWkbOYnCXkLCXnInIuJmcZOcvBTaMXTCBXms6rIiGhQiCXoFIaJ9p+RNhgdPNdS8iUbxkXiK8avoracP5A9rpifZIak8TwsFq2GORbcNKQuW10JU0YoG5aCIIsci8nVdImr6dcLjLMVWEyrMjfQscmMWPlbWkz+8m4Z0BM3SdiOuw0rEAy2swIjgiT6LlBeWQFJUNt88I0gm+M0O4Dq3IeP6y2EO8PWp0vR4Qbotsgu01p4Cx+PLOeuUWPYLEYGbPf1YmiuGW08cwMP71Y61lt0dhaQsxs44twWPVCLFVerKKpjgXYVvBpANpvsYjXBlOmJrkSvC6Cs4E2x9sEcStohJoHelAN0RDeQE/ZZpGP9EJOW284zKeykMe+EIFK240I3UAzLAMN5bDdiJKDmhqdVaSBemnMwA43kQAjJ0clBObxifoKV0KBBWLCbBYeYrMcfbXC5m0M9vuNSHuiQxq/FEPM5UkPHRBhSvFaYaHyidtOMI3DqeYlqQUeU4jTkbcwmNPa1MGt0AbRiAXFdC7MaA2TAuGrlkykIOKGJbO4RqnefA5QV46iepqQcq5MnosSaaS+QTh9BIi4V/MPzkEp3CJHgKuXAHVSP3EFZ9CBtDXMPI8VyM/l3JEoHeyIXEMiWomJ1U6Xo3zwLE0UlYI0kRDzashFEIsu9Qf7CBvXm/SzmGWwCc+cOW9KcO1YVx2YHXU/D8o4kKCj0+Fasm5IIecgUDzvxm0vtGuh8jmYUl16gu0XsasHkI9CG8oC4Qizzip2HK0jgjiLzMh7XSyWVlWh5YRNwCNCEQfRcI4KOecUBUBxxigGDfUOSSGAMEUKZEiWqSQlyFlSBYiWy1c+LgxRiou3jJbFQYQTllDfVGmMxZvG3EAc7MV6beXuROyKG8lNRmy7ic1BNS9tOEqRe0OR9NRHpClHMTT755ZUx7isALPjajfbEBKObguG+fYiygabp1ntUo5y2sXtPxYp4SVtpHVAeQcvrjhPX9SK2eukjmBcTndZ45FQo5kCBQhfYxl5cA4F1UPHhIB4aR0vF1EsKUZ/I60CiV1vKKYwr1qJMRwLjJC1PMqhR0KZmY0RtUtjndzg5Db2kKUjwnkYbpE7kZBNAZqZQj755KpUHNwYopKbIRrxeLLSQvfOUVfzhHwi/xBAl2H0VAZ5d7KnN4zgmaOC5bT2Eg9MTo9nwxVVwigR4pjAxI5EoocfPjDleE2qFoocYAbotCphYjOBiyboBTgvp/fpDTE/yDGt3p5tqq6OvoQwEErRZ6AotVNHkoZbjqnu9UO+SVuYCWkqZWIo2oUSbsbFkZ0i9MhlvTpe5SuxAfJ8FUwh0IAPrYB1bRJGM0/CLsUo61W9tYZBZgimsDoYl+sg8gccEMQHK9bw/CiC5ivD1Nqa0m05Zlx5yy0v2llrqQu2B8WkLJ9j5HABxSK0NiqZiFPCUq7Xx6KtYsmzhGApU1HINeNr1Mximu0Xtm7QvkdETK3hyQMqfzNuD0yUgDVJJ3+fNVU142JiPtmMU3e2qygu6yjDjF+BnbqAP1YqmSbd7mRur+RrM01oXJu8xIyYjrcaCEPZTqo8E824MM1SW6PMFP1Psce/8ZBHwPZgu6pPsYbHIoa889E8kgz2WNxlm8h9H7GN5Ngi9RFhXZhxu79iXUnzmRISqhZLxRMHiosZ3WZvN/HqnFOoIiCv6hqXfYaJgel6AXneQL9kwrdPYYIzZPGLVdbEB+0t1pRAo8w0qbsEViRpZd4eQS/9mAiWf2NE2oBcaQwwHUKFTJCOloKYK0KqEgVnRdJjMUUxdB2DY8hjxWCdiLNyG8SevZrEmfJJA3B3Rs0IZHYZA01oS14tf9FhjLlpxcKK1mh3pXiOplI+W1Mpnq2pPPvLYeCXyafOCCsWjzPVtOeiP5KamrPtKxb9e0n6UpMUs9FxlzotsRkysVFmDhWLFlrRo1miOJ+cIqOjzsUFrDGcfo4ZtQgmyCiak+FshkQLZ6kzLCivo6d03YqFZ45elB5Np9R4nZnWZ3H2O01F90bWRTaaNJLLisBSlMrIOO8FmIk4pjdLwbBPy9PzygakFC8GjwNBSbITY3EKRprKYoyLLKYJgzWGM3oJTHdG28qCY5dbtUixcu8Bk3TyLTu9k2xpSqJkEzljV6TGoi5wxi6zs6ThXwgVtsxqBe3dRo8cbRu3LDqOA1r5IkjtREizDCtijYyln7NReSz6wHlslqnQZhmlPVINxysu+vd6UQ7awYk6sdhD3sZoCPURLVJQSNyQFEhiBcAVxuEjIywW7cI8c8Ugz13Lwim7WWLgLxS7EC1q58bNJyVzyBXreQUpZ83g/JSgOlU28vexqkfHG98P/1wwOvHIvQQ39VEUdnTRotlEP1xCK6fO1ZQ41htGW3P1EgqpbiXWMbIJJFbQpLGRwz/lJTZN2dvc0du9jcMFIhxVK6kcpJrLtupogcX5OHdOzh+VYNTitnZhE+PImEOnD+RqVhhnoqB1Q2E3F1xOhHF63h3ssS0NDAjboaQ7GOuyHlTlrZZSB6hJZAtlBJPiSEY2j5yzCOiU73SEQkIgVSdTLlFhe4gvUiCVUxZLAi1ASI/Dos2TIDFp6w72QzE6sm3FygzkIKTJMCI4U6PIlh6VFa97EySOsRQo4IC1gp8hDPgicRBNrf96YEEawNpsk0eT5bat3TCz0wisDV0HzswROHTPwI4/Ny2elIm1/yfsvxosqrooI728jpbJ3jiBlAFa3m3E2g2xbEvKGGcbyszjGLXhlKV+ZWWqOg+mNgQcrOXLuHV9EVRJk7vVMwrCRFo90CROTBTLCNp9FSor2zoZr5LgdXRsFLM9xsceUB5zutFKbKFjadjAZI0JrQhzu9Mufhj8+JlizPkTXpMkqpw9SowayBXwKp7UCP8l1LdauCNNdELoiX8BLuyW6lhuoBWJsL2jltndFTLphEZ3nPcyirqxKk2UENSibIdmdLO172aNnEFuHIojRp9lA5k03cxDiHUWxh3hbTxy66m3koeeiYPcSLRNzcIm2v6W7oEWWR8w2QGmX9WzInIitMrLx+CLLK88ol9iAaxNxoKIw9jF4nFwADKjYu2nIJq60ZNuf4EebUPktrY4mtKl4rt6oDYaMninFMVTwHjvjA5ct6ijLrnRnoR8GBAT6OEaQIyE9cwW5i1CchkqC4NcOznoEa/ZMSFp3yzxjcPcaKy9St2CquKLcVXpz2nIreL8aMxspwV7VvGlVsh6MgrmOM6/nllJz3NgqsUXgavOeDqQ5zqQ6Yfs7O4/YkStTEeVx3BDjkmoA/2cdPSRnOb0WHO/zB6x3TqxhxcP1VCgdk8z+KchIJs/1DpT2KdOV6XsSubIKGwSdw9Vp5tkEgp7xF0hFBSe4jrCPL3LoXCzmaCNGRZilJIsqfegSHrsvR0JEFtvZT3q2SfuXwKoUMSRoFJnCPPECSEUSBhv2caxkB1BLGQOf3i8yeCfe8SU6NMS6eVtqQy+yQt5/JFbiUUcEItqLEK5DBAbi9k9aI7RfgvWBM3ZZ4mfy+xNRCtpUWtkS89OQxhtNPGk4Yxs3VJ5INv6UdBLPCjFBBt5BnsktWcEZvoxoQrH71COLbVzHFhnFtjpDszRehXFjtOemj0CeWSC5zGOXMoc2469ZFS08Zmx549KO5KVST2p+kjuuWJnTIWvEwpuSho4vsr6ndEJPeqglL2vFceupqD8S7hQ0uOYmIixKIdALbzGm8Ve0tG0w66eHMXOgiH7EFQRBZ0noDJ66FdrkGeh5NJ+ra9UguVtN3H4RcFQO7H1ne8AxLGbKHXJ1ci/OJZSbdN7HHao/OXcuKUhcqyrI8g6emnWyicAKV/nj+2SbhAARzpYC8KIxMoTHnmlnNJVhuwk9UN91MZGwrI6shG+3aSsCnp6t4XN1hbxo7iYKAWxjjPE6rirh9bG0KHDq3liP0ruKMpAnH3iJGOx8qEpJ+NYL5BMuOlyBUwSF5FIKcgfoeWrOjB7VDjfOrT7zAickYI6eQROrzjgOGeMiJH9ae5YmCNzm0ioqxBTyPrGSFeEziNNJTCbmCMOM0A5xbVQ6tK2QNUuqGZwDC0+tDiUWktcEmbG+NlYyIjRcVBw8wZTNrksmTnkw3FvyWLIk162mHJloDHYY/lpIZEpeUOLfX4TR5Fi8glbRVKU2RDhEjA/5jh0gnyJ9fISVAjRGJ+FU4fg8hyLXJQ4axech6HOu8KIQrkFIbGm/VIlwRNiRjsyb8RWOU5UQ1YMLSUa/8qkJ/WothOozmJNjsmjGql7V3GYpCKUvpCTw0y+R8bVSV+S+mLhdSz0FwqIdQ67KDUcVyRyTZw6MFYyGdnNUaiI0Q6ZutKDE+Ou3lEG2ALGsqybXA42Rxvj7Za/GafUxdLfY2FmM4QmLUXK16ImARYAh2Oea8phW1z6plofubxIjWitKlLJrQDVzYTRlhwpXVQ266XqoXbjS7CUFHvEBoIMiP1sxwNgxGfKa2BUJvnjzqUx+723Fil4WXL4gok8X1U32K0CoGBi9yUjKmTGxayjJEZPnqPctNe3yQ2TvBhtJbRs5/V7Vwxtrxx05CnnArF0Wi+3dUtHrqQiF2L9HvLiwTZ7t4kC0voviWNZhKJYJ+1/N/0YJ+STS6uYJm1KTaCVWPvnYdnwQ0PYWp+FrLi47Q3Z7OGJRdygZ5mhSHyNkDqVMWn0JVxkUhwFCw40xmHyGMu4ImKU1VtBbh1uosAoO80TeenW+VNWTDthtAVfKFfrvA6VKez+qaPEKGVaNsrqMFYnAj+CupRMD8GqXCIWrMpAyikRx4qxQBq5fAzFatXYKkrZKOvIEpi6fIxNZUTED1IWsILn84vYL1FAeMQX5w/iai4RT5ksxLlI9lFWSiJhvQGEcqVukjdHxXOq4KL5LaLR6RxEM8k6yqH3AYOolyKUacJvRExadEN2E9Z70zwtYckSRx4MYiX14EJRvMNso21jdd4vN94R7ZN2Sx75W+LSiIl39La1hQ1rDu3mI6cZcT4elSVHUxQT4ZHmmNh9RaNGglV4sgyrNQ7Z+4khGaEAGfGwYaBq5I/Yf6Z2HkDJI4hzK4ded5jM0KvoGvB6ZY+JE49xtIsMq5zZfICLL2UoX4t16LHQAon1Onecfn67QOhztbddnL4qhHmnQcTPwXNmUp9PVz4aH61DR7Z+EUfzSYdOlauG9onUFjqRyqZvnozjeW4Jz5NEr5Q3LSYyaMQ95CxlfOTTslW8JdjNWiVDnN3N5g+d+ctzbIogkTwgUSQ9lo7MZYA4dJrBlz2ws/GdD/sKiDRQywTcMWPDuaEE8i8YW8BMBraIL13oUteU3DTrxDZA176ekmNtogivUM1Z8d5uMq+RRnThkTsxKD0jgSR9CO2IRSP0AjDZVZliYIfCROpRzzwR5vNZUCADUjCKZFAkTmvEAuCUlHSIkpSShDRgkCU1r0hYcyAr1p51TBIgKUE2Va6AsyjlSz+fX4apcg41qmDJOBasc9RSlloP4Ecfor0JTzCRMLp7EjXg5rlWDrniuGJmQoypRQn6uYb6NjV8Q5kEkPFnARG7t3sb9g9KOLQO5yYZCV6QyUtE6WCkWIrDAFm2IpAvAnJZLjsRlcPxROVDU9eBUK7AwnXEuBN0PmAeueOe5Y9AHjmHQGYTwbAcMws4YJ9T5aAY9vPZ3yIvIRWKkDUIiFTkMpEbWe9GgtiA9eA2lgtnjzTVopFx9YD4DQoeISdizAZhiTvveZQiWFwOIwomBVciEcbUaR08j9w6uYqbQYE4lFP2zutF8gBwDcznGFV4sUXRQ88iBGn1gScJ823silGwR2JlqYlTprgsApN6I6POG/IcW6ciIA+BQK5YEGF1n2/76TKGCFl7EL1yZpLX6ziWVuwIiBW9QgWRzeLu5TN+5CqVnUUB3o3rVfZtRi/PnnO20+6g2NLbLjcKxYvuUKrCjiN6U7ePtpkoBrV8FcfdeaIKbXA8l4sD5XbenMviz6qE9Kxrg0z2EEIrHSvL3s5vYyLLucLHApBFMx1STRnCZs7hz4LunqXIPXmtx8WRGwq3yL5bQA+y2pKbwe+UIjmOoN2ci9YHxbS1tNpI9Bl8Goo2FuTOta0B6eYjTDvDhjaU9uHkkDZpuzzyHi8qMxsmJ4z2UFI6Ii4Ek22Y42oHIrtpOwZJ0B2RhQUbJYv0OMrCgqVmcUEfnwyyHl4Sb5WIcY4fXFJzjnyBKKcnGX10kRNy+MOLA7kD+IeeRQihXrX9niuvvKS7+xKcop27c7YcAmdfMltsv9BaqIOd2Z8Efecn4Yvaf/zHmuU7Z5PCw3ZB7JDRP3v+bOryiEjGRCVd8sEIWT6M7AjGK/liJeYRn30JKsy4MX92txmpDPaYsy9ZNH92vCNYuQhpFi5fGly+YtHii4PGwotWXNx60YoVi5YuCbVdtCx48UVLL156cbBt8fIli5ZgolL8kGhF1eKqpZUhYzvymHfky0f+66XPvvTpI08dOYT/v33k0CW5oG/QfpNfvl5v0l/JqMgu/lvOTPa6p/4tZ3LE9kZtb4++EUmm50xetewT2rJ7tcmd6Cx7Ql+2Rpv8w2XLNRGdbUWbFL1cW7ZYm+xXcZfJuKvpK7zBdLp2osN/39WtRPOsyA4ZuZxzXqxR5j6BNAmREOPHEiNPW9apTd5qfW+2w5jUSszYSkTxN4FToOhVhBnVJj8nE3tUR+JlX9LTqJ/UU0pXiyQU/ZQs4FpMRcQVTr5cAewktsnEv6TLavquQM4VpUXktjEwqCKtrIzUrLJEJWNESH4dEZiWg8W1WjpJaypJjtVgXVTRDaoSbS4Ec6pu11iVkWm12FZkXUGzUqENyIyIcFsRH5eAZbsoTcLaaovgNbY3ZnvjduIsGZiubEIbJ5FS0meJAfH9uKrz9IirVTidcvQI1eZ25WVixK8lMx0pzPQKhIzJV1Ipo/RPERVhIY5pTtGz2quL6uOJNEkYN/4PbPy/fsD0//oB0i9AfAeyWyfxUPgpKY8D8wd2lW23vX2299MOYUG+Pp5Sy9eJSFdK/Vfkuv+Ws6xWt5toF8Uue5Kar8OSxtUWkQSsSQfUpQNWpgMuk4AWVV5HXlttb6sDavc6B1fp0RmYlQO2iWBauWFL4wNcGRZKi11W1fe6rR7plng2465lNziDGRJBAIN29X7d9n4jvROnZhmVytTnaJB7U5rqEdv7qEqr3krLijto1/XNKTxaNeIsxs3p3e6Q7f2OIrxMol29rFqzopdN0m3/NN3G/SjL/ketAronXyEBW22k2zWZZksKLEUYqGY+r9lMjwBHxwRHUxJW3N+upUs0gz+fCo6OCY46wVJ2qx3lXCkBjnKmM+9KrQolWVFHk6Sk4bY5SoNEnc22bKaeIl5rUzNcm4LrcTTceQ7/+brdCX+NtoFuq7Gtqt9iUiRv33ck/lRK4hfoUoVkYmCeI/X5Dn+lw3+Rw3+pnlotKR1g2WqHkFmc2ZK/7KN6SkM0puiKO7Rlq7XJTy6bJweb6vSx6Qo5NlWnDN1MRrAGq0NaQ6GIu0yzNAanrcwxmemFCqNDYvjSxedjdm7VaWld7LA9RG6fkMKZoafELHEyKJWdmVIKjNR10Y6PWsJB1Bk6kgo7ghBSjFAEXD0iLo3oUS2N6FHNIno0VdNwizRTcHb5ivK15VeU15bXl88vX1leXT6v3Fu+rLymfEr5leUN5XXlS8unl08tv7h8Sfnc8snl08pnlJeXX1g+qXx1+Zryy8tXla8rXw6gw6Q9niMvatoez3svaqBpGnzOU/7pe91P/1ibUvLqjzVA2FQM3/MTbVrJ+xjWEedzHvw3HaEHf6LNKDn+E8aaieF9R7RZJe+JsAfDx49o55Q8coTDszF820vauSXvHKFUwFWsFVdce617/0sY7YbzrrvW/eQvNC37uZc0Lff9l9E58nOMccFcjLnnVxhzEqNzj6MD50PG1Pm7r3W/+hVdw9hT+Nnteedmcl/9oq7lHkT03JO3ou/4Z9F5H2HZJ6+niM+hc98gBu/4KvpOEt47/4POW/+HGexOIuzUSQw+8keKIOeRP6Gz7110XrkHY3d9GZ137kVn9y2YyjtfR9/Tf6bYv6Bz4BTl+y90dlFub9+Ozut70Xl1HzoH7kbn219A55XbKKk7dIAc2IAFuPYJDN9xkKIfRufaQ5j0tx9D373fRt+RbxDTj6Bzz8NIkgdbkOTmwxizbxidw+ScegadHz2NzusUPIDB3H2Yavau71Lsk+i88t9InA9tSHzgCDH2Y6qaZ9HZ/yJG3/ccOZTqc+jk/ogi7v0hlfYnhPx9jHjkeV2HBZA5VcuNEQu3uqgKMfa9/6EK+R06h18ljij5k3+jgv4ZnSNvoXPbr6l0J6ka/pey/I1OdYvOk+9Swf5OtH+hFroFEz1yDFO5+aeU6P8RB68RL6+j8xw5T5+gRCmB596h2DfJ9x4l+luK+CtlTnj7CG/3S0R7jJIn3/43qDX+SVkepca+FnO7+QZ0DuxB5z4q0R3ExpH/RxlREQ7/UYcsFA4W+rR/X9Goq3zW9VWt5L3PuqhvFDH8Pwm+/zbXfVrJc7e5IEfTSkYjF/+GCPnt21z3ayX3fY4TmcrwBwj++udc+7WSa29n+HSGP0jwe293fU0reVrAZzL8Ica/3XUA8T/PcA/Dv874n3d9A/ERnoldcQQPDzPt512PaCW33cG0cxj+TS7gHa5vaSWvI9ylaRcqkkcp6to7XQeR7Ts5qlJFHaKo5+50PaaVvINRMB0ep256236s3acfcGnZtw26sKvu30/uW+xe+yC597F7mN232N31NYaze5LdHz1E7jvoZh/GZHLvvR9979yFzsm70XnrCxRxDzr7voTOri8TjJwDX0Hnva+i88p/onPvfRQ7hNxlwjHi7tuPIuDmA5ji7m+i79qH0Xmfgnd8ywVuDRaivqK/P9dQaR151AVF7syloI39t/gXhPnKd0m4nnDp2dc+5nLnvv2E61UNnv6OS4f3MUM4cBCdkwcx+Nwh9P3oKcpLq3U0zVtUmbuedv1aKznwtAsyNO2K1Lb7X0I4/rTrN9juzzC53xH7NsUeeMb1W63k+DPcTOtV1P+jqFPPuH6nldz7PY5qUlHvUNTh77l+r5W8LaI2qaj/o6ibh11/0EoeGUZxcmf/TFuJFbOSq2flXtcez6nn3VT0O15wa/qRF9yZ+n0vuu92wSs/xGp8n5z7foQInlfZ3fWiG+viRYTuOoLOc0fcUDStcHOpXqrp+L9U/Zd/dXJPUvKHf+nS9Dt+hs7Tv3K59Cf/x+XWj7yOwedOoHPqhEvXnzzqytR3/y/hvY3Bfc+j7+3nXe7dnqf/j6To9T8g9O2jCN11DJ3Xj6NzklJ88hV09r+EzpGT6Bx8AZ3jf0Tn5ncJ7ydIduAIZrr/5653NTg+7HLDe69h+938usv9ae34G9ie+5/F8LXfR+fUD9F5nZxTz7lccOC3Lte1njv+nwsLf9vvyT38e4I8cpI6HoSxGr+kk+S8y40ZcTTml3Ws+/ffdd2rl9z7HjZLVmasmMbRrxD+wVPI2uF/IlfPvYfOO3+mcrxPnJ7G4Mm/u+7TP629/U9k7eCfKdc7EAyn/oHh16/Dij/8NxbzASnm36QkD1zvJpY+ieFvUfjU9W53nqvgU+Ua/73ebnb5l/2PEur+pDsv98BnUAJuu8V9ECHZr+8iqci+bTd/nhOf/TfwZ/eN/HlbhJ68iT+n8AOv38z+e3a5XbD/VhKQWwXqZ9xkVQxTjbx1q/uwXnLqVkTJglu1qVqD1vCsvsdzx2fdGloJB/CDHfxH+Ml9i5yDt7lRB32OnNupgPrniPEfEdv77nC/qMMrn8eMTmEcGkg/0ZH28B2Y6b3uV+8gdO3zojVeorz33ek+qpe8dwdzc4wgr97pPq6XfPtONxSUlt8papPUQfrflynhIw+59eyDQ+j86BvovHIfVth7d7ozUc99wU167qvI5R33urOyX73f7cp9/xCy//Rj6Bx+3K3nvvMk+l59yp2d++p+9N37IDr/vzJzj22qDMP4155TZJXABl6mwTGw6/wDIzEou0WRqxpETdRARMQ4Fe+I8wpikQkDCg4oUKDiYDMMglgVdYTCKgwsOLDDiY2MMGSRKQUqQRlawec95xk9q/EP15z3977Pd3/P953TQsUGmMQG/aBdJTfiaCXQufIEJX3Lsao2P0z7CpiNK2E8q3RdJYPwwp/I6RTT8ilMxWcw8c/Q1lctLdahnrcOvYS2S+V6GH8YxheQZuK1Sr1AjbFj1soCLxr74CsdzyxVk1q3R4McgYy+13c+LWdqSEbVDiPLdTt05bSpj/6dMlvmLGkcRQXcmnJtZndlwyUjfimls6W0aiemYFdbpNs50m14p9FtbKcxta1Sca5UTEDI0tS2S7v3Pz/zpJtQg9FNvEH2TYNM376/87G0QPqLf61rdv9uHV8iynfh5IZ2GbvlgLlbFmnyGo7oi7WscESXr9rNUJeI2hrRfVpWMmJU/96svlQK/Hv0ZVpW3R5j4odl4tUyUHOT3ldNUUew0//P5+7OT42GU5HwO4zX3goHtllVs2y2lrjYDsOGForekRDf95uxFY+LjRr12w3rWSnWb9igaY2aVUarRsOPGHr5CWxK73cwZ5tgmmc68JUSJc4GL4x/AYz3HZiweD4M7azDjJzB73VbxuZVDltGOzrLqP0dmucPGN9J8SpRL4lSZwgDOpNHZIxzcr5nSS+H4dWI1rEMHSTOw4tJz7HlCFv+Qn+RJLRWjOGsPAhTJ8b3A0xUjC8m84+dFFt+SmytYYMBWVGbYRtPycBw53q6e1aj84olMDViGiUMyLoTUin4kxzQY5KBC3J8Y3KQq1ClQ0xgjbS4KDOeI2mQsMXQ1kp+TssxFxOqlgXAZHg/lJwtliHbZN3LZMk/y1Jq4QXXwyQ80n2FVGmX0X4Rc1QG3watdjt6iW+X0jBM5Tcw5fuMlYQBZ9t+mCq5UZ5vZYZiPFHRmmDiYrxLpWGzwzYk26HrNl2TQ3F1zpNq+V02l9LdLQ6XynQP7ec+a3NX9NE1l9epl47KrbePinysckYp1X3UpF+Vu13rVuoe9IquSdSt1JV9pTL/8HtFua+zqxtxdfoFFn+4xR+LCz8X1XjyddJHBsl95BlycI7JqeRCcit5mDxP9u5nciA5knycfJtcSm4i6/ul5tpk8Vss/nHWPWPRLs81tavIm8j7yKfJd3JTbfzUqskvyePk1f1NjiZfI9f0T/XxjcU/aPGPsm7col2gljEgpWUPMLU8soS8h5xIvkh6yEXkVvJH9nkOPEUt63qT+WQheef1qfEfsfiTWf4yWU6utNRZb/E3s7zeokWpHSLjpHKl6vSy+Ne6uH6LVmjxR7L8fvIJ8lVyHrma/JxsJI+xr3MuMy/i98xL+Xl5qXyV5DH35ERyKjmTXEKuJTfnpea7m9oB8pil7IzFv2DxnW7uXXKoO1V2r8V/2OJPtvhlFt9j8b3sz2/RNlD7wqLtpXaQ2nF3KieOfFPLyU9pRfkmJ5Cd+jTGXrapzU/leYel/SHWayPVDSavIW8hx5LPkG+ApUPsyvo3JS2enhZXpMWDC8y++jB2yDMK2mBwI+reAvZFfCtYg3iI3F+wAAyAhWAILAKbwWJ5doK3yRZHu9vBHuBQ8ArwDlAHR4BnUQ9Pc9UCjgYbwDFgErxHzgo4VvYueK/kBxxa0HX+Y9LiSYxtjF/CNQ6aj3o36tMkN9DeknMNziDny/0C3wPLQD/4HBhj+wy2r8NVBW0L87CL69sj9xncK+8OsFHqgvvAMBgFI2ATGAXb0uafSIv1QvultcjfMMkP6mSn6YdwZULz45I59qA+Xu45tInko/LMBB8jS8knwErwSdZ7inyW5b+BpeAZyQd4XnIP/g2OAC/IuxPsiUkNBHuBg0H8hFElYBY4AexN9gFd4HXgA2BNYdd1RwrNc9KpTuU8XpE8gv3RLggOAOvA5rR8vMl67dQ7y2ZQvxlCK9ijyK7kV4HG8lmSc2jvkrPJ+Rx/CNqdBQvJYWASzC7qOv+BRV3HXcz7M1bWjbKCtPrtiC+XOoyX4opBW8bxV8i8wZWyb8BVnE+A5e+Tq8FW8APJIVjFemvIBzH+COgPgbXgOJt5/54FxyB+XvIKviD3CXwRLAWnyH0Cp4LPgS+DZWCZ5B981Wbuh2ngdMTTwXLwLbASnAEGJNfFXfOyVvYz9Lcln2Bmcde89E2LC4rNfdHZfhOuQdCCzO8sFNyAeERau/vS4glp8eS0uCwtrijuur/k/HqgBdLWc5jzqYbgA+8s4fuc5UfkfVZi/IJVrSjfCGYi3sz4c3KL3B9wK1iA8hA4CNwGhqBvl3Uirgdzwd1gO/Sv2X4vmAAb5X9JUL6P+n7yW7KJPAB2gEcZHyPbyBNkHMxGf6cZJ8AG8E/GSVJuUlByYzfXZyc1MApdZ3yZ3azfHYyBvRn3IbOYN5uFC/AQn8JngzxPaq41z+86+PEcZZznv+GHbzbfZ2+gsHaY+ewfLu9IPAxtudhvqBPFA0+D/wn8k8/Iv7codZnkuNym7LnmWNMX2ZQOfy6/H6hc8zncgPevI9ccuwfeAVqmWWcS/G7QXbLvSuzGoZZ5DxpuV/8AqKeMuQ==
# __DEX_END__
