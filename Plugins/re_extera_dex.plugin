__id__ = "re_extera_dex"

__name__ = "re:extera v2"

__description__ = "Enable ghost mode, save deleted messages and more!"

__author__ = "@shikaatux | @shikaatuxplugins \noriginal author: @bleizixPlugins"

__version__ = "2.3"

__icon__ = "myadestes_1_amashiro_natsuki_plus_nacho_neko/30"

__min_version__ = "12.2.3"

from typing import Any
from base_plugin import BasePlugin, MenuItemData, MenuItemType
from client_utils import get_last_fragment
import requests
from android.app import Activity
from hook_utils import find_class
from ui.bulletin import BulletinHelper
from ui.settings import EditText, Text, Switch
from org.telegram.ui import LaunchActivity
from org.telegram.messenger import LocaleController
from java.nio import ByteBuffer
from dalvik.system import InMemoryDexClassLoader
from org.telegram.messenger import ApplicationLoader
import os

DEFAULT_URL = "https://github.com/logopek/re-extera-pub/raw/refs/heads/main/re-extera.1.3.dex"
VERSION_URL = "https://github.com/logopek/re-extera-pub/raw/refs/heads/main/actual.txt"
CLASS_NAME = "ni.shikatu.re_extera.Main"
METHOD_NAME = "start"


