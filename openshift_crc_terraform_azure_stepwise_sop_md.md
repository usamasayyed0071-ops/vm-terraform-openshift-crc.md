# OpenShift Local (CRC) on Azure VM using Terraform - Step-by-Step SOP

## Objective
This SOP explains how to:

1. Create an Azure VM using Terraform
2. Install OpenShift Local (CRC)
3. Configure OpenShift CLI (`oc`)
4. Deploy applications on OpenShift
5. Expose applications using Routes
6. Scale applications
7. Create multiple OpenShift users
8. Verify cluster access and connectivity

---

# Architecture Overview

## Components Used

- Azure Virtual Machine (Ubuntu 22.04 LTS)
- Terraform
- OpenShift Local (CRC)
- OpenShift CLI (`oc`)
- Azure Networking Components
  - Virtual Network
  - Subnet
  - Public IP
  - Network Security Group

---

# Prerequisites

## Required Tools

### On Local Windows Machine

- Terraform
- Azure CLI
- Git Bash or PowerShell
- SSH Key Pair

### Azure Requirements

- Azure Subscription
- Contributor Access

### OpenShift Requirements

- Red Hat Account
- Pull Secret from Red Hat

---

# Step 1 - Install Terraform on Windows

## Verify Terraform Installation

```powershell
terraform -v
```

Expected Output:

```powershell
Terraform v1.14.9
```

---

# Step 2 - Create SSH Key Pair

Run:

```powershell
ssh-keygen
```

SSH Public Key Path:

```powershell
C:\Users\HP\.ssh\id_rsa.pub
```

---

# Step 3 - Create Terraform Project Structure

## Recommended Folder Structure

```text
terraform-openshift/
│
├── main.tf
├── provider.tf
├── variables.tf
├── output.tf
└── terraform.tfstate
```

---

# Step 4 - Terraform Configuration Files

# provider.tf

```hcl
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 4.0"
    }
  }
}

provider "azurerm" {
  features {}
}
```

---

# variables.tf

```hcl
variable "location" {
  default = "Central India"
}

variable "vm_size" {
  default = "Standard_D4s_v3"
}

variable "admin_username" {
  default = "azureuser"
}
```

---

# output.tf

```hcl
output "public_ip" {
  value = azurerm_public_ip.pip.ip_address
}
```

---

# main.tf

```hcl
resource "azurerm_resource_group" "rg" {
  name     = "openshift-rg"
  location = var.location
}

resource "azurerm_virtual_network" "vnet" {
  name                = "openshift-vnet"
  address_space       = ["10.0.0.0/16"]
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
}

resource "azurerm_subnet" "subnet" {
  name                 = "openshift-subnet"
  resource_group_name  = azurerm_resource_group.rg.name
  virtual_network_name = azurerm_virtual_network.vnet.name
  address_prefixes     = ["10.0.1.0/24"]
}

resource "azurerm_public_ip" "pip" {
  name                = "openshift-public-ip"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  allocation_method   = "Static"
}

resource "azurerm_network_security_group" "nsg" {
  name                = "openshift-nsg"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name

  security_rule {
    name                       = "SSH"
    priority                   = 1001
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "22"
    source_address_prefix      = "*"
    destination_address_prefix = "*"
  }

  security_rule {
    name                       = "OpenShift"
    priority                   = 1002
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_ranges    = ["6443", "80", "443"]
    source_address_prefix      = "*"
    destination_address_prefix = "*"
  }
}

resource "azurerm_network_interface" "nic" {
  name                = "openshift-nic"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name

  ip_configuration {
    name                          = "internal"
    subnet_id                     = azurerm_subnet.subnet.id
    private_ip_address_allocation = "Dynamic"
    public_ip_address_id          = azurerm_public_ip.pip.id
  }
}

resource "azurerm_network_interface_security_group_association" "nsgconnect" {
  network_interface_id      = azurerm_network_interface.nic.id
  network_security_group_id = azurerm_network_security_group.nsg.id
}

resource "azurerm_linux_virtual_machine" "vm" {
  name                = "openshift-vm"
  resource_group_name = azurerm_resource_group.rg.name
  location            = azurerm_resource_group.rg.location
  size                = var.vm_size
  admin_username      = var.admin_username

  disable_password_authentication = true

  network_interface_ids = [
    azurerm_network_interface.nic.id,
  ]

  admin_ssh_key {
    username   = "azureuser"
    public_key = file("C:/Users/HP/.ssh/id_rsa.pub")
  }

  os_disk {
    caching              = "ReadWrite"
    storage_account_type = "Standard_LRS"
    disk_size_gb         = 80
  }

  source_image_reference {
    publisher = "Canonical"
    offer     = "0001-com-ubuntu-server-jammy"
    sku       = "22_04-lts"
    version   = "latest"
  }

  computer_name = "openshiftvm"
}
```

