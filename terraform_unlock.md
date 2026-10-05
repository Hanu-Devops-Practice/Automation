# Terraform Workspace Unlock Google Chat Bot

This project is an Azure Functions Python application that receives a Google Chat message, verifies the sender, and calls a second Azure Function to unlock a Terraform workspace.

## 1. Architecture

```text
Google Chat
    |
    | POST /api/google_chat_bot
    v
Azure Function: google_chat_bot
    |
    | validates sender and command
    | GET UNLOCK_FUNCTION_URL with organization, workspace, and code
    v
Azure Function: unlock_workspace
    |
    v
Terraform Enterprise workspace
```

The application exposes these HTTP endpoints:

| Endpoint | Method | Purpose |
| --- | --- | --- |
| `/api/health` | `GET` | Confirms that the function app is running |
| `/api/google_chat_bot` | `POST` | Receives Google Chat events and processes unlock commands |

## 2. Prerequisites

Install or obtain the following:

- Python 3.9 or a compatible supported Python version
- Azure CLI
- Azure Functions Core Tools v4
- An Azure subscription and an existing Azure Function App configured for Python
- Access to the deployed `unlock_workspace` function and its function key
- A Google Chat space or app configuration that can send events to an HTTP endpoint

Verify the local tools:

```powershell
python --version
az --version
func --version
```

Sign in to Azure before deployment:

```powershell
az login
az account show
```

## 3. Project files

| File | Purpose |
| --- | --- |
| `function_app.py` | HTTP routes, authorization, command parsing, and unlock-function call |
| `requirements.txt` | Python dependencies |
| `host.json` | Azure Functions host configuration |
| `local.settings.json` | Local environment variables; do not commit secrets |
| `.funcignore` | Files excluded from deployment |

## 4. Configure local settings

`local.settings.json` contains the values used when running locally. Replace the placeholders with real values:

```json
{
  "IsEncrypted": false,
  "Values": {
    "AzureWebJobsStorage": "UseDevelopmentStorage=true",
    "FUNCTIONS_WORKER_RUNTIME": "python",
    "UNLOCK_FUNCTION_URL": "https://<unlock-function-app>.azurewebsites.net/api/unlock_workspace",
    "UNLOCK_FUNCTION_CODE": "<unlock-function-key>",
    "ALLOWED_USERS": "user1@company.com,user2@company.com",
    "ALLOWED_DOMAINS": "company.com",
    "REQUEST_TIMEOUT_SECONDS": "30"
  }
}
```

Configuration meanings:

- `UNLOCK_FUNCTION_URL`: URL of the downstream workspace-unlock function.
- `UNLOCK_FUNCTION_CODE`: function key for the downstream function.
- `ALLOWED_USERS`: optional comma-separated exact email allow-list.
- `ALLOWED_DOMAINS`: comma-separated email domains allowed to use the bot.
- `REQUEST_TIMEOUT_SECONDS`: maximum time to wait for the downstream function.

The function authorizes a user when the email is listed in `ALLOWED_USERS` or its domain is listed in `ALLOWED_DOMAINS`. Keep `ALLOWED_DOMAINS` restricted to the organization domain.

Do not commit real function keys or other secrets. Use Azure Function App Configuration for deployed values.

## 5. Create the virtual environment

From the project directory:

```powershell
py -3 -m venv .venv
Set-ExecutionPolicy -Scope Process -ExecutionPolicy RemoteSigned
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
```

For subsequent sessions, activate the environment with:

```powershell
.\.venv\Scripts\Activate.ps1
```

## 6. Run locally

Start the Functions host:

```powershell
func start
```

The default local base URL is:

```text
http://localhost:7071
```

## 7. Test the health endpoint

In another PowerShell terminal:

```powershell
Invoke-RestMethod `
  -Uri "http://localhost:7071/api/health" `
  -Method Get
```

Expected response:

```json
{
  "status": "ok",
  "service": "google_chat_bot"
}
```

## 8. Test the bot endpoint locally

Use an email that is allowed by `ALLOWED_USERS` or `ALLOWED_DOMAINS`:

```powershell
$body = @{
  user = @{
    email = "user1@company.com"
  }
  message = @{
    text = "unlock TFE-State-Unlock terraform"
  }
} | ConvertTo-Json -Depth 5

Invoke-RestMethod `
  -Uri "http://localhost:7071/api/google_chat_bot" `
  -Method Post `
  -ContentType "application/json" `
  -Body $body
```

The command format is:

```text
unlock <organization> <workspace>
```

For example:

```text
unlock TFE-State-Unlock terraform
```

The bot returns a Google Chat-compatible response. A successful response confirms the organization, workspace, action, and downstream message.

## 9. Deploy to Azure

### 9.1 Identify the target Function App

```powershell
az functionapp list `
  --query "[].{name:name,resourceGroup:resourceGroup,location:location}" `
  --output table
```

Set the deployment values in the current PowerShell session:

