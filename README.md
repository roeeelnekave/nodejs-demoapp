### Note: The nodejs app is taken from [benc-uk](https://github.com/benc-uk/nodejs-demoapp.git) github

# Prerequisties

- Nodejs
- docker
- terraform
- ansible
- azure-cli

### Setup the Infra

- Fork the Repository https://github.com/benc-uk/nodejs-demoapp.git 
    - Click on the URL
    - Click on the **Fork** button
    - Give a **repository-name** like `nodejs-demoapp`
    - Click on **Create fork**

- After that Clone the repository to your pc

- Open the project 

## Setup the folder structure

- **Run the following to setup folder**
```bash
mkdir -p ./terraform
mkdir -p ./ansible/roles
cd ./ansible/roles
ansible-galaxy init jenkins
ansible-galaxy init jenkins-agent
```

### Terraform 

- We have set provider to do it create `./terraform/providers.tf` Copy and paste following in it
```hcl
terraform {
  required_version = ">=0.12"

  required_providers {
    azapi = {
      source  = "azure/azapi"
      version = "~>1.5"
    }
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~>2.0"
    }
    random = {
      source  = "hashicorp/random"
      version = "~>3.0"
    }
  }
}

provider "azurerm" {
  features {}
}

```
- Create a template to launch the vm in terraform `./terraform/main.tf` Copy and paste the following in it
```hcl
resource "random_pet" "rg_name" {
  prefix = var.resource_group_name_prefix
}

resource "azurerm_resource_group" "rg" {
  location = var.resource_group_location
  name     = random_pet.rg_name.id
}

# Create virtual network
resource "azurerm_virtual_network" "my_terraform_network" {
  name                = "myVnet"
  address_space       = ["10.0.0.0/16"]
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
}

# Create subnet
resource "azurerm_subnet" "my_terraform_subnet" {
  name                 = "mySubnet"
  resource_group_name  = azurerm_resource_group.rg.name
  virtual_network_name = azurerm_virtual_network.my_terraform_network.name
  address_prefixes     = ["10.0.1.0/24"]
}

# Create public IPs
# Create public IPs for each VM
resource "azurerm_public_ip" "my_terraform_public_ip" {
  count               = 3
  name                = "myPublicIP-${count.index}"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  allocation_method   = "Dynamic"
}


# Create Network Security Group and rule
resource "azurerm_network_security_group" "my_terraform_nsg" {
  name                = "myNetworkSecurityGroup"
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
    name                       = "sonarqube01"
    priority                   = 1002
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "9000"
    source_address_prefix      = "*"
    destination_address_prefix = "*"
  }
  security_rule {
    name                       = "sonarqube1"
    priority                   = 1003
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "9001"
    source_address_prefix      = "*"
    destination_address_prefix = "*"
  }
  security_rule {
    name                       = "jenkins"
    priority                   = 1004
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "8080"
    source_address_prefix      = "*"
    destination_address_prefix = "*"
  }
  security_rule {
    name                       = "http"
    priority                   = 1005
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "80"
    source_address_prefix      = "*"
    destination_address_prefix = "*"
  }
}

# Create network interface
# Create network interfaces for each VM
# Create network interfaces for each VM
resource "azurerm_network_interface" "my_terraform_nic" {
  count               = 3
  name                = "myNIC-${count.index}"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name

  ip_configuration {
    name                          = "my_nic_configuration"
    subnet_id                     = azurerm_subnet.my_terraform_subnet.id
    private_ip_address_allocation = "Dynamic"
    public_ip_address_id          = azurerm_public_ip.my_terraform_public_ip[count.index].id
  }
}



# Connect the security group to each network interface
resource "azurerm_network_interface_security_group_association" "example" {
  count                    = 3
  network_interface_id      = azurerm_network_interface.my_terraform_nic[count.index].id
  network_security_group_id = azurerm_network_security_group.my_terraform_nsg.id
}


# Generate random text for a unique storage account name
resource "random_id" "random_id" {
  keepers = {
    # Generate a new ID only when a new resource group is defined
    resource_group = azurerm_resource_group.rg.name
  }

  byte_length = 8
}

# Create storage account for boot diagnostics
resource "azurerm_storage_account" "my_storage_account" {
  name                     = "diag${random_id.random_id.hex}"
  location                 = azurerm_resource_group.rg.location
  resource_group_name      = azurerm_resource_group.rg.name
  account_tier             = "Standard"
  account_replication_type = "LRS"
}

# Create virtual machine
resource "azurerm_linux_virtual_machine" "my_terraform_vm" {
  count                 = 3
  name                  = var.vm_names[count.index]  
  location              = azurerm_resource_group.rg.location
  resource_group_name   = azurerm_resource_group.rg.name
  network_interface_ids = [azurerm_network_interface.my_terraform_nic[count.index].id]
  size                  = "Standard_DC1ds_v3"

  os_disk {
    name                 = "${var.vm_names[count.index]}-os-disk"
    caching              = "ReadWrite"
    storage_account_type = "Premium_LRS"
  }

  source_image_reference {
    publisher = "Canonical"
    offer     = "0001-com-ubuntu-server-jammy"
    sku       = "22_04-lts-gen2"
    version   = "latest"
  }

  computer_name  = "hostname"
  admin_username = var.username

  admin_ssh_key {
    username   = var.username
    public_key = azapi_resource_action.ssh_public_key_gen.output.publicKey
  }

  boot_diagnostics {
    storage_account_uri = azurerm_storage_account.my_storage_account.primary_blob_endpoint
  }
}

output "private_key" {
  value = azapi_resource_action.ssh_public_key_gen.output.privateKey
}
```

