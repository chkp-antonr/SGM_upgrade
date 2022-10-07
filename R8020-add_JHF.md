# Install R80.20 Jumbo hotfix

## Issues and specific solutions

### 50-pythonwo.yml

#### /etc/profile

`/etc/profile` is not loaded when Ansible runs commands (in R80.20 only; r=there is no such an issue in R80.30).

``` shell
ssh -tt [ansible@172.23.23.29] sh -c "/opt/CPsuite-R8*/fw1/Python/bin/python"
(Ansible uses this way)

import os
print(os.environ["PATH"])
exit()
```

In R80.30 $PATH includes /opt/CPinfo and others. In R80.20 it is very limited ("/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin")

#### Workaround

A) Force loading `/etc/profile`  by adding BASH_ENV to every playbook

```
  environment:
    BASH_ENV: /etc/profile
```

B) Use a bash script instead of a direct python executable call because BASH_ENV is effective for bash only
`/home/admin/python`

``` shell
#!/bin/bash
/opt/CPsuite-R8*/fw1/Python/bin/python $*
```

And set `ansible_python_interpreter: "/home/admin/python"`

## 8020_jhf-complete_install.yml

Use to run all playbooks from R80.20 JHF installation

### 51-R8020-pre_checks-log.yml

Collects data (clish and expert mode) and stores on the Ansible host.
By default `/tmp/pre_checks.log` and `/tmp/pre_checks.log.prev`

### 56-R8020-add_JHF.yml

Upgrades SMO

- Transfer packages from the Ansible host to SMO
- Import hotfix
- Collect and store some data for validation (licenses, BGP neighbors, etc.) `include/validation_collect.yml`
- set smo image auto-clone state off
- Verify and Install package
  - Use ActionID to display progress.
  - Due to some Ansible limitation I have to "fail" and "resque" to repeat the step. So "fail" status for the task is expected here.
- `include/reconnect_wait_active.yml` several times to handle possible SMO drift
- `include/validate_state.yml`
  - `include/validation_collect.yml`
  - Again failed/resque to retry validation by user request

### 58-R8020-reboot_SGMs.yml

Upgrade SGMs by rebooting one by one.

- set smo image auto-clone state on
- Loop for each SGM: `include/reboot_SGM.yml`
  - `include/validation_collect.yml`
  - Confirm reboot
  - Reboot
  - Wait for all Active
  - `include/validate_state.yml`
- Wait for all Active
- Restart DA on all SGMs (fix an issue)
- Wait for all Active
- Done!
- optionally `51-R8020-pre_checks-log.yml` to collect more detailed info after upgrade
