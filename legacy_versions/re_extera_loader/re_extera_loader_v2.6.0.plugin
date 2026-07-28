# metadata
__id__ = "re_extera_loader"
__name__ = "aartzz's re:extera"
__description__ = "Actively maintained FOSS fork. Enable ghost mode, save deleted messages and more!"
__author__ = "@shiawasez | @shikaatuxplugins \noriginal author: @bleizixPlugins\nFOSS recovery by @fossSquad & @migor1103"
__version__ = "2.6.0"
__icon__ = "myadestes_1_amashiro_natsuki_plus_nacho_neko/30"
__min_version__ = "12.8.1"

# imports
from typing import Any, List
from base_plugin import BasePlugin, MethodHook
from client_utils import get_last_fragment
import requests
import json
import io
import zipfile
import shutil
import os
import time
import threading
from android.app import Activity
from hook_utils import find_class
from ui.bulletin import BulletinHelper
from ui.settings import Header, Divider, Text, Switch
from org.telegram.ui import LaunchActivity
from org.telegram.messenger import LocaleController, BuildVars, ApplicationLoader, FileLoader, NotificationCenter, MessageObject, AndroidUtilities
from java.nio import ByteBuffer
from dalvik.system import InMemoryDexClassLoader, DexClassLoader
from java import dynamic_proxy
from java.lang import Runnable

class UIRunnable(dynamic_proxy(Runnable)):
    def __init__(self, func):
        super().__init__()
        self.func = func
    def run(self):
        self.func()


# constants
CLASS_NAME = "ni.shikatu.re_extera.Main"
METHOD_NAME = "start"
DEV_ARTIFACT_URL = "https://nightly.link/fossSquad/re-extera/workflows/build/master/re-extera-dev.zip"
DEV_API_URL = "https://api.github.com/repos/fossSquad/re-extera/actions/workflows/build.yml/runs?branch=master&per_page=1&status=success"
RELEASE_API_URL = "https://api.github.com/repos/fossSquad/re-extera/releases"
LOCAL_DEX_PATH = "/storage/emulated/0/Android/media/com.exteragram.messenger/classes.dex"
CACHE_DIR_NAME = "re_extera_cache"
DEX_OPT_DIR_NAME = "dex_opt"
VERSION_FIELD_NAME = "VERSION"
USER_AGENT = "Mozilla/5.0 (X11; Linux x86_64; rv:152.0) Gecko/20100101 Firefox/152.0"
DEV_RUN_URL_TEMPLATE = "https://nightly.link/fossSquad/re-extera/actions/runs/{}/re-extera-dev.zip"
DEV_DOWNLOAD_HEADERS = {
    "User-Agent": USER_AGENT,
    "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8",
    "Accept-Language": "en-US,en;q=0.9",
    "Accept-Encoding": "gzip, deflate, br, zstd",
    "Connection": "keep-alive",
    "Referer": "https://nightly.link/fossSquad/re-extera/workflows/build/master?preview",
    "Upgrade-Insecure-Requests": "1",
    "Sec-Fetch-Dest": "document",
    "Sec-Fetch-Mode": "navigate",
    "Sec-Fetch-Site": "same-origin",
    "Sec-Fetch-User": "?1",
    "Priority": "u=0, i",
}