- Create output file `./terraform/outputs.tf`, copy and paste the following in it 
```hcl
output "resource_group_name" {
  value = azurerm_resource_group.rg.name
}

output "public_ip_addresses" {
  value = [
    for vm in azurerm_linux_virtual_machine.my_terraform_vm : vm.public_ip_address
  ]
}
```

- To create a ssh keys create a file `./terraform/ssh.tf` , copy and paste the following in it
```hcl
resource "random_pet" "ssh_key_name" {
  prefix    = "ssh"
  separator = ""
}

resource "azapi_resource_action" "ssh_public_key_gen" {
  type        = "Microsoft.Compute/sshPublicKeys@2022-11-01"
  resource_id = azapi_resource.ssh_public_key.id
  action      = "generateKeyPair"
  method      = "POST"

  response_export_values = ["publicKey", "privateKey"]
}

resource "azapi_resource" "ssh_public_key" {
  type      = "Microsoft.Compute/sshPublicKeys@2022-11-01"
  name      = random_pet.ssh_key_name.id
  location  = azurerm_resource_group.rg.location
  parent_id = azurerm_resource_group.rg.id
}

output "key_data" {
  value = azapi_resource_action.ssh_public_key_gen.output.publicKey
}

```

- Run the following to launch your infrastructure
```bash
cd ./terraform
terraform init
terraform apply --auto-approve
cd ..
```

## Ansible

- Run the following to get public ip address and private key of your instances
```bash
cd ./terraform
terraform output public_ip_addresses
terraform output private_key | awk '/-----BEGIN RSA PRIVATE KEY-----/,/-----END RSA PRIVATE KEY-----/' > ../ansible/machine.pem
cd ..
```


- Create a inventory file `./ansible/inventory.yaml` and  copy and replace the ip with the ip in the output when you ran above script
```yaml
jenkins:
  hosts:
  # replace one of the ip here
    13.82.102.158:
      ansible_user: azureadmin
      ansible_connection: ssh
agent:
# replace one of the ip here
  hosts:
    13.82.102.121:
      ansible_user: azureadmin
      ansible_connection: ssh
```
- Create a `./ansible/ansible.cfg` to disable host checking and paste the following in it
```yaml
[defaults]
host_key_checking = False
```
- Create a playbook file to run the  configuration `./ansible/main.yaml`

```yaml
- hosts: jenkins 
  become: true
  tasks: 
    - name: install jenkins
      include_role:
        name: jenkins

- hosts: agent
  become: true
  tasks: 
    - name: install jenkins-agent dependency
      include_role:
        name: agent
```
- Now we create a configuration to install and grab  initial admin password for jenkins `./ansible/roles/jenkins/tasks/main.yml` copy the following into it
```yaml
---
# tasks file for jenkins
- name: update cache
  apt:
   update_cache: yes

- name: install java
  apt:
   name: openjdk-17-jdk
   state: latest    

- name: Download Jenkins keyring
  ansible.builtin.get_url:
    url: https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
    dest: /usr/share/keyrings/jenkins-keyring.asc

- name: Add Jenkins repository
  ansible.builtin.apt_repository:
    repo: "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/"
    filename: jenkins
    state: present

- name: Update apt cache
  ansible.builtin.apt:
    update_cache: yes

- name: Install Jenkins
  ansible.builtin.apt:
    name: jenkins
    state: present

- name: Fetch Jenkins initialAdminPassword
  fetch:
    src: /var/lib/jenkins/secrets/initialAdminPassword
    dest: ./initialAdminPassword
    flat: yes
  
```

