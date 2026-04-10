import json
import os
from operator import ge

from android_utils import log
from base_plugin import BasePlugin, MethodHook
from debugpy.adapter.sessions import get
from hook_utils import find_class
from java import jclass
from ui.bulletin import BulletinHelper

__id__ = "favstickers"
__name__ = "Unlim favorite stickers"
__description__ = "Remove limits on adding stickers to favorites"
__author__ = "@DaShMore"
__version__ = "2.0.0"
__icon__ = "plugins_covers/0"
__min_version__ = "11.12.0"


class Jclass:
    def __getattr__(self, name: str):
        return jclass(f"java.lang.{name.title()}")


J = Jclass()
TLRPC = jclass("org.telegram.tgnet.TLRPC")
ArrayList = jclass("java.util.ArrayList")


def serialize_sticker(sticker) -> dict[str, str | int]:
    # Иногда обертка стикера содержит .document
    if hasattr(sticker, "document") and sticker.document is not None:
        sticker = sticker.document

    data = {
        "id": int(sticker.id),
        "access_hash": int(sticker.access_hash),
        "dc_id": int(sticker.dc_id),
        "mime_type": str(sticker.mime_type),
    }

    if hasattr(sticker, "attributes") and sticker.attributes is not None:
        attr_list = sticker.attributes
        for i in range(attr_list.size()):
            att = attr_list.get(i)

            if hasattr(att, "stickerset") and att.stickerset is not None:
                data["sticker_set_id"] = getattr(att.stickerset, "id")
    return data


def deserialize_sticker(data: dict):
    """Десериализует словарь в телеграм стикер"""
    doc = TLRPC.TL_document()

    doc.id = data["id"]
    doc.access_hash = data["access_hash"]
    doc.dc_id = data["dc_id"]
    doc.mime_type = data["mime_type"]
    # FIXME
    doc.attributes = ArrayList()

    a = TLRPC.TL_documentAttributeSticker()
    ss = TLRPC.TL_inputStickerSetID()
    ss.id = data["sticker_set_id"]
    a.stickerset = ss
    doc.attributes.add(a)

    return doc


class StickersDB:
    """
    Класс для работы с базой данных стикеров

    {
        "accounts": {
            <account_id>: [
                <sticker_id>,
                <sticker_id>,
            ]
        },
        "stickers": {
            <sticker_id>: {
                "id": <sticker_id>,
                "access_hash": <access_hash>,
                "dc_id": <dc_id>,
                "mime_type": <mime_type>,
                "sticker_set_id": <sticker_set_id>
            }
        }
    }
    """

    # HACK все ключи при сохранении становятся строками, так что иногда надо приводить их вручную через str()
    def __init__(self, db_path: str):
        self.__db_path = db_path
        self.__load_db()

    def __load_db(self):
        """Чтение всех стикеров из базы в self.stickers"""
        if not os.path.exists(self.__db_path):
            self.__stickers = {"accounts": {}, "stickers": {}}
        else:
            with open(self.__db_path) as f:
                self.__stickers = json.load(f)

    def __save_db(self):
        """Сохранение всех стикеров из self.stickers в базу"""
        log(self.__stickers["accounts"])
        with open(self.__db_path, "w") as f:
            json.dump(self.__stickers, f)

    def get_all_stickers(self, account: int) -> list[dict[str, str | int]]:
        """Получение всех стикеров в виде объектов TLRPC$TL_document"""
        return [
            deserialize_sticker(self.__stickers["stickers"][i])
            for i in reversed(self.__stickers["accounts"].get(account, []))
        ]

    def add_sticker(self, sticker, account: int):
        """Сериализация и добавление стикера в базу без дубликатов"""
        serialized_sticker = serialize_sticker(sticker)
        sticker_id = str(serialized_sticker["id"])
        if sticker_id not in self.__stickers["stickers"]:
            self.__stickers["stickers"][sticker_id] = serialized_sticker
        if account not in self.__stickers["accounts"]:
            self.__stickers["accounts"][account] = []
        if sticker_id not in self.__stickers["accounts"][account]:
            self.__stickers["accounts"][account].append(sticker_id)
            self.__save_db()

    def remove_sticker(self, sticker, account: int):
        """Удаление стикера из базы и self.stickers."""
        serialized_sticker = serialize_sticker(sticker)
        sticker_id = str(serialized_sticker["id"])
        if sticker_id in self.__stickers["accounts"][account]:
            self.__stickers["accounts"][account].remove(sticker_id)
            self.__save_db()
        # Если ни у кого стикер не сохранен - удаляем из общего списка
        if sticker_id in self.__stickers["stickers"] and not any(
            sticker_id in i for i in self.__stickers["accounts"].values()
        ):
            del self.__stickers["stickers"][sticker_id]
            self.__save_db()

    def is_sticker_favorite(self, sticker, account: int):
        """Проверка, есть ли стикер в избранных"""
        serialized_sticker = serialize_sticker(sticker)
        return str(serialized_sticker["id"]) in self.__stickers["accounts"][account]


