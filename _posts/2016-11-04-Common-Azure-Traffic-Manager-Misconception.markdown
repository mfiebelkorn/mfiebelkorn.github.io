---
layout: post
title:  "A Common[ish] Azure Traffic Manager Misconception"
date:   2016-11-04 14:13:00
comments: false
modified: 2016-11-04
---

I call this a common[ish] misconception only because I was the one who assumed the Azure Traffic Manager to be something it wasn't, and it makes me feel better if I pretend other people have too. 

##The Problem
Before I started, the team was in need of a quick-fix application load balancer for our Performance Testers who were working against an agressive dead line to get the application performance metrics delivered soon. In production we had used Brocade's Stingray Traffic Manager (which works great by the way) but we had trouble in the past getting it work in Azure (the new home for our environments). Instead of taking the time to get Brocade's Marketplace Image working, we sought refuge in using the quick to provision Azure Traffic Manager to fulfill our application load balancing needs. Bad idea. 

Not long after implemeting Azure traffic manager, the performance team was peppering me with good questions. I sat down with them to figure out what was going on and found that they were not getting the round robin results they might expect. Specifically, when they hammered the application with 10k requests per minute, only one node started smoking while the other two application nodes sat happily nearby and watched the fireworks. To troubleshoot, they took down node one and hit the application only to find that now the applicatoin wasn't accessible at all now. It was as if the load balancer (Azure Traffic Manager) was only aware of one node. A nearby user (who had never hit the application before) also tried to access the application while the others were getting 404 errors - and they were able to successfully access the application. If they let it sit for awhile, it would start working again, but inconsistently. 

##The Research
I started by focusing on why an offline node, would still be getting traffic passed to it. Most traffic managers are smarter than that. I quickly came to find that our Traffic Manager was reporting all nodes degraded, and as such, it was treating them all as though they were healthy. 

![downNodes](/images/downNodes.PNG)

It didn't take long to figure out why; Azure Traffic Manager requires the nodes to operate on a public NIC (Which is unfortunate as this was an internal application so the only use for public facing Azure IP Addresses was for this Azure Traffic Manager). In order to establish a heart beat to the nodes, it uses what it knows (the public facing IP Addresses) to try and open the application. If it returns a status of 200, it reports the node is online (yes, it thankfully ignores cert issues which are bound to happen when using the Azure Public DNS Names). The solution then, to get these nodes to start returning status 200, was to simply add the public Azure DNS name of the public NIC to a binding of one of the sites. Simple as that. Nodes started to come online in the Azure Traffic Manager Portal. One problem down. Let's step back to the orginal problem and see if it still persists. 

All nodes online, user accesses the application. Everythign works. Baseline test done. One node down, same user access the application, receives 404 errors across the board, application is not accessible. This persists through closing browsers, switching browsers. New user who has never access the application on their machine before accesses the application, works fine. Our problem still exists. Why is the Azure Traffic Manager still routing folks to degraded nodes (and yes, it was indeed marking these down nodes as degraded now that we fixed the binding issue). 

##The Resolution
I took a step back and decided I needed to better understand how the Azure Traffic Manager works, I probably should have started there in the first place. I found this diagram, which immediately explained our issue. 
![azureTrafficManager](/images/azureTrafficManager.png)

You'll notice the smoking gun is line 8 (and the process leading up to it for that matter). The Azure Traffic Manager is essentially a glorified DNS Manager exchanging CNAME Entries! Once the DNS entry is cached client side, the client will connect directly to the node. This will not refresh until a DNS Refresh. In this way, Azure Traffic Manager is not a true Layer 7 Traffic Manager (it doesn't claim to be one by the way, oops).

Our issue, was clearly DNS Caching. I pinged the external Azure Traffic Manager IP, it resolves to node 1. I took down node one, waited for it's status to change to 'degraded' in the Azure Traffic Manager Portal. I pinged the external address again, and found that I'm still getting node 1 (cached). Then I flushed my DNS and pinged again. This time, it resolves to a healthy node, node 2. That's because the flush forced me to ask the Traffic Manager for an updated CNAME, to which it replied with a healthy node. Just as it should. 

The only place I can see this SAAS being useful would be if our application was deployed across different geographical locations, and we wanted users to be [directly] connected to the closest application node (or Layer 7 Load Balancer). 

The solution was to stop treating the Azure Traffic Manager as a Layer 7 Load Balancer, because it's not one. Azure now has a true Layer 7 Load Balancer SAAS called "Application Gateways." I was put in charge of getting the Brocade Stringray Load Balancer (a true Layer 7 Load Balancer) to work in Azure. It went smoothly. I'm now pushing to use Azure's Load Balancer SAAS (which is true Layer 4 Load Balancing) to manage two clusters of Brocade. In this way, our application will be truely highly available via Layer 4 and Layer 7 load balancing.


##Sources
[How Traffic Manager Works](https://azure.microsoft.com/en-us/documentation/articles/traffic-manager-how-traffic-manager-works/)

[Traffic Manager Monitoring](https://azure.microsoft.com/en-us/documentation/articles/traffic-manager-monitoring/)

[Traffic Manager Troubleshooting Degraded](https://azure.microsoft.com/en-us/documentation/articles/traffic-manager-troubleshooting-degraded/)
