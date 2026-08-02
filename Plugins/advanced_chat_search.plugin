from base_plugin import BasePlugin, MethodHook
from hook_utils import find_class, get_private_field, set_private_field
from android_utils import run_on_ui_thread
from java import dynamic_proxy, jclass
from org.telegram.tgnet import TLRPC


__id__ = "advanced_chat_search"
__name__ = "Advanced Chat Search"
__description__ = (
    "Extends chat search-by-sender with people from all dialogs and Telegram's "
    "global user search."
)
__author__ = "@anivplugins & @mraniv"
__version__ = "1.0.0"
__icon__ = "UtyaDuckFull/17"
__app_version__ = ">=12.5.1"
__sdk_version__ = ">=1.4.0"


MENTIONS_ADAPTER_CLASS = "org.telegram.ui.Adapters.MentionsAdapter"
SEARCH_DEBOUNCE_MS = 250
SEARCH_LIMIT = 20

ArrayList = jclass("java.util.ArrayList")
LongSparseArray = jclass("android.util.LongSparseArray")
System = jclass("java.lang.System")
ConnectionsManager = jclass("org.telegram.tgnet.ConnectionsManager")
MessagesController = jclass("org.telegram.messenger.MessagesController")
RequestDelegate = jclass("org.telegram.tgnet.RequestDelegate")


class _GlobalSearchDelegate(dynamic_proxy(RequestDelegate)):
    def __init__(self, plugin, adapter_key, generation):
        super().__init__()
        self.plugin = plugin
        self.adapter_key = adapter_key
        self.generation = generation

    def run(self, response, error):
        try:
            self.plugin._on_global_search_response(
                self.adapter_key,
                self.generation,
                response,
                error,
            )
        except Exception as exc:
            self.plugin._log_once(
                "global_callback",
                f"Global search callback failed: {exc}",
            )


class _SearchingMentionsHook(MethodHook):
    def __init__(self, plugin):
        self.plugin = plugin

    def after_hooked_method(self, param):
        try:
            self.plugin._after_set_searching_mentions(param)
        except Exception as exc:
            self.plugin._log_once(
                "search_mode_hook",
                f"Search mode hook failed: {exc}",
            )


class _SearchQueryHook(MethodHook):
    def __init__(self, plugin):
        self.plugin = plugin

    def before_hooked_method(self, param):
        try:
            self.plugin._before_search_query(param)
        except Exception as exc:
            self.plugin._log_once(
                "search_query_hook",
                f"Search query hook failed: {exc}",
            )


class _ShowUsersResultHook(MethodHook):
    def __init__(self, plugin):
        self.plugin = plugin

    def after_hooked_method(self, param):
        try:
            self.plugin._after_show_users_result(param)
        except Exception as exc:
            self.plugin._log_once(
                "show_results_hook",
                f"Result merge hook failed: {exc}",
            )


