# App Registration Credential Monitoring

## 1. Purpose

This solution discovers all Microsoft Entra ID app registrations in a tenant and reports their client secrets and certificates. It provides:

- Application name
- Client ID
- Created date
- Credential type
- Credential name
- Credential ID
- Expiry date
- Days remaining
- Owner names

The solution can be scheduled daily in Azure Automation. Notifications can be sent at 30, 15, and 7 days before expiry.

The script never reads, exports, or emails secret values. It reads credential metadata only.

## 2. Architecture

```text
Azure Automation Account
  System-assigned managed identity
          |
          | Microsoft Graph application permissions
          v
Microsoft Graph REST API
  Applications, credentials, and owners
          |
          v
PowerShell report and expiry evaluation
          |
          +--> CSV report
          |
          +--> Logic App Gmail notification
```

Cloud Shell is useful for setup and testing. It should not be the daily scheduler because Cloud Shell is an interactive environment.

## 3. Prerequisites

- Azure subscription
- Microsoft Entra tenant
- Permission to create an Automation Account
- Permission to assign Azure RBAC roles
- An administrator who can grant Microsoft Graph application permissions
- An email mailbox or Logic App for notifications

No Azure resource provider is required for Microsoft Graph. Azure resource providers are only needed when creating the Azure resources themselves.

## 4. Create the Automation Account

In the Azure portal:

1. Open **Automation Accounts**.
2. Select **Create**.
3. Select the subscription, resource group, region, and name.
4. Create the account.
5. Open **Identity**.
6. Enable **System assigned** identity.
7. Select **Save**.
8. Copy the **Object (principal) ID**.

Example value:

```text
171feb91-b19d-4c8a-b5c4-99e9025cadf6
```

The object ID is the service principal object ID of the Automation managed identity. Use this ID as `principalId` when assigning Microsoft Graph app roles.

## 5. Microsoft Graph permissions

The Automation managed identity requires these Microsoft Graph **application permissions**:

| Permission | Fixed app role ID | Purpose |
|---|---|---|
| Application.Read.All | 9a5d68dd-52b0-4cc2-bd40-abcf44ac3a30 | Read app registrations, client secrets, certificates, and application owner relationships |
| Directory.Read.All | 7ab1d382-f21e-4acd-a863-ba3e13f7da61 | Resolve owner object IDs to user, group, or service principal names |

These app role IDs are fixed for Microsoft Graph. The Microsoft Graph service principal object ID (`resourceId`) is tenant-specific and must be queried in each tenant.

Both are application permissions and require tenant admin consent. The managed identity does not use delegated permissions and does not require interactive sign-in.

### Optional email permission

If the runbook sends email directly through Microsoft Graph, add:

| Permission | Purpose |
|---|---|
| Mail.Send | Send notification email as an approved mailbox |

The recommended design is to use a Logic App for email. In that design, the Automation identity does not need `Mail.Send`.

## 6. Assign Graph permissions from Cloud Shell

Run Cloud Shell in PowerShell mode. Replace the managed identity object ID with the value from the Automation Account.

```powershell
$managedIdentityObjectId = "171feb91-b19d-4c8a-b5c4-99e9025cadf6"

$graphId = az ad sp show `
    --id 00000003-0000-0000-c000-000000000000 `
    --query id `
    --output tsv
```

The well-known ID `00000003-0000-0000-c000-000000000000` identifies Microsoft Graph. `$graphId` is the Microsoft Graph service principal object ID in the current tenant.

### Assign Application.Read.All

```powershell
$body = @{
    principalId = $managedIdentityObjectId
    resourceId  = $graphId
    appRoleId   = "9a5d68dd-52b0-4cc2-bd40-abcf44ac3a30"
} | ConvertTo-Json -Compress

az rest `
    --method POST `
    --url "https://graph.microsoft.com/v1.0/servicePrincipals/$managedIdentityObjectId/appRoleAssignments" `
    --body $body `
    --headers "Content-Type=application/json"
```

### Assign Directory.Read.All

```powershell
$body = @{
    principalId = $managedIdentityObjectId
    resourceId  = $graphId
    appRoleId   = "7ab1d382-f21e-4acd-a863-ba3e13f7da61"
} | ConvertTo-Json -Compress

az rest `
    --method POST `
    --url "https://graph.microsoft.com/v1.0/servicePrincipals/$managedIdentityObjectId/appRoleAssignments" `
    --body $body `
    --headers "Content-Type=application/json"
