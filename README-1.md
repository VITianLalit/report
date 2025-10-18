# Azure VM Management Implementation Report

## Architecture Overview

### Two Implementation Approaches

1. **Azure SDK-Based Implementation** (`/vm/*` endpoints)
   - Uses Azure Python SDKs directly
   - Requires Service Principal authentication

2. **Azure CLI-Based Implementation** (`/vm-cli/*` endpoints)
   - Uses Azure CLI commands under the hood
   - Leverages existing Azure CLI login session

### Core Service Layer

#### 1. Azure SDK Service (`app/services/azure_vm_service.py`)
- **Purpose**: Core VM management using Azure Python SDKs
- **Key Features**:
  - Automatic credential detection (Managed Identity → Service Principal)
  - VM status checking via Azure Compute Management API
  - Command execution via Azure Run Command

#### 2. Azure CLI Service (`app/services/azure_cli_vm_service.py`)
- **Purpose**: VM management using Azure CLI commands
- **Key Features**:
  - Uses existing `az login` session
  - Cross-platform Azure CLI detection

### API Route Layer

#### 3. Azure SDK Routes (`app/routes/vm.py`)
- **Endpoints**:
  - `GET /vm/status` - VM connectivity and status
  - `POST /vm/execute` - PowerShell command execution
  - `GET /vm/health` - Service health check

#### 4. Azure CLI Routes (`app/routes/vm_cli.py`)
- **Endpoints**:
  - `GET /vm-cli/status` - VM connectivity and status
  - `POST /vm-cli/execute` - PowerShell command execution
  - `POST /vm-cli/execute-as-user` - Execute commands as specific user
  - `GET /vm-cli/health` - Service health check

### Schema and Validation Layer

#### 5. Pydantic Schemas (`app/schemas/vm.py`)
- **Models Created**:
  - `VMStatusResponse` - VM status information
  - `CommandExecutionRequest/Response` - Command execution data
  - `FileTransferRequest/Response` - File transfer operations
  - `ErrorResponse` - Standardized error responses

### Application Integration

#### 7. Main Application (`app/main.py`)
- Added VM router imports and registration
- Updated to include both `/vm` and `/vm-cli` endpoint groups

#### 8. Routes Package (`app/routes/__init__.py`)
- Exported new VM routers for application integration

#### 9. Schemas Package (`app/schemas/__init__.py`) 
- Exported new VM-related Pydantic models

## 🔧 Dependencies Added

### Core Azure SDKs
```txt
azure-identity>=1.15.0          # Authentication (Managed Identity, Service Principal)
azure-mgmt-compute>=32.0.0      # VM management and Run Command
azure-mgmt-resource>=23.1.0     # Resource management
tenacity>=8.2.0                 # Retry logic for resilience
```

## 🔄 How the APIs Work

### 1. VM Status Checking

#### Azure SDK Implementation
```python
# Uses Azure Compute Management Client
vm = compute_client.virtual_machines.get(
    resource_group_name, vm_name, expand='instanceView'
)
# Returns: power_state, provisioning_state, vm_size, location
```

#### Azure CLI Implementation  
```bash
# Uses Azure CLI command
az vm show --resource-group {rg} --name {vm} --show-details
# Parses JSON output for status information
```

### 2. Command Execution

Both implementations use **Azure Run Command** but through different interfaces:

#### Azure SDK Flow
1. Validate VM accessibility via Compute API
2. Create `RunCommandInput` with PowerShell script
3. Execute via `virtual_machines.begin_run_command()`
4. Wait for completion and parse results
5. Return stdout, stderr, exit code

#### Azure CLI Flow
1. Check VM status via `az vm show`
2. Execute command via `az vm run-command invoke`
3. Parse JSON output from CLI
4. Support for user impersonation with credentials

## How Each Implementation Works

### Azure SDK Implementation Theory

#### Authentication Flow
```
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│   Application   │───▶│  Azure Identity  │───▶│  Azure AD Auth  │
│                 │    │   (Credential    │    │   (Token Mgmt)  │
│                 │    │   Detection)     │    │                 │
└─────────────────┘    └──────────────────┘    └─────────────────┘
         │                        │                       │
         │              ┌─────────▼─────────┐            │
         │              │ Credential Chain: │            │
         │              │ 1. Managed ID     │            │
         │              │ 2. Service Prin.  │            │
         │              │ 3. Interactive    │            │
         │              └───────────────────┘            │
         │                                               │
         └─────────────────────JWT Token◄─────────────────┘
```