- Now we create the launch configuration for the jenkins agent `./ansible/roles/agent/tasks/main.yml` copy the following in it
```yaml
# tasks file for agent

- name: update cache
  apt:
    update_cache: yes

- name: install java
  apt:
    name: openjdk-17-jdk
    state: latest    

- name: install git
  apt:
    name: git 
    state: latest

- name: Download NodeSource setup script
  get_url:
    url: https://deb.nodesource.com/setup_20.x
    dest: /tmp/nodesource_setup.sh
    mode: '0755'

- name: Run NodeSource setup script
  command: bash /tmp/nodesource_setup.sh
  become: true

- name: Update APT cache
  apt:
    update_cache: yes

- name: Install Node.js
  apt:
    name: nodejs
    state: present

- name: Verify Node.js installation
  command: node -v
  register: node_version

- name: Print Node.js version
  debug:
    msg: "Node.js version is {{ node_version.stdout }}"

```

- Now let's run our configuration run the following
```bash
cd ./ansible
chmod 400 machine.key
ansible-playbook -i inventory.yaml --private-key machine.key main.yaml
echo "##################Inital Admin Password #################"
cat initialAdminPassword
cd ..
```
- Access jenkins through the ip `13.82.102.158:8080` **replace the ip to your ip:8080** you have given for jenkins in inventory copy the intial admin password from the terminal or do a `cat ./ansible/initialAdminPassword` 

- Click on **Install suggested plugins** then wait for sometime to get it installed the jenkins after that enter the details and click on **Save and continue** then click on **Save and Finish** then Click on **Start Using Jenkins**

- Click on **Manage Jenkins**
- CLick on **Plugins** click on **Available Plugins**
- On search bar type `sonarqube` select `sonar scanner` ,`Sonar Quality Gates`,`CodeSonar` and click **Install**  then click on **Restart Jenkins when installation is complete and no jobs are running** check box

### Now configure your Jenkins Agent to run the pipeline 

1. Access your Jenkins server 
2. Click on **Manage Jenkins**.
3. Click on **Nodes**.
4. Click on **New Nodes**.
5. Give it a name as `sonar` and Check the Type `Permanent Agent` then click on **Create**.
6. Add **Remote root directory** as `/home/azureadmin`.
7. Add **Labels** as `sonar`.
8. On **Launch method** select `Launch agents via ssh`
9. On **Host** give the ip the agent server.
10. On **Credentials**  click on `Add` button select `Jenkins`.
11. On **Kind** select as `SSH username with private key`
12. On **ID** give it a unique name like `sonaragent`
13. On **Description** give it a Description like `agent for sonar and jenkins pipeline`.
14. On **Username** give it the server username in our case it's `azureadmin`
15. On **Private Key** section select `Enter Directly` under key click `Add` and copy the contents of the keypair `machine.pem` and paste it there then click on **Add** 
16. Again in **Credentials** select the `id` of credentails that you have just created.
17. Under **Host Key Verification Strategy** select `Non verifying verification strategy`
18. Click on **Save**

## Now Let's configure sonarqube for jenkins


- **Open a terminal** and run the SSH command using your key pair file replace the `192.168.0.1` with actual ip of your sonarqube vm:

```bash
ssh -i ./ansible/machine.pem azureadmin@192.168.0.1
```
- Once you are connected run the following to install docker and docker compose
```bash
sudo apt update
sudo apt install docker.io -y
sudo apt update
sudo apt install docker-compose -y
sudo usermod -aG docker $USER
sudo systemctl start docker
sudo systemctl enable docker
exit
```

- Again ssh to sonarqube vm
```bash
ssh -i ./ansible/machine.pem azureadmin@192.168.0.1
```
- Then create a `nano ./docker-compose.yaml` and paste the following in it 
```yaml
version: '3.8'

services:
  sonarqube:
    image: sonarqube:community
    container_name: sonarqube
    ports:
      - "9000:9000"
    environment:
      SONAR_JDBC_URL: jdbc:postgresql://db:5432/sonarqube
      SONAR_JDBC_USERNAME: sonarqube
      SONAR_JDBC_PASSWORD: sonarqube
    depends_on:
      - db
    volumes:
      - sonarqube_conf:/opt/sonarqube/conf
      - sonarqube_data:/opt/sonarqube/data
      - sonarqube_logs:/opt/sonarqube/logs
      - sonarqube_extensions:/opt/sonarqube/extensions

  db:
    image: postgres:14
    container_name: sonarqube_db
    environment:
      POSTGRES_USER: sonarqube
      POSTGRES_PASSWORD: sonarqube
      POSTGRES_DB: sonarqube
    volumes:
      - postgresql_data:/var/lib/postgresql/data

volumes:
  sonarqube_conf:
  sonarqube_data:
  sonarqube_logs:
  sonarqube_extensions:
  postgresql_data:

``` 
- Press `ctrl+x` then click `y` and hit **Enter**
- After that run the following
```bash
docker-compose up -d
```
- Once the build is completed verify if sonarqube is running or not
```
docker ps
```
- Now Access your sonarqube through ip of your sonarqube vm with port `9000` like `192.168.0.1:9000` replace the ip with your vm ip

