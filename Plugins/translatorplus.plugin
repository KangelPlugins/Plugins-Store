import traceback
import requests
import json
import os
import time
import random
import hashlib
import threading
from typing import Any, Optional, Callable
from concurrent.futures import ThreadPoolExecutor

from base_plugin import BasePlugin, MethodReplacement, HookResult, HookStrategy, MethodHook
from client_utils import get_messages_controller, get_last_fragment
from android_utils import run_on_ui_thread, log
from ui.bulletin import BulletinHelper
from ui.alert import AlertDialogBuilder
from ui.settings import Header, Divider, Text, Switch
from hook_utils import get_private_field

DEBUG_TRANSLATION_LOGS = True

_session = None
_executor = ThreadPoolExecutor(max_workers=50, thread_name_prefix="translator_")

CACHE_DIR = os.path.expanduser("~/.ayugram_plugin_cache")
os.makedirs(CACHE_DIR, exist_ok=True)
TRANSLATIONS_CACHE_FILE = os.path.join(CACHE_DIR, "translations_cache.json")

_offline_state = False
_last_offline_check = 0

in_progress_messages = set()

_translation_cache = {}
_cache_dirty = False
_cache_last_save = 0
_cache_save_interval = 30

_plugin_instance = None

_in_message_translated = set()
_in_message_classes = {}


def debug_log(message: str):
    if DEBUG_TRANSLATION_LOGS:
        log(message)


def safe_execute(func: Callable, error_prefix: str = "", default=None):
    try:
        return func()
    except Exception as e:
        if error_prefix:
            debug_log(f"{error_prefix}: {e}")
        return default


def load_cache():
    global _translation_cache
    if _translation_cache:
        return _translation_cache
    if os.path.exists(TRANSLATIONS_CACHE_FILE):
        result = safe_execute(
            lambda: json.load(open(TRANSLATIONS_CACHE_FILE, "r", encoding="utf-8")),
            "[CACHE] Failed to load cache"
        )
        if result:
            _translation_cache = result
            return _translation_cache
    _translation_cache = {}
    return _translation_cache


def save_cache(cache_data):
    global _cache_dirty, _cache_last_save
    _cache_dirty = True
    current_time = time.time()
    if current_time - _cache_last_save < _cache_save_interval:
        return
    _cache_last_save = current_time
    
    def _save():
        global _cache_dirty
        with open(TRANSLATIONS_CACHE_FILE, "w", encoding="utf-8") as f:
            json.dump(cache_data, f, ensure_ascii=False, indent=2)
        _cache_dirty = False
    
    safe_execute(_save, "[CACHE] Failed to save cache")


def is_online() -> bool:
    global _offline_state, _last_offline_check
    current_time = time.time()
    if current_time - _last_offline_check < 10:
        return not _offline_state
    _last_offline_check = current_time
    try:
        get_session().get("https://1.1.1.1", timeout=1)
        _offline_state = False
        return True
    except Exception:
        _offline_state = True
        return False


def get_cache_key(text: str, target_lang: str) -> str:
    return f"{hashlib.md5(text.encode('utf-8')).hexdigest()}_{target_lang}"


def get_cache_size():
    if os.path.exists(TRANSLATIONS_CACHE_FILE):
        return safe_execute(lambda: os.path.getsize(TRANSLATIONS_CACHE_FILE) / 1024, default=0)
    return 0


def get_cache_entry_count():
    return safe_execute(lambda: len(load_cache()), default=0)


def _get_telegram_target_language():
    try:
        messages_controller = get_messages_controller()
        if messages_controller:
            translate_controller = messages_controller.getTranslateController()
            fragment = get_last_fragment()
            if translate_controller and fragment and hasattr(fragment, "getDialogId"):
                dialog_lang = translate_controller.getDialogTranslateTo(fragment.getDialogId())
                if dialog_lang:
                    return dialog_lang
    except Exception as e:
        debug_log(f"[LANG] Failed to read dialog language: {e}")
    
    try:
        MessagesController_class = jclass("org.telegram.messenger.MessagesController")
        telegram_lang = MessagesController_class.getGlobalMainSettings().getString("translate_to_language", None)
        if telegram_lang:
            return telegram_lang
    except Exception:
        pass
    return "en"


