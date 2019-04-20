# Using Azure Front Door Service (AFD) with Azure Kubernetes Service (AKS) Walkthrough
Recently we [announced](https://azure.microsoft.com/en-us/blog/azure-front-door-service-is-now-generally-available/) the general availabilty of the Azure Front Door Service (AFD). If you're not familiar with the Azure Front Door service and what it is, you can simply think of it as a global endpoint service for your web applications. For me what's really exciting about the AFD service is the ability to use it along with AKS to take advantage of some key functionality AFD possesses like global https load balancing, custom domains, WAF capablities, session affinity, and URL rewite just to name a few. Having all of that functionality with ADF is really going to help streamline some of your current and future architectures and make it really easy to deploy and host global applications with AKS. Let's begin!

## Understanding the options for AKS with Azure's Application Delivery Suite
So if you take a look at the current documentation concerning [AKS BC/DR strategy]( https://docs.microsoft.com/en-us/azure/aks/operator-best-practices-multi-region ), specifically the network ingress flow for a multi-region cluster deployment of an AKS service, you will see that Azure Traffic Manager is load balancing the global DNS-based traffic. Downstream from the Traffic Manager service are network appliances (NVA). These NVAs most likely will be an Azure Application Gateway, or similar Azure Marketplace device,  to provide any WAF and session affinity needs. 

![alt text](https://github.com/phillipgibson/Cloud/blob/master/Azure/AKS/using-afd-with-aks/images/aks-azure-traffic-manager.png)

Depending on your application needs, there is a variety of ways and possibilities of exposing your web applications. You can explore these options in more detail with this article on [Building with Azure's application delivery suite](https://docs.microsoft.com/en-us/azure/frontdoor/front-door-lb-with-azure-app-delivery-suite#building-with-azures-application-delivery-suite). For the purposes of this walkthrough, we will be using ADF to load balance the global DNS-based traffic and http traffic between two Azure regions with identical AKS services deployed. I did personalize each AKS service so that the web page of each application identifies the Azure region the application is hosted in. This will allow us to see the live region servicing the application when we take the service offline in each region's AKS cluster. AFD will provide global high availablity by building a backend pool containing both public IPs of the AKS services in each region. 

![alt text](https://github.com/phillipgibson/Cloud/blob/master/Azure/AKS/using-afd-with-aks/images/aks-azure-front-door.png)

As you can tell, using AFD has already streamlined the architecture. Typical patterns would have a deployment of an Application Gateway in each region downstream from the global DNS traffic service. Remember that Azure Application Gateway is a regional service, so you would have to deploy it N number of times per region hosting your application. With using AFD I have eliminated the use of Azure Traffic Manager for global DNS-based traffic, and I'm able to push the functionality provied by Azure Application Gateway upstream to AFD providing me WAF, layer 7 path/url routing, and session state configuration. 

## Deploy an AKS Cluster in Two Azure Regions
Since I'm highlighting AFD functionality, we'll be doing a simple deployment of AKS in each region. For this demo I'll be using Azure regions East US 2 and West US 2. 

> NOTE: I'm using the Azure Cloud Shell for all of my CLI commands. Specifically I'm using https://shell.azure.com

First thing we'll do is create two Service Principles for the two AKS clusters
```
az ad sp create-for-rbac -n demo-afd-aks-eastus2-cluster --skip-assignment
az ad sp create-for-rbac -n demo-afd-aks-westus2-cluster --skip-assignment
```

Please take note of the output of the service principles created. We will be using both the appId and password properties in later commands. You should see output similar to this below for each command. 
```
{
 "appID": "8cea7e76-0cda-45d2-a62b-bf75dfb8da91",
 "displayName": "demo-afd-aks-eastus2-cluster",
 "name": "http://demo-afd-aks-eastus2-cluster",
 "password": "9a7beaa9-902a-42ac-b03b-f9b4590c2190",
 "tenant": "f1d38a10-7214-4702-b571-8a1b70718c42"
}

{
 "appID": "ec1bc114-f7e0-4ed6-8a5c-80eb6e9d3856",
 "displayName": "demo-afd-aks-westus2-cluster",
 "name": "http://demo-afd-aks-westus2-cluster",
 "password": "26482765-da5f-46e5-acf9-fb58675b5533",
 "tenant": "f1d38a10-7214-4702-b571-8a1b70718c42"
}
```
> NOTE: Yes, all the service principle information listed here is fake :)

Now create two resource groups. One for each Azure Region
```
az group create -l eastus2 -n demo-afd-aks-eastus2-cluster
az group create -l westus2 -n demo-afd-aks-westus2-cluster
```

We will now create the Azure Virtual Networks for each Azure Region

For the Azure East US 2 Region
```
az network vnet create \
 -g demo-afd-aks-eastus2-cluster \
 -n demo-afd-aks-eastus2-cluster-vnet \
 --address-prefixes 10.50.0.0/16 \
 --subnet-name demo-afd-aks-eastus2-cluster-aks-subnet \ 
 --subnet-prefix 10.50.1.0/24
 ```
 
We're going to create two additional subnets in each VNet. I like to create a subnet specific to the AKS service range. I do this so that if I try to use that range in Azure somewhere else Azure will tell me that it is in use, so I won't have any conflict with other devices. We are also going to make a Azure Firewall subnet. This will be used to show you an additional configuration of using the Azure Firewall's public IP to NAT back to a AKS service that is using an internal load balancer. 
 
Create the Additional Subnet for the AKS Service Range
```
az network vnet subnet create \
 -g demo-afd-aks-eastus2-cluster \
 --vnet-name demo-afd-aks-eastus2-cluster-vnet \
 --name demo-afd-aks-eastus2-cluster-vnet-akssvc-subnet \
 --address-prefix 10.50.2.0/24
```
Create the Additional Subnet for the Azure Firewall
> **PLEASE NOTE:** The name of the subnet for the Azure Firewall has to be named "azurefirewallsubnet". Failure to name the subnet the correct name will result in an error when you try to deploy the Azure Firewall to that subnet.
```
az network vnet subnet create \
 -g demo-afd-aks-eastus2-cluster \
 --vnet-name demo-afd-aks-eastus2-cluster-vnet \
 --name azurefirewallsubnet \
 --address-prefix 10.50.0.0/24
``` 

Before we can deploy the AKS cluster, we need to add the contributor role to the service principle for the Azure Vnet we created. Make sure you are using the appID from the service principle as the assignee parameter when assigning the role.
```
VNETID=$(az network vnet show -g demo-afd-aks-eastus2-cluster --name demo-afd-aks-eastus2-cluster-vnet --query id -o tsv)
az role assignment create --assignee 8cea7e76-0cda-45d2-a62b-bf75dfb8da91 --scope $VNETID --role Contributor
```
We also need to identify the specific subnet in the VNet where the AKS cluster will be deployed.
```
SUBNET_ID=$(az network vnet subnet show --resource-group demo-afd-aks-eastus2-cluster --vnet-name demo-afd-aks-eastus2-cluster-vnet --name demo-afd-aks-eastus2-cluster-aks-subnet --query id -o tsv)
```
Now we're ready to deploy the AKS cluster in the Azure East US 2 region. 
```
az aks create \ 
  --resource-group demo-afd-aks-eastus2-cluster \ 
  --name demo-afd-aks-eastus2-cluster \ 
  --kubernetes-version 1.12.6 \ 
  --node-count 1 \ 
  --node-vm-size Standard_B2s \
  --generate-ssh-keys \ 
  --network-plugin azure \
  --network-policy azure \ 
  --service-cidr 10.50.2.0/24 \ 
  --dns-service-ip 10.50.2.10 \ 
  --docker-bridge-address 172.17.0.1/16 \ 
  --vnet-subnet-id $SUBNET_ID \ 
  --service-principal 8cea7e76-0cda-45d2-a62b-bf75dfb8da91 \ 
  --client-secret 9a7beaa9-902a-42ac-b03b-f9b4590c2190 \
  --no-wait 
  ```
  
  We'll build the exact same AKS cluster in Azure West US 2 Region. We already have the Resource Group and Service Principle created. 
  ```
  # Create the WestUS 2 AKS Cluster VNet
az network vnet create \
 -g demo-afd-aks-westus2-cluster \
 -n demo-afd-aks-westus2-cluster-vnet \
 --address-prefixes 10.60.0.0/16 \
 --subnet-name demo-afd-aks-westus2-cluster-aks-subnet \ 
 --subnet-prefix 10.60.1.0/24

# Create the Additional VNet Subnets for the AKS Service Range and Azure Firewall
az network vnet subnet create \
 -g demo-afd-aks-westus2-cluster \
 --vnet-name demo-afd-aks-westus2-cluster-vnet \
 --name demo-afd-aks-westus2-cluster-vnet-akssvc-subnet \
 --address-prefix 10.60.2.0/24

az network vnet subnet create \
 -g demo-afd-aks-westus2-cluster \
 --vnet-name demo-afd-aks-westus2-cluster-vnet \
 --name azurefirewallsubnet \
 --address-prefix 10.60.0.0/24

# Assign the West US 2 Service Principle Contributor Role to the AKS VNet
VNETID=$(az network vnet show -g demo-afd-aks-westus2-cluster --name demo-afd-aks-westus2-cluster-vnet --query id -o tsv)
az role assignment create --assignee ec1bc114-f7e0-4ed6-8a5c-80eb6e9d3856 --scope $VNETID --role Contributor

# Identify the AKS Cluster Subnet for Deployment
SUBNET_ID=$(az network vnet subnet show --resource-group demo-afd-aks-westus2-cluster --vnet-name demo-afd-aks-westus2-cluster-vnet --name demo-afd-aks-westus2-cluster-aks-subnet --query id -o tsv)

# Deploy the West US 2 AKS Cluster
az aks create \ 
  --resource-group demo-afd-aks-westus2-cluster \ 
  --name demo-afd-aks-westus2-cluster \ 
  --kubernetes-version 1.12.6 \ 
  --node-count 1 \ 
  --node-vm-size Standard_B2s \
  --generate-ssh-keys \ 
  --network-plugin azure \
  --network-policy azure \ 
  --service-cidr 10.60.2.0/24 \ 
  --dns-service-ip 10.60.2.10 \ 
  --docker-bridge-address 172.17.0.1/16 \ 
  --vnet-subnet-id $SUBNET_ID \ 
  --service-principal ec1bc114-f7e0-4ed6-8a5c-80eb6e9d3856 \ 
  --client-secret 26482765-da5f-46e5-acf9-fb58675b5533 \
  --no-wait
  ```

We now have two AKS clusters up and running in both the East US 2 and West US 2 Azure Regions. You can verify the AKS clusters are ready by getting the credentials and ensuring the nodes are in a ready status.
```
az aks get-credentials -n demo-afd-aks-eastus2-cluster -g demo-afd-aks-eastus2-cluster
kubectl get nodes

az aks get-credentials -n demo-afd-aks-westus2-cluster -g demo-afd-aks-westus2-cluster
kubectl get nodes
```

## Deploy the Sample Applications to each AKS Cluster
Now let's deploy a simple Node.js app to each cluster. Each app is personalized for the Azure region it will be deployed to. For this round we'll be using the deployment that creates a public IP address for the service. 

For the Azure East US 2 deployment use the following:
```
kubectl config use-context demo-afd-aks-eastus2-cluster
kubectl create -f https://raw.githubusercontent.com/phillipgibson/Cloud/master/Azure/AKS/using-afd-with-aks/phillipgibson-azure-frontdoor-eastus2-elb-app.yaml
```
Verify the deployed app has received a public IP and then browse to that endpoint.
```
kubectl get svc
```
![alt text](https://github.com/phillipgibson/Cloud/blob/master/Azure/AKS/using-afd-with-aks/images/aks-afd-verify-eastus2-app-kubectl.png)

![alt text](https://github.com/phillipgibson/Cloud/blob/master/Azure/AKS/using-afd-with-aks/images/aks-afd-verify-eastus2-app-browser.png)

Repeat the same deployment for the West US 2 AKS cluster and verify you can browse to the endpoint.
```
kubectl config use-context demo-afd-aks-westus2-cluster
kubectl create -f https://raw.githubusercontent.com/phillipgibson/Cloud/master/Azure/AKS/using-afd-with-aks/phillipgibson-azure-frontdoor-westus2-elb-app.yaml
kubectl get svc
```
![alt text](https://github.com/phillipgibson/Cloud/blob/master/Azure/AKS/using-afd-with-aks/images/aks-afd-verify-westus2-app-kubectl.png)

![alt text](https://github.com/phillipgibson/Cloud/blob/master/Azure/AKS/using-afd-with-aks/images/aks-afd-verify-westus2-app-browser.png)

Make note of both the external public IP address of both your AKS services endpoints in each Azure region. We will use them to configure the backend configuration of the AFD service.

## Deploy the Azure Front Door Service
Now we'll tie it all together and use AFD as a global endpoint for the two AKS services running in seperate Azure Regions (EastUS2 and WestUS2)

Create a resource group for the AFD instance. AFD is a global service, just like Azure Traffic Manager, and we must just associate it to a region for the ARM deployment.
```
az group create -l eastus2 -n demo-afd-aks-global
```
Most likely if this is your first time using AFD you will need to install the Azure CLI extension, even when using Azure Cloud Shell
```
az extension add --name front-door
```
> NOTE: At the time of this writing the AFD Azure CLI extension is in preview

AFD has a lot of configuration options for the backend pools and routing rules. Since we're just hosting a simple web application we'll keep this deployment configuration simple. I'll create additional posts to show off some of the URL rewite features and how you can take advantage of that with AKS for building out microservices routing.

No we'll deploy AFD with the inital backend of the AKS service located in the East US 2 Azure datacenter.
```
az network front-door create \
 -n demoafdaks \
 -g demo-afd-aks-global \
 --backend-address 40.84.40.122 \
 --backend-host-header 40.84.40.122 \
 --protocol Http \
 --forwarding-protocol HttpOnly
 ```
 > NOTE: I could not figure out how to initially name the backend pool and routing rule configuration using the CLI. When using the portal/UI you do have this option. For now the backend pool and routing rule configuration names will be DefaultBackendPool and DefaultRoutingRule.
 
 Next is to add the AKS service endpoint from the West US 2 Azure datacenter to the backend pool for AFD
 ```
 az network front-door backend-pool backend add \
 --resource-group demo-afd-aks-global \
 --front-door-name demoafdaks \
 --pool-name DefaultBackendPool \
 --address 52.183.68.177 \
 --backend-host-header 52.183.68.177
 ```
You can now check that your backends have been configured
```
az network front-door backend-pool list  --front-door-name demoafdaks  --resource-group demo-afd-aks-global --query '[].backends' -o json
```
At this point AFD is fully configured and you should be able to query the AFD frontend host endpoint and be routed to one of the AKS services endpoints in either Azure region via your browser.
```
az network front-door frontend-endpoint list  --front-door-name demoafdaks  --resource-group demo-afd-aks-global --query '[].hostName' -o json
```
Before we call this complete, there's one last thing we need to do from a hardening perspective to ensure we're funneling all internet traffic to our AKS services endpoint from AFD. Right now the configuration still allows someone the ability to access the AKS service endpoints by going directly to thier external public IP and essentially bypassing any WAF or other configurations we may have setup in AFD. To ensure that the AKS services endpoints only receive traffic from AFD, we will need to modify the default NSGs created for those services to only accept traffic from the AFD service. 

The AFD FAQ already has this process documented and on the roadmap for NSGs will be a specific service tag for AFD which will make this even easier to configure. For right now we simply just need to configure our AKS service NSGs to only accept traffic from the AFD IPv4 CIRD of 147.243.0.0/16. More details on this can be found here https://docs.microsoft.com/en-us/azure/frontdoor/front-door-faq#how-do-i-lock-down-the-access-to-my-backend-to-only-azure-front-door-service. 

Locate the NSG of the AKS service. This will be found in the MC_* resource group for the AKS cluster. You will know you're working on the correct NSG rule becasue the destination IP address should match the external IP address of the AKS service.

![alt text](https://github.com/phillipgibson/Cloud/blob/master/Azure/AKS/using-afd-with-aks/images/aks-svc-identify-nsg.png)

In the properties of the NSG rule you will notice that it is using the Service Tag Internet as the accepting source. This configuration is what allows traffic from anyone on the internet to the AKS service and will need to be changed to only accept traffic from the AFD service.

![alt text](https://github.com/phillipgibson/Cloud/blob/master/Azure/AKS/using-afd-with-aks/images/aks-svc-default-nsg-config.png)

Change the Source option to IP Addresses and in the Source IP addresses/CIDR ranges property enter the AFD service CIDR range of 147.243.0.0/16 and save the configuration.

![alt text](https://github.com/phillipgibson/Cloud/blob/master/Azure/AKS/using-afd-with-aks/images/aks-svc-afd-nsg-config.png)

You should now see the updated configuration of the NSG in the inbound security rules.

![alt text](https://github.com/phillipgibson/Cloud/blob/master/Azure/AKS/using-afd-with-aks/images/aks-svc-afd-nsg-config-complete.png)

You will need to repeat those steps for every AKS cluster service, and you should now not be able to go directly to the AKS service endpoint and only be able to view the AKS service endpoints through the AFD service. 

The only thing left to do now is to test AFD's global loadbalancing feature. You can do this easily by just stopping the cluster node VM to simulate an outage for that region. Browsing the AFD endpoint, you should immediately see the browser pick up the West US 2 AKS service endpoint. You can then test it in reverse by starting back up the East US 2 AKS cluster node and then stoping the West US 2 AKS cluster node. 

## Coming Soon: Using AFD with Azure Firewall to Expose a Interally (Private VNet IP) Load Balanced AKS Service
