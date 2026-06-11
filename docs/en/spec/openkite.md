
# 1. Preface

## 1.1 Software License Agreement

This software is based on LGPL V3 protocol. Please pay attention to the requirements of the LGPL protocol. The most important point is that **forking the project for closed source is not allowed**.

## 1.2 Software Purpose

## 1.3 Developer List

## 1.4 Development Lifecycle

## 1.5 Feature Development Order

# 2. Development Standards and Conventions

## 2.1 Window Control Naming Convention

- Control original name_Window_Control name combination with first letter capitalized
- Example:

  ```cpp
  Button original name: pushButton Main Window Menu Button
  Naming convention: pushButton_MainWindow_Menu

  Button original name: toolButton Main Window Upload Button
  Naming convention: toolButton_MainWindow_UpLoad
  ```

## 2.2 Backend Function Implementation Naming Convention

- Variables, constants, functions, classes, containers, etc.

## 2.3 Software Package Filename Naming Convention

## 2.4 File Naming Convention

## 2.5 Annotations

- Delete, move, rename, permission settings

# 3. Main Window Control Names, Sizes, and Purposes

## 3.1 Menu Function Categories

- PushButton controls are used for menu category window calls
- Control sizes:
    - Fixed size 80*25

| Control Name | Control Type | Control Name | Purpose |
|-----------|-----------|-------------------------------------|----------------|
| Menu | PushButton | pushButton_MainWindow_Menu | Call menu window |
| Help | PushButton | pushButton_MainWindow_Help | Call help window |
| Tools | PushButton | pushButton_MainWindow_Tool | Call tools window |
| Error Analysis | PushButton | pushButton_MainWindow_ErrorAnalysis | Call error analysis window |
| Monitor | PushButton | pushButton_MainWindow_Monitor | Call monitor window |
| Operation Log | PushButton | pushButton_MainWindow_OperationLog | Call operation log window |

### 3.1.1 Menu Subcategories

- Settings
- Software Theme

### 3.1.2 Help Categories

- Community
- Version Update
- User Manual

### 3.1.3 Tool Categories

- Plugin Repository
- IMG Image Tool
- MD5 Verification Tool
- OpenStack Module Function Test
- Stress Test

### 3.1.4 Error Analysis Categories

- System Errors (Node Error Analysis)
- OpenStack Errors
- K8S Errors

### 3.1.5 Monitor Categories

- OPS Monitor Status and Performance Usage Analysis
- K8S Monitor Status and Performance Usage Analysis

### 3.1.6 Operation Log Categories

- View Historical Operation Logs
- Export Logs

## 3.2 Data Visualization Categories

### 3.2.1 Computer Hardware Information Category

- ProgressBar controls display computer hardware performance usage
- Control sizes:
    - Minimum size 116*27
    - Height fixed

| Control Name | Control Type | Control Name | Purpose |
|-----------|-----------|--------------------------------------|-----------------------|
| Local CPU | ProgressBar | progressBar_MainWindow_LocalCPU | Display local CPU usage |
| Target CPU | ProgressBar | progressBar_MainWindow_TargetCPU | Display target CPU usage |
| Local RAM | ProgressBar | progressBar_MainWindow_LocalRAM | Display local RAM usage |
| Target RAM | ProgressBar | progressBar_MainWindow_TargetRAM | Display target RAM usage |
| Local Network | ProgressBar | progressBar_MainWindow_LocalNetwork | Display local network bandwidth usage |
| Target Network | ProgressBar | progressBar_MainWindow_TargetNetwork | Display target network bandwidth usage |
| Local Disk | ProgressBar | progressBar_MainWindow_LocalDisk | Display local disk IO usage |
| Target Disk | ProgressBar | progressBar_MainWindow_TargetDisk | Display target disk IO usage |

### 3.2.2 Computer Software Information Category

- Label controls display system IP and DNS
- Control sizes:
    - Fixed size 110*27

| Control Name | Control Type | Control Name | Purpose |
|-----------|-------|----------------------------|------------|
| Local IP | Label | label_MainWindow_LocalIP | Display local IP |
| Target IP | Label | label_MainWindow_TargetIP | Display target IP |
| Local DNS | Label | label_MainWindow_LocalNDS | Display local DNS |
| Target DNS | Label | label_MainWindow_TargetNDS | Display target DNS |

- ListWidget controls display system essential information items
- Control sizes:
    - Fixed size 200*111

| Control Name | Control Type | Control Name | Purpose |
|-------------|-------------|-----------------------------|----------------|
| System Info Display | ListWidgets | listWidget_MainWidow_SystemShow | Display system essential information |

- API interfaces for system essential information display variables

| Name | Variable Type | Variable Name | Purpose |
|---------|-------------|---------------------|---------------------|
| Distribution | QStringList | systemNameShow | Linux distribution name |
| Version Number | QStringList | systemVersion | Linux distribution version number |
| Kernel Version | QStringList | systemKernel | Linux distribution kernel version |
| Admin Rights | QStringList | systemAdminPower | Current account operation permissions |
| Service Name | QStringList | systemServiceName | Current operation software service name |
| Service Version | QStringList | systemServicVersion | Current operation software version |

