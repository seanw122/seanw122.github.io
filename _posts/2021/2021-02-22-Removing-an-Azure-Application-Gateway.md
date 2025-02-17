---
layout: posts
title:  "Removing an Azure Application Gateway"
date:   2021-02-23 15:09:00 +0000
categories: PowerShell, Azure, Application Gateway
---

While working with Terraform scripts I created many Azure Application Gateways. Sometime after they were created I would delete them as they were only needed to prove my scripts were working with Azure DevOps. I was using Terraform functions and special *magic* to get things just right. Then I noticed one of my App Gateways refused to delete.

I was using the Azure Portal as I have done many times. Simply selected the resources, then Delete, applied 'yes' when prompted. After a few minutes they were all gone as expected. One day, one of the App Gateways with the required resources like Public Ip Address, Virtual Network, etc, were still there after an attempt to delete.

Selecting the App Gateway showed the details that included the IP address, version, etc. But, it also showed in a large bar: `Failed`. I have seen it show `Deleting` before but never failed. I selected the Delete option again and after many minutes nothing changed. So, I tried to delete it using PowerShell.

{% highlight powershell %}
$gtw = Get-AzApplicationGateway -Name "dev-example-appgateway"
$gtw
{% endhighlight %}

Executing the two lines above showed the App Gateway's Provisioning State as `Failed` and Operation State as `Stopping`.

I did some research and tried several things:
{% highlight powershell %}
Start-AzApplicationGateway -ApplicationGateway $gtw
Stop-AzApplicationGateway -ApplicationGateway $gtw
Set-AzApplicationGateway -ApplicationGateway $gtw
{% endhighlight %}
Each took a few minutes and either did nothing or gave an error message. One message caught my eye.

    /subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/myResourceGroup1/providers/Microsoft.Network/publicIPAddresses/dev-example-public-ip used by resource

    /subscriptions/ xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx /resourceGroups/ myResourceGroup1/providers/Microsoft.Network/applicationGateways/dev-example-appgateway is not in Succeeded state. Resource is in Failed state.

Looking closely I noticed the issue may not be the App Gateway after all, but a dependent resource; Public Ip Address.

{% highlight powershell %}
$pip = Get-AzPublicIpAddress -Name dev-example-public-ip -ResourceGroupName myResourceGroup1
$pip
{% endhighlight %}
Showing the details of the PIP also showed it was in a failed state.

All this time I thought it was the Application Gateway. So, with this extra knowledge, I tried a different approach. Some of the research suggested executing the Set command with no changes.
{% highlight powershell %}
Set-AzPublicIpAddress -PublicIpAddress $pip
$pip
{% endhighlight %}
It worked! Well, at least for that resource. I tried the Set again for the App Gateway and it didn't show a change. Instead, it gave the same error. Ok, let's try the delete again just to see. Mind you, I have tried this command before including the -Force switch.
{% highlight powershell %}
Remove-AzApplicationGateway -Force -Name dev-example-appgateway -ResourceGroupName myResourceGroup1
{% endhighlight %}
After a few minutes, it simply returned to a prompt. No error message this time. So, I went back to the Azure Portal and refreshed. It worked! Problem solved.

The significance here is that one resource can show a failed state when it's really a dependent resource that is in trouble. I hope this helps someone else and you don't spend the research time like I did.
