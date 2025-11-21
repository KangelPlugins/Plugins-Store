# MIT License
# Copyright (c) 2025 shareui
# Permission is hereby granted, free of charge, to any person obtaining a copy
# of this software and associated documentation files (the "Software"), to deal
# in the Software without restriction, including without limitation the rights
# to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
# copies of the Software, and to permit persons to whom the Software is
# furnished to do so, subject to the following conditions:
# The above copyright notice and this permission notice shall be included in all
# copies or substantial portions of the Software.
# THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
# IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
# FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
# AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
# LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
# OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
# SOFTWARE.

import requests
import traceback
from datetime import datetime
from base_plugin import BasePlugin, HookResult, HookStrategy
from ui.settings import Header, Divider, Switch, Selector, Input, Text
from java.util import Locale
from org.telegram.tgnet import TLRPC
from client_utils import run_on_queue, send_message, get_last_fragment
from android_utils import run_on_ui_thread, log
from ui.bulletin import BulletinHelper

__id__ = "shareui_exteragit"
__name__ = "exteraGit"
__version__ = "1.0.0 realease"
__author__ = "@shareui"
__description__ = "Search GitHub and GitLab resources. Commands in the plugin settings."
__icon__ = "plugin232/10"
__min_version__ = "12.0.1"

GITHUB_API_URL = "https://api.github.com"
GITLAB_API_URL = "https://gitlab.com/api/v4"

class Locales:
    ru = {
        "NOT_FOUND": "Ничего не найдено :(",
        "ERROR": "Ошибка при поиске: ",
        "SEARCHING": "Идет поиск...",
        "GITHUB_SETTINGS": "Настройки GitHub",
        "GITLAB_SETTINGS": "Настройки GitLab",
        "COMMANDS_SETTINGS": "Настройки команд",
        "SHOW_ORGS": "Показывать организации",
        "SHOW_ORGS_SUB": "Отображение организаций пользователя",
        "SHOW_GROUPS": "Показывать группы",
        "SHOW_GROUPS_SUB": "Отображение групп пользователя",
        "REPO_SORT": "Сортировка репозиториев",
        "PROJECT_SORT": "Сортировка проектов",
        "SORT_DATE": "По дате",
        "SORT_STARS": "По звездам",
        "SHOW_LICENSE": "Показывать лицензию",
        "SHOW_LICENSE_SUB": "Отображение лицензии в результатах",
        "COMMANDS": """Смд по умолчанию:
    exg gitsrc - search GitHub repos
    exg gituser - search GitHub users
    exg gitorg - search GitHub orgs
    exg glsrc - search GitLab projects
    exg gluser - search GitLab users
    exg glgroup - search GitLab groups""","COMMAND_PREFIX": "Префикс",
        "GITHUB_REPO_CMD": "GitHub репо",
        "GITHUB_USER_CMD": "GitHub юзер",
        "GITHUB_ORG_CMD": "GitHub орг",
        "GITLAB_PROJECT_CMD": "GitLab проект",
        "GITLAB_USER_CMD": "GitLab юзер",
        "GITLAB_GROUP_CMD": "GitLab группа",
        "COMMANDS_INFO": "Настройте команды. Префикс добавляется."
    }
    en = {
        "NOT_FOUND": "Nothing found :(",
        "ERROR": "Search error: ",
        "SEARCHING": "Searching...",
        "GITHUB_SETTINGS": "GitHub Settings",
        "GITLAB_SETTINGS": "GitLab Settings",
        "COMMANDS_SETTINGS": "Commands Settings",
        "SHOW_ORGS": "Show organizations",
        "SHOW_ORGS_SUB": "Display user organizations",
        "SHOW_GROUPS": "Show groups",
        "SHOW_GROUPS_SUB": "Display user groups",
        "REPO_SORT": "Repository sorting",
        "PROJECT_SORT": "Project sorting",
        "SORT_DATE": "By date",
        "SORT_STARS": "By stars",
        "SHOW_LICENSE": "Show license",
        "SHOW_LICENSE_SUB": "Display license in results",
        "COMMANDS": """Default commands
    exg gitsrc - search GitHub repos
    exg gituser - search GitHub users
    exg gitorg - search GitHub orgs
    exg glsrc - search GitLab projects
    exg gluser - search GitLab users
    exg glgroup - search GitLab groups""",
        "COMMAND_PREFIX": "Command prefix",
        "GITHUB_REPO_CMD": "GitHub repo search cmd",
        "GITHUB_USER_CMD": "GitHub user search cmd",
        "GITHUB_ORG_CMD": "GitHub org search cmd",
        "GITLAB_PROJECT_CMD": "GitLab project search cmd",
        "GITLAB_USER_CMD": "GitLab user search cmd",
        "GITLAB_GROUP_CMD": "GitLab group search cmd",
        "COMMANDS_INFO": "Configure search commands."
    }
    default = en