- Label and ProgressBar controls display current running command and progress
- Control sizes:
    - Current running command control size:
        - Minimum size: 500*31
        - Height fixed
    - Current command progress control size:
        - Minimum size: 171*31
        - Height fixed

| Name | Control Type | Control Name | Purpose |
|------|-------|-----|----|
| Current Running Command | Label | label_MainWindow_ShowCurrentCommand | Display command currently running on cluster or node |
| Current Command Progress | ProgressBar | progressBar_MainWindow_ShowCommandProgress | Display progress of command currently running on cluster or node |

## 3.3 Add Cluster Category

### 3.3.1 Cluster Add Category

- ToolButton controls add cluster node information
- Control sizes:
    - Fixed size: 300*31

| Name | Control Type | Control Name | Purpose |
|------|-------|------|----|
| Add Cluster/Node | ToolButton | toolButton_MainWindow_AddNode | Pop up window to add cluster or node |

- Single node add
- Batch node add
- Cluster add

### 3.3.2 Cluster Display Category

- TreeWidget controls display cluster information
- Control sizes:
    - Minimum size: 200*438
    - Width fixed

| Name | Control Type | Control Name | Purpose |
|------|-------|------|---|
| Node Information | TreeWidget | treeWidget_MainWindow_ShowNode | Display cluster and node information or create SSH remote window interface after clicking information |

- Cluster name
- Node name
- Node IP address

### 3.4 Script and Deployment Category

- TerrWidget controls pop up windows
- Control sizes:
    - Upload, script button fixed size: 63*31
    - Deploy button fixed size: 65*31

| Name | Control Type | Control Name | Purpose |
|--------|------------|------------------------------|-----------------------|
| Upload | terrWidget | toolButton_MainWindow_UpLoad | Pop up upload window: load.ui |
| Script | terrWidget | toolButton_MainWindow_Shell | Pop up script window: shell.ui |
| Deploy | terrWidget | toolButton_MainWindow_Deploy | Pop up deployment window: deploy.ui |

### 3.4.1 Upload and Download Function Category

- Script compiler
    - YAML compiler
    - Script compiler
- Load local policies
    - Load cluster configuration policies
    - Load node configuration policies
- Upload files to target computer
    - Single node
    - Multiple nodes
- Download files to local computer
    - Single node
    - Multiple nodes
- File transfer between target computers
    - Point-to-point transfer
    - Point-to-multipoint transfer

### 3.4.2 Script Category

- Edit
    - Edit submodule scripts
    - Edit cluster module scripts
- View
    - View submodule scripts
    - View cluster module scripts
- Export
    - Export submodule scripts
    - Export cluster module scripts
    - Export all scripts

### 3.4.3 Deployment Category

- Deploy
    - Can batch select nodes to deploy different function scripts
    - Can deploy different nodes with different function scripts at cluster level
    - Can deploy different function scripts on single node
- Terminate
    - Can batch terminate multiple nodes, single node, cluster for current deployment

### 3.5 Function Plugin Category

### 3.5.1 Basic Operations Category

- Modify server computer name
- Modify server username
- Modify server password
- Modify firewall configuration
- Modify host
- Modify DNS
- Modify gateway
- Modify IP
- Deploy time service
- Deploy DNS service

### 3.5.2 Other Function Plugin Categories

- OpenStack plugin category
- K8S plugin category
- Ceph plugin category

## 3.6 SSH Remote Display Category

- Can copy and paste commands, Chinese display comprehensive port

### 3.6.1 Cluster SSH Remote Display Category

- Comprehensive port display, point-to-multipoint SSH remote

### 3.6.2 Single Node SSH Remote Display Category

- Point-to-point SSH remote

# 4. Main Window Function Plugin Addition Methods, Standards, API and Function Comments

## 4.1 Tool Category

- Development standard:
- API interface:
- Function comment:
- Panel addition method:
- Backend function module addition method:
- Folder location:

## 4.2 Function Plugin Category

- Development standard:
- API interface:
- Function comment:
- Panel addition method:
- Backend function module addition method:
- Folder location:

# 5. Backend API Call, Standards and Usage Instructions

## 5.1 Computer Hardware

### 5.1.1 CPU

### 5.1.2 RAM

## 5.2 Computer Software

### 5.2.1 Local Software Package

### 5.2.2 Source Software Package

# 6. Development Notes

- Before various operations, check whether local network and target network are connected
- When target network is unreachable, prompt: Target IP network unreachable
- When all cluster nodes are unreachable, cluster node font turns gray
- When cluster operation or multi-node operation fails, prompt unreachable target information and confirm whether to continue. If continued, batch deploy by blocking unreachable nodes
- Interface information refresh frequency
    - Software and hardware information refresh frequency
        - CPU, memory and other percentage display information refresh frequency is 0.5s
    - SSH interface screen refresh is real-time
    - Cluster display information is real-time
    - System essential information display area is real-time
