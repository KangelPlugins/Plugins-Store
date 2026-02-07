import re
import math
import requests
import traceback
from base_plugin import BasePlugin, HookResult, HookStrategy, MenuItemData, MenuItemType
from ui.settings import Header, Switch, Divider, Input
from ui.bulletin import BulletinHelper
from client_utils import get_last_fragment, run_on_queue, send_message
from android_utils import run_on_ui_thread, log
from markdown_utils import parse_markdown
from com.exteragram.messenger.plugins import PluginsController
from com.exteragram.messenger.plugins.ui import PluginSettingsActivity
from android.content import Context, ClipData
from org.telegram.messenger import LocaleController

__id__ = "shareui_advcalc"
__name__ = "Advanced Calculator"
__description__ = """**Advanced math, BMI, Currency, etc.**
Features & Guide -> [List](https://t.me/shuiilog/73)  
Bug reports -> [Developer](https://t.me/shareui)  

**WARNING**
If you have an older version, uninstall it before installing! I changed the plugin ID.
"""
__author__ = "@shareui"
__version__ = "1.3.0"
__min_version__ = "12.1.1"
__icon__ = "plugin232/1"

ALLOWED_NAMES = {k: v for k, v in math.__dict__.items() if not k.startswith("__")}
SAFE_EVAL_GLOBALS = {"__builtins__": None}
SAFE_EVAL_GLOBALS.update(ALLOWED_NAMES)