def get_locale():
    lang = Locale.getDefault().getLanguage()
    return getattr(Locales, lang, Locales.default)

class ExteraGitPlugin(BasePlugin):
    def __init__(self):
        super().__init__()

    def on_plugin_load(self):
        self.add_on_send_message_hook()
        log(f"[{__name__}] Loaded")

    def _create_github_settings(self):
        loc = get_locale()
        return [
            Header(text=loc["GITHUB_SETTINGS"]),
            Switch(
                key="github_show_orgs",
                text=loc["SHOW_ORGS"],
                subtext=loc["SHOW_ORGS_SUB"],
                default=True,
                icon="msg_contacts"
            ),
            Selector(
                key="github_repo_sort",
                text=loc["REPO_SORT"],
                items=[loc["SORT_DATE"], loc["SORT_STARS"]],
                default=0,
                icon="tabs_reorder_solar"
            ),
            Switch(
                key="github_show_license",
                text=loc["SHOW_LICENSE"],
                subtext=loc["SHOW_LICENSE_SUB"],
                default=True,
                icon="msg_info"
            ),
        ]

    def _create_gitlab_settings(self):
        loc = get_locale()
        return [
            Header(text=loc["GITLAB_SETTINGS"]),
            Switch(
                key="gitlab_show_groups",
                text=loc["SHOW_GROUPS"],
                subtext=loc["SHOW_GROUPS_SUB"],
                default=True,
                icon="msg_group"
            ),
            Selector(
                key="gitlab_project_sort",
                text=loc["PROJECT_SORT"],
                items=[loc["SORT_DATE"], loc["SORT_STARS"]],
                default=0,
                icon="tabs_reorder_solar"
            ),
            Switch(
                key="gitlab_show_license",
                text=loc["SHOW_LICENSE"],
                subtext=loc["SHOW_LICENSE_SUB"],
                default=True,
                icon="msg_info"
            ),
        ]

    def _create_commands_settings(self):
        loc = get_locale()
        return [
            Header(text=loc["COMMANDS_SETTINGS"]),
            Input(
                key="command_prefix",
                text=loc["COMMAND_PREFIX"],
                default="exg",
                icon="msg_text"
            ),
            Input(
                key="github_repo_cmd",
                text=loc["GITHUB_REPO_CMD"],
                default="gitsrc",
                icon="msg_link"
            ),
            Input(
                key="github_user_cmd",
                text=loc["GITHUB_USER_CMD"],
                default="gituser",
                icon="msg_contacts"
            ),
            Input(
                key="github_org_cmd",
                text=loc["GITHUB_ORG_CMD"],
                default="gitorg",
                icon="msg_groups"
            ),
            Input(
                key="gitlab_project_cmd",
                text=loc["GITLAB_PROJECT_CMD"],
                default="glsrc",
                icon="msg_link"
            ),
            Input(
                key="gitlab_user_cmd",
                text=loc["GITLAB_USER_CMD"],
                default="gluser",
                icon="msg_contacts"
            ),
            Input(
                key="gitlab_group_cmd",
                text=loc["GITLAB_GROUP_CMD"],
                default="glgroup",
                icon="msg_groups"
            ),
            Divider(text=loc["COMMANDS_INFO"])
        ]

    def create_settings(self):
        loc = get_locale()
        return [
            Text(
                text=loc["GITHUB_SETTINGS"],
                icon="msg_link",
                create_sub_fragment=self._create_github_settings
            ),
            Text(
                text=loc["GITLAB_SETTINGS"],
                icon="msg_link",
                create_sub_fragment=self._create_gitlab_settings
            ),
            Text(
                text=loc["COMMANDS_SETTINGS"],
                icon="msg_text",
                create_sub_fragment=self._create_commands_settings
            ),
            Divider(text=loc["COMMANDS"])
        ]

    def on_send_message_hook(self, account, params):
        if not hasattr(params, "message") or not isinstance(params.message, str):
            return HookResult()

        msg = params.message.strip()
        prefix = self.get_setting("command_prefix", "exg")
        
        if not msg.lower().startswith(f"{prefix} "):
            return HookResult()

        loc = get_locale()
        
        github_repo_cmd = self.get_setting("github_repo_cmd", "gitsrc")
        github_user_cmd = self.get_setting("github_user_cmd", "gituser")
        github_org_cmd = self.get_setting("github_org_cmd", "gitorg")
        gitlab_project_cmd = self.get_setting("gitlab_project_cmd", "glsrc")
        gitlab_user_cmd = self.get_setting("gitlab_user_cmd", "gluser")
        gitlab_group_cmd = self.get_setting("gitlab_group_cmd", "glgroup")

        command_map = {
            f"{prefix} {github_repo_cmd}": ("github_repo", len(f"{prefix} {github_repo_cmd}"), f"Example: {prefix} {github_repo_cmd} exteraGram"),
            f"{prefix} {github_user_cmd}": ("github_user", len(f"{prefix} {github_user_cmd}"), f"Example: {prefix} {github_user_cmd} torvalds"),
            f"{prefix} {github_org_cmd}": ("github_org", len(f"{prefix} {github_org_cmd}"), f"Example: {prefix} {github_org_cmd} exteragram"),
            f"{prefix} {gitlab_project_cmd}": ("gitlab_project", len(f"{prefix} {gitlab_project_cmd}"), f"Example: {prefix} {gitlab_project_cmd} exteraGram"),
            f"{prefix} {gitlab_user_cmd}": ("gitlab_user", len(f"{prefix} {gitlab_user_cmd}"), f"Example: {prefix} {gitlab_user_cmd} username"),
            f"{prefix} {gitlab_group_cmd}": ("gitlab_group", len(f"{prefix} {gitlab_group_cmd}"), f"Example: {prefix} {gitlab_group_cmd} community")
        }

        for cmd, (search_type, length, usage_msg) in command_map.items():
            if msg.lower().startswith(cmd):
                query = msg[length:].strip()
                if not query:
                    send_message({"peer": params.peer, "message": usage_msg, "replyToMsg": getattr(params, "replyToMsg", None)})
                    return HookResult(strategy=HookStrategy.CANCEL)

                run_on_ui_thread(lambda: BulletinHelper.show_info(loc["SEARCHING"], get_last_fragment()))
                run_on_queue(lambda: self._process_search(params, query, search_type))
                return HookResult(strategy=HookStrategy.CANCEL)

        return HookResult()

    def _process_search(self, params, query, search_type):
        loc = get_locale()
        try:
            if search_type.startswith("github_"):
                result = self._search_github(query, search_type)
            elif search_type.startswith("gitlab_"):
                result = self._search_gitlab(query, search_type)
            else:
                result = {"text": loc["NOT_FOUND"]}

            run_on_ui_thread(lambda: self._send_result(params, result))
        except requests.exceptions.RequestException as e:
            log(f"[{__name__}] Network error: {e}\n{traceback.format_exc()}")
            run_on_ui_thread(lambda: BulletinHelper.show_error(loc["ERROR"] + str(e), get_last_fragment()))
        except Exception as e:
            log(f"[{__name__}] Error: {e}\n{traceback.format_exc()}")
            run_on_ui_thread(lambda: BulletinHelper.show_error(loc["ERROR"] + str(e), get_last_fragment()))

    def _send_result(self, params, result):
        loc = get_locale()
        message_data = {
            "peer": params.peer,
            "replyToMsg": getattr(params, "replyToMsg", None)
        }

        if isinstance(result, dict) and "text" in result:
            message_data["message"] = result.get("text", loc["NOT_FOUND"])
            message_data["entities"] = result.get("entities", [])
        else:
            message_data["message"] = str(result)

        if not message_data["message"] and message_data.get("entities"):
            message_data["message"] = " "

        send_message(message_data)

    def _format_date(self, date_str):
        if not date_str:
            return "N/A"
        try:
            dt = datetime.fromisoformat(date_str.replace('Z', '+00:00'))
            return dt.strftime('%d.%m.%Y')
        except:
            return date_str[:10]

    def _search_github(self, query, search_type):
        loc = get_locale()
        headers = {"Accept": "application/vnd.github.v3+json"}

        if search_type == "github_repo":
            return self._github_search_repo(query, headers, loc)
        elif search_type == "github_user":
            return self._github_search_user(query, headers, loc)
        elif search_type == "github_org":
            return self._github_search_org(query, headers, loc)

        return {"text": loc["NOT_FOUND"]}
    
    def _github_search_repo(self, query, headers, loc):
        r = requests.get(f"{GITHUB_API_URL}/search/repositories", params={"q": query}, headers=headers, timeout=10)
        if r.status_code != 200 or not r.json().get("items"):
            return {"text": loc["NOT_FOUND"]}

        repo = r.json()["items"][0]
        name = repo.get('full_name', '')
        url = repo.get('html_url', '')
        owner = repo.get('owner', {})
        owner_login = owner.get('login', '')
        owner_url = owner.get('html_url', '')
        owner_type = owner.get('type', '')
        stars = repo.get('stargazers_count', 0)
        forks = repo.get('forks_count', 0)
        issues = repo.get('open_issues_count', 0)
        updated = self._format_date(repo.get('updated_at', ''))

        license_info = repo.get('license')
        license_name = "None"
        if license_info:
            license_name = license_info.get('spdx_id') or license_info.get('name') or "None"

        author_type = "Organization" if owner_type == "Organization" else "Author"
        message_text = (
            f"[{name}]\n"
            f"{author_type}: [{owner_login}]\n"
            f"Last update: {updated}\n"
            f"Stars: {stars}★\n"
            f"Forks: {forks}\n"
            f"Issues: {issues}"
        )

        if self.get_setting("github_show_license", True):
            message_text += f"\nLicense: {license_name}"

        entities = []
        repo_entity = TLRPC.TL_messageEntityTextUrl()
        repo_entity.offset = 1
        repo_entity.length = len(name.encode('utf_16_le')) // 2
        repo_entity.url = url
        entities.append(repo_entity)

        author_start = message_text.find(f"[{owner_login}]")
        author_entity = TLRPC.TL_messageEntityTextUrl()
        author_entity.offset = len(message_text[:author_start + 1].encode('utf_16_le')) // 2
        author_entity.length = len(owner_login.encode('utf_16_le')) // 2
        author_entity.url = owner_url
        entities.append(author_entity)

        return {"text": message_text, "entities": entities}

    def _github_search_user(self, query, headers, loc):
        r = requests.get(f"{GITHUB_API_URL}/search/users", params={"q": query}, headers=headers, timeout=10)
        if r.status_code != 200 or not r.json().get("items"):
            return {"text": loc["NOT_FOUND"]}

        user_login = r.json()["items"][0].get('login', '')
        user_resp = requests.get(f"{GITHUB_API_URL}/users/{user_login}", headers=headers, timeout=10)
        if user_resp.status_code != 200:
            return {"text": loc["NOT_FOUND"]}

        profile = user_resp.json()
        login = profile.get('login', '')
        url = profile.get('html_url', '')
        bio = profile.get('bio', '') or ''

        message_text = f"[{login}]\n"
        entities = []

        username_entity = TLRPC.TL_messageEntityTextUrl()
        username_entity.offset = 1
        username_entity.length = len(login.encode('utf_16_le')) // 2
        username_entity.url = url
        entities.append(username_entity)

        if bio:
            message_text += f"> {bio}\n\n"

        if self.get_setting("github_show_orgs", True):
            orgs_resp = requests.get(f"{GITHUB_API_URL}/users/{login}/orgs", headers=headers, timeout=10)
            if orgs_resp.status_code == 200 and orgs_resp.json():
                message_text += "Organizations:\n"
                for org in orgs_resp.json():
                    org_name = org.get('login', '')
                    if not org_name:
                        continue
                    org_url = f"https://github.com/{org_name}"
                    org_start = len(message_text)
                    message_text += f"[{org_name}]\n"
                    org_entity = TLRPC.TL_messageEntityTextUrl()
                    org_entity.offset = len(message_text[:org_start + 1].encode('utf_16_le')) // 2
                    org_entity.length = len(org_name.encode('utf_16_le')) // 2
                    org_entity.url = org_url
                    entities.append(org_entity)
                message_text += "\n"

        message_text += "Repositories:\n"
        sort_order = self.get_setting("github_repo_sort", 0)
        repo_list = []

        if sort_order == 1:
            repos_resp = requests.get(
                f"{GITHUB_API_URL}/search/repositories",
                headers=headers,
                timeout=10,
                params={"q": f"user:{login}", "sort": "stars", "order": "desc", "per_page": 5}
            )
            if repos_resp.status_code == 200 and repos_resp.json().get("items"):
                repo_list = repos_resp.json().get("items")
        else:
            repos_resp = requests.get(
                f"{GITHUB_API_URL}/users/{login}/repos",
                headers=headers,
                timeout=10,
                params={"per_page": 5, "sort": "updated"}
            )
            if repos_resp.status_code == 200 and repos_resp.json():
                repo_list = repos_resp.json()

        if repo_list:
            for repo in repo_list:
                repo_name = repo.get('name', '')
                repo_url = repo.get('html_url', '')
                stars = repo.get('stargazers_count', 0)
                updated = self._format_date(repo.get('updated_at', ''))
                repo_start = len(message_text)
                message_text += f"[{repo_name}]\n[{stars}★] [{updated}]\n"
                repo_entity = TLRPC.TL_messageEntityTextUrl()
                repo_entity.offset = len(message_text[:repo_start + 1].encode('utf_16_le')) // 2
                repo_entity.length = len(repo_name.encode('utf_16_le')) // 2
                repo_entity.url = repo_url
                entities.append(repo_entity)
        else:
            message_text += f"{loc['NOT_FOUND']}\n"

        return {"text": message_text.strip(), "entities": entities}

    def _github_search_org(self, query, headers, loc):
        r = requests.get(f"{GITHUB_API_URL}/orgs/{query}", headers=headers, timeout=10)
        if r.status_code != 200:
            return {"text": loc["NOT_FOUND"]}

        org = r.json()
        login = org.get('login', '')
        url = org.get('html_url', '')
        desc = org.get('description', '') or ''
        public_repos = org.get('public_repos', 0)
        location = org.get('location', '') or None
        followers = org.get('followers', 0)
        email = org.get('email', '') or None
        blog = org.get('blog', '') or None

        message_text = f"[{login}]\n"
        entities = []

        org_entity = TLRPC.TL_messageEntityTextUrl()
        org_entity.offset = 1
        org_entity.length = len(login.encode('utf_16_le')) // 2
        org_entity.url = url
        entities.append(org_entity)

        if desc:
            message_text += f"> {desc}\n\n"

        message_text += f"Public repos: {public_repos}\n"
        if location:
            message_text += f"Location: {location}\n"
        if followers is not None:
            message_text += f"Followers: {followers}\n"
        if email:
            message_text += f"Mail: {email}\n"
        if blog:
            blog_start = len(message_text)
            message_text += "Blog: [link]\n"
            blog_entity = TLRPC.TL_messageEntityTextUrl()
            blog_entity.offset = len(message_text[:blog_start + len('Blog: [')].encode('utf_16_le')) // 2
            blog_entity.length = len("link".encode('utf_16_le')) // 2
            blog_entity.url = blog if blog.startswith('http') else 'http://' + blog
            entities.append(blog_entity)

        sort_order = self.get_setting("github_repo_sort", 0)
        repo_list = []

        if sort_order == 1:
            repos_resp = requests.get(
                f"{GITHUB_API_URL}/search/repositories",
                headers=headers,
                timeout=10,
                params={"q": f"org:{query}", "sort": "stars", "order": "desc", "per_page": 5}
            )
            if repos_resp.status_code == 200 and repos_resp.json().get("items"):
                repo_list = repos_resp.json().get("items")
        else:
            repos_resp = requests.get(
                f"{GITHUB_API_URL}/orgs/{query}/repos",
                headers=headers,
                timeout=10,
                params={"per_page": 5, "sort": "updated"}
            )
            if repos_resp.status_code == 200 and repos_resp.json():
                repo_list = repos_resp.json()

        if repo_list:
            message_text += "\nTop repositories:\n"
            for repo in repo_list:
                repo_name = repo.get('name', '')
                repo_url = repo.get('html_url', '')
                stars = repo.get('stargazers_count', 0)
                updated = self._format_date(repo.get('updated_at', ''))
                repo_start = len(message_text)
                message_text += f"[{repo_name}]\n[{stars}★] [{updated}]\n"
                repo_entity = TLRPC.TL_messageEntityTextUrl()
                repo_entity.offset = len(message_text[:repo_start + 1].encode('utf_16_le')) // 2
                repo_entity.length = len(repo_name.encode('utf_16_le')) // 2
                repo_entity.url = repo_url
                entities.append(repo_entity)

        return {"text": message_text.strip(), "entities": entities}

    def _search_gitlab(self, query, search_type):
        loc = get_locale()
        headers = {"User-Agent": "exteraGram-exteraGit/2.0"}

        if search_type == "gitlab_project":
            return self._gitlab_search_project(query, headers, loc)
        elif search_type == "gitlab_user":
            return self._gitlab_search_user(query, headers, loc)
        elif search_type == "gitlab_group":
            return self._gitlab_search_group(query, headers, loc)

        return {"text": loc["NOT_FOUND"]}
        
    def _gitlab_search_project(self, query, headers, loc):
        sort_order = self.get_setting("gitlab_project_sort", 0)
        sort_param = "updated_at" if sort_order == 0 else "star_count"

        params = {"search": query, "order_by": sort_param, "sort": "desc", "per_page": 1}
        r = requests.get(f"{GITLAB_API_URL}/projects", params=params, headers=headers, timeout=10)
        r.raise_for_status()
        projects = r.json()

        if not projects:
            return {"text": loc["NOT_FOUND"]}

        project = projects[0]
        name = project.get('name_with_namespace', loc["NOT_FOUND"])
        url = project.get('web_url', '')
        desc = project.get('description', '')
        stars = project.get('star_count', 0)
        forks = project.get('forks_count', 0)
        last_activity = self._format_date(project.get('last_activity_at', ''))

        license_info = project.get('license', None)
        license_name = "None"
        if license_info and license_info.get('name'):
            license_name = license_info.get('name')

        message_text = (
            f"[{name}]\n"
            f"Stars: {stars}★\n"
            f"Forks: {forks}\n"
            f"Last activity: {last_activity}"
        )

        if desc:
            message_text += f"\nDescription: {desc.strip()}"

        if self.get_setting("gitlab_show_license", True) and license_name != "None":
            message_text += f"\nLicense: {license_name}"

        entities = []
        if url:
            project_entity = TLRPC.TL_messageEntityTextUrl()
            project_entity.offset = 1
            project_entity.length = len(name.encode('utf_16_le')) // 2
            project_entity.url = url
            entities.append(project_entity)

        return {"text": message_text, "entities": entities}

    def _gitlab_search_user(self, query, headers, loc):
        params = {"username": query, "per_page": 1}
        r = requests.get(f"{GITLAB_API_URL}/users", params=params, headers=headers, timeout=10)
        r.raise_for_status()
        users = r.json()

        if not users:
            return {"text": loc["NOT_FOUND"]}

        user = users[0]
        user_id = user.get('id')
        username = user.get('username', loc["NOT_FOUND"])
        url = user.get('web_url', '')
        bio = user.get('bio', '') or ''

        message_text = f"[{username}]\n"
        entities = []

        if url:
            user_entity = TLRPC.TL_messageEntityTextUrl()
            user_entity.offset = 1
            user_entity.length = len(username.encode('utf_16_le')) // 2
            user_entity.url = url
            entities.append(user_entity)

        if bio:
            message_text += f"> {bio.strip()}\n\n"

        if self.get_setting("gitlab_show_groups", True) and user_id:
            groups_resp = requests.get(f"{GITLAB_API_URL}/users/{user_id}/groups", headers=headers, timeout=10, params={"per_page": 5})
            if groups_resp.status_code == 200 and groups_resp.json():
                message_text += "Groups:\n"
                for group in groups_resp.json():
                    group_name = group.get('name', '')
                    if group_name:
                        group_url = group.get('web_url', f"https://gitlab.com/{group.get('full_path')}")
                        group_start = len(message_text)
                        message_text += f"[{group_name}]\n"
                        group_entity = TLRPC.TL_messageEntityTextUrl()
                        group_entity.offset = len(message_text[:group_start + 1].encode('utf_16_le')) // 2
                        group_entity.length = len(group_name.encode('utf_16_le')) // 2
                        group_entity.url = group_url
                        entities.append(group_entity)
                message_text += "\n"

        message_text += "Projects:\n"
        sort_order = self.get_setting("gitlab_project_sort", 0)
        project_params = {
            "user_id": user_id,
            "per_page": 5,
            "order_by": "updated_at" if sort_order == 0 else "star_count",
            "sort": "desc"
        }
        projects_resp = requests.get(f"{GITLAB_API_URL}/users/{user_id}/projects", headers=headers, timeout=10, params=project_params)

        project_list = []
        if projects_resp.status_code == 200 and projects_resp.json():
            project_list = projects_resp.json()

        if project_list:
            for project in project_list:
                project_name = project.get('name', '')
                project_url = project.get('web_url', '')
                stars = project.get('star_count', 0)
                updated = self._format_date(project.get('last_activity_at', ''))
                project_start = len(message_text)
                message_text += f"[{project_name}]\n[{stars}★] [{updated}]\n"
                project_entity = TLRPC.TL_messageEntityTextUrl()
                project_entity.offset = len(message_text[:project_start + 1].encode('utf_16_le')) // 2
                project_entity.length = len(project_name.encode('utf_16_le')) // 2
                project_entity.url = project_url
                entities.append(project_entity)
        else:
            message_text += f"{loc['NOT_FOUND']}\n"

        return {"text": message_text.strip(), "entities": entities}

    def _gitlab_search_group(self, query, headers, loc):
        params = {"search": query, "per_page": 1}
        r = requests.get(f"{GITLAB_API_URL}/groups", params=params, headers=headers, timeout=10)
        r.raise_for_status()
        groups = r.json()

        if not groups:
            return {"text": loc["NOT_FOUND"]}

        group = groups[0]
        group_id = group.get('id')
        name = group.get('name', loc["NOT_FOUND"])
        url = group.get('web_url', '')
        desc = group.get('description', '') or ''

        message_text = f"[{name}]\n"
        entities = []

        if url:
            group_entity = TLRPC.TL_messageEntityTextUrl()
            group_entity.offset = 1
            group_entity.length = len(name.encode('utf_16_le')) // 2
            group_entity.url = url
            entities.append(group_entity)

        if desc:
            message_text += f"> {desc.strip()}\n\n"

        projects_count = group.get('projects_count', None)
        if projects_count is not None:
            message_text += f"Public projects: {projects_count}\n"

        sort_order = self.get_setting("gitlab_project_sort", 0)
        project_params = {
            "per_page": 5,
            "order_by": "updated_at" if sort_order == 0 else "star_count",
            "sort": "desc"
        }
        projects_resp = requests.get(f"{GITLAB_API_URL}/groups/{group_id}/projects", headers=headers, timeout=10, params=project_params)

        project_list = []
        if projects_resp.status_code == 200 and projects_resp.json():
            project_list = projects_resp.json()

        if project_list:
            message_text += "\nTop projects:\n"
            for project in project_list:
                project_name = project.get('name', '')
                project_url = project.get('web_url', '')
                stars = project.get('star_count', 0)
                updated = self._format_date(project.get('last_activity_at', ''))
                project_start = len(message_text)
                message_text += f"[{project_name}]\n[{stars}★] [{updated}]\n"
                project_entity = TLRPC.TL_messageEntityTextUrl()
                project_entity.offset = len(message_text[:project_start + 1].encode('utf_16_le')) // 2
                project_entity.length = len(project_name.encode('utf_16_le')) // 2
                project_entity.url = project_url
                entities.append(project_entity)
        else:
            message_text += f"{loc['NOT_FOUND']}\n"

        return {"text": message_text.strip(), "entities": entities}