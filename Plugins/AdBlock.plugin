from typing import Any, List, Optional, Set

from android_utils import run_on_ui_thread
from base_plugin import AppEvent, BasePlugin, HookResult, HookStrategy, MethodHook
from hook_utils import find_class

from java.lang import Boolean
from org.telegram.messenger import MessagesController, UserConfig, Utilities

__id__ = "AdBlock"
__name__ = "AdBlock"
__description__ = "Hides ads in Telegram."
__author__ = "@kvucoPlugins"
__version__ = "1.1"
__icon__ = "SpottyAnimated/47"
__app_version__ = ">=12.5.1"
__sdk_version__ = ">=1.4.3.3"


BLOCKED_REQUESTS: Set[str] = {
    "TL_help_getPromoData",
}

ONE_YEAR_SECONDS = 365 * 24 * 60 * 60


class ReturnValueHook(MethodHook):
    def __init__(self, plugin: "AbsoluteAdBlockPlugin", value: Any):
        self.plugin = plugin
        self.value = value

    def before_hooked_method(self, param: Any):
        try:
            param.setResult(self.value)
        except Exception as e:
            self.plugin.log(f"ReturnValueHook error: {e}")


class PromoNoOpHook(MethodHook):
    def __init__(self, plugin: "AbsoluteAdBlockPlugin"):
        self.plugin = plugin

    def before_hooked_method(self, param: Any):
        try:
            param.setResult(None)
        except Exception as e:
            self.plugin.log(f"Promo hook suppress error: {e}")
            return

        try:
            controller = param.thisObject
            self.plugin._run_on_stage_queue(
                lambda: self.plugin._cleanup_messages_controller(controller)
            )
        except Exception as e:
            self.plugin.log(f"Promo cleanup schedule error: {e}")


class VideoAdsLoadHook(MethodHook):
    def __init__(self, plugin: "AbsoluteAdBlockPlugin"):
        self.plugin = plugin

    def before_hooked_method(self, param: Any):
        video_ads = param.thisObject
        try:
            self.plugin._set_private_field(video_ads, "loading", False)
            self.plugin._set_private_field(video_ads, "loaded", True)
            ads = self.plugin._get_private_field(video_ads, "ads")
            if ads is not None:
                try:
                    ads.clear()
                except Exception:
                    pass
        except Exception as e:
            self.plugin.log(f"VideoAds load cleanup error: {e}")

        try:
            param.setResult(None)
        except Exception as e:
            self.plugin.log(f"VideoAds load suppress error: {e}")


