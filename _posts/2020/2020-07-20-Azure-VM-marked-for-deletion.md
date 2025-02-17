---
layout: posts
title:  "Azure VMs Marked for Deletion"
date:   2021-02-23 15:09:00 +0000
categories: Terraform, Azure, Virtual Machines
---

## Are You Serious?!?

The other day I was building out two Virtual Machines using an ARM template. I had been building many other VMs with no issues. Now, I have a failure. And two other VMs had issues being created. All four VMs had a Provision Status of `Failed`.

I checked the quota and determined that was not the issue. Looking at the failure message of each VM only said a failure occurred and provided no real details.

## Why are you here?

This was ultimately fixed. This post is about a few things that occurred that allowed me to get these VMs out of a failed state.

The two VMs built using an ARM template worked successfully in another region. That other region had Availability Zones where the region with the failure did not. I did modify the ARM template to use Availability Sets instead.

The other two VMs were created by hand by a coworker. The steps used worked on other VMs in the same region. So, why the failures at all? Unfortunately, I don’t know. Fluke in the “matrix” for all I know. In those cases, were still stuck with determining the next step.

## The Right SKU

One suggestion worked on the two VMs not created by the ARM template. I changed the size of the VM. They went from “Standard_F4s_v2” to “Standard_F4s”. After the size change, I was able to start the VMs. A coworker then confirmed he was able to install needed software.

I tried to change the size on the VMs created by the ARM template. But it failed to work. Why? Because I tried to delete the VMs so I could start over. That actually hurt me. I didn’t yet know about attempting to change the VM size. So, what finally fixed it for me?

## Finally, My Answer

This info was what I got from a Microsoft ticket. The first round of discussions with them resolved nothing. It wasn’t until they realize another piece of my puzzle that they gave a different suggestion.

Our VMs were in Availability Sets. That alone is not an issue. The trick was to stop each VM in the Availability Sets. Once each VM was stopped I retried the delete of the problematic VMs. **IT WORKED!!** I was finally able to remove those VMs and recreate them.

Once recreated I then went to each of the other VMs and started them. They all come back online. **Eureka!!!**

Hopefully, this extra tidbit of detail will help you. If a VM fails on creation, don’t delete. Instead, try to change the VM size. If that fails, look at what else help dictate how a VM is allocated. For me, the Availability Set was the extra detail. Stopping all the VMs in an Availability Set allowed me to delete the problematic VMs and recreate them.