# localization
def _localize(key):
    ru = LocaleController.getInstance().getCurrentLocale().getLanguage() == "ru"
    strings = {
        "settings":        ("Настройки re:extera",  "re:extera Settings"),
        "channel_dev":     ("Dev-сборки",            "Dev builds"),
        "channel_release": ("Релизные сборки",       "Release builds"),
        "check_updates":   ("Проверить обновления",  "Check for updates"),
        "install_file":    ("Установить из файла",   "Install from file"),
        "update_avail":    ("Доступна новая версия re:extera! Перезапустите приложение для применения обновления.",
                            "New re:extera version available! Restart the app to apply the update."),
        "downloading":     ("Загрузка...",           "Downloading..."),
        "updated_cache":   ("Обновлено из кеша",     "Updated from cache"),
        "installed":       ("Установка завершена",   "Install completed"),
        "channel_switch":  ("Канал изменён. Перезапустите приложение.", "Channel changed. Restart the app."),
        "up_to_date":      ("Уже последняя версия",  "Already up to date"),
        "file_not_found":  ("Файл не найден",        "File not found"),
        "no_official_update": ("re:extera ещё не имеет официального обновления для этой версии, возможны баги.", 
                               "re:extera doesn't have official update for this version, expect bugs."),
        "copy_logs":       ("Скопировать логи",      "Copy logs"),
        "logs_copied":     ("Логи скопированы в буфер обмена", "Logs copied to clipboard"),
    }
    return strings[key][1 if not ru else 0]


# file download handler from dev
class DownloadListener(dynamic_proxy(NotificationCenter.NotificationCenterDelegate)):
    def __init__(self, plugin, account, file_name, download_path, target_path):
        super().__init__()
        self.plugin = plugin
        self.account = account
        self.file_name = file_name
        self.download_path = download_path
        self.target_path = target_path
        self.nc = NotificationCenter.getInstance(account)

    def didReceivedNotification(self, id, account, args):
        try:
            loaded_file_name = str(args[0])
            if loaded_file_name == self.file_name:
                if id == NotificationCenter.fileLoaded:
                    self.plugin.log(f"File {self.file_name} downloaded success")
                    if os.path.exists(self.download_path):
                        shutil.copy2(self.download_path, self.target_path)
                        self.plugin.log(f"Copied to {self.target_path}")
                        BulletinHelper.show_info(_localize("installed"))
                    self.cleanup()
                elif id == NotificationCenter.fileLoadFailed:
                    self.plugin.log(f"File {self.file_name} download failed")
                    self.cleanup()
        except Exception as e:
            self.plugin.log(f"Listener error: {e}")

    def cleanup(self):
        self.nc.removeObserver(self, NotificationCenter.fileLoaded)
        self.nc.removeObserver(self, NotificationCenter.fileLoadFailed)


# default config
class Config:
    MIN_CHECK_INTERVAL = 60  # seconds between version checks

    def __init__(self, cache_dir):
        self.path = os.path.join(cache_dir, "config.json")
        self.data = self._load()

    def _load(self):
        try:
            if os.path.exists(self.path):
                with open(self.path, "r") as f:
                    return json.load(f)
        except Exception:
            pass
        return {"channel": "release", "versions": {}, "last_check": 0, "last_error": ""}

    def _save(self):
        try:
            d = os.path.dirname(self.path)
            if not os.path.exists(d):
                os.makedirs(d)
            with open(self.path, "w") as f:
                json.dump(self.data, f)
        except Exception as e:
            pass

    @property
    def channel(self):
        return self.data.get("channel", "release")

    @channel.setter
    def channel(self, value):
        self.data["channel"] = value
        self._save()

    def get_version(self, channel):
        return self.data.get("versions", {}).get(channel, 0)

    def set_version(self, channel, version):
        self.data.setdefault("versions", {})[channel] = version
        self._save()

    def clear_cache_for(self, channel):
        self.data.setdefault("versions", {})[channel] = 0
        self._save()

    def can_check(self):
        now = time.time()
        last = self.data.get("last_check", 0)
        return now - last >= self.MIN_CHECK_INTERVAL

    def mark_checked(self):
        self.data["last_check"] = time.time()
        self._save()