```

If the response says:

```text
Permission being assigned already exists on the object
```

that is not a failure. It means the permission is already assigned.

## 7. Verify Graph permissions

The following command resolves the app role IDs to permission names and avoids the nested PowerShell `$_` variable issue:

```powershell
$assignments = az rest `
    --method GET `
    --url "https://graph.microsoft.com/v1.0/servicePrincipals/$managedIdentityObjectId/appRoleAssignments" |
    ConvertFrom-Json |
    Select-Object -ExpandProperty value

$appRoles = az rest `
    --method GET `
    --url "https://graph.microsoft.com/v1.0/servicePrincipals/$graphId/appRoles" |
    ConvertFrom-Json |
    Select-Object -ExpandProperty value

foreach ($assignment in $assignments) {
    $role = $appRoles | Where-Object { $_.id -eq $assignment.appRoleId }

    [PSCustomObject]@{
        Permission = $role.value
        AppRoleId   = $assignment.appRoleId
        Resource    = $assignment.resourceDisplayName
    }
}
```

Expected permissions:

```text
Application.Read.All
Directory.Read.All
```

A Cloud Shell `az rest` call uses the signed-in Cloud Shell user. The runbook uses the Automation managed identity. Always verify the managed identity object ID is correct.

## 8. Azure RBAC permissions

Azure RBAC is separate from Microsoft Graph permissions.

### Minimum RBAC for the runbook

For the current report-only script, no Azure resource RBAC role is required unless the runbook reads or writes another Azure resource.

The managed identity only needs to obtain its own token through:

```powershell
Connect-AzAccount -Identity
Get-AzAccessToken -ResourceUrl "https://graph.microsoft.com"
```

### If using Azure Storage for notification history

Assign this role on the storage account, table, or resource group:

```text
Storage Table Data Contributor
```

This allows the runbook to read and write notification-history records. Do not assign `Owner` or `Contributor` when this data-plane role is sufficient.

### If using Key Vault

If any configuration or secret is stored in Key Vault, assign:

```text
Key Vault Secrets User
```

Scope it to the specific vault. Do not place client secrets in the runbook source.

## 9. Import the required Automation module

In the Automation Account:

1. Open **Modules**.
2. Confirm `Az.Accounts` is available.
3. If it is missing, select **Browse gallery**.
4. Import `Az.Accounts`.
5. Wait until the module status is **Available**.

The script uses the Azure Automation managed identity and Microsoft Graph REST API. It does not require `Microsoft.Graph.Authentication` or `Microsoft.Graph.Applications`.

## 10. Create the runbook

In the Automation Account:

1. Open **Runbooks**.
2. Select **Create a runbook**.
3. Use:

```text
Name: Monitor-AppRegistrationCredentials
Runbook type: PowerShell
Runtime: PowerShell 7.2
```

4. Create the runbook.
5. Select **Edit**.
6. Paste the contents of `App_registrations.ps1`.
7. Select **Save**.

The default output path is temporary storage:

```text
$env:TEMP\AppRegistrations_Inventory.csv
```

For a persistent report, pass a path backed by Azure Storage or send the results to a Logic App. A temporary Automation path should not be treated as permanent storage.

## 11. Test the runbook

1. Select **Test pane**.
2. Select **Start**.
3. Wait for the job to complete.
4. Check the output and warnings.

Expected output includes:

```text
Script version: 2.0-owner-fix
Applications found: <number>
Report rows exported: <number>
Report path: <path>
```

Expected owner output is a name or UPN, for example:

```text
Hanumanth N
```

The script may show an owner ID only when the directory object cannot be read. This normally means `Directory.Read.All` is missing, assigned to a different identity, or permission propagation has not completed.

## 12. Save, publish, and run

Azure Automation has draft and published versions.

- **Test pane** runs the draft version.
- **Start** runs the published version.
- A schedule runs the published version.

Use this order after every change:

1. Edit the runbook.
2. Select **Save**.
3. Test the draft.
4. Select **Publish**.
5. Start a new job.

Wait several minutes after a new Graph app-role assignment before testing with a new job. Existing jobs use an old token and do not gain newly assigned permissions.

## 13. Create a daily schedule

1. Open the published runbook.
2. Select **Schedules**.
3. Select **Add a schedule**.
4. Select **Create a new schedule**.
5. Configure:

```text
Name: Daily-App-Credential-Check
Frequency: Recurring
Repeat every: 1 day
Time zone: UTC
```

6. Link the schedule to the runbook.
7. Save.

A daily schedule is sufficient because expiry thresholds are measured in days. Use UTC consistently for predictable results.

## 14. Expiry notification rules

For each secret and certificate, calculate:

```text
Days remaining = Expiry date - current UTC date
```

Recommended notification thresholds:

```text
30 days remaining: warning
15 days remaining: warning
7 days remaining: urgent warning
Less than 0 days: expired notification
```

Use a notification-history store to prevent duplicate daily emails. A record should use a key such as:

```text
Application object ID + credential ID + threshold
```

Example keys:

```text
application-id|credential-id|30
application-id|credential-id|15
application-id|credential-id|7
application-id|credential-id|expired
```

## 15. Email notification design

Recommended architecture:

```text
Automation Runbook -> Logic App HTTP trigger -> Gmail connector -> Email
```

The runbook sends a JSON payload containing:

```text
Application name
Client ID
Credential type
Credential name
Credential ID
Expiry date
Days remaining
Owners
```

The Logic App sends the email to the application owners and a support mailbox.

Example subjects:

```text
APP CREDENTIAL WARNING: expires in 30 days
APP CREDENTIAL WARNING: expires in 15 days
URGENT: APP CREDENTIAL expires in 7 days
EXPIRED: APP CREDENTIAL requires rotation
```

### Logic App permissions

The Logic App needs an authenticated Gmail OAuth connection to the sending mailbox. If the Logic App uses a managed identity to access Azure Storage, assign only the required storage data-plane role.

The Automation identity does not need `Mail.Send` when the Logic App sends the email.

### Enable the webhook in the runbook

The runbook supports the optional `NotificationWebhookUri` parameter. In Azure Automation, create an encrypted variable named:

```text
AppCredentialNotificationWebhookUri
```

Set its value to the Logic App HTTP trigger URL. The runbook reads this encrypted variable through `Get-AutomationVariable` and posts notifications only when `Days remaining` is exactly `30`, `15`, or `7`.

For a manual test, the parameter can also be supplied directly:

```powershell
./App_registrations.ps1 -NotificationWebhookUri "https://example.invalid/logic-app-trigger"
```

Do not put the real webhook URL in source control or ordinary runbook output.

The current runbook sends one webhook request for each matching credential. It does not yet persist notification history, so the Logic App must implement duplicate protection before the daily schedule is enabled. Use `NotificationKey` as the duplicate key. Recommended format:

```text
Client ID | Credential ID | Days remaining
```

The webhook payload contains:

```text
ApplicationName
ClientId
CredentialType
CredentialName
CredentialId
ExpiryDate
DaysRemaining
Owners
NotificationKey
RecommendedAction
```

The Logic App should split `Owners`, resolve each owner to an approved email address, and send one message per owner. Do not send email when the owner value is an owner ID or an owner lookup failure.

### Workload identity federation recommendation

The notification includes a recommendation to assess **workload identity federation**. Federation can remove the need for long-lived client secrets or certificates when the workload supports an OIDC-based identity provider, such as GitHub Actions, Azure DevOps, or another supported CI/CD platform.

Owners should confirm:

1. The workload supports OIDC tokens.
2. The issuer, subject, and audience can be restricted to the intended workload.
3. The federated credential can be scoped to the correct application and deployment context.
4. The application code or pipeline can use federated authentication.
5. The required permissions are least-privilege.

Federation is not universal. If the workload cannot support it, the owner must rotate the secret or certificate before expiry. Do not delete an existing credential until the replacement authentication method has been tested successfully.

### Direct Microsoft Graph email option

If the runbook sends email directly through Graph instead of using a Logic App:

1. Assign Microsoft Graph `Mail.Send` application permission to the Automation identity.
2. Grant tenant admin consent.
3. Restrict mailbox usage according to tenant policy.
4. Send only from an approved mailbox.

This option has a larger security impact than using a Logic App.

## 16. Troubleshooting

### Owner ID instead of owner name

Check:

1. The managed identity has both `Application.Read.All` and `Directory.Read.All`.
2. The permission was assigned to the Automation Account identity, not another app.
3. The runbook was saved and published.
4. A new job was started after permission propagation.
5. The owner account still exists in the tenant.

Test the owner with the same tenant, but remember that Cloud Shell tests use the signed-in user token, not the Automation identity token.

### `Authorization_RequestDenied`

The managed identity is missing the required Graph application permission, admin consent, or the token was issued before the permission was assigned.

### `Permission being assigned already exists on the object`

The permission is already assigned. Verify the existing app-role assignments instead of posting it again.

### `Connect-MgGraph` or `RefreshCacheAsync` errors

The production script intentionally uses:

```text
Az.Accounts + Microsoft Graph REST API
```

It does not use the Microsoft Graph PowerShell SDK. This avoids Graph SDK assembly-version conflicts.

### Old output such as `Total credentials found`

That output belongs to an older script version. Confirm the output contains:

```text
Script version: 2.0-owner-fix
```

Save and publish the current code before starting a scheduled or published job.

### No notification is sent

The current script sends notifications when the calculated value is exactly `30`, `15`, or `7`, and also when it is negative (expired):

```powershell
Days remaining -in @(7, 15, 30) -or Days remaining -lt 0
```

For example, a credential at 29 days does not notify, one at 15 days does, and an expired credential also notifies. Expired credentials use the stable `expired` notification key. Do not enable the daily schedule until duplicate-history tracking is configured, otherwise an expired credential can generate an email every day.

## 17. Security practices

- Use managed identity instead of client secrets.
- Grant only the required Graph permissions.
- Do not retrieve or log secret values.
- Do not place credentials in the runbook.
- Use HTTPS Graph endpoints only.
- Restrict Storage and Key Vault RBAC scopes.
- Send notifications only to trusted owner and support addresses.
- Avoid putting access tokens in output or logs.
- Review Automation job output for accidental sensitive data.
- Rotate or remove unused app credentials after owner approval.

## 18. Final checklist

- [ ] Automation Account created.
- [ ] System-assigned managed identity enabled.
- [ ] Managed identity object ID recorded.
- [ ] `Az.Accounts` available in Automation.
- [ ] Graph `Application.Read.All` assigned.
- [ ] Graph `Directory.Read.All` assigned.
- [ ] Admin consent completed.
- [ ] Runbook code saved.
- [ ] Test pane succeeds.
- [ ] Owner names appear.
- [ ] Runbook published.
- [ ] Daily schedule linked.
- [ ] Notification history configured in the Logic App or another persistent store.
- [ ] Logic App or direct email path tested.
- [ ] 30-day, 15-day, 7-day, and expired notifications tested.
- [ ] CSV/report retention configured.

## 19. Create and configure the Logic App

This section uses a **Consumption Logic App** and the Gmail connector.

### 19.1 Create the Logic App

1. Open the Azure portal.
2. Search for **Logic Apps**.
3. Select **Add**.
4. Select the same subscription and resource group used for the monitoring solution.
5. Choose **Consumption** as the plan type.
6. Provide a Logic App name, for example:

```text
la-app-credential-notifications
```

7. Select the region.
8. Select **Review + create**.
9. Select **Create**.
10. Open the Logic App and select **Logic app designer**.

### 19.2 Add the HTTP trigger

1. Select **Blank Logic App**.
2. Search for:

```text
When an HTTP request is received
```

3. Add the trigger.
4. Set **Request Body JSON Schema** to:

```json
{
    "type": "object",
    "properties": {
        "ApplicationName": { "type": "string" },
        "ClientId": { "type": "string" },
        "CredentialType": { "type": "string" },
        "CredentialName": { "type": "string" },
        "CredentialId": { "type": "string" },
        "ExpiryDate": { "type": "string" },
        "DaysRemaining": { "type": "integer" },
        "Owners": {
            "type": "array",
            "items": { "type": "string" }
        },
        "NotificationKey": { "type": "string" }
    },
    "required": [
        "ApplicationName",
        "ClientId",
        "CredentialType",
        "CredentialId",
        "ExpiryDate",
        "DaysRemaining",
        "Owners",
        "NotificationKey"
    ]
}
```

5. Save the Logic App.
6. Copy the generated **HTTP POST URL** from the trigger.

Treat this URL like a secret. Anyone who has it can submit a notification to the workflow.

### 19.3 Store the webhook URL in Automation

In the Automation Account:

1. Open **Variables**.
2. Select **Add a variable**.
3. Use:

```text
Name: AppCredentialNotificationWebhookUri
Type: String
Value: <Logic App HTTP POST URL>
Encrypted: Yes
```

4. Save the variable.

The runbook reads this variable automatically. Do not paste the URL into the runbook source or write it to job output.

### 19.4 Add duplicate protection before production

The runbook sends a `NotificationKey` based on client ID, credential ID, and threshold. Expired credentials use `expired` as the threshold. For production, add duplicate protection before sending email.

The current PowerShell runbook does not implement this storage check itself. Configure the Logic App to perform it before Gmail sends the message.

Recommended approach:

1. Create a Storage Account.
2. Create a table named:

```text
CredentialNotificationHistory
```

3. Add a **Get entity** or equivalent lookup before sending email.
4. Use the notification key as the lookup value.
5. If the key already exists, terminate the workflow with status **Succeeded** and do not send email.
6. If the key does not exist, send the email and insert the key into the table.

Use these values for the table entity:

```text
PartitionKey: AppCredentialExpiry
RowKey: <NotificationKey>
ApplicationName: <ApplicationName>
ClientId: <ClientId>
CredentialId: <CredentialId>
DaysRemaining: <DaysRemaining>
SentOnUtc: <utcNow()>
```

The identity used by the workflow requires the minimum storage data permission needed to read and write this table. If the Logic App uses its managed identity, assign `Storage Table Data Contributor` to that identity at the storage-account scope. Do not enable the production schedule until this duplicate check is configured, or repeated runs can send duplicate messages at a matching threshold.

### 19.5 Configure Gmail authentication

Use the Gmail connector instead of the Office 365 Outlook connector when mail must be sent through Gmail.

1. In the Logic App designer, add a new action.
2. Search for **Gmail**.
3. Select **Send email**.
4. Select **Sign in**.
5. Sign in with the approved Gmail or Google Workspace mailbox.
6. Approve the requested OAuth permissions.
7. Confirm that the connector connection is shown as connected.

The Gmail connector uses OAuth. Do not place a Gmail password, app password, or OAuth token in the runbook. The Logic App stores the connector connection separately.

For production, use a dedicated Google Workspace mailbox, for example:

```text
app-credential-notifications@contoso.com
```

Confirm that the Google Workspace administrator allows the Azure Logic Apps Gmail connector and that the mailbox is permitted to send external mail. Consumer Gmail and Google Workspace policies may restrict connector access.

### 19.6 Add the owner loop

1. Add **For each** after the HTTP trigger.
2. Select the dynamic property:

```text
Owners
```

3. Inside the loop, add the Gmail action:

```text
Gmail - Send email
```

4. Configure the **To** field with the current owner value.
5. Configure the **Subject** field:

```text
APP CREDENTIAL WARNING: @{triggerBody()?['ApplicationName']} expires in @{triggerBody()?['DaysRemaining']} days
```

6. Configure the email body with:

```text
Hello,

