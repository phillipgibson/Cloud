# Using Azure Front Door Service (AFD) with Azure Kubernetes Service (AKS) Walkthrough
Recently we [announced](https://azure.microsoft.com/en-us/blog/azure-front-door-service-is-now-generally-available/) the general availabilty of the Azure Front Door Service (AFD). If you're not familiar with the Azure Front Door service and what it is, you can simply think of it as a global endpoint service for your web applications. For me what's really exciting about the AFD service is the ability to use it along with AKS to take advantage of some key functionality AFD possesses like global https load balancing, custom domains, WAF capablities, session affinity, and URL rewite just to name a few. Having all of that functionality with ADF is really going to help streamline some of your current and future architectures and make it really easy to deploy and host global applications with AKS. Let's begin!

## Understanding the options for AKS with Azure's Application Delivery Suite
So if you take a look at the current documentation concerning [AKS BC/DR strategy]( https://docs.microsoft.com/en-us/azure/aks/operator-best-practices-multi-region ), specifically the network ingress flow for a multi-region cluster deployment of an AKS service, you will see that Azure Traffic Manager is load balancing the global DNS-based traffic. Downstream from the Traffic Manager service are network appliances (NVA). These NVAs most likely will be an Azure Application Gateway, or similar Azure Marketplace device,  to provide any WAF and session affinity needs. 

![alt text](https://github.com/phillipgibson/Cloud/blob/master/Azure/AKS/using-afd-with-aks/aks-azure-traffic-manager.png)

Depending on your application needs, there is a variety of ways and possibilities of exposing your web applications. You can explore these options in more detail with this article on [Building with Azure's application delivery suite](https://docs.microsoft.com/en-us/azure/frontdoor/front-door-lb-with-azure-app-delivery-suite#building-with-azures-application-delivery-suite). For the purposes of this walkthrough, we will be using ADF to load balance the global DNS-based traffic and http traffic between two Azure regions with identical AKS services deployed. I did personalize each AKS service so that the web page of each application identifies the Azure region the application is hosted in. This will allow us to see the live region servicing the application when we take the service offline in each region's AKS cluster. AFD will provide global high availablity by building a backend pool containing both public IPs of the AKS services in each region. 

![alt text](https://github.com/phillipgibson/Cloud/blob/master/Azure/AKS/using-afd-with-aks/aks-azure-front-door.png)

As you can tell, using AFD has already streamlined our architecture. Typical patterns would have a deployment of an Application Gateway in each region downstream from the global DNS traffic service. Remember that Azure Application Gateway is a regional service, so you would have to deploy it N number of times for each region. With using AFD I have eliminated the use of Azure Traffic Manager for global DNS-based traffic, and I'm able to push the functionality provied by Azure Application Gateway upstream to AFD providing me WAF, layer 7 path/url routing, and session state configuration. 