class Loader:
    def __init__(self, plugin: BasePlugin, activity: Activity):
        self.plugin = plugin
        self.activity = activity
        self.instance = None
        self.cache_dir = os.path.join(ApplicationLoader.applicationContext.getFilesDir().getAbsolutePath(),
                                      "re_extera_cache")
        self.cache_file = os.path.join(self.cache_dir, "cached.dat")
        self.download_url = DEFAULT_URL  # URL для скачивания

        if not os.path.exists(self.cache_dir):
            os.makedirs(self.cache_dir)

    def get_cached_version(self):
        try:
            if self.instance is None:
                return 0
            version_field = self.instance.getClass().getDeclaredField("VERSION_CODE")
            version_field.setAccessible(True)
            return int(version_field.get(None))
        except Exception as e:
            self.plugin.log(f"Error getting cached version: {e}")
            return 0

    def check_for_updates(self):
        try:
            self.plugin.log("Checking for updates...")
            r = requests.get(VERSION_URL, timeout=5)
            r.raise_for_status()

            lines = r.text.strip().split('\n')

            remote_version = int(lines[0].strip())

            if len(lines) >= 2:
                self.download_url = lines[1].strip()
                self.plugin.log(f"Download URL from actual.txt: {self.download_url}")
            else:
                self.download_url = DEFAULT_URL
                self.plugin.log(f"Using default URL: {self.download_url}")

            current_version = self.get_cached_version()
            self.plugin.log(f"Remote version: {remote_version}, Current version: {current_version}")

            if remote_version > current_version:
                self.plugin.log("Update available!")
                return True
            else:
                self.plugin.log("Already up to date")
                return False
        except Exception as e:
            self.plugin.log(f"Error checking updates: {e}")
            self.download_url = DEFAULT_URL
            return False

    def download_and_cache_dex(self):
        try:
            self.plugin.log(f"Downloading from {self.download_url}")
            r = requests.get(self.download_url, timeout=30)
            r.raise_for_status()
            dex_bytes = r.content

            with open(self.cache_file, 'wb') as f:
                f.write(dex_bytes)

            self.plugin.log(f"Cached to {self.cache_file}")
            return dex_bytes
        except Exception as e:
            self.plugin.log(f"Error downloading: {e}")
            raise

    def load_from_cache(self):
        try:
            if os.path.exists(self.cache_file):
                with open(self.cache_file, 'rb') as f:
                    return f.read()
            return None
        except Exception as e:
            self.plugin.log(f"Error loading from cache: {e}")
            return None

    def getInstance(self):
        if self.instance is None:
            try:
                method = find_class("ni.shikatu.re_extera.Main").getClass().getMethod("getInstance")
                self.instance = method.invoke(None)
            except Exception as e:
                self.plugin.log(f"Error getting instance: {e}")
        return self.instance

    def open_settings(self):
        self.plugin.log("Opening settings")
        try:
            method = self.getInstance().getClass().getMethod("showSettings")
            method.invoke(self.getInstance())
        except Exception as e:
            self.plugin.log(f"Error: {e}")

    def unload(self):
        try:
            method = self.getInstance().getClass().getMethod("onUnload")
            method.invoke(self.getInstance())
            self.plugin.log("Success unload")
        except Exception as e:
            self.plugin.log(f"Error: {e}")

    def start_from_bytes(self, bytesdex):
        try:
            buffer = ByteBuffer.wrap(bytesdex)
            dex_loader = InMemoryDexClassLoader(buffer, ApplicationLoader.applicationContext.getClassLoader())
            clazz = dex_loader.loadClass(CLASS_NAME)
            start_method = clazz.getMethod(METHOD_NAME)
            instance_method = clazz.getMethod("getInstance")
            instance = instance_method.invoke(None)
            self.instance = instance
            start_method.invoke(instance)
            self.plugin.log(f"Loaded {CLASS_NAME}")
            self.plugin.log(f"Method {METHOD_NAME} invoked on {CLASS_NAME}")
        except Exception as e:
            self.plugin.log(f"Error: {e}")
            raise

    def load_and_start(self):
        try:
            cached_bytes = self.load_from_cache()

            if cached_bytes is not None:
                self.plugin.log("Loading from cache")
                self.start_from_bytes(cached_bytes)

                try:
                    if self.check_for_updates():
                        self.plugin.log("Downloading new version...")
                        self.download_and_cache_dex()

                        if LocaleController.getInstance().getCurrentLocale().getLanguage() == "ru":
                            message = "Доступна новая версия re:extera! Перезапустите приложение для применения обновления."
                        else:
                            message = "New re:extera version available! Restart the app to apply the update."

                        BulletinHelper.show_info(message, get_last_fragment())
                except Exception as e:
                    self.plugin.log(f"Error during update check: {e}")
            else:
                self.plugin.log("No cache found, downloading...")
                try:
                    r = requests.get(VERSION_URL, timeout=5)
                    r.raise_for_status()
                    lines = r.text.strip().split('\n')
                    if len(lines) >= 2:
                        self.download_url = lines[1].strip()
                        self.plugin.log(f"Initial download URL: {self.download_url}")
                except Exception as e:
                    self.plugin.log(f"Error getting URL from actual.txt: {e}, using default")
                    self.download_url = DEFAULT_URL

                dex_bytes = self.download_and_cache_dex()
                self.start_from_bytes(dex_bytes)

        except Exception as e:
            if "proxy" in str(e):
                return
            self.plugin.log(f"Fatal error: {e}")
            BulletinHelper.show_info(f"Error loading re:extera: {e}", get_last_fragment())


class Plugin(BasePlugin):
    def __init__(self):
        self.loader = None

    def localizedOpenSettings(self):
        if LocaleController.getInstance().getCurrentLocale().getLanguage() == "ru":
            return "Настройки re:extera"
        return "re:extera Settings"

    def on_plugin_load(self) -> None:
        try:
            self.log(f"Init {__version__}")
            self.create_settings()
            self.load()
        except Exception as e:
            BulletinHelper.show_info(f"Error: {e}", get_last_fragment())
            self.log(f"Error: {e}")

    def create_settings(self) -> list[Any]:
        return [
            Text(text=self.localizedOpenSettings(), on_click=lambda v: self.open_settings(None))
        ]

    def on_plugin_unload(self) -> None:
        if self.loader is not None:
            self.loader.unload()

    def open_settings(self, context):
        if self.loader is not None:
            self.loader.open_settings()

    def load(self):
        self.log("Loading")
        launchActivity: LaunchActivity = get_last_fragment().getContext()
        self.loader = Loader(self, launchActivity)
        self.loader.load_and_start()
