"""
    Считаешь что плагин хороший, или нашел баг?
    Пиши @shikaatux

    Do you think the plugin is good, or did you find a bug?
    Write to @shikaatux
"""

__id__ = "always-tabs-forums"
__name__ = "Always tabs forums"
__description__ = "Enable tab view in all forums"
__author__ = "@shikaatux | @shikaatuxplugins"
__version__ = "1.0.3"
__icon__ = "shextera_by_fStikBot/0"
__min_version__ = "11.12.0"

from base_plugin import MethodReplacement, BasePlugin, MethodHook
from hook_utils import *



class areTabsEnabledHook(MethodReplacement):
    def __init__(self, plugin):
        self.plugin = plugin

    def replace_hooked_method(self, param):
        self.plugin.log(param.args[0])
        if param.args[0] is not None:
            isForum = None
            try:
                isForum = param.args[0].forum
            except Exception as e:
                isForum = False
            return isForum
        return False


class CommandHelperPlugin(BasePlugin):
    def __init__(self):
        super().__init__()
    def on_plugin_load(self) -> None:
        try:
            chat_obj = find_class("org.telegram.messenger.ChatObject")
            TLRPC_chat = find_class("org.telegram.tgnet.TLRPC$Chat")
            are_tabs_enabled = chat_obj.getClass().getDeclaredMethod("areTabsEnabled", TLRPC_chat)
            are_tabs_enabled.setAccessible(True)
            handler = areTabsEnabledHook(self)
            self.are_enabled = self.hook_method(are_tabs_enabled, handler, priority=100)
        except Exception as e:
            self.log(f"Failed {e}")