```powershell
$resourceGroup = "<resource-group-name>"
$functionApp = "<function-app-name>"
```

### 9.2 Configure Azure App Settings

Use the Azure Portal under **Function App > Settings > Environment variables**, or run:

```powershell
az functionapp config appsettings set `
  --resource-group $resourceGroup `
  --name $functionApp `
  --settings `
    FUNCTIONS_WORKER_RUNTIME=python `
    UNLOCK_FUNCTION_URL="https://<unlock-function-app>.azurewebsites.net/api/unlock_workspace" `
    UNLOCK_FUNCTION_CODE="<unlock-function-key>" `
    ALLOWED_USERS="user1@company.com" `
    ALLOWED_DOMAINS="company.com" `
    REQUEST_TIMEOUT_SECONDS=30
```

Do not put the real downstream function key in source code or documentation.

### 9.3 Deploy the project

From the project directory, with the virtual environment active:

```powershell
func azure functionapp publish $functionApp --python
```

Wait for the publish command to complete successfully. The deployment URL is:

```text
https://<function-app-name>.azurewebsites.net
```

## 10. Verify the deployed function

### 10.1 Health check

```powershell
$baseUrl = "https://<function-app-name>.azurewebsites.net"

Invoke-RestMethod `
  -Uri "$baseUrl/api/health" `
  -Method Get
```

Expected response:

```json
{
  "status": "ok",
  "service": "google_chat_bot"
}
```

### 10.2 End-to-end test

```powershell
$body = @{
  user = @{
    email = "user1@company.com"
  }
  message = @{
    text = "unlock TFE-State-Unlock terraform"
  }
} | ConvertTo-Json -Depth 5

Invoke-RestMethod `
  -Uri "$baseUrl/api/google_chat_bot" `
  -Method Post `
  -ContentType "application/json" `
  -Body $body
```

This test validates the deployed bot route, authorization, command parsing, downstream function call, and response formatting.

## 11. Configure Google Chat

In the Google Chat app or space configuration, set the HTTP endpoint to:

```text
https://<function-app-name>.azurewebsites.net/api/google_chat_bot
```

Configure the app to send message events to this URL. Then send a message in the configured Google Chat space:

```text
unlock TFE-State-Unlock terraform
```

The Google Chat event parser accepts the Google Chat event structure and also accepts the simpler payload used by the PowerShell test above.

## 12. Monitor and troubleshoot

### View live logs

```powershell
func azure functionapp logstream $functionApp --resource-group $resourceGroup
```

You can also use **Azure Portal > Function App > Functions > `google_chat_bot` > Monitor** or **Application Insights > Logs**.

### Common responses

| Symptom | Likely cause | Check |
| --- | --- | --- |
| `404 Not Found` | Incorrect URL or route | Use `/api/health` and `/api/google_chat_bot` |
| `500 Bot configuration is incomplete` | Missing downstream URL or key | Check Azure App Settings |
| `500 Bot authorization is not configured` | Missing allowed domain | Set `ALLOWED_DOMAINS` |
| User is unauthorized | Email/domain is not allow-listed | Check `ALLOWED_USERS` and `ALLOWED_DOMAINS` |
| Invalid Google Chat request | Request body is not valid JSON or has an unexpected shape | Review the logged event payload |
| Unlock request timed out | Downstream function or Terraform Enterprise is slow/unavailable | Check downstream logs and timeout setting |
| Unlock blocked | Terraform run is active, queued, waiting for apply, or otherwise protected | Review the returned reason and TFE run status |
| Changes are not visible after publish | Old deployment or stale settings | Confirm publish output and restart the Function App |

Restart after changing settings if needed:

```powershell
az functionapp restart `
  --resource-group $resourceGroup `
  --name $functionApp
```

## 13. Security and operational checklist

- Keep `UNLOCK_FUNCTION_CODE` secret and rotate it when required.
- Store secrets in Azure App Settings or a managed secret store, not in Git.
- Restrict `ALLOWED_DOMAINS` to the intended organization domain.
- Use `ALLOWED_USERS` when access must be limited to specific people.
- Review logs because request payloads and response details are logged for troubleshooting.
- Avoid logging function keys or other credentials.
- Confirm that the downstream unlock function also validates its inputs and protects destructive operations.
- Test with a non-production Terraform workspace before using a production workspace.

## 14. End-to-end completion checklist

- [ ] Python environment created and dependencies installed.
- [ ] Local settings configured with a valid downstream function URL and key.
- [ ] `func start` runs without import or configuration errors.
- [ ] `/api/health` returns `status: ok` locally.
- [ ] Authorized local unlock test returns a response.
- [ ] Azure App Settings configured without exposing secrets.
- [ ] Function published successfully.
- [ ] Deployed `/api/health` returns `status: ok`.
- [ ] Deployed bot endpoint passes the PowerShell test.
- [ ] Google Chat endpoint configured.
- [ ] A real Google Chat command produces the expected response.
- [ ] Azure logs and monitoring are available to the support team.
