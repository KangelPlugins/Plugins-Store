import requests
from typing import Any, Optional, List, Tuple
from android_utils import log, run_on_ui_thread
from base_plugin import BasePlugin, HookResult, HookStrategy
from client_utils import run_on_queue, send_message, get_last_fragment
from markdown_utils import parse_markdown
from ui.settings import Header, Input, Divider, Switch, Text
from ui.bulletin import BulletinHelper
from org.telegram.tgnet import TLRPC

__id__ = "shareui_plnf"
__name__ = "Pluginify search"
__description__ = """Search plugins in the @plnufy. 

• plnf search [name] — srch plugin
• plnf update — updates the list of plugins
"""
__author__ = "@plnufy"
__version__ = "0.5.0"
__icon__ = "plnfy/0"
__min_version__ = "12.2.10"

PLNF_API_URL = "https://raw.githubusercontent.com/shareui/plugins/main/assets/plnf.json"

#cmd cfg
DEFAULT_SEARCH_COMMAND = "plnf search"
DEFAULT_UPDATE_COMMAND = "plnf update"

# autoupdate cfg
AUTO_UPDATE_INTERVALS = {
    "Never": 0,
    "30m": 30 * 60 * 1000,
    "1h": 60 * 60 * 1000,
    "2h": 2 * 60 * 60 * 1000,
    "6h": 6 * 60 * 60 * 1000,
    "1d": 24 * 60 * 60 * 1000
}

def fetchPluginDatabase():
    try:
        response = requests.get(PLNF_API_URL, timeout=10)
        if response.status_code != 200:
            log(f"Failed to fetch plugin database (status code: {response.status_code})")
            return None
        
        jsonData = response.json()
        return jsonData.get("plugins", {})
    except Exception as e:
        log(f"Plugin database fetch error: {str(e)}")
        import traceback
        log(f"Traceback: {traceback.format_exc()}")
        return None

def findPluginsByPartialMatch(pluginsDatabase, searchQuery):
    matches = []
    searchLower = searchQuery.lower()
    
    for pluginKey, pluginData in pluginsDatabase.items():
        pluginName = pluginData.get("displayName", pluginKey).lower()
        
        if searchLower in pluginKey.lower():
            matches.append((pluginKey, pluginData, 2))
        elif searchLower in pluginName:
            matches.append((pluginKey, pluginData, 1))
    
    matches.sort(key=lambda x: x[2], reverse=True)
    return [(key, data) for key, data, _ in matches]

def formatPluginResult(pluginKey, pluginData, settings):
    try:
        parts = []
        
        show_name = settings.get("show_name", True)
        show_description = settings.get("show_description", True)
        show_usage = settings.get("show_usage", True)
        show_links = settings.get("show_links", True)
        
        name = pluginData.get("displayName", pluginKey)
        description = pluginData.get("description", "No description")
        usage = pluginData.get("usage", "No usage info")
        link = pluginData.get("link", "")
        
        if show_name:
            parts.append(f"*Name*: {name}")
        
        if show_description:
            parts.append(f"*Description*: {description}")
        
        if show_usage:
            parts.append(f"*Usage*: {usage}")
        
        if show_links and link:
            parts.append(f"\n[Open link]({link}) • `{link}`")
        
        markdownText = "\n".join(parts)
        
        return markdownText
    except Exception as e:
        log(f"Error formatting plugin result: {str(e)}")
        return f"Error formatting result: {str(e)}"

def formatMultipleResults(results, settings):
    try:
        resultParts = []
        show_description = settings.get("show_description", True)
        show_usage = settings.get("show_usage", True)
        
        for i, (pluginKey, pluginData) in enumerate(results, 1):
            name = pluginData.get("displayName", pluginKey)
            description = pluginData.get("description", "No description")
            usage = pluginData.get("usage", "No usage info")
            
            part = f"{i}. *{name}*"
            if show_description:
                part += f"\n   {description}"
            if show_usage:
                part += f"\n   Usage: `{usage}`"
            
            resultParts.append(part)
        
        return "\n\n".join(resultParts)
    except Exception as e:
        log(f"Error formatting multiple results: {str(e)}")
        return f"Error formatting results: {str(e)}"

