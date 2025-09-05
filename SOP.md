# SOP: Running the RHEL VM Patching Workflow in AAP 2.5

## Overview

This Standard Operating Procedure explains how to use the RHEL VM Patching Workflow in Red Hat Ansible Automation Platform (AAP) 2.5. The workflow automates end-to-end patching of RHEL virtual machines, including taking VM snapshots, applying OS updates, rebooting systems, performing health checks, and generating patch reports.

### Workflow Components

The workflow consists of a series of playbooks executed in sequence:

- **Discovery** – finds target VMs via vCenter and validates access
- **Contact** – checks SSH connectivity to the discovered hosts
- **Snapshot Creation** – takes VM snapshots before patching
- **Snapshot Verification** – ensures snapshots were created successfully
- **Patching** – applies package updates on the systems
- **Reboot** – restarts the systems and confirms they come back online
- **Health Check** – verifies post-patch system health (disk usage, services)
- **Report Generation** – compiles an HTML/CSV report of patch outcomes
- **Report Publishing** – uploads the report to a GitLab repository for audit/tracking

### Manual Approval Checkpoints

Two manual approval checkpoints are built into the workflow with no time limit:

1. **Post-Snapshot Verification / Pre-Patching** – requires operator confirmation before applying patches
2. **Post-Patching / Pre-Reboot** – requires operator confirmation before rebooting systems

These approval steps ensure oversight at critical junctures and will pause the workflow until approved by an authorized user.

## Prerequisites

### System Requirements

- **AAP 2.5 Controller**: Installed, accessible, and healthy with URL and login credentials
- **Workflow Template Setup**: The "Complete VM Patching Workflow" template configured in AAP with all necessary job templates, credentials, and inventory
- **User Permissions**: Appropriate permissions in AAP to run the workflow (admin or execute rights on the workflow template and relevant inventory)

### Configuration Requirements

**Job Templates**: Each playbook component should be configured with proper project, inventory (e.g., "Production RHEL Fleet"), and credentials (SSH key, vCenter account).

**Infrastructure Readiness**:
- Maintenance window or stakeholder approval in place
- Sufficient snapshot space available in VMware vCenter
- Valid credentials for all systems (vCenter, GitLab token, SSH keys)

## Procedure: Executing the Patching Workflow

### Step 1: Access the AAP Web Interface

1. Open a web browser and navigate to your AAP controller URL (e.g., `https://your-aap-controller.company.com`)
2. Log in with your AAP username and password
3. Verify successful login by confirming you reach the Dashboard or Overview page

*Screenshot: AAP 2.5 Login Screen and Dashboard upon login (placeholder)*

### Step 2: Navigate to Templates

1. In the left-hand navigation menu, click **"Automation Execution"**
2. Select **"Templates"** to display all Job Templates and Workflow Templates
3. Locate the workflow template named **"Complete VM Patching Workflow"**
4. Identify the template by its description (e.g., "End-to-end VM patching with snapshots, updates, and reporting")

*Screenshot: AAP Templates list, highlighting the "Complete VM Patching Workflow" template (placeholder)*

### Step 3: Launch the Patching Workflow

1. Click on the **Complete VM Patching Workflow** template to open its details
2. Click the **Launch** button (rocket icon or green Launch button)
3. Configure the following launch parameters:

**Inventory Selection**: 
- If prompted, choose the inventory containing target hosts (e.g., "Production RHEL Fleet")
- Use the **Limit** field to narrow down hosts if needed (leave blank for all hosts, or specify host/group patterns for subset patching)

**Extra Variables**: 
- Complete any survey prompts or extra variables if enabled
- Most configurations use pre-set global variables (vCenter credentials, GitLab details)

4. Click **"Launch"** to begin the workflow

*Screenshot: Workflow Template launch dialog showing inventory selection and host limiting options (placeholder)*

### Step 4: Monitor Workflow Execution