---

# Step 5 - Initialize Terraform

```powershell
terraform init
```

---

# Step 6 - Validate Terraform Files

```powershell
terraform validate
```

---

# Step 7 - Create Azure Infrastructure

```powershell
terraform apply
```

Type:

```powershell
yes
```

---

# Step 8 - Verify Terraform Output

## Get Public IP

```powershell
terraform output
```

Example:

```powershell
public_ip = "74.225.196.66"
```

---

# Step 9 - Connect to Azure VM

```powershell
ssh azureuser@74.225.196.66
```

---

# Step 10 - Install CRC Prerequisites

## Update Packages

```bash
sudo apt update -y
```

## Install Required Packages

```bash
sudo apt install -y qemu-kvm libvirt-daemon-system libvirt-clients network-manager
```

## Enable Services

```bash
sudo systemctl enable --now libvirtd
sudo systemctl enable --now NetworkManager
```

## Add User to Required Groups

```bash
sudo usermod -aG libvirt $USER
sudo usermod -aG kvm $USER
```

## Logout

```bash
exit
```

Reconnect again using SSH.

---

# Step 11 - Download CRC

```bash
wget https://developers.redhat.com/content-gateway/file/pub/openshift-v4/clients/crc/latest/crc-linux-amd64.tar.xz
```

---

# Step 12 - Extract CRC

```bash
tar -xvf crc-linux-amd64.tar.xz
```

---

# Step 13 - Move CRC Binary

```bash
sudo mv crc-linux-2.60.1-amd64/crc /usr/local/bin/
```

---

# Step 14 - Verify CRC Installation

```bash
crc version
```

---

# Step 15 - Setup CRC

```bash
crc setup
```

---

# Step 16 - Start OpenShift Cluster

```bash
crc start
```

During startup:

- Paste Red Hat Pull Secret
- Wait for cluster stabilization

---

# Step 17 - OpenShift Access Details

Example:

```text
Web Console:
https://console-openshift-console.apps-crc.testing

Developer User:
username: developer
password: developer

Admin User:
username: kubeadmin
password: <generated-password>
```

---

# Step 18 - Configure OC CLI

## Linux

```bash
eval $(crc oc-env)
```

## Verify

```bash
which oc
oc version
```

---

# Step 19 - Login to OpenShift

## Developer Login

```bash
oc login -u developer -p developer https://api.crc.testing:6443 --insecure-skip-tls-verify
```

## Admin Login

```bash
oc login -u kubeadmin -p <password> https://api.crc.testing:6443 --insecure-skip-tls-verify
```

---

# Step 20 - Verify Cluster Nodes

```bash
oc get nodes
```

---

# Step 21 - Create Project

```bash
oc new-project demo
```

---

# Step 22 - Deploy Initial NGINX App

```bash
oc new-app nginx
```

---

# Step 23 - Check Pods

```bash
oc get pods
```

---

# Step 24 - Troubleshoot CrashLoopBackOff

## View Logs

```bash
oc logs nginx-7dc54b55ff-ztnmz
```

## Describe Pod

```bash
oc describe pod nginx-7dc54b55ff-ztnmz
```

Issue:

- The default image expected Source-to-Image (S2I)
- Container entered CrashLoopBackOff

---

# Step 25 - Delete Failed Deployment

```bash
oc delete deployment nginx
```

---

# Step 26 - Deploy Working NGINX Image

```bash
oc new-app nginxinc/nginx-unprivileged
```

---

# Step 27 - Verify Running Pods

```bash
oc get pods
```

Expected:

```text
1/1 Running
```

---

# Step 28 - Expose Application