def _post_ui_refresh(account=None):
    """Centralized UI refresh notification"""
    try:
        if account is None:
            messages_controller = get_messages_controller()
            if messages_controller:
                account = messages_controller.getCurrentAccount()
            else:
                return
        NotificationCenter.getInstance(account).postNotificationName(
            NotificationCenter.updateInterfaces, JInteger(4)
        )
    except Exception as e:
        debug_log(f"[UI] Failed to post refresh: {e}")


def _clear_native_translations():
    messages_controller = get_messages_controller()
    fragment = get_last_fragment()
    if not messages_controller or not fragment or not hasattr(fragment, "getDialogId"):
        return
    
    dialog_id = fragment.getDialogId()
    
    def toggle_translation():
        translate_controller = messages_controller.getTranslateController()
        if translate_controller:
            translate_controller.toggleTranslatingDialog(dialog_id, False)
            translate_controller.toggleTranslatingDialog(dialog_id, True)
            log("[NATIVE] Cleared native Telegram translations")
    
    safe_execute(toggle_translation, "[NATIVE] Error clearing native translations")
    _post_ui_refresh()


def _clear_translation_cache_with_native(view):
    global _in_message_translated
    try:
        cache_size = get_cache_size()
        entry_count = get_cache_entry_count()
        _clear_native_translations()
        if os.path.exists(TRANSLATIONS_CACHE_FILE):
            os.remove(TRANSLATIONS_CACHE_FILE)
            BulletinHelper.show_success(f"Cache cleared! Removed {entry_count} entries (~{cache_size:.1f}KB)")
        else:
            BulletinHelper.show_info("Translation cache is already empty")
        _in_message_translated.clear()
    except Exception as e:
        log(f"Failed to clear translation cache: {e}")
        BulletinHelper.show_error(f"Failed to clear cache: {str(e)}")


def is_already_translated(original_text: str, target_lang_code: str) -> bool:
    if not original_text or len(original_text) < 3:
        return True
    
    LANG_CHAR_RANGES = {
        ("zh", "ja", "ko"): [("\u4e00", "\u9fff"), ("\u3040", "\u309f"), ("\u30a0", "\u30ff"), ("\uac00", "\ud7af")],
        ("ru", "uk", "bg"): [("\u0400", "\u04ff")],
    }
    
    for lang_group, ranges in LANG_CHAR_RANGES.items():
        if target_lang_code in lang_group:
            return any(start <= c <= end for c in original_text for start, end in ranges)
    return False


def _init_in_message_classes():
    global _in_message_classes
    if _in_message_classes:
        return True
    try:
        _in_message_classes.update({
            'CU': jclass("com.exteragram.messenger.utils.ChatUtils"),
            'TA': jclass("org.telegram.ui.Components.TranslateAlert2"),
            'CA': jclass("org.telegram.ui.ChatActivity"),
            'CM': jclass("org.telegram.ui.Cells.ChatMessageCell"),
        })
        return True
    except:
        return False


def _translate_with_google(text: str, target_lang: str) -> Optional[str]:
    try:
        params = {"client": "gtx", "sl": "auto", "tl": target_lang, "dt": "t", "q": text}
        headers = {"User-Agent": get_random_user_agent()}
        response = get_session().get(
            "https://clients5.google.com/translate_a/t",
            params=params, headers=headers, timeout=3
        )
        response.raise_for_status()
        result = response.json()
        if result and isinstance(result, list) and result[0] and isinstance(result[0], list):
            return result[0][0]
    except Exception as e:
        debug_log(f"[GOOGLE] {str(e)[:40]}")
    return None


def _translate_with_google_async(text: str, target_lang: str, callback: Callable):
    def task():
        translated = _translate_with_google(text, target_lang)
        if translated:
            callback(translated)
    threading.Thread(target=task, daemon=True).start()