# .dex loader
class Loader:
    def __init__(self, plugin: BasePlugin, activity: Activity):
        self.plugin = plugin
        self.activity = activity
        self.instance = None
        self.dex_loader = None

        base_cache = os.path.join(
            ApplicationLoader.applicationContext.getFilesDir().getAbsolutePath(),
            CACHE_DIR_NAME
        )
        self.config = Config(base_cache)
        self.channel = self.config.channel
        self.cache_dir = os.path.join(base_cache, self.channel)
        self.cache_file = os.path.join(self.cache_dir, "cached.dat")

        if not os.path.exists(self.cache_dir):
            os.makedirs(self.cache_dir)

    def _switch_cache(self, channel):
        self.channel = channel
        base_cache = os.path.join(
            ApplicationLoader.applicationContext.getFilesDir().getAbsolutePath(),
            CACHE_DIR_NAME
        )
        self.cache_dir = os.path.join(base_cache, channel)
        self.cache_file = os.path.join(self.cache_dir, "cached.dat")
        if not os.path.exists(self.cache_dir):
            os.makedirs(self.cache_dir)

    def getInstance(self):
        if self.instance is None:
            try:
                method = find_class(CLASS_NAME).getClass().getMethod("getInstance")
                self.instance = method.invoke(None)
            except Exception as e:
                self.plugin.log(f"Error getting instance: {e}")
        return self.instance

    def _load_dex_inmemory(self, bytesdex):
        buffer = ByteBuffer.wrap(bytesdex)
        loader = InMemoryDexClassLoader(
            buffer, ApplicationLoader.applicationContext.getClassLoader()
        )
        clazz = loader.loadClass(CLASS_NAME)
        return clazz, loader

    def _load_dex_from_file(self, dex_path):
        loader = DexClassLoader(
            dex_path,
            ApplicationLoader.applicationContext.getDir(DEX_OPT_DIR_NAME, 0).getAbsolutePath(),
            None,
            ApplicationLoader.applicationContext.getClassLoader()
        )
        clazz = loader.loadClass(CLASS_NAME)
        return clazz, loader

    def _call_start(self, clazz):
        try:
            start_method = clazz.getMethod("initAndStart")
            start_method.invoke(None)
            self.plugin.log("Called initAndStart")
        except Exception as e:
            self.plugin.log(f"initAndStart failed: {e}")
            try:
                # Fallback for older DEX versions
                get_instance = clazz.getMethod("getInstance")
                instance = get_instance.invoke(None)
                start_method = clazz.getMethod("start")
                start_method.invoke(instance)
                self.plugin.log("Called start() via getInstance() fallback")
            except Exception as e2:
                self.plugin.log(f"Fallback start failed: {e2}")

    def start_from_bytes(self, bytesdex):
        cache_dir = self.cache_dir
        dex_path = os.path.join(cache_dir, "classes.dex")

        try:
            if not os.path.exists(cache_dir):
                os.makedirs(cache_dir)
            with open(dex_path, 'wb') as f:
                f.write(bytesdex)
        except Exception as e:
            self.plugin.log(f"Failed to cache DEX to file: {e}")

        try:
            clazz, loader = self._load_dex_inmemory(bytesdex)
            self.dex_loader = loader
            self.dex_main_class = clazz
            self._call_start(clazz)
            self.plugin.log(f"Loaded {CLASS_NAME} (in-memory)")
            return
        except Exception as e:
            proxy_err = "proxy" in str(e).lower()
            self.plugin.log(f"InMemory load {'proxy issue' if proxy_err else 'failed'}: {e}")

        if os.path.exists(dex_path):
            try:
                clazz, loader = self._load_dex_from_file(dex_path)
                self.dex_loader = loader
                self.dex_main_class = clazz
                self._call_start(clazz)
                self.plugin.log(f"Loaded {CLASS_NAME} (from file)")
                return
            except Exception as e:
                self.plugin.log(f"File load also failed: {e}")

        self.plugin.log(f"DEX proxy unavailable — hooks from static init may still work")
        self.instance = None

    def open_settings(self):
        try:
            inst = self.getInstance()
            if inst:
                method = inst.getClass().getMethod("showSettings")
                method.invoke(inst)
        except Exception as e:
            self.plugin.log(f"Error opening settings: {e}")

    def unload(self):
        try:
            inst = self.getInstance()
            if inst:
                method = inst.getClass().getMethod("onUnload")
                method.invoke(inst)
                self.plugin.log("Unloaded successfully")
        except Exception as e:
            self.plugin.log(f"Error unloading: {e}")

    def _get_cached_version(self):
        try:
            inst = self.getInstance()
            if inst is None:
                return 0
            version_field = inst.getClass().getDeclaredField("VERSION_CODE")
            version_field.setAccessible(True)
            return int(version_field.get(None))
        except Exception as e:
            self.plugin.log(f"Error reading cached version: {e}")
            return 0

    def _check_dev_version(self, force=False):
        if not force and not self.config.can_check():
            self.plugin.log("Dev check skipped (rate limit cooldown)")
            self.config.last_error = "Rate limit: wait 1 min"
            return None, None
        try:
            self.config.mark_checked()
            headers = {"User-Agent": USER_AGENT}
            r = requests.get(DEV_API_URL, headers=headers, timeout=5)
            self.plugin.log(f"Dev API HTTP {r.status_code}")
            r.raise_for_status()
            data = r.json()
            runs = data.get("workflow_runs", [])
            if not runs:
                self.plugin.log("No successful dev runs found")
                return None, None

            run = runs[0]
            run_id = run.get("id", 0)
            self.plugin.log(f"Latest dev run: #{run_id}")
            dev_url = DEV_RUN_URL_TEMPLATE.format(run_id)
            return str(run_id), (dev_url, None)
        except Exception as e:
            self.plugin.log(f"Error checking dev version: {e}")
            return None, None

    def _check_release_version(self, force=False):
        if not force and not self.config.can_check():
            self.plugin.log("Release check skipped (rate limit cooldown)")
            self.config.data["last_error"] = "Rate limit: wait 1 min"
            return None, None
        try:
            self.config.mark_checked()
            headers = {"User-Agent": USER_AGENT}
            r = requests.get(RELEASE_API_URL, headers=headers, timeout=5)
            self.plugin.log(f"Release API HTTP {r.status_code}")
            r.raise_for_status()
            releases = r.json()
            
            tg_version = BuildVars.BUILD_VERSION_STRING
            suffix = f"-{tg_version}"
            
            target_release = None
            for release in releases:
                if release.get("tag_name", "").endswith(suffix):
                    target_release = release
                    break
            
            if not target_release:
                self.plugin.log(f"No release found for Telegram version {tg_version}")
                return "UNSUPPORTED_VERSION", None

            tag = target_release.get("tag_name", "")
            assets = target_release.get("assets", [])
            
            if not tag or not assets:
                self.plugin.log("No release assets found in target release")
                return None, None

            dex_url = None
            plugin_url = None
            for asset in assets:
                name = asset.get("name", "")
                if name.endswith(".dex"):
                    dex_url = asset.get("browser_download_url", "")
                elif name.endswith("loader.plugin"):
                    plugin_url = asset.get("browser_download_url", "")
            if not dex_url:
                self.plugin.log(f"No .dex asset found in release {tag}")
                return None, None

            self.plugin.log(f"Latest release for TG {tg_version}: {tag}")
            return tag, (dex_url, plugin_url)
        except Exception as e:
            self.plugin.log(f"Error checking release version: {e}")
            return None, None

    def _needs_update(self, remote_version):
        cached = self.config.get_version(self.channel)
        self.plugin.log(f"Remote: {remote_version}, Cached: {cached}")
        return str(remote_version) != str(cached)

    def _download_dev_dex(self):
        self.plugin.log(f"Downloading dev DEX from {DEV_ARTIFACT_URL}")
        r = requests.get(DEV_ARTIFACT_URL, headers=DEV_DOWNLOAD_HEADERS, timeout=60)
        r.raise_for_status()
        with zipfile.ZipFile(io.BytesIO(r.content)) as zf:
            dex_path = next((n for n in zf.namelist() if n.endswith("classes.dex")), "classes.dex")
            dex_bytes = zf.read(dex_path)
            try:
                plugin_path = next((n for n in zf.namelist() if n.endswith("loader.plugin")), "loader.plugin")
                plugin_bytes = zf.read(plugin_path)
            except Exception:
                plugin_bytes = None
        self.plugin.log(f"Downloaded {len(dex_bytes)} bytes from nightly.link")
        return dex_bytes, plugin_bytes

    def _download_release_dex(self, url):
        self.plugin.log(f"Downloading release DEX from {url}")
        headers = {"User-Agent": USER_AGENT}
        r = requests.get(url, headers=headers, timeout=60)
        r.raise_for_status()
        self.plugin.log(f"Downloaded {len(r.content)} bytes from releases")
        return r.content

    def download_and_cache(self, remote_version, urls):
        dex_url, plugin_url = urls
        plugin_bytes = None

        if self.channel == "dev":
            dex_bytes, plugin_bytes = self._download_dev_dex()
        else:
            dex_bytes = self._download_release_dex(dex_url)
            if plugin_url:
                try:
                    headers = {"User-Agent": USER_AGENT}
                    r = requests.get(plugin_url, headers=headers, timeout=60)
                    r.raise_for_status()
                    plugin_bytes = r.content
                except Exception as e:
                    self.plugin.log(f"Failed to download release plugin: {e}")

        with open(self.cache_file, 'wb') as f:
            f.write(dex_bytes)

        if plugin_bytes:
            self._update_plugin_file(plugin_bytes)

        self.config.set_version(self.channel, remote_version)
        return dex_bytes

    def _update_plugin_file(self, plugin_bytes):
        try:
            import inspect
            plugin_path = inspect.getfile(self.plugin.__class__)
            if plugin_path and os.path.exists(plugin_path):
                with open(plugin_path, "rb") as f:
                    old_bytes = f.read()
                if old_bytes != plugin_bytes:
                    with open(plugin_path, "wb") as f:
                        f.write(plugin_bytes)
                    self.plugin.log("Plugin updated successfully (requires restart)")
        except Exception as e:
            self.plugin.log(f"Failed to update plugin: {e}")

    def _load_from_local_path(self):
        if os.path.exists(LOCAL_DEX_PATH):
            self.plugin.log(f"Local DEX found at {LOCAL_DEX_PATH}")
            with open(LOCAL_DEX_PATH, 'rb') as f:
                return f.read()
        return None

    def _load_from_cache(self):
        if os.path.exists(self.cache_file):
            with open(self.cache_file, 'rb') as f:
                return f.read()
        return None

    def load_and_start(self):
        self.plugin.log(f"Loading (channel: {self.channel})")

        local_bytes = self._load_from_local_path()
        if local_bytes is not None:
            try:
                self.start_from_bytes(local_bytes)
                self.plugin.log("Loaded from local storage")
                try:
                    self._check_async_update()
                except Exception as e:
                    self.plugin.log(f"Update check failed: {e}")
                return
            except Exception as e:
                self.plugin.log(f"Local DEX failed ({e}), falling through to cache/download")
                try:
                    os.remove(LOCAL_DEX_PATH)
                    self.plugin.log("Removed stale local DEX")
                except Exception:
                    pass

        cached_bytes = self._load_from_cache()
        if cached_bytes is not None:
            try:
                self.start_from_bytes(cached_bytes)
                self.plugin.log("Loaded from cache")
                try:
                    self._check_async_update()
                except Exception as e:
                    self.plugin.log(f"Update check failed: {e}")
            except Exception as e:
                self.plugin.log(f"Cached DEX failed ({e}), will download fresh")
        else:
            self.plugin.log("No cache found")

        if self.instance is None:
            self.plugin.log("Downloading fresh DEX...")
            try:
                remote_version, download_url = self._check_version()

                if remote_version is None:
                    return

                dex_bytes = self.download_and_cache(remote_version, download_url)
                self.start_from_bytes(dex_bytes)
                self.plugin.log("Loaded from fresh download")
            except Exception as e:
                err = str(e)
                if "proxy" in err:
                    return
                self.plugin.log(f"Fatal error: {e}")

    def _check_version(self, force=False):
        if self.channel == "dev":
            return self._check_dev_version(force)
        return self._check_release_version(force)

    def _check_async_update(self):
        def run_check():
            try:
                remote_version, download_url = self._check_version()
                if remote_version == "UNSUPPORTED_VERSION":
                    def show_warn():
                        BulletinHelper.show_info(_localize("no_official_update"), get_last_fragment())
                    AndroidUtilities.runOnUIThread(UIRunnable(show_warn))
                    return
                if remote_version is None:
                    return

                if self._needs_update(remote_version):
                    self.plugin.log("Update available, downloading...")
                    self.download_and_cache(remote_version, download_url)

                    def show_ui():
                        BulletinHelper.show_info(_localize("update_avail"), get_last_fragment())
                    AndroidUtilities.runOnUIThread(UIRunnable(show_ui))
            except Exception as e:
                self.plugin.log(f"Async update failed: {e}")

        threading.Thread(target=run_check).start()

    def check_updates_now(self):
        try:
            BulletinHelper.show_info(_localize("downloading"), get_last_fragment())
            def run_manual():
                try:
                    remote_version, download_url = self._check_version(force=True)
                    if remote_version == "UNSUPPORTED_VERSION":
                        def show_warn():
                            BulletinHelper.show_info(_localize("no_official_update"), get_last_fragment())
                        AndroidUtilities.runOnUIThread(UIRunnable(show_warn))
                        return
                    if remote_version is None:
                        err = self.config.data.get("last_error", "")
                        msg = f"Check failed: {err}" if err else "Failed to check updates"
                        def show_err():
                            BulletinHelper.show_info(msg, get_last_fragment())
                        AndroidUtilities.runOnUIThread(UIRunnable(show_err))
                        return

                    if self._needs_update(remote_version):
                        self.download_and_cache(remote_version, download_url)
                        def show_success():
                            BulletinHelper.show_info(_localize("update_avail"), get_last_fragment())
                        AndroidUtilities.runOnUIThread(UIRunnable(show_success))
                    else:
                        def show_uptodate():
                            BulletinHelper.show_info(_localize("up_to_date"), get_last_fragment())
                        AndroidUtilities.runOnUIThread(UIRunnable(show_uptodate))

                except Exception as e:
                    self.plugin.log(f"Manual update check failed: {e}")
                    def show_exc():
                        BulletinHelper.show_info(f"Error: {e}", get_last_fragment())
                    AndroidUtilities.runOnUIThread(UIRunnable(show_exc))

            threading.Thread(target=run_manual).start()

        except Exception as e:
            self.plugin.log(f"Manual check start failed: {e}")
            BulletinHelper.show_info(f"Error: {e}", get_last_fragment())

    def get_version_display(self):
        try:
            if self.dex_main_class is not None:
                v = self.dex_main_class.getDeclaredField(VERSION_FIELD_NAME)
                v.setAccessible(True)
                ver = str(v.get(None))
                return f"v{ver}"
            return "not loaded"
        except Exception:
            cached = self.config.get_version(self.channel)
            return f"cached:{cached}" if cached else "?"


