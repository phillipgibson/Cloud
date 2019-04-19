# Using Azure Front Door Service (AFD) with Azure Kubernetes Service (AKS) Walkthrough
Recently we [announced](https://azure.microsoft.com/en-us/blog/azure-front-door-service-is-now-generally-available/) the general availabilty of the Azure Front Door Service (AFD). If you're not familiar with the Azure Front Door service and what it is, you can simply think of it as a global endpoint service for your web applications. For me what's really exciting about the AFD service is the ability to use it along with AKS to take advantage of some key functionality AFD possesses like global https load balancing, custom domains, WAF capablities, session affinity, and URL rewite just to name a few. Having all of that functionality with ADF is really going to help streamline some of your current and future architectures and make it really easy to deploy and host global applications with AKS. Let's begin!

## Understanding the options for AKS with Azure's Application Delivery Suite
So if you take a look at the current documentation concerning [AKS BC/DR strategy]( https://docs.microsoft.com/en-us/azure/aks/operator-best-practices-multi-region ), specifically the network ingress flow for a multi-region cluster deployment of an AKS service, you will see that Azure Traffic Manager is load balancing the global DNS-based traffic. Downstream from the Traffic Manager service are network appliances (NVA). These NVAs most likely will be an Azure Application Gateway, or similar Azure Marketplace device,  to provide any WAF and session affinity needs. 

![alt text](https://github.com/phillipgibson/Cloud/blob/master/Azure/AKS/using-afd-with-aks/aks-azure-traffic-manager.png)

Depending on your application needs, there is a variety of ways and possibilities of exposing your web applications. You can explore these options in more detail with this article on [Building with Azure's application delivery suite](https://docs.microsoft.com/en-us/azure/frontdoor/front-door-lb-with-azure-app-delivery-suite#building-with-azures-application-delivery-suite). For the purposes of this walkthrough, we will be using ADF to load balance the global DNS-based traffic and http traffic between two Azure regions with identical AKS services deployed. I did personalize each AKS service so that the web page of each application identifies the Azure region the application is hosted in. This will allow us to see the live region servicing the application when we take the service offline in each region's AKS cluster. AFD will provide global high availablity by building a backend pool containing both public IPs of the AKS services in each region. 

![alt text](https://github.com/phillipgibson/Cloud/blob/master/Azure/AKS/using-afd-with-aks/aks-azure-front-door.png)

As you can tell, using AFD has already streamlined the architecture. Typical patterns would have a deployment of an Application Gateway in each region downstream from the global DNS traffic service. Remember that Azure Application Gateway is a regional service, so you would have to deploy it N number of times per region hosting your application. With using AFD I have eliminated the use of Azure Traffic Manager for global DNS-based traffic, and I'm able to push the functionality provied by Azure Application Gateway upstream to AFD providing me WAF, layer 7 path/url routing, and session state configuration. 

## Deploy an AKS Cluster in Two Azure Regions
Since I'm highlighting AFD functionality, we'll be doing a simple deployment of AKS in each region. For this demo I'll be using Azure regions East US 2 and West US 2. 

> NOTE: I'm using the Azure Cloud Shell for all of my CLI commands

First thing we'll do is create two Service Principles for the two AKS clusters
```
az ad sp create-for-rbac -n demo-adf-aks-eastus2-cluster --skip-assignment
az ad sp create-for-rbac -n demo-adf-aks-westus2-cluster --skip-assignment
```

Please take note of the output of the service principles created. We will be using both the appId and password properties in later commands. You should see output similar to this below for each command. 
```
{
 "appID": "8cea7e76-0cda-45d2-a62b-bf75dfb8da91",
 "displayName": "demo-adf-aks-eastus2-cluster",
 "name": "http://demo-adf-aks-eastus2-cluster",
 "password": "9a7beaa9-902a-42ac-b03b-f9b4590c2190",
 "tenant": "f1d38a10-7214-4702-b571-8a1b70718c42"
}
```
> NOTE: Yes that information is fake :)

Now create two resource groups. One for each Azure Region
```
az group create -l eastus2 -n demo-adf-aks-eastus2-cluster
az group create -l eastus2 -n demo-adf-aks-westus2-cluster
```

We will now create the Azure Virtual Networks for each Azure Region

For the Azure East US 2 Region
```
az network vnet create \
 -g demo-adf-aks-eastus2-cluster \
 -n demo-adf-aks-eastus2-cluster-vnet \
 --address-prefixes 10.50.0.0/16 \
 --subnet-name demo-adf-aks-eastus2-cluster-aks-subnet \ 
 --subnet-prefix 10.50.1.0/24
 ```
 
We're going to create two additional subnets in each VNet. I like to create a subnet specific to the AKS service range. I do this so that if I try to use that range in Azure somewhere else Azure will tell me that it is in use, so I won't have any conflict with other devices. We are also going to make a Azure Firewall subnet. This will be used to show you an additional configuration of using the Azure Firewall's public IP to NAT back to a AKS service that is using an internal load balancer. 
 
Create the Additional Subnet for the AKS Service Range
```
az network vnet subnet create \
 -g demo-adf-aks-eastus2-cluster \
 --vnet-name demo-adf-aks-eastus2-cluster-vnet \
 --name demo-adf-aks-eastus2-cluster-vnet-akssvc-subnet
 --address-prefix 10.50.2.0/24
```


