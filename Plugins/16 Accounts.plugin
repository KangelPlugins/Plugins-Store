from base_plugin import BasePlugin, MethodHook
from hook_utils import find_class
from java.lang import Boolean, Thread

__id__ = "accounts_limit_16"
__name__ = "16 Accounts"
__description__ = "Raises the account limit to 16."
__author__ = "@kvucoplugins"
__version__ = "1.0"
__icon__ = "PremiumGifts/23"
__app_version__ = ">=12.5.1"
__sdk_version__ = ">=1.4.0"


USER_CONFIG_CLASS = "org.telegram.messenger.UserConfig"
PREMIUM_CHECK_METHOD = "hasPremiumOnAccounts"

ACCOUNT_LIMIT_CALLERS = (
    "org.telegram.ui.MainTabsActivity",
    "org.telegram.ui.UserInfoActivity",
    "org.telegram.ui.LogoutActivity",
    "com.exteragram.messenger.drawer.DrawerAccountPickerView",
)


class AccountLimitHook(MethodHook):
    def __init__(self, plugin):
        self.plugin = plugin

    def before_hooked_method(self, param):
        try:
            if self.plugin.is_account_limit_check():
                param.setResult(Boolean(True))
        except Exception as exc:
            self.plugin.log_once(
                "account_limit_hook",
                f"Account limit hook failed: {exc}",
            )


class AccountLimit16Plugin(BasePlugin):
    def __init__(self):
        super().__init__()
        self.unhook_objects = []
        self.logged_messages = set()

    def on_plugin_load(self):
        self.on_plugin_unload()

        try:
            user_config = find_class(USER_CONFIG_CLASS)
            if user_config is None:
                self.log_once(
                    "missing_user_config",
                    f"Class not found: {USER_CONFIG_CLASS}",
                )
                return

            unhooks = self.hook_all_methods(
                user_config,
                PREMIUM_CHECK_METHOD,
                AccountLimitHook(self),
                priority=10000,
            )
            self.keep_unhooks(unhooks)

            if not self.unhook_objects:
                self.log_once(
                    "missing_premium_check",
                    f"Method not hooked: {USER_CONFIG_CLASS}.{PREMIUM_CHECK_METHOD}",
                )
        except Exception as exc:
            self.log_once("hook_setup", f"Hook setup failed: {exc}")

    def on_plugin_unload(self):
        for unhook in list(reversed(getattr(self, "unhook_objects", []))):
            try:
                self.unhook_method(unhook)
            except Exception:
                pass
        self.unhook_objects.clear()

    def is_account_limit_check(self):
        try:
            stack = Thread.currentThread().getStackTrace()
            for element in stack:
                class_name = str(element.getClassName())
                for caller in ACCOUNT_LIMIT_CALLERS:
                    if class_name == caller or class_name.startswith(caller + "$"):
                        return True
        except Exception as exc:
            self.log_once("stack_check", f"Stack check failed: {exc}")
        return False

    def keep_unhooks(self, unhooks):
        if not unhooks:
            return
        try:
            items = list(unhooks)
        except TypeError:
            items = [unhooks]
        self.unhook_objects.extend(item for item in items if item is not None)

    def log_once(self, key, message):
        if key in self.logged_messages:
            return
        self.logged_messages.add(key)
        self.log(message)