def _get_valid_text(message_object) -> Optional[str]:
    """Extract valid text from message object - centralized validation"""
    original_text = message_object.messageText
    if not original_text or not isinstance(original_text, str) or not original_text.strip():
        mo = getattr(message_object, "messageOwner", None)
        if mo and hasattr(mo, "message") and mo.message:
            original_text = mo.message
    
    if original_text and isinstance(original_text, str) and original_text.strip():
        return original_text
    return None


def _create_text_with_entities(text: str):
    """Create TLRPC.TL_textWithEntities - shared helper"""
    te = TLRPC.TL_textWithEntities()
    te.text = text
    te.entities = ArrayList()
    return te


def apply_in_message_translation(msg, activity, txt, lang):
    try:
        mo = msg.messageOwner
        if not mo:
            return

        is_transcription = (
            (hasattr(mo, "voiceTranscription") and mo.voiceTranscription) or
            (hasattr(mo, "transcription") and mo.transcription)
        )

        if not is_transcription:
            mo.translatedText = _create_text_with_entities(txt)
            mo.translatedToLanguage = lang
            _in_message_translated.add((msg.getDialogId(), msg.getId()))

        def ui():
            try:
                if is_transcription:
                    for attr in ("voiceTranscription", "transcription"):
                        if hasattr(mo, attr) and getattr(mo, attr):
                            setattr(mo, attr, txt)
                            break
                else:
                    msg.translated = True
                    msg.applyNewText(txt)
                
                msg.generateCaption()
                msg.resetLayout()
                
                adapter = get_private_field(activity, "chatAdapter")
                if adapter:
                    adapter.notifyDataSetChanged()
            except:
                pass

        run_on_ui_thread(ui)
    except:
        pass


class BlockRevertHook(MethodHook):
    def before_hooked_method(self, p):
        try:
            m = p.thisObject
            if (m and (m.getDialogId(), m.getId()) in _in_message_translated 
                and m.translated and m.messageOwner and m.messageOwner.translatedText):
                p.setResult(False)
        except:
            pass


class InMessageAlertHook(MethodHook):
    def __init__(self):
        self.pending = False

    def before_hooked_method(self, p):
        try:
            if _plugin_instance and not _plugin_instance.get_setting("enable_in_message_translation", False):
                return
            if len(p.args) < 7:
                return
            
            f = p.args[1]
            if not isinstance(f, _in_message_classes['CA']):
                return
            
            obj = get_private_field(f, "selectedObject")
            if not obj:
                return
            
            txt = next((a for a in p.args if isinstance(a, str) and len(a) > 5), None)
            if not txt:
                txt = _in_message_classes['CU'].getInstance().getMessageText(obj, None)
            if not txt:
                return
            
            lang = _in_message_classes['TA'].getToLanguage()
            _translate_with_google_async(str(txt), lang, lambda r: apply_in_message_translation(obj, f, r, lang))
            self.pending = True
        except:
            pass

    def after_hooked_method(self, p):
        if self.pending:
            safe_execute(lambda: p.getResult().dismiss() if p.getResult() else None)
            self.pending = False


class ResultHook(MethodHook):
    def __init__(self, result_value):
        self.result_value = result_value
    
    def before_hooked_method(self, param):
        param.setResult(self.result_value)


class LocalPremiumHook:
    def __init__(self):
        self._premium_unhooks = []

    def premium_hooking(self):
        clazz = jclass("org.telegram.messenger.UserConfig")
        clazz2 = jclass("org.telegram.messenger.MessagesController")
        
        self._premium_unhooks = [
            self.hook_method(clazz.getClass().getDeclaredMethod("isPremium"), ResultHook(True)),
            self.hook_method(clazz.getClass().getDeclaredMethod("hasPremiumOnAccounts"), ResultHook(True)),
            self.hook_method(clazz2.getClass().getDeclaredMethod("premiumFeaturesBlocked"), ResultHook(False)),
        ]

    def premium_unhooking(self):
        for unhook in self._premium_unhooks:
            safe_execute(lambda u=unhook: self.unhook_method(u))
        self._premium_unhooks = []