# loader settings
class Plugin(BasePlugin):
    def __init__(self):
        self.loader = None
        self._logs = []

    def log(self, message):
        import datetime
        ts = datetime.datetime.now().strftime("%H:%M:%S")
        self._logs.append(f"[{ts}] {message}")
        try:
            super().log(message)
        except AttributeError:
            pass

    def create_settings(self) -> List[Any]:
        items = []

        try:
            channel = self.loader.config.channel if self.loader else "release"
            version = self.loader.get_version_display() if self.loader else "?"
        except Exception:
            channel = "release"
            version = "?"

        items.append(Text(text=f"Channel: {channel.upper()}", on_click=lambda v: None))
        items.append(Text(text=f"DEX: {version}", on_click=lambda v: None))
        items.append(Divider())

        items.append(Switch(
            key="dev_channel",
            text=_localize("channel_dev"),
            default=channel == "dev",
            on_change=lambda v: self._on_channel_switch(v)
        ))

        items.append(Divider())

        items.append(Text(
            text=_localize("check_updates"),
            on_click=lambda v: self._on_check_updates()
        ))
        items.append(Text(
            text=_localize("install_file"),
            on_click=lambda v: self._on_install_file()
        ))
        items.append(Text(
            text=_localize("copy_logs"),
            on_click=lambda v: self._on_copy_logs()
        ))
        items.append(Divider())
        items.append(Text(
            text="DEX Settings",
            on_click=lambda v: self._open_re_extera_settings()
        ))

        return items

    def _on_channel_switch(self, is_dev):
        if self.loader is None:
            return
        new_channel = "dev" if is_dev else "release"
        old_channel = self.loader.config.channel
        if new_channel == old_channel:
            return

        self.loader.config.channel = new_channel
        self.loader._switch_cache(new_channel)
        self.log(f"Switched to {new_channel} channel")
        BulletinHelper.show_info(_localize("channel_switch"), get_last_fragment())

    def _on_check_updates(self):
        if self.loader is None:
            return
        self.loader.check_updates_now()

    def _on_install_file(self):
        if self.loader is None:
            return
        if os.path.exists(LOCAL_DEX_PATH):
            try:
                with open(LOCAL_DEX_PATH, 'rb') as f:
                    dex_bytes = f.read()
                self.loader.start_from_bytes(dex_bytes)
                BulletinHelper.show_info(_localize("updated_cache"), get_last_fragment())
                self.log("Reloaded from local file")
            except Exception as e:
                self.log(f"Install from file failed: {e}")
                BulletinHelper.show_info(f"Error: {e}", get_last_fragment())
        else:
            BulletinHelper.show_info(_localize("file_not_found"), get_last_fragment())

    def _on_copy_logs(self):
        try:
            log_text = "\n".join(self._logs) if self._logs else "No logs"
            AndroidUtilities.addToClipboard(log_text)
            BulletinHelper.show_info(_localize("logs_copied"), get_last_fragment())
        except Exception as e:
            self.log(f"Error copying logs: {e}")
            BulletinHelper.show_info(f"Error: {e}", get_last_fragment())

    def _open_re_extera_settings(self):
        try:
            if self.loader is None or self.loader.dex_main_class is None:
                self.log("DEX not loaded")
                return
            try:
                show_method = self.loader.dex_main_class.getMethod("showSettingsExternal")
                show_method.invoke(None)
            except Exception as e:
                self.log(f"showSettingsExternal failed, trying fallback: {e}")
                get_instance = self.loader.dex_main_class.getMethod("getInstance")
                instance = get_instance.invoke(None)
                show_method = self.loader.dex_main_class.getMethod("showSettings")
                show_method.invoke(instance)
        except Exception as e:
            self.log(f"Error opening DEX settings: {e}")

    def on_plugin_load(self) -> None:
        try:
            self.log(f"Init {__version__}")
            launch_activity = get_last_fragment().getContext()
            self.loader = Loader(self, launch_activity)
            self.loader.load_and_start()
        except Exception as e:
            BulletinHelper.show_info(f"Error: {e}", get_last_fragment())
            self.log(f"Error: {e}")

    def on_plugin_unload(self) -> None:
        return
        if self.loader is not None:
            self.loader.unload()
