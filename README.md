# SGM_upgrade to R81.10

## General

### Prerequisites

Ansible host must have:

- sshpass

## Flow by steps

### 00-complete_upgrade_to_R81.yml

Main playbook to run all steps

### 01-pre_checks.yml

Optional preparations (normally must be already fixed).

- Uncomment sftp subsytem in /etc/ssh/sshd_config
- Fix trailing spaces in /etc/cp-release

### 06-R80_add_JHF.yml

**Step 6 - Install the required Take of the Jumbo Hotfix Accumulator**
First install to logical group B, than to A to avoid extra SMO switches and reconnects.

#### include/install_JHF.yml

**Important:**
`expect` module is missed on Gaia, so run it locally on the Ansible host using `delegate_to: 127.0.0.1`

```
ssh -t ansible@{{ inventory_hostname }} g_clusterXL_admin -b {{ logical_range }} down
include/import_package.yml
include/get_pkg_index.yml
gclish -c "installer verify {{ pkg_index }}"

ssh -t ansible@{{ inventory_hostname }} member {{ logical_member }}
gclish -c \"installer install {{ pkg_index }} member_ids {{ logical_range }}\"
```

Installer terminates by timeout and continues in background. It is OK for manual operations as well.
Reconnect SSH (if SMO changed).
**Expect a very long waiting time (~20 minutes)**

```
[Wait for all Active]
while [ $(asg stat -i sgm_info | grep -c Inactive) -ne 0 ]; do printf "."; sleep 10; done
```

Repeat for logical member A

```
include/06-install_JHF.yml
```

### 07-upgrade_DA_on_sg.yml

**Step 7 - Upgrade the CPUSE Deployment Agent on the Security Group**

```
// download DA package (get_url)
upload (sftp) from Ansible host to SMO
update_sp_da "{{ local_repo }}/{{ da_R80 }}"
```

### 08-import_R81_package.yml

**Step 8 -  Import the R81.10 upgrade package on the Security Group**

```
include/import_package.yml
Get pkg_index
```

### 09-upgrade_R81_B.yml

**Step 9 - Upgrade the Security Group Members in the Logical Group B**

```
include/upgrade_to_R81.yml
```

#### include/upgrade_to_R81.yml

```
ssh -t ansible@{{ inventory_hostname }} g_clusterXL_admin -b {{ logical_range }} down

ssh -t ansible@{{ inventory_hostname }} member {{ logical_member }}
gclish -c \"installer upgrade {{ pkg_index }} member_ids {{ logical_range }}\"
```

// 'gexec ... gclish...' simplification looks not working

Wait in the loop (3 times in case disconnects due to SMO switching)

#### include/wait_for_during_upgrade.yml

Wait for all upgraded PNOTEs have state "FSYNC, during_upgrade"

``` bash
    while [ $(asg stat -i all_ids | wc -l) -ne \
      $(g_all -a cphaprob state | grep "Active PNOTEs" | wc -l) ] || \
      [ $(asg stat -i proc | egrep -c "DOWN|DETACHED") -ne \
      $(g_all -a cphaprob state | grep -c "FSYNC, during_upgrade") ]; \
    do
      printf "."; sleep 10;
    done
```

### 11-mgmt_actions.yml

**Step 11, 12 - Edit the Security Gateway object version, install policy**
**Important**:
It is approved by R&D to make these changes before starting `sp_upgrade`.

There are 2 plays actually.
1) Get policy package from the SGM (SMO)

```
fw stat | tail -1 | awk '{ print $2 }'"
```

2) Modify Management

```
mgmt_cli -u "{{ gui_user }}" -p "{{ gui_pass }}" --format json -d "{{ domain_name }}" /

set simple-gateway name "{{ gw_name }}" version 'R81.10'

publish

install-policy policy-package "{{ policy_name }}" --sync false targets "{{ gw_name }}" prepare-only true
```

### 13-sp_upgrade_B.yml

**Step 13 - sp_upgrade (as one step) on B**

```
ssh -t ansible@{{ inventory_hostname }} member {{ logicalBmember }}
sp_upgrade; exit
// ! simplify using gexec
```

- Y for failover
- N for "Upgrade all Security Group Members"

Wait for "FSYNC, during_upgrade" on local member B.

``` bash
	while [ $(asg stat -i proc | egrep -c "DOWN|DETACHED") -ne \
	$(g_all -a cphaprob state | grep -c "FSYNC, during_upgrade") ];    do
	  printf "."; sleep 10;
	done
```

### 14-upgrade_R81_A.yml

**Step 14 - Upgrade the Security Group Members in the Logical Group A**
Repeat Step 9 (B)

### 15-sp_upgrade_A.yml

**Step 15 - sp_upgrade for A (--continue on B)**

```
ssh -t ansible@{{ inventory_hostname }} member {{ logicalBmember }}
sp_upgrade --continue; exit
```

Wait for all members are active:

``` bash
	while [ $(asg stat -i active_ids | wc -l) -ne \
	$(asg stat -i all_ids | wc -l) ];    do
	  printf "."; sleep 10;
	done
```

### 19-install_R81_policy.yml

**Step 19 - Install policy to all members**
Extra step. To ensure that policy can be installed on the upgraded security group.

### 20-R81-add_JHF.yml

Similar to 06-R80-add_JHF. Simply use {{ jhf_R81 }}.

### 19-install_R81_policy.yml

Repeat once more to ensure policy can be installed after Jumbo installation.