class AllOfSearchInChatPlugin(BasePlugin):
    def __init__(self):
        super().__init__()
        self._active = False
        self._unhooks = []
        self._states = {}
        self._logged = set()

    def on_plugin_load(self):
        self.on_plugin_unload()
        self._active = True

        try:
            adapter_class = find_class(MENTIONS_ADAPTER_CLASS)
            if adapter_class is None:
                self._log_once(
                    "missing_adapter",
                    f"Class not found: {MENTIONS_ADAPTER_CLASS}",
                )
                self._active = False
                return

            self._install_hook(
                adapter_class,
                "setSearchingMentions",
                _SearchingMentionsHook(self),
            )
            self._install_hook(
                adapter_class,
                "searchUsernameOrHashtag",
                _SearchQueryHook(self),
            )
            self._install_hook(
                adapter_class,
                "showUsersResult",
                _ShowUsersResultHook(self),
            )
        except Exception as exc:
            self._log_once("load", f"Plugin load failed: {exc}")

    def on_plugin_unload(self):
        self._active = False

        for state in list(getattr(self, "_states", {}).values()):
            try:
                self._cancel_request(state)
            except Exception:
                pass
            try:
                adapter = state.get("adapter")
                if adapter is not None:
                    adapter.setSearchInDialogs(False)
            except Exception:
                pass
        self._states.clear()

        for unhook in list(getattr(self, "_unhooks", [])):
            try:
                self.unhook_method(unhook)
            except Exception:
                pass
        self._unhooks.clear()

    def _install_hook(self, adapter_class, method_name, hook):
        try:
            unhooks = self.hook_all_methods(
                adapter_class,
                method_name,
                hook,
                priority=10000,
            )
            if unhooks:
                self._unhooks.extend(unhooks)
            else:
                self._log_once(
                    f"missing_method_{method_name}",
                    f"Method not hooked: {MENTIONS_ADAPTER_CLASS}.{method_name}",
                )
        except Exception as exc:
            self._log_once(
                f"hook_{method_name}",
                f"Could not hook {method_name}: {exc}",
            )

    def _after_set_searching_mentions(self, param):
        args = param.args
        if args is None or len(args) != 1:
            return

        adapter = param.thisObject
        if adapter is None:
            return

        enabled = bool(args[0])
        adapter.setSearchInDialogs(enabled)

        adapter_key = self._adapter_key(adapter)
        if enabled:
            self._get_or_create_state(adapter_key, adapter)
        else:
            state = self._states.pop(adapter_key, None)
            if state is not None:
                self._cancel_request(state)

    def _before_search_query(self, param):
        args = param.args
        if args is None or len(args) != 5:
            return
        if not bool(args[3]) or not bool(args[4]):
            return

        adapter = param.thisObject
        if adapter is None:
            return

        text = "" if args[0] is None else str(args[0])
        query = text[1:] if text.startswith("@") else text
        query = query.strip()

        adapter_key = self._adapter_key(adapter)
        state = self._get_or_create_state(adapter_key, adapter)
        self._cancel_request(state)
        state["generation"] += 1
        generation = state["generation"]
        state["query"] = query
        state["users"] = []

        if not query:
            return

        run_on_ui_thread(
            lambda: self._start_global_search(adapter_key, generation),
            SEARCH_DEBOUNCE_MS,
        )

    def _after_show_users_result(self, param):
        adapter = param.thisObject
        if adapter is None:
            return
        state = self._states.get(self._adapter_key(adapter))
        if state is None or not state["users"]:
            return
        self._merge_global_users(state)

    def _start_global_search(self, adapter_key, generation):
        try:
            if not self._active:
                return

            state = self._states.get(adapter_key)
            if state is None or state["generation"] != generation:
                return

            query = state["query"]
            if not query:
                return

            request = TLRPC.TL_contacts_search()
            request.q = query
            request.limit = SEARCH_LIMIT

            delegate = _GlobalSearchDelegate(self, adapter_key, generation)
            state["delegate"] = delegate
            manager = ConnectionsManager.getInstance(state["account"])
            state["request_id"] = int(manager.sendRequest(request, delegate))
        except Exception as exc:
            self._log_once(
                "global_request",
                f"Could not start global user search: {exc}",
            )

    def _on_global_search_response(
        self,
        adapter_key,
        generation,
        response,
        error,
    ):
        try:
            run_on_ui_thread(
                lambda: self._apply_global_search_response(
                    adapter_key,
                    generation,
                    response,
                    error,
                )
            )
        except Exception as exc:
            self._log_once(
                "global_response_handoff",
                f"Could not hand off global search response: {exc}",
            )

    def _apply_global_search_response(
        self,
        adapter_key,
        generation,
        response,
        error,
    ):
        try:
            if not self._active:
                return

            state = self._states.get(adapter_key)
            if state is None or state["generation"] != generation:
                return

            state["request_id"] = 0
            state["delegate"] = None
            if error is not None or response is None:
                return

            try:
                MessagesController.getInstance(state["account"]).putUsers(
                    response.users,
                    False,
                )
            except Exception as exc:
                self._log_once(
                    "cache_users",
                    f"Could not cache global user results: {exc}",
                )

            users_by_id = {}
            for index in range(int(response.users.size())):
                user = response.users.get(index)
                users_by_id[int(user.id)] = user

            ordered_users = []
            seen = set()
            for peers in (response.my_results, response.results):
                for index in range(int(peers.size())):
                    peer = peers.get(index)
                    user_id = int(peer.user_id)
                    if user_id == 0 or user_id in seen:
                        continue
                    user = users_by_id.get(user_id)
                    if user is None or bool(user.deleted):
                        continue
                    seen.add(user_id)
                    ordered_users.append(user)

            state["users"] = ordered_users
            if ordered_users:
                self._merge_global_users(state)
        except Exception as exc:
            self._log_once(
                "global_response",
                f"Could not apply global user results: {exc}",
            )

    def _merge_global_users(self, state):
        adapter = state.get("adapter")
        if adapter is None or not state["users"]:
            return

        result_list = get_private_field(adapter, "searchResultUsernames")
        result_map = get_private_field(adapter, "searchResultUsernamesMap")

        if result_list is None:
            result_list = ArrayList()
            if not set_private_field(adapter, "searchResultUsernames", result_list):
                self._log_once(
                    "set_result_list",
                    "Could not replace MentionsAdapter.searchResultUsernames",
                )
                return

        if result_map is None:
            result_map = LongSparseArray()
            if not set_private_field(adapter, "searchResultUsernamesMap", result_map):
                self._log_once(
                    "set_result_map",
                    "Could not replace MentionsAdapter.searchResultUsernamesMap",
                )
                return

        added = False
        for user in state["users"]:
            user_id = int(user.id)
            if result_map.indexOfKey(user_id) >= 0:
                continue
            result_list.add(user)
            result_map.put(user_id, user)
            added = True

        if not added:
            return

        adapter.notifyDataSetChanged()
        delegate = get_private_field(adapter, "delegate")
        if delegate is not None:
            delegate.needChangePanelVisibility(True)

    def _get_or_create_state(self, adapter_key, adapter):
        state = self._states.get(adapter_key)
        if state is None:
            account = get_private_field(adapter, "currentAccount")
            state = {
                "adapter": adapter,
                "account": int(account),
                "generation": 0,
                "query": "",
                "users": [],
                "request_id": 0,
                "delegate": None,
            }
            self._states[adapter_key] = state
        return state

    @staticmethod
    def _adapter_key(adapter):
        return int(System.identityHashCode(adapter))

    @staticmethod
    def _cancel_request(state):
        request_id = int(state.get("request_id", 0) or 0)
        try:
            if request_id:
                ConnectionsManager.getInstance(state["account"]).cancelRequest(
                    request_id,
                    True,
                )
        finally:
            state["request_id"] = 0
            state["delegate"] = None

    def _log_once(self, key, message):
        if key in self._logged:
            return
        self._logged.add(key)
        self.log(message)
