import time
import threading
import zipfile
import json
from datetime import datetime
from typing import Any, Dict, List, Set

from base_plugin import BasePlugin, HookResult, HookStrategy
from client_utils import get_account_instance, get_last_fragment, run_on_queue
from android_utils import log, run_on_ui_thread, OnClickListener
from ui.alert import AlertDialogBuilder
from ui.settings import Header, Switch, Selector, Divider, Input, Text
from ui.bulletin import BulletinHelper
from org.telegram.messenger import ApplicationLoader, SendMessagesHelper, AndroidUtilities
from org.telegram.ui.Components import EditTextBoldCursor
from org.telegram.ui.ActionBar import Theme
from android.text import InputType
from android.util import TypedValue
from android.widget import LinearLayout, TextView, CheckBox
from java.io import File

__id__ = "compact_text"
__name__ = "Compact Text"
__description__ = "🇷🇺 | Объединяет длинные сообщения в один файл (порог длины и тип файла (по умолчанию .TXT) можно изменить в настройках плагина) \n\n🇺🇸 | Merges long messages into a single file (the length threshold and file type (.TXT by default) can be changed in the plugin settings)."
__author__ = "@incLu_01 & @buligaEplugins"
__version__ = "1.0.5"
__icon__ = "BuligaPlugins/0"
__min_version__ = "11.9.0"

DEFAULT_THRESHOLD = 5000
COLLECT_DELAY = 0.05
TEMP_DIR_NAME = "CompactText"
TELEGRAM_MAX_CHARS = 4096

FORMATTING_LEGEND = """Форматирование текста:
**Жирный**
*Курсив*
_Подчёркнутый_
*_Жирный курсив_*
~Зачеркнутый~
    > Цитата
`Моноширный`
||Скрытый||
```код\nтекст```
"""