**Theoretical Process:**
1. **Credential Discovery**: The Azure Identity library attempts authentication in order:
   - **Managed Identity**: Checks if running in Azure (App Service, VM, etc.)
   - **Service Principal**: Uses AZURE_CLIENT_ID + AZURE_CLIENT_SECRET
   - **Interactive**: Falls back to user login (development only)

2. **Token Management**: Once authenticated, Azure Identity handles:
   - JWT token acquisition from Azure AD
   - Automatic token renewal before expiration
   - Token caching for performance
   - Multi-resource token scoping

#### VM Management API Flow
```
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│   FastAPI App   │───▶│  Azure SDK Mgmt  │───▶│   Azure ARM     │
│                 │    │   (Compute API)  │    │  (REST APIs)    │
│   VM Request    │    │                  │    │                 │
└─────────────────┘    └──────────────────┘    └─────────────────┘
         │                        │                       │
         │              ┌─────────▼─────────┐            │
         │              │  SDK Operations:  │            │
         │              │ • VM Status Query │            │
         │              │ • Run Command     │            │
         │              │ • Blob Operations │            │
         │              └───────────────────┘            │
         │                                               │
         └─────────────────Response JSON◄──────────────────┘
```

**Theoretical Operations:**

1. **VM Status Checking**:
   ```python
   # The SDK creates an HTTP request to:
   # GET https://management.azure.com/subscriptions/{id}/resourceGroups/{rg}/
   #     providers/Microsoft.Compute/virtualMachines/{name}?$expand=instanceView
   
   # Azure ARM API returns JSON with:
   # - provisioningState: "Succeeded" | "Failed" | "Updating"
   # - powerState: "running" | "stopped" | "deallocated"
   # - vmSize, location, network info
   ```

2. **Command Execution via Run Command**:
   ```python
   # The SDK sends POST request to:
   # /subscriptions/{id}/resourceGroups/{rg}/providers/Microsoft.Compute/
   # virtualMachines/{name}/runCommand
   
   # With payload:
   # {
   #   "commandId": "RunPowerShellScript",
   #   "script": ["Your-PowerShell-Command"],
   #   "parameters": []
   # }
   
   # Azure VM Agent receives command through secure channel
   # Executes PowerShell in isolated context
   # Returns stdout/stderr through same secure channel
   ```

3. **File Transfer Mechanism**:
   ```
   File Upload ──┐
                 ▼
   ┌─────────────────────────┐    ┌──────────────────────┐
   │   Azure Blob Storage    │───▶│    Generate SAS      │
   │   (Temporary Storage)   │    │   (1-hour expiry)    │
   └─────────────────────────┘    └──────────────────────┘
                 │                           │
                 │              ┌────────────▼────────────┐
                 │              │     SAS URL sent to     │
                 │              │     VM via Run Cmd      │
                 │              └─────────────────────────┘
                 │                           │
                 └───────────────────────────▼
                          VM Downloads File via 
                         PowerShell Invoke-WebRequest
   ```

### Azure CLI Implementation Theory

#### Authentication Flow
```
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│   Application   │───▶│    Azure CLI     │───▶│  Cached Tokens  │
│                 │    │  (az command)    │    │ (~/.azure/*)    │
│                 │    │                  │    │                 │
└─────────────────┘    └──────────────────┘    └─────────────────┘
         │                        │                       │
         │              ┌─────────▼─────────┐            │
         │              │  CLI Token Mgmt:  │            │
         │              │ • User Login      │            │
         │              │ • Service Prin.   │            │
         │              │ • Managed ID      │            │
         │              └───────────────────┘            │
         │                                               │
         └─────────────────Command Output◄─────────────────┘
```

**Theoretical Process:**
1. **Session Reuse**: CLI implementation leverages existing `az login` session
2. **Token Sharing**: Uses same token cache as interactive Azure CLI
3. **Command Execution**: Spawns subprocess to execute `az` commands
4. **JSON Parsing**: Parses structured output from CLI commands

#### VM Operations Flow
```
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│   FastAPI App   │───▶│    Subprocess    │───▶│   Azure CLI     │
│                 │    │  (az vm show)    │    │   (Binary)      │
│   CLI Request   │    │                  │    │                 │
└─────────────────┘    └──────────────────┘    └─────────────────┘
         │                        │                       │
         │              ┌─────────▼─────────┐            │
         │              │ Process Commands: │            │
         │              │ • az vm show      │            │
         │              │ • az vm run-cmd   │            │
         │              │ • JSON output     │            │
         │              └───────────────────┘            │
         │                                               │
         └─────────────────Parsed JSON◄────────────────────┘
```

**Theoretical Operations:**