class AdvancedTranslatorPlugin(BasePlugin, LocalPremiumHook):
    _global_generation = 0

    def on_plugin_load(self):
        global _plugin_instance
        _plugin_instance = self
        self._in_message_hooks = []
        
        if not JAVA_CLASSES_FOUND:
            BulletinHelper.show_error("Translator: Failed to load (core classes not found).")
            return
        
        self._toggle_premium_hooks(self.get_setting("enable_entire_chat_translation", False))
        
        try:
            translate_controller_class = TranslateController.getClass()
            log("Translator: Attempting to hook 'pushToTranslate'...")
            
            target_method = safe_execute(
                lambda: translate_controller_class.getDeclaredMethod("pushToTranslate", MessageObject, String, Callback4),
                "Translator [ERROR]: Failed to find 'pushToTranslate'"
            )
            
            if target_method:
                target_method.setAccessible(True)
                AdvancedTranslatorPlugin._global_generation += 1
                self.translate_hook_instance = TranslateHook(self, AdvancedTranslatorPlugin._global_generation)
                self.hook_method(target_method, self.translate_hook_instance)
                log("Translator: Successfully found method 'pushToTranslate'.")
            else:
                BulletinHelper.show_error("Translator: Failed to find the target translation method to hook.")
        except Exception:
            log(f"Translator [FATAL]: An exception occurred during hooking: {traceback.format_exc()}")
            BulletinHelper.show_error("Translator: An error occurred while hooking.")
        
        self.add_on_send_message_hook()
        self._setup_in_message_hooks()

    def _toggle_premium_hooks(self, enable: bool):
        if enable:
            safe_execute(self.premium_hooking, "Premium hooking failed")
        else:
            safe_execute(self.premium_unhooking, "Premium unhooking failed")

    def _setup_in_message_hooks(self):
        if self.get_setting("enable_in_message_translation", False):
            if not _init_in_message_classes():
                log("[IN-MESSAGE] Failed to initialize classes")
                return
            try:
                if not self._in_message_hooks:
                    self._in_message_hooks = [
                        self.hook_all_methods(_in_message_classes['TA'], "showAlert", InMessageAlertHook()),
                        self.hook_all_methods(MessageObject, "updateTranslation", BlockRevertHook()),
                    ]
                log("[IN-MESSAGE] Hooks enabled")
            except Exception as e:
                log(f"[IN-MESSAGE] Failed to setup hooks: {e}")
        else:
            self._teardown_in_message_hooks()

    def _teardown_in_message_hooks(self):
        for hook_list in self._in_message_hooks:
            for unhook in hook_list:
                safe_execute(lambda u=unhook: self.unhook_method(u))
        self._in_message_hooks = []
        _in_message_translated.clear()
        log("[IN-MESSAGE] Hooks disabled")

    def on_send_message_hook(self, account, params):
        global DEBUG_TRANSLATION_LOGS
        if not hasattr(params, "message") or not isinstance(params.message, str):
            return None
        if params.message.strip().lower() == "!debug":
            DEBUG_TRANSLATION_LOGS = not DEBUG_TRANSLATION_LOGS
            BulletinHelper.show_info(f"Translation debug logs {'ENABLED' if DEBUG_TRANSLATION_LOGS else 'DISABLED'}")
            return HookResult(strategy=HookStrategy.CANCEL)
        return None

    def on_plugin_unload(self):
        self._teardown_in_message_hooks()
        log("Unloaded")

    def _on_premium_toggle(self, value):
        safe_execute(
            lambda: get_messages_controller().getTranslateController().setChatTranslateEnabled(value),
            "[TranslateToggle] Failed to sync with Telegram native toggle"
        )
        self._toggle_premium_hooks(value)
        self.reload_settings()

    def _on_in_message_toggle(self, value):
        self._setup_in_message_hooks()
        self.reload_settings()

    def create_settings(self):
        current_lang_code = _get_telegram_target_language()
        current_lang_name = next((name for name, code in LANG_CODE_MAP.items() if code == current_lang_code), "English")
        cache_size = get_cache_size()
        entry_count = get_cache_entry_count()
        
        return [
            Divider(),
            Header(text="Translation Settings"),
            Switch(
                key="enable_entire_chat_translation",
                text="Translate Entire Chat",
                default=False,
                icon="menu_feature_premium",
                on_change=self._on_premium_toggle,
                subtext="Automatically translate all messages in your chat.",
            ),
            Switch(
                key="enable_in_message_translation",
                text="In-Message Translation",
                default=False,
                icon="msg_translate",
                on_change=self._on_in_message_toggle,
                subtext="Translate individual messages inline when using the translate option.",
            ),
            Divider(),
            Header(text="Translation Provider"),
            Text(text="Provider: Google Translate", icon="msg_translate"),
            Divider(),
            Header(text="Active Language"),
            Text(text=f"Current: {current_lang_name}", icon="msg_language"),
            Divider(),
            Header(text="Cache Management"),
            Text(text=f"Cache: {entry_count} entries (~{cache_size:.1f}KB)", icon="msg_storage"),
            Text(text="Clear Translation Cache", icon="msg_delete", red=True, on_click=_clear_translation_cache_with_native),
            Divider(),
        ]


