import subprocess
import time
from typing import Any
from android_utils import run_on_ui_thread
from base_plugin import BasePlugin, HookResult, HookStrategy
from client_utils import run_on_queue, send_message
from org.telegram.tgnet import TLRPC
from ui.settings import Header, Switch
from org.telegram.messenger import LocaleController

__id__ = "shareui_sh"
__name__ = "Shell Executor"
__description__ = "Execute shell commands, subprocesses, and Junis [.sh/.sproc/.jn]"
__author__ = "@shareui"
__version__ = "1.0.0"
__icon__ = "plugin232/5"
__min_version__ = "12.1.1"

class ShellExecPlugin(BasePlugin):
    def on_plugin_load(self):
        self.add_on_send_message_hook()

    def _create_blockquote_entity(self, offset: int, length: int) -> TLRPC.MessageEntity:
        entity = TLRPC.TL_messageEntityBlockquote()
        entity.offset = offset
        entity.length = length
        return entity

    def _create_bold_entity(self, offset: int, length: int) -> TLRPC.MessageEntity:
        entity = TLRPC.TL_messageEntityBold()
        entity.offset = offset
        entity.length = length
        return entity

    def _format_output(self, shell_name: str, cmd: str, output: str, exec_time: float) -> tuple[str, list]:
        lines = []
        entities = []
        current_offset = 0
        
        use_quotes = self.get_setting("use_quotes", True)
        show_input = self.get_setting("show_input", True)
        
        header = f"{shell_name} [{exec_time:.2f}s]"
        lines.append(header)
        shell_name_len = len(shell_name)
        entities.append(self._create_bold_entity(current_offset, shell_name_len))
        current_offset += len(header) + 2
        lines.append("")
        
        if show_input:
            input_label = "Input"
            lines.append(input_label)
            entities.append(self._create_bold_entity(current_offset, len(input_label)))
            current_offset += len(input_label) + 1
            
            cmd_text = cmd
            lines.append(cmd_text)
            if use_quotes:
                entities.append(self._create_blockquote_entity(current_offset, len(cmd_text)))
            current_offset += len(cmd_text) + 2
            lines.append("")
        
        output_label = "Stdout"
        lines.append(output_label)
        entities.append(self._create_bold_entity(current_offset, len(output_label)))
        current_offset += len(output_label) + 1
        
        output_text = output if output else "(empty)"
        lines.append(output_text)
        if use_quotes:
            entities.append(self._create_blockquote_entity(current_offset, len(output_text)))
        
        final_text = "\n".join(lines)
        return final_text, entities

    def _execute_command(self, cmd: str, shell_type: str, peer_id: Any):
        start_time = time.time()
        
        try:
            # shell
            if shell_type == "sh":
                result = subprocess.run(
                    ["sh", "-c", cmd],
                    capture_output=True,
                    text=True,
                    timeout=30
                )
                shell_name = "Bash"
            # subproc
            elif shell_type == "sproc":
                result = subprocess.run(
                    cmd.split(),
                    capture_output=True,
                    text=True,
                    timeout=30
                )
                shell_name = "Subprocess"
            # junis
            elif shell_type == "jn":
                result = subprocess.run(
                    ["sh", "-c", cmd],
                    capture_output=True,
                    text=True,
                    timeout=30,
                    shell=True
                )
                shell_name = "Junis"
            else:
                raise ValueError(f"Unknown shell type: {shell_type}")
            
            exec_time = time.time() - start_time
            
            output = result.stdout.strip()
            if result.stderr:
                output += f"\n[stderr]\n{result.stderr.strip()}"
            
            if result.returncode != 0:
                output += f"\n[exit code: {result.returncode}]"
            
            message_text, entities = self._format_output(
                shell_name, cmd, output, exec_time
            )
            
        except subprocess.TimeoutExpired:
            exec_time = time.time() - start_time
            message_text, entities = self._format_output(
                shell_name, cmd, "⏱ Command timeout (30s)", exec_time
            )
        except Exception as e:
            exec_time = time.time() - start_time
            message_text, entities = self._format_output(
                shell_name if 'shell_name' in locals() else "Unknown",
                cmd,
                f"Error: {str(e)}",
                exec_time
            )
        
        def _send_result():
            message_params = {
                "message": message_text,
                "peer": peer_id,
                "entities": entities
            }
            send_message(message_params)
        
        run_on_ui_thread(_send_result)

    def on_send_message_hook(self, account: int, params: Any) -> HookResult:
        if not isinstance(params.message, str):
            return HookResult()
        
        message = params.message.strip()
        
        if message.startswith(".sh "):
            cmd = message[4:].strip()
            if not cmd:
                params.message = "Usage: .sh [command]"
                return HookResult(strategy=HookStrategy.MODIFY, params=params)
            
            peer_id = params.peer
            run_on_queue(lambda: self._execute_command(cmd, "sh", peer_id))
            return HookResult(strategy=HookStrategy.CANCEL)
        
        elif message.startswith(".sproc "):
            cmd = message[7:].strip()
            if not cmd:
                params.message = "Usage: .sproc [command]"
                return HookResult(strategy=HookStrategy.MODIFY, params=params)
            
            peer_id = params.peer
            run_on_queue(lambda: self._execute_command(cmd, "sproc", peer_id))
            return HookResult(strategy=HookStrategy.CANCEL)
        
        elif message.startswith(".jn "):
            cmd = message[4:].strip()
            if not cmd:
                params.message = "Usage: .jn [command]"
                return HookResult(strategy=HookStrategy.MODIFY, params=params)
            
            peer_id = params.peer
            run_on_queue(lambda: self._execute_command(cmd, "jn", peer_id))
            return HookResult(strategy=HookStrategy.CANCEL)
        
        return HookResult()

    def create_settings(self):
        lang = LocaleController.getInstance().getCurrentLocale().getLanguage()
        
        strings = {
            'ru': {
                'settings_title': 'Настройки вывода',
                'use_quotes': 'Использовать цитаты',
                'use_quotes_desc': 'Отображать команду и вывод в блоках цитирования',
                'show_input': 'Отправлять ввод',
                'show_input_desc': 'Включать секцию Input с командой в сообщение'
            },
            'en': {
                'settings_title': 'Output Settings',
                'use_quotes': 'Use quotes',
                'use_quotes_desc': 'Display command and output in blockquote format',
                'show_input': 'Send input',
                'show_input_desc': 'Include Input section with command in message'
            }
        }
        
        lang_key = 'ru' if lang.startswith('ru') else 'en'
        s = strings[lang_key]
        
        return [
            Header(text=s['settings_title']),
            Switch(
                key="use_quotes",
                text=s['use_quotes'],
                default=True,
                subtext=s['use_quotes_desc'],
                icon="msg_quote"
            ),
            Switch(
                key="show_input",
                text=s['show_input'],
                default=True,
                subtext=s['show_input_desc'],
                icon="msg_input"
            )
        ]