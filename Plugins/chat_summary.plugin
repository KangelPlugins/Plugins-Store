from typing import Any, List

from base_plugin import BasePlugin, MenuItemData, MenuItemType
from client_utils import (
    get_last_fragment, get_messages_controller,
    run_on_queue, send_request, PLUGINS_QUEUE,
)
from android_utils import run_on_ui_thread, log, OnClickListener, copy_to_clipboard
from ui.settings import Header, Divider, Selector, Input, Text
from ui.bulletin import BulletinHelper
from org.telegram.messenger import AndroidUtilities
from org.telegram.ui.ActionBar import Theme

__id__ = "chat_summary"
__name__ = "Chat Summary"
__description__ = "Summarize any chat using AI — OpenAI, Anthropic, Gemini, Ollama or a custom endpoint."
__author__ = "@aaxnet"
__version__ = "2.0.0"
__icon__ = "KOSHAKIEBANIYE/44"
__app_version__ = ">=12.5.1"
__sdk_version__ = ">=1.4.3.3"
__requirements__ = ["requests"]

PROVIDER_OPENAI = 0
PROVIDER_ANTHROPIC = 1
PROVIDER_GEMINI = 2
PROVIDER_OLLAMA = 3
PROVIDER_CUSTOM = 4

PROVIDERS = [
    "OpenAI (GPT-4o)",
    "Anthropic (Claude)",
    "Gemini",
    "Ollama (local)",
    "Custom endpoint",
]

PROVIDER_ICONS = [
    "msg_robot",
    "msg_robot",
    "msg_robot",
    "msg_devices",
    "msg_link",
]

DEFAULT_MODELS = [
    "gpt-4o",
    "claude-sonnet-4-5",
    "gemini-2.0-flash",
    "llama3",
    "",
]

STYLE_BRIEF = 0
STYLE_DETAILED = 1
STYLE_BULLETS = 2

STYLES = ["Краткий", "Подробный", "По пунктам"]
STYLE_HINTS = [
    "2–3 предложения о главном",
    "Все темы и детали",
    "Список ключевых моментов",
]

LANG_AUTO = 0
LANG_RU = 1
LANG_EN = 2

LANGS = ["Авто", "Русский", "English"]

MSG_COUNTS = [50, 100, 200, 500]

MAX_CONTENT_CHARS = 48000


def get_theme_color(key, fallback):
    try:
        return Theme.getColor(getattr(Theme, key))
    except Exception:
        return fallback


def build_system_prompt(style, lang):
    lang_instructions = {
        LANG_AUTO: "Respond in the same language as the conversation.",
        LANG_RU: "Respond in Russian.",
        LANG_EN: "Respond in English.",
    }
    style_instructions = {
        STYLE_BRIEF: "Write a concise 2-3 sentence summary of the key points.",
        STYLE_DETAILED: "Write a detailed summary covering all important topics discussed.",
        STYLE_BULLETS: "Write a bullet-point list of the main topics and decisions.",
    }
    lang_part = lang_instructions.get(lang, "")
    style_part = style_instructions.get(style, "")
    return (
        "You are a helpful assistant that summarizes Telegram chat conversations. "
        + style_part + " " + lang_part + " "
        "Focus only on content. Do not include meta-commentary."
    )


def parse_java_list(java_list):
    result = {}
    if not java_list:
        return result
    try:
        size = java_list.size()
        for i in range(size):
            item = java_list.get(i)
            try:
                result[int(item.id)] = item
            except Exception:
                pass
    except Exception:
        pass
    return result


def get_sender_name(raw_msg, user_map, chat_map):
    fid = getattr(raw_msg, "from_id", None)
    if fid is None:
        fid = getattr(raw_msg, "peer_id", None)
    if fid is None:
        return "Unknown"

    uid = getattr(fid, "user_id", None)
    channel_id = getattr(fid, "channel_id", None)
    chat_id = getattr(fid, "chat_id", None)

    if uid:
        uid = int(uid)
        user = user_map.get(uid)
        if user:
            first = getattr(user, "first_name", "") or ""
            last = getattr(user, "last_name", "") or ""
            name = (first + " " + last).strip()
            return name if name else "User" + str(uid)
        return "User" + str(uid)

    if channel_id:
        channel_id = int(channel_id)
        chat = chat_map.get(channel_id)
        if chat:
            return getattr(chat, "title", None) or "Channel" + str(channel_id)
        return "Channel" + str(channel_id)

    if chat_id:
        chat_id = int(chat_id)
        chat = chat_map.get(chat_id)
        if chat:
            return getattr(chat, "title", None) or "Chat" + str(chat_id)
        return "Chat" + str(chat_id)

    return "Unknown"


