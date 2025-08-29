# Complete AAP 2.5 Implementation Guide for VM Patching Workflow

This guide provides step-by-step instructions to implement your 8-playbook VM patching workflow in Ansible Automation Platform 2.5, assuming no prior Ansible or AAP experience.

## Table of Contents

1. [Prerequisites and Environment Setup](#prerequisites)
2. [AAP 2.5 Installation and Initial Access](#installation)
3. [Understanding AAP 2.5 Interface](#interface)
4. [Initial Platform Configuration](#configuration)
5. [Creating Organizations and Users](#organizations)
6. [Setting Up Credentials](#credentials)
7. [Creating Projects](#projects)
8. [Building Inventories](#inventories)
9. [Creating Job Templates](#job-templates)
10. [Building the Workflow Template](#workflow)
11. [Testing and Validation](#testing)
12. [Production Deployment](#production)
13. [Troubleshooting Common Issues](#troubleshooting)

## Prerequisites and Environment Setup {#prerequisites}

### What You'll Need Before Starting

**Infrastructure Requirements:**
- AAP 2.5 installed and running (minimum 16GB RAM, 4 vCPUs)
- Network access to target RHEL systems
- VMware vCenter server access (for VM operations)
- GitLab repository (for storing playbooks and reports)
- Admin access to AAP 2.5 web interface

**Account Requirements:**
- AAP administrative account
- VMware vCenter service account with VM management permissions
- GitLab account with API token
- SSH access to target Linux systems

**Network Connectivity:**
- Port 443 HTTPS access to AAP controller
- Port 22 SSH access to target systems
- Port 443 HTTPS access to vCenter
- Port 443 HTTPS access to GitLab

### Understanding the Workflow

Your VM patching workflow consists of 8 sequential playbooks:

1. **00-discovery.yml**: Discovers VMs in vCenter and validates access
2. **01-contact.yml**: Tests SSH connectivity to discovered systems
3. **02-snapshot_init.yml**: Creates VM snapshots before patching
4. **03-snapshot_verify.yml**: Verifies snapshots were created successfully
5. **04-patch.yml**: Applies system updates using package managers
6. **05-reboot.yml**: Reboots systems and validates connectivity
7. **06-healthcheck.yml**: Performs post-patch health validation
8. **07-report.yml**: Generates HTML and CSV reports
9. **08-publish.yml**: Publishes reports to GitLab repository

## AAP 2.5 Installation and Initial Access {#installation}

### Accessing AAP 2.5 for the First Time

1. **Open your web browser** and navigate to your AAP controller URL:
   ```
   https://your-aap-controller.company.com
   ```

2. **Log in** using your administrator credentials provided during installation

3. **Accept the license agreement** if prompted

4. **Verify installation** by checking that all services show as "Healthy" in the dashboard

### Understanding the AAP 2.5 Interface Layout

The AAP 2.5 interface is organized into main sections accessible from the left navigation:

- **Overview**: Dashboard showing system status and recent activity
- **Jobs**: View running and completed automation jobs
- **Schedules**: Manage automated job execution
- **Activity Stream**: Audit log of all system activities
- **Workflow Approvals**: Pending manual approvals in workflows
- **Automation Execution**: Core automation components
- **Automation Decisions**: Event-driven automation rules
- **Administration**: User management and system settings
- **Access Management**: Users, teams, and permissions

## Understanding AAP 2.5 Interface {#interface}

### Navigation Overview

**Primary Navigation (Left Sidebar):**
```
├── Overview (Dashboard)
├── Jobs (Execution history)
├── Schedules (Automated runs)
├── Activity Stream (Audit log)
└── Workflow Approvals (Manual gates)

├── Automation Execution
│   ├── Templates (Job & Workflow templates)
│   ├── Jobs (Individual executions)
│   ├── Projects (Git repositories)
│   ├── Inventories (Host collections)
│   ├── Credentials (Authentication)
│   └── Execution Environments (Containers)

├── Administration
│   ├── Activity Stream
│   ├── Management Jobs
│   ├── Notifications
│   └── Settings

└── Access Management
    ├── Organizations
    ├── Users
    ├── Teams
    └── Credentials
```

## Initial Platform Configuration {#configuration}

### Step 1: Configure System Settings

1. **Navigate to Administration → Settings**

2. **Configure System Settings:**
   ```yaml
   - Click "System" in the left sidebar
   - Update these critical settings:
     * System Base URL: https://your-aap-controller.company.com
     * Activity Stream Enabled: ON (for auditing)
     * Debug Information: OFF (for production)
     * Custom Base Path: / (leave default)
   - Click "Save"
   ```

3. **Configure Job Settings:**
   ```yaml
   - Click "Jobs" in settings sidebar
   - Update these values:
     * Default Job Timeout: 0 (no timeout)
     * Default Inventory Update Timeout: 0
     * Maximum Number of Forks: 200
     * Maximum Number of Hosts per Organization: 1000
   - Click "Save"
   ```

### Step 2: Configure Authentication (Optional)

If using LDAP/Active Directory:

1. **Navigate to Administration → Settings → Authentication**
2. **Click "LDAP"**
3. **Configure LDAP settings:**
   ```yaml
   LDAP Server URI: ldaps://dc.company.com:636
   LDAP Bind DN: CN=ansible-service,OU=Service Accounts,DC=company,DC=com
   LDAP User Search: OU=Users,DC=company,DC=com
   LDAP Group Search: OU=Groups,DC=company,DC=com
   ```
4. **Click "Save"**

## Creating Organizations and Users {#organizations}

### Step 1: Create Your Organization

Organizations provide the top-level boundary for all AAP resources.

1. **Navigate to Access Management → Organizations**

2. **Click the "Create organization" button**

3. **Fill in organization details:**
   ```yaml
   Name: Production Automation
   Description: Production VM patching and automation workflows
   Max Hosts: 1000 (adjust based on your license)
   Default Execution Environment: Default execution environment
   Galaxy Credentials: (leave blank for now)
   ```

4. **Click "Create"**

### Step 2: Create Teams

Teams simplify permission management by grouping users.

1. **Navigate to Access Management → Teams**

2. **Click "Create team"**

3. **Create the following teams:**

   **Operations Team:**
   ```yaml
   Name: Operations Team
   Description: Day-to-day operations staff
   Organization: Production Automation
   ```

   **Automation Engineers:**
   ```yaml
   Name: Automation Engineers  
   Description: Automation development and maintenance
   Organization: Production Automation
   ```

   **Read-Only Auditors:**
   ```yaml
   Name: Read-Only Auditors
   Description: Compliance and audit access
   Organization: Production Automation
   ```

### Step 3: Create Users (If Not Using LDAP)

1. **Navigate to Access Management → Users**

2. **Click "Create user"**

3. **Create an operations user:**
   ```yaml
   Username: ops-user
   Email: ops-user@company.com
   First Name: Operations
   Last Name: User
   Password: [secure password]
   Confirm Password: [secure password]
   User Type: Normal User
   Organization: Production Automation
   ```

4. **After creation, assign to teams:**
   - Click on the user
   - Click "Teams" tab
   - Click "Associate"
   - Select "Operations Team"
   - Click "Save"

## Setting Up Credentials {#credentials}

Credentials securely store authentication information for external systems.

### Step 1: Create SSH Machine Credential

1. **Navigate to Resources → Credentials** (under Automation Execution)

2. **Click "Create credential"**

3. **Configure SSH credential:**
   ```yaml
   Name: Production SSH Key
   Description: SSH access to production Linux systems
   Organization: Production Automation
   Credential Type: Machine
   Username: ansible (or your service account)
   SSH Private Key: [paste your private key content]
   Private Key Passphrase: [if your key has a passphrase]
   Privilege Escalation Method: sudo
   Privilege Escalation Username: root
   Privilege Escalation Password: [if sudo requires password]
   ```

4. **Click "Create"**

### Step 2: Create VMware vCenter Credential

1. **Click "Create credential" again**

2. **Configure vCenter credential:**
   ```yaml
   Name: Production vCenter
   Description: VMware vCenter access for VM operations
   Organization: Production Automation
   Credential Type: VMware vCenter
   vCenter Host: vcenter.company.com
   Username: ansible-service@vsphere.local
   Password: [vCenter service account password]
   ```

3. **Click "Create"**

### Step 3: Create GitLab Credential

1. **Click "Create credential" again**

2. **Configure GitLab credential:**
   ```yaml
   Name: GitLab API Token
   Description: GitLab access for repository and report publishing
   Organization: Production Automation
   Credential Type: GitLab Personal Access Token
   Token: [your GitLab personal access token]
   ```

   **Note**: Generate GitLab token at GitLab → Settings → Access Tokens with `read_repository`, `write_repository`, and `api` scopes.

3. **Click "Create"**

### Step 4: Create Ansible Vault Credential (For Sensitive Data)

1. **Click "Create credential" again**

2. **Configure vault credential:**
   ```yaml
   Name: Production Vault Password
   Description: Ansible Vault encryption for sensitive variables
   Organization: Production Automation  
   Credential Type: Vault
   Vault Password: [your vault password]
   Vault Identifier: prod_vault
   ```

3. **Click "Create"**

## Creating Projects {#projects}

Projects connect AAP to your Git repositories containing Ansible playbooks.

### Step 1: Create the VM Patching Project

1. **Navigate to Automation Execution → Projects**

2. **Click "Create project"**

3. **Configure project settings:**
   ```yaml
   Name: VM Patching Automation
   Description: VM patching workflow playbooks and automation
   Organization: Production Automation
   Execution Environment: Default execution environment
   Source Control Type: Git
   Source Control URL: https://gitlab.company.com/automation/vm-patching.git
   Source Control Branch: main (or leave blank for default)
   Source Control Credential: GitLab API Token (if private repo)
   ```

4. **Configure update options:**
   ```yaml
   ✓ Update Revision on Launch (recommended for production)
   ✓ Delete on Update (clean working directory)
   □ Cache Timeout (0 = always update)
   □ Allow Branch Override (enable if you want job-level branch selection)
   ```

5. **Click "Create"**

6. **Wait for project sync** - You'll see a spinning icon that changes to green checkmark when complete

### Step 2: Verify Project Contents

1. **Click on your created project**
2. **Click the "Jobs" tab** to see sync history
3. **Verify all 8 playbooks are visible** in the project

## Building Inventories {#inventories}

Inventories define the systems your automation will manage.

### Step 1: Create Static Inventory for Testing

1. **Navigate to Automation Execution → Inventories**

2. **Click "Create inventory"**

3. **Configure basic inventory:**
   ```yaml
   Name: Production RHEL Fleet
   Description: Production Red Hat Linux systems for patching
   Organization: Production Automation
   Instance Groups: (leave default)
   ```

4. **Click "Create"**

### Step 2: Add Hosts to Inventory

1. **Click on your created inventory**

2. **Click the "Hosts" tab**

3. **Click "Create host"**

4. **Add your first host:**
   ```yaml
   Host Name: server-m9001.company.com
   Description: Production server M9001
   Variables (in YAML format):
   ---
   ansible_host: 10.1.1.10
   vm_name: Server-m9001
   ansible_user: ansible
   ```

5. **Repeat for additional hosts:**
   ```yaml
   Host Name: server-t0002.company.com
   Variables:
   ---
   ansible_host: 10.1.1.11
   vm_name: Server-t0002
   ansible_user: ansible
   ```

   ```yaml
   Host Name: server-t0004.company.com
   Variables:
   ---
   ansible_host: 10.1.1.12
   vm_name: Server-t0004
   ansible_user: ansible
   ```

### Step 3: Create Groups (Optional but Recommended)

1. **Click the "Groups" tab**

2. **Click "Create group"**

3. **Create environment groups:**
   ```yaml
   Name: production
   Description: Production environment systems
   Variables:
   ---
   environment: production
   maintenance_window: "weekend"
   ```

4. **Create functional groups:**
   ```yaml
   Name: web_servers
   Description: Web application servers
   
   Name: database_servers  
   Description: Database servers
   ```

5. **Associate hosts with groups:**
   - Click on a group
   - Click "Hosts" tab
   - Click "Associate"
   - Select hosts to add to group

## Creating Job Templates {#job-templates}

Job templates define how individual playbooks execute. We'll create one for each of your 8 playbooks.

### Step 1: Create Discovery Job Template

1. **Navigate to Automation Execution → Templates**

2. **Click "Create template" → "Create job template"**

3. **Configure discovery template:**
   ```yaml
   Name: 01-VM-Discovery
   Description: Discover VMs in vCenter and validate access
   Job Type: Run
   Inventory: Production RHEL Fleet
   Project: VM Patching Automation  
   Playbook: 00-discovery.yml
   Execution Environment: Default execution environment
   Credentials: 
     - Production SSH Key
     - Production vCenter
   Forks: 10
   Limit: (leave blank to run on all hosts)
   Verbosity: 1 (Verbose)
   Job Tags: (leave blank)
   Skip Tags: (leave blank)
   ```

4. **Configure extra variables:**
   ```yaml
   ---
   vcenter_server: vcenter.company.com
   vcenter_user: ansible-service@vsphere.local
   vcenter_pass: "{{ vcenter_password }}"
   datacenter_name: "Production-DC"
   vmware_validate_certs: false
   ```

5. **Configure options:**
   ```yaml
   ✓ Privilege Escalation (Enable sudo)
   □ Provisioning Callbacks
   □ Enable Webhook
   □ Enable Concurrent Jobs
   □ Use Fact Cache
   □ Prompt on Launch: Variables (enable for runtime variable changes)
   □ Prompt on Launch: Inventory (optional)
   □ Prompt on Launch: Credentials (optional)
   ```

6. **Click "Create"**

### Step 2: Create Contact Job Template

1. **Click "Create template" → "Create job template"**

2. **Configure contact template:**
   ```yaml
   Name: 02-SSH-Contact-Check
   Description: Test SSH connectivity to discovered systems
   Job Type: Run
   Inventory: Production RHEL Fleet
   Project: VM Patching Automation
   Playbook: 01-contact.yml
   Execution Environment: Default execution environment
   Credentials: Production SSH Key
   Forks: 10
   Verbosity: 1 (Verbose)
   ```

3. **Configure extra variables:**
   ```yaml
   ---
   contact_timeout: 120
   ```

4. **Configure options:**
   ```yaml
   ✓ Privilege Escalation
   □ Prompt on Launch: Variables
   ```

5. **Click "Create"**

### Step 3: Create Snapshot Job Template

1. **Create new job template with these settings:**
   ```yaml
   Name: 03-VM-Snapshot-Creation
   Description: Create VM snapshots before patching
   Job Type: Run
   Inventory: Production RHEL Fleet
   Project: VM Patching Automation
   Playbook: 02-snapshot_init.yml
   Execution Environment: Default execution environment
   Credentials:
     - Production SSH Key
     - Production vCenter
   Forks: 5 (lower for vCenter operations)
   Verbosity: 1 (Verbose)
   ```

2. **Extra variables:**
   ```yaml
   ---
   vcenter_server: vcenter.company.com
   vcenter_user: ansible-service@vsphere.local
   vcenter_pass: "{{ vcenter_password }}"
   datacenter_name: "Production-DC"
   vmware_validate_certs: false
   snapshot_label: "Patching"
   snapshot_quiesce: false
   snapshot_memory_dump: true
   snapshot_async_timeout: 10800
   snapshot_poll_interval: 60
   ```

### Step 4: Create Snapshot Verification Template

1. **Create job template:**
   ```yaml
   Name: 04-VM-Snapshot-Verification
   Description: Verify snapshots were created successfully
   Job Type: Run
   Inventory: Production RHEL Fleet
   Project: VM Patching Automation
   Playbook: 03-snapshot_verify.yml
   Credentials:
     - Production SSH Key
     - Production vCenter
   Forks: 5
   Verbosity: 1
   ```

2. **Extra variables:**
   ```yaml
   ---
   vcenter_server: vcenter.company.com
   vcenter_user: ansible-service@vsphere.local
   vcenter_pass: "{{ vcenter_password }}"
   datacenter_name: "Production-DC"
   vmware_validate_certs: false
   ```

### Step 5: Create Patch Job Template

1. **Create job template:**
   ```yaml
   Name: 05-System-Patching
   Description: Apply system updates using package managers
   Job Type: Run
   Inventory: Production RHEL Fleet
   Project: VM Patching Automation
   Playbook: 04-patch.yml
   Credentials: Production SSH Key
   Forks: 10
   Verbosity: 1
   ```

2. **Extra variables:**
   ```yaml
   ---
   patch_async_timeout: 14400
   patch_poll_interval: 60
   patch_strategy: "auto"
   ```

3. **Options:**
   ```yaml
   ✓ Privilege Escalation (Required for package installation)
   ✓ Prompt on Launch: Variables (for different patch strategies)
   ```

### Step 6: Create Reboot Job Template

1. **Create job template:**
   ```yaml
   Name: 06-System-Reboot
   Description: Reboot systems and validate connectivity
   Job Type: Run
   Inventory: Production RHEL Fleet
   Project: VM Patching Automation
   Playbook: 05-reboot.yml
   Credentials: Production SSH Key
   Forks: 10
   Verbosity: 1
   ```

2. **Extra variables:**
   ```yaml
   ---
   reboot_timeout: 1800
   pre_reboot_delay: 0
   post_reboot_delay: 0
   ```

3. **Options:**
   ```yaml
   ✓ Privilege Escalation (Required for reboot)
   ```

### Step 7: Create Health Check Job Template

1. **Create job template:**
   ```yaml
   Name: 07-Post-Patch-Health-Check
   Description: Perform post-patch health validation
   Job Type: Run
   Inventory: Production RHEL Fleet
   Project: VM Patching Automation
   Playbook: 06-healthcheck.yml
   Credentials: Production SSH Key
   Forks: 10
   Verbosity: 1
   ```

2. **Extra variables:**
   ```yaml
   ---
   disk_util_warn: 80
   ```

3. **Options:**
   ```yaml
   ✓ Privilege Escalation
   ```

### Step 8: Create Report Generation Template

1. **Create job template:**
   ```yaml
   Name: 08-Report-Generation
   Description: Generate HTML and CSV reports
   Job Type: Run
   Inventory: Production RHEL Fleet (localhost only)
   Project: VM Patching Automation
   Playbook: 07-report.yml
   Credentials: Production SSH Key
   Forks: 1
   Verbosity: 1
   Limit: localhost
   ```

2. **Extra variables:**
   ```yaml
   ---
   # Report generation doesn't need extra variables
   # All data comes from previous workflow steps
   ```

### Step 9: Create Publish Template

1. **Create job template:**
   ```yaml
   Name: 09-Report-Publishing
   Description: Publish reports to GitLab repository
   Job Type: Run
   Inventory: Production RHEL Fleet (localhost only)
   Project: VM Patching Automation
   Playbook: 08-publish.yml
   Credentials: 
     - Production SSH Key
     - GitLab API Token
   Forks: 1
   Verbosity: 1
   Limit: localhost
   ```

2. **Extra variables:**
   ```yaml
   ---
   gitlab_repo_url: "https://gitlab.company.com/ops/patch-reports.git"
   gitlab_token: "{{ gitlab_api_token }}"
   gitlab_branch: "main"
   gitlab_user_name: "AAP Automation"
   gitlab_user_email: "aap-automation@company.com"
   publish_base_path: "patch-reports"
   ```

3. **Options:**
   ```yaml
   ✓ Prompt on Launch: Variables (for different repositories)
   ```

## Building the Workflow Template {#workflow}

Now we'll create a workflow template that orchestrates all 8 job templates in sequence with proper error handling.

### Step 1: Create Workflow Template

1. **Navigate to Automation Execution → Templates**

2. **Click "Create template" → "Create workflow template"**

3. **Configure workflow:**
   ```yaml
   Name: Complete VM Patching Workflow
   Description: End-to-end VM patching with snapshots, updates, and reporting
   Organization: Production Automation
   Inventory: Production RHEL Fleet (optional default)
   Limit: (leave blank)
   ```

4. **Extra variables for entire workflow:**
   ```yaml
   ---
   # Global workflow variables
   environment: "production"
   maintenance_window: "{{ ansible_date_time.iso8601 }}"
   notification_email: "ops-team@company.com"
   
   # VMware settings
   vcenter_server: "vcenter.company.com"
   vcenter_user: "ansible-service@vsphere.local"  
   vcenter_pass: "{{ vcenter_password }}"
   datacenter_name: "Production-DC"
   vmware_validate_certs: false
   
   # GitLab settings
   gitlab_repo_url: "https://gitlab.company.com/ops/patch-reports.git"
   gitlab_token: "{{ gitlab_api_token }}"
   gitlab_branch: "main"
   ```

5. **Options:**
   ```yaml
   ✓ Prompt on Launch: Variables (for runtime customization)
   ✓ Prompt on Launch: Inventory (for different host sets)
   □ Enable Webhook
   □ Enable Concurrent Jobs
   ```

6. **Click "Create"**

### Step 2: Build Workflow in Visualizer

After creating the workflow template, the **Workflow Visualizer** opens automatically.

1. **Click "Start" to add first node**

2. **Add Discovery Node:**
   ```yaml
   Node Type: Job Template
   Job Template: 01-VM-Discovery
   Convergence: Any (first node)
   ```
   - Click "Save"

3. **Add success path from Discovery:**
   - Click on Discovery node
   - Click "+" to add next node
   - Select "On Success" 
   ```yaml
   Node Type: Job Template
   Job Template: 02-SSH-Contact-Check
   Convergence: Any
   ```

4. **Continue building the success path:**

   **Discovery → Contact:**
   ```yaml
   Edge Type: On Success
   Node: 02-SSH-Contact-Check
   ```

   **Contact → Snapshot:**
   ```yaml
   Edge Type: On Success
   Node: 03-VM-Snapshot-Creation
   ```

   **Snapshot → Verification:**
   ```yaml
   Edge Type: On Success
   Node: 04-VM-Snapshot-Verification
   ```

   **Verification → Patching:**
   ```yaml
   Edge Type: On Success
   Node: 05-System-Patching
   ```

   **Patching → Reboot:**
   ```yaml
   Edge Type: On Success
   Node: 06-System-Reboot
   ```

   **Reboot → Health Check:**
   ```yaml
   Edge Type: On Success
   Node: 07-Post-Patch-Health-Check
   ```

   **Health Check → Report:**
   ```yaml
   Edge Type: On Success
   Node: 08-Report-Generation
   ```

   **Report → Publish:**
   ```yaml
   Edge Type: On Success
   Node: 09-Report-Publishing
   ```

### Step 3: Add Failure Handling

1. **Add failure notification job** (create this template first):
   ```yaml
   Name: Workflow-Failure-Notification
   Description: Send notification when workflow fails
   Playbook: [create a simple notification playbook]
   ```

2. **Add failure edges from each critical step:**
   - Click on Discovery node
   - Click "+" 
   - Select "On Failure"
   - Add: Workflow-Failure-Notification

3. **Add "Always" edges for cleanup:**
   - From final nodes, add "Always" edges to 08-Report-Generation
   - This ensures reports are generated even if workflow fails

### Step 4: Advanced Workflow Configuration

1. **Add conditional branching** for different environments:
   ```yaml
   # Add approval nodes for production
   Node Type: Approval
   Name: Production Approval Required
   Description: Manual approval required for production systems
   Timeout: 3600 (1 hour)
   ```

2. **Add convergence points** where multiple paths meet:
   ```yaml
   Convergence: All (wait for all previous nodes)
   Convergence: Any (proceed when any previous node completes)
   ```

## Testing and Validation {#testing}

### Step 1: Test Individual Job Templates

Before running the full workflow, test each job template individually.

1. **Test Discovery Template:**
   - Navigate to Templates
   - Click on "01-VM-Discovery"
   - Click "Launch"
   - Monitor job output for errors
   - Verify variables are passed correctly

2. **Check job output:**
   ```yaml
   Look for these success indicators:
   - VM information retrieved from vCenter
   - discovery_success and discovery_failure groups populated
   - No authentication errors
   ```

3. **Test each subsequent template** following the same process

### Step 2: Test Workflow Template

1. **Launch workflow with limited scope:**
   - Navigate to your workflow template
   - Click "Launch"
   - In the survey/prompt, set:
     ```yaml
     Limit: server-m9001.company.com (test single host first)
     ```

2. **Monitor workflow progress:**
   - Watch the visualizer show real-time progress
   - Click on individual nodes to see job details
   - Check for proper variable passing between jobs

3. **Verify workflow behavior:**
   ```yaml
   Expected flow for successful run:
   Discovery (Success) → Contact (Success) → Snapshot (Success) → 
   Verification (Success) → Patching (Success) → Reboot (Success) → 
   Health Check (Success) → Report → Publish
   ```

### Step 3: Test Failure Scenarios

1. **Test with unreachable host:**
   ```yaml
   Limit: nonexistent-server.company.com
   Expected: Discovery fails → Failure notification triggered
   ```

2. **Test with SSH connectivity issues:**
   ```yaml
   Temporarily break SSH access to test host
   Expected: Contact fails → Workflow stops → Notification sent
   ```

## Production Deployment {#production}

### Step 1: Create Production Inventory

1. **Import production hosts:**
   ```yaml
   Method 1: Bulk import via YAML file
   Method 2: Import from existing inventory file
   Method 3: Add hosts individually (for smaller environments)
   ```

2. **Example bulk import YAML:**
   ```yaml
   all:
     children:
       production:
         children:
           web_servers:
             hosts:
               web01.prod.company.com:
                 ansible_host: 10.1.1.10
                 vm_name: web01-prod
               web02.prod.company.com:
                 ansible_host: 10.1.1.11
                 vm_name: web02-prod
           database_servers:
             hosts:
               db01.prod.company.com:
                 ansible_host: 10.1.2.10
                 vm_name: db01-prod
         vars:
           environment: production
           maintenance_window: weekend
   ```

### Step 2: Configure Production Schedules

1. **Navigate to Schedules**

2. **Create maintenance window schedule:**
   ```yaml
   Name: Monthly Patch Schedule
   Description: Automated patching every second Saturday
   Template: Complete VM Patching Workflow
   Schedule Type: Rule-based
   Rules: 
     - Frequency: Monthly
     - Day: Second Saturday
     - Time: 02:00 AM
     - Timezone: America/New_York
   ```

### Step 3: Configure Notifications

1. **Navigate to Administration → Notifications**

2. **Create email notification:**
   ```yaml
   Name: Patch Workflow Notifications
   Type: Email
   Host: smtp.company.com
   Port: 587
   Username: aap-notifications@company.com
   Password: [email password]
   Recipient List: ops-team@company.com, management@company.com
   Use TLS: Yes
   ```

3. **Associate notifications with templates:**
   - Edit workflow template
   - Add notifications for:
     * Success
     * Failure
     * Start

### Step 4: Implement Approval Gates

1. **Add approval nodes** for production workflows:
   ```yaml
   Approval Node Configuration:
   Name: Production Patching Approval
   Description: Requires management approval for production patching
   Timeout: 86400 (24 hours)
   ```

2. **Configure approvers:**
   - Assign approval permissions to management team
   - Set up notification for pending approvals

### Step 5: Set Up Monitoring and Alerting

1. **Configure job monitoring:**
   ```yaml
   Navigate to Administration → Settings → System
   Configure:
   - Maximum number of simultaneous jobs: 100
   - Job timeout: 3600 (1 hour default)
   - Activity stream days: 30
   ```

2. **Set up external monitoring** (if available):
   ```yaml
   Prometheus integration:
   - Enable metrics endpoint
   - Configure scraping interval
   - Set up alerting rules
   ```

## Troubleshooting Common Issues {#troubleshooting}

### Issue 1: "hosts field is required and cannot have empty values"

**Problem**: Workflow fails at snapshot step with empty host list.

**Cause**: Variable passing between job templates not working properly.

**Solution**:
```yaml
1. Check that discovery job completed successfully
2. Verify discovery_success variable is populated:
   - Go to completed discovery job
   - Check "Extra Variables" output
   - Look for discovery_success: [list of hosts]

3. Ensure contact job uses correct host pattern:
   - hosts: "{{ discovery_success | join(',') }}"
   
4. If using original playbooks, consider upgrading to fixed versions
   that use discovery_success_hosts instead of discovery_success
```

### Issue 2: VMware Authentication Failures

**Problem**: vCenter operations fail with authentication errors.

**Cause**: Incorrect credentials or insufficient permissions.

**Solution**:
```yaml
1. Verify vCenter credential:
   - Test credential in Templates → Credentials
   - Ensure service account has proper permissions

2. Check required vCenter permissions:
   - Virtual Machine → All Privileges (for snapshot operations)
   - Datastore → Browse Datastore
   - Network → Assign Network
   - Global → Manage Custom Attributes

3. Verify network connectivity:
   - Test: telnet vcenter.company.com 443
   - Check firewall rules between AAP and vCenter
```

### Issue 3: SSH Connectivity Issues

**Problem**: Contact check fails for some hosts.

**Cause**: SSH connectivity, authentication, or privilege escalation issues.

**Solution**:
```yaml
1. Test SSH manually:
   ssh -i /path/to/key ansible@target-host
   
2. Check SSH credential configuration:
   - Verify username matches target system accounts
   - Ensure private key is correct format (RSA/ED25519)
   - Test privilege escalation: sudo -l

3. Update SSH configuration if needed:
   - Add SSH key to authorized_keys on targets
   - Configure sudo access for ansible user
   - Verify SSH daemon is running
```

### Issue 4: Workflow Variables Not Passing

**Problem**: Variables from one job template not available in next.

**Cause**: Incorrect set_stats usage or variable naming.

**Solution**:
```yaml
1. Check set_stats configuration in playbooks:
   - Ensure aggregate: true
   - Verify per_host: false (for workflow passing)
   
2. Check variable names are consistent:
   - discovery_success vs discovery_success_hosts
   - contact_success vs contact_success_hosts

3. Monitor variable passing:
   - Check job output for set_stats execution
   - Verify variables appear in next job's "Extra Variables"
```

### Issue 5: Performance Issues

**Problem**: Workflow takes too long or times out.

**Cause**: Resource constraints or inefficient parallelization.

**Solution**:
```yaml
1. Adjust fork settings:
   - Discovery/Contact: 10-20 forks
   - VMware operations: 5 forks (lower for API limits)
   - Patching: 10 forks

2. Optimize timeouts:
   - patch_async_timeout: 14400 (4 hours)
   - snapshot_async_timeout: 10800 (3 hours)
   - contact_timeout: 120 seconds

3. Consider host limits:
   - Use Limit field to batch process large inventories
   - Process in groups of 50-100 hosts
```

### Issue 6: Report Generation Failures

**Problem**: Report or publish steps fail.

**Cause**: Missing variables from previous steps or GitLab access issues.

**Solution**:
```yaml
1. Check report dependencies:
   - Verify all *_success and *_failure variables exist
   - Ensure ex_report_* variables are set by report step

2. Verify GitLab access:
   - Test GitLab token: curl -H "PRIVATE-TOKEN: token" gitlab-url/api/v4/user
   - Check repository permissions
   - Verify repository URL format

3. Check file system permissions:
   - Ensure /runner/aap_reports directory is writable
   - Verify report generation creates files successfully
```

### Getting Help

1. **Check AAP Logs:**
   ```bash
   # On AAP controller
   sudo journalctl -u automation-controller -f
   ```

2. **Review Job Output:**
   - Always check full job logs in AAP interface
   - Look for specific error messages
   - Check variable values in debug output

3. **Community Resources:**
   - Red Hat Customer Portal (with subscription)
   - Ansible Community Documentation
   - AAP GitHub Issues

4. **Professional Support:**
   - Red Hat Support (with subscription)
   - Ansible Professional Services
   - Red Hat Partner Network

## Next Steps

After successful implementation:

1. **Expand to additional environments** (staging, development)
2. **Implement advanced workflows** with parallel execution
3. **Add integration** with ITSM tools (ServiceNow, Jira)
4. **Enhance reporting** with custom dashboards
5. **Implement compliance scanning** and vulnerability assessment
6. **Add automated rollback** capabilities using snapshots

This guide provides the foundation for enterprise-grade VM patching automation using AAP 2.5. Remember to always test changes in non-production environments before deploying to production systems.