class CompactText(BasePlugin):
    def __init__(self):
        super().__init__()
        self.message_parts: Dict[int, List[str]] = {}
        self.collect_timers: Dict[int, threading.Timer] = {}
        self.params_cache: Dict[int, Any] = {}
        self._busy_peers = set()
        self._bypass_peers = set()

    def on_plugin_load(self):
        self.add_on_send_message_hook(priority=1000)

    def _on_threshold_change(self, value: str):
        try:
            threshold = int(str(value).strip())
        except Exception:
            threshold = DEFAULT_THRESHOLD
        threshold = max(100, min(500000, threshold))
        self.set_setting("threshold", str(threshold))

    def create_settings(self):
        items = [
            Header(text="⚙️ Настройки Compact Text"),
            Switch(key="show_dialog", text="Диалог переименования", default=True, subtext="Показывать окно для редактирования имени файла перед отправкой"),
            Divider(),
            Header(text="📊 Порог длины сообщения"),
            Input(
                key="threshold",
                text="Порог символов (100-500000)",
                default=str(self._get_threshold()),
                subtext="Сообщения длиннее этого значения будут упакованы в файл. Минимум: 100, Максимум: 500000",
                on_change=self._on_threshold_change
            ),
            Divider(),
            Header(text="📄 Формат файла"),
            Selector(key="file_format", text="Тип файла", default=0, items=["TXT", "DOCX", "Custom"]),
        ]
        fmt = self._get_int_setting("file_format", 0)
        if fmt == 2:
            items.append(Input(key="custom_ext", text="Своё расширение (без точки)", default="txt"))
        items.extend([
            Divider(),
            Header(text="✉️ Разделение при отправке сообщениями"),
            Selector(key="split_mode", text="Режим разбиения", default=0, items=["По правилам Telegram", "По порогу"]),
        ])
        items.append(Divider())
        items.append(Header(text="📝 Заготовки расширений"))
        items.append(Text(
            text="Просмотр заготовок",
            icon="msg_text",
            accent=True,
            on_click=lambda v: self._open_presets_dialog()
        ))
        items.extend([
            Divider(),
            Header(text="📏 Отображение размера"),
            Selector(key="size_unit", text="Единица измерения", default=0, items=["Килобайты (КБ)", "Мегабайты (МБ)"]),
            Divider(),
            Header(text="🕐 Формат времени в имени файла"),
            Switch(key="include_sec", text="Включить секунды", default=True, subtext="[ЧЧ.ММ.СС] вместо [ЧЧ.ММ]")
        ])
        if self._get_bool_setting("include_sec", True):
            items.append(Switch(key="include_ms", text="Включить миллисекунды", default=False, subtext="[ЧЧ.ММ.СС.МС]"))
        
        items.extend([
            Divider(),
            Header(text="📝 Форматирование текста"),
            Switch(key="apply_formatting", text="Учитывать форматирование", default=True, subtext="Переносить оформление (жирный, курсив, цитаты и т.д.) в файлы и при отправке сообщениями"),
            Switch(key="format_skip_enabled", text="Отключать формат. выше порога", default=False, subtext="Ускоряет упаковку очень длинных сообщений", on_change=lambda v: self._trigger_settings_reload()),
        ])
        if self._get_bool_setting("format_skip_enabled", False):
            items.append(Input(key="format_skip_threshold", text="Порог символов (≥100)", default=str(max(100, self._get_int_setting("format_skip_threshold", 20000))), on_change=self._on_format_skip_threshold_change))
        items.extend([
            Divider(),
            Switch(key="format_use_all", text="Использовать все форматирования", default=True, on_change=lambda v: self._trigger_settings_reload()),
        ])
        if not self._get_bool_setting("format_use_all", True):
            items.extend([
                Switch(key="fmt_bold", text="Жирный", default=True),
                Switch(key="fmt_italic", text="Курсив", default=True),
                Switch(key="fmt_underline", text="Подчёркнутый", default=True),
                Switch(key="fmt_strike", text="Зачеркнутый", default=True),
                Switch(key="fmt_code", text="Моноширинный", default=True),
                Switch(key="fmt_pre", text="Код-блок", default=True),
                Switch(key="fmt_quote", text="Цитата", default=True),
                Switch(key="fmt_link", text="Ссылка", default=True),
                Switch(key="fmt_spoiler", text="Скрытый", default=True),
            ])
        items.extend([
            Text(
                text="Показать список форматирования",
                icon="msg_help",
                on_click=lambda v: self._show_formatting_legend()
            ),
            Switch(key="insert_legend", text="Вставлять легенду в начало файлов", default=False, subtext="Добавляет список форматирования в начало каждого файла"),
            Divider(),
            Header(text="⚙️ Команды"),
            Switch(key="enable_cpt_command", text="Команда .cpt", default=True, subtext="Позволяет принудительно свернуть текст в файл БЕЗ форматирования. Используйте обычную отправку по порогу для сохранения форматирования"),
        ])
        if self._get_bool_setting("enable_cpt_command", True):
            items.extend([
                Switch(key="cpt_activator_all", text="Активатор команды", default=True, subtext="Если включено — команда реагирует на ! . /; если выключено — выберите префиксы ниже", on_change=lambda v: self._trigger_settings_reload()),
            ])
            if not self._get_bool_setting("cpt_activator_all", True):
                items.extend([
                    Switch(key="cpt_use_excl", text="Использовать !", default=True),
                    Switch(key="cpt_use_dot", text="Использовать .", default=True),
                    Switch(key="cpt_use_slash", text="Использовать /", default=True),
                ])
        items.extend([
            Divider(),
            Text(
                text="Помощь",
                icon="msg_help",
                accent=True,
                on_click=lambda v: self._show_help_dialog()
            )
        ])
        return items

    def on_send_message_hook(self, account: int, params: Any) -> HookResult:
        if not hasattr(params, "message"):
            return HookResult()
        try:
            message = str(getattr(params, "message", "") or "")
        except Exception:
            return HookResult()
        if not message:
            return HookResult()
        
        force_file = False
        if self._get_bool_setting("enable_cpt_command", True):
            msg = message
            lead = 0
            try:
                lead = len(msg) - len(msg.lstrip())
                msg = msg.lstrip()
            except Exception:
                pass
            if self._get_bool_setting("cpt_activator_all", True):
                allowed_symbols = ["!", ".", "/"]
            else:
                allowed_symbols = []
                if self._get_bool_setting("cpt_use_excl", True):
                    allowed_symbols.append("!")
                if self._get_bool_setting("cpt_use_dot", True):
                    allowed_symbols.append(".")
                if self._get_bool_setting("cpt_use_slash", True):
                    allowed_symbols.append("/")
            prefix = None
            for sym in allowed_symbols:
                cmd = sym + "cpt"
                if msg.startswith(cmd):
                    prefix = cmd
                    break
            if prefix:
                rest = msg[len(prefix):]
                if not rest:
                    return HookResult(strategy=HookStrategy.CANCEL)
                sep_trimmed = rest.lstrip()
                removed = lead + len(prefix) + (len(rest) - len(sep_trimmed))
                if sep_trimmed:
                    force_file = True
                    try:
                        setattr(params, "_force_file", True)
                    except Exception:
                        pass
                    try:
                        setattr(params, "_cpt_removed_prefix_len", removed)
                    except Exception:
                        pass
                    # Don't modify params.message - keep the clean text for file only
                    message = sep_trimmed
                else:
                    return HookResult(strategy=HookStrategy.CANCEL)
        
        peer_id = getattr(params, "peer", None)
        if peer_id is None:
            return HookResult()
        if peer_id in self._bypass_peers:
            return HookResult()
        
        threshold = self._get_threshold()
        if peer_id in self._busy_peers:
            if len(message) >= threshold or force_file:
                return HookResult(strategy=HookStrategy.CANCEL)
            return HookResult()
        
        if peer_id in self.message_parts and not force_file:
            self.message_parts[peer_id].append(message)
            self.params_cache[peer_id] = params
            self._restart_timer(peer_id)
            return HookResult(strategy=HookStrategy.CANCEL)
        if len(message) >= threshold or force_file:
            self._cleanup_peer(peer_id)
            self.params_cache[peer_id] = params
            run_on_queue(lambda: self._prepare_and_send_file(message, params, peer_id))
            return HookResult(strategy=HookStrategy.CANCEL)
        return HookResult()

    def _restart_timer(self, peer_id: int):
        if peer_id in self.collect_timers:
            self.collect_timers[peer_id].cancel()
        t = threading.Timer(COLLECT_DELAY, self._send_collected_parts, args=(peer_id,))
        self.collect_timers[peer_id] = t
        t.start()

    def _send_collected_parts(self, peer_id: int):
        if peer_id in getattr(self, "_busy_peers", set()):
            self._cleanup_peer(peer_id)
            return
        if peer_id not in self.message_parts:
            return
        parts = self.message_parts.get(peer_id)
        params = self.params_cache.get(peer_id)
        if not params or not parts:
            self._cleanup_peer(peer_id)
            return
        full_text = "".join(parts)
        self._cleanup_peer(peer_id)
        run_on_queue(lambda: self._prepare_and_send_file(full_text, params, peer_id))

    def _cleanup_peer(self, peer_id: int):
        self.message_parts.pop(peer_id, None)
        self.params_cache.pop(peer_id, None)
        if peer_id in self.collect_timers:
            try:
                self.collect_timers[peer_id].cancel()
            except Exception:
                pass
            del self.collect_timers[peer_id]

    def _prepare_and_send_file(self, text: str, params: Any, peer_id: int):
        try:
            if peer_id in self._busy_peers:
                return
            self._busy_peers.add(peer_id)
            default_filename = self._generate_filename(peer_id)
            show_dlg = self._get_bool_setting("show_dialog", True)
            if show_dlg:
                run_on_ui_thread(lambda: self._show_filename_dialog(text, params, default_filename))
            else:
                self._create_and_send_file(text, params, default_filename)
        except Exception:
            try:
                self._busy_peers.discard(peer_id)
            except Exception:
                pass

    def _generate_filename(self, peer_id: int) -> str:
        now = datetime.now()
        if peer_id < 0:
            cid = str(abs(peer_id))
            if cid.startswith("100"):
                cid = cid[3:]
        else:
            cid = str(peer_id)
        date_str = now.strftime("%d.%m.%Y")
        include_sec = self._get_bool_setting("include_sec", True)
        include_ms = self._get_bool_setting("include_ms", False) if include_sec else False
        if include_ms and include_sec:
            ms = int(now.microsecond / 1000)
            time_str = now.strftime("%H.%M.%S") + f".{ms:03d}"
        elif include_sec:
            time_str = now.strftime("%H.%M.%S")
        else:
            time_str = now.strftime("%H.%M")
        fmt = self._get_int_setting("file_format", 0)
        if fmt == 0:
            ext = "txt"
        elif fmt == 1:
            ext = "docx"
        else:
            ext = str(self.get_setting("custom_ext", "txt")).strip().lstrip(".") or "txt"
        return f"{cid}_{date_str}_{time_str}.{ext}"

    def _show_filename_dialog(self, text: str, params: Any, default_filename: str):
        try:
            log("[CompactText] showing dialog...")
            
            frag = get_last_fragment()
            
            if not frag:
                log("[CompactText] no fragment after 3 attempts, sending without dialog")
                run_on_queue(lambda: self._create_and_send_file(text, params, default_filename))
                try:
                    peer = getattr(params, "peer", None)
                    if peer is not None:
                        self._busy_peers.discard(peer)
                except Exception:
                    pass
                return
            
            act = frag.getParentActivity()
            if not act:
                log("[CompactText] no activity, sending without dialog")
                run_on_queue(lambda: self._create_and_send_file(text, params, default_filename))
                try:
                    peer = getattr(params, "peer", None)
                    if peer is not None:
                        self._busy_peers.discard(peer)
                except Exception:
                    pass
                return
            
            log("[CompactText] building dialog UI...")
            size_str = self._format_file_size(len(text))
            if "." in default_filename:
                base = default_filename.rsplit(".", 1)[0]
                ext = "." + default_filename.rsplit(".", 1)[1]
            else:
                base = default_filename
                ext = ""
            selected_ext = [ext]
            container = LinearLayout(act)
            container.setOrientation(LinearLayout.VERTICAL)
            container.setPadding(AndroidUtilities.dp(24), AndroidUtilities.dp(10), AndroidUtilities.dp(24), AndroidUtilities.dp(10))
            inp = EditTextBoldCursor(act)
            inp.setHint("Имя файла (без расширения)")
            inp.setText(base)
            inp.setInputType(InputType.TYPE_CLASS_TEXT)
            inp.setMaxLines(1)
            inp.setTextSize(TypedValue.COMPLEX_UNIT_DIP, 16)
            inp.setPadding(AndroidUtilities.dp(16), AndroidUtilities.dp(12), AndroidUtilities.dp(16), AndroidUtilities.dp(12))
            inp.setTextColor(-1)
            inp.setHintTextColor(Theme.getColor(Theme.key_dialogTextHint))
            ext_tv = TextView(act)
            ext_tv.setText(f"Расширение: {selected_ext[0]}")
            ext_tv.setTextSize(TypedValue.COMPLEX_UNIT_DIP, 14)
            ext_tv.setTextColor(Theme.getColor(Theme.key_dialogTextGray2))
            ext_tv.setPadding(0, AndroidUtilities.dp(8), 0, 0)
            clear_tv = TextView(act)
            clear_tv.setText("Очистить поле")
            clear_tv.setTextSize(TypedValue.COMPLEX_UNIT_DIP, 14)
            clear_tv.setTextColor(Theme.getColor(Theme.key_dialogTextBlue2))
            clear_tv.setPadding(0, AndroidUtilities.dp(8), 0, 0)
            def _on_clear(v):
                try:
                    inp.setText("")
                except Exception:
                    pass
            clear_tv.setOnClickListener(OnClickListener(_on_clear))
            
            change_ext_tv = TextView(act)
            change_ext_tv.setText("Изменить расширение")
            change_ext_tv.setTextSize(TypedValue.COMPLEX_UNIT_DIP, 14)
            change_ext_tv.setTextColor(Theme.getColor(Theme.key_dialogTextBlue2))
            change_ext_tv.setPadding(0, AndroidUtilities.dp(8), 0, 0)
            def _on_change_ext(v):
                try:
                    self._show_extension_dialog(act, ext_tv, selected_ext)
                except Exception:
                    pass
            change_ext_tv.setOnClickListener(OnClickListener(_on_change_ext))
            
            send_msgs_tv = TextView(act)
            send_msgs_tv.setText("Отправить сообщениями")
            send_msgs_tv.setTextSize(TypedValue.COMPLEX_UNIT_DIP, 14)
            send_msgs_tv.setTextColor(Theme.getColor(Theme.key_dialogTextBlue2))
            send_msgs_tv.setPadding(0, AndroidUtilities.dp(8), 0, 0)
            def _on_send_msgs(v):
                try:
                    bld.dismiss()
                    threading.Thread(target=lambda: self._send_as_messages(text, params), daemon=True).start()
                except Exception:
                    pass
            send_msgs_tv.setOnClickListener(OnClickListener(_on_send_msgs))
            
            multi_send_tv = TextView(act)
            multi_send_tv.setText("Отправить несколькими файлами")
            multi_send_tv.setTextSize(TypedValue.COMPLEX_UNIT_DIP, 14)
            multi_send_tv.setTextColor(Theme.getColor(Theme.key_dialogTextBlue2))
            multi_send_tv.setPadding(0, AndroidUtilities.dp(8), 0, 0)
            def _on_multi_send(v):
                try:
                    cur_base = str(inp.getText()).strip() or base
                    bld.dismiss()
                    run_on_ui_thread(lambda: self._show_multi_send_dialog(text, params, cur_base))
                except Exception:
                    pass
            multi_send_tv.setOnClickListener(OnClickListener(_on_multi_send))
            
            container.addView(inp)
            container.addView(ext_tv)
            container.addView(clear_tv)
            container.addView(change_ext_tv)
            container.addView(send_msgs_tv)
            container.addView(multi_send_tv)
            bld = AlertDialogBuilder(act)
            bld.set_title("Имя файла")
            bld.set_message(f"Размер: {len(text)} символов ({size_str})")
            bld.set_view(container)
            peer_local = getattr(params, "peer", None)
            def on_ok(b, w):
                try:
                    name = str(inp.getText()).strip() or base
                    cur_ext = selected_ext[0] if selected_ext[0] else ext
                    full = name + cur_ext
                    b.dismiss()
                    run_on_queue(lambda: self._create_and_send_file(text, params, full))
                    try:
                        if peer_local is not None:
                            self._busy_peers.discard(peer_local)
                    except Exception:
                        pass
                except Exception:
                    pass
                    b.dismiss()
                    run_on_queue(lambda: self._create_and_send_file(text, params, default_filename))
                    try:
                        if peer_local is not None:
                            self._busy_peers.discard(peer_local)
                    except Exception:
                        pass
            def on_cancel(b, w):
                b.dismiss()
                try:
                    if peer_local is not None:
                        self._busy_peers.discard(peer_local)
                except Exception:
                    pass
            bld.set_positive_button("Отправить", on_ok)
            bld.set_negative_button("Отмена", on_cancel)
            bld.show()
        except Exception:
            run_on_queue(lambda: self._create_and_send_file(text, params, default_filename))
            try:
                peer = getattr(params, "peer", None)
                if peer is not None:
                    self._busy_peers.discard(peer)
            except Exception:
                pass

    def _show_extension_dialog(self, act, ext_tv, selected_ext_ref):
        try:
            ext_builder = AlertDialogBuilder(act)
            ext_builder.set_title("Выбор расширения")
            
            options = ["TXT", "DOCX"]
            presets = self._get_presets()
            for preset in presets:
                options.append(preset[0])
            options.append("Custom (своё)")
            
            def on_item_click(b, which):
                b.dismiss()
                if which == 0:
                    ext_tv.setText("Расширение: .txt")
                    self._show_apply_choice_dialog(act, ".txt", False, "", selected_ext_ref)
                elif which == 1:
                    ext_tv.setText("Расширение: .docx")
                    self._show_apply_choice_dialog(act, ".docx", False, "", selected_ext_ref)
                elif which == len(options) - 1:
                    self._show_custom_extension_dialog(act, ext_tv, selected_ext_ref)
                else:
                    preset = presets[which - 2]
                    ext_tv.setText(f"Расширение: .{preset[1]}")
                    self._show_apply_choice_dialog(act, f".{preset[1]}", True, preset[1], selected_ext_ref)
            
            ext_builder.set_items(options, on_item_click)
            ext_builder.set_negative_button("Отмена", lambda b, w: b.dismiss())
            ext_builder.show()
        except Exception:
            pass
    
    def _show_custom_extension_dialog(self, act, ext_tv, selected_ext_ref):
        try:
            custom_builder = AlertDialogBuilder(act)
            custom_builder.set_title("Своё расширение")
            custom_builder.set_message("Введите расширение без точки (например: md, json, log)")
            
            custom_input = EditTextBoldCursor(act)
            custom_input.setHint("Расширение")
            custom_input.setInputType(InputType.TYPE_CLASS_TEXT)
            custom_input.setMaxLines(1)
            custom_input.setTextSize(TypedValue.COMPLEX_UNIT_DIP, 16)
            custom_input.setPadding(AndroidUtilities.dp(16), AndroidUtilities.dp(12), AndroidUtilities.dp(16), AndroidUtilities.dp(12))
            custom_input.setTextColor(-1)
            custom_input.setHintTextColor(Theme.getColor(Theme.key_dialogTextHint))
            custom_builder.set_view(custom_input)
            
            def on_ok(b, w):
                custom_ext = str(custom_input.getText()).strip().lstrip(".")
                if custom_ext:
                    b.dismiss()
                    ext_tv.setText(f"Расширение: .{custom_ext}")
                    self._show_apply_choice_dialog(act, f".{custom_ext}", True, custom_ext, selected_ext_ref)
            
            custom_builder.set_positive_button("OK", on_ok)
            custom_builder.set_negative_button("Отмена", lambda b, w: b.dismiss())
            custom_builder.show()
        except Exception:
            pass
    
    def _show_multi_send_dialog(self, text: str, params: Any, base_name: str):
        try:
            frag = get_last_fragment()
            if not frag:
                run_on_queue(lambda: self._create_and_send_file(text, params, f"{base_name}.txt"))
                return
            act = frag.getParentActivity()
            if not act:
                run_on_queue(lambda: self._create_and_send_file(text, params, f"{base_name}.txt"))
                return
            peer = getattr(params, "peer", None)
            bld = AlertDialogBuilder(act)
            bld.set_title("Выберите форматы")
            container = LinearLayout(act)
            container.setOrientation(LinearLayout.VERTICAL)
            container.setPadding(AndroidUtilities.dp(24), AndroidUtilities.dp(10), AndroidUtilities.dp(24), AndroidUtilities.dp(10))
            cbs = []
            cb_txt = CheckBox(act); cb_txt.setText("TXT (.txt)"); cb_txt.setChecked(True)
            container.addView(cb_txt)
            cbs.append((cb_txt, "txt"))
            cb_docx = CheckBox(act); cb_docx.setText("DOCX (.docx)"); cb_docx.setChecked(True)
            container.addView(cb_docx)
            cbs.append((cb_docx, "docx"))
            presets = self._get_presets()
            for preset in presets:
                cb = CheckBox(act)
                cb.setText(f"{preset[0]} (.{preset[1]})")
                cb.setChecked(False)
                container.addView(cb)
                cbs.append((cb, preset[1]))
            bld.set_view(container)
            def on_ok(b, w):
                try:
                    b.dismiss()
                    exts = []
                    for cb, ext in cbs:
                        try:
                            if cb.isChecked(): exts.append(ext)
                        except Exception:
                            pass
                    if not exts:
                        return
                    def run_send():
                        for e in exts:
                            try:
                                run_on_queue(lambda e=e: self._create_and_send_file(text, params, f"{base_name}.{e}"))
                                time.sleep(0.05)
                            except Exception:
                                pass
                    threading.Thread(target=run_send, daemon=True).start()
                except Exception:
                    pass
            def on_cancel(b, w):
                b.dismiss()
                try:
                    if peer is not None:
                        self._busy_peers.discard(peer)
                except Exception:
                    pass
            bld.set_positive_button("Отправить", on_ok)
            bld.set_negative_button("Отмена", on_cancel)
            bld.show()
        except Exception:
            pass

    def _show_apply_choice_dialog(self, act, new_ext: str, is_custom: bool = False, custom_value: str = "", selected_ext_ref=None):
        try:
            choice_builder = AlertDialogBuilder(act)
            choice_builder.set_title("Применить изменение")
            choice_builder.set_message("Как применить выбранное расширение?\n\n(Настройки всегда можно изменить в разделе плагина)")
            
            def on_temp(b, w):
                b.dismiss()
                try:
                    if selected_ext_ref is not None:
                        selected_ext_ref[0] = new_ext
                    pass
                except Exception:
                    pass
            
            def on_permanent(b, w):
                try:
                    if is_custom:
                        self.set_setting("file_format", 2)
                        self.set_setting("custom_ext", custom_value)
                        pass
                    else:
                        if new_ext == ".txt":
                            self.set_setting("file_format", 0)
                        elif new_ext == ".docx":
                            self.set_setting("file_format", 1)
                        pass
                    if selected_ext_ref is not None:
                        selected_ext_ref[0] = new_ext
                    b.dismiss()
                except Exception:
                    pass
            
            choice_builder.set_positive_button("Изменить навсегда", on_permanent)
            choice_builder.set_negative_button("Только в этот раз", on_temp)
            choice_builder.show()
        except Exception:
            pass
    
    def _format_file_size(self, size_bytes: int) -> str:
        unit = self._get_int_setting("size_unit", 0)
        if unit == 0:
            return f"{size_bytes/1024:.2f} КБ"
        mb = size_bytes/(1024*1024)
        return f"{mb:.3f} МБ" if mb < 1 else f"{mb:.2f} МБ"

    def _create_and_send_file(self, text: str, params: Any, filename: str):
        peer = getattr(params, "peer", None)
        try:
            force_cpt = bool(getattr(params, "_force_file", False))
            ctx = ApplicationLoader.applicationContext
            plugin_dir = File(ctx.getExternalCacheDir(), TEMP_DIR_NAME)
            if not plugin_dir.exists():
                plugin_dir.mkdirs()
            fn = filename
            path = File(plugin_dir, fn).getAbsolutePath()
            ext = fn.rsplit(".", 1)[-1].lower() if "." in fn else "txt"
            mime = "text/plain"
            if force_cpt:
                if ext == "docx":
                    ok = False
                    try:
                        ok = self._create_docx(text, path, [])
                    except Exception:
                        ok = False
                    if ok:
                        mime = "application/vnd.openxmlformats-officedocument.wordprocessingml.document"
                    else:
                        self._write_bytes(path, text.encode("utf-8"))
                        mime = "text/plain"
                else:
                    self._write_bytes(path, text.encode("utf-8"))
                    mime = "text/plain"
            elif ext == "docx":
                insert_legend_cfg = self._get_bool_setting("insert_legend", False)
                legend_prefix = (FORMATTING_LEGEND + "\n" + ("="*50) + "\n\n") if insert_legend_cfg else ""
                out_txt = (legend_prefix + text) if legend_prefix else text
                ok = False
                try:
                    ok = self._create_docx(out_txt, path, [])
                except Exception:
                    ok = False
                if ok:
                    mime = "application/vnd.openxmlformats-officedocument.wordprocessingml.document"
                else:
                    self._write_bytes(path, out_txt.encode("utf-8"))
                    mime = "text/plain"
            else:
                insert_legend_cfg = self._get_bool_setting("insert_legend", False)
                legend_prefix = (FORMATTING_LEGEND + "\n" + ("="*50) + "\n\n") if insert_legend_cfg else ""
                out_txt = (legend_prefix + text) if legend_prefix else text
                try:
                    self._write_bytes(path, out_txt.encode("utf-8"))
                except Exception:
                    pass
                mime = "text/plain"
            account = get_account_instance()
            dialog_id = params.peer
            reply_to_msg = getattr(params, "replyToMsg", None)
            reply_to_top_msg = getattr(params, "replyToTopMsg", None)
            def send_doc():
                try:
                    SendMessagesHelper.prepareSendingDocument(account, path, path, None, None, mime, dialog_id, reply_to_msg, reply_to_top_msg, None, None, None, True, 0, None, None, 0, False)
                    try:
                        from android.os import Handler, Looper
                        Handler(Looper.getMainLooper()).postDelayed(lambda: self._clear_input_field_safe(), 150)
                    except Exception:
                        run_on_ui_thread(self._clear_input_field_safe)
                except Exception:
                    pass
                finally:
                    try:
                        if peer is not None:
                            self._busy_peers.discard(peer)
                    except Exception:
                        pass
            run_on_ui_thread(send_doc)
        except Exception:
            try:
                if peer is not None:
                    self._busy_peers.discard(peer)
            except Exception:
                pass

    def _write_bytes(self, path: str, data: bytes):
        with open(path, "wb") as f:
            f.write(data)
    
    def _parse_formatting_to_docx_runs(self, text: str) -> list:
        def escape_xml(s):
            return s.replace("&", "&amp;").replace("<", "&lt;").replace(">", "&gt;")
        esc = escape_xml(text)
        return [f'<w:r><w:t xml:space="preserve">{esc}</w:t></w:r>']

    def _get_text_and_entities_from_ui(self, expected_text: str) -> Dict[str, Any]:
        return {"text": expected_text or "", "entities": []}

    def _apply_entities_to_plain_text(self, text: str, entities: List[dict], allowed_types: Set[str] = None) -> str:
        return text

    def _apply_entities_runs_for_range(self, full_text: str, entities: List[dict], start: int, end: int, is_line_start: bool=False, rels_map: Dict[str, str]=None, allowed_types: Set[str] = None) -> list:
        def escape_xml(s):
            return s.replace("&","&amp;").replace("<","&lt;").replace(">","&gt;")
        n = len(full_text)
        a = max(0, min(start, n))
        b = max(0, min(end, n))
        if b <= a:
            return []
        esc = escape_xml(full_text[a:b])
        return [f'<w:r><w:t xml:space="preserve">{esc}</w:t></w:r>']

    def _extract_entities_from_params(self, params: Any, text: str) -> List[dict]:
        return []

    def _create_docx(self, text: str, path: str, entities: List[dict] = None, allowed_types: Set[str] = None) -> bool:
        try:
            buf = []
            rels_map: Dict[str, str] = {}
            lines = text.split('\n')
            offset = 0
            for idx, line in enumerate(lines):
                start = offset
                end = offset + len(line)
                if entities:
                    try:
                        pre_starting = next((e for e in entities if e.get('type')=='pre' and int(e.get('start',-1))>=start and int(e.get('start',-1))<end and e.get('lang')), None)
                    except Exception:
                        pre_starting = None
                    if pre_starting is not None:
                        lang = str(pre_starting.get('lang','')).strip()
                        if lang:
                            caption_xml = (
                                '<w:p>'
                                '<w:r><w:rPr><w:i/>'
                                '<w:shd w:val="clear" w:color="auto" w:fill="ECECEC"/>'
                                '</w:rPr><w:t xml:space="preserve">'
                                f'Код: {lang}'
                                '</w:t></w:r>'
                                '</w:p>'
                            )
                            buf.append(caption_xml)
                if entities is not None:
                    if entities:
                        runs = self._apply_entities_runs_for_range(text, entities, start, end, is_line_start=True, rels_map=rels_map, allowed_types=allowed_types)
                    else:
                        esc = line.replace("&","&amp;").replace("<","&lt;").replace(">","&gt;")
                        runs = [f'<w:r><w:t xml:space="preserve">{esc}</w:t></w:r>']
                else:
                    runs = self._parse_formatting_to_docx_runs(line)
                para_xml = "<w:p>" + "".join(runs) + "</w:p>"
                buf.append(para_xml)
                offset = end + 1
            document_xml = "".join([
                "<?xml version=\"1.0\" encoding=\"UTF-8\" standalone=\"yes\"?>",
                "<w:document xmlns:w=\"http://schemas.openxmlformats.org/wordprocessingml/2006/main\" xmlns:r=\"http://schemas.openxmlformats.org/officeDocument/2006/relationships\">",
                "<w:body>",
                "".join(buf),
                "<w:sectPr><w:pgSz w:w=\"12240\" w:h=\"15840\"/><w:pgMar w:top=\"1440\" w:right=\"1440\" w:bottom=\"1440\" w:left=\"1440\"/></w:sectPr>",
                "</w:body></w:document>"
            ])
            content_types = """<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<Types xmlns="http://schemas.openxmlformats.org/package/2006/content-types">
 <Default Extension="rels" ContentType="application/vnd.openxmlformats-package.relationships+xml"/>
 <Default Extension="xml" ContentType="application/xml"/>
 <Override PartName="/word/document.xml" ContentType="application/vnd.openxmlformats-officedocument.wordprocessingml.document.main+xml"/>
</Types>"""
            rels = """<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<Relationships xmlns="http://schemas.openxmlformats.org/package/2006/relationships">
 <Relationship Id="rId1" Type="http://schemas.openxmlformats.org/officeDocument/2006/relationships/officeDocument" Target="word/document.xml"/>
</Relationships>"""
            doc_rels_xml = "<?xml version=\"1.0\" encoding=\"UTF-8\" standalone=\"yes\"?>\n<Relationships xmlns=\"http://schemas.openxmlformats.org/package/2006/relationships\">" + \
                "".join([
                    f"<Relationship Id=\"{rid}\" Type=\"http://schemas.openxmlformats.org/officeDocument/2006/relationships/hyperlink\" Target=\"{url}\" TargetMode=\"External\"/>"
                    for url, rid in rels_map.items()
                ]) + "</Relationships>"

            try:
                with zipfile.ZipFile(path, "w", compression=zipfile.ZIP_DEFLATED, compresslevel=9) as z:
                    z.writestr("[Content_Types].xml", content_types)
                    z.writestr("_rels/.rels", rels)
                    z.writestr("word/document.xml", document_xml)
                    if rels_map:
                        z.writestr("word/_rels/document.xml.rels", doc_rels_xml)
            except Exception:
                with zipfile.ZipFile(path, "w", compression=zipfile.ZIP_DEFLATED) as z:
                    z.writestr("[Content_Types].xml", content_types)
                    z.writestr("_rels/.rels", rels)
                    z.writestr("word/document.xml", document_xml)
                    if rels_map:
                        z.writestr("word/_rels/document.xml.rels", doc_rels_xml)
            return True
        except Exception:
            return False

    def _on_format_skip_threshold_change(self, value: str):
        try:
            v = int(str(value).strip())
        except Exception:
            v = 20000
        v = max(100, v)
        self.set_setting("format_skip_threshold", v)

    def _get_allowed_types(self) -> Set[str]:
        if self._get_bool_setting("format_use_all", True):
            return {"bold","italic","underline","strike","code","pre","quote","link","spoiler"}
        allowed = set()
        if self._get_bool_setting("fmt_bold", True): allowed.add("bold")
        if self._get_bool_setting("fmt_italic", True): allowed.add("italic")
        if self._get_bool_setting("fmt_underline", True): allowed.add("underline")
        if self._get_bool_setting("fmt_strike", True): allowed.add("strike")
        if self._get_bool_setting("fmt_code", True): allowed.add("code")
        if self._get_bool_setting("fmt_pre", True): allowed.add("pre")
        if self._get_bool_setting("fmt_quote", True): allowed.add("quote")
        if self._get_bool_setting("fmt_link", True): allowed.add("link")
        if self._get_bool_setting("fmt_spoiler", True): allowed.add("spoiler")
        return allowed

    def _get_bool_setting(self, key: str, default: bool) -> bool:
        try:
            v = self.get_setting(key, default)
            if isinstance(v, bool):
                return v
            if isinstance(v, (int, float)):
                return bool(v)
            if isinstance(v, str):
                s = v.strip().lower()
                if s in ("1", "true", "on", "yes", "y"):
                    return True
                if s in ("0", "false", "off", "no", "n"):
                    return False
                return default
            return default
        except Exception:
            return default

    def _get_int_setting(self, key: str, default: int) -> int:
        try:
            v = self.get_setting(key, default)
            if isinstance(v, int):
                return v
            if isinstance(v, float):
                return int(v)
            s = str(v).strip()
            try:
                return int(s)
            except Exception:
                s_low = s.lower()
                if key == "file_format":
                    if s_low in ("0", "txt", "text"):
                        return 0
                    if s_low in ("1", "docx", "word"):
                        return 1
                    if s_low in ("2", "custom", "other"):
                        return 2
                if key == "size_unit":
                    if s_low in ("0", "kb", "кб"):
                        return 0
                    if s_low in ("1", "mb", "мб"):
                        return 1
                return default
        except Exception:
            return default

    def _get_split_mode(self) -> int:
        try:
            return self._get_int_setting("split_mode", 0)
        except Exception:
            return 0

    def _split_text_by_limit(self, text: str, limit: int) -> List[str]:
        if not text:
            return []
        if len(text) <= limit:
            return [text]

        chunks: List[str] = []
        start = 0
        min_chunk_size = max(100, int(limit * 0.3))
        
        while start < len(text):
            end = min(start + limit, len(text))
            
            if end < len(text):
                last_newline = text.rfind('\n', start, end)
                
                if last_newline > start and (last_newline - start) >= min_chunk_size:
                    end = last_newline + 1
                else:
                    last_space = text.rfind(' ', start + min_chunk_size, end)
                    if last_space > start:
                        end = last_space + 1
            
            chunk = text[start:end]
            if chunk and chunk.strip():
                chunks.append(chunk)
            
            start = end
            
            if start == end:
                start += 1
        
        return chunks

    def _get_threshold(self) -> int:
        try:
            val = self.get_setting("threshold", DEFAULT_THRESHOLD)
            if isinstance(val, int):
                threshold = val
            else:
                threshold = int(str(val).strip())
            if threshold < 100:
                threshold = 100
            elif threshold > 500000:
                threshold = 500000
            return threshold
        except Exception:
            return DEFAULT_THRESHOLD
    
    def _get_presets(self) -> List[tuple]:
        try:
            raw = self.get_setting("presets_json", [])
            arr = []
            if isinstance(raw, str):
                try:
                    arr = json.loads(raw)
                except Exception:
                    arr = []
            else:
                if hasattr(raw, "size") and hasattr(raw, "get"):
                    try:
                        arr = [raw.get(i) for i in range(raw.size())]
                    except Exception:
                        arr = []
                elif hasattr(raw, "__iter__"):
                    try:
                        arr = list(raw)
                    except Exception:
                        arr = []

            out: List[tuple] = []
            for it in arr:
                name = ""
                ext = ""
                if isinstance(it, dict):
                    name = str(it.get("name", "")).strip()
                    ext = str(it.get("ext", "")).strip()
                elif hasattr(it, "get"):
                    try:
                        name = str(it.get("name")).strip()
                        ext = str(it.get("ext")).strip()
                    except Exception:
                        name = ext = ""
                elif isinstance(it, (list, tuple)) and len(it) >= 2:
                    name = str(it[0]).strip()
                    ext = str(it[1]).strip()
                if name and ext:
                    out.append((name, ext.lstrip(".")))
            pass
            return out[:20]
        except Exception:
            pass
            return []

    def _save_presets(self, presets: List[tuple]):
        try:
            data = [{"name": n, "ext": e} for n, e in presets[:20]]
            self.set_setting("presets_json", data)
            pass
        except Exception:
            pass
    
    def _trigger_settings_reload(self):
        try:
            def do_reload():
                try:
                    current = self.get_setting("_ui_refresh", 0)
                    self.set_setting("_ui_refresh", current + 1)
                    
                    frag = get_last_fragment()
                    if frag:
                        for method_name in ["recreateSettings", "reloadSettings", "updateSettings", "invalidate"]:
                            if hasattr(frag, method_name):
                                method = getattr(frag, method_name)
                                method()
                                log(f"[CompactText] Settings reloaded via {method_name}")
                                return
                    
                    current_format = self._get_int_setting("file_format", 0)
                    self.set_setting("file_format", (current_format + 1) % 3)
                    time.sleep(0.01)
                    self.set_setting("file_format", current_format)
                    log("[CompactText] Settings reload triggered via format toggle")
                except Exception as e:
                    log(f"[CompactText] reload attempt error: {e}")
            
            run_on_ui_thread(do_reload)
        except Exception as e:
            log(f"[CompactText] trigger reload error: {e}")

    def _show_add_preset_dialog(self):
        try:
            frag = get_last_fragment()
            if not frag or not frag.getParentActivity():
                return
            act = frag.getParentActivity()
            presets = self._get_presets()
            if len(presets) >= 20:
                try:
                    BulletinHelper.show_error("Достигнут лимит заготовок (20)")
                except Exception:
                    log("Достигнут лимит заготовок (20)")
                return
            ll = LinearLayout(act)
            ll.setOrientation(LinearLayout.VERTICAL)
            ll.setPadding(AndroidUtilities.dp(24), AndroidUtilities.dp(10), AndroidUtilities.dp(24), AndroidUtilities.dp(10))
            name_inp = EditTextBoldCursor(act)
            name_inp.setHint("Название заготовки")
            name_inp.setMaxLines(1)
            name_inp.setInputType(InputType.TYPE_CLASS_TEXT)
            name_inp.setTextSize(TypedValue.COMPLEX_UNIT_DIP, 16)
            name_inp.setPadding(AndroidUtilities.dp(16), AndroidUtilities.dp(12), AndroidUtilities.dp(16), AndroidUtilities.dp(12))
            name_inp.setTextColor(-1)
            name_inp.setHintTextColor(Theme.getColor(Theme.key_dialogTextHint))
            ext_inp = EditTextBoldCursor(act)
            ext_inp.setHint("Расширение (без точки)")
            ext_inp.setMaxLines(1)
            ext_inp.setInputType(InputType.TYPE_CLASS_TEXT)
            ext_inp.setTextSize(TypedValue.COMPLEX_UNIT_DIP, 16)
            ext_inp.setPadding(AndroidUtilities.dp(16), AndroidUtilities.dp(12), AndroidUtilities.dp(16), AndroidUtilities.dp(12))
            ext_inp.setTextColor(-1)
            ext_inp.setHintTextColor(Theme.getColor(Theme.key_dialogTextHint))
            ll.addView(name_inp)
            ll.addView(ext_inp)
            b = AlertDialogBuilder(act)
            b.set_title("Новая заготовка")
            b.set_view(ll)
            def on_ok(dlg, w):
                name = str(name_inp.getText()).strip()
                ext = str(ext_inp.getText()).strip().lstrip(".")
                if not name or not ext:
                    try:
                        BulletinHelper.show_error("Укажите название и расширение")
                    except Exception:
                        log("Укажите название и расширение")
                    return
                presets2 = self._get_presets()
                presets2.append((name, ext))
                log(f"[CompactText] Adding preset: {name} (.{ext})")
                self._save_presets(presets2)
                dlg.dismiss()
                try:
                    BulletinHelper.show_success("Заготовка добавлена")
                except Exception:
                    log("Заготовка добавлена")
                threading.Timer(0.2, self._trigger_settings_reload).start()
            b.set_positive_button("Добавить", on_ok)
            b.set_negative_button("Отмена", lambda d, w: d.dismiss())
            b.show()
        except Exception as e:
            log(f"[CompactText] add preset error: {e}")

    def _open_presets_dialog(self):
        try:
            frag = get_last_fragment()
            act = frag.getParentActivity() if frag and frag.getParentActivity() else ApplicationLoader.applicationContext
            bld = AlertDialogBuilder(act)
            bld.set_title("Заготовки расширений")
            root = LinearLayout(act)
            root.setOrientation(LinearLayout.VERTICAL)
            root.setPadding(AndroidUtilities.dp(24), AndroidUtilities.dp(10), AndroidUtilities.dp(24), AndroidUtilities.dp(10))

            list_container = LinearLayout(act)
            list_container.setOrientation(LinearLayout.VERTICAL)
            list_container.setPadding(0, AndroidUtilities.dp(6), 0, 0)

            name_inp = EditTextBoldCursor(act)
            name_inp.setHint("Название")
            name_inp.setMaxLines(1)
            name_inp.setInputType(InputType.TYPE_CLASS_TEXT)
            name_inp.setTextSize(TypedValue.COMPLEX_UNIT_DIP, 16)
            name_inp.setPadding(AndroidUtilities.dp(16), AndroidUtilities.dp(10), AndroidUtilities.dp(16), AndroidUtilities.dp(10))
            name_inp.setTextColor(-1)
            name_inp.setHintTextColor(Theme.getColor(Theme.key_dialogTextHint))

            ext_inp = EditTextBoldCursor(act)
            ext_inp.setHint("Расширение (без точки)")
            ext_inp.setMaxLines(1)
            ext_inp.setInputType(InputType.TYPE_CLASS_TEXT)
            ext_inp.setTextSize(TypedValue.COMPLEX_UNIT_DIP, 16)
            ext_inp.setPadding(AndroidUtilities.dp(16), AndroidUtilities.dp(10), AndroidUtilities.dp(16), AndroidUtilities.dp(10))
            ext_inp.setTextColor(-1)
            ext_inp.setHintTextColor(Theme.getColor(Theme.key_dialogTextHint))

            add_btn = TextView(act)
            add_btn.setText("Добавить")
            add_btn.setTextSize(TypedValue.COMPLEX_UNIT_DIP, 14)
            add_btn.setTextColor(Theme.getColor(Theme.key_dialogTextBlue2))
            add_btn.setPadding(0, AndroidUtilities.dp(8), 0, AndroidUtilities.dp(8))

            def _do_delete(index: int):
                try:
                    arr = self._get_presets()
                    if 0 <= index < len(arr):
                        arr.pop(index)
                        self._save_presets(arr)
                        try:
                            BulletinHelper.show_success("Удалено")
                        except Exception:
                            pass
                        render()
                except Exception as e:
                    log(f"[CompactText] del err: {e}")

            def _show_edit_dialog(index: int):
                try:
                    arr = self._get_presets()
                    if not (0 <= index < len(arr)):
                        return
                    old_name, old_ext = arr[index]
                    eb = AlertDialogBuilder(act)
                    eb.set_title("Изменить заготовку")
                    lay = LinearLayout(act)
                    lay.setOrientation(LinearLayout.VERTICAL)
                    lay.setPadding(AndroidUtilities.dp(24), AndroidUtilities.dp(10), AndroidUtilities.dp(24), AndroidUtilities.dp(10))
                    in1 = EditTextBoldCursor(act)
                    in1.setHint("Название")
                    in1.setText(str(old_name))
                    in1.setMaxLines(1)
                    in1.setInputType(InputType.TYPE_CLASS_TEXT)
                    in1.setTextSize(TypedValue.COMPLEX_UNIT_DIP, 16)
                    in1.setPadding(AndroidUtilities.dp(16), AndroidUtilities.dp(10), AndroidUtilities.dp(16), AndroidUtilities.dp(10))
                    in1.setTextColor(-1)
                    in1.setHintTextColor(Theme.getColor(Theme.key_dialogTextHint))
                    in2 = EditTextBoldCursor(act)
                    in2.setHint("Расширение (без точки)")
                    in2.setText(str(old_ext))
                    in2.setMaxLines(1)
                    in2.setInputType(InputType.TYPE_CLASS_TEXT)
                    in2.setTextSize(TypedValue.COMPLEX_UNIT_DIP, 16)
                    in2.setPadding(AndroidUtilities.dp(16), AndroidUtilities.dp(10), AndroidUtilities.dp(16), AndroidUtilities.dp(10))
                    in2.setTextColor(-1)
                    in2.setHintTextColor(Theme.getColor(Theme.key_dialogTextHint))
                    lay.addView(in1)
                    lay.addView(in2)
                    eb.set_view(lay)
                    def on_ok(db, w):
                        try:
                            n = str(in1.getText()).strip()
                            e = str(in2.getText()).strip().lstrip(".")
                            if not n or not e:
                                try: BulletinHelper.show_error("Укажите название и расширение")
                                except Exception: pass
                                return
                            nl, el = n.lower(), e.lower()
                            for j, (xn, xe) in enumerate(arr):
                                if j == index: continue
                                if nl == str(xn).lower() or el == str(xe).lower():
                                    try: BulletinHelper.show_error("Такое название/расширение уже есть")
                                    except Exception: pass
                                    return
                            arr2 = list(arr)
                            arr2[index] = (n, e)
                            self._save_presets(arr2)
                            try: BulletinHelper.show_success("Сохранено")
                            except Exception: pass
                            db.dismiss()
                            render()
                        except Exception as ie:
                            log(f"[CompactText] edit err: {ie}")
                    eb.set_positive_button("Сохранить", on_ok)
                    eb.set_negative_button("Отмена", lambda d, w: d.dismiss())
                    eb.show()
                except Exception as e:
                    log(f"[CompactText] edit dlg err: {e}")

            def render():
                try:
                    list_container.removeAllViews()
                except Exception:
                    pass
                presets = self._get_presets()
                if not presets:
                    tv = TextView(act)
                    tv.setText("Пока пусто")
                    tv.setTextSize(TypedValue.COMPLEX_UNIT_DIP, 14)
                    tv.setTextColor(Theme.getColor(Theme.key_dialogTextGray2))
                    tv.setPadding(0, AndroidUtilities.dp(6), 0, AndroidUtilities.dp(6))
                    list_container.addView(tv)
                else:
                    for i, (pname, pext) in enumerate(presets):
                        row = TextView(act)
                        row.setText(f"{pname} (.{pext}) — нажмите, чтобы изменить/удалить")
                        row.setTextSize(TypedValue.COMPLEX_UNIT_DIP, 15)
                        row.setTextColor(-1)
                        row.setPadding(0, AndroidUtilities.dp(6), 0, AndroidUtilities.dp(6))
                        def on_choose(v, idx=i, n=pname, e=pext):
                            try:
                                chooser = AlertDialogBuilder(act)
                                chooser.set_title(f"{n} (.{e})")
                                options = ["Изменить", "Удалить"]
                                def on_item(b, which):
                                    b.dismiss()
                                    if which == 0:
                                        _show_edit_dialog(idx)
                                    else:
                                        bb = AlertDialogBuilder(act)
                                        bb.set_title("Удалить")
                                        bb.set_message(f"Удалить '{n} (.{e})'?")
                                        bb.set_positive_button("Удалить", lambda d, w: _do_delete(idx))
                                        bb.set_negative_button("Отмена", lambda d, w: d.dismiss())
                                        bb.show()
                                chooser.set_items(options, on_item)
                                chooser.set_negative_button("Отмена", lambda d, w: d.dismiss())
                                chooser.show()
                            except Exception as ee:
                                log(f"[CompactText] choose err: {ee}")
                        row.setOnClickListener(OnClickListener(on_choose))
                        list_container.addView(row)

            def on_add(v):
                try:
                    name = str(name_inp.getText()).strip()
                    ext = str(ext_inp.getText()).strip().lstrip(".")
                    if not name or not ext:
                        try:
                            BulletinHelper.show_error("Укажите название и расширение")
                        except Exception:
                            pass
                        return
                    arr = self._get_presets()
                    if len(arr) >= 20:
                        try:
                            BulletinHelper.show_error("Достигнут лимит (20)")
                        except Exception:
                            pass
                        return
                    nl, el = name.lower(), ext.lower()
                    for (xn, xe) in arr:
                        if nl == str(xn).lower() or el == str(xe).lower():
                            try:
                                BulletinHelper.show_error("Такое название/расширение уже есть")
                            except Exception:
                                pass
                            return
                    arr.append((name, ext))
                    self._save_presets(arr)
                    name_inp.setText("")
                    ext_inp.setText("")
                    try:
                        BulletinHelper.show_success("Добавлено")
                    except Exception:
                        pass
                    render()
                except Exception as e:
                    log(f"[CompactText] add err: {e}")

            add_btn.setOnClickListener(OnClickListener(on_add))

            root.addView(list_container)
            try:
                vdiv = TextView(act)
                vdiv.setText(" ")
                vdiv.setHeight(AndroidUtilities.dp(1))
                try:
                    div_color = Theme.getColor(Theme.key_divider)
                except Exception:
                    div_color = 0x19FFFFFF
                vdiv.setBackgroundColor(div_color)
                root.addView(vdiv)
            except Exception:
                pass
            try:
                spacer = TextView(act)
                spacer.setText(" ")
                spacer.setHeight(AndroidUtilities.dp(8))
                root.addView(spacer)
            except Exception:
                pass
            root.addView(name_inp)
            root.addView(ext_inp)
            root.addView(add_btn)

            bld.set_view(root)
            bld.set_negative_button("Закрыть", lambda d, w: d.dismiss())
            bld.show()
            render()
        except Exception as e:
            log(f"[CompactText] open presets dialog error: {e}")

    def _send_as_messages(self, text: str, params: Any):
        dialog_id = getattr(params, "peer", None)
        try:
            mode = self._get_split_mode()
            limit = TELEGRAM_MAX_CHARS if mode == 0 else self._get_threshold()
            chunks = self._split_text_by_limit(text, limit)
            
            if not chunks:
                return
            
            from client_utils import send_message
            if dialog_id is not None:
                self._bypass_peers.add(dialog_id)
            reply_to_msg = getattr(params, "replyToMsg", None)
            reply_to_top_msg = getattr(params, "replyToTopMsg", None)

            for i, chunk in enumerate(chunks):
                try:
                    chunk_to_send = chunk
                    if not chunk_to_send:
                        continue
                    send_message({
                        "message": chunk_to_send,
                        "peer": dialog_id,
                        "replyToMsg": reply_to_msg,
                        "replyToTopMsg": reply_to_top_msg,
                    })
                    time.sleep(0.2)
                except Exception:
                    pass
            
            time.sleep(0.3)
            run_on_ui_thread(self._clear_input_field_safe)
        except Exception:
            pass
        finally:
            try:
                peer = getattr(params, "peer", None)
                if peer is not None:
                    self._busy_peers.discard(peer)
                if dialog_id is not None:
                    self._bypass_peers.discard(dialog_id)
            except Exception:
                pass
    
    def _clear_input_field_safe(self):
        try:
            frag = get_last_fragment()
            if not frag:
                return
            enter_getter = getattr(frag, 'getChatActivityEnterView', None)
            if enter_getter is None:
                return
            enter = enter_getter()
            if enter is not None:
                enter.setFieldText("")
        except Exception as e:
            log(f"[CompactText] clear input error: {e}")

    def _show_formatting_legend(self):
        try:
            frag = get_last_fragment()
            if not frag:
                return
            act = frag.getParentActivity() if frag.getParentActivity() else ApplicationLoader.applicationContext
            bld = AlertDialogBuilder(act)
            bld.set_title("Форматирование текста")
            bld.set_message(FORMATTING_LEGEND)
            bld.set_positive_button("ОК", lambda d, w: d.dismiss())
            bld.show()
        except Exception as e:
            log(f"[CompactText] formatting legend error: {e}")
    
    def _show_help_dialog(self):
        try:
            frag = get_last_fragment()
            if not frag:
                return
            act = frag.getParentActivity() if frag.getParentActivity() else ApplicationLoader.applicationContext
            bld = AlertDialogBuilder(act)
            bld.set_title("Помощь")
            root = LinearLayout(act)
            root.setOrientation(LinearLayout.HORIZONTAL)
            try:
                pad = AndroidUtilities.dp(8)
                root.setPadding(pad, pad, pad, pad)
            except Exception:
                pass
            
            left = LinearLayout(act)
            left.setOrientation(LinearLayout.VERTICAL)
            right = LinearLayout(act)
            right.setOrientation(LinearLayout.VERTICAL)
            
            try:
                from java import jclass
                LP = jclass("android.widget.LinearLayout$LayoutParams")
                left.setLayoutParams(LP(0, -2, 1.0))
                right.setLayoutParams(LP(0, -2, 1.0))
            except Exception:
                pass
            
            try:
                color_text = Theme.getColor(Theme.key_windowBackgroundWhiteBlackText)
                color_acc = Theme.getColor(Theme.key_dialogTextBlue2)
            except Exception:
                color_text = -1
                color_acc = 0xFF57A9FF
            
            l_txt = TextView(act)
            l_txt.setText("Если вы нашли баг, ошибку, просьба написать создателю:")
            try:
                l_txt.setTextColor(color_text)
                l_txt.setPadding(0, 0, 0, AndroidUtilities.dp(8))
            except Exception:
                pass
            
            l_btn = TextView(act)
            l_btn.setText("Написать")
            try:
                l_btn.setTextColor(color_acc)
                from java import jclass
                GD = jclass("android.graphics.drawable.GradientDrawable")
                bg = GD()
                bg.setCornerRadius(AndroidUtilities.dp(12))
                bg.setColor(0x00000000)
                bg.setStroke(AndroidUtilities.dp(1), color_acc)
                l_btn.setBackground(bg)
                l_btn.setPadding(AndroidUtilities.dp(12), AndroidUtilities.dp(8), AndroidUtilities.dp(12), AndroidUtilities.dp(8))
                try:
                    Gravity = jclass("android.view.Gravity")
                    l_btn.setGravity(Gravity.CENTER)
                except Exception:
                    pass
            except Exception:
                pass
            
            def on_l_click(v):
                try:
                    bld.dismiss()
                except Exception:
                    pass
                self._open_telegram_link(frag, "https://t.me/m/6tvbgKPaY2Uy")
            
            l_btn.setOnClickListener(OnClickListener(on_l_click))
            left.addView(l_txt)
            left.addView(l_btn)
            
            r_txt = TextView(act)
            r_txt.setText("Если вы не понимаете использование/интерфейс и тд, зайдите наш канал:")
            try:
                r_txt.setTextColor(color_text)
                r_txt.setPadding(0, 0, 0, AndroidUtilities.dp(8))
            except Exception:
                pass
            
            r_btn = TextView(act)
            r_btn.setText("Зайти")
            try:
                r_btn.setTextColor(color_acc)
                from java import jclass
                GD = jclass("android.graphics.drawable.GradientDrawable")
                bg2 = GD()
                bg2.setCornerRadius(AndroidUtilities.dp(12))
                bg2.setColor(0x00000000)
                bg2.setStroke(AndroidUtilities.dp(1), color_acc)
                r_btn.setBackground(bg2)
                r_btn.setPadding(AndroidUtilities.dp(12), AndroidUtilities.dp(8), AndroidUtilities.dp(12), AndroidUtilities.dp(8))
                try:
                    Gravity = jclass("android.view.Gravity")
                    r_btn.setGravity(Gravity.CENTER)
                except Exception:
                    pass
            except Exception:
                pass
            
            def on_r_click(v):
                try:
                    bld.dismiss()
                except Exception:
                    pass
                self._open_telegram_link(frag, "https://t.me/buligaEplugins")
            
            r_btn.setOnClickListener(OnClickListener(on_r_click))
            right.addView(r_txt)
            right.addView(r_btn)
            
            root.addView(left)
            root.addView(right)
            bld.set_view(root)
            bld.set_negative_button("Закрыть", lambda d, w: d.dismiss())
            bld.show()
        except Exception as e:
            log(f"[CompactText] help dialog error: {e}")
    
    def _open_telegram_link(self, fragment, url: str):
        try:
            activity = fragment.getParentActivity() if fragment and fragment.getParentActivity() else ApplicationLoader.applicationContext
            from java import jclass
            Intent = jclass("android.content.Intent")
            Uri = jclass("android.net.Uri")
            intent = Intent(Intent.ACTION_VIEW, Uri.parse(url))
            try:
                activity.startActivity(intent)
            except Exception:
                try:
                    intent2 = Intent(Intent.ACTION_VIEW, Uri.parse(url))
                    activity.startActivity(intent2)
                except Exception:
                    pass
        except Exception as e:
            log(f"[CompactText] open link error: {e}")
