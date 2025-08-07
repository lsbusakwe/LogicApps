# Scan Machine

**Author**: Lungelo Shaun Busakwe

For any technical questions, please contact [lungelo.busakwe@conosco.com](mailto:lungelo.busakwe@conosco.com).

Deploy this Logic App to Azure:

[![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Flsbusakwe%2FLogicApps%2Fmaster%2FScanMachine%2Fazuredeploy.json)
[![Deploy to Azure Gov](https://aka.ms/deploytoazuregovbutton)](https://portal.azure.us/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Flsbusakwe%2FLogicApps%2Fmaster%2FScanMachine%2Fazuredeploy.json)

## Overview
This Logic App automates initiating an antivirus scan on a machine in Microsoft Defender for Endpoint as part of a security incident response. It is triggered by an HTTP POST request containing `machineid`, `machineName`, and `incident_id`, retrieves a client secret from Azure Key Vault, authenticates with the Defender for Endpoint API, runs a quick scan on the specified machine, and includes placeholder actions for notifications.

## Minimum Requirements
- **Azure Subscription**: With permissions to create Logic Apps, Key Vaults, and manage Azure AD app registrations.
- **Microsoft 365 E5 or Microsoft Defender for Endpoint License**: Required for machine scanning via Defender for Endpoint API.
- **Azure Sentinel Workspace**: Optional, for future integration with Sentinel (Workspace ID is parameterized but unused in the current workflow).
- **Azure Key Vault**: To store the client secret for Defender API authentication.
- **Azure AD Permissions**: Ability to create app registrations and grant admin consent.

## Technical Prerequisites
Before deploying the Logic App, complete the following setup steps in the Azure portal.

### 1. Create an Azure AD App Registration
The Logic App uses an Azure AD application to authenticate with the Microsoft Defender for Endpoint API to run antivirus scans on machines.

1. **Navigate to Azure AD**:
   - In the Azure portal, go to **Azure Active Directory** > **App registrations**.
   - Click **New registration**.
2. **Register the Application**:
   - **Name**: Enter a name (e.g., `MachineScanApp`).
   - **Supported account types**: Select **Accounts in this organizational directory only**.
   - Click **Register**.
3. **Save App ID and Tenant ID**:
   - Copy the **Application (client) ID**. This is the `clientId` for the Logic App deployment.
   - Copy the **Directory (tenant) ID** from **Azure Active Directory** > **Overview**. This is the `tenantId`.
   - Store these values securely for use during deployment.
4. **Add API Permissions**:
   - In the app registration, go to **API permissions** > **Add a permission**.
   - Select **APIs my organization uses**, search for **WindowsDefenderATP** (Microsoft Defender for Endpoint), and select it.
   - Choose **Application permissions** and add:
     - `Machine.Scan`
   - Click **Add permissions**.
5. **Grant Admin Consent**:
   - In **API permissions**, click **Grant admin consent for <your-tenant>**.
   - Confirm the consent to allow the app to manage machine scans without user interaction.

### 2. Identify the Azure Sentinel Workspace ID
The Logic App requires a `workspaceId` parameter for potential future integration with Azure Sentinel. If you plan to add Sentinel actions later, obtain the workspace ID.

1. **Navigate to Azure Sentinel**:
   - In the Azure portal, go to **Microsoft Sentinel**.
   - Select your workspace (e.g., `Conosco_Sentinel_RG`).
2. **Copy the Workspace ID**:
   - In the workspace’s **Settings** > **Workspace settings**, find the **Workspace ID**.
   - Save this ID for use during deployment (required, even though unused in the current workflow).

### 3. Create and Configure Azure Key Vault
The Logic App retrieves the client secret for Defender API authentication from a Key Vault secret named `MDR-ScanMachine`.

1. **Create a Key Vault**:
   - In the Azure portal, go to **Key vaults** > **Create**.
   - **Resource group**: Choose or create a resource group (e.g., `Conosco_Sentinel_RG`).
   - **Key vault name**: Enter a unique name (e.g., `ConoscoKeyVault`).
   - **Region**: Select the same region as your Logic App (e.g., `uksouth`).
   - **Pricing tier**: Choose **Standard**.
   - Click **Review + create** and then **Create**.
2. **Switch to Access Policies**:
   - In the Key Vault, go to **Access configuration**.
   - Select **Vault access policy** (not Azure RBAC) to allow fine-grained access control.
   - Click **Save**.
3. **Add a Secret**:
   - In the Key Vault, go to **Objects** > **Secrets** > **Generate/Import**.
   - **Name**: Enter `MDR-ScanMachine`.
   - **Value**: Paste the client secret for the Azure AD app registration:
     - In the app registration, go to **Certificates & secrets** > **Client secrets** > **New client secret**.
     - Set a description (e.g., `ScanMachineSecret`) and expiry (e.g., 1 year).
     - Copy the **Value** (not the Secret ID) immediately and paste it into the Key Vault secret.
   - Click **Create**.
4. **Add Access Policy for the App Registration**:
   - In the Key Vault, go to **Access policies** > **Create**.
   - **Permissions**: Select **Secret permissions** > **Get**.
   - **Select principal**: Search for the app registration by its name or Client ID (e.g., `MachineScanApp`).
   - Click **Add** and then **Save**.

### 4. Deploy the Logic App
1. **Click the Deploy Button**:
   - Use the **Deploy to Azure** button above to start the deployment.
   - You will be redirected to the Azure portal’s custom deployment page.
2. **Enter Parameters**:
   - **logicAppName**: Enter a unique name (default: `MachineScan`).
   - **location**: Select the Azure region (e.g., `uksouth`).
   - **tenantId**: Enter the Tenant ID from the app registration.
   - **clientId**: Enter the Client ID from the app registration.
   - **workspaceId**: Enter the Sentinel Workspace ID.
3. **Deploy**:
   - Select your subscription and resource group.
   - Click **Review + create** and then **Create**.
   - Wait for the deployment to complete (typically a few minutes).

### 5. Post-Deployment Configuration
After deployment, configure the Key Vault connection and grant the Logic App access to the Key Vault.

1. **Configure the Key Vault Connection**:
   - In the Azure portal, navigate to the deployed Logic App (e.g., `MachineScan`).
   - Go to **Logic app designer** under **Development Tools**.
   - The `Get_secret` action will show an error due to the missing `keyvault` connection.
   - Click the `Get_secret` action and select **Azure Key Vault** as the connector.
   - Click **Add new** to create a connection:
     - **Connection Name**: Enter `keyvault`.
     - **Authentication**: Select **Connect with managed identity**.
     - **Key Vault**: Choose the Key Vault created earlier (e.g., `ConoscoKeyVault`).
   - Click **Create** and save the connection.
2. **Add Access Policy for the Logic App**:
   - In the Azure portal, go to the Key Vault (e.g., `ConoscoKeyVault`).
   - Go to **Access policies** > **Create**.
   - **Permissions**: Select **Secret permissions** > **Get**.
   - **Select principal**: Search for the Logic App by its name (e.g., `MachineScan`).
     - The Logic App uses a System Assigned Managed Identity, which appears with the same name as the Logic App.
   - Click **Add** and then **Save**.
3. **Save the Logic App**:
   - In the Logic App designer, click **Save** to apply the connection configuration.

### 6. Test the Logic App
1. **Get the HTTP Trigger URL**:
   - In the Logic App designer, expand the **When_a_HTTP_request_is_received** trigger.
   - Copy the **HTTP POST URL**.
2. **Send a Test Request**:
   - Use a tool like Postman or `curl` to send a `POST` request to the URL with the following JSON body:
     ```json
     {
         "machineid": "machine123",
         "machineName": "TestMachine",
         "incident_id": "12345"
     }
     ```
3. **Verify Execution**:
   - In the Azure portal, go to **Logic App** > **Overview** > **Run history**.
   - Check that the Logic App:
     - Retrieves the `MDR-ScanMachine` secret from Key Vault.
     - Authenticates with the Microsoft Defender for Endpoint API.
     - Initiates an antivirus scan on the specified machine.
     - Executes the placeholder actions (`PlaceholderforSubject`, `PlaceHolderforMessage`).
     - Returns a `200` response with `{ "status": "success" }`.
   - If errors occur, check the **Run history** details for specific issues.

### 7. Optional: Add Notification Actions
The Logic App includes placeholder actions (`PlaceholderforSubject`, `PlaceHolderforMessage`) for notifications. To add Microsoft Teams, Office 365 email, or Azure Sentinel actions:

1. **Open Logic App Designer**:
   - Navigate to the Logic App and open **Logic app designer**.
2. **Add Actions**:
   - After `PlaceHolderforMessage`, add actions like:
     - **Office 365 Outlook** > **Send an email (V2)** to send an email.
     - **Microsoft Teams** > **Post message in a chat or channel** to post to a Teams channel.
     - **Azure Sentinel** > **Add comment to incident (V3)** to update a Sentinel incident.
   - Create new connections for each action:
     - Use names like `office365_1`, `teams`, or `azuresentinel-1` to match the original Logic App if needed.
     - For Office 365 and Teams, sign in with an Azure AD account (OAuth).
     - For Sentinel, use Managed Service Identity and ensure the Logic App’s identity has the `Microsoft Sentinel Contributor` role on the workspace.
3. **Save and Test**:
   - Save the workflow and test with a new HTTP request.

### Troubleshooting
- **Deployment Errors**:
  - Ensure `logicAppName` is unique and `location` is valid (e.g., `uksouth`).
  - Verify that `tenantId`, `clientId`, and `workspaceId` are provided during deployment.
  - Check deployment logs in the Azure portal under **Resource group** > **Deployments**.
- **Key Vault Errors**:
  - Verify the Key Vault exists and contains the `MDR-ScanMachine` secret.
  - Ensure the Logic App’s Managed Identity and app registration have `Get` permissions in the Key Vault’s access policies.
- **Defender API Errors**:
  - Confirm the `clientId` and `tenantId` are correct and the app registration has `Machine.ReadWrite.All` permission with admin consent.
  - Check **Run history** for authentication or API errors.
- **Connection Errors**:
  - If the `keyvault` connection fails in the designer, verify the Key Vault name and Managed Identity permissions.
  - For future Teams or Office 365 actions, ensure OAuth authentication is completed.

### Support
For additional help, contact [lungelo.busakwe@conosco.com](mailto:lungelo.busakwe@conosco.com).