class TranslateHook(MethodReplacement):
    def __init__(self, plugin_instance: AdvancedTranslatorPlugin, generation: int):
        super().__init__()
        self.plugin = plugin_instance
        self._generation = generation

    def cleanup(self):
        log(f"[PLUGIN] TranslateHook.cleanup: (gen {self._generation}).")

    def _show_error_dialog(self, message: str):
        fragment = get_last_fragment()
        if not fragment or not fragment.getParentActivity():
            return
        
        builder = AlertDialogBuilder(fragment.getParentActivity())
        builder.set_title("Translator Error")
        builder.set_message(message)
        
        def on_copy_error_click(b, w):
            jclass("org.telegram.messenger.AndroidUtilities").addToClipboard(message)
            BulletinHelper.show_info("copied_to_clipboard")
            b.dismiss()
        
        builder.set_negative_button("close_button", lambda b, w: b.dismiss())
        builder.set_positive_button("copy_button", on_copy_error_click)
        builder.show()

    def replace_hooked_method(self, param: Any) -> Any:
        try:
            message_object = param.args[0]
            if not message_object or not isinstance(message_object, MessageObject):
                return None
            
            mo = getattr(message_object, "messageOwner", None)
            if not mo:
                return None
            
            target_lang = _get_telegram_target_language()
            if (hasattr(mo, "translatedText") and mo.translatedText and
                hasattr(mo, "translatedToLanguage") and mo.translatedToLanguage == target_lang):
                debug_log(f"[SKIP] Already translated to {target_lang}")
                return None
            
            original_text = _get_valid_text(message_object)
            if not original_text or message_object.isOut():
                return None
            
            message_key = f"{message_object.getId()}"
            if message_key in in_progress_messages:
                return None
            
            if is_already_translated(original_text, target_lang):
                debug_log(f"[SKIP] Already {target_lang}")
                return None
            
            in_progress_messages.add(message_key)
            _executor.submit(self._process_translation, original_text, message_key, target_lang, message_object)
        except Exception as e:
            self._handle_error(e, "replace_hooked_method")
        return None

    def _handle_error(self, e: Exception, context: str):
        if isinstance(e, (requests.exceptions.ConnectionError, requests.exceptions.Timeout, requests.exceptions.ReadTimeout)):
            debug_log(f"[OFFLINE] Translation unavailable ({context})")
            return
        error_message = f"Translator Error ({context}):\n\n{type(e).__name__}: {e}\n\nTraceback:\n{traceback.format_exc()}"
        log(error_message)
        if DEBUG_TRANSLATION_LOGS:
            run_on_ui_thread(lambda: self._show_error_dialog(error_message))

    def _process_translation(self, original_text: str, message_key: str, target_lang_code: str, message_object):
        try:
            cache_key = get_cache_key(original_text, target_lang_code)
            cache_data = load_cache()
            
            if cache_key in cache_data:
                debug_log("[CACHE HIT]")
                self._store_translation_in_native_storage(message_object, cache_data[cache_key], target_lang_code)
                return
            
            if not is_online():
                debug_log("[OFFLINE] Skipping translation...")
                return
            
            start_time = time.time()
            translated_text = _translate_with_google(original_text, target_lang_code)
            elapsed = time.time() - start_time
            
            if not translated_text:
                debug_log(f"[FAIL] GOOGLE in {elapsed:.2f}s")
                return
            
            cache_data[cache_key] = translated_text
            save_cache(cache_data)
            debug_log(f"[GOOGLE] {elapsed:.2f}s")
            self._store_translation_in_native_storage(message_object, translated_text, target_lang_code)
        except Exception as e:
            self._handle_error(e, "_process_translation")
        finally:
            in_progress_messages.discard(message_key)

    def _store_translation_in_native_storage(self, message_object, translated_text: str, target_lang_code: str):
        def ui_update_task():
            try:
                mo = getattr(message_object, "messageOwner", None)
                if not mo:
                    return
                
                mo.translatedText = _create_text_with_entities(translated_text)
                mo.translatedToLanguage = target_lang_code
                
                account = getattr(message_object, "currentAccount", 0)
                nc = NotificationCenter.getInstance(account)
                nc.postNotificationName(NotificationCenter.messageTranslated, message_object)
                _post_ui_refresh(account)
            except Exception as e:
                log(f"[UI] Failed to store translated text: {e}")
        
        run_on_ui_thread(ui_update_task)


