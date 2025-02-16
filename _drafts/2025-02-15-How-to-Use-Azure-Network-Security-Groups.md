---
layout: post
title:  "How to Use Azure Network Security Groups"
date:   2025-02-15 22:38:11 +0000
categories: New blog setup
---

## Intro

There has been a lot of confusion around Azure's [Network Security Groups](https://learn.microsoft.com/en-us/azure/virtual-network/network-security-groups-overview) "NSG". This post aims to explain details and hopefully alleviate misunderstandings as well as suggest some good ways to use NSGs.

## What is an Azure Network Security Group?

First off, let's cover what an NSG really is. They are a type of firewall for layer 3 and 4 traffic. The *layer 3 and 4* is referring to the [Open Systems Interconnection](https://en.wikipedia.org/wiki/OSI_model) model. Layer 3, the Network Layer, is for Internet Protocol "IP" addresses such as 192.168.0.1. Layer 4, the Transport Layer, is for protocols like TCP, UDP, and ICMP. These are also the only protocols that the NSG will work with.

An NSG provides firewall functionality on these two layers. Each rule can work with a single IP address or a range of IP addresses. You can also specify ports and even [Azure Service Tags](https://learn.microsoft.com/en-us/azure/virtual-network/service-tags-overview). Each rule has a priority and is processed in order. If one rule allows some traffic but another rule of higher priority denies the traffic then traffic is denied because of that priority. There are cases where one rule can allow a range while another rule denies a specific IP address. The traffic allowed or denied is still based on the priority and not specificity.

<!-- Don't forget to mention the VirtualNetowrk service tag -->

Since the NSG is a firewall, how does it compare to Azure Firewall?

The NSG does process rules like the Azure Firewall, however there are three drastic differences. First, the Azure Firewall requires a Public IP address or a Public IP Prefix (IP range). The public IP endpoint(s) are used for the incoming and outgoing traffic. The NSG does *not* have the same kind of entrypoints. More on this later in this post. Second, Azure Firewall has collections and groups of rules each with their own associated priorities and capabilities such as Destination Network Address Translation "DNAT". The NSG only has a single list of rules. Lastly, the Azure Firewall helps route traffic to its destination. The NSG does nothing in regards to routing.bing

<!-- Add image of AZ FW with public IP -->

## NSG and Subnet Association

The NSG can have several rules but none of them are in effect until the NSG is associated to either a subnet of a Virtual Network "Vnet" *and/or* directly to a Network Interface Card "NIC". This association can be confusing so I want to spell this out as clearly as I can. Let's start with a scenario of having a single Virtual Machine "VM". You notice that in order to create the VM a NIC must be created and in the process the NIC must be associated with a subnet in a Vnet. This association to the subnet it what gives the NIC its IP address.

When an NSG is associated with the subnet, the rules then apply to all traffic in that subnet. Of course you can associate an NSG to some or all of the subnets within an Vnet and also to include other subnets of other Vnets. For now, I'm keeping it simple for this explanation.

In an NSG, rules are either inbound or outbound and have a Source and a Destination. If a rule is inbound and has a destination IP of that of an associated subnet, then the inbound network traffic is subjected to this rule. 

{% highlight %}
Inbound
Source = Any
Source Port = Any
Destination = 192.168.0.5
Destination Port = 22
Action = Allow
{% endhighlight %}

<!-- Add image of NSG associated with a subnet -->
<!-- Add image of NSG associated with a NIC -->
