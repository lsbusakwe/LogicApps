# AutoClose Known IPs

**Author**: Lungelo Shaun Busakwe

For any technical questions, please contact [lungelo.busakwe@conosco.com](mailto:lungelo.busakwe@conosco.com).

Deploy this Logic App to Azure:

[![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Flsbusakwe%2FLogicApps%2Fmaster%2FAutoCloseKnownIPs%2Fazuredeploy.json)
[![Deploy to Azure Gov](https://aka.ms/deploytoazuregovbutton)](https://portal.azure.us/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Flsbusakwe%2FLogicApps%2Fmaster%2FAutoCloseKnownIPs%2Fazuredeploy.json)

## Overview
This Logic App automates closing Microsoft Sentinel incidents involving known IP addresses. It is designed to be triggered by a Sentinel incident webhook (configured post-deployment), checks if any IP in the incident’s related entities matches a known IP from a JSON file in Azure Blob Storage (`/knownips/knownips.json`), and includes a placeholder action to indicate a match and intent to close the incident. The current workflow uses a temporary HTTP trigger for testing, which should be replaced with a Sentinel webhook trigger after deployment.

## Minimum Requirements
- **Azure Subscription**: With permissions to create Logic Apps, Blob Storage accounts, and manage Sentinel workspaces.
- **Microsoft Sentinel Workspace**: Required for the incident webhook trigger and future incident updates.
- **Azure Blob Storage**: To store the `knownips.json` file containing known IPs.
- **Azure AD Permissions**: Ability to assign roles to the Logic App’s Managed Identity.

## Technical Prerequisites
Before deploying the Logic App, complete the following setup steps in the Azure portal.

### 1. Configure Azure Blob Storage
The Logic App (once configured) retrieves a JSON file (`/knownips/knownips.json`) from a Blob Storage account.

1. **Create a Storage Account**:
   - In the Azure portal, go to **Storage accounts** > **Create**.
   - **Resource group**: Choose or create a resource group (e.g., `Conosco_Sentinel_RG`).
   - **Storage account name**: Enter a unique name (e.g., `conoscosentinelstorage`).
   - **Region**: Select the same region as your Logic App (e.g., `uksouth`).
   - **Performance**: Choose **Standard**.
   - **Redundancy**: Select **Locally-redundant storage (LRS)** or as needed.
   - Click **Review + create** and then **Create**.
2. **Create a Container**:
   - In the storage account, go to **Containers** > **+ Container**.
   - **Name**: Enter `knownips`.
   - **Public access level**: Select **Private (no anonymous access)**.
   - Click **Create**.
3. **Upload the JSON File**:
   - In the `knownips` container, click **Upload**.
   - Upload a file named `knownips.json` with the following structure:
     ```json
     [
         {
             "Name": "TrustedServer1",
             "IP Address": "192.168.1.100"
         },
         {
             "Name": "TrustedServer2",
             "IP Address": "10.0.0.50"
         }
     ]
     ```
   - Ensure the JSON is valid and contains `Name` and `IP Address` fields for each entry.
   - Click **Upload**.

### 2. Identify the Azure Sentinel Workspace ID
The Logic App requires a `workspaceId` parameter for potential future integration with Azure Sentinel.

1. **Navigate to Azure Sentinel**:
   - In the Azure portal, go to **Microsoft Sentinel**.
   - Select your workspace (e.g., `Conosco_Sentinel_RG`).
2. **Copy the Workspace ID**:
   - In the workspace’s **Settings** > **Workspace settings**, find the **Workspace ID**.
   - Save this ID for use during deployment (required, even though unused in the current workflow).

### 3. Deploy the Logic App
1. **Click the Deploy Button**:
   - Use the **Deploy to Azure** button above to start the deployment.
   - You will be redirected to the Azure portal’s custom deployment page.
2. **Enter Parameters**:
   - **logicAppName**: Enter a unique name (default: `MachineScan`). **Note**: Ensure the name is unique within the resource group to avoid conflicts with other Logic Apps (e.g., "Scan Machine" or "Revoke Entra ID Sign-In Sessions" using the same default name). Consider appending a suffix (e.g., `MachineScan-AutoCloseIPs`).
   - **location**: Select the Azure region (e.g., `uksouth`).
   - **tenantId**: Enter your Azure AD Tenant ID (optional, unused in this workflow).
   - **clientId**: Enter a Client ID for Microsoft Graph API authentication (optional, unused in this workflow).
   - **workspaceId**: Enter the Sentinel Workspace ID.
3. **Deploy**:
   - Select your subscription and resource group.
   - Click **Review + create** and then **Create**.
   - Wait for the deployment to complete (typically a few minutes).

### 4. Post-Deployment Configuration
After deployment, replace the HTTP trigger with a Sentinel webhook trigger, configure the Blob Storage connection, and assign the necessary permissions.

1. **Replace the HTTP Trigger with a Sentinel Webhook**:
   - In the Azure portal, navigate to the deployed Logic App (e.g., `MachineScan`).
   - Go to **Logic app designer** under **Development Tools**.
   - Delete the `When_a_HTTP_request_is_received` trigger.
   - Add a new trigger: **Microsoft Sentinel** > **When an incident is created**.
   - Configure the trigger:
     - **Connection Name**: Enter `azuresentinel`.
     - **Authentication**: Select **Connect with managed identity**.
     - **Workspace**: Choose your Sentinel workspace (e.g., `Conosco_Sentinel_RG`).
   - Click **Create** and save the connection.
   - Copy the **Callback URL** from the trigger (visible in the designer after saving).
2. **Configure the Sentinel Webhook**:
   - In Microsoft Sentinel, go to **Automation** > **Create** > **Playbook with incident trigger**.
   - Instead of creating a new playbook, select **Existing playbook** and choose the deployed Logic App (e.g., `MachineScan`).
   - Paste the **Callback URL** from the Logic App trigger.
   - Save the automation rule to trigger the Logic App on new Sentinel incidents.
3. **Add Blob Storage Connection**:
   - In the Logic App designer, add a new action after the trigger and before `PlaceholderforKnownIPs`.
   - Select **Azure Blob Storage** > **Get blob content (V2)**.
   - Configure the action:
     - **Connection Name**: Enter `azureblob`.
     - **Authentication**: Select **Connect with managed identity** or use an Access Key (copy from the storage account’s **Access keys**).
     - **Storage Account**: Choose the storage account (e.g., `conoscosentinelstorage`).
     - **Blob Path**: Enter `/knownips/knownips.json`.
   - Click **Create** and save the connection.
4. **Add Parse JSON Action**:
   - After the `Get blob content (V2)` action, add a **Parse JSON** action.
   - Configure the action:
     - **Content**: Select `@base64ToString(body('Get_blob_content_(V2)')['$content'])`.
     - **Schema**: Use:
       ```json
       {
           "type": "array",
           "items": {
               "type": "object",
               "properties": {
                   "Name": {
                       "type": "string"
                   },
                   "IP Address": {
                       "type": "string"
                   }
               },
               "required": [
                   "Name",
                   "IP Address"
               ]
           }
       }
       ```
   - Save the action.
5. **Update For_each Action**:
   - In the `For_each` action, update the `Condition` to use the output of `Parse_JSON` instead of `PlaceholderforKnownIPs`:
     - Change `@item()['IP Address']` to reference `@body('Parse_JSON')`.
   - Move the `PlaceholderforMessage` action outside the `For_each` loop if you want it to execute only once per match.
6. **Assign Permissions for Blob Storage**:
   - In the Azure portal, go to the storage account (e.g., `conoscosentinelstorage`).
   - Go to **Access control (IAM)** > **Add role assignment**.
   - **Role**: Select **Storage Blob Data Contributor**.
   - **Assign access to**: Select **Managed identity** > **Logic App**.
   - **Select**: Choose the Logic App (e.g., `MachineScan`).
   - Click **Save**.
7. **Assign Permissions for Sentinel**:
   - In the Azure portal, go to the Sentinel workspace (e.g., `Conosco_Sentinel_RG`).
   - Go to **Access control (IAM)** > **Add role assignment**.
   - **Role**: Select **Microsoft Sentinel Contributor**.
   - **Assign access to**: Select **Managed identity** > **Logic App**.
   - **Select**: Choose the Logic App (e.g., `MachineScan`).
   - Click **Save**.
8. **Save the Logic App**:
   - In the Logic App designer, click **Save** to apply all changes.

### 5. Optional: Add Sentinel Actions
The Logic App includes a `PlaceholderforMessage` action for incident updates. To add Sentinel actions for commenting and closing incidents:

1. **Open Logic App Designer**:
   - Navigate to the Logic App and open **Logic app designer**.
2. **Add Actions**:
   - After `PlaceholderforMessage`, add:
     - **Microsoft Sentinel** > **Add comment to incident (V3)**:
       - **Connection Name**: Use `azuresentinel` (created earlier).
       - **Incident ARM ID**: `@triggerBody()?['object']?['id']`.
       - **Message**: `IP @{item()['IP Address']} is a known IP: @{item()['Name']}`.
     - **Microsoft Sentinel** > **Update incident**:
       - **Connection Name**: Use `azuresentinel`.
       - **Incident ARM ID**: `@triggerBody()?['object']?['id']`.
       - **Status**: Select **Closed**.
   - Ensure the Logic App’s Managed Identity has the `Microsoft Sentinel Contributor` role (assigned above).
3. **Save and Test**:
   - Save the workflow and test with a Sentinel incident.

### 6. Test the Logic App
1. **Test with HTTP Trigger (Temporary)**:
   - In the Logic App designer, expand the `When_a_HTTP_request_is_received` trigger.
   - Copy the **HTTP POST URL**.
   - Send a `POST` request using Postman or `curl` with a sample payload:
     ```json
     {
         "object": {
             "id": "inc-12345",
             "properties": {
                 "relatedEntities": [
                     {
                         "id": "192.168.1.100"
                     },
                     {
                         "id": "10.1.1.1"
                     }
                 ]
             }
         }
     }
     ```
   - Verify in **Run history** that the Logic App:
     - Processes the `PlaceholderforKnownIPs` output.
     - Matches IPs in `relatedEntities` against the placeholder IPs.
     - Outputs the `PlaceholderforMessage` for matched IPs (e.g., `192.168.1.100`).
2. **Test with Sentinel Webhook**:
   - After configuring the Sentinel webhook (step 4.1 and 4.2), create a test incident in Sentinel.
   - Verify in **Run history** that the Logic App:
     - Triggers on the incident.
     - Retrieves and parses `knownips.json` from Blob Storage.
     - Matches IPs and executes the `PlaceholderforMessage` or Sentinel actions (if added).

### Troubleshooting
- **Deployment Errors**:
  - Ensure `logicAppName` is unique within the resource group to avoid conflicts (e.g., with "Scan Machine"). Consider using `MachineScan-AutoCloseIPs`.
  - Verify that `tenantId`, `clientId`, and `workspaceId` are provided during deployment (though `tenantId` and `clientId` are unused).
  - Check deployment logs in **Resource group** > **Deployments**.
- **Blob Storage Errors**:
  - Verify the storage account exists, the `knownips` container is created, and `knownips.json` is uploaded with the correct format.
  - Ensure the Logic App’s Managed Identity has the `Storage Blob Data Contributor` role.
- **Sentinel Errors**:
  - Confirm the `azuresentinel` connection is configured and the webhook is set up in Sentinel.
  - Ensure the Logic App’s Managed Identity has the `Microsoft Sentinel Contributor` role.
- **Condition Errors**:
  - Verify the `relatedEntities` schema in Sentinel incidents includes an `id` field for IPs. If the field name differs (e.g., `ipAddress`), update the `Condition` expression (e.g., `@items('For_each')?['ipAddress']`).
  - Check **Run history** for parsing or matching issues.

### Support
For additional help, contact [lungelo.busakwe@conosco.com](mailto:lungelo.busakwe@conosco.com).