def get_message_text(msg, raw_msg):
    mt = getattr(msg, "messageText", None)
    if mt:
        return str(mt)
    raw_text = getattr(raw_msg, "message", None)
    if raw_text:
        return str(raw_text)
    return ""


def build_transcript(messages, users=None, chats=None):
    user_map = parse_java_list(users)
    chat_map = parse_java_list(chats)

    size = messages.size() if hasattr(messages, "size") else len(messages)
    lines = []
    used = 0

    for i in range(size - 1, -1, -1):
        try:
            msg = messages.get(i) if hasattr(messages, "get") else messages[i]
            raw = getattr(msg, "messageOwner", msg)

            cls = type(raw).__name__
            if "Service" in cls or "Empty" in cls:
                continue

            sender = get_sender_name(raw, user_map, chat_map)
            text = get_message_text(msg, raw)

            if text.strip():
                lines.append(sender + ": " + text)
                used += 1
        except Exception:
            continue

    full_text = "\n".join(lines)
    if len(full_text) > MAX_CONTENT_CHARS:
        full_text = "...[обрезано]\n" + full_text[-MAX_CONTENT_CHARS:]

    return full_text, used


def show_result(act, summary, msg_count):
    try:
        from ui.alert import AlertDialogBuilder

        def on_copy(b, w):
            copy_to_clipboard(summary)
            b.dismiss()

        builder = AlertDialogBuilder(act)
        builder.set_title("Саммари · " + str(msg_count) + " сообщений")
        builder.set_message(summary)
        builder.set_positive_button("Закрыть", lambda b, w: b.dismiss())
        builder.set_negative_button("Копировать", on_copy)
        builder.show()
    except Exception as e:
        log("chat_summary: show_result error: " + str(e))


def show_spinner(act, title="Анализирую…"):
    try:
        from ui.alert import AlertDialogBuilder
        d = AlertDialogBuilder(act, AlertDialogBuilder.ALERT_TYPE_SPINNER)
        d.set_title(title)
        d.set_cancelable(False)
        d.show()
        return d
    except Exception as e:
        log("chat_summary: spinner error: " + str(e))
        return None


def dismiss_dialog(dlg):
    if dlg is None:
        return
    try:
        dlg.dismiss()
    except Exception:
        pass


def call_openai(api_key, model, system, content, url=None):
    import requests
    model = model or "gpt-4o"
    url = url or "https://api.openai.com/v1/chat/completions"
    resp = requests.post(
        url,
        headers={
            "Authorization": "Bearer " + api_key,
            "Content-Type": "application/json",
        },
        json={
            "model": model,
            "messages": [
                {"role": "system", "content": system},
                {"role": "user", "content": content},
            ],
        },
        timeout=60,
    )
    resp.raise_for_status()
    return resp.json()["choices"][0]["message"]["content"]


def call_anthropic(api_key, model, system, content):
    import requests
    model = model or "claude-sonnet-4-5"
    resp = requests.post(
        "https://api.anthropic.com/v1/messages",
        headers={
            "x-api-key": api_key,
            "anthropic-version": "2023-06-01",
            "Content-Type": "application/json",
        },
        json={
            "model": model,
            "max_tokens": 1024,
            "system": system,
            "messages": [{"role": "user", "content": content}],
        },
        timeout=60,
    )
    resp.raise_for_status()
    return resp.json()["content"][0]["text"]


def call_gemini(api_key, model, system, content):
    import requests
    model = model or "gemini-2.0-flash"
    url = (
        "https://generativelanguage.googleapis.com/v1beta/models/"
        + model
        + ":generateContent?key="
        + api_key
    )
    resp = requests.post(
        url,
        json={"contents": [{"parts": [{"text": system + "\n\n" + content}]}]},
        timeout=60,
    )
    resp.raise_for_status()
    return resp.json()["candidates"][0]["content"]["parts"][0]["text"]


def call_ollama(model, system, content):
    import requests
    model = model or "llama3"
    resp = requests.post(
        "http://localhost:11434/api/chat",
        json={
            "model": model,
            "stream": False,
            "messages": [
                {"role": "system", "content": system},
                {"role": "user", "content": content},
            ],
        },
        timeout=120,
    )
    resp.raise_for_status()
    return resp.json()["message"]["content"]


