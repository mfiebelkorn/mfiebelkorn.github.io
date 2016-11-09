---
layout: post
title:  "Application Load Balancing From a DevOps System Perspective"
date:   2016-11-09 13:39:00
comments: false
modified: 2016-11-09
---

I was introduced to load balancing a few years ago when I was put in charge of architecting a highly available application environment for our application. Truth be told, I didn't know a enourmous about load balancing at the time, I had no reason to learn about it. So I invested some time in learning the how/why/why of load balancing, I thought I'd share what I learned here. 

##	The 30,000 foot view
Yes I know "The 30,000 foot view" is one of the most annoying business phrases still in use today. I don't care, I still like it. Application load balancing can be broken down into two major groups: Layer 4 Load Balancing and Layer 7 Load Balancing. Most folks stumble into Layer 7 Load Balancing right away when their supervisor tells them to find a Load Balancing solution. In fact, when most people say "We have load balancing" they're usually referring to Layer 7 Load Balanacing; It's sort of implied. The less popular Layer 4 Load Balancing has an important role too, we'll get to that. 

These two layers, Layer 7 and Layer 4 are best used together. Together they compliment one anothers redundancy and high availability by eliminating single points of failure in your infrastructure. That is a good segway to an important thing I want to mention: load balancing is different than highly available. I've heard many people assume that "load balanced" implies "highly available," but they're simply not the same thing. Not necessarily.

## Layer 7 Load Balancing
Layer 7 refers to the most common layer to load balance at: the application layer. At this layer a load balancing appliance or VM is privy to information (such as the URL) in the request header. This information essentially runs the show within a typical load balancer setup. Here's a diagram of a typical Layer 7 load balanced application:
![Layer7LoadBalancedApplication](/images/layer7loadbalancing.png)
User makes request which is routed to the load balancer, the load balancer routes the request to the proper application server based on the URL; in this case, if the URL is 'yourdomain.com/blog' they are routed to the blog backend, else they are routed to the web backend. 

I have worked extensively with just a few Layer 7 Load Balancers, IIS ARR and Stingray Traffic Manager. They operate in much the same way, in IIS you create "Server Farms" which can be one or many nodes
![IISServerFarms](/images/IISServerFarms.PNG)
Traffic is managed by way of "URL Rewrite" Rules (similar concept, but different name in Stringray Traffic Manager). These rules can get pretty complex, but generally only simple logic is needed. A pretty common way to do some simple load balancing is to use different DNS Names for the different applications, all of which point to the same Layer 7 Load Balancer. In IIS you'd simply add the appropriate Server Farms and add a rule like this... 
![IISRoutingRules](/images/IISRoutingRules.PNG)
Which passes all traffic to TestFarm2 on one condition, that the header parameter "host header" matches the wildcard pattern "*.mytest.com*". Using this methodlogy you can put any number of application behind the same load balancer so long as those applications can be differentiated at Layer 7 by a different DNS Name. 

Pretty straightforward. Point your applications DNS Entries to the load balancer and make routing rules. But what happens when your Load Balancer needs updates, or goes down unexpectedly... all of your applications go down with it. Enter Layer 4 Load Balancing.

## Layer 4 Load Balancing
I like to call Layer 4 Load Balancing the Load Balancer's Load Balancer. I know that sounds rediculous, that part of the reason I like to call it that. Layer 4 Load Balancing happens at the transport layer, so it operates on the IP Level. It's not privy to information in the request header, it makes its routing decisions based only on source IP and destination IP. With this layer of load balancing you can specify where you want a group of source IPs to be directed, or you can use it to be aware of your two Layer 7 Load Balancers. You heard me, two Layer 7 load Balancers. By using one Layer 4 load balancing appliance we can increase the availability of our application by making it aware of two layer 7 backend load balancers. Here's a diagram... 
<Insert Cool Diagram>
You'd have one Layer 4 Appliance load balancing two Layer 7 Load Balancers. The advantage here is that your [presumably more reliable] Layer 4 Load Balancer Appliance is now your single point of failure and not your [presumably more volitile] Layer 7 Load Balancers can undergo scheduled maintenance and updates. Is this passing the buck of SPOF (Single Point of Failure) off to a different location? Yes. The idea is still useful if you're using a less stable Layer 7 load balancer (not an appliance) that may need updates and maintenance (such as Windows IIS ARR). 

## Final Thoughts 
It would be beneficial to anyone looking into implementing load balancing to brush up on your load balancing algorithms and understand some bucket terms like Server Affinity, Load Distribution, and Proxy Settings. Also, if you're looking to implement a Layer 7 load balancer make sure you understand how your particular Layer 7 Load Balancer does it's health checks. That is, how does your load balancer know when nodes are online (or does it know at all). This is important because it serves as the big divider between high availability and load balancing - if a load balancer is not aware of an application node being offline, it will continue to pass traffic to it, and that's bad. If your layer 7 load balancer IS aware of the application server's heart beat, you'll want to understand how EXACTLY it works; does it try and open a web page and call the node unhealthy if it receives 10 Status 200 messages in a row? Does it simply ping the server and call it healthy so long as it responds? These interworking are important if you're to understand just how highly available (or not) your application is after it's behind a load balancer. You need to understand how the client experience is going to be effected whenever you take down a node or a load balancer. 
