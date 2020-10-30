# Update single cloud
This procedure shows how to remove designer patch 2 (dp2) and apply designer patch 3 (dp3) 

# Prerequisites
## On the subcloud - create a temporary patch directory for dp3
```
mkdir ~/dp3
```
## Copy dp3 to subcloud
Using scp or some other means copy the patch to the subcloud to be updated for example
```
cloud=172.25.55.55
scp WRCP_20.06_DEV_0005_APP_ISOLCPU.patch sysadmin@$cloud:ISOLCPU_dp3/.
```

## Check alarms
Check alarms and verify that all alarms are cleared
```
fm alarm-list | tee $(source /etc/platform/openrc;system show | grep "region_name " | awk '{print $4}')-prerequisites.log
```

## Verify pods are running and in good health
```
kubectl get pods -A | tee -a $(source /etc/platform/openrc;system show | grep "region_name " | awk '{print $4}')-prerequisites.log
```
# Uninstall dp2
## On the subcloud - remove the patch
```
sudo sw-patch remove WRCP_20.06_DEV_0005_RTPRIO
system host-lock controller-0
sudo sw-patch host-install controller-0
system host-unlock controller-0
```
## After reboot on unlock -  delete the patch
```
sudo sw-patch delete WRCP_20.06_DEV_0005_RTPRIO
```

# Install dp3
## On the subcloud - Upload the patch
```
sudo sw-patch upload ~/dp3/WRCP_20.06_DEV_0005_APP_ISOLCPU.patch
    WRCP_20.06_DEV_0005_APP_ISOLCPU is now available
```

## On the subcloud - Apply the patch
```
sudo sw-patch apply --all
    WRCP_20.06_DEV_0005_APP_ISOLCPU is now in the repo
```

## On the subcloud - Install on each node
For all nodes in the cloud do the following.
```
system host-lock controller-0
sudo sw-patch host-install controller-0
system host-unlock controller-0
```

- <b><u>There is an existing open issue where sometimes the system will not reboot on an unlock.  In order to verify the system reboot open ILO to verify the host is rebooting.</b></u>

# Verify the patch was installed
## On the subcloud - Verify the patch is installed
```
sw-patch query | tee $(source /etc/platform/openrc;system show | grep "region_name " | awk '{print $4}')-post-patch-verification.log
Password:            Patch ID             RR  Release  Patch State
==============================  ==  =======  ===========
WRCP_20.06_DEV_0005_SOFT_PANIC  Y    20.06     Applied
WRCP_20.06_PATCH_0001           Y    20.06    Committed
WRCP_20.06_PATCH_0002           Y    20.06    Committed
WRCP_20.06_PATCH_0003           N    20.06     Applied
WRCP_20.06_PATCH_0004           N    20.06     Applied
WRCP_20.06_PATCH_0005           Y    20.06     Applied

sw-patch query-hosts | tee -a $(source /etc/platform/openrc;system show | grep "region_name " | awk '{print $4}')-post-patch-verification.log
  Hostname     IP Address    Patch Current  Reboot Required  Release  State
============  =============  =============  ===============  =======  =====
controller-0  192.168.220.3       Yes             No          20.06   idle
```

# Instructions for setting isolated cpus with designer patch #3
The designer patch WRCP_20.06_DEV_0005_APP_ISOLCPU.patch disables the
normal handling of application-isolated CPUs and allows the user to
allocate them to containers exactly like regular CPUs.

# For the subclouds; ensure the platform CPUs are already configured:
## Make sure the platform cores are already configured, For AIO systems, there will be either 2 if HT is turned off and 4 if HT is turned on.
```
[sysadmin@controller-0 ~(keystone_admin)]$ system host-cpu-list controller-0 | grep Platform
| 6ea65da5-d8dd-4e89-b7f9-73f1a8b2ed3a | 0     | 0         | 0     | 0      | Intel(R) Xeon(R) Gold 6212U CPU @ 2. | Platform          |
| 6e86ab61-2639-4bc0-af9b-f8f63e04c099 | 1     | 0         | 1     | 0      | Intel(R) Xeon(R) Gold 6212U CPU @ 2. | Platform          |
| 1808c54c-8de4-4336-ad57-f743199d7ca8 | 24    | 0         | 0     | 1      | Intel(R) Xeon(R) Gold 6212U CPU @ 2. | Platform          |
| 0dd46292-631a-4cd2-ba23-7fbc5f108c4e | 25    | 0         | 1     | 1      | Intel(R) Xeon(R) Gold 6212U CPU @ 2. | Platform          |
```

-  Confirm there are four CPUs (or 2 cores if hyperthreading is disabled) with an "assigned_function" of "Platform"
-  Confirm all other CPUs have an "assigned_function" of "Application"

# Steps to assign one or more application cores to isolcpu
1) The subcloud must first be locked.  ssh onto the subcloud and then:
```
system host-lock controller-0
```
2) Disable the WRCP isolcpu handling feature for Kubernetes application-isolated CPUs.  Enable special isolcpu handling that is provided in the patch:
```
system host-label-assign controller-0 kube-ignore-isol-cpus=enabled
```
> <b>Note: With this label applied, "application-isolated" CPUs will now be allocated to pods in the same way as regular CPUs. There's no differentiation in kubelet.</b>

3) Identify which CPUs that you want to be isolated.  Specify them via:
```
system host-cpu-modify -f application-isolated -c <range spec> controller-0
```
- The syntax for the range specification is the same as for the "-c" option of the "taskset" command. 
- For example, "-c 3,5,20-23" would set CPUs 3,5,20,21,22,23 as application-isolated. 
- The "-c" option must only be used when specifying the application-isolated CPUs, and it must be done after the platform CPUs have already been configured.

