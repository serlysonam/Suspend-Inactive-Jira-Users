# Suspend-Inactive-Jira-Users
Python script to Suspend Jira Users who are inactive past 30 days.
_______________________________________________________________________

import requests
from requests.auth import HTTPBasicAuth
from datetime import datetime, timedelta

# Jira credentials
JIRA_DOMAIN = "serlysonam.atlassian.net"
EMAIL = "serly.sonam@gmail.com "
API_TOKEN = "Atlassian API Token"
ORG_ID = "Atlassian org id"
AUTH_TOKEN = "Auth Token"
# API URL to get all users
user_url   = f"https://{JIRA_DOMAIN}/rest/api/3/users/search?maxResults=1000"
suspend_url = f"https://api.atlassian.com/admin/v1/orgs/{ORG_ID}/directory/users/{{account_id}}/suspend-access"
headers = {"Accept": "application/json"}

# Load exempt users from file
def load_exempt_users(file_path="exempt_users.txt"):
    try:
        with open(file_path, "r") as f:
            return {line.strip() for line in f if line.strip()}
    except FileNotFoundError:
        print("Exempt users file not found. Proceeding without exemptions.")
        return set()

EXEMPT_USERS = load_exempt_users()

def get_atlassian_users():
    response = requests.get(user_url, auth=HTTPBasicAuth(EMAIL, API_TOKEN), headers=headers)
    if response.status_code == 200:
        users = response.json()
        return [(user["accountId"], user["displayName"]) for user in users if user.get("accountType") == "atlassian"]
    else:
        print(f"Error fetching users: {response.status_code}, {response.text}")
        return []

def get_last_active_date(account_id,display_name):
    if display_name in EXEMPT_USERS:
        print(f"Skipping {display_name} (exempt from suspension)")
        return
    url = f"https://api.atlassian.com/admin/v1/orgs/{ORG_ID}/directory/users/{account_id}/last-active-dates"
    headers = {
        "Authorization": f"Bearer {AUTH_TOKEN}",
        "Accept": "application/json"
    }
    response = requests.get(url, headers=headers)
    if response.status_code == 200:
        last_active_data = response.json()
#        last_active = last_active_data["last_active"]
        product_access = last_active_data.get("data", {}).get("product_access", [])
        if product_access:
            last_active = product_access[0].get("last_active", "No data available")
            if last_active != "No data available":
                last_active_date = datetime.strptime(last_active, "%Y-%m-%d")
                if last_active_date < datetime.utcnow() - timedelta(days=30):
                    suspend_user(account_id, display_name)
        else:
            last_active = "No data available"
        print(f"Last active date for {display_name}: {last_active}")
#        print(f"Last active date for {display_name}: {last_active_data}")
    else:
        print(f"Failed to get last active date for {account_id}: {response.status_code}, {response.text}")
def suspend_user(account_id, display_name):
    url = suspend_url.format(account_id=account_id)
    headers = {
        "Authorization": f"Bearer {AUTH_TOKEN}",
        "Accept": "application/json"
    }
    response = requests.post(url, headers=headers)
    if response.status_code == 200:
        print(f"Successfully suspended {display_name}")
    else:
        print(f"Failed to suspend {display_name}: {response.status_code}, {response.text}")
if __name__ == "__main__":
    user_ids = get_atlassian_users()
    for account_id,display_name in user_ids:
        get_last_active_date(account_id, display_name)



