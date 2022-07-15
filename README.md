# Boot2RootCTF: *OSCP - Sar*

*Note: This box was completed long ago and I am going off of the VMware snapshot I saved after completion, some visuals will be missing and explained instead.*

## Objective

We must go from visiting a simple website to having root access over the entire web server.

We'll download the VM from [here](https://www.vulnhub.com/entry/sar-1,425/) and set it up with VMware Workstation 16.

Once the machine is up, we get to work.

## Step 1 - Reconnaissance

After finding our IP address using ```ifconfig``` and locating the second host on the network, we can run an Nmap scan to probe it for information.

```