def call_ai(provider, api_key, model, system, content, custom_url):
    if provider == PROVIDER_OPENAI:
        return call_openai(api_key, model, system, content)
    if provider == PROVIDER_ANTHROPIC:
        return call_anthropic(api_key, model, system, content)
    if provider == PROVIDER_GEMINI:
        return call_gemini(api_key, model, system, content)
    if provider == PROVIDER_OLLAMA:
        return call_ollama(model, system, content)
    if provider == PROVIDER_CUSTOM:
        return call_openai(api_key, model, system, content, url=custom_url)
    raise ValueError("Unknown provider: " + str(provider))


class ChatSummaryPlugin(BasePlugin):

    def on_plugin_load(self):
        self.add_menu_item(MenuItemData(
            menu_type=MenuItemType.CHAT_ACTION_MENU,
            text="Summarize Chat",
            subtext="Краткое AI-саммари",
            icon="msg_info",
            on_click=self.on_menu_click,
        ))

    def on_plugin_unload(self):
        pass

    def on_menu_click(self, context):
        dialog_id = context.get("dialog_id")
        if not dialog_id:
            return
        fragment = context.get("fragment") or get_last_fragment()
        if not fragment:
            return
        act = fragment.getParentActivity()
        if not act:
            return
        run_on_ui_thread(lambda: self.show_count_picker(act, dialog_id))

    def show_count_picker(self, act, dialog_id):
        try:
            from android.widget import LinearLayout, TextView
            from android.view import Gravity
            from android.util import TypedValue
            from android.graphics.drawable import GradientDrawable
            from org.telegram.ui.ActionBar import BottomSheet

            dp = AndroidUtilities.dp
            sheet_holder = [None]

            def pick(count):
                if sheet_holder[0]:
                    sheet_holder[0].dismiss()
                BulletinHelper.show_info("Загружаю " + str(count) + " сообщений…")
                run_on_queue(
                    lambda: self.fetch_and_summarize(act, dialog_id, count),
                    PLUGINS_QUEUE, 0,
                )

            root = LinearLayout(act)
            root.setOrientation(LinearLayout.VERTICAL)
            root.setPadding(dp(20), dp(16), dp(20), dp(24))
            try:
                root.setBackgroundColor(Theme.getColor(Theme.key_dialogBackground))
            except Exception:
                pass

            title = TextView(act)
            title.setText("Сколько сообщений проанализировать?")
            title.setTextSize(TypedValue.COMPLEX_UNIT_DIP, 16)
            title.setGravity(Gravity.CENTER)
            title.setTextColor(get_theme_color("key_dialogTextBlack", 0xFF212121))
            try:
                title.setTypeface(AndroidUtilities.bold())
            except Exception:
                pass
            lp = LinearLayout.LayoutParams(-1, -2)
            lp.bottomMargin = dp(16)
            root.addView(title, lp)

            for count in MSG_COUNTS:
                btn = TextView(act)
                btn.setText(str(count) + " сообщений")
                btn.setTextSize(TypedValue.COMPLEX_UNIT_DIP, 15)
                btn.setGravity(Gravity.CENTER)
                btn.setPadding(dp(16), dp(14), dp(16), dp(14))
                btn.setClickable(True)
                btn.setFocusable(True)
                btn.setTextColor(get_theme_color("key_windowBackgroundWhiteBlackText", 0xFF212121))

                bg = GradientDrawable()
                bg.setCornerRadius(dp(12))
                bg.setColor(get_theme_color("key_windowBackgroundWhite", 0xFFF5F5F5))
                btn.setBackground(bg)

                n = count
                btn.setOnClickListener(OnClickListener(lambda v, c=n: pick(c)))

                lp = LinearLayout.LayoutParams(-1, -2)
                lp.bottomMargin = dp(8)
                root.addView(btn, lp)

            sheet = BottomSheet.Builder(act).setCustomView(root).create()
            sheet_holder[0] = sheet
            sheet.show()

        except Exception as e:
            log("chat_summary: count picker error: " + str(e))

    def fetch_and_summarize(self, act, dialog_id, count):
        try:
            from org.telegram.tgnet import TLRPC

            req = TLRPC.TL_messages_getHistory()
            req.peer = get_messages_controller().getInputPeer(int(dialog_id))
            req.offset_id = 0
            req.offset_date = 0
            req.add_offset = 0
            req.limit = count
            req.max_id = 0
            req.min_id = 0
            req.hash = 0

            self_ref = [self]
            act_ref = [act]

            def on_response(response, error):
                if error:
                    err = getattr(error, "text", "unknown")
                    run_on_ui_thread(lambda: BulletinHelper.show_error("Ошибка загрузки: " + err))
                    return
                try:
                    msgs = getattr(response, "messages", None)
                    if not msgs:
                        run_on_ui_thread(lambda: BulletinHelper.show_error("Нет сообщений."))
                        return

                    users = getattr(response, "users", None)
                    chats = getattr(response, "chats", None)
                    text, used = build_transcript(msgs, users, chats)

                    if not text.strip():
                        run_on_ui_thread(lambda: BulletinHelper.show_error("Текстовых сообщений не найдено."))
                        return

                    spinner = [None]

                    def start_spinner():
                        spinner[0] = show_spinner(act_ref[0])

                    run_on_ui_thread(start_spinner)
                    run_on_queue(
                        lambda: self_ref[0].run_ai(act_ref[0], text, used, spinner),
                        PLUGINS_QUEUE, 0,
                    )
                except Exception as ex:
                    log("chat_summary: parse error: " + str(ex))
                    run_on_ui_thread(lambda: BulletinHelper.show_error("Ошибка парсинга сообщений."))

            send_request(req, on_response)

        except Exception as e:
            log("chat_summary: fetch error: " + str(e))
            run_on_ui_thread(lambda: BulletinHelper.show_error("Не удалось начать загрузку."))

    def run_ai(self, act, text, used, spinner):
        try:
            provider = self.get_setting("provider", PROVIDER_OPENAI)
            api_key = self.get_setting("api_key", "") or ""
            model = self.get_setting("model", "") or ""
            style = self.get_setting("style", STYLE_BRIEF)
            lang = self.get_setting("lang", LANG_AUTO)
            custom_url = self.get_setting("custom_endpoint", "") or ""

            system = build_system_prompt(style, lang)
            content = "Chat conversation to summarize:\n\n" + text

            result = call_ai(provider, api_key, model, system, content, custom_url)

            def on_done():
                dismiss_dialog(spinner[0])
                show_result(act, result, used)

            run_on_ui_thread(on_done)

        except Exception as e:
            err = str(e)
            log("chat_summary: ai error: " + err)

            def on_error():
                dismiss_dialog(spinner[0])
                BulletinHelper.show_error("Ошибка AI: " + err[:120])

            run_on_ui_thread(on_error)

    def create_settings(self):
        provider = self.get_setting("provider", PROVIDER_OPENAI)
        style = self.get_setting("style", STYLE_BRIEF)
        default_model = DEFAULT_MODELS[provider] if 0 <= provider < len(DEFAULT_MODELS) else ""
        style_hint = STYLE_HINTS[style] if 0 <= style < len(STYLE_HINTS) else ""

        items = [
            Header(text="AI провайдер"),
            Selector(
                key="provider",
                text="Провайдер",
                default=PROVIDER_OPENAI,
                items=PROVIDERS,
                icon="msg_robot",
                on_change=lambda idx: self.set_setting("provider", idx, reload_settings=True),
            ),
            Input(
                key="api_key",
                text="API ключ",
                subtext="Не нужен для Ollama",
                icon="msg_secret",
                default="",
            ),
            Input(
                key="model",
                text="Модель",
                subtext="По умолчанию: " + default_model if default_model else "Укажи название модели",
                icon="msg_topic",
                default="",
            ),
        ]

        if provider == PROVIDER_CUSTOM:
            items.append(Input(
                key="custom_endpoint",
                text="URL endpoint",
                subtext="OpenAI-совместимый адрес",
                icon="msg_link",
                default="",
            ))

        items += [
            Divider(),
            Header(text="Саммари"),
            Selector(
                key="style",
                text="Стиль",
                default=STYLE_BRIEF,
                items=STYLES,
                icon="msg_list",
                on_change=lambda idx: self.set_setting("style", idx, reload_settings=True),
            ),
            Text(
                text=style_hint,
                icon="msg_info",
            ),
            Selector(
                key="lang",
                text="Язык ответа",
                default=LANG_AUTO,
                items=LANGS,
                icon="msg_translate",
            ),
        ]

        return items
