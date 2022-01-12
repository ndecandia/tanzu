---
layout: post
title: NSX-T Upgrade to 3.2
comments: false
category: blog
---
# NSX-T Upgrade

## NSX Considerations
Before you begin the upgrade process, it is important to test the NSX-T Data Center working state. Otherwise, you cannot determine if the upgrade caused post-upgrade problems or if the problem existed before the upgrade.



## Reqirement

For Upgrade NSX-T to is important to have Upgrade Coordinator up and running and ensure the Backup SFTP repository is configured for pre and post Backup

## Upgrade Bundle

1. The upgrade bundle contains all the files to upgrade the NSX-T Data Center infrastructure. Before you begin the upgrade process, you must download the correct upgrade bundle version at 3.2.


2. Locate the NSX-T Data Center build on the VMware download portal at 3.2.0.


![MUB download](/images/mub-download.png)


3. Select the NSX 3.2 Upgrade Bundle section and click download now,
the doenloaded file result in .mub format


![linux add host IMG](/images/mub-select.png)

*note* Download the file in a client machine that will be able to connect NSX-T manager for uploading

4. login in the NSX-T Manager GUI
- Go to System → Upgrade→ UPGRADE NSX
- select Upload Local MUB file and click browse
- select from folder the VMware-NSX-upgrade-bundle-3.2.0 mub file downloaded before
- click upload and wait the upload process end

![linux add host IMG](/images/upgrade-button.png)


![linux loading config](/images/upload-load.png)

few minutes and the upload and the check completes:

![linux finish config](/images/upload-completed.png)  

In this moment, the upgrade bundle is uploaded and we need to clic UPGRADE
then accept the EULA and press continue. Waith the Extraction of the bundle.
![linux tn ready](/images/Upgrade-bigbutton.png)

5. Now perform the pre-checks on EDGES, Hosts and NSX-T managers:
- press RUN PRE CHECKS button → All Pre-Checks → RUN PRE CHECKS
- wait until the pre-check process is ended
![precheck](/images/pre-checks.png) 
   
If any problem is noticed you need resolve it before continue,
*note* if the problem is a warning you can check the impact and continue anyway.

![precheck](/images/precheck-result.png) 

6. When is all ready you can continue by Upgrading edges nodes by clicking next.




## Upgrade Edges
In this phase we are going to upgrade the edge clusters, in this case the nodes of a cluster will be upgraded in Serial order.
*note* for upgrading edge is necessary HugePages1gb support on Host CPU

![win if show](/images/edge-upgrade-start.png)


1. press start and wait until the process of upgrading is completed, the edges will be upgraded sequentially.





![PS config](/images/edge-upgrade-compl.png)

Go next.

## Upgrade Hosts

1. Verify that ESXi hosts that are part of a disabled DRS cluster or standalone ESXi hosts are placed in maintenance mode.
For ESXi hosts that are part of a fully enabled DRS cluster, if the host is not in maintenance mode, the upgrade coordinator requests the host to be put in maintenance mode. vSphere DRS migrates the VMs to another host in the same cluster during the upgrade and places the host in maintenance mode.

![download km](/images/drs-fully.png)

Ensure the hosts are able to enter maintenance mode and the Upgrade will be sequentially host by host in a serial way for groups (clusters).


Click start and wait until all hosts vibs are upgraded.

![install km](/images/hosts-upgrade.png)

Then go next

## Upgrade NSX-T Managers

1. The upgrade sequence upgrades the Management Plane at the end. When the Management Plane upgrade is in progress, avoid any configuration changes from any of the nodes.
 
2. Ensure that You have a recent backup of your Management Plane and resolv any conflict that will be noticed.

3. On the Upgrade coordinator panel, in the 4.NSX Manager section click start.
During the nodes Upgrade the GUI page will be lost and few minutes later can be available.

4. To check the upgrade status on nodes, you can launch command in NSX-cli (ssh to node):

```bash
get upgrade progress-status
```

When the process is finished you can click **Done**

![win nsx config](/images/nsx-mgr-finish.png)

At this Time your NSX-T Platform is totally Updated at version 3.2.0!!
*note: is raccomanded to perform a post-upgrade backup*