4) Confirm the desired CPUs now have an "assigned_function" of "Application-isolated".  This can be seen from the output of the command run in step #3.  Also the same output can be seen by running:
```
system host-cpu-list controller-0
```
-  Confirm the desired CPUs have an "assigned_function" of "Application-isolated"
-  Confirm no change to the platform CPUs and that they still have an "assigned_function" of "Platform"

5) Unlock the subcloud:
```
system host-unlock controller-0
```

6) After the subcloud has rebooted, confirm the new label is applied
```
system host-label-list controller-0
+--------------+--------------------------+-------------+
| hostname     | label key                | label value |
+--------------+--------------------------+-------------+
| controller-0 | kube-ignore-isol-cpus    | enabled     |
+--------------+--------------------------+-------------+
```

7) After the subcloud has rebooted, confirm that Kubernetes can see all the application CPUs as "Allocatable". Run the describe command and look for the number of "cpu" under the "Allocatable" section of the output
```
kubectl describe node controller-0
```
- The number of "cpu" under the "Allocatable" section should equal the total number of CPUs minus the platform CPUs.  So for a system with 48 CPUs where 4 have an "assigned_function" of platform you should see cpu = 44.  For example:

```
[sysadmin@controller-0 ~(keystone_admin)]$ kubectl describe node controller-0 | grep 'cpu:' -B 1
Capacity:
  cpu:                           48
--
Allocatable:
  cpu:                           44
```
> Note: Allocatable values will be different if Intel Hyperthreading is off.

8) Confirm the linux command line is updated to include the CPUs you set in step 3.  Look for ISOLCPUs on the command line.
```
cat /proc/cmdline
```

- This can be run from the System Controller via ssh.  For example `ssh <subcloudname> sw-patch query`

# Additional verification steps
## Check alarms
```
fm alarm-list | tee -a  $(source /etc/platform/openrc;system show | grep "region_name " | awk '{print $4}')-post-patch-verification.log
```

## Make sure all pods are running
```
kubectl get pods -A | tee -a $(source /etc/platform/openrc;system show | grep "region_name " | awk '{print $4}')-post-patch-verification.log

```
## Make sure the SRIOV PCI device is properly configured
View the below file and make sure that both iavf and vfio-pci drivers listed
```
cat /etc/pcidp/config.json | tee -a $(source /etc/platform/openrc;system show | grep "region_name " | awk '{print $4}')-post-patch-verification.log
```

## Verify the new command line
Look at the cmd lines for 'softlockup_panic=1'
```
cat /proc/cmdline | tee -a $(source /etc/platform/openrc;system show | grep "region_name " | awk '{print $4}')-post-patch-verification.log
```

## Verify networking is setup (Unverified)
```
echo "ip link show ens3f1";ip link show ens3f1 | tee -a $(source /etc/platform/openrc;system show | grep "region_name " | awk '{print $4}')-post-patch-verification.log
echo "ip link show enp179s0f0";ip link show enp179s0f0 | tee -a $(source /etc/platform/openrc;system show | grep "region_name " | awk '{print $4}')-post-patch-verification.log
echo "ip link show enp181s0f0";ip link show enp181s0f0 | tee -a $(source /etc/platform/openrc;system show | grep "region_name " | awk '{print $4}')-post-patch-verification.log
```

## Verify pod networking is up (Unverified)
```
kubectl get pods -A | grep dpp | awk '{print $1 " " $2}'  | xargs -n1 -I '{}'  sh -c 'echo {};kubectl exec -n {} -- ip link show' | tee -a $(source /etc/platform/openrc;system show | grep "region_name " | awk '{print $4}')-post-patch-verification.log
kubectl get pods -A | grep rmp | awk '{print $1 " " $2}'  | xargs -n1 -I '{}'  sh -c 'echo {};kubectl exec -n {} -- ip link show' | tee -a $(source /etc/platform/openrc;system show | grep "region_name " | awk '{print $4}')-post-patch-verification.log
kubectl get pods -A | grep dip | awk '{print $1 " " $2}'  | xargs -n1 -I '{}'  sh -c 'echo {};kubectl exec -n {} -- ip link show' | tee -a $(source /etc/platform/openrc;system show | grep "region_name " | awk '{print $4}')-post-patch-verification.log
```

# Notes
- After the update the System Controller will show the subcloud is out of patch sync.  This is expected
```
dcmanager subcloud show tm0-subcloud
    +-----------------------------+----------------------------+
    | Field                       | Value                      |
    +-----------------------------+----------------------------+
    | id                          | 6                          |
    | name                        | tm0-subcloud               |
    | description                 | HPE TM0                    |
    | location                    | SOHO eraineri              |
    | software_version            | 20.06                      |
    | management                  | managed                    |
    | availability                | online                     |
    | deploy_status               | complete                   |
    | management_subnet           | 192.168.220.0/24           |
    | management_start_ip         | 192.168.220.2              |
    | management_end_ip           | 192.168.220.15             |
    | management_gateway_ip       | 192.168.220.1              |
    | systemcontroller_gateway_ip | 192.168.221.1              |
    | group_id                    | 1                          |
    | created_at                  | 2020-09-26 16:51:11.265632 |
    | updated_at                  | 2020-10-22 03:35:49.879175 |
    | dc-cert_sync_status         | in-sync                    |
    | firmware_sync_status        | in-sync                    |
    | identity_sync_status        | in-sync                    |
    | load_sync_status            | in-sync                    |
    | patching_sync_status        | out-of-sync                |
    | platform_sync_status        | in-sync                    |
    +-----------------------------+----------------------------+
```

- <b>Once a subcloud is patched this way it will show not in  sync; using 'dcmanager patch-strategy' will remove the patch installed manually</b>