The Workflow Visualizer displays the workflow sequence as a flowchart with color-coded status indicators:
- **Yellow**: Currently running
- **Green**: Successful completion
- **Red**: Failed execution

**Monitoring Phases**:

**Discovery Phase**: 
- Identifies hosts/VMs and gathers initial data
- Should complete quickly if vCenter is reachable and credentials are correct

**Contact Phase (SSH Check)**: 
- Tests SSH connectivity to each target host
- Unreachable hosts are excluded from subsequent steps

**Snapshot Creation**: 
- Initiates VMware snapshots for each reachable VM
- Duration depends on VM count and storage performance

**Snapshot Verification**: 
- Confirms successful snapshot creation for all targets
- Ensures rollback capability before proceeding

Click individual nodes to view real-time log output and monitor for any errors. The workflow continues with remaining hosts even if some fail individual steps.

*Screenshot: Workflow Visualizer showing patching workflow in progress with Discovery, Contact, Snapshot nodes (placeholder)*

### Step 5: First Approval Checkpoint (Post-Snapshot / Pre-Patch)

After Snapshot Verification completes, the workflow pauses for manual approval.

**Pre-Approval Verification**:
- Review snapshot status in vCenter
- Check Snapshot Verification log output in AAP for any issues

**Approval Process**:
1. Navigate to **"Workflow Approvals"** in the AAP left sidebar
2. Locate the pending approval entry for the patching workflow
3. Click the pending approval item to view details:
   - Approval node name (e.g., "Snapshot Verification Approval")
   - Requestor information (the workflow)
   - Comments or description
4. Click **"Approve"** to proceed or **"Deny"** to halt the workflow
5. The Patching phase begins immediately after approval

*Screenshot: Workflow Approvals screen showing pending approval for patching workflow (placeholder)*
*Screenshot: Approval dialog with Approve/Deny options (placeholder)*

### Step 6: Patching Phase

After first approval, the System Patching step commences automatically.

**Patching Process**:
- Applies OS updates using appropriate package manager (`yum`/`dnf` on RHEL)
- Processes all target hosts that passed earlier checks
- Duration varies based on update count and host quantity
- Successfully patched hosts marked in success group
- Failed hosts flagged but workflow continues with others

**Monitoring**: Click the Patching node to watch live Ansible output and monitor package installation progress. No operator action required during this phase.

### Step 7: Second Approval Checkpoint (Post-Patch / Pre-Reboot)

The workflow pauses for the second manual approval after patching completes.

**Pre-Approval Review**:
- Verify patching results are acceptable
- Check for critical failures in the Patching node logs
- Consider external factors that might affect reboot timing

**Approval Process**:
1. Return to **"Workflow Approvals"** in the sidebar
2. Locate the new pending approval entry (e.g., "Reboot Approval")
3. Click the pending approval to open details
4. Click **"Approve"** to proceed with reboots or **"Deny"** to stop the workflow
5. The System Reboot phase begins after approval

Since approval nodes have no timeout, the workflow waits indefinitely until action is taken.

*Screenshot: Pending approval for reboot phase in Workflow Approvals list (placeholder)*

### Step 8: Post-Patch Operations

After second approval, the workflow continues automatically through remaining steps:

**System Reboot**:
- Reboots patched hosts in controlled manner
- Waits for systems to go down and return online
- Verifies SSH connectivity post-reboot

**Post-Patch Health Check**:
- Validates critical services and system health
- Checks disk space and other vital metrics
- Confirms patching didn't introduce issues
- Problem hosts marked as failed, healthy hosts marked as success

**Report Generation**:
- Creates comprehensive HTML report and CSV files
- Summarizes workflow outcomes, statistics, and timestamps
- Saves reports on AAP controller (typically in `/runner/aap_reports/`)
- Registers report file paths as workflow variables

**Report Publishing**:
- Publishes reports to configured GitLab repository
- Clones repo using GitLab token, adds report files to structured folders
- Organized by date/timestamp for easy tracking
- Pushes changes back to GitLab for audit trail