class ChangeFavoriteStickerHook(MethodHook):
    def __init__(self, on_add_favorite, on_remove_favorite, on_update, get_account_id):
        self.__on_add_favorite = on_add_favorite
        self.__on_remove_favorite = on_remove_favorite
        self.__on_update = on_update
        self.__get_account_id = get_account_id

    def before_hooked_method(self, param):
        sticker = param.args[2]
        account_id = self.__get_account_id()
        # Проверяем, что выбран пункт добавления / удаления из избранное (TYPE_FAVE = 2)
        # И что стикер не находится в избранном (not inFavs)
        if param.args[0] == 2 and not param.args[4]:
            self.__on_add_favorite(sticker, account_id)
            BulletinHelper.show_success("Sticker added to favorites")
        elif param.args[0] == 2:
            self.__on_remove_favorite(sticker, account_id)
            BulletinHelper.show_error("Sticker removed from favorites")
        self.__on_update()
        param.setResult(None)


class GetFavoriteStickersHook(MethodHook):
    def __init__(self, get_favorite_stickers, get_account_id):
        self.__get_favorite_stickers = get_favorite_stickers
        self.__get_account_id = get_account_id

    def after_hooked_method(self, param):
        account = self.__get_account_id()
        favorite_stickers = self.__get_favorite_stickers(account)
        if not favorite_stickers:
            return
        new_list = jclass("java.util.ArrayList")()
        for sticker in favorite_stickers:
            new_list.add(sticker)
        param.setResult(new_list)


class IsStickerInFavoritesHook(MethodHook):
    def __init__(self, is_favorite_sticker, get_account_id):
        self.__is_favorite_sticker = is_favorite_sticker
        self.__get_account_id = get_account_id

    def before_hooked_method(self, param):
        sticker = param.args[0]
        account = self.__get_account_id()
        param.setResult(self.__is_favorite_sticker(sticker, account))


class MyPlugin(BasePlugin):
    __DB = None

    def __get_context(self):
        """
        Возвращает контекст приложения
        """
        current_app = jclass("android.app.ActivityThread").currentApplication()
        if not current_app:
            RuntimeError("app not find")
        return current_app

    @property
    def db(self):
        if self.__DB is None:
            self.__DB = StickersDB(
                os.path.join(str(self.__get_context().getFilesDir()), "stickers.json")
            )
        return self.__DB

    @staticmethod
    def __get_current_account() -> int:
        UserConfig = find_class("org.telegram.messenger.UserConfig")
        # currentAccount — это индекс аккаунта (обычно 0)
        return UserConfig.selectedAccount

    @staticmethod
    def __get_current_account_id() -> str:
        UserConfig = find_class("org.telegram.messenger.UserConfig")
        user_id = UserConfig.getInstance(UserConfig.selectedAccount).clientUserId
        return str(user_id)

    def __load_favorite_stickers(self, mediaController):
        account = self.__get_current_account_id()
        if not self.db.get_all_stickers(account):
            stickers = mediaController.getRecentStickers(2)
            for sticker in range(stickers.size() - 1, -1, -1):
                self.db.add_sticker(stickers.get(sticker), account)

    def on_plugin_load(self):
        MediaController = find_class("org.telegram.messenger.MediaDataController")
        TLRPCDocument = find_class("org.telegram.tgnet.TLRPC$Document")
        media_instance = MediaController.getInstance(self.__get_current_account())
        media_class = media_instance.getClass()

        # Перехват метода добавления стикера в избраные
        addRecentStickerMethod = media_class.getDeclaredMethod(
            "addRecentSticker",
            J.Integer.TYPE,
            J.Object,
            TLRPCDocument,
            J.Integer.TYPE,
            J.Boolean.TYPE,
        )
        addRecentStickerMethod.setAccessible(True)
        self.hook_method(
            addRecentStickerMethod,
            ChangeFavoriteStickerHook(
                on_add_favorite=self.db.add_sticker,
                on_remove_favorite=self.db.remove_sticker,
                on_update=media_instance.processLoadedRecentDocuments,
                get_account_id=self.__get_current_account_id,
            ),
        )

        # Перехват метода получения списка избранных стикеров
        getRecentStickersMethod = media_class.getDeclaredMethod(
            "getRecentStickers",
            J.Integer.TYPE,
        )
        getRecentStickersMethod.setAccessible(True)
        self.unhook_obj = self.hook_method(
            getRecentStickersMethod,
            GetFavoriteStickersHook(
                self.db.get_all_stickers, self.__get_current_account_id
            ),
        )

        # Перехват метода проверки наличия стикера в избранных
        isStickerInFavoritesMethod = media_class.getDeclaredMethod(
            "isStickerInFavorites", jclass("org.telegram.tgnet.TLRPC$Document")
        )
        isStickerInFavoritesMethod.setAccessible(True)
        self.unhook_obj = self.hook_method(
            isStickerInFavoritesMethod,
            IsStickerInFavoritesHook(
                self.db.is_sticker_favorite, self.__get_current_account_id
            ),
        )

        self.__load_favorite_stickers(media_instance)