The following app registration credential requires attention.

Application: @{triggerBody()?['ApplicationName']}
Client ID: @{triggerBody()?['ClientId']}
Credential type: @{triggerBody()?['CredentialType']}
Credential name: @{triggerBody()?['CredentialName']}
Credential ID: @{triggerBody()?['CredentialId']}
Expiry date: @{triggerBody()?['ExpiryDate']}
Days remaining: @{triggerBody()?['DaysRemaining']}

Please rotate the credential before it expires and update the application owner records after rotation.
```

The Gmail connector must be authorized with the approved Google Workspace or Gmail service mailbox. Do not use a personal mailbox for production notifications.

### 19.7 Handle owner values that are not email addresses

The runbook returns user UPNs when available. A UPN can normally be used as the Gmail recipient, but a group display name or service-principal display name is not necessarily an email address.

Add a support mailbox fallback:

```text
app-support@contoso.com
```

For non-routable owner values, send the notification to the support mailbox and include the owner value in the message. Do not attempt to send mail to values beginning with:

```text
Owner ID:
Service Principal:
```

### 19.8 Return a response to the runbook

Add a **Response** action at the end of the Logic App:

```json
{
    "status": "accepted"
}
```

Set the status code to:

```text
202
```

This gives the runbook a clear successful webhook response.

### 19.9 Save and obtain the final URL

1. Select **Save**.
2. Open the HTTP trigger.
3. Copy the current **HTTP POST URL**.
4. Update the encrypted Automation variable if the URL changed.
5. Save and publish the runbook.

### 19.10 Test the Logic App directly

Use a test payload from Cloud Shell or another authorized test client. Do not place the real URL in source control or shared chat.

```powershell
$payload = @{
        ApplicationName = "Test application"
        ClientId = "00000000-0000-0000-0000-000000000000"
        CredentialType = "Client Secret"
        CredentialName = "Test credential"
        CredentialId = "11111111-1111-1111-1111-111111111111"
        ExpiryDate = "2026-10-30"
        DaysRemaining = 30
        Owners = @("owner@contoso.com")
        NotificationKey = "test|credential|30"
} | ConvertTo-Json

Invoke-RestMethod `
        -Uri $logicAppWebhookUrl `
        -Method Post `
        -ContentType "application/json" `
        -Body $payload
```

Confirm that the email arrives and that the Logic App run history shows a successful run.

### 19.11 Test the complete solution

1. Create or identify a test credential that expires in 30 days.
2. Run the Automation runbook in the Test pane.
3. Confirm `Notifications submitted: 1`.
4. Confirm the Logic App run history shows a successful trigger.
5. Confirm the owner receives the message.
6. Run the workflow a second time.
7. Confirm duplicate protection prevents a second message.
8. Repeat with 15-day and 7-day test data.
9. Publish the runbook.
10. Enable the daily schedule.