```bash
oc expose svc/nginx-unprivileged
```

---

# Step 29 - Verify Routes

```bash
oc get route
```

Example Route:

```text
http://nginx-unprivileged-demo.apps-crc.testing
```

---

# Step 30 - Test Application

## Using Curl

```bash
curl http://nginx-unprivileged-demo.apps-crc.testing
```

## Using Browser

Open:

```text
http://nginx-unprivileged-demo.apps-crc.testing
```

---

# Step 31 - Scale Application

```bash
oc scale deployment nginx-unprivileged --replicas=3
```

---

# Step 32 - Verify Scaling

```bash
oc get pods
```

Expected:

```text
3 Running Pods
```

---

# Step 33 - Check OpenShift Status

```bash
oc status
```

---

# Step 34 - View Services

```bash
oc get svc
```

---

# Step 35 - View Routes

```bash
oc get routes
```

---

# Step 36 - Get CRC IP

```bash
crc ip
```

---

# Step 37 - Install htpasswd Utility

```bash
sudo apt update
sudo apt install apache2-utils -y
```

---

# Step 38 - Create User Authentication File

```bash
touch users.htpasswd
```

---

# Step 39 - Create 30 OpenShift Users

```bash
for i in {1..30}; do htpasswd -bB users.htpasswd user$i Password@123; done
```

---

# Step 40 - Verify Users File

```bash
cat users.htpasswd
```

---

# Important Troubleshooting Notes

## Error: `Started: command not found`

Reason:

You copied output text directly into terminal.

Correct:

Only run actual commands.

---

## Error: `eval is not recognized`

Reason:

`eval $(crc oc-env)` works only on Linux/macOS.

Windows Alternative:

Use:

```powershell
oc version
```

or manually add `oc.exe` to PATH.

---

## Error: `oc: command not found`

Fix:

```bash
eval $(crc oc-env)
```

or

```bash
export PATH=$PATH:/path/to/oc
```

---

## Error: `Forbidden: cannot list nodes`

Reason:

Developer user lacks cluster-admin permissions.

Fix:

Login as kubeadmin:

```bash
oc login -u kubeadmin -p <password> https://api.crc.testing:6443 --insecure-skip-tls-verify
```

---

# Important OpenShift Commands

## Login

```bash
oc login
```

## Current User

```bash
oc whoami
```

## Get Pods

```bash
oc get pods
```

## Get Nodes

```bash
oc get nodes
```

## Get Services

```bash
oc get svc
```

## Get Routes

```bash
oc get routes
```

## Describe Pod

```bash
oc describe pod <pod-name>
```

## View Logs

```bash
oc logs <pod-name>
```

---

# Terraform State Overview

## Infrastructure Created

- Resource Group
- Virtual Network
- Subnet
- Public IP
- Network Security Group
- Network Interface
- Ubuntu Linux VM

## VM Configuration

| Component | Value |
|---|---|
| VM Name | openshift-vm |
| OS | Ubuntu 22.04 LTS |
| VM Size | Standard_D4s_v3 |
| Disk Size | 80 GB |
| Location | Central India |

---

# Final Validation Checklist

| Task | Status |
|---|---|
| Terraform Installed | Completed |
| Azure VM Created | Completed |
| CRC Installed | Completed |
| OpenShift Cluster Running | Completed |
| OC CLI Working | Completed |
| Project Created | Completed |
| Application Deployed | Completed |
| Route Exposed | Completed |
| Scaling Tested | Completed |
| Multiple Users Created | Completed |

---

# Useful Commands Reference

## CRC Commands

```bash
crc setup
crc start
crc stop
crc delete
crc status
crc ip
```

---

## Terraform Commands

```powershell
terraform init
terraform validate
terraform plan
terraform apply
terraform destroy
terraform output
```

---

# Conclusion

This SOP successfully demonstrates:

- Azure infrastructure provisioning using Terraform
- OpenShift Local installation using CRC
- OpenShift application deployment
- Route exposure and testing
- Scaling applications
- Creating multiple users using htpasswd
- Troubleshooting common CRC/OpenShift issues

This setup can be used for:

- OpenShift Practice Labs
- Kubernetes Learning
- DevOps Training
- CI/CD Testing
- OpenShift Administration Practice

