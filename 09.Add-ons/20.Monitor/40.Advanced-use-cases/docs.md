---
title: Advanced use cases
taxonomy:
    category: docs
    label: user guide
---


## Low level architecture

The `mender-monitor` service supports the following directory structure:


```bash
/etc/mender-monitor
`-- monitor.d
    |-- available
    |   |-- log_auth_root_session.sh
    |   `-- service_mender-connect.sh
    |-- dbus.sh
    |-- enabled
    |   |-- log_auth_root_session.sh -> /etc/mender-monitor/monitor.d/available/log_auth_root_session.sh
    |   `-- service_mender-connect.sh -> /etc/mender-monitor/monitor.d/available/service_mender-connect.sh
    |-- log.sh
    `-- service.sh
```


What is listed under the root level of `monitor.d` are the subsystems (`log.sh`, `dbus.sh` and `service.sh`).

The directory `monitor.d/available` lists the created Checks. By using `create` and `delete`, `mender-monitorctl` will create or delete a Check which means it will create the file in the correct naming convention and define variables within it. The name convention follows the structure:

```
<monitoring_subsystem_name>_<check_name>.sh
```

For examples listed one is a Check for the log subsystem (**log**_auth_root_session.sh) and the other for the service (**service**_mender-connect.sh).


A Check needs to be enabled before it will be taken into consideration. By runing mender-monitorctl with the enable or disable parameters, it will create a symbolic link inside the enabled folder to the right check from the available folder. From this folder the mender-monitor service executes the defined subsystems based on the enabled checks.


## Advanced use cases

The following use cases go beyond the standard usage of mender monitor but are possible to achieve given the tools customisable design.

### Bypassing mender-monitorctl

Since the Checks and subsystems are represented by a directory structure there is an option to modify the files directly instead of using the CLI tool. 


### Using the library 


### Pseudo subsystems


[A] Is this really a thing?


### Writing new subsystems

[A] Examples from the demo can serve as references
