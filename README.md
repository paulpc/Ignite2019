# Ignite2019
Templates and auxiliary materials for the presentation at Palo Alto Ignite 2019. Remember, this is a cusrtomization of the Palo Alto Networks Transit VNET template stack and architecture (https://github.com/PaloAltoNetworks/Azure-Transit-VNet/) - read through that before getting started with this.

In order to get started you will need to have an Azure account with a subscription of your choosing. This set of templates assumes an account already existing with Palo Alto Networks support as well as a Panorama device on your network with the apropriate template stacks and device groups. A big assumption, but for more details, check out:

## Setup the Resource Group
The focus of this stack is the ARM templates for the Transit-VNET architecture. In order to start deploying resources, you will need to log in and create a resource group. There are several lines of throught on how do deploy to Azure. Some utilize Subscriptions as the billing / security boundary, others use resource groups. Regardless, you will end up with two or more resource groups for your Hub and Spoke resources.
```
az login
az account set --subscription [your hub subscription name goes here]
cd ARM
az group create --name ignite2019hub --location westus --tags "function=CyberRangeIgniteHub" "manager=Manager-Name-Here" "project=CloudMigration" "owner=InformationSecurity" "app=PaloAltoNetworksVirtual"
# if different, switch to spoke subscription
az account set --subscription [your spoke subscription name goes here]
az group create --name ignite2019spoke1 --location westus --tags "function=CyberRangeIgniteSpoke1" "manager=Manager-Name-Here" "project=CloudMigration" "owner=InformationSecurity" "app=PaloAltoNetworksVirtual"
```

Notice that we are using tags - they come in very handy for multiple reasons:
- cost accounting (when you get asked about the Palo Alto Resources' IaaS costs)
- keeping track of the hub / spoke resources
- metadata that you can't get from other Azure fields

## Deploying the Hub

After selecting the hub subscription, there are a few things that need to happen before being able to create firewalls. Sometimes, the Hub subscription will already have the VNET configured - as you are probably using it for other reasons and you should have the VPN / Express Route gateway configured in that VNET's gateway subnet. Otherwise, if it is a vanilla deployment, we let us also create the VNET for it:
```
cd Cyber-Range-Hub
az group deployment create --name ignitehuboptionalvnet --template-file azureDeploy-hubVNETGateway.json --parameters CyR-optional-network-parameters.json --resource-group ignite2019hub
```
and then start the setup:
```
az group deployment create --name ignitehubsetup --template-file azureDeploy-hubSetup.json  --parameters CyR-hub-setup-parameters.json  --resource-group ignite2019hub
```
edit the Azure-Cyber-Range-Hub/config/init-cfg.txt file with the panorama configs:
vm-auth-key=[enter key here]
panorama-server=192.168.12.4

You should know the panorama server. You can generate a VM-Auth-Key in panorama:
- https://docs.paloaltonetworks.com/vm-series/7-1/vm-series-deployment/bootstrap-the-vm-series-firewall/generate-the-vm-auth-key-on-panorama
- More bootstrap info:
https://docs.paloaltonetworks.com/vm-series/7-1/vm-series-deployment/bootstrap-the-vm-series-firewall

Make sure the right folders are in the storage account; you will need a stub of:
- config
    - bootstrap.xml
    - init-cfg.txt
- license
    - authcodes
- content
    - any content packs you want to pre-stage (i.e. antivirus)
- software
    - any software updates you want to pre-stage (i.e. 9.0)

Copy the bootstrap configs to the storage account created in the setup. You'll need to create a file share and place the folders there. Keep track of them for the fw deployments. (e.g. file-share=bootstrap,share-directory=Azure-Cyber-Range-Spoke). You will also need to create folders for 

Edit the parameters file and deploy the firewalls
```
az group deployment create --name fwsignitehub --template-file azureDeploy-hub.json --parameters CyRhub-fws-parameters.json --resource-group ignite2019hub
```
Firewalls are deployed internally, so you need a way to access the VNET (like a jump server, VPN, or ExpressRoute)

## Deploying the Spoke
Deploy the setup template / parameters:

```
cd ../Cyber-Range-Spole/
az group deployment create --name ignitespoke --template-file azureDeploy-spokeSetup.json --parameters CyR-spoke-setup-parameters.json --resource-group ignite2019spoke1
```

Make sure that the init-cfg.txt is properly configured in the storage account
Azure-Cyber-Range-Spokes/config/init-cfg.txt:
vm-auth-key=[enter key here]
panorama-server=192.168.12.4

Now you can deploy the firewalls. Change the CyRspoke-fws-parameters.json file to reflect your deployment. Notice the numberOfInstances parameter - this tells Azure how many vm-series to deploy in this config.
```
az group deployment create --name fwsignitespoke --template-file azureDeploy-spoke.json --parameters CyRspoke-fws-parameters.json  --resource-group ignite2019spoke1 
```
Firewalls are deployed internally, so you need a way to access the VNET (like a jump server, VPN, or ExpressRoute)

## next steps
- Working on improving the 4-interface model
- working on spokes front-loading kubernetes (Azure Kubernetes Services)
- Autoscaling based on load through the standard load balancer