1. **VM Status via CLI**:
   ```bash
   # Command executed:
   az vm show --resource-group <resource-group> --name <vm-name> --show-details --output json
   
   # CLI internally makes same REST API calls as SDK
   # Returns JSON that gets parsed by Python application
   ```

2. **Command Execution via CLI**:
   ```bash
   # Command executed:
   az vm run-command invoke \
     --resource-group <resource-group> \
     --name <vm-name> \
     --command-id RunPowerShellScript \
     --scripts "Your-PowerShell-Command"
   
   # Same Azure Run Command mechanism as SDK
   # CLI handles authentication and API calls
   ```

3. **File Transfer via Base64**:
   ```
   File Read ──┐
               ▼
   ┌─────────────────────────┐    ┌──────────────────────┐
   │   Base64 Encoding       │───▶│   PowerShell Script  │
   │   (Python Application) │    │   Generation         │
   └─────────────────────────┘    └──────────────────────┘
                 │                           │
                 │              ┌────────────▼────────────┐
                 │              │  Script sent via CLI    │
                 │              │  az vm run-command      │
                 │              └─────────────────────────┘
                 │                           │
                 └───────────────────────────▼
                          VM Decodes Base64 and
                         Creates File via PowerShell
   ```

### Comparison: SDK vs CLI Approaches

#### Performance Theory
```
Azure SDK:
┌─────────────┐    Direct    ┌─────────────┐
│ Python App  │──────────────▶│ Azure APIs  │
└─────────────┘    HTTPS     └─────────────┘
    ~50-100ms latency per operation

Azure CLI:
┌─────────────┐  Subprocess  ┌─────────────┐  HTTPS   ┌─────────────┐
│ Python App  │─────────────▶│ Azure CLI   │─────────▶│ Azure APIs  │
└─────────────┘              └─────────────┘          └─────────────┘
    ~100-200ms latency (process overhead + API call)
```

#### Security Model Theory
```
Azure SDK Security:
• Direct TLS connection to Azure
• Certificate validation
• JWT token authentication
• No intermediate processes

Azure CLI Security:
• TLS via CLI binary
• Token reuse from CLI cache
• Process isolation
• Command injection protection
```

## 📋 Environment Configuration & Variable Definitions

### VM Configuration Environment Variables

Each environment variable serves a specific purpose in the Azure VM management system:

#### Core Azure Identity Variables

**`AZURE_TENANT_ID`** = `<your-tenant-id>`
```
Purpose: Azure Active Directory (Azure AD) Tenant Identifier
Usage: Specifies which Azure AD tenant to authenticate against
Theory: 
- Every Azure subscription belongs to an Azure AD tenant
- This GUID identifies your organization's Azure AD instance
- Used by both SDK and CLI for authentication scope
- Required for Service Principal authentication
- Maps to: Your organization's Azure AD tenant
```

**`AZURE_SUBSCRIPTION_ID`** = `<your-subscription-id>`
```
Purpose: Azure Subscription Identifier
Usage: Targets specific Azure subscription for all operations
Theory:
- Azure resources are contained within subscriptions
- This GUID determines billing and access scope
- Required for all Azure Resource Manager (ARM) API calls
- Used to construct REST API URLs: /subscriptions/{subscription-id}/...
- Maps to: Your specific Azure subscription
```

**`AZURE_CLIENT_ID`** = `<your-client-id>`
```
Purpose: Service Principal Application (Client) Identifier
Usage: Identifies the application registration in Azure AD
Theory:
- Service Principals are non-human identities for automation
- This GUID identifies your specific app registration
- Used for OAuth 2.0 client credentials flow
- Paired with AZURE_CLIENT_SECRET for authentication
- Alternative to using personal user accounts
```

**`AZURE_CLIENT_SECRET`** = `<your-client-secret>`
```
Purpose: Service Principal Secret Key
Usage: Proves identity of the application to Azure AD
Theory:
- Acts as a password for the Service Principal
- Used in OAuth 2.0 token exchange
- Has expiration date (typically 1-2 years)
- Should be rotated regularly for security
- Grants access based on assigned RBAC roles
Security Note: Never commit this to version control
```

#### Resource Targeting Variables

**`AZURE_RESOURCE_GROUP_NAME`** = `<your-resource-group>`
```
Purpose: Azure Resource Group Name
Usage: Logical container for your Azure resources
Theory:
- Resource Groups organize related Azure resources
- All resources in a group share lifecycle and permissions
- Used in Azure REST API paths: /resourceGroups/{name}/...
- Provides management boundary for resources
- Maps to: Your resource group containing the VM
```

