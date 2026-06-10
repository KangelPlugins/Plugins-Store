import hashlib
import os
import urllib.parse
import urllib.request

from android_utils import run_on_ui_thread
from base_plugin import AppEvent, BasePlugin, MethodHook
from client_utils import PLUGINS_QUEUE, get_last_fragment, run_on_queue
from file_utils import ensure_dir_exists, get_plugins_dir
from java import jclass
from java.lang import ClassLoader

__id__ = "etg_max"
__name__ = "MAX Tab"
__description__ = "Adds a rightmost MAX tab to ExteraGram chat folders and opens web.max.ru in a native WebView."
__author__ = "@bsod4ik_plugins"
__version__ = "1.0.0"
__icon__ = "msg_plugins"
__app_version__ = ">=12.5.1"
__sdk_version__ = ">=1.4.3.3"

ENTRY_CLASS = "com.etgmax.bridge.MaxBridge"
DEFAULT_DEX_URL = "https://github.com/nulls-brawl-site/etg-max-tab/releases/download/v1.0.0/etg-max-bridge-ab39eb11.dex"
DEFAULT_DEX_SHA256 = "ab39eb11401728d22a18e5ff5ffa7a91fbc0863900221ebe1c3b1f5ad818315d"


class _AfterCreateView(MethodHook):
    def __init__(self, plugin):
        self.plugin = plugin

    def after_hooked_method(self, param):
        self.plugin.install_tab(param.thisObject)


class _AfterUpdateTabs(MethodHook):
    def __init__(self, plugin):
        self.plugin = plugin

    def after_hooked_method(self, param):
        self.plugin.install_tab(param.thisObject)


class _BeforeUpdateTabs(MethodHook):
    def __init__(self, plugin):
        self.plugin = plugin

    def before_hooked_method(self, param):
        self.plugin.before_update_tabs(param.thisObject)


class _BeforeDestroy(MethodHook):
    def __init__(self, plugin):
        self.plugin = plugin

    def before_hooked_method(self, param):
        self.plugin.hide_tab(param.thisObject)


class _BeforeOpenUrl(MethodHook):
    def __init__(self, plugin):
        self.plugin = plugin

    def before_hooked_method(self, param):
        self.plugin.handle_browser_open(param)


class _BeforeUrlSpanClick(MethodHook):
    def __init__(self, plugin):
        self.plugin = plugin

    def before_hooked_method(self, param):
        self.plugin.handle_url_span_click(param)


