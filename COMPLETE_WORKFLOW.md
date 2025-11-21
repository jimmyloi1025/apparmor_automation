# Complete AppArmor Profile Management Workflow
## From Zero-Day Deployment to Automated Fine-Tuning

**Document Version:** 1.1  
**Last Updated:** 2025-11-20  
**Target Environment:** Raspberry Pi production fleet with DEV mirror machine

---

## Table of Contents

1. [Overview](#overview)
2. [Visual Workflow Diagrams](#visual-workflow-diagrams)
3. [Phase 0: Zero-Day Deployment](#phase-0-zero-day-deployment)
4. [Phase 1: Regular OS Update Workflow](#phase-1-regular-os-update-workflow)
5. [Phase 2: Ad-Hoc/Emergency Update Workflow](#phase-2-ad-hoc-emergency-update-workflow)
6. [Phase 3: Auto-Finetuning Process](#phase-3-auto-finetuning-process)
7. [Tools and Components Reference](#tools-and-components-reference)
8. [Rollback Procedures](#rollback-procedures)
9. [Troubleshooting](#troubleshooting)

---

## Overview

This document describes the complete, automatic lifecycle of AppArmor profile management for Apache services running on Raspberry Pi devices. The workflow leverages a DEV machine (identical to production) for testing and profile fine-tuning before any changes reach production.

### Architecture

```
┌─────────────────┐      ┌──────────────────┐      ┌─────────────────────┐
│  Control Panel  │──────│  DEV Machine     │──────│  Production Fleet   │
│  (Ansible)      │      │  (Test Site)     │      │  (Multiple RPis)    │
└─────────────────┘      └──────────────────┘      └─────────────────────┘
```

### Key Principles

1. **DEV-First Testing**: All OS updates and profile changes are tested on DEV before production
2. **Automated Detection**: Changes are detected by comparing system baselines before/after updates
3. **Audit Mode**: AppArmor profiles use `aa-audit` mode for comprehensive logging during fine-tuning
4. **Human Approval**: Final deployment to production requires human approval
5. **Rollback Ready**: All profile changes are backed up for quick rollback if needed

---

## Visual Workflow Diagrams

This section provides comprehensive flowcharts for all workflow phases. These diagrams illustrate the complete automation lifecycle, decision points, and interactions between the Control Panel, DEV machine, and Production fleet.

### High-Level Overview: Complete Lifecycle

```mermaid
graph TB
    Start([🚀 Project Start]) --> Phase0{Phase 0:<br/>Zero-Day<br/>Deployment?}
    Phase0 -->|Yes| Setup[Setup Infrastructure]
    Phase0 -->|Already Done| Monitor[Continuous Monitoring]
    
    Setup --> InitBaseline[Create Initial Baselines]
    InitBaseline --> DeployProfiles[Deploy Initial Profiles]
    DeployProfiles --> Monitor
    
    Monitor --> Trigger{Trigger Event?}
    
    Trigger -->|Scheduled Update| Phase1[Phase 1:<br/>Regular Update]
    Trigger -->|Emergency Patch| Phase2[Phase 2:<br/>Emergency Update]
    Trigger -->|App Change| Phase3[Phase 3:<br/>Auto-Finetuning]
    Trigger -->|No Event| Monitor
    
    Phase1 --> Prod1[Production Updated]
    Phase2 --> Prod2[Production Updated]
    Phase3 --> Prod3[Profile Refined]
    
    Prod1 --> Monitor
    Prod2 --> Monitor
    Prod3 --> Monitor
    
    style Start fill:#4CAF50,stroke:#2E7D32,color:#fff
    style Setup fill:#2196F3,stroke:#1565C0,color:#fff
    style Monitor fill:#FFC107,stroke:#F57C00,color:#000
    style Phase1 fill:#9C27B0,stroke:#6A1B9A,color:#fff
    style Phase2 fill:#F44336,stroke:#C62828,color:#fff
    style Phase3 fill:#00BCD4,stroke:#00838F,color:#fff
```

### Phase 0: Zero-Day Deployment Workflow

```mermaid
graph TB
    Start([📦 Phase 0 Start:<br/>Zero-Day Deployment]) --> CheckReq{Prerequisites<br/>Met?}
    
    CheckReq -->|No| SetupReq[Setup Ansible,<br/>SSH Keys,<br/>Network Access]
    SetupReq --> CheckReq
    CheckReq -->|Yes| CloneRepo[Clone/Setup Project<br/>on Control Panel]
    
    CloneRepo --> ConfigInventory[Configure hosts<br/>Inventory File]
    ConfigInventory --> TestConn[Test Ansible<br/>Connectivity]
    
    TestConn -->|Failed| FixConn[Fix SSH/Network<br/>Issues]
    FixConn --> TestConn
    
    TestConn -->|Success| DeployInfra[Deploy Monitoring<br/>Infrastructure<br/>apparmor-deploy.yml]
    
    DeployInfra --> VerifyDeploy{Verify Scripts<br/>& Directories<br/>Deployed?}
    VerifyDeploy -->|No| FixDeploy[Check Playbook<br/>Errors]
    FixDeploy --> DeployInfra
    
    VerifyDeploy -->|Yes| CreateProdBaseline[Create Initial Baseline<br/>on Production Fleet<br/>task3-create-baseline.yml]
    
    CreateProdBaseline --> DeployProfile[Deploy Initial<br/>AppArmor Profile<br/>to Production]
    
    DeployProfile --> ReloadProfile[Reload Profile &<br/>Set to Enforce Mode]
    
    ReloadProfile --> TestProd{Test Production<br/>Apache?}
    TestProd -->|Failed| RollbackProfile[Rollback Profile<br/>Fix Issues]
    RollbackProfile --> DeployProfile
    
    TestProd -->|Success| SyncDev[Sync Profile<br/>to DEV Machine]
    
    SyncDev --> CreateDevBaseline[Create Initial Baseline<br/>on DEV Machine]
    
    CreateDevBaseline --> Complete([✅ Phase 0 Complete:<br/>Ready for Operations])
    
    style Start fill:#4CAF50,stroke:#2E7D32,color:#fff
    style Complete fill:#4CAF50,stroke:#2E7D32,color:#fff
    style DeployInfra fill:#2196F3,stroke:#1565C0,color:#fff
    style DeployProfile fill:#2196F3,stroke:#1565C0,color:#fff
    style TestProd fill:#FF9800,stroke:#E65100,color:#fff
    style CheckReq fill:#FF9800,stroke:#E65100,color:#fff
```

### Phase 1: Regular OS Update Workflow (Main Workflow)

```mermaid
graph TB
    Start([📅 Phase 1 Start:<br/>Regular OS Update]) --> PreBaseline[Step 1.1: Create Pre-Update<br/>Baseline on DEV<br/>task3-create-baseline.yml]
    
    PreBaseline --> OSUpdate[Step 1.2: Perform OS Update<br/>on DEV Machine<br/>apt upgrade]
    
    OSUpdate --> RebootCheck{Reboot<br/>Required?}
    RebootCheck -->|Yes| Reboot[Reboot DEV<br/>Wait for Recovery]
    RebootCheck -->|No| FineTune
    Reboot --> FineTune
    
    FineTune[Step 1.3: Automated Fine-Tuning<br/>fine-tune-profile-dev.yml<br/>• Create post-update baseline<br/>• Detect changes<br/>• Generate suggestions<br/>• Apply to profile<br/>• Toggle aa-audit mode<br/>• Run tests<br/>• Toggle aa-enforce mode]
    
    FineTune --> DevTest{DEV Tests<br/>Passed?}
    
    DevTest -->|Failed| Analyze[Analyze Failures:<br/>• Review delta report<br/>• Check denied paths<br/>• Verify suggestions]
    
    Analyze --> RefineType{Refinement<br/>Approach?}
    RefineType -->|Auto| RerunFineTune[Re-run fine-tune-profile-dev.yml<br/>with adjusted parameters]
    RefineType -->|Manual| ManualFix[Manual Profile Edit<br/>on DEV]
    
    RerunFineTune --> DevTest
    ManualFix --> RunTestAgain[Run Tests Again<br/>test_apache_functions.sh]
    RunTestAgain --> DevTest
    
    DevTest -->|Passed| GenApproval[Step 1.4: Generate<br/>Approval Document<br/>task3-approve-profile-update.yml]
    
    GenApproval --> FetchApproval[Fetch Approval Doc<br/>to Control Panel]
    
    FetchApproval --> HumanReview{Step 1.5: Human Review<br/>& Approval}
    
    HumanReview -->|Rejected| InvestigateDelta[Investigate Unexpected<br/>Changes]
    InvestigateDelta --> RefineType
    
    HumanReview -->|Approved| DeployProd[Step 1.6: Deploy Profile<br/>to Production<br/>deploy-profile-to-production.yml<br/>• Backup current profiles<br/>• Copy from DEV<br/>• Reload profiles<br/>• Verify Apache]
    
    DeployProd --> VerifyProdProfile{Profile Deployed<br/>Successfully?}
    
    VerifyProdProfile -->|No| EmergencyRollback[Emergency Rollback<br/>Restore Backup]
    EmergencyRollback --> InvestigateProdIssue[Investigate DEV/Prod<br/>Differences]
    InvestigateProdIssue --> RefineType
    
    VerifyProdProfile -->|Yes| UpdateProd[Step 1.7: Perform OS Update<br/>on Production<br/>Rolling Update Strategy]
    
    UpdateProd --> UpdateFirst[Update First<br/>Production Machine]
    UpdateFirst --> TestFirst{First Machine<br/>OK?}
    
    TestFirst -->|No| RollbackFirst[Rollback First Machine]
    RollbackFirst --> InvestigateProdIssue
    
    TestFirst -->|Yes| UpdateRemaining[Update Remaining<br/>Production Machines<br/>One at a Time]
    
    UpdateRemaining --> MonitorProd[Step 1.8: Monitor Production<br/>24-72 Hours<br/>• Check denials<br/>• Review audit logs<br/>• Monitor performance]
    
    MonitorProd --> MonitorResult{Production<br/>Stable?}
    
    MonitorResult -->|Issues Found| MinorIssue{Critical<br/>Issues?}
    MinorIssue -->|Yes| EmergencyRollback
    MinorIssue -->|No| EnableAudit[Enable aa-audit mode<br/>Gather More Data]
    EnableAudit --> FixOnDev[Fix Profile on DEV<br/>Redeploy]
    FixOnDev --> DeployProd
    
    MonitorResult -->|Stable| FinalizeEnforce[Finalize: Ensure<br/>aa-enforce Mode]
    
    FinalizeEnforce --> CreateFinalBaseline[Create Post-Update<br/>Baseline on Production]
    
    CreateFinalBaseline --> Complete([✅ Phase 1 Complete:<br/>Production Updated & Stable])
    
    style Start fill:#9C27B0,stroke:#6A1B9A,color:#fff
    style Complete fill:#4CAF50,stroke:#2E7D32,color:#fff
    style FineTune fill:#2196F3,stroke:#1565C0,color:#fff
    style DevTest fill:#FF9800,stroke:#E65100,color:#fff
    style HumanReview fill:#FF9800,stroke:#E65100,color:#fff
    style DeployProd fill:#2196F3,stroke:#1565C0,color:#fff
    style MonitorProd fill:#FFC107,stroke:#F57C00,color:#000
    style EmergencyRollback fill:#F44336,stroke:#C62828,color:#fff
```

### Phase 2: Ad-Hoc/Emergency Update Workflow

```mermaid
graph TB
    Start([🚨 Phase 2 Start:<br/>Emergency Update]) --> Assess[Step 2.1: Emergency Assessment<br/>• Identify affected packages<br/>• Check Apache dependency<br/>• Assess urgency vs risk]
    
    Assess --> DevFirst{Can Test<br/>on DEV First?}
    
    DevFirst -->|No - Critical| DirectProd[⚠️ Direct to Production<br/>High Risk Path]
    DirectProd --> BackupAll[Backup All Profiles<br/>Take Snapshots]
    BackupAll --> UpdateProdEmerg[Apply Emergency Update<br/>First Machine Only]
    UpdateProdEmerg --> TestEmergProd
    
    DevFirst -->|Yes - Preferred| EmergBaseline[Step 2.2: Create Emergency<br/>Baseline on DEV]
    
    EmergBaseline --> FastUpdate[Apply Emergency Update<br/>on DEV<br/>apt install specific package]
    
    FastUpdate --> FastFineTune[Fast Fine-Tuning<br/>fine-tune-profile-dev.yml<br/>audit_all_paths=false<br/>Faster Detection]
    
    FastFineTune --> QuickReview[Quick Review Delta Report<br/>cat delta JSON]
    
    QuickReview --> DevTestEmerg{DEV Tests<br/>Passed?}
    
    DevTestEmerg -->|Failed| AssessFailure{Changes<br/>Significant?}
    AssessFailure -->|Yes| EscalateEmerg[Escalate Decision<br/>Consider Workaround]
    EscalateEmerg --> End([⛔ Emergency Aborted:<br/>Needs Further Analysis])
    
    AssessFailure -->|No - Minor| QuickFix[Quick Manual Fix]
    QuickFix --> DevTestEmerg
    
    DevTestEmerg -->|Passed| DeployEmergProd[Step 2.3: Deploy to Production<br/>deploy-profile-to-production.yml]
    
    DeployEmergProd --> UpdateProdEmerg
    
    UpdateProdEmerg --> TestEmergProd{First Machine<br/>Stable?}
    
    TestEmergProd -->|No| RollbackEmerg[Immediate Rollback<br/>Profile + Package]
    RollbackEmerg --> EscalateEmerg
    
    TestEmergProd -->|Yes| ContinueRollout{Continue to<br/>Other Machines?}
    
    ContinueRollout -->|Yes| UpdateNextMachine[Update Next Machine<br/>serial=1 Controlled Rollout]
    UpdateNextMachine --> AllDone{All Machines<br/>Updated?}
    
    AllDone -->|No| UpdateNextMachine
    AllDone -->|Yes| IntensiveMonitor
    
    ContinueRollout -->|Monitor First| IntensiveMonitor[Step 2.4: Intensive Monitoring<br/>Check every 15 min<br/>for 1-2 hours]
    
    IntensiveMonitor --> StableCheck{Production<br/>Stable?}
    
    StableCheck -->|Issues| RollbackEmerg
    StableCheck -->|Stable 1-2h| ReduceMonitor[Reduce to Normal<br/>Monitoring Schedule]
    
    ReduceMonitor --> FinalCheck[Continue Monitoring<br/>24 Hours]
    
    FinalCheck --> CompleteEmerg([✅ Phase 2 Complete:<br/>Emergency Update Deployed])
    
    style Start fill:#F44336,stroke:#C62828,color:#fff
    style CompleteEmerg fill:#4CAF50,stroke:#2E7D32,color:#fff
    style End fill:#9E9E9E,stroke:#616161,color:#fff
    style DirectProd fill:#FF5722,stroke:#D84315,color:#fff
    style EscalateEmerg fill:#FF5722,stroke:#D84315,color:#fff
    style DevTestEmerg fill:#FF9800,stroke:#E65100,color:#fff
    style TestEmergProd fill:#FF9800,stroke:#E65100,color:#fff
    style IntensiveMonitor fill:#FFC107,stroke:#F57C00,color:#000
```

### Phase 3: Auto-Finetuning Process

```mermaid
graph TB
    Start([🔄 Phase 3 Start:<br/>Auto-Finetuning]) --> SetupMonitor[Step 3.1: Setup Continuous<br/>Monitoring<br/>• Create daily_monitor.sh<br/>• Add to crontab<br/>• Configure alerts]
    
    SetupMonitor --> DailyRun[Daily Cron Job Runs<br/>Collect Denial Logs]
    
    DailyRun --> CheckDenials{Denials<br/>Detected?}
    
    CheckDenials -->|No| WaitNext[Wait for Next Day]
    WaitNext --> DailyRun
    
    CheckDenials -->|Yes| SendAlert[Send Email Alert<br/>to Admin]
    
    SendAlert --> AnalyzeDenials[Step 3.2: Analyze Denials<br/>• Parse logs<br/>• Group by path/type<br/>• Identify patterns]
    
    AnalyzeDenials --> Categorize{Step 3.2: Categorize<br/>Denials}
    
    Categorize -->|Expected - New Feature| ValidChange[Valid Application<br/>Change]
    Categorize -->|Unexpected| InvestigateSecurity[Investigate Potential<br/>Security Issue]
    Categorize -->|Noise/False Positive| TuneApp[Tune Application<br/>or Adjust Profile Logic]
    
    InvestigateSecurity --> SecThreat{Security<br/>Threat?}
    SecThreat -->|Yes| IncidentResponse[Trigger Incident<br/>Response]
    SecThreat -->|No| ValidChange
    
    ValidChange --> ReplicateDev[Step 3.3: Replicate on DEV<br/>• Deploy same app change<br/>• Or create test scenario]
    
    ReplicateDev --> CreateAutoBaseline[Create Pre-Change<br/>Baseline on DEV<br/>task3-create-baseline.yml]
    
    CreateAutoBaseline --> ApplyChange[Apply Application Change<br/>on DEV]
    
    ApplyChange --> AutoFineTune[Run Fine-Tuning<br/>fine-tune-profile-dev.yml<br/>audit_all_paths=true]
    
    AutoFineTune --> IterativeTest[Step 3.4: Iterative Testing<br/>• Enable aa-audit<br/>• Run comprehensive tests<br/>• Check for denials]
    
    IterativeTest --> TestResult{Tests Clean?<br/>No Denials?}
    
    TestResult -->|Denials Found| AnalyzeNewDenials[Analyze New Denials]
    AnalyzeNewDenials --> AddToProfile{Add to<br/>Profile?}
    
    AddToProfile -->|Yes| UpdateProfileDev[Update Profile on DEV<br/>Apply new rules]
    UpdateProfileDev --> IterativeTest
    
    AddToProfile -->|No - App Issue| FixApp[Fix Application<br/>Behavior]
    FixApp --> IterativeTest
    
    TestResult -->|Clean| DeployRefined[Step 3.5: Deploy Refined<br/>Profile to Production<br/>deploy-profile-to-production.yml]
    
    DeployRefined --> EnableProdAudit[Enable aa-audit Mode<br/>on Production<br/>Optional Monitoring Period]
    
    EnableProdAudit --> MonitorAuto[Monitor Production<br/>24-48 Hours]
    
    MonitorAuto --> ProdClean{Production<br/>Clean?}
    
    ProdClean -->|Issues| MinorProdIssue{Minor or<br/>Critical?}
    MinorProdIssue -->|Critical| RollbackAuto[Rollback Profile]
    RollbackAuto --> InvestigateSecurity
    
    MinorProdIssue -->|Minor| TweakProfile[Tweak Profile on DEV<br/>Redeploy]
    TweakProfile --> DeployRefined
    
    ProdClean -->|Clean| FinalizeAuto[Finalize: Set to<br/>aa-enforce Mode]
    
    FinalizeAuto --> DocumentChange[Document Change<br/>Update Baseline]
    
    DocumentChange --> CompleteAuto([✅ Phase 3 Complete:<br/>Profile Refined])
    
    CompleteAuto --> DailyRun
    
    IncidentResponse --> EndIncident([🛑 Incident Response<br/>Follow Security Procedures])
    TuneApp --> DailyRun
    
    style Start fill:#00BCD4,stroke:#00838F,color:#fff
    style CompleteAuto fill:#4CAF50,stroke:#2E7D32,color:#fff
    style DailyRun fill:#FFC107,stroke:#F57C00,color:#000
    style CheckDenials fill:#FF9800,stroke:#E65100,color:#fff
    style Categorize fill:#FF9800,stroke:#E65100,color:#fff
    style SecThreat fill:#FF9800,stroke:#E65100,color:#fff
    style IncidentResponse fill:#F44336,stroke:#C62828,color:#fff
    style EndIncident fill:#F44336,stroke:#C62828,color:#fff
    style IterativeTest fill:#2196F3,stroke:#1565C0,color:#fff
```

### Actor Interaction Diagram

This diagram shows the interaction between different actors (Control Panel, DEV Machine, Production Fleet, and Human Operator) during a typical Phase 1 workflow:

```mermaid
sequenceDiagram
    actor Human as 👤 Human Operator
    participant CP as 💻 Control Panel<br/>(Ansible)
    participant DEV as 🔧 DEV Machine<br/>(Raspberry Pi)
    participant PROD as 🏭 Production Fleet<br/>(Multiple RPis)
    
    Note over Human,PROD: Phase 1: Regular OS Update Workflow
    
    Human->>CP: Initiate OS Update Process
    CP->>DEV: Create pre-update baseline
    DEV-->>CP: Baseline JSON returned
    
    Human->>DEV: Perform OS update (apt upgrade)
    DEV->>DEV: Reboot if needed
    
    CP->>DEV: Run fine-tune-profile-dev.yml
    DEV->>DEV: Create post-update baseline
    DEV->>DEV: Detect changes (compare baselines)
    DEV->>DEV: Generate profile suggestions
    DEV->>DEV: Apply suggestions to profile
    DEV->>DEV: Toggle aa-audit mode ON
    DEV->>DEV: Run Apache functional tests
    DEV->>DEV: Toggle aa-enforce mode ON
    DEV-->>CP: Delta report + Test results
    
    alt Tests Failed
        CP->>Human: Alert: DEV tests failed
        Human->>Human: Analyze failures
        Human->>CP: Refine and retry
        CP->>DEV: Re-run fine-tuning
    end
    
    CP->>DEV: Generate approval document
    DEV-->>CP: Approval document
    
    CP->>Human: Present approval doc for review
    Human->>Human: Review changes & test results
    
    alt Approved
        Human->>CP: Approve deployment
        CP->>DEV: Fetch tested profile
        DEV-->>CP: Profile file
        CP->>PROD: Deploy profile to all machines
        PROD->>PROD: Backup current profiles
        PROD->>PROD: Apply new profile
        PROD->>PROD: Reload AppArmor
        PROD-->>CP: Deployment successful
        
        CP->>PROD: Perform OS update (rolling)
        loop For each machine
            PROD->>PROD: Update packages
            PROD->>PROD: Reboot if needed
            PROD-->>CP: Update status
        end
        
        CP->>PROD: Monitor for 24-72 hours
        PROD-->>CP: Logs & metrics
        
        alt Stable
            CP->>PROD: Finalize (aa-enforce mode)
            CP->>Human: Report: Update successful
        else Issues detected
            CP->>Human: Alert: Issues detected
            Human->>CP: Initiate rollback
            CP->>PROD: Rollback profile & packages
        end
    else Rejected
        Human->>CP: Reject deployment
        Human->>Human: Investigate unexpected changes
        Human->>CP: Refine profile on DEV
    end
    
    Note over Human,PROD: Workflow Complete - Return to Monitoring
```

---

---

## Phase 0: Zero-Day Deployment

### Objective
Set up the infrastructure and deploy initial AppArmor profiles to the production fleet from scratch.

### Step 0.1: Initial Setup on Control Panel

**Location:** Control Panel

```bash
# 1. Clone or set up the project
cd /home/jimmyloi
git clone <repository-url> ansible_apparmor
cd ansible_apparmor

# 2. Configure inventory file
cat > hosts << 'INVENTORY'
[control]
localhost ansible_connection=local

[dev_machines]
dev-pi.local ansible_host=192.168.1.10 ansible_user=pi

[production]
prod-pi-01.local ansible_host=192.168.1.21 ansible_user=pi
prod-pi-02.local ansible_host=192.168.1.22 ansible_user=pi
prod-pi-03.local ansible_host=192.168.1.23 ansible_user=pi

[raspberry_pis:children]
dev_machines
production
INVENTORY

# 3. Test connectivity
ansible -i hosts all -m ping

# 4. Verify AppArmor is installed on all targets
ansible -i hosts raspberry_pis -b -m command -a "aa-status"
```

**Tools Used:**
- `ansible` (inventory management, connectivity testing)
- `git` (version control)

### Step 0.2: Deploy Baseline Monitoring Infrastructure

**Location:** Control Panel → All Raspberry Pis

```bash
# Deploy monitoring scripts and directory structure to all RPis
ansible-playbook -i hosts apparmor-deploy.yml

# What this does:
# - Creates /root/apache-monitor/ directory structure
# - Deploys baseline creation scripts
# - Deploys change detection scripts
# - Sets up log rotation
# - Creates backup directories for profiles
```

**Tools Used:**
- **Playbook:** `apparmor-deploy.yml`
- **Deployed Scripts:**
  - `scripts/create_baseline.sh` → `/root/apache-monitor/scripts/`
  - `scripts/detect_apache_changes.sh` → `/root/apache-monitor/scripts/`
  - `scripts/test_apache_functions.sh` → `/root/apache-monitor/scripts/`

**Verification:**

```bash
# Verify deployment
ansible -i hosts raspberry_pis -b -m command -a "ls -la /root/apache-monitor"

# Expected output: scripts/, data/baselines/, data/deltas/
```

### Step 0.3: Create Initial Baseline (Production State)

**Location:** Control Panel → Production Fleet

```bash
# Create initial baseline on ALL production machines (not DEV yet)
ansible-playbook -i hosts ansible/playbooks/task3-create-baseline.yml \
  -e "target_group=production" \
  -e "baseline_name=initial-production"

# What this captures:
# - Current Apache processes (PIDs, command lines, user/group)
# - Current file access patterns from AppArmor logs
# - Apache version
# - Timestamp
```

**Tools Used:**
- **Playbook:** `ansible/playbooks/task3-create-baseline.yml`
- **Backend Script:** `scripts/create_baseline.sh`

**Output:** JSON baseline files stored at:
- `/root/apache-monitor/data/baselines/initial-production-<timestamp>.json` (on each Pi)

**Verification:**

```bash
# Fetch baselines to control panel for review
ansible -i hosts production -b -m fetch \
  -a "src=/root/apache-monitor/data/baselines/initial-production-*.json dest=data/baselines/ flat=yes"

# Review baseline
cat data/baselines/initial-production-*.json | jq .
```

### Step 0.4: Deploy Initial AppArmor Profiles

**Location:** Control Panel → Production Fleet

```bash
# Deploy the initial profile (usually vendor-default or manually crafted)
ansible -i hosts production -b -m copy \
  -a "src=profiles/usr.sbin.apache2 dest=/etc/apparmor.d/usr.sbin.apache2 backup=yes"

# Reload profiles
ansible -i hosts production -b -m command \
  -a "apparmor_parser -r /etc/apparmor.d/usr.sbin.apache2"

# Set to enforce mode
ansible -i hosts production -b -m command \
  -a "aa-enforce /usr/sbin/apache2"

# Verify profile is loaded
ansible -i hosts production -b -m command \
  -a "aa-status | grep apache2"
```

**Tools Used:**
- `ansible` (ad-hoc commands)
- `apparmor_parser` (profile loading)
- `aa-enforce` (mode setting)

**Verification:**

```bash
# Check Apache is running normally
ansible -i hosts production -m uri \
  -a "url=http://{{ inventory_hostname }}/ return_content=no"

# Check AppArmor logs for denials (should be minimal/none)
ansible -i hosts production -b -m command \
  -a "grep DENIED /var/log/syslog | tail -20"
```

### Step 0.5: Configure DEV Machine

**Location:** Control Panel → DEV Machine

```bash
# Sync production state to DEV (copy profile, create baseline)
ansible -i hosts dev_machines -b -m copy \
  -a "src=profiles/usr.sbin.apache2 dest=/etc/apparmor.d/usr.sbin.apache2"

ansible -i hosts dev_machines -b -m command \
  -a "apparmor_parser -r /etc/apparmor.d/usr.sbin.apache2"

ansible -i hosts dev_machines -b -m command \
  -a "aa-enforce /usr/sbin/apache2"

# Create initial baseline on DEV
ansible-playbook -i hosts ansible/playbooks/task3-create-baseline.yml \
  -e "target_group=dev_machines" \
  -e "baseline_name=initial-dev"
```

**Tools Used:**
- Same as production setup
- **Purpose:** Ensure DEV mirrors production state exactly

---

## Phase 1: Regular OS Update Workflow

### Objective
Handle scheduled OS updates (e.g., monthly security patches) with automated profile fine-tuning.

### Workflow Overview

```
┌────────────────────────────────────────────────────────────────────┐
│                    REGULAR UPDATE WORKFLOW                         │
└────────────────────────────────────────────────────────────────────┘

1. [DEV] Create Pre-Update Baseline
2. [DEV] Perform OS Update
3. [DEV] Auto-Detect Changes & Generate Profile
4. [DEV] Apply & Test Profile (with aa-audit mode)
5. [HUMAN] Review & Approve
6. [PROD] Deploy Profile to Production
7. [PROD] Perform OS Update on Production
8. [PROD] Monitor & Finalize
```

### Step 1.1: Pre-Update Baseline (DEV Only)

**Location:** Control Panel → DEV Machine

```bash
# Create baseline before OS update
ansible-playbook -i hosts ansible/playbooks/task3-create-baseline.yml \
  -e "target_group=dev_machines" \
  -e "baseline_name=pre-update-$(date +%Y%m%d)"
```

**Tools Used:**
- **Playbook:** `ansible/playbooks/task3-create-baseline.yml`
- **Script:** `scripts/create_baseline.sh`

**Output:** Baseline stored at `/root/apache-monitor/data/baselines/pre-update-YYYYMMDD-<timestamp>.json`

**Important:** This baseline captures the "known good" state before any changes.

### Step 1.2: Perform OS Update on DEV

**Location:** DEV Machine (manual or automated)

```bash
# SSH to DEV machine
ssh pi@dev-pi.local

# Perform OS update (Raspberry Pi OS)
sudo apt update
sudo apt upgrade -y
sudo apt autoremove -y

# Reboot if kernel updated
sudo reboot

# Wait for machine to come back online
# Verify Apache is running
curl http://localhost/
```

**Tools Used:**
- `apt` (package management)
- Manual verification

**Note:** Production machines do NOT update yet. They wait for profile validation.

### Step 1.3: Automated Fine-Tuning on DEV

**Location:** Control Panel → DEV Machine

This is the core automation step that combines multiple operations:

```bash
# Run the complete DEV fine-tuning workflow
ansible-playbook -i hosts ansible/playbooks/fine-tune-profile-dev.yml \
  -e "baseline_before=/root/apache-monitor/data/baselines/pre-update-20251120-*.json" \
  -e "create_post_update_baseline=yes" \
  -e "apply_suggestions=yes" \
  -e "run_tests=yes" \
  -e "audit_all_paths=true"

# What this does:
# 1. Creates post-update baseline (current state after OS update)
# 2. Detects changes (compares pre vs post baselines)
# 3. Generates profile suggestions (new paths, denied paths)
# 4. Applies suggestions to the profile
# 5. Toggles profile to aa-audit mode for comprehensive logging
# 6. Runs Apache functional tests (HTTP requests, file access, etc.)
# 7. Toggles profile back to aa-enforce mode
# 8. Generates summary report
```

**Tools Used:**
- **Playbook:** `ansible/playbooks/fine-tune-profile-dev.yml`
- **Scripts:**
  - `scripts/create_baseline.sh` (post-update baseline)
  - `scripts/detect_apache_changes.sh` (change detection)
  - `scripts/test_apache_functions.sh` (functional testing)
- **AppArmor Commands:**
  - `aa-audit /usr/sbin/apache2` (enable audit mode)
  - `aa-enforce /usr/sbin/apache2` (return to enforce mode)
- **Ansible Role:** `apparmor-profile` (backup, apply rules)

**Output Files:**
- `/root/apache-monitor/data/baselines/post-update-<timestamp>.json` (new baseline)
- `/root/apache-monitor/data/deltas/apache-delta-<timestamp>.json` (changes detected)
- `/root/apache-monitor/data/deltas/apparmor-profile-suggestions-<timestamp>.txt` (generated rules)
- Updated profile: `/etc/apparmor.d/usr.sbin.apache2` (on DEV)
- Test results in Ansible facts

**Verification:**

```bash
# Fetch results to control panel
ansible -i hosts dev_machines -b -m fetch \
  -a "src=/root/apache-monitor/data/deltas/apache-delta-* dest=data/deltas/ flat=yes"

# Review delta report
cat data/deltas/apache-delta-*.json | jq .

# Review profile suggestions
cat data/deltas/apparmor-profile-suggestions-*.txt

# Check test results from Ansible output
# Look for: "DEV testing: PASSED" or "DEV testing: FAILED"
```

**Iteration:** If tests fail, refine the profile:

```bash
# Option 1: Manual refinement
# Edit the profile on DEV, then re-run tests
ansible -i hosts dev_machines -b -m command \
  -a "/root/apache-monitor/scripts/test_apache_functions.sh"

# Option 2: Re-run fine-tuning without creating new baseline
ansible-playbook -i hosts ansible/playbooks/refine-profile-dev.yml \
  -e "apply_suggestions=yes" \
  -e "run_tests=yes"
```

### Step 1.4: Generate Approval Document

**Location:** Control Panel → DEV Machine → Control Panel

```bash
# Generate approval document with DEV test results
ansible-playbook -i hosts ansible/playbooks/task3-approve-profile-update.yml \
  -e "delta_file=/root/apache-monitor/data/deltas/apache-delta-<timestamp>.json" \
  -e "suggestions_file=/root/apache-monitor/data/deltas/apparmor-profile-suggestions-<timestamp>.txt" \
  -e "dev_test_passed=true" \
  -e "dev_test_output='All HTTP checks passed. Apache functioning normally.'"

# Fetch approval document
ansible -i hosts dev_machines -b -m fetch \
  -a "src=/root/apache-monitor/approvals/apache-approval-*.md dest=approvals/ flat=yes"
```

**Tools Used:**
- **Playbook:** `ansible/playbooks/task3-approve-profile-update.yml`
- **Script:** `scripts/create_approval_document.sh`

**Output:** Approval document at `approvals/apache-approval-<timestamp>.md`

**Approval Document Contains:**
- Delta summary (new paths, removed paths, denied paths)
- Profile suggestions
- DEV test results
- Version changes
- Human approval section (to be signed off)

### Step 1.5: Human Review and Approval

**Location:** Control Panel (human review)

```bash
# Review the approval document
cat approvals/apache-approval-<latest>.md

# Human decision points:
# - Are the detected changes expected? (e.g., new modules, log paths)
# - Did DEV tests pass?
# - Are there any security concerns with new paths?
# - Is the timing right for production deployment?

# If approved: proceed to Step 1.6
# If rejected: investigate unexpected changes, refine profile, repeat Step 1.3
```

**Tools Used:**
- Text editor / markdown viewer
- Human judgment

### Step 1.6: Deploy Profile to Production

**Location:** Control Panel → Production Fleet

```bash
# Deploy the tested profile from DEV to all production machines
ansible-playbook -i hosts ansible/playbooks/deploy-profile-to-production.yml \
  -e "source_host=dev-pi.local" \
  -e "profile_file=/etc/apparmor.d/usr.sbin.apache2"

# What this does:
# 1. Backs up current production profiles
# 2. Copies tested profile from DEV to production
# 3. Reloads profiles on production
# 4. Optionally enables aa-audit mode for monitoring period
# 5. Verifies Apache is still running
```

**Tools Used:**
- **Playbook:** `ansible/playbooks/deploy-profile-to-production.yml`
- **Ansible Modules:** `fetch`, `copy`, `command`
- **AppArmor Commands:** `apparmor_parser`, `aa-audit` (optional)

**Verification:**

```bash
# Verify profile is loaded on all production machines
ansible -i hosts production -b -m command \
  -a "aa-status | grep apache2"

# Verify Apache is running
ansible -i hosts production -m uri \
  -a "url=http://{{ inventory_hostname }}/ return_content=no"

# Check for immediate denials (should be none if DEV testing was thorough)
ansible -i hosts production -b -m command \
  -a "grep DENIED /var/log/syslog | grep apache2 | tail -5"
```

### Step 1.7: Perform OS Update on Production

**Location:** Production Fleet

Now that the profile is deployed and verified, production can be updated.

```bash
# Update production machines (rolling update recommended)
# Update one machine at a time or in batches

# Example: Update prod-pi-01
ansible -i hosts -l prod-pi-01.local production -b -m apt \
  -a "upgrade=dist update_cache=yes"

# Reboot if needed
ansible -i hosts -l prod-pi-01.local production -b -m reboot

# Wait for reboot and verify
ansible -i hosts -l prod-pi-01.local production -m ping

# Verify Apache and profile
ansible -i hosts -l prod-pi-01.local production -m uri \
  -a "url=http://{{ inventory_hostname }}/"

ansible -i hosts -l prod-pi-01.local production -b -m command \
  -a "aa-status | grep apache2"

# Repeat for remaining production machines
# Or use a rolling update strategy with Ansible serial parameter
```

**Tools Used:**
- `ansible` (ad-hoc commands or playbook with `serial` parameter)
- `apt` (package updates)

**Important:** Stagger updates. Don't update all production at once in case issues arise.

### Step 1.8: Monitor and Finalize

**Location:** Control Panel → Production Fleet

```bash
# Monitor AppArmor logs for denials over the next 24-72 hours
ansible -i hosts production -b -m command \
  -a "grep DENIED /var/log/syslog | grep apache2 | tail -20"

# If profile was in aa-audit mode, review logs for unexpected access
ansible -i hosts production -b -m command \
  -a "grep 'apparmor=\"AUDIT\"' /var/log/syslog | grep apache2 | tail -50"

# If no issues after monitoring period, finalize by ensuring enforce mode
ansible -i hosts production -b -m command \
  -a "aa-enforce /usr/sbin/apache2"

# Create post-production baseline for future comparison
ansible-playbook -i hosts ansible/playbooks/task3-create-baseline.yml \
  -e "target_group=production" \
  -e "baseline_name=post-update-$(date +%Y%m%d)"
```

**Tools Used:**
- `grep` (log analysis)
- `aa-enforce` (mode setting)
- Baseline creation playbook

---

## Phase 2: Ad-Hoc/Emergency Update Workflow

### Objective
Handle urgent security patches or critical updates that can't wait for the regular update cycle.

### Key Differences from Regular Workflow

1. **Speed Priority**: Automated testing is faster, human approval window is shorter
2. **Selective Updates**: May only affect specific packages, not full OS update
3. **Risk Assessment**: Higher urgency = more tolerance for minor issues
4. **Rollback Readiness**: Ensure rollback plan is ready before deployment

### Step 2.1: Emergency Assessment

**Location:** Control Panel

```bash
# 1. Assess the emergency update scope
# Questions:
# - What packages are affected?
# - Does it affect Apache or its dependencies?
# - Can we test on DEV first or must we go direct to production?

# 2. Check current production state
ansible -i hosts production -b -m command \
  -a "dpkg -l | grep <affected-package>"

ansible -i hosts production -b -m command \
  -a "systemctl status apache2"
```

### Step 2.2: Fast-Track DEV Testing

**Location:** Control Panel → DEV Machine

```bash
# Create emergency baseline
ansible-playbook -i hosts ansible/playbooks/task3-create-baseline.yml \
  -e "target_group=dev_machines" \
  -e "baseline_name=emergency-$(date +%Y%m%d-%H%M%S)"

# Apply emergency update on DEV
ansible -i hosts dev_machines -b -m apt \
  -a "name=<package-name> state=latest update_cache=yes"

# Run fast fine-tuning (skip some optional steps)
ansible-playbook -i hosts ansible/playbooks/fine-tune-profile-dev.yml \
  -e "baseline_before=/root/apache-monitor/data/baselines/emergency-*.json" \
  -e "create_post_update_baseline=yes" \
  -e "apply_suggestions=yes" \
  -e "run_tests=yes" \
  -e "audit_all_paths=false"  # Faster: only audit system paths

# Quick review of results
ansible -i hosts dev_machines -b -m fetch \
  -a "src=/root/apache-monitor/data/deltas/apache-delta-* dest=data/deltas/ flat=yes"

cat data/deltas/apache-delta-*.json | jq .
```

**Decision Point:**

- **If DEV tests pass and changes are minimal:** Proceed to emergency deployment
- **If DEV tests fail or changes are significant:** Escalate, consider workarounds

### Step 2.3: Emergency Deployment to Production

**Location:** Control Panel → Production Fleet

```bash
# Deploy profile to production
ansible-playbook -i hosts ansible/playbooks/deploy-profile-to-production.yml \
  -e "source_host=dev-pi.local" \
  -e "profile_file=/etc/apparmor.d/usr.sbin.apache2"

# Apply emergency update to production (one machine at a time)
ansible -i hosts -l prod-pi-01.local production -b -m apt \
  -a "name=<package-name> state=latest update_cache=yes"

# Immediate verification
ansible -i hosts -l prod-pi-01.local production -m uri \
  -a "url=http://{{ inventory_hostname }}/"

ansible -i hosts -l prod-pi-01.local production -b -m command \
  -a "systemctl status apache2"

# If successful on first machine, continue to others
# Use serial=1 for controlled rollout
```

**Rollback Command (if needed):**

```bash
# Restore previous profile from backup
ansible -i hosts production -b -m copy \
  -a "src=/etc/apparmor.d/backups/usr.sbin.apache2.<timestamp> dest=/etc/apparmor.d/usr.sbin.apache2 remote_src=yes"

ansible -i hosts production -b -m command \
  -a "apparmor_parser -r /etc/apparmor.d/usr.sbin.apache2"

# Downgrade package if needed
ansible -i hosts production -b -m apt \
  -a "name=<package-name>=<previous-version> allow_downgrade=yes"
```

### Step 2.4: Intensive Monitoring

**Location:** Control Panel → Production Fleet

```bash
# Monitor logs every 15 minutes for the first hour
watch -n 900 'ansible -i hosts production -b -m command -a "grep -E \"DENIED|ERROR\" /var/log/syslog | grep apache2 | tail -10"'

# Check Apache access logs for errors
ansible -i hosts production -b -m command \
  -a "tail -20 /var/log/apache2/error.log"

# If stable after 1-2 hours, reduce monitoring to normal schedule
```

---

## Phase 3: Auto-Finetuning Process

### Objective
Continuous refinement of AppArmor profiles based on observed behavior, even outside of update cycles.

### When to Use Auto-Finetuning

1. **New Application Features**: When developers deploy new web applications
2. **Configuration Changes**: When Apache configuration changes (new modules, vhosts)
3. **Behavioral Drift**: When legitimate application behavior triggers denials
4. **Periodic Review**: Quarterly reviews of AppArmor logs for optimization

### Step 3.1: Continuous Monitoring

**Location:** Control Panel → Production Fleet

```bash
# Set up periodic log collection (e.g., daily cron job)
# Create a script on control panel: /home/jimmyloi/ansible_apparmor/scripts/daily_monitor.sh

#!/bin/bash
DATE=$(date +%Y%m%d)
ansible -i /home/jimmyloi/ansible_apparmor/hosts production -b \
  -m command -a "grep DENIED /var/log/syslog | grep apache2" \
  > /home/jimmyloi/ansible_apparmor/monitoring/denials-$DATE.log

# Alert if denials are found
if [ -s /home/jimmyloi/ansible_apparmor/monitoring/denials-$DATE.log ]; then
  echo "AppArmor denials detected on $DATE" | mail -s "AppArmor Alert" admin@example.com
fi

# Add to crontab
crontab -e
# Add: 0 2 * * * /home/jimmyloi/ansible_apparmor/scripts/daily_monitor.sh
```

**Tools Used:**
- `cron` (scheduling)
- `grep` (log parsing)
- `mail` (alerting)

### Step 3.2: Analyze Denials

**Location:** Control Panel

```bash
# Review collected denials
cat monitoring/denials-*.log | grep DENIED | sort | uniq -c | sort -rn

# Common patterns to look for:
# - New file paths being accessed
# - Permission denials (read vs write)
# - Network operations
# - Capability requirements

# Categorize denials:
# - Expected (new features) → add to profile
# - Unexpected (potential attack) → investigate further
# - Noise (false positives) → refine profile or adjust application
```

**Tools Used:**
- Text processing (`grep`, `sort`, `uniq`, `awk`)
- Log analysis

### Step 3.3: Replicate on DEV

**Location:** Control Panel → DEV Machine

```bash
# Replicate the production scenario on DEV
# Example: If new web application was deployed, deploy it to DEV too

# Create pre-change baseline
ansible-playbook -i hosts ansible/playbooks/task3-create-baseline.yml \
  -e "target_group=dev_machines" \
  -e "baseline_name=auto-tune-before-$(date +%Y%m%d)"

# Deploy the application/configuration change to DEV
# (this is application-specific)

# Create post-change baseline and detect
ansible-playbook -i hosts ansible/playbooks/fine-tune-profile-dev.yml \
  -e "baseline_before=/root/apache-monitor/data/baselines/auto-tune-before-*.json" \
  -e "create_post_update_baseline=yes" \
  -e "apply_suggestions=yes" \
  -e "run_tests=yes" \
  -e "audit_all_paths=true"
```

**Tools Used:**
- Same as regular update workflow
- Application deployment tools (depends on your app)

### Step 3.4: Iterative Testing

**Location:** DEV Machine

```bash
# Test the application thoroughly on DEV
# Include edge cases and stress testing

# Monitor AppArmor logs during testing
ansible -i hosts dev_machines -b -m command \
  -a "aa-audit /usr/sbin/apache2"

# Run application-specific tests
ansible -i hosts dev_machines -m command \
  -a "/root/apache-monitor/scripts/test_apache_functions.sh"

# Review any remaining denials
ansible -i hosts dev_machines -b -m command \
  -a "grep DENIED /var/log/syslog | grep apache2"

# If denials found, analyze and add to profile manually or re-run detection
```

**Iteration Loop:**

```
Test → Detect Denials → Update Profile → Test → Repeat until no denials
```

### Step 3.5: Deploy Refined Profile

**Location:** Control Panel → Production Fleet

```bash
# Once DEV testing is clean, deploy to production
ansible-playbook -i hosts ansible/playbooks/deploy-profile-to-production.yml \
  -e "source_host=dev-pi.local" \
  -e "profile_file=/etc/apparmor.d/usr.sbin.apache2"

# Enable audit mode on production for monitoring period (optional)
ansible -i hosts production -b -m command \
  -a "aa-audit /usr/sbin/apache2"

# After 24-48 hours, if stable, return to enforce mode
ansible -i hosts production -b -m command \
  -a "aa-enforce /usr/sbin/apache2"
```

---

## Tools and Components Reference

### Ansible Playbooks

| Playbook | Purpose | Key Parameters |
|----------|---------|----------------|
| `apparmor-deploy.yml` | Deploy monitoring infrastructure | None |
| `task3-create-baseline.yml` | Create system baseline snapshot | `target_group`, `baseline_name` |
| `fine-tune-profile-dev.yml` | Complete DEV fine-tuning workflow | `baseline_before`, `create_post_update_baseline`, `apply_suggestions`, `run_tests`, `audit_all_paths` |
| `task3-detect-changes.yml` | Standalone change detection | `baseline_before`, `baseline_after`, `output_format` |
| `task3-approve-profile-update.yml` | Generate approval document | `delta_file`, `suggestions_file`, `dev_test_passed` |
| `deploy-profile-to-production.yml` | Deploy profile to production | `source_host`, `profile_file` |
| `apparmor-dev-to-prod-workflow.yml` | Master orchestration workflow | Combines multiple playbooks |

### Shell Scripts

| Script | Location | Purpose | Key Options |
|--------|----------|---------|-------------|
| `create_baseline.sh` | `/root/apache-monitor/scripts/` (on RPi) | Capture system state | `--name`, `--profile` |
| `detect_apache_changes.sh` | `/root/apache-monitor/scripts/` (on RPi) | Compare baselines | `--audit-all-paths`, `--audit-system-paths`, `--output-profile-only`, `--profile-format` |
| `test_apache_functions.sh` | `/root/apache-monitor/scripts/` (on RPi) | Test Apache functionality | None (self-contained) |
| `create_approval_document.sh` | `/root/apache-monitor/scripts/` (on RPi) | Generate approval doc | `--delta`, `--suggestions`, `--dev-test-passed` |

### Ansible Roles

| Role | Purpose | Key Tasks |
|------|---------|-----------|
| `apparmor-profile` | Profile management | `backup-profile.yml`, `apply-profile.yml` |

### AppArmor Commands

| Command | Purpose | When to Use |
|---------|---------|-------------|
| `aa-status` | Check profile status | Verification after any change |
| `aa-enforce <profile>` | Set profile to enforce mode | After testing is complete |
| `aa-audit <profile>` | Set profile to audit mode | During testing/monitoring periods |
| `aa-complain <profile>` | Set profile to complain mode | Rarely used (logs only, no enforcement) |
| `apparmor_parser -r <file>` | Reload profile | After editing profile file |

### Key File Locations

**On Raspberry Pi (DEV and Production):**

```
/etc/apparmor.d/
├── usr.sbin.apache2                    # Main AppArmor profile
└── backups/
    └── usr.sbin.apache2.<timestamp>    # Automatic backups

/root/apache-monitor/
├── scripts/
│   ├── create_baseline.sh
│   ├── detect_apache_changes.sh
│   └── test_apache_functions.sh
├── data/
│   ├── baselines/
│   │   ├── pre-update-<timestamp>.json
│   │   └── post-update-<timestamp>.json
│   └── deltas/
│       ├── apache-delta-<timestamp>.json
│       ├── apache-delta-<timestamp>.txt
│       └── apparmor-profile-suggestions-<timestamp>.txt
└── approvals/
    └── apache-approval-<timestamp>.md

/var/log/
├── syslog                              # AppArmor DENIED/AUDIT logs
└── apache2/
    ├── access.log
    └── error.log
```

**On Control Panel:**

```
/home/jimmyloi/ansible_apparmor/
├── hosts                               # Ansible inventory
├── ansible/
│   ├── playbooks/                      # All playbooks
│   └── roles/                          # Ansible roles
├── scripts/                            # Helper scripts
├── profiles/                           # Master profile templates
├── data/
│   ├── baselines/                      # Fetched baselines
│   └── deltas/                         # Fetched delta reports
├── approvals/                          # Approval documents
└── monitoring/                         # Daily monitoring logs
```

---

## Rollback Procedures

### Scenario 1: Profile Causes Apache Failure

**Symptoms:**
- Apache won't start
- Apache is running but non-functional
- Critical denials in logs

**Immediate Rollback:**

```bash
# On affected machine(s)
ssh pi@<affected-host>

# Find latest backup
ls -lt /etc/apparmor.d/backups/usr.sbin.apache2.*

# Restore backup
sudo cp /etc/apparmor.d/backups/usr.sbin.apache2.<timestamp> \
        /etc/apparmor.d/usr.sbin.apache2

# Reload profile
sudo apparmor_parser -r /etc/apparmor.d/usr.sbin.apache2

# Restart Apache
sudo systemctl restart apache2

# Verify
curl http://localhost/
```

**Automated Rollback from Control Panel:**

```bash
# Rollback all production machines
ansible -i hosts production -b -m copy \
  -a "src=/etc/apparmor.d/backups/usr.sbin.apache2.<timestamp> \
      dest=/etc/apparmor.d/usr.sbin.apache2 \
      remote_src=yes"

ansible -i hosts production -b -m command \
  -a "apparmor_parser -r /etc/apparmor.d/usr.sbin.apache2"

ansible -i hosts production -b -m systemd \
  -a "name=apache2 state=restarted"
```

### Scenario 2: OS Update Causes System Instability

**Symptoms:**
- System performance degradation
- Unexpected service failures
- Kernel issues

**Rollback Steps:**

```bash
# List installed packages and their previous versions
ssh pi@<affected-host>
dpkg -l | grep ^ii

# Downgrade specific package
sudo apt install <package-name>=<previous-version> --allow-downgrades

# Or: Full system snapshot rollback (if using LVM snapshots or similar)
# This is system-specific and depends on your backup strategy
```

**Prevention:** Always test OS updates on DEV first. Never update production until DEV is verified stable for 24+ hours.

### Scenario 3: Profile Deployed but Has Minor Issues

**Symptoms:**
- Most functionality works
- Some specific features are blocked
- Non-critical denials in logs

**Mitigation (Temporary):**

```bash
# Switch to audit mode (allows everything, logs all access)
ansible -i hosts production -b -m command \
  -a "aa-audit /usr/sbin/apache2"

# Monitor for 24 hours to gather data
# Then fix profile on DEV and redeploy
```

---

## Troubleshooting

### Issue: Baseline creation hangs or fails

**Symptoms:**
- `create_baseline.sh` script doesn't complete
- JSON file is empty or malformed

**Diagnosis:**

```bash
# Check if Apache is running
ansible -i hosts <target> -b -m command -a "systemctl status apache2"

# Run baseline script manually with debug
ansible -i hosts <target> -b -m command \
  -a "/root/apache-monitor/scripts/create_baseline.sh --name debug --profile /usr/sbin/apache2"

# Check for file system issues
ansible -i hosts <target> -b -m command -a "df -h"
ansible -i hosts <target> -b -m command -a "ls -la /root/apache-monitor/data/baselines/"
```

**Solutions:**
- Ensure `/root/apache-monitor/` has sufficient space
- Verify AppArmor is running: `aa-status`
- Check syslog for AppArmor messages: `grep apparmor /var/log/syslog`

### Issue: Change detection finds no changes

**Symptoms:**
- Delta report shows all zeros
- No suggestions generated

**Diagnosis:**

```bash
# Verify baseline files exist and are valid JSON
ansible -i hosts dev_machines -b -m command \
  -a "cat /root/apache-monitor/data/baselines/pre-update-*.json | jq ."

ansible -i hosts dev_machines -b -m command \
  -a "cat /root/apache-monitor/data/baselines/post-update-*.json | jq ."

# Check if Apache was actually using new paths
ansible -i hosts dev_machines -b -m command \
  -a "grep apache2 /var/log/syslog | tail -50"
```

**Solutions:**
- Ensure Apache was actively used between baselines (run test requests)
- Verify AppArmor logging is enabled
- Check if post-update baseline captured enough activity (may need to wait longer)

### Issue: Profile syntax errors after applying suggestions

**Symptoms:**
- `apparmor_parser` fails with syntax error
- Profile won't reload

**Diagnosis:**

```bash
# Test profile syntax manually
ansible -i hosts <target> -b -m command \
  -a "apparmor_parser -QT /etc/apparmor.d/usr.sbin.apache2"

# Check profile contents
ansible -i hosts <target> -b -m command \
  -a "cat /etc/apparmor.d/usr.sbin.apache2"
```

**Common Issues:**
- **Duplicate rules**: Clean up with `sort | uniq`
- **Invalid characters**: Check for special chars in paths
- **Mismatched braces**: Ensure profile has proper `{ }` structure
- **Unsupported `audit` keyword**: Older AppArmor versions don't support `audit` qualifier on allow rules

**Solutions:**

```bash
# Clean up profile manually
ansible -i hosts <target> -b -m command \
  -a "sed -i 's/^audit //' /etc/apparmor.d/usr.sbin.apache2"

# Or restore backup and re-apply suggestions with corrections
ansible -i hosts <target> -b -m copy \
  -a "src=/etc/apparmor.d/backups/usr.sbin.apache2.<timestamp> \
      dest=/etc/apparmor.d/usr.sbin.apache2 \
      remote_src=yes"
```

### Issue: aa-audit or aa-enforce commands fail

**Symptoms:**
- Error: `ERROR: Cannot find <profile> in /etc/apparmor.d`
- Or: Profile name mismatch

**Diagnosis:**

```bash
# Check profile name vs file path
ansible -i hosts <target> -b -m command -a "aa-status"

# List profiles in enforce mode vs complain mode
ansible -i hosts <target> -b -m command -a "aa-status | grep -A 20 'profiles are loaded'"
```

**Solutions:**
- Ensure profile name matches binary path: `/usr/sbin/apache2`
- Check symlinks: `/usr/sbin/apache2` should be the actual binary or canonical path
- Verify profile file exists: `/etc/apparmor.d/usr.sbin.apache2`

### Issue: DEV tests pass but production has denials

**Symptoms:**
- DEV environment shows no issues
- Production immediately has DENIED logs

**Diagnosis:**

```bash
# Compare configurations
# DEV:
ansible -i hosts dev_machines -b -m command -a "apache2 -V"
ansible -i hosts dev_machines -b -m command -a "ls -la /etc/apache2/sites-enabled/"

# Production:
ansible -i hosts production -b -m command -a "apache2 -V"
ansible -i hosts production -b -m command -a "ls -la /etc/apache2/sites-enabled/"

# Check for environment differences
# - Different loaded modules
# - Different virtual hosts
# - Different file paths (symlinks, mounts)
```

**Solutions:**
- Ensure DEV truly mirrors production (configuration, workload, modules)
- Run more comprehensive tests on DEV (cover all production use cases)
- Consider environment-specific profile variants if necessary

### Issue: Ansible playbook fails with "recursive loop detected"

**Symptoms:**
- Playbook aborts with templating error
- Variable references itself

**Diagnosis:**

```bash
# Check playbook for circular variable definitions
grep -n "{{ variable_name }}" ansible/playbooks/<playbook>.yml
```

**Solutions:**
- Use intermediate variables to break recursion
- Ensure `default()` filters don't reference the variable being defined
- Example fix: Use `_data` suffix for decoded content, then set main variable from it

---

# AppArmor Profile Management - Visual Flowcharts

## High-Level System Overview

```mermaid
graph TB
    Start([Start: New Deployment]) --> Phase0[Phase 0: Zero-Day Deployment]
    Phase0 --> Production[Production Running with Baseline Profiles]
    
    Production --> Decision{What Triggers Change?}
    
    Decision -->|Scheduled OS Update| Phase1[Phase 1: Regular Update Workflow]
    Decision -->|Emergency Security Patch| Phase2[Phase 2: Ad-Hoc/Emergency Update]
    Decision -->|Application Changes or Denials| Phase3[Phase 3: Auto-Finetuning]
    
    Phase1 --> Monitor[Continuous Monitoring]
    Phase2 --> Monitor
    Phase3 --> Monitor
    
    Monitor --> Decision
    
    style Phase0 fill:#e1f5ff,stroke:#01579b,stroke-width:3px
    style Phase1 fill:#e8f5e9,stroke:#2e7d32,stroke-width:3px
    style Phase2 fill:#fff3e0,stroke:#e65100,stroke-width:3px
    style Phase3 fill:#f3e5f5,stroke:#4a148c,stroke-width:3px
    style Production fill:#c8e6c9,stroke:#388e3c,stroke-width:2px
    style Monitor fill:#fff9c4,stroke:#f57f17,stroke-width:2px
```

---

## Phase 0: Zero-Day Deployment - Detailed Flow

```mermaid
flowchart TD
    Start([Start: Fresh Infrastructure]) --> Setup[Setup Control Panel<br/>Install Ansible, Clone Repo]
    Setup --> Inventory[Configure Inventory File<br/>hosts: control, dev_machines, production]
    Inventory --> TestConn[Test SSH Connectivity<br/>ansible -m ping]
    
    TestConn -->|Failed| FixSSH[Fix SSH Keys/Permissions]
    FixSSH --> TestConn
    
    TestConn -->|Success| Deploy[Deploy Monitoring Infrastructure<br/>apparmor-deploy.yml]
    
    Deploy --> DeploySteps[Deploy to All RPis:<br/>- /root/apache-monitor/ structure<br/>- create_baseline.sh<br/>- detect_apache_changes.sh<br/>- test_apache_functions.sh]
    
    DeploySteps --> CreateProdBase[Create Initial Production Baselines<br/>task3-create-baseline.yml<br/>target_group=production]
    
    CreateProdBase --> ProdBaseFiles[Production Baseline Files Created<br/>initial-production-timestamp.json]
    
    ProdBaseFiles --> DeployProfile[Deploy Initial AppArmor Profiles<br/>Copy profiles/usr.sbin.apache2<br/>to /etc/apparmor.d/]
    
    DeployProfile --> ReloadProd[Reload Profiles on Production<br/>apparmor_parser -r<br/>aa-enforce /usr/sbin/apache2]
    
    ReloadProd --> VerifyProd{Verify Production<br/>Apache Running?<br/>No DENIED logs?}
    
    VerifyProd -->|Issues Found| TroubleshootProd[Troubleshoot:<br/>Check aa-status<br/>Review syslog<br/>Test HTTP access]
    TroubleshootProd --> VerifyProd
    
    VerifyProd -->|Success| SyncDev[Sync DEV Machine<br/>Copy same profile<br/>Create initial-dev baseline]
    
    SyncDev --> VerifyDev{Verify DEV<br/>Mirrors Production?}
    
    VerifyDev -->|No| FixDev[Fix Configuration Differences]
    FixDev --> SyncDev
    
    VerifyDev -->|Yes| Complete([Phase 0 Complete<br/>Ready for Operations])
    
    style Start fill:#e1f5ff,stroke:#01579b,stroke-width:3px
    style Complete fill:#c8e6c9,stroke:#388e3c,stroke-width:3px
    style VerifyProd fill:#fff3e0,stroke:#e65100,stroke-width:2px
    style VerifyDev fill:#fff3e0,stroke:#e65100,stroke-width:2px
    style TroubleshootProd fill:#ffebee,stroke:#c62828,stroke-width:2px
    style DeploySteps fill:#e3f2fd,stroke:#1976d2,stroke-width:2px
```

---

## Phase 1: Regular OS Update Workflow - Detailed Flow

```mermaid
flowchart TD
    Start([Scheduled OS Update Triggered]) --> PreBaseline[Step 1.1: Create Pre-Update Baseline on DEV<br/>task3-create-baseline.yml<br/>target_group=dev_machines<br/>baseline_name=pre-update-YYYYMMDD]
    
    PreBaseline --> PreBaseFile[Baseline File Created:<br/>pre-update-YYYYMMDD-timestamp.json]
    
    PreBaseFile --> OSUpdate[Step 1.2: Perform OS Update on DEV<br/>SSH to DEV<br/>apt update && apt upgrade<br/>Reboot if needed]
    
    OSUpdate --> VerifyDev{Verify DEV<br/>System Stable?<br/>Apache Running?}
    
    VerifyDev -->|Failed| RollbackDev[Rollback DEV<br/>Investigate Issues]
    RollbackDev --> PreBaseline
    
    VerifyDev -->|Success| FineTune[Step 1.3: Automated Fine-Tuning<br/>fine-tune-profile-dev.yml]
    
    FineTune --> FineTuneSteps[Fine-Tuning Process:<br/>1. Create post-update baseline<br/>2. Detect changes detect_apache_changes.sh<br/>3. Generate profile suggestions<br/>4. Apply suggestions to profile<br/>5. Toggle aa-audit mode ON<br/>6. Run functional tests<br/>7. Toggle aa-audit mode OFF]
    
    FineTuneSteps --> FineTuneOutput[Outputs Generated:<br/>- apache-delta-timestamp.json<br/>- apparmor-profile-suggestions-timestamp.txt<br/>- Updated profile on DEV<br/>- Test results]
    
    FineTuneOutput --> TestResult{DEV Tests<br/>Passed?}
    
    TestResult -->|Failed| Iterate[Manual Profile Refinement<br/>or<br/>Re-run refine-profile-dev.yml]
    Iterate --> TestResult
    
    TestResult -->|Passed| GenApproval[Step 1.4: Generate Approval Document<br/>task3-approve-profile-update.yml<br/>Include DEV test results]
    
    GenApproval --> ApprovalDoc[Approval Document Created:<br/>apache-approval-timestamp.md<br/>Contains: delta, suggestions, test results]
    
    ApprovalDoc --> HumanReview{Step 1.5: Human Review<br/>Changes Expected?<br/>Security Concerns?<br/>Ready for Production?}
    
    HumanReview -->|Rejected| InvestigateChanges[Investigate Unexpected Changes<br/>Refine Profile<br/>Re-test]
    InvestigateChanges --> FineTune
    
    HumanReview -->|Approved| DeployProd[Step 1.6: Deploy Profile to Production<br/>deploy-profile-to-production.yml<br/>Copy from DEV to all production]
    
    DeployProd --> ProdDeploySteps[Production Deployment:<br/>1. Backup current profiles<br/>2. Copy tested profile from DEV<br/>3. Reload profiles apparmor_parser<br/>4. Optional: Enable aa-audit mode<br/>5. Verify Apache running]
    
    ProdDeploySteps --> VerifyProdDeploy{Verify Production<br/>Profile Loaded?<br/>Apache Running?<br/>No Immediate Denials?}
    
    VerifyProdDeploy -->|Failed| RollbackProd[Emergency Rollback<br/>Restore from backup<br/>Restart Apache]
    RollbackProd --> Investigation[Root Cause Analysis]
    Investigation --> FineTune
    
    VerifyProdDeploy -->|Success| ProdOSUpdate[Step 1.7: OS Update on Production<br/>Rolling Update Strategy<br/>One or few machines at a time]
    
    ProdOSUpdate --> UpdateLoop[For Each Production Machine:<br/>1. apt update && apt upgrade<br/>2. Reboot if needed<br/>3. Verify Apache<br/>4. Check for denials<br/>5. Wait 10-15 mins<br/>6. Proceed to next machine]
    
    UpdateLoop --> AllUpdated{All Production<br/>Machines Updated?}
    
    AllUpdated -->|No| UpdateLoop
    
    AllUpdated -->|Yes| Monitor[Step 1.8: Monitor Production<br/>24-72 hours]
    
    Monitor --> MonitorSteps[Monitoring Activities:<br/>- Check AppArmor logs daily<br/>- Review DENIED events<br/>- Analyze AUDIT logs if enabled<br/>- Monitor Apache error logs<br/>- Check application functionality]
    
    MonitorSteps --> MonitorResult{Monitoring Period<br/>Stable?<br/>No Issues?}
    
    MonitorResult -->|Issues Found| HandleIssues[Analyze Issues:<br/>- Minor: Add to next fine-tuning<br/>- Major: Emergency profile update]
    HandleIssues --> MonitorResult
    
    MonitorResult -->|Stable| Finalize[Finalize:<br/>- Ensure aa-enforce mode<br/>- Create post-update baseline<br/>- Archive approval document<br/>- Update documentation]
    
    Finalize --> Complete([Phase 1 Complete<br/>Production Stable & Updated])
    
    style Start fill:#e8f5e9,stroke:#2e7d32,stroke-width:3px
    style Complete fill:#c8e6c9,stroke:#388e3c,stroke-width:3px
    style VerifyDev fill:#fff3e0,stroke:#e65100,stroke-width:2px
    style TestResult fill:#fff3e0,stroke:#e65100,stroke-width:2px
    style HumanReview fill:#fff3e0,stroke:#e65100,stroke-width:2px
    style VerifyProdDeploy fill:#fff3e0,stroke:#e65100,stroke-width:2px
    style MonitorResult fill:#fff3e0,stroke:#e65100,stroke-width:2px
    style RollbackDev fill:#ffebee,stroke:#c62828,stroke-width:2px
    style RollbackProd fill:#ffebee,stroke:#c62828,stroke-width:2px
    style FineTuneSteps fill:#e3f2fd,stroke:#1976d2,stroke-width:2px
    style ProdDeploySteps fill:#e3f2fd,stroke:#1976d2,stroke-width:2px
    style MonitorSteps fill:#fff9c4,stroke:#f57f17,stroke-width:2px
```

---

## Phase 2: Ad-Hoc/Emergency Update Workflow - Detailed Flow

```mermaid
flowchart TD
    Start([Emergency Security Alert Received]) --> Assess[Step 2.1: Emergency Assessment<br/>- What packages affected?<br/>- Does it affect Apache?<br/>- Severity level?<br/>- Can we test on DEV first?]
    
    Assess --> Decision{Risk Assessment<br/>Critical Security Issue?<br/>Time Available?}
    
    Decision -->|Low Risk / Time Available| RegularPath[Use Regular Update Workflow<br/>Phase 1 Process]
    RegularPath --> End1([Follow Phase 1])
    
    Decision -->|High Risk / Urgent| CheckProd[Check Current Production State<br/>- Package versions<br/>- Apache status<br/>- Current profile status]
    
    CheckProd --> EmergencyBaseline[Step 2.2: Fast-Track DEV Testing<br/>Create emergency baseline<br/>baseline_name=emergency-YYYYMMDD-HHMMSS]
    
    EmergencyBaseline --> EmergencyUpdate[Apply Emergency Update on DEV<br/>apt install package-name=version<br/>or<br/>apt upgrade specific-packages]
    
    EmergencyUpdate --> FastFineTune[Run Fast Fine-Tuning<br/>fine-tune-profile-dev.yml<br/>audit_all_paths=false faster<br/>create_post_update_baseline=yes<br/>apply_suggestions=yes<br/>run_tests=yes]
    
    FastFineTune --> FastTest{DEV Quick Test<br/>Apache Functional?<br/>Critical Features Work?}
    
    FastTest -->|Failed| CriticalDecision{Changes Significant?<br/>Can We Fix Quickly?}
    
    CriticalDecision -->|Yes, Fixable| QuickFix[Quick Profile Fix<br/>Manual adjustment]
    QuickFix --> FastTest
    
    CriticalDecision -->|No, Cannot Fix| Escalate[Escalate Issue<br/>- Consider workarounds<br/>- Delay deployment<br/>- Consult security team]
    Escalate --> End2([Alternative Solution Required])
    
    FastTest -->|Passed| MinimalApproval[Generate Emergency Approval<br/>Abbreviated approval doc<br/>Focus on critical changes]
    
    MinimalApproval --> QuickReview{Emergency Approval<br/>by On-Call Team?}
    
    QuickReview -->|Rejected| Escalate
    
    QuickReview -->|Approved| RolloutPlan[Step 2.3: Emergency Deployment Plan<br/>- Select canary machine<br/>- Prepare rollback commands<br/>- Set up monitoring alerts]
    
    RolloutPlan --> DeployProfile[Deploy Profile to Production<br/>deploy-profile-to-production.yml<br/>Enable aa-audit for monitoring]
    
    DeployProfile --> CanaryDeploy[Apply Update to Canary Machine<br/>First production machine only]
    
    CanaryDeploy --> CanaryTest{Canary Verification<br/>15-30 min observation<br/>Apache working?<br/>No denials?<br/>Functionality OK?}
    
    CanaryTest -->|Failed| EmergencyRollback[EMERGENCY ROLLBACK<br/>- Restore profile backup<br/>- Downgrade package<br/>- Restart Apache<br/>- Alert team]
    EmergencyRollback --> PostMortem[Post-Mortem Analysis<br/>Document issues<br/>Plan alternative approach]
    PostMortem --> End3([Emergency Deployment Failed])
    
    CanaryTest -->|Success| RollingDeploy[Continue Rolling Deployment<br/>One machine at a time<br/>Faster intervals than regular<br/>but still controlled]
    
    RollingDeploy --> NextMachine[Deploy to Next Production Machine:<br/>1. Apply update<br/>2. Verify immediately<br/>3. Wait 5-10 mins<br/>4. Check logs]
    
    NextMachine --> MachineOK{Machine Stable?}
    
    MachineOK -->|Failed| PauseAndFix[Pause Rollout<br/>Investigate issue<br/>Fix or rollback this machine]
    PauseAndFix --> PauseDecision{Continue Rollout?}
    PauseDecision -->|No| End4([Partial Deployment<br/>Investigate Further])
    PauseDecision -->|Yes| NextMachine
    
    MachineOK -->|Success| AllDone{All Production<br/>Machines Updated?}
    
    AllDone -->|No| NextMachine
    
    AllDone -->|Yes| IntenseMonitor[Step 2.4: Intensive Monitoring<br/>First 1-2 hours critical]
    
    IntenseMonitor --> MonitorActivities[Monitoring Every 15 Minutes:<br/>- Check syslog for denials<br/>- Apache error logs<br/>- Service status checks<br/>- Application functionality<br/>- Performance metrics]
    
    MonitorActivities --> FirstHour{First Hour<br/>Stable?}
    
    FirstHour -->|Issues Detected| HandleEmergency[Emergency Response:<br/>- Assess severity<br/>- Rollback if critical<br/>- Quick fix if minor<br/>- Document issue]
    HandleEmergency --> PostMortem
    
    FirstHour -->|Stable| ExtendMonitor[Continue Monitoring<br/>24 hours with reduced frequency<br/>Every 2-4 hours]
    
    ExtendMonitor --> FinalCheck{24-Hour Period<br/>All Stable?}
    
    FinalCheck -->|Issues| HandleEmergency
    
    FinalCheck -->|Stable| FinalizeEmergency[Finalize Emergency Deployment:<br/>- Document actions taken<br/>- Update runbooks<br/>- Schedule follow-up review<br/>- Return to normal monitoring]
    
    FinalizeEmergency --> Complete([Emergency Update Complete<br/>Return to Normal Operations])
    
    style Start fill:#fff3e0,stroke:#e65100,stroke-width:3px
    style Complete fill:#c8e6c9,stroke:#388e3c,stroke-width:3px
    style Decision fill:#ffccbc,stroke:#d84315,stroke-width:2px
    style QuickReview fill:#ffccbc,stroke:#d84315,stroke-width:2px
    style CanaryTest fill:#ffccbc,stroke:#d84315,stroke-width:2px
    style MachineOK fill:#ffccbc,stroke:#d84315,stroke-width:2px
    style FirstHour fill:#ffccbc,stroke:#d84315,stroke-width:2px
    style FinalCheck fill:#ffccbc,stroke:#d84315,stroke-width:2px
    style EmergencyRollback fill:#ffebee,stroke:#c62828,stroke-width:3px
    style Escalate fill:#ffebee,stroke:#c62828,stroke-width:2px
    style MonitorActivities fill:#fff9c4,stroke:#f57f17,stroke-width:2px
```

---

## Phase 3: Auto-Finetuning Process - Detailed Flow

```mermaid
flowchart TD
    Start([Continuous Operations<br/>Production Running]) --> Monitor[Step 3.1: Continuous Monitoring<br/>Daily Cron Job]
    
    Monitor --> DailyCheck[Daily Log Collection:<br/>- Collect AppArmor DENIED logs<br/>- Grep for apache2 denials<br/>- Store in monitoring/denials-DATE.log<br/>- Send alert if denials found]
    
    DailyCheck --> DenialsFound{Denials<br/>Detected?}
    
    DenialsFound -->|No| Monitor
    
    DenialsFound -->|Yes| Alert[Alert Sent to Admin<br/>Email or notification]
    
    Alert --> Analyze[Step 3.2: Analyze Denials<br/>Review collected denial logs]
    
    Analyze --> AnalysisSteps[Analysis Process:<br/>1. Aggregate denials<br/>2. Count occurrences<br/>3. Identify patterns<br/>4. Group by path/operation<br/>5. Review timestamps]
    
    AnalysisSteps --> Categorize[Categorize Denials:<br/>- New file paths<br/>- Permission changes<br/>- Network operations<br/>- Capabilities<br/>- System calls]
    
    Categorize --> Assessment{Denial<br/>Assessment}
    
    Assessment -->|Expected: New Feature| LegitimateChange[Legitimate Change Identified:<br/>- New web application deployed<br/>- Configuration change made<br/>- Module enabled<br/>- Log rotation paths changed]
    
    Assessment -->|Unexpected: Potential Attack| SecurityReview[SECURITY REVIEW REQUIRED<br/>- Investigate source<br/>- Review application logs<br/>- Check for intrusion indicators<br/>- Consult security team]
    
    SecurityReview --> SecDecision{Security<br/>Assessment}
    
    SecDecision -->|False Positive/Safe| LegitimateChange
    SecDecision -->|Real Threat| Incident[Security Incident Response<br/>Not covered in this workflow]
    Incident --> End1([Follow Security Procedures])
    
    Assessment -->|Noise: False Positive| RefineProfile[Profile Refinement Needed<br/>Adjust overly restrictive rules]
    
    LegitimateChange --> ReplicatePlan[Step 3.3: Replicate on DEV<br/>Plan DEV replication]
    RefineProfile --> ReplicatePlan
    
    ReplicatePlan --> PreChangeBase[Create Pre-Change Baseline on DEV<br/>task3-create-baseline.yml<br/>baseline_name=auto-tune-before-YYYYMMDD]
    
    PreChangeBase --> ApplyChange[Apply Change on DEV<br/>Options:<br/>- Deploy new application<br/>- Change Apache config<br/>- Enable new module<br/>- Simulate production scenario]
    
    ApplyChange --> ChangeType{Type of Change?}
    
    ChangeType -->|Application Deployment| DeployApp[Deploy Application to DEV<br/>Use app-specific deployment tools]
    ChangeType -->|Configuration Change| ApplyConfig[Apply Configuration on DEV<br/>Copy configs, restart Apache]
    ChangeType -->|Profile Refinement| DirectProfile[Directly edit profile on DEV<br/>Add/modify rules based on analysis]
    
    DeployApp --> RunFineTune[Run Fine-Tuning Workflow<br/>fine-tune-profile-dev.yml<br/>baseline_before=auto-tune-before-*<br/>create_post_update_baseline=yes<br/>apply_suggestions=yes<br/>run_tests=yes<br/>audit_all_paths=true]
    
    ApplyConfig --> RunFineTune
    DirectProfile --> RunFineTune
    
    RunFineTune --> EnableAudit[Enable Audit Mode on DEV<br/>aa-audit /usr/sbin/apache2<br/>For comprehensive logging]
    
    EnableAudit --> TestPhase[Step 3.4: Iterative Testing<br/>Comprehensive test suite]
    
    TestPhase --> TestActivities[Testing Activities:<br/>1. Basic functionality tests<br/>2. Edge case scenarios<br/>3. Stress testing<br/>4. User workflow simulation<br/>5. Feature-specific tests]
    
    TestActivities --> ReviewLogs[Review DEV Logs:<br/>- Check for remaining denials<br/>- Review audit logs for issues<br/>- Verify all features work<br/>- Check performance]
    
    ReviewLogs --> TestResult{DEV Testing<br/>Result?}
    
    TestResult -->|Denials Still Present| AnalyzeDenials[Analyze Remaining Denials:<br/>- Are they expected?<br/>- Need more profile rules?<br/>- Application behavior issue?]
    
    AnalyzeDenials --> FixNeeded{Fix Required?}
    
    FixNeeded -->|Yes: Update Profile| AddRules[Add/Modify Profile Rules<br/>Manual or re-run detection<br/>Apply changes to DEV profile]
    AddRules --> EnableAudit
    
    FixNeeded -->|Yes: Fix Application| FixApp[Fix Application Code/Config<br/>Application issue, not profile issue]
    FixApp --> EnableAudit
    
    FixNeeded -->|No: Acceptable| TestResult
    
    TestResult -->|Clean: No Issues| DisableAudit[Disable Audit Mode on DEV<br/>aa-enforce /usr/sbin/apache2]
    
    DisableAudit --> FinalVerify[Final Verification on DEV:<br/>Run full test suite again<br/>Ensure enforce mode works<br/>No denials for normal operations]
    
    FinalVerify --> FinalResult{Final DEV<br/>Verification?}
    
    FinalResult -->|Issues| TestPhase
    
    FinalResult -->|Success| PrepDeploy[Prepare for Deployment:<br/>- Generate approval document<br/>- Document changes made<br/>- Backup current prod profile<br/>- Schedule deployment window]
    
    PrepDeploy --> ApprovalReview{Human Approval<br/>for Production<br/>Deployment?}
    
    ApprovalReview -->|Rejected| Revise[Revise Approach<br/>Address concerns<br/>Re-test if needed]
    Revise --> PrepDeploy
    
    ApprovalReview -->|Approved| DeployRefined[Step 3.5: Deploy Refined Profile<br/>deploy-profile-to-production.yml<br/>source_host=dev-pi.local]
    
    DeployRefined --> ProdAuditMode[Optional: Enable Audit Mode on Production<br/>aa-audit /usr/sbin/apache2<br/>For monitoring period]
    
    ProdAuditMode --> MonitorPeriod[Monitoring Period: 24-48 Hours<br/>- Watch for denials<br/>- Review audit logs<br/>- Monitor application behavior<br/>- Check user reports]
    
    MonitorPeriod --> MonitorResult{Monitoring<br/>Result?}
    
    MonitorResult -->|Issues Found| AssessIssue{Issue<br/>Severity?}
    
    AssessIssue -->|Critical| RollbackProfile[Rollback Profile<br/>Restore from backup<br/>Investigate issue]
    RollbackProfile --> PostIssue[Post-Issue Analysis<br/>Root cause investigation<br/>Plan corrective action]
    PostIssue --> ReplicatePlan
    
    AssessIssue -->|Minor| DocumentIssue[Document Minor Issue<br/>Plan for next iteration<br/>Continue monitoring]
    DocumentIssue --> MonitorResult
    
    MonitorResult -->|Stable| Finalize[Finalize Auto-Finetuning:<br/>- Ensure aa-enforce mode<br/>- Create new baseline<br/>- Document changes<br/>- Update profile version<br/>- Archive approval docs]
    
    Finalize --> UpdateMonitoring[Update Continuous Monitoring:<br/>Reset denial tracking<br/>Continue daily monitoring]
    
    UpdateMonitoring --> Complete([Auto-Finetuning Complete<br/>Return to Normal Monitoring])
    
    Complete --> Monitor
    
    style Start fill:#f3e5f5,stroke:#4a148c,stroke-width:3px
    style Complete fill:#c8e6c9,stroke:#388e3c,stroke-width:3px
    style DenialsFound fill:#e1bee7,stroke:#6a1b9a,stroke-width:2px
    style Assessment fill:#e1bee7,stroke:#6a1b9a,stroke-width:2px
    style SecDecision fill:#ffccbc,stroke:#d84315,stroke-width:2px
    style TestResult fill:#e1bee7,stroke:#6a1b9a,stroke-width:2px
    style FinalResult fill:#e1bee7,stroke:#6a1b9a,stroke-width:2px
    style ApprovalReview fill:#e1bee7,stroke:#6a1b9a,stroke-width:2px
    style MonitorResult fill:#e1bee7,stroke:#6a1b9a,stroke-width:2px
    style SecurityReview fill:#ffebee,stroke:#c62828,stroke-width:2px
    style RollbackProfile fill:#ffebee,stroke:#c62828,stroke-width:2px
    style TestActivities fill:#e3f2fd,stroke:#1976d2,stroke-width:2px
    style MonitorPeriod fill:#fff9c4,stroke:#f57f17,stroke-width:2px
```

---

## Rollback Decision Tree

```mermaid
flowchart TD
    Issue([Issue Detected in Production]) --> Assess{Assess<br/>Severity}
    
    Assess -->|Critical: Apache Down| Critical[CRITICAL ISSUE<br/>Immediate Action Required]
    Assess -->|Major: Functionality Broken| Major[MAJOR ISSUE<br/>Urgent Action Needed]
    Assess -->|Minor: Some Denials| Minor[MINOR ISSUE<br/>Can Monitor/Fix Later]
    
    Critical --> ImmediateRollback[Immediate Rollback:<br/>1. Restore profile from backup<br/>2. Reload profile<br/>3. Restart Apache<br/>4. Verify service restored]
    
    Major --> QuickDecision{Can Fix<br/>in < 30 mins?}
    
    QuickDecision -->|No| ImmediateRollback
    QuickDecision -->|Yes| QuickFix[Attempt Quick Fix:<br/>1. Edit profile<br/>2. Reload profile<br/>3. Test immediately<br/>4. If fails → rollback]
    
    QuickFix --> FixResult{Fix<br/>Successful?}
    
    FixResult -->|No| ImmediateRollback
    FixResult -->|Yes| Monitor[Continue Monitoring]
    
    Minor --> AuditMode{Switch to<br/>Audit Mode?}
    
    AuditMode -->|Yes| EnableAudit[Enable aa-audit mode<br/>Allow all, log everything<br/>Buys time to fix properly]
    AuditMode -->|No| DocumentMinor[Document Issue<br/>Plan proper fix<br/>Continue monitoring]
    
    EnableAudit --> ScheduleFix[Schedule Proper Fix<br/>on DEV environment<br/>Follow auto-finetuning process]
    DocumentMinor --> ScheduleFix
    
    ImmediateRollback --> VerifyRollback{Rollback<br/>Successful?}
    
    VerifyRollback -->|No| Escalate[ESCALATE:<br/>Contact senior admin<br/>May need system rollback]
    VerifyRollback -->|Yes| PostMortem[Post-Mortem Analysis:<br/>1. Root cause identification<br/>2. Document what went wrong<br/>3. Plan corrective action<br/>4. Update procedures]
    
    PostMortem --> ReTest[Re-test on DEV:<br/>Reproduce issue<br/>Fix properly<br/>Verify solution]
    
    ScheduleFix --> ReTest
    
    ReTest --> ReDeploy[Re-deployment:<br/>Follow normal workflow<br/>Extra validation<br/>More cautious rollout]
    
    Monitor --> Complete([Issue Resolved])
    ReDeploy --> Complete
    Escalate --> End([System-Level Intervention Required])
    
    style Issue fill:#fff3e0,stroke:#e65100,stroke-width:3px
    style Critical fill:#ffebee,stroke:#c62828,stroke-width:3px
    style Major fill:#ffccbc,stroke:#d84315,stroke-width:2px
    style Minor fill:#fff9c4,stroke:#f57f17,stroke-width:2px
    style ImmediateRollback fill:#ef5350,stroke:#b71c1c,stroke-width:3px,color:#fff
    style Complete fill:#c8e6c9,stroke:#388e3c,stroke-width:3px
    style Escalate fill:#d32f2f,stroke:#b71c1c,stroke-width:3px,color:#fff
```

---

## Tools and Components Interaction Map

```mermaid
graph TB
    subgraph Control["Control Panel (Ansible Controller)"]
        Operator[Human Operator]
        Ansible[Ansible Engine]
        Playbooks[Playbooks Directory]
        Inventory[Inventory File hosts]
        LocalData[Local Data Storage]
    end
    
    subgraph DEV["DEV Machine (Test Environment)"]
        DEVApache[Apache Service]
        DEVProfile[AppArmor Profile<br/>/etc/apparmor.d/usr.sbin.apache2]
        DEVMonitor[Monitoring Scripts<br/>/root/apache-monitor/]
        DEVBaselines[Baseline Data<br/>JSON files]
        DEVLogs[System Logs<br/>/var/log/syslog]
    end
    
    subgraph Production["Production Fleet (Multiple RPis)"]
        ProdApache[Apache Services]
        ProdProfile[AppArmor Profiles]
        ProdMonitor[Monitoring Scripts]
        ProdBaselines[Baseline Data]
        ProdLogs[System Logs]
    end
    
    subgraph Scripts["Key Scripts"]
        CreateBaseline[create_baseline.sh<br/>Captures system state]
        DetectChanges[detect_apache_changes.sh<br/>Compares baselines]
        TestFunc[test_apache_functions.sh<br/>Functional tests]
        CreateApproval[create_approval_document.sh<br/>Generates approval docs]
    end
    
    subgraph PlaybookFiles["Key Playbooks"]
        Deploy[apparmor-deploy.yml<br/>Initial deployment]
        CreateBase[task3-create-baseline.yml<br/>Baseline creation]
        FineTune[fine-tune-profile-dev.yml<br/>DEV fine-tuning]
        DeployProd[deploy-profile-to-production.yml<br/>Production deployment]
        Workflow[apparmor-dev-to-prod-workflow.yml<br/>Master orchestration]
    end
    
    subgraph AppArmorCmds["AppArmor Commands"]
        AAStatus[aa-status<br/>Check profile status]
        AAEnforce[aa-enforce<br/>Enable enforcement]
        AAAudit[aa-audit<br/>Enable audit logging]
        AAParser[apparmor_parser<br/>Reload profiles]
    end
    
    Operator -->|Executes| Ansible
    Ansible -->|Reads| Inventory
    Ansible -->|Runs| Playbooks
    Playbooks -->|Contains| PlaybookFiles
    
    Ansible -->|SSH: Deploy & Execute| DEV
    Ansible -->|SSH: Deploy & Execute| Production
    
    DEVMonitor -->|Contains| Scripts
    ProdMonitor -->|Contains| Scripts
    
    CreateBaseline -->|Captures| DEVApache
    CreateBaseline -->|Creates| DEVBaselines
    DetectChanges -->|Compares| DEVBaselines
    DetectChanges -->|Generates| Suggestions[Profile Suggestions]
    
    FineTune -->|Orchestrates| CreateBaseline
    FineTune -->|Orchestrates| DetectChanges
    FineTune -->|Orchestrates| TestFunc
    FineTune -->|Modifies| DEVProfile
    FineTune -->|Calls| AAEnforce
    FineTune -->|Calls| AAAudit
    
    DeployProd -->|Copies| DEVProfile
    DeployProd -->|Updates| ProdProfile
    DeployProd -->|Calls| AAParser
    
    TestFunc -->|Tests| DEVApache
    TestFunc -->|Reads| DEVLogs
    
    Ansible -->|Fetches Results| LocalData
    CreateApproval -->|Uses| LocalData
    CreateApproval -->|Produces| ApprovalDoc[Approval Documents]
    Operator -->|Reviews| ApprovalDoc
    
    AAStatus -->|Queries| DEVProfile
    AAStatus -->|Queries| ProdProfile
    
    style Control fill:#e3f2fd,stroke:#1976d2,stroke-width:3px
    style DEV fill:#f1f8e9,stroke:#558b2f,stroke-width:3px
    style Production fill:#fff3e0,stroke:#ef6c00,stroke-width:3px
    style Scripts fill:#e1f5fe,stroke:#0277bd,stroke-width:2px
    style PlaybookFiles fill:#ede7f6,stroke:#512da8,stroke-width:2px
    style AppArmorCmds fill:#fce4ec,stroke:#c2185b,stroke-width:2px
```

---

## Decision Matrix for Workflow Selection

```mermaid
flowchart TD
    Start{What Triggered<br/>the Need?}
    
    Start -->|Scheduled Maintenance| Scheduled{Timeline?}
    Start -->|Security Alert| Security{Severity?}
    Start -->|Application Change| AppChange{Change Type?}
    Start -->|AppArmor Denials| Denials{Frequency?}
    
    Scheduled -->|Monthly/Quarterly| RegularUpdate[Use Phase 1:<br/>Regular OS Update Workflow<br/><br/>Full process with:<br/>- Complete testing<br/>- Comprehensive approval<br/>- Gradual rollout]
    
    Security -->|Critical CVE| Emergency[Use Phase 2:<br/>Ad-Hoc/Emergency Workflow<br/><br/>Fast-track with:<br/>- Minimal testing<br/>- Quick approval<br/>- Canary deployment]
    
    Security -->|Standard Security Update| RegularUpdate
    
    AppChange -->|New Feature Deployment| Feature[Use Phase 3:<br/>Auto-Finetuning<br/><br/>Focus on:<br/>- Application testing<br/>- Profile refinement<br/>- Iterative validation]
    
    AppChange -->|Configuration Change| Feature
    
    Denials -->|Ongoing/Multiple| Pattern[Use Phase 3:<br/>Auto-Finetuning<br/><br/>Focus on:<br/>- Denial analysis<br/>- Pattern identification<br/>- Profile optimization]
    
    Denials -->|One-time/Rare| Monitor[Continue Monitoring<br/>Document for review<br/>No immediate action]
    
    style Start fill:#e1f5ff,stroke:#01579b,stroke-width:3px
    style RegularUpdate fill:#e8f5e9,stroke:#2e7d32,stroke-width:3px
    style Emergency fill:#fff3e0,stroke:#e65100,stroke-width:3px
    style Feature fill:#f3e5f5,stroke:#4a148c,stroke-width:3px
    style Monitor fill:#fff9c4,stroke:#f57f17,stroke-width:2px
```

---

## End-to-End Timeline Visualization

```mermaid
gantt
    title AppArmor Profile Management - Regular Update Timeline
    dateFormat  HH:mm
    axisFormat %H:%M
    
    section Phase 0 (One-Time)
    Setup Control Panel           :done, setup, 00:00, 30m
    Deploy Infrastructure         :done, deploy, after setup, 30m
    Initial Production Baselines  :done, initbase, after deploy, 20m
    Deploy Initial Profiles       :done, initprof, after initbase, 20m
    Configure DEV Machine         :done, devsetup, after initprof, 30m
    
    section Phase 1 (Regular Update)
    Pre-Update Baseline DEV       :active, prebase, 00:00, 10m
    OS Update on DEV              :osupdate, after prebase, 30m
    Post-Update Baseline          :postbase, after osupdate, 10m
    Detect Changes                :detect, after postbase, 5m
    Generate Suggestions          :suggest, after detect, 5m
    Apply to Profile              :apply, after suggest, 5m
    Enable aa-audit Mode          :audit, after apply, 2m
    Run Functional Tests          :test, after audit, 15m
    Disable aa-audit Mode         :enforce, after test, 2m
    Generate Approval Doc         :approval, after enforce, 5m
    Human Review                  :crit, review, after approval, 1h
    Deploy to Production          :prod, after review, 15m
    OS Update Production (Serial) :produpdate, after prod, 2h
    Monitor Production            :monitor, after produpdate, 24h
    
    section Phase 2 (Emergency - Parallel Path)
    Emergency Assessment          :emerg1, 00:00, 15m
    Fast DEV Testing              :emerg2, after emerg1, 30m
    Quick Approval                :crit, emerg3, after emerg2, 15m
    Canary Deployment             :emerg4, after emerg3, 30m
    Rolling Emergency Update      :emerg5, after emerg4, 1h
    Intensive Monitoring          :crit, emerg6, after emerg5, 2h
```


## Summary Checklist

### Zero-Day Deployment
- [ ] Control panel set up with Ansible
- [ ] Inventory file configured
- [ ] SSH access to all machines verified
- [ ] Monitoring infrastructure deployed (`apparmor-deploy.yml`)
- [ ] Initial baselines created on production
- [ ] Initial profiles deployed to production
- [ ] DEV machine synced with production state

### Regular Update Workflow
- [ ] Pre-update baseline created on DEV
- [ ] OS update performed on DEV
- [ ] Fine-tuning completed on DEV (`fine-tune-profile-dev.yml`)
- [ ] DEV tests passed
- [ ] Approval document generated and reviewed
- [ ] Human approval obtained
- [ ] Profile deployed to production
- [ ] OS update performed on production
- [ ] Post-update monitoring completed (24-72 hours)
- [ ] Final baseline created

### Ad-Hoc Update Workflow
- [ ] Emergency assessed and justified
- [ ] DEV fast-track testing completed
- [ ] Profile deployed to production
- [ ] Emergency update applied to production (rolling)
- [ ] Intensive monitoring for first 1-2 hours
- [ ] Continued monitoring for 24 hours

### Auto-Finetuning
- [ ] Continuous monitoring configured (daily cron)
- [ ] Denials analyzed and categorized
- [ ] Change replicated on DEV
- [ ] Iterative testing completed
- [ ] Refined profile deployed to production
- [ ] Monitoring period completed

---

**End of Document**

For questions or issues not covered in this document, consult:
- `docs/WORK_SUMMARY.md` - Technical implementation details
- `docs/WORK_NEXT_STEPS.md` - Future enhancements
- `scripts/README.md` - Script-specific documentation