**`AZURE_VM_NAME`** = `<your-vm-name>`
```
Purpose: Target Virtual Machine Name
Usage: Identifies the specific VM for all operations
Theory:
- VM names are unique within a resource group
- Used to construct Azure Resource Manager resource IDs
- Required for all VM-specific operations (status, commands, etc.)
- Forms part of REST API paths: /virtualMachines/{name}
- Maps to: Your Windows VM for remote operations
```

#### Storage Configuration (SDK Implementation)

**`AZURE_STORAGE_ACCOUNT_NAME`** = `<your-storage-account>`
```
Purpose: Azure Storage Account for File Transfers
Usage: Temporary file storage during VM file transfer operations
Theory:
- Blob Storage provides secure, scalable file storage
- Used as intermediate storage for large file transfers
- Enables SAS (Shared Access Signature) URLs for secure access
- Files uploaded here, then downloaded by VM via PowerShell
- Auto-cleanup prevents storage bloat
Storage Pattern:
1. Upload file to blob storage
2. Generate time-limited SAS URL
3. VM downloads using SAS URL
4. Delete blob after transfer
```

**`AZURE_STORAGE_CONTAINER_NAME`** = `vm-transfers` (Default)
```
Purpose: Blob Container within Storage Account
Usage: Logical folder structure within blob storage
Theory:
- Containers organize blobs (files) within storage accounts
- Provide security boundary for blob access
- Support container-level permissions and policies
- Used to namespace temporary transfer files
- Default: "vm-transfers" (can be customized)
```

### Authentication Flow Mapping

```
Environment Variables → Authentication Method:

With AZURE_CLIENT_ID + AZURE_CLIENT_SECRET:
┌─────────────────────────┐
│   Service Principal     │ ← Automated, production-ready
│   Authentication        │
└─────────────────────────┘

Without CLIENT_ID/SECRET:
┌─────────────────────────┐
│   Managed Identity      │ ← When running in Azure
│   (Auto-detection)     │
└─────────────────────────┘
            OR
┌─────────────────────────┐
│   Azure CLI Session    │ ← Uses `az login` tokens
│   (CLI Implementation) │
└─────────────────────────┘
```

### Variable Usage by Implementation

#### Azure SDK Implementation
```
Required Variables:
✅ AZURE_TENANT_ID          → OAuth tenant scope
✅ AZURE_SUBSCRIPTION_ID    → API endpoint construction  
✅ AZURE_RESOURCE_GROUP_NAME → Resource targeting
✅ AZURE_VM_NAME           → VM identification
✅ AZURE_CLIENT_ID         → Service Principal auth
✅ AZURE_CLIENT_SECRET     → Service Principal auth
✅ AZURE_STORAGE_ACCOUNT_NAME → File transfer storage

Optional Variables:
⚠️ AZURE_STORAGE_CONTAINER_NAME (defaults to "vm-transfers")
```

#### Azure CLI Implementation
```
Required Variables:
✅ AZURE_SUBSCRIPTION_ID    → CLI command targeting
✅ AZURE_RESOURCE_GROUP_NAME → CLI command parameters
✅ AZURE_VM_NAME           → CLI command parameters

Optional Variables:
⚠️ AZURE_TENANT_ID         → CLI uses active session
⚠️ AZURE_CLIENT_ID         → CLI uses active session  
⚠️ AZURE_CLIENT_SECRET     → CLI uses active session
❌ AZURE_STORAGE_ACCOUNT_NAME → Not needed (uses base64)
```

### Security Implications by Variable

| Variable | Security Level | Rotation Required | Storage Location |
|----------|----------------|-------------------|------------------|
| `AZURE_TENANT_ID` | Low | No | Public (non-sensitive) |
| `AZURE_SUBSCRIPTION_ID` | Low | No | Public (non-sensitive) |  
| `AZURE_RESOURCE_GROUP_NAME` | Low | No | Public (non-sensitive) |
| `AZURE_VM_NAME` | Low | No | Public (non-sensitive) |
| `AZURE_CLIENT_ID` | Medium | Rarely | Can be public |
| `AZURE_CLIENT_SECRET` | **HIGH** | Yes (1-2 years) | **SECRET** (Azure Key Vault) |
| `AZURE_STORAGE_ACCOUNT_NAME` | Low | No | Public (non-sensitive) |

# Storage for SDK-based file transfers
AZURE_STORAGE_ACCOUNT_NAME=<your-storage-account>
```

## 🎯 Current Status

### ✅ Successfully Implemented
- [x] VM status checking (both SDK and CLI)
- [x] PowerShell command execution (both approaches)
- [x] User impersonation (CLI only)
- [x] Configuration validation and error handling
- [x] Comprehensive API documentation