class AbsoluteAdBlockPlugin(BasePlugin):
    def __init__(self):
        super().__init__()
        self._unhooks: List[Any] = []
        self.installed_hooks = 0
        self._field_cache = {}

    def on_plugin_load(self):
        self.on_plugin_unload()
        self.installed_hooks = 0
        for name in BLOCKED_REQUESTS:
            self.add_hook(name, priority=10000)

        self._install_method_hooks()
        self._cleanup_all_accounts_soon()
        self.log(f"{__name__} loaded: {self.installed_hooks} hooks installed")

    def on_plugin_unload(self):
        for unhook in list(self._unhooks):
            try:
                self.unhook_method(unhook)
            except Exception:
                pass
        self._unhooks.clear()
        self.installed_hooks = 0

    def on_app_event(self, event_type: AppEvent):
        if event_type in (AppEvent.START, AppEvent.RESUME):
            self._cleanup_all_accounts_soon()

    def pre_request_hook(self, request_name: str, account: int, request: Any) -> HookResult:
        if request_name in BLOCKED_REQUESTS:
            return HookResult(strategy=HookStrategy.CANCEL)
        return HookResult(strategy=HookStrategy.DEFAULT)

    def _install_method_hooks(self):
        self._hook_method(
            "org.telegram.messenger.MessagesController",
            "checkPromoInfo",
            1,
            PromoNoOpHook(self),
        )
        self._hook_method(
            "org.telegram.messenger.MessagesController",
            "checkPromoInfoInternal",
            1,
            PromoNoOpHook(self),
        )
        self._hook_method(
            "org.telegram.messenger.MessagesController",
            "isPromoDialog",
            2,
            ReturnValueHook(self, Boolean(False)),
        )

        video_noop = ReturnValueHook(self, None)
        self._hook_method("org.telegram.messenger.video.VideoAds", "load", 0, VideoAdsLoadHook(self))
        self._hook_method("org.telegram.messenger.video.VideoAds", "schedule", 0, video_noop)
        self._hook_method("org.telegram.messenger.video.VideoAds", "show", 0, video_noop)
        self._hook_method(
            "org.telegram.messenger.video.VideoAds",
            "isPopupShown",
            0,
            ReturnValueHook(self, Boolean(False)),
        )

    def _hook_method(self, class_name: str, method_name: str, param_count: int, hook: Any):
        try:
            method = self._find_method(class_name, method_name, param_count)
            if method is None:
                self.log(f"Method not found: {class_name}.{method_name}/{param_count}")
                return
            unhook = self.hook_method(method, hook, priority=10000)
            if unhook is not None:
                self._unhooks.append(unhook)
                self.installed_hooks += 1
        except Exception as e:
            self.log(f"Hook install failed for {class_name}.{method_name}: {e}")

    def _find_method(self, class_name: str, method_name: str, param_count: int) -> Optional[Any]:
        clazz = find_class(class_name)
        if clazz is None:
            self.log(f"Class not found: {class_name}")
            return None

        for candidate in self._class_candidates(clazz):
            current = candidate
            while current is not None:
                try:
                    methods = current.getDeclaredMethods()
                except Exception:
                    methods = []

                for method in methods:
                    try:
                        if str(method.getName()) != method_name:
                            continue
                        types = method.getParameterTypes()
                        if self._java_array_len(types) != param_count:
                            continue
                        method.setAccessible(True)
                        return method
                    except Exception:
                        continue

                try:
                    current = current.getSuperclass()
                except Exception:
                    current = None

        return None

    def _class_candidates(self, clazz: Any) -> List[Any]:
        candidates: List[Any] = []
        seen = set()

        for candidate in (clazz, self._safe_get_class(clazz)):
            if candidate is None:
                continue
            try:
                key = str(candidate.getName())
            except Exception:
                key = str(candidate)
            if key in seen:
                continue
            seen.add(key)
            candidates.append(candidate)

        return candidates

    def _safe_get_class(self, obj: Any) -> Optional[Any]:
        try:
            return obj.getClass()
        except Exception:
            return None

    def _java_array_len(self, value: Any) -> int:
        try:
            return len(value)
        except Exception:
            pass
        try:
            return int(value.length)
        except Exception:
            return -1

    def _cleanup_all_accounts_soon(self):
        for delay in (0, 500, 2000):
            try:
                run_on_ui_thread(
                    lambda: self._run_on_stage_queue(self._cleanup_all_accounts),
                    delay,
                )
            except Exception:
                pass

    def _cleanup_all_accounts(self):
        try:
            max_accounts = int(UserConfig.MAX_ACCOUNT_COUNT)
        except Exception:
            max_accounts = 4

        for account in range(max_accounts):
            try:
                controller = MessagesController.getInstance(account)
                if controller is not None:
                    self._cleanup_messages_controller(controller)
            except Exception:
                continue

    def _cleanup_messages_controller(self, controller: Any):
        if controller is None:
            return

        should_remove_dialog = False

        try:
            promo_dialog = self._get_private_field(controller, "promoDialog")
            promo_dialog_id = self._to_int(self._get_private_field(controller, "promoDialogId"), 0)
            should_remove_dialog = promo_dialog is not None or bool(promo_dialog_id)
        except Exception:
            pass

        try:
            request_id = self._to_int(
                self._get_private_field(controller, "checkingPromoInfoRequestId"),
                0,
            )
            if request_id:
                try:
                    controller.getConnectionsManager().cancelRequest(request_id, True)
                except Exception:
                    pass
            self._set_private_field(controller, "checkingPromoInfoRequestId", 0)
        except Exception:
            pass

        try:
            current_time = int(controller.getConnectionsManager().getCurrentTime())
            next_check_time = current_time + ONE_YEAR_SECONDS
            self._set_private_field(controller, "checkingPromoInfo", False)
            self._set_private_field(controller, "nextPromoInfoCheckTime", next_check_time)
            self._set_private_field(controller, "promoDialogId", 0)
            self._set_private_field(controller, "promoDialogType", 0)
            self._set_private_field(controller, "proxyDialogAddress", None)
            self._clear_promo_preferences(controller, next_check_time)
        except Exception:
            pass

        if should_remove_dialog:
            try:
                run_on_ui_thread(lambda: self._cleanup_promo_dialog_on_ui(controller))
            except Exception:
                pass

    def _cleanup_promo_dialog_on_ui(self, controller: Any):
        if controller is None:
            return

        try:
            self._invoke_no_arg(controller, "removePromoDialog")
        except Exception:
            pass

        try:
            self._set_private_field(controller, "promoDialog", None)
            self._set_private_field(controller, "promoDialogId", 0)
            self._set_private_field(controller, "promoDialogType", 0)
            self._set_private_field(controller, "checkingPromoInfo", False)
            self._set_private_field(controller, "checkingPromoInfoRequestId", 0)
            self._set_private_field(controller, "proxyDialogAddress", None)
        except Exception:
            pass

    def _clear_promo_preferences(self, controller: Any, next_check_time: int):
        seen = set()
        try:
            self._clear_promo_preferences_in(controller.getMainSettings(), next_check_time, seen)
        except Exception:
            pass
        try:
            self._clear_promo_preferences_in(
                MessagesController.getGlobalMainSettings(),
                next_check_time,
                seen,
            )
        except Exception:
            pass

    def _clear_promo_preferences_in(self, preferences: Any, next_check_time: int, seen: Set[str]):
        if preferences is None:
            return

        key = self._object_identity_key(preferences)
        if key in seen:
            return
        seen.add(key)

        editor = preferences.edit()
        editor.putLong("proxy_dialog", 0)
        editor.putInt("promo_dialog_type", 0)
        editor.putInt("nextPromoInfoCheckTime", int(next_check_time))
        editor.remove("proxyDialogAddress")
        try:
            editor.commit()
        except Exception:
            editor.apply()

    def _object_identity_key(self, obj: Any) -> str:
        try:
            return f"{obj.getClass().getName()}:{obj.hashCode()}"
        except Exception:
            return str(id(obj))

    def _run_on_stage_queue(self, fn: Any):
        try:
            Utilities.stageQueue.postRunnable(lambda: fn())
        except Exception:
            try:
                fn()
            except Exception:
                pass

    def _get_private_field(self, obj: Any, name: str, default: Any = None) -> Any:
        field = self._find_field(obj, name)
        if field is None:
            return default
        try:
            return field.get(obj)
        except Exception:
            return default

    def _set_private_field(self, obj: Any, name: str, value: Any) -> bool:
        field = self._find_field(obj, name)
        if field is None:
            return False

        try:
            type_name = str(field.getType().getName())
            if type_name == "int":
                field.setInt(obj, int(value))
            elif type_name == "long":
                field.setLong(obj, int(value))
            elif type_name == "boolean":
                field.setBoolean(obj, bool(value))
            else:
                field.set(obj, value)
            return True
        except Exception:
            return False

    def _find_field(self, obj: Any, name: str) -> Optional[Any]:
        if obj is None:
            return None
        try:
            current = obj.getClass()
        except Exception:
            return None

        while current is not None:
            try:
                key = (str(current.getName()), name)
                cached = self._field_cache.get(key, None)
                if cached is not None:
                    return cached
                field = current.getDeclaredField(name)
                field.setAccessible(True)
                self._field_cache[key] = field
                return field
            except Exception:
                try:
                    current = current.getSuperclass()
                except Exception:
                    current = None

        return None

    def _to_int(self, value: Any, default: Any = 0) -> Any:
        if value is None:
            return default
        try:
            return int(value)
        except Exception:
            pass
        try:
            return int(value.intValue())
        except Exception:
            return default

    def _invoke_no_arg(self, obj: Any, method_name: str):
        method = self._find_method_on_object(obj, method_name, 0)
        if method is not None:
            return method.invoke(obj)
        return None

    def _find_method_on_object(self, obj: Any, method_name: str, param_count: int) -> Optional[Any]:
        if obj is None:
            return None
        try:
            current = obj.getClass()
        except Exception:
            return None

        while current is not None:
            try:
                methods = current.getDeclaredMethods()
            except Exception:
                methods = []

            for method in methods:
                try:
                    if str(method.getName()) != method_name:
                        continue
                    if self._java_array_len(method.getParameterTypes()) != param_count:
                        continue
                    method.setAccessible(True)
                    return method
                except Exception:
                    continue

            try:
                current = current.getSuperclass()
            except Exception:
                current = None

        return None