class PluginFinderPlugin(BasePlugin):
    def __init__(self):
        super().__init__()
        self.searchCommand = DEFAULT_SEARCH_COMMAND
        self.updateCommand = DEFAULT_UPDATE_COMMAND
        self.cachedPluginsDatabase = None
        self.autoUpdateTimer = None

    def on_plugin_load(self):
        self.add_on_send_message_hook()
        self.searchCommand = self.get_setting("search_command", DEFAULT_SEARCH_COMMAND)
        self.updateCommand = self.get_setting("update_command", DEFAULT_UPDATE_COMMAND)
        
        run_on_queue(lambda: self._loadPluginDatabase(showBulletin=False))
        self._startAutoUpdate()

    def on_plugin_unload(self):
        self._stopAutoUpdate()

    def _startAutoUpdate(self):
        from client_utils import run_on_queue, PLUGINS_QUEUE
        
        self._stopAutoUpdate()
        
        autoUpdateSetting = self.get_setting("auto_update", 0)
        intervalKeys = list(AUTO_UPDATE_INTERVALS.keys())
        
        if autoUpdateSetting >= len(intervalKeys):
            return
        
        intervalKey = intervalKeys[autoUpdateSetting]
        intervalMs = AUTO_UPDATE_INTERVALS[intervalKey]
        
        if intervalMs <= 0:
            return
        
        def _scheduleUpdate():
            self._loadPluginDatabase(showBulletin=False)
            if self.autoUpdateTimer:
                run_on_queue(_scheduleUpdate, PLUGINS_QUEUE, intervalMs)
        
        self.autoUpdateTimer = True
        run_on_queue(_scheduleUpdate, PLUGINS_QUEUE, intervalMs)

    def _stopAutoUpdate(self):
        self.autoUpdateTimer = None

    def _create_commands_settings(self):
        return [
            Header(text="Commands Settings"),
            Input(
                key="search_command",
                text="Search Command",
                default=DEFAULT_SEARCH_COMMAND,
                icon="msg_search"
            ),
            Input(
                key="update_command",
                text="Update Command",
                default=DEFAULT_UPDATE_COMMAND,
                icon="msg_retry"
            ),
            Divider(text="""• plnf search — looking for a plugin
            flags: -e — precise search
• plnf update — update plugin database""")
        ]

    def _create_other_settings(self):
        from ui.settings import Selector
        
        return [
            Header(text="Other Settings"),
            Selector(
                key="auto_update",
                text="Auto Update",
                default=0,
                items=["Never", "30m", "1h", "2h", "6h", "1d"],
                icon="msg_autodelete"
            )
        ]

    def _create_output_settings(self):
        return [
            Header(text="Output Settings"),
            Switch(
                key="show_name",
                text="Show Name",
                default=True,
                icon="msg_mention"
            ),
            Switch(
                key="show_description",
                text="Show Description",
                default=True,
                icon="msg_info"
            ),
            Switch(
                key="show_usage",
                text="Show Usage",
                default=True,
                icon="msg_help"
            ),
            Switch(
                key="show_links",
                text="Show Links",
                subtext="Display clickable and copyable links",
                default=True,
                icon="msg_link"
            )
        ]

    def create_settings(self):
        return [
            Text(
                text="Commands Settings",
                icon="input_bot1",
                create_sub_fragment=self._create_commands_settings
            ),
            Text(
                text="Other Settings",
                icon="msg_settings",
                create_sub_fragment=self._create_other_settings
            ),
            Text(
                text="Output Settings",
                icon="msg_customize",
                create_sub_fragment=self._create_output_settings
            )
        ]

    def _loadPluginDatabase(self, showBulletin=True):
        try:
            pluginsDatabase = fetchPluginDatabase()
            
            if pluginsDatabase is None:
                def _showErrorBulletin():
                    if showBulletin:
                        currentFragment = get_last_fragment()
                        BulletinHelper.show_error("Failed to update database", currentFragment)
                
                run_on_ui_thread(_showErrorBulletin)
                return False
            
            self.cachedPluginsDatabase = pluginsDatabase
            pluginCount = len(pluginsDatabase)
            
            def _showSuccessBulletin():
                if showBulletin:
                    currentFragment = get_last_fragment()
                    BulletinHelper.show_success(f"Database updated: {pluginCount} plugins", currentFragment)
            
            run_on_ui_thread(_showSuccessBulletin)
            return True
            
        except Exception as e:
            log(f"Error loading plugin database: {str(e)}")
            import traceback
            log(f"Traceback: {traceback.format_exc()}")
            
            def _showErrorBulletin():
                if showBulletin:
                    currentFragment = get_last_fragment()
                    BulletinHelper.show_error(f"Update failed: {str(e)}", currentFragment)
            
            run_on_ui_thread(_showErrorBulletin)
            return False

    def _processPluginUpdate(self):
        self._loadPluginDatabase(showBulletin=True)

    def _processPluginSearch(self, searchQuery: str, exactSearch: bool, peerId: Any, replyToMsg: Any, replyToTopMsg: Any):
        pluginsDatabase = self.cachedPluginsDatabase
        
        if pluginsDatabase is None:
            pluginsDatabase = fetchPluginDatabase()
            
            if pluginsDatabase is None:
                def _showErrorBulletin():
                    currentFragment = get_last_fragment()
                    BulletinHelper.show_error("Failed to load database", currentFragment)
                
                run_on_ui_thread(_showErrorBulletin)
                return
            
            self.cachedPluginsDatabase = pluginsDatabase
        
        settings = {
            "show_name": self.get_setting("show_name", True),
            "show_description": self.get_setting("show_description", True),
            "show_usage": self.get_setting("show_usage", True),
            "show_links": self.get_setting("show_links", True)
        }
        
        if exactSearch:
            if searchQuery in pluginsDatabase:
                pluginData = pluginsDatabase[searchQuery]
                messageContent = formatPluginResult(searchQuery, pluginData, settings)
            else:
                def _showNothingFoundBulletin():
                    currentFragment = get_last_fragment()
                    BulletinHelper.show_error(f"Plugin {searchQuery} not found", currentFragment)
                
                run_on_ui_thread(_showNothingFoundBulletin)
                return
        else:
            foundMatches = findPluginsByPartialMatch(pluginsDatabase, searchQuery)
            
            if not foundMatches:
                def _showNoMatchesBulletin():
                    currentFragment = get_last_fragment()
                    BulletinHelper.show_error(f"No plugins matching {searchQuery}", currentFragment)
                
                run_on_ui_thread(_showNoMatchesBulletin)
                return
            elif len(foundMatches) == 1:
                pluginKey, pluginData = foundMatches[0]
                messageContent = formatPluginResult(pluginKey, pluginData, settings)
            else:
                maxResults = 5
                displayResults = foundMatches[:maxResults]
                messageContent = f"Found {len(foundMatches)} plugins:\n\n"
                messageContent += formatMultipleResults(displayResults, settings)
                
                if len(foundMatches) > maxResults:
                    messageContent += f"\n\n... and {len(foundMatches) - maxResults} more"
        
        try:
            parsedMessage = parse_markdown(messageContent)
            messageParams = {
                "message": parsedMessage.text,
                "peer": peerId,
                "entities": [],
                "searchLinks": False
            }
            
            for rawEntity in parsedMessage.entities:
                tlrpcEntity = rawEntity.to_tlrpc_object()
                messageParams["entities"].append(tlrpcEntity)
            
            if replyToMsg:
                messageParams["replyToMsg"] = replyToMsg
            if replyToTopMsg:
                messageParams["replyToTopMsg"] = replyToTopMsg
            
            def _sendFormattedMessage():
                send_message(messageParams)
            
            run_on_ui_thread(_sendFormattedMessage)
        except Exception as e:
            log(f"Error parsing markdown: {str(e)}")
            import traceback
            log(f"Traceback: {traceback.format_exc()}")
            
            def _showParseErrorBulletin():
                currentFragment = get_last_fragment()
                BulletinHelper.show_error("Failed to format message", currentFragment)
            
            run_on_ui_thread(_showParseErrorBulletin)

    def on_send_message_hook(self, account: int, params: Any) -> HookResult:
        if not isinstance(params.message, str):
            return HookResult()
        
        messageText = params.message.strip()
        
        updateCommandPrefix = self.updateCommand.strip()
        if messageText == updateCommandPrefix or messageText.startswith(updateCommandPrefix + " "):
            run_on_queue(lambda: self._processPluginUpdate())
            return HookResult(strategy=HookStrategy.CANCEL)
        
        searchCommandPrefix = self.searchCommand.strip()
        if not messageText.startswith(searchCommandPrefix):
            return HookResult()
        
        try:
            exactSearch = "-e" in messageText
            
            if exactSearch:
                messageText = messageText.replace("-e", "").strip()
            
            messageParts = messageText.split(" ", 2)
            
            if len(messageParts) < 3:
                params.message = f"Usage: {searchCommandPrefix} [plugin_name] [-e]"
                return HookResult(strategy=HookStrategy.MODIFY, params=params)
            
            searchQuery = messageParts[2].strip()
            if not searchQuery:
                params.message = f"Usage: {searchCommandPrefix} [plugin_name] [-e]"
                return HookResult(strategy=HookStrategy.MODIFY, params=params)
            
            peerId = params.peer
            replyToMsg = getattr(params, "replyToMsg", None)
            replyToTopMsg = getattr(params, "replyToTopMsg", None)
            
            run_on_queue(lambda: self._processPluginSearch(searchQuery, exactSearch, peerId, replyToMsg, replyToTopMsg))
            
            return HookResult(strategy=HookStrategy.CANCEL)
        except Exception as e:
            log(f"Plugin finder error: {str(e)}")
            params.message = f"Error: {str(e)}"
            return HookResult(strategy=HookStrategy.MODIFY, params=params)