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


What is listed under the root level of `monitor.d` are the Subsystems (`log.sh`, `dbus.sh` and `service.sh`).

The directory `monitor.d/available` lists the created Checks. By using `create` and `delete`, `mender-monitorctl` will create or delete a Check which means it will create the file in the correct naming convention and define variables within it. The name convention follows the structure:

```
<monitoring_subsystem_name>_<check_name>.sh
```

For examples listed one is a Check for the log Subsystem (**log**_auth_root_session.sh) and the other for the service (**service**_mender-connect.sh).


A Check needs to be enabled before it will be taken into consideration. By runing `mender-monitorctl` with the `enable` or `disable` parameters, it will create a symbolic link inside the `enabled` folder to the right check from the `available` folder. From this folder the `mender-monitor` service executes the defined Subsystems based on the enabled Checks.


## Advanced use cases

The following use cases extend beyond the typical usage of Mender Monitor, but they are attainable due to the tool's customizable design.

### Bypassing mender-monitorctl

Since the Checks and Subsystems are represented by a directory structure there is an option to modify the files directly instead of using the CLI tool. 

### Using the library 
To use the Mender Monitor library, first you need to source the enviroment with the function set provided to interact with the Mender Server and Monitor logic.

```bash
cd /usr/share/mender-monitor
source lib/monitor-lib.sh
```

Once the enviroment is sourced, there will be new functions available to use. For example `monitor_send_alert`, this function is used to send the alert data (_OK_ or _CRITICAL_) to the Mender Server.

This function takes the following parameters:

```bash
monitor_send_alert "alert_type" "alert_description" "alert_details" "subject_name" "subject_status" "subject_type" "log_pattern" "log_file_path" "lines_before" "line_matching" "lines_after"
```

**Sending a OK alert**

By sending an _OK_ alert you can clean your alert level. Assuming you did not implement it on your
subsystem, then you can force it by running a command similar to the one below (assuming a _service subsystem_):

```bash
SERVICE_NAME = "your-service-name"
monitor_send_alert OK "Service ${SERVICE_NAME} running" "The main process is present again" "${SERVICE_NAME}" "running" "service"
```

### Writing new Subsystems

[A] Examples from the demo can serve as references
