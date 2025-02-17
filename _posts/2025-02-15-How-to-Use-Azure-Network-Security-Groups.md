---
layout: post
title:  "How to Use Azure Network Security Groups"
date:   2025-02-15 22:38:11 +0000
categories: New blog setup
tags: [Network Security Group, Azure]
---

## Intro

There has been a lot of confusion around Azure's [Network Security Groups](https://learn.microsoft.com/en-us/azure/virtual-network/network-security-groups-overview) "NSG". This post aims to explain details and hopefully alleviate misunderstandings as well as suggest some good ways to use NSGs. A bit of warning, this is not meant to be an introduction to NSGs so much as it is to explain some details and best uses.

## What is an Azure Network Security Group?

First off, let's cover what an NSG really is. They are a type of firewall for layer 3 and 4 traffic. The *layer 3 and 4* is referring to the [Open Systems Interconnection](https://en.wikipedia.org/wiki/OSI_model) model. Layer 3, the Network Layer, is for Internet Protocol "IP" addresses such as 192.168.0.1. Layer 4, the Transport Layer, is for protocols like TCP, UDP, ICMP, and others.

An NSG provides firewall functionality on these two layers. Each rule can work with a single IP address or a range of IP addresses. You can also specify ports and even [Azure Service Tags](https://learn.microsoft.com/en-us/azure/virtual-network/service-tags-overview). Each rule has a priority and is processed in order. If one rule allows some traffic but another rule of higher priority denies the traffic then traffic is denied because of that priority. There are cases where one rule can allow a range while another rule denies a specific IP address. The traffic allowed or denied is still based on the priority and not specificity.

<!-- Don't forget to mention the VirtualNetowrk service tag -->

## How does an NSG compare to Azure Firewall?

The NSG does process rules like the Azure Firewall, however there are three drastic differences. First, the Azure Firewall requires a Public IP address or a Public IP Prefix (IP range). The public IP endpoint(s) are used for the incoming and outgoing traffic. The NSG does *not* have the same kind of entrypoints. More on this [later](#nsg-and-subnet-association) in this post. Second, Azure Firewall has collections and groups of rules each with their own associated priorities and capabilities such as Destination Network Address Translation "DNAT". The NSG only has a single list of rules. Lastly, the Azure Firewall helps route traffic to its destination. The NSG does nothing in regards to routing.

<!-- Add image of AZ FW with public IP -->

## NSG and Subnet Association

The NSG can have several rules but none of them are in effect until the NSG is associated to either a subnet of a Virtual Network "VNet" *and/or* directly to a Network Interface Card "NIC". This association can be confusing so I want to spell this out as clearly as I can. Let's start with a scenario of having a single Virtual Machine "VM". You notice that in order to create the VM a NIC must be created and in the process the NIC must be associated with a subnet in a VNet. This association to the subnet it what gives the NIC its IP address.

When an NSG is associated with the subnet, the rules then apply to all traffic in that subnet. Yes, you can associate an NSG to some or all of the subnets within an VNet and also to include other subnets of other VNets. For now, I'm keeping it simple for this explanation.

In an NSG, rules are either inbound or outbound and have a Source and a Destination. If a rule is inbound and has a destination IP of that of an associated subnet, then the inbound network traffic is subjected to this rule. Consider this rule in our example.

{% highlight verilog %}
Priority = 1001
Direction = Inbound
Source = Any
Source Port = Any
Destination = 192.168.0.5
Destination Port = 22
Action = Allow
{% endhighlight %}

This rule is stating that traffic coming from *Any* source that is destined to *192.168.0.5* on port *22* will be allowed. Now let's add another rule. This time the rule is to deny all traffic bound to the same destination regardless of destination port.

{% highlight verilog %}
Priority = 1000
Direction = Inbound
Source = Any
Source Port = Any
Destination = 192.168.0.5
Destination Port = Any
Action = Deny
{% endhighlight %}

The problem with this is that no traffic is getting to any port on that IP address. This is because the priority is set to a higher priority than the other rule allowing traffic to port 22. Priority 1000 is higher than 1001.

> **Note:** NSG rules have a priority from 100 to 4096. The lower the number, the higher the priority!

So, for traffic to successfully reach *192.168.0.5* on port *22* we need to swap the priority.

{% highlight verilog %}
Priority = 1000
Direction = Inbound
Source = Any
Source Port = Any
Destination = 192.168.0.5
Destination Port = 22
Action = Allow

Priority = 1001
Direction = Inbound
Source = Any
Source Port = Any
Destination = 192.168.0.5
Destination Port = Any
Action = Deny
{% endhighlight %}

Remember, the rules regarding the destination address of *192.168.0.5* are only applicable if the NSG is associated to the subnet range that includes that address, OR is directly associated to the NIC that has that address.

<!-- Add image of NSG associated with a subnet -->
<!-- Add image of NSG associated with a NIC -->

## Direction of Rules

When creating a rule we need to understand where the traffic is originating and where it is destined. For example, if traffic is coming from the internet to a public IP address then the rule will be Inbound likely with a Service Tag of "Internet". This Inbound rule might have a source of a collection of public IP addresses that are from business partners or specific people. Perhaps the traffic originated with a private IP address.

Let's go through a quick example of a VM with a public IP address. You associate the NSG directly to the NIC of the VM. A quick note; this [not advised](#associating-an-nsg-to-nics) and I explain further down. The expectation is to allow yourself direct access from your home. Since the traffic will originate from your home we need to create an Inbound rule. For this example, let's assume that your home IP address is 1.2.3.4 and the IP address of the VMs public IP is 5.6.7.8. Create the rule as shown below and test.

{% highlight verilog %}
Priority = 2000
Direction = Inbound
Source = 1.2.3.4
Source Port = Any
Destination = 5.6.7.8
Destination Port = 22
Action = Allow
{% endhighlight %}

Did you notice it didn't work? Most of the details are correct except for one. This is a big item to remember. Public IP address on a NIC are first converted to their private IP address before an NSG rule is applied. So, we need to change the rule to have the destination of the private IP address.

{% highlight verilog %}
Priority = 2000
Direction = Inbound
Source = 1.2.3.4
Source Port = Any
Destination = 192.168.0.5
Destination Port = 22
Action = Allow
{% endhighlight %}

What if the traffic is originating from code running on the VM and needs to get to a public endpoint in the internet? For this, we'll create an Outbound rule. Why outbound? Because the traffic originates from inside the VNet it's direction is outbound. It doesn't make sense to have an Outbound rule with a Source of Internet so that rule will not be effective.

An Outbound rule is created with a source IP of the NIC. The destination is to some random public IP for this example. Notice the destination port is 443 but the source port is *Any*. When web-based traffic occurs, it first starts with an agreement to further the conversation on an open source port. It's rare that you will set a source port. However, there are those special cases when there is tight control over that traffic. Web-based traffic is of the few types of conversations that leverage other open ports.

{% highlight verilog %}
Priority = 2000
Direction = Outbound
Source = 192.168.0.5
Source Port = Any
Destination = 15.16.17.18
Destination Port = 443
Action = Allow
{% endhighlight %}

Wait! Don't we already have a rule with that priority? Yes, we do. However, the list of Inbound and Outbound rules are independent of each other. So, it's ok to have matching rule priority numbers as the are not conflicting.

> **Tip:** You can get your public IP address by executing `curl ifconfig.me`

## Associating an NSG to NICs

While you technically can associate an NSG to a NIC it's not advised. If you have a rule that is applicable to an associated subnet and another to an associated NIC, you do *not* get more security. The reason it's advised to not associate an NSG to a NIC is simply to due to management of the rules. In a larger environment, you're more likely to want to apply rules that affect multiple NICs on the same subnet. This is much easier than creating multiple rules that seem like duplicates each to a single NIC.

> **Note:** Have you noticed that I refer to NICs and not VMs? This is because the NSG rules apply to all NICs even those of Private Endpoints.

## Reply Traffic

The NSG is stateful meaning that it will allow (until traffic is idle long enough) a reply to a request without a rule for the opposite direction. For example, calls from a VM out to some location on the internet will have replies allowed in.

## A Dirty Little Secret

When I was first learning about Azure networking I thought an NSG provided something like a perimeter around my VMs. Having a rule to deny traffic to a subnet meant I was scrutinizing the traffic to those VMs and that those VMs would be allowed to openly talk to each other. This is not the case at all!

The truth is, there is no virtual network! Every bit of information about VNet, subnet, routing, DNS, and NSG rules are explicitly applied to all of the NICs of the associated subnet(s). So when an NSG rule is for a subnet, it's really being applied to each NIC on that subnet! Blocking SSH in a rule to a subnet means each NIC in that subnet has that block.

In reality, the NICs are simply programs running on an [FPGA](https://en.wikipedia.org/wiki/Field-programmable_gate_array)  chip. Even the server that is "virtual" is made to believe, at the OS level, that the NIC is a real piece of hardware. It's just code.

This also explains why you cannot simply move a VM to another VNet. You can change association to another subnet but to move to another VNet requires several underlying things to be recreated. At this time, it's just not possible.

<!-- ## NSG Design - Building Tight Network Security -->

<!-- Scenarios

talk about 5 tuple source, source port, destination, destination port, and protocol

NSG to one Subnet same Vnet
NSG to 1+ Subnet same Vnet
NSG to 1+ Subnet 1+ Vnets
NSG 1 - Subnet 1 - NSG 2
NSG 1 - NIC 1 - NSG 2
NSG 1 - NIC 1 - NSG 2 

Outbound port being same as inbound port

VM public IP to private IP then to NSG
-->