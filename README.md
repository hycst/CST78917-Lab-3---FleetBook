#### CST8917 Lab 3 — FleetBook

For Lab3, FleetBook is a serverless vehicle-booking workflow built with Azure Service Bus, Azure Logic Apps, Azure Functions, Outlook, and a browser-based HTML client.

#### Architecture

```text
FleetBook Web Client
        |
        v
Service Bus Queue: booking-queue
        |
        v
Logic App: process-booking
        |
        v
Azure Function: check-booking
        |
        +--> Confirmation or rejection email
        |
        v
Service Bus Topic: booking-results
        |
        +--> confirmed-sub
        |
        +--> rejected-sub
        |
        v
FleetBook dashboard polls and displays the result
```

#### Features

- Submits vehicle-booking requests from a single-file HTML client.
- Sends booking requests to Azure Service Bus through its REST API.
- Uses an Azure Logic App to decode and parse queue messages.
- Calls an Azure Function to check vehicle availability and calculate pricing.
- Routes confirmed and rejected bookings through conditional branches.
- Sends confirmation or rejection emails through Outlook.
- Publishes results to filtered Service Bus topic subscriptions.
- Updates the browser dashboard from Pending to Confirmed or Rejected.
- Applies add-on charges and a 10% weekly discount for rentals of seven days or more.

#### Repository Files

```text
.
├── function_app.py
├── requirements.txt
├── test-function.http
├── client.html
├── local.settings.example.json
├── host.json
├── .gitignore
└── README.md
```

Do not commit `local.settings.json`, SAS keys, connection strings, or other secrets.

#### Azure Resources

The solution requires:

- One Azure Service Bus namespace
- Queue: `booking-queue`
- Topic: `booking-results`
- Subscription: `confirmed-sub`
- Subscription: `rejected-sub`
- One Python Azure Function App
- One Consumption Logic App
- One Outlook.com or Microsoft 365 connection

#### Subscription filters

Configure the subscriptions with SQL filters:

```sql
-- confirmed-sub
sys.label = 'confirmed'
```

```sql
-- rejected-sub
sys.label = 'rejected'
```

#### Azure Function

The Function App includes:

- `check-booking`: evaluates availability and calculates the estimated price.
- `health`: verifies that the deployed Function App is available.

Health-check URL:

```text
https://fleetbook-func-hy-eef8eue9ddaae2g4.canadacentral-01.azurewebsites.net/api/health
```

Expected response:

```json
{
  "status": "healthy",
  "service": "FleetBook Function App",
  "fleet_size": 10
}
```

#### Local Setup

#### Prerequisites

- Python 3.12
- Azure Functions Core Tools 4
- Azure CLI
- Visual Studio Code
- Python extension for VS Code
- Azure Functions extension for VS Code
- Azurite extension
- Live Server extension

#### Create and activate a virtual environment

PowerShell:

```powershell
py -3.12 -m venv .venv
Set-ExecutionPolicy -Scope Process -ExecutionPolicy RemoteSigned
.\.venv\Scripts\Activate.ps1
pip install -r requirements.txt
```

#### Local settings

Create `local.settings.json` from the example file:

```json
{
  "IsEncrypted": false,
  "Values": {
    "AzureWebJobsStorage": "UseDevelopmentStorage=true",
    "FUNCTIONS_WORKER_RUNTIME": "python"
  },
  "Host": {
    "CORS": "*"
  }
}
```

Start Azurite and then run:

```powershell
func start
```

#### Deploy the Function App

Sign in:

```powershell
az login
```

Deploy from VS Code using:

```text
Azure extension → Workspace → Deploy to Function App
```

After deployment, verify:

```text
https://<function-app-name>.azurewebsites.net/api/health
Actual URL in this lab: https://fleetbook-func-hy-eef8eue9ddaae2g4.canadacentral-01.azurewebsites.net/api/health
```

#### Logic App Workflow

The `process-booking` Logic App contains:

1. Service Bus queue trigger using auto-complete.
2. Compose action to decode `ContentData`.
3. Parse JSON action for the booking request.
4. Azure Function action calling `check-booking`.
5. Parse JSON action for the Function response.
6. Condition: `status` equals `confirmed`.
7. True branch:
   - Send confirmation email.
   - Publish to `booking-results` with label `confirmed`.
8. False branch:
   - Send rejection email.
   - Publish to `booking-results` with label `rejected`.

#### Run the Web Client

Open `client.html` using Live Server:

```text
http://127.0.0.1:5500/client.html
```

Expand **Service Bus Configuration** and enter:

- Service Bus namespace name
- SAS policy name: `RootManageSharedAccessKey`
- Primary SAS key

The SAS key must be entered only for local testing. Never commit it to GitHub.

#### Test a Confirmed Booking

Use values similar to:

```text
Customer Name: Jane Ottawa
Customer Email: h*****@outlook.com
Vehicle Type: Sedan
Pickup Location: Ottawa
```

Expected result:

- New Logic App run
- True branch executed
- Confirmation email sent
- Result published with label `confirmed`
- Dashboard updated to Confirmed with vehicle and pricing information

#### Test a Rejected Booking

Use values similar to:

```text
Customer Name: John Montreal
Customer Email: h*****@outlook.com
Vehicle Type: Sedan
Pickup Location: Montreal
```

Expected result:

- New Logic App run
- False branch executed
- Rejection email sent
- Result published with label `rejected`
- Dashboard updated to Rejected with the reason

#### Important Testing Note

The web client retrieves topic messages using receive-and-delete mode. After the dashboard consumes a result, the active-message count in the corresponding subscription can return to zero. Peek at subscription messages before the client consumes them when a screenshot of the count is required.

#### Security

- Never commit `local.settings.json`.
- Never commit SAS keys or connection strings.
- Do not expose production Service Bus credentials in browser code.
- The direct browser-to-Service-Bus approach is used only for this lab.
- A production system should use a backend API or managed identity.

#### Explanation of the Topic subscriptions showing filtered message counts

In this lab, there are the two topic subscriptions, confirmed-sub and rejected-sub. The Logic App publishes the booking result to the topic, and the Service Bus filter routes it to the correct subscription. In this lab, the FleetBook client uses Receive-and-Delete mode, so once the dashboard receives the message, the subscription count returns to zero.


#### Demo Video


Demo Video: https://youtu.be/zk9Y5aXq_7E

Demo Video: https://youtu.be/UqbAysO--ZY