- Login using username and password as `admin` and `admin` change the password
- Again Login with your credentails
- Click on **Profile Icon** then click on **My Account** then Click on **Security** 
- Under **Generate Token**  Give it a **Name** like `jenkissonar` Type **Global Analysis Token** Expires in **No expiration** for now click **Generate** Copy  the token and put it on note 

#### Sonarqube integration with jenkins
1. Go jenkins Dashboard
2. Click on **Manage Jenkins** then click on **Tools**
3. Scroll down until you find **SonarQube Scanner installations** then Under **SonarQube Scanner installations** click on **Add SOnarQube Scanner** give it a name **SonarScanner** leave to default click on **Apply** then Click on **Save**
4. Again Go to jenkins Dashboard
5. Click on **Manage Jenkins** then click on **Credentials** 
6. Under **Credentials** click on **System**
7. Click on **Global credentials (unrestricted)**
8. Click on **+ Add Credentials**
9. Kind **Secret Text** in **secret** paste your *token* `sqa_XXXXXXXXXXXXXX` give it a **Id** like `sonarjenkins`  then **Description** like `Azure VM sonarqube` click **Create**
5. Click on **Manage Jenkins** then click on **System** 
6. Scroll Down Until you find **SonarQube servers** then check **Environment Variables**
7. Under **SonarQube Installations** click that **Add** Button 
8. Give it a name like **SonarJenkins**
9. In server url give your sonarqube server url `http://192.168.0.1:9000` replace `192.168.0.1` with the sonar ip
10. Under **Server authentication token** select the credentials that you have just created click **Apply** and then **Save**

### Configure jenkins pipeline

- Create a jenkinsfile for pipeline script `./Jenkinsfile` and paste the following in it
```groovy
node('sonar') {
  stage('SCM') {
    checkout scm
  }
  stage('Initialize'){
    dir('src') {
      sh "npm install"
    }
  }
  stage('test'){
    dir('src') {
      sh "npm run test-report"
      sh "npm run test"
      def postmanResult = sh(script: "npm run test-postman", returnStatus: true)
      if (postmanResult != 0) {
        echo "Postman tests failed with status ${postmanResult}. Continuing the build..."
      }
    }
  }
  stage('fix'){
    dir('src') {
      sh "npm run lint"
      sh "npm run lint-fix"
    }
  }
  stage('SonarQube Analysis') {
    def scannerHome = tool 'SonarScanner'
    withSonarQubeEnv() {
      sh "${scannerHome}/bin/sonar-scanner"
    }
  }
    stage('deploy'){
    dir('src') {
      sh "npm run start-bg"
   
    }
  }
}

```
Then also create a `./sonar-project.properties` copy and paste the following 
```bash
sonar.projectKey=node-app
sonar.sources=.
sonar.language=js
sonar.javascript.lcov.reportPaths=coverage/lcov.info
sonar.inclusions=**/*.js,**/*.dockerfile,**/*.yml,**/*.yaml
```
- Go to sonarqube server click a **Create a Local Project** then in **Project display name** give it a name `node-app` then in **Main branch name** give `master` then click **Next** and click on **Use the global setting** then click on **Create project**

Now run the following to push on github
```bash
git add .
git commit -m "added the files"
git push
```


## **Create a Pipeline Job**
**Create a Pipeline in order to automatically analyze your project.**

- From Jenkins' **dashboard**, click New Item and create a **Pipeline Job.**
- Under **Build Triggers**, choose Trigger builds remotely. You must set a unique, secret token for this field.(Optional)
- Under **Pipeline**, make sure the parameters are set as follows:
- **Definition: Pipeline script from SCM**
- **SCM**: Configure your SCM. Make sure to only build your main branch. For example, if your main branch is called **"master"**, put **"*/master"** under Branches to build.
- Script Path: Jenkinsfile
- Click **Save**.

## Click on `Builld Now` wait until pipeline become success