class MaxTabPlugin(BasePlugin):
    def on_plugin_load(self):
        self._bridge = None
        self._bridge_install = None
        self._bridge_before_update = None
        self._bridge_hide = None
        self._bridge_open_max_url = None
        self._bridge_ready = False
        self._pending_fragments = []
        self._pending_max_links = []
        run_on_queue(self._load_bridge, PLUGINS_QUEUE)
        self._install_hooks()

    def on_plugin_unload(self):
        self._pending_fragments = []
        self._pending_max_links = []

    def _install_hooks(self):
        DialogsActivity = self._class_ref("org.telegram.ui.DialogsActivity")
        MainTabsActivity = self._class_ref("org.telegram.ui.MainTabsActivity")
        Browser = self._class_ref("org.telegram.messenger.browser.Browser")
        Context = self._class_ref("android.content.Context")
        View = self._class_ref("android.view.View")
        Boolean = jclass("java.lang.Boolean")
        if DialogsActivity is None or Context is None:
            self._log(f"MAX Tab: class lookup failed DialogsActivity={DialogsActivity is not None} Context={Context is not None}")
            return
        try:
            create_view = DialogsActivity.getDeclaredMethod("createView", Context)
            create_view.setAccessible(True)
            self.hook_method(create_view, _AfterCreateView(self))

            update_tabs = DialogsActivity.getDeclaredMethod("updateFilterTabs", Boolean.TYPE, Boolean.TYPE)
            update_tabs.setAccessible(True)
            self.hook_method(update_tabs, _BeforeUpdateTabs(self))
            self.hook_method(update_tabs, _AfterUpdateTabs(self))

            destroy = DialogsActivity.getDeclaredMethod("onFragmentDestroy")
            destroy.setAccessible(True)
            self.hook_method(destroy, _BeforeDestroy(self))

            if MainTabsActivity is not None:
                main_create_view = MainTabsActivity.getDeclaredMethod("createView", Context)
                main_create_view.setAccessible(True)
                self.hook_method(main_create_view, _AfterCreateView(self))

                main_resume = MainTabsActivity.getDeclaredMethod("onResume")
                main_resume.setAccessible(True)
                self.hook_method(main_resume, _AfterCreateView(self))

                Bundle = self._class_ref("android.os.Bundle")
                if Bundle is not None:
                    prepare_dialogs = MainTabsActivity.getDeclaredMethod("prepareDialogsActivity", Bundle)
                    prepare_dialogs.setAccessible(True)
                    self.hook_method(prepare_dialogs, _AfterCreateView(self))

            browser_hooks = 0
            if Browser is not None:
                browser_hooks += len(self.hook_all_methods(Browser, "openUrl", _BeforeOpenUrl(self)))
                browser_hooks += len(self.hook_all_methods(Browser, "openUrlInSystemBrowser", _BeforeOpenUrl(self)))

            span_hooks = 0
            if View is not None:
                for class_name in (
                    "org.telegram.ui.Components.URLSpanNoUnderline",
                    "org.telegram.ui.Components.URLSpanBrowser",
                    "org.telegram.ui.Components.URLSpanReplacement",
                ):
                    span = self._class_ref(class_name)
                    if span is None:
                        continue
                    try:
                        on_click = span.getDeclaredMethod("onClick", View)
                        on_click.setAccessible(True)
                        self.hook_method(on_click, _BeforeUrlSpanClick(self))
                        span_hooks += 1
                    except Exception as e:
                        self._log(f"MAX Tab: span hook failed {class_name}: {e}")

            self._log(f"MAX Tab: hooks installed mainTabs={MainTabsActivity is not None} browserHooks={browser_hooks} spanHooks={span_hooks}")
        except Exception as e:
            self._log(f"MAX Tab: hook install failed: {e}")

    def _load_bridge(self):
        try:
            dex_path = self._ensure_dex()
            if not dex_path:
                return
            ctx = jclass("org.telegram.messenger.ApplicationLoader").applicationContext
            opt_dir = os.path.join(self._dex_dir(), "dex_opt")
            ensure_dir_exists(opt_dir)
            DexClassLoader = jclass("dalvik.system.DexClassLoader")
            loader = DexClassLoader(dex_path, opt_dir, None, ctx.getClassLoader() or ClassLoader.getSystemClassLoader())
            self._bridge = loader.loadClass(ENTRY_CLASS)
            Object = self._class_ref("java.lang.Object")
            String = self._class_ref("java.lang.String")
            self._bridge_install = self._bridge.getDeclaredMethod("installWithStatus", Object)
            self._bridge_install.setAccessible(True)
            self._bridge_before_update = self._bridge.getDeclaredMethod("beforeUpdateFilterTabs", Object)
            self._bridge_before_update.setAccessible(True)
            self._bridge_hide = self._bridge.getDeclaredMethod("hide", Object)
            self._bridge_hide.setAccessible(True)
            self._bridge_open_max_url = self._bridge.getDeclaredMethod("openMaxUrl", Object, String)
            self._bridge_open_max_url.setAccessible(True)
            self._bridge_ready = True
            self._log(f"MAX Tab: dex bridge loaded from {dex_path}")
            for fragment in list(self._pending_fragments):
                self.install_tab(fragment)
            self._pending_fragments = []
            for source, url in list(self._pending_max_links):
                run_on_ui_thread(lambda source=source, url=url: self._route_max_url(source, url), 0)
            self._pending_max_links = []
            self._schedule_current_fragment_install()
        except Exception as e:
            self._log(f"MAX Tab: dex load failed: {e}")

    def on_app_event(self, event_type: AppEvent):
        if event_type == AppEvent.RESUME:
            self._schedule_current_fragment_install()

    def _ensure_dex(self):
        dex_dir = self._dex_dir()
        ensure_dir_exists(dex_dir)
        dex_path = os.path.join(dex_dir, "etg-max-bridge.dex")
        url = self._dex_url()
        expected_sha = self._expected_dex_sha()
        if os.path.exists(dex_path) and os.path.getsize(dex_path) > 1024:
            if not expected_sha or self._sha256(dex_path) == expected_sha:
                self._make_read_only(dex_path)
                self._log(f"MAX Tab: using cached dex at {dex_path}")
                return dex_path
        tmp_path = dex_path + ".tmp"
        try:
            self._log(f"MAX Tab: downloading dex to {dex_path}")
            try:
                if os.path.exists(tmp_path):
                    os.remove(tmp_path)
            except Exception:
                pass
            with urllib.request.urlopen(url, timeout=25) as response:
                data = response.read()
            if len(data) < 1024:
                self._log("MAX Tab: downloaded dex is too small")
                return None
            got_sha = hashlib.sha256(data).hexdigest()
            if expected_sha and got_sha != expected_sha:
                self._log(f"MAX Tab: dex sha mismatch: {got_sha}")
                return None
            with open(tmp_path, "wb") as f:
                f.write(data)
            self._make_read_only(tmp_path)
            os.replace(tmp_path, dex_path)
            self._make_read_only(dex_path)
            self._log(f"MAX Tab: dex downloaded sha256={got_sha}")
            return dex_path
        except Exception as e:
            self._log(f"MAX Tab: dex download failed: {e}")
            return None
        finally:
            try:
                if os.path.exists(tmp_path):
                    os.remove(tmp_path)
            except Exception:
                return None

    def install_tab(self, fragment):
        if fragment is None:
            return
        if not self._bridge_ready:
            if fragment not in self._pending_fragments:
                self._pending_fragments.append(fragment)
            return
        try:
            result = self._bridge_install.invoke(None, fragment)
            self._log(f"MAX Tab: {result}")
        except Exception as e:
            self._log(f"MAX Tab: install failed: {e}")

    def before_update_tabs(self, fragment):
        if self._bridge_ready and self._bridge_before_update is not None and fragment is not None:
            try:
                self._bridge_before_update.invoke(None, fragment)
            except Exception as e:
                self._log(f"MAX Tab: before update failed: {e}")

    def _schedule_current_fragment_install(self):
        if not self._bridge_ready:
            return
        for delay in (0, 250, 1000, 2500):
            run_on_ui_thread(lambda: self._install_current_fragment(), delay)

    def _install_current_fragment(self):
        try:
            fragment = get_last_fragment()
            if fragment is None:
                self._log("MAX Tab: current fragment is null")
                return
            self._log(f"MAX Tab: current fragment={fragment.getClass().getName()}")
            self.install_tab(fragment)
        except Exception as e:
            self._log(f"MAX Tab: current fragment install failed: {e}")

    def hide_tab(self, fragment):
        if self._bridge_ready and self._bridge_hide is not None and fragment is not None:
            try:
                self._bridge_hide.invoke(None, fragment)
            except Exception:
                return

    def handle_browser_open(self, param):
        try:
            args = list(param.args or [])
            url = self._extract_url_arg(args)
            if not url or not self._is_max_url(url):
                return
            source = args[0] if args else None
            if self._route_max_url(source, url):
                param.setResult(None)
        except Exception as e:
            self._log(f"MAX Tab: link hook failed: {e}")

    def handle_url_span_click(self, param):
        try:
            url = self._get_span_url(param.thisObject)
            if not url or not self._is_max_url(url):
                return
            args = list(param.args or [])
            source = args[0] if args else None
            if self._route_max_url(source, url):
                param.setResult(None)
        except Exception as e:
            self._log(f"MAX Tab: span link hook failed: {e}")

    def _route_max_url(self, source, url):
        if not url or not self._is_max_url(url):
            return False
        if not self._bridge_ready or self._bridge_open_max_url is None:
            if (source, url) not in self._pending_max_links:
                self._pending_max_links.append((source, url))
                self._pending_max_links = self._pending_max_links[-8:]
            self._log(f"MAX Tab: max link queued until bridge is ready url={url}")
            return True
        try:
            if source is None:
                source = get_last_fragment()
            result = self._bridge_open_max_url.invoke(None, source, url)
            result_text = str(result)
            self._log(f"MAX Tab: {result_text}")
            return result_text.startswith("link: opened")
        except Exception as e:
            self._log(f"MAX Tab: route max link failed: {e}")
            return False

    def _get_span_url(self, span):
        if span is None:
            return None
        try:
            return str(span.getURL())
        except Exception:
            pass
        try:
            method = span.getClass().getMethod("getURL")
            return str(method.invoke(span))
        except Exception:
            return None

    def _extract_url_arg(self, args):
        for arg in args[1:] if len(args) > 1 else args:
            if arg is None:
                continue
            if isinstance(arg, str):
                return arg
            try:
                class_name = arg.getClass().getName()
            except Exception:
                class_name = ""
            if class_name == "java.lang.String":
                return str(arg)
            if class_name == "android.net.Uri":
                try:
                    return str(arg.toString())
                except Exception:
                    return str(arg)
            try:
                text = str(arg.toString())
                if self._looks_like_max_url(text):
                    return text
            except Exception:
                pass
            try:
                text = str(arg)
                if self._looks_like_max_url(text):
                    return text
            except Exception:
                pass
        return None

    def _looks_like_max_url(self, url):
        value = (str(url) if url is not None else "").strip().lower()
        return (
            value.startswith(("max://", "oneme://", "https://max.ru", "http://max.ru", "https://web.max.ru", "http://web.max.ru"))
            or value == "max.ru"
            or value.startswith("max.ru/")
            or ".max.ru/" in value
        )

    def _is_max_url(self, url):
        value = (str(url) if url is not None else "").strip()
        if not value:
            return False
        lower = value.lower()
        if lower.startswith(("max://", "oneme://")):
            return True
        candidate = value if "://" in value else f"https://{value}"
        try:
            parsed = urllib.parse.urlparse(candidate)
            host = (parsed.hostname or "").lower()
        except Exception:
            return False
        return host == "max.ru" or host.endswith(".max.ru")

    def _dex_dir(self):
        return os.path.join(get_plugins_dir(), __id__)

    def _expected_dex_sha(self):
        return DEFAULT_DEX_SHA256

    def _dex_url(self):
        return DEFAULT_DEX_URL

    def _class_ref(self, name):
        try:
            ctx = jclass("org.telegram.messenger.ApplicationLoader").applicationContext
            loader = ctx.getClassLoader()
            if loader is not None:
                return loader.loadClass(name)
        except Exception:
            pass
        try:
            Class = jclass("java.lang.Class")
            return Class.forName(name)
        except Exception:
            pass
        try:
            cls = jclass(name)
            if hasattr(cls, "getDeclaredMethod"):
                return cls
            if hasattr(cls, "class_"):
                return cls.class_
        except Exception:
            return None
        return None

    def _make_read_only(self, path):
        try:
            os.chmod(path, 0o444)
            return
        except Exception:
            pass
        try:
            File = jclass("java.io.File")
            File(path).setReadOnly()
        except Exception as e:
            self._log(f"MAX Tab: failed to make dex read-only: {e}")

    def _log(self, _message):
        return

    def _sha256(self, path):
        h = hashlib.sha256()
        with open(path, "rb") as f:
            for chunk in iter(lambda: f.read(1024 * 128), b""):
                h.update(chunk)
        return h.hexdigest()
