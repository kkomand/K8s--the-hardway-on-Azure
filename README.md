# K8s--the-hardway-on-Azure
Source : https://github.com/ivanfioravanti/kubernetes-the-hard-way-on-azure

Step 1 : Pre-Requisites
1a. Install Azure CLI on Windows (https://docs.microsoft.com/en-us/cli/azure/install-azure-cli-windows?tabs=azure-cli). Used MSI quick Installer
    Launch Windows Powershell in admin mode
    az login 
    az --version
    ![image](https://user-images.githubusercontent.com/56612071/187729539-6a6d5072-04b7-4904-a7bc-6d623752ceee.png)

1b. Create Resource Group
    - To view locations for the RG creation, use command az account list-locations
    - Create RG in EastUS2
      az group create -n kubernetes_eastus_rg -l eastus2
      
    - Login to Azure Portal (https://portal.azure.com), Click on Resource Group  and Verify RG kubernetes is created 

Step 2: Installing Windows Client Tools (Refer to source link above if you are using other OS)

 2a. Install CFSSL and CFSSLJSON 
     CSFSSL : Described as  CloudFlare's PKI/TLS swiss army knife. It is both a command line tool and an HTTP API server for signing, verifying, and bundling TLS 
     certificates.
     CSFSSLJSON : Program, which takes the JSON output from the cfssl and multirootca programs and writes certificates, keys, CSRs, and bundles to disk.
     
     - Lauch powershell as admin and execute 
       cd C:\Windows\system32
       Invoke-WebRequest -Uri https://github.com/cloudflare/cfssl/releases/download/v1.4.1/cfssl_1.4.1_windows_amd64.exe -OutFile cfssl.exe
       Invoke-WebRequest -Uri https://github.com/cloudflare/cfssl/releases/download/v1.4.1/cfssljson_1.4.1_windows_amd64.exe -OutFile cfssljson.exe
       
   2b. Install Chocolately package manager as a pre-req for installing kubectl on Windows. 
   
       - Execute below command in Powershell (https://docs.chocolatey.org/en-us/choco/setup)
         Set-ExecutionPolicy Bypass -Scope Process -Force; [System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072; iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))
       
       ![image](https://user-images.githubusercontent.com/56612071/187760649-d37a10a4-76a1-4e1c-ae34-8f586e75e35f.png)
       
       - Upgrade Chocolately using command below
         choco upgrade chocolatey
         
    2c. Install Kubectl using Choco Manager
    
        - Install Command
          choco install kubernetes-cli
          ![image](https://user-images.githubusercontent.com/56612071/187762031-e7b29c2d-c37b-4163-8641-885bdd3161fd.png)

        - Kubectl Version Check
          kubectl version --short
          
 Step 3: Provisioning Computing Resourcs
 
      3a. Network Setup - Cluster Networking. Regarding Pods, Container and Service Network setup. Refer to (https://kubernetes.io/docs/concepts/cluster-administration/networking/#kubernetes-model)
          Highly-coupled container-to-container communications: this is solved by Pods and localhost communications.
          Pod-to-Pod communications: this is the primary focus of this document.
          Pod-to-Service communications: this is covered by services.
          External-to-Service communications: this is covered by services.
          
          - Setup Virtual Networking (Creates Vnet and Subnet)
            az network vnet create -g kubernetes_eastus_rg -n kubernetes-vnet --address-prefix 10.240.0.0/24 --subnet-name kubernetes-subnet
            
          - Setup Firewalls (Network Service Group): NSG contains security rules that allow or deny inbound network traffic to, or outbound network traffic from,  
            several types of Azure resources. For each rule, you can specify source and destination, port, and protocol.
            
            az network nsg create -g kubernetes_eastus_rg -n kubernetes-nsg
            
        3b. Create Firewall rules
        
            - Creatre a firewall rule that allows external SSH
           
            az network nsg rule create -g kubernetes_eastus_rg -n kubernetes-allow-ssh --access allow --destination-address-prefix '*' --destination-port-range 22               --direction inbound --nsg-name kubernetes-nsg --protocol tcp --source-address-prefix '*' --source-port-range '*' --priority 1000
            
            - Create a firewall rule that allows external HTTPS
            
            az network nsg rule create -g kubernetes_eastus_rg -n kubernetes-allow-api-server --access allow --destination-address-prefix '*' --destination-port- 
            range 6443 --direction inbound --nsg-name kubernetes-nsg --protocol tcp --source-address-prefix '*' --source-port-range '*' --priority 1001
            
             - Listing the firewall rules in the Kubernetes-vnet
             
            az network nsg rule list -g kubernetes_eastus_rg --nsg-name kubernetes-nsg --query "[].{Name:name, Direction:direction, Priority:priority,
            Port:destinationPortRange}" -o table
           Name                         Direction    Priority    Port
          ---------------------------  -----------  ----------  ------
           kubernetes-allow-ssh         Inbound      1000        22
           kubernetes-allow-api-server  Inbound      1001        6443
            
            
           3c.   Expose the Kubernetes API Servers to remote clients by using external loadbalancer
           
            - Create external load balancer for Kubernetes API Servers and allocate a static IP
            
            az network lb create -g kubernetes_eastus_rg -n kubernetes-lb --backend-pool-name kubernetes-lb-pool --public-ip-address kubernetes-pip --public-ip-
            address-allocation static
            
            - Verify the kubernetes-pip static IP address was created correctly
            
             az network public-ip list --query="[?name=='kubernetes-pip']. 
            {ResourceGroup:resourceGroup,Region:location,Allocation:publicIpAllocationMethod,IP:ipAddress}" -o table
             
             ResourceGroup         Region    Allocation    IP
            --------------------  --------  ------------  -------------
             kubernetes_eastus_rg  eastus2   Static        <publicipaddress> 
            
            
            3d. Creating Virtual Machines
            
               - Select the latest Ubuntu Server for the VM's. Check latest available release and run following command 
               
               az vm image list --location eastus2 --publisher Canonical --offer UbuntuServer --sku 18.04-LTS --all -o table
               
               - Replace UBUNTULTS variable below with latest row in the table
                 
                Set-Variable UBUNTULTS Canonical:UbuntuServer:18.04-LTS:18.04.202208100
                
                - Validate Variable
                
                 Get-Variable -Name UBUNTULTS
                 
               - Create Availability Set
                 
                  az vm availability-set create -g kubernetes_eastus_rg -n controller-as
                 [An availability set is a logical grouping of VMs that allows Azure to understand how your  application is built to provide for redundancy and 
                 availability]
                 
                - Create 3 public ips for the controller
                
                az network public-ip create -n controller-0-pip -g kubernetes_eastus_rg
                az network public-ip create -n controller-1-pip -g kubernetes_eastus_rg
                az network public-ip create -n controller-2-pip -g kubernetes_eastus_rg
                
                 - Create 3 NICS for the controller
                 
                az network nic create -g kubernetes_eastus_rg -n controller-0-nic --private-ip-address 10.240.0.10 --public-ip-address controller-0-pip --vnet 
                kubernetes-vnet --subnet kubernetes-subnet --ip-forwarding --lb-name kubernetes-lb --lb-address-pools kubernetes-lb-pool
                
                az network nic create -g kubernetes_eastus_rg -n controller-1-nic --private-ip-address 10.240.0.11 --public-ip-address controller-1-pip --vnet 
                kubernetes-vnet --subnet kubernetes-subnet --ip-forwarding --lb-name kubernetes-lb --lb-address-pools kubernetes-lb-pool
                
                az network nic create -g kubernetes_eastus_rg -n controller-2-nic --private-ip-address 10.240.0.12 --public-ip-address controller-2-pip --vnet 
                kubernetes-vnet --subnet kubernetes-subnet --ip-forwarding --lb-name kubernetes-lb --lb-address-pools kubernetes-lb-pool
                
                  - Create 3 VMS for the controller
                az vm create -g kubernetes_eastus_rg -n controller-0 --image Canonical:UbuntuServer:18.04-LTS:18.04.202208100 --nics controller-0-nic --
                availability-set controller-as  --admin-username 'kuberoot' --generate-ssh-keys 

                az vm create -g kubernetes_eastus_rg -n controller-1 --image Canonical:UbuntuServer:18.04-LTS:18.04.202208100 --nics controller-1-nic -- 
                availability-set controller-as --nsg kubernetes-nsg --admin-username 'kuberoot' --generate-ssh-keys

                az vm create -g kubernetes_eastus_rg -n controller-2 --image Canonical:UbuntuServer:18.04-LTS:18.04.202208100 --nics controller-2-nic --
                availability-set controller-as --admin-username 'kuberoot' --generate-ssh-keys
                
                - Verify Creation of 3 VMS
                
                PS C:\Users\kkomand> az vm list -d -g kubernetes_eastus_rg -o table
                Name          ResourceGroup         PowerState    PublicIps       Fqdns    Location    Zones
                ------------  --------------------  ------------  --------------  -------  ----------  -------
                controller-0  kubernetes_eastus_rg  VM running    <IP>             eastus2
                controller-1  kubernetes_eastus_rg  VM running    <IP>            eastus2
                controller-2  kubernetes_eastus_rg  VM running    <IP>         eastus2
                
                
                
           
           
            
            
            
            
            
            
            
            
          

         
      
       
       
       
  




    

   