LANG_CODE_MAP = {
    "Bulgarian": "bg", "Czech": "cs", "Danish": "da", "German": "de", "Greek": "el",
    "English": "en", "Spanish": "es", "Estonian": "et", "Finnish": "fi", "French": "fr",
    "Hungarian": "hu", "Indonesian": "id", "Italian": "it", "Japanese": "ja", "Korean": "ko",
    "Lithuanian": "lt", "Latvian": "lv", "Norwegian (Bokmål)": "nb", "Dutch": "nl", "Polish": "pl",
    "Portuguese": "pt", "Romanian": "ro", "Russian": "ru", "Slovak": "sk", "Slovenian": "sl",
    "Swedish": "sv", "Turkish": "tr", "Ukrainian": "uk", "Chinese (simplified)": "zh",
}

USER_AGENTS = [
    "Mozilla/5.0 (Linux; Android 11; Pixel 5) AppleWebKit/537.36",
    "Mozilla/5.0 (Linux; Android 10; SM-G973F) AppleWebKit/537.36",
    "Mozilla/5.0 (Linux; Android 9; Mi 9T Pro) AppleWebKit/537.36",
]


def get_session():
    global _session
    if _session is None:
        _session = requests.Session()
        adapter = requests.adapters.HTTPAdapter(pool_connections=100, pool_maxsize=100, max_retries=0)
        _session.mount("http://", adapter)
        _session.mount("https://", adapter)
    return _session


def get_random_user_agent():
    return random.choice(USER_AGENTS)


JAVA_CLASSES_FOUND = False

try:
    from java import jclass
    from java.lang import Integer as JInteger
    from java.util import ArrayList

    MessagesController = jclass("org.telegram.messenger.MessagesController")
    TranslateController = jclass("org.telegram.messenger.TranslateController")
    MessageObject = jclass("org.telegram.messenger.MessageObject")
    String = jclass("java.lang.String")
    Utilities = jclass("org.telegram.messenger.Utilities")
    Callback4 = Utilities.Callback4
    TLRPC = jclass("org.telegram.tgnet.TLRPC")
    NotificationCenter = jclass("org.telegram.messenger.NotificationCenter")
    JAVA_CLASSES_FOUND = True
except Exception as e:
    log(f"Translator [FATAL]: Failed to import core Java classes: {e}")
    JAVA_CLASSES_FOUND = False

__name__ = "TranslatorPlus"
__description__ = "Ultra-fast multi-provider translation for Telegram with in-message support"
__icon__ = "luvztroyicons/1"
__id__ = "translatorplus"
__version__ = "1.4.2"
__author__ = "@xwvux"
__min_version__ = "11.12.0"