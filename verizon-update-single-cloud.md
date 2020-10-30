# Update single cloud
This procedure shows how to update a single cloud.  

# Prerequisites
## On the subcloud - create a temporary patch directory
```
mkdir ~/soft_panic_patch
```

## Copy patch to subcloud
Using scp or some other means copy the patch to the subcloud to be updated for example
```
cloud=172.25.55.55
scp WRCP_20.06_DEV_0005_SOFT_PANIC.patch sysadmin@$cloud:soft_panic_patch/.
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

## Verify networking is setup (Unverified)
```
echo "ip link show ens3f1";ip link show ens3f1 | tee -a $(source /etc/platform/openrc;system show | grep "region_name " | awk '{print $4}')-prerequisites.log
echo "ip link show enp179s0f0";ip link show enp179s0f0 | tee -a $(source /etc/platform/openrc;system show | grep "region_name " | awk '{print $4}')-prerequisites.log
echo "ip link show enp181s0f0";ip link show enp181s0f0 | tee -a $(source /etc/platform/openrc;system show | grep "region_name " | awk '{print $4}')-prerequisites.log
```

## Verify pod networking is up (Unverified)
```
kubectl get pods -A | grep dpp | awk '{print $1 " " $2}'  | xargs -n1 -I '{}'  sh -c 'echo {};kubectl exec -n {} -- ip link show' | tee -a $(source /etc/platform/openrc;system show | grep "region_name " | awk '{print $4}')-prerequisites.log 
kubectl get pods -A | grep rmp | awk '{print $1 " " $2}'  | xargs -n1 -I '{}'  sh -c 'echo {};kubectl exec -n {} -- ip link show' | tee -a $(source /etc/platform/openrc;system show | grep "region_name " | awk '{print $4}')-prerequisites.log
kubectl get pods -A | grep dip | awk '{print $1 " " $2}'  | xargs -n1 -I '{}'  sh -c 'echo {};kubectl exec -n {} -- ip link show' | tee -a $(source /etc/platform/openrc;system show | grep "region_name " | awk '{print $4}')-prerequisites.log
```

# Install Patch
## On the subcloud - Upload the patch
```
sudo sw-patch upload ~/soft_panic_patch/WRCP_20.06_DEV_0005_SOFT_PANIC.patch
    WRCP_20.06_DEV_0005_SOFT_PANIC is now available
```

## On the subcloud - Apply the patch
```
sudo sw-patch apply --all
    WRCP_20.06_DEV_0005_SOFT_PANIC is now in the repo
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