class CalculatorPlugin(BasePlugin):
    def __init__(self):
        super().__init__()
        self.strings = {
            'ru': {
                'settings_general': 'Общие настройки',
                'only_answer': 'Отправлять только ответ',
                'only_answer_desc': "Отправляет '4' вместо '2+2=4'",
                'cmd_prefixes': 'Префиксы команд',
                'cmd_calc': 'Команда калькулятора',
                'cmd_bmi': 'Команда BMI',
                'help_text': 'Валюты: 100 usd to eur. BMI: вес(кг) рост(см).',
                'settings_title': 'Настройки калькулятора',
                'calculating': 'Вычисляю...',
                'enter_expression': 'Введите выражение',
                'not_valid_expression': 'Не является математическим выражением',
                'reply_to_message': 'Ответьте на сообщение для вычисления',
                'bmi_usage': 'Использование: вес(кг) рост(см)',
                'bmi_must_be_integers': 'Вес и рост должны быть целыми числами',
                'bmi_error': 'Ошибка BMI',
                'clipboard_copied': 'Текст скопирован в буфер обмена',
                'clipboard_error': 'Ошибка буфера обмена',
                'settings_open_error': 'Не удалось открыть настройки плагина',
                'send_error': 'Ошибка отправки'
            },
            'en': {
                'settings_general': 'General Settings',
                'only_answer': 'Send only answer',
                'only_answer_desc': "Sends '4' instead of '2+2=4'",
                'cmd_prefixes': 'Command Prefixes',
                'cmd_calc': 'Calculator command',
                'cmd_bmi': 'BMI command',
                'help_text': 'Currencies: 100 usd to eur. BMI: weight(kg) height(cm).',
                'settings_title': 'Calculator Settings',
                'calculating': 'Calculating...',
                'enter_expression': 'Enter expression',
                'not_valid_expression': 'Not a valid expression',
                'reply_to_message': 'Reply to a message to calculate it',
                'bmi_usage': 'Usage: weight(kg) height(cm)',
                'bmi_must_be_integers': 'Weight and height must be whole numbers',
                'bmi_error': 'BMI Error',
                'clipboard_copied': 'Text copied to clipboard',
                'clipboard_error': 'Clipboard error',
                'settings_open_error': 'Failed to open plugin settings',
                'send_error': 'Send error'
            }
        }

    def _get_string(self, key):
        try:
            lang = LocaleController.getInstance().getCurrentLocale().getLanguage()
            lang_key = 'ru' if lang.startswith('ru') else 'en'
            return self.strings[lang_key].get(key, self.strings['en'].get(key, key))
        except:
            return self.strings['en'].get(key, key)

    def on_plugin_load(self):
        self.add_on_send_message_hook()
        self.add_menu_item(
            MenuItemData(
                menu_type=MenuItemType.CHAT_ACTION_MENU,
                text=self._get_string('settings_title'),
                icon="msg_settings",
                on_click=self._open_plugin_settings
            )
        )

    def _open_plugin_settings(self, context: dict):
        def action():
            try:
                java_plugin = PluginsController.getInstance().plugins.get(self.id)
                if java_plugin:
                    last_fragment = get_last_fragment()
                    if last_fragment:
                        last_fragment.presentFragment(PluginSettingsActivity(java_plugin))
            except Exception as e:
                log(f"{self._get_string('settings_open_error')}: {e}")
        run_on_ui_thread(action)

    def create_settings(self):
        return [
            Header(text=self._get_string('settings_general')),
            Switch(
                key="only_answer",
                text=self._get_string('only_answer'),
                subtext=self._get_string('only_answer_desc'),
                default=False
            ),
            Header(text=self._get_string('cmd_prefixes')),
            Input(
                key="cmd_calc",
                text=self._get_string('cmd_calc'),
                default=".calc"
            ),
            Input(
                key="cmd_bmi",
                text=self._get_string('cmd_bmi'),
                default=".calcbmi"
            ),
            Divider(text=self._get_string('help_text'))
        ]

    def _copy_to_clipboard(self, text):
        try:
            act = get_last_fragment().getParentActivity()
            cm = act.getSystemService(Context.CLIPBOARD_SERVICE)
            clip = ClipData.newPlainText("calc_error", text)
            cm.setPrimaryClip(clip)
            BulletinHelper.show_success(self._get_string('clipboard_copied'), get_last_fragment())
        except Exception as e:
            log(f"{self._get_string('clipboard_error')}: {e}")

    def _preprocess_expression(self, expression):
        fraction_map = {
            '½': '(1/2)', '⅓': '(1/3)', '⅔': '(2/3)', '¼': '(1/4)', '¾': '(3/4)',
            '⅕': '(1/5)', '⅖': '(2/5)', '⅗': '(3/5)', '⅘': '(4/5)', '⅙': '(1/6)',
            '⅚': '(5/6)', '⅛': '(1/8)', '⅜': '(3/8)', '⅝': '(5/8)', '⅞': '(7/8)'
        }
        for char, value in fraction_map.items():
            expression = expression.replace(char, value)

        expression = re.sub(r'√\s*(\d+(\.\d*)?)', r'sqrt(\1)', expression)
        processed = expression.replace('^', '**')
        superscript_map = {
            '⁰': '0', '¹': '1', '²': '2', '³': '3', '⁴': '4',
            '⁵': '5', '⁶': '6', '⁷': '7', '⁸': '8', '⁹': '9'
        }
        def power_replacer(match):
            normal_power = "".join(superscript_map.get(char, '') for char in match.group(0))
            if normal_power:
                return f"**{normal_power}"
            return ""
        pattern = r'[⁰¹²³⁴⁵⁶⁷⁸⁹]+'
        return re.sub(pattern, power_replacer, processed)

    def _convert_currency(self, expression):
        match = re.match(r'^(\d+(\.\d+)?)\s+([a-zA-Z]{3})\s+to\s+([a-zA-Z]{3})$', expression.strip(), re.IGNORECASE)
        if not match:
            return None
        
        amount = float(match.group(1))
        base = match.group(3).upper()
        target = match.group(4).upper()
        
        try:
            url = f"https://api.exchangerate-api.com/v4/latest/{base}"
            resp = requests.get(url, timeout=5).json()
            rate = resp.get('rates', {}).get(target)
            if rate:
                return f"{amount} {base} = {round(amount * rate, 2)} {target}"
            return "Exchange rate not found"
        except Exception as e:
            return f"Currency API Error: {e}"

    def _is_valid_expression(self, text):
        if not text or not text.strip():
            return False
        
        test_expr = text.strip()
        
        if re.match(r'^\d+(\.\d+)?\s+[a-zA-Z]{3}\s+to\s+[a-zA-Z]{3}$', test_expr, re.IGNORECASE):
            return True
        
        has_digit = bool(re.search(r'\d', test_expr))
        if not has_digit:
            return False
        
        has_operator = bool(re.search(r'[\+\-\*\/\(\)√\^]', test_expr))
        has_function = bool(re.search(r'\b(sin|cos|tan|log|sqrt|exp|pow)\b', test_expr, re.IGNORECASE))
        has_special = bool(re.search(r'[½⅓⅔¼¾⅕⅖⅗⅘⅙⅚⅛⅜⅝⅞⁰¹²³⁴⁵⁶⁷⁸⁹]', test_expr))
        
        text_without_functions = re.sub(r'\b(sin|cos|tan|log|sqrt|exp|pow)\b', '', test_expr, flags=re.IGNORECASE)
        text_without_functions = re.sub(r'\b[a-zA-Z]{3}\s+to\s+[a-zA-Z]{3}\b', '', text_without_functions, flags=re.IGNORECASE)
        has_invalid_letters = bool(re.search(r'[a-zA-Z]', text_without_functions))
        
        return has_digit and (has_operator or has_function or has_special) and not has_invalid_letters

    def _calculate_logic(self, expression_str, peer_id, show_bulletin_on_error=False):
        try:
            currency_result = self._convert_currency(expression_str)
            if currency_result:
                self._send_markdown_response(currency_result, peer_id)
                return

            preprocessed_str = self._preprocess_expression(expression_str)
            processed_expression = preprocessed_str.replace(':', '/').replace(',', '.')
            
            result = eval(processed_expression, SAFE_EVAL_GLOBALS, {})
            if isinstance(result, (float, int)):
                result = round(result, 10)
            
            show_only_answer = self.get_setting("only_answer", False)
            if show_only_answer:
                md_text = f"`{result}`"
            else:
                md_text = f"`{expression_str}` = *{result}*"
            
            self._send_markdown_response(md_text, peer_id)

        except Exception as e:
            error_msg = f"Error: {str(e)}"
            if show_bulletin_on_error:
                run_on_ui_thread(lambda: BulletinHelper.show_error(
                    self._get_string('not_valid_expression'),
                    get_last_fragment()
                ))
            else:
                run_on_ui_thread(lambda: BulletinHelper.show_with_button(
                    error_msg,
                    "msg_copy",
                    "RECOVER",
                    lambda: self._copy_to_clipboard(expression_str),
                    get_last_fragment()
                ))

    def _calculate_bmi_logic(self, args_str, peer_id):
        try:
            parts = args_str.split()
            if len(parts) != 2:
                raise ValueError(self._get_string('bmi_usage'))
            
            weight_str, height_str = parts[0], parts[1]
            
            if not (weight_str.isdigit() and height_str.isdigit()):
                run_on_ui_thread(lambda: BulletinHelper.show_error(
                    self._get_string('bmi_must_be_integers'),
                    get_last_fragment()
                ))
                return

            weight = int(weight_str)
            height = int(height_str)
            
            bmi = weight / ((height / 100) ** 2)
            bmi_formatted = round(bmi, 2)
            
            msg = (
                f"Your BMI: *{bmi_formatted}*\n"
                f"Height: *{height}*cm\n"
                f"Weight: *{weight}*kg"
            )
            self._send_markdown_response(msg, peer_id)

        except ValueError as ve:
             run_on_ui_thread(lambda: BulletinHelper.show_error(str(ve), get_last_fragment()))
        except Exception as e:
             run_on_ui_thread(lambda: BulletinHelper.show_error(f"{self._get_string('bmi_error')}: {e}", get_last_fragment()))

    def _send_markdown_response(self, text, peer_id):
        try:
            parsed = parse_markdown(text)
            params = {
                "peer": peer_id,
                "message": parsed.text,
                "entities": [e.to_tlrpc_object() for e in parsed.entities]
            }
            run_on_ui_thread(lambda: send_message(params))
        except Exception as e:
            log(f"{self._get_string('send_error')}: {e}")

    def on_send_message_hook(self, account, params):
        if not hasattr(params, "message") or not isinstance(params.message, str):
            return HookResult()

        msg = params.message
        calc_cmd_pref = self.get_setting("cmd_calc", ".calc")
        bmi_cmd_pref = self.get_setting("cmd_bmi", ".calcbmi")

        if msg.lower().startswith(calc_cmd_pref.lower() + " "):
            expr = msg[len(calc_cmd_pref):].strip()
            if not expr:
                BulletinHelper.show_error(self._get_string('enter_expression'), get_last_fragment())
                return HookResult(strategy=HookStrategy.CANCEL)
            
            run_on_queue(lambda: self._calculate_logic(expr, params.peer))
            return HookResult(strategy=HookStrategy.CANCEL)

        elif msg.lower().startswith(bmi_cmd_pref.lower() + " "):
            args = msg[len(bmi_cmd_pref):].strip()
            run_on_queue(lambda: self._calculate_bmi_logic(args, params.peer))
            return HookResult(strategy=HookStrategy.CANCEL)

        return HookResult()