No operator intervention required during these phases. The workflow attempts report generation and publishing even if some hosts failed earlier stages.

### Step 9: Verify Completion and Collect Results

Once all steps finish, the workflow job marks as **Completed** in AAP.

**Status Verification**:
- Check Workflow Visualizer for final node statuses (green/red)
- Overall workflow shows green (successful) if critical steps succeeded for at least one host
- Red (failed) if failure path taken for all hosts

**Result Review**:
- **Check Final Status**: Verify most/all hosts were patched and rebooted successfully
- **Review Generated Report**: Download/open HTML report for executive summary and detailed breakdown
- **Access Published Report**: Navigate to GitLab repository to view uploaded report folder
- **Share Results**: Distribute GitLab link to stakeholders for patch result visibility

**Report Contents**:
- Executive summary of workflow results
- Host success/failure breakdown by phase
- Total counts and percentages
- Detailed CSV files for analysis and record-keeping

*Screenshot: Completed workflow run in AAP showing all successful steps (placeholder)*
*Screenshot: Generated HTML patch report summary (placeholder)*

### Step 10: Follow-Up Actions

Based on workflow outcomes, perform necessary follow-up activities:

**Failure Remediation**:
- Investigate logs for hosts that failed during patching or health checks
- Schedule remediation activities
- Consider re-running workflow on failed hosts using Limit field

**Snapshot Management**:
- Plan snapshot cleanup after confirming patch stability
- Coordinate with VM administration team for snapshot removal
- Follow organizational policies for snapshot retention

**Communication**:
- Share success/failure summary with relevant teams using report data
- Verify email notifications were received (if configured)
- Update change management systems as required

## Performance and Scaling Considerations

### Concurrency and Parallelism

**Forks Configuration**:
- **Discovery and Contact**: ~10 forks (default)
- **VMware Snapshot Operations**: ~5 forks (to avoid overloading vCenter)
- **Patching**: ~10 forks (default)
- **Maximum AAP Forks**: 200 (default limit)

**Scaling Recommendations**:
- **Large Inventories**: Increase forks to 10-20 for faster execution
- **Resource Constraints**: Lower forks or implement batching
- **Infrastructure Load**: Monitor system and network impact when increasing parallelism

### Batching Strategy

**Batch Processing**:
- Use **Limit** field to target host subsets (50-100 hosts recommended)
- Run workflow in smaller chunks rather than processing all hosts simultaneously
- Reduces infrastructure strain and makes process more manageable
- Ideal for tight maintenance windows or large environments

**Implementation**:
1. First batch: Launch workflow with `Limit: batch1_hosts`
2. Monitor completion and review results
3. Subsequent batches: Re-run workflow with `Limit: batch2_hosts`
4. Continue until all hosts are processed

### Timeout Configuration

**Default Timeouts**:
- **Job Templates**: Generous or unlimited timeouts
- **Patch Operations**: 4 hours (`patch_async_timeout`)
- **Snapshot Operations**: 3 hours (`snapshot_async_timeout`)
- **Manual Approvals**: No expiration (waits indefinitely)

**Tuning Recommendations**:
- Adjust timeout variables based on environment expectations
- Consider adding approval node timeouts (e.g., 24 hours) for automated failure
- Balance timeout length with operational requirements

## Conclusion

The Complete VM Patching Workflow in AAP 2.5 provides a controlled, automated patching process for RHEL servers with built-in validation and comprehensive reporting. This SOP enables operators to confidently execute the workflow from launch through completion, with appropriate oversight through manual approval checkpoints.

**Key Success Factors**:
- Ensure VM snapshots are available for rollback capability
- Maintain appropriate maintenance windows before execution
- Monitor workflow progress and address failures promptly
- Review generated reports for comprehensive outcome assessment
- Follow organizational change management procedures
