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
In this example, we will guide you through the necessary steps to create a new subsystem that monitors disk usage on the device.

First, let us start by creating the subsystem file:

```bash
cat >/etc/mender-monitor/monitor.d/diskusage.sh <<EOF
#!/usr/bin/env bash
# Copyright 2022 Northern.tech AS
#
#    All Rights Reserved
#
#
# Monitor the disk space of a given disk.
#
#
# More specifically
#
# DISKUSAGE_NAME=<some name>
# DISKUSAGE_THRESHOLD=<1-100> (default: 80)
#
EOF
```

In the file, it is necessary to source the `mender-monitor` library to enable the required functions for interacting with the Mender Server:

```bash
. common/common.sh
. lib/monitor-lib.sh
. lib/alert-lib.sh
. lib/subsystem-storage-lib.sh
```

Next, we need to validate the input it may require. In this example, we will only validate the `DISKUSAGE_NAME` variable:

```bash
#
# Parse the input
#
if [[ -z "${DISKUSAGE_NAME}" ]]; then
    log_error "DISKUSAGE_NAME not set, this is an error."
    exit 0
fi
```

The definition of the actual command or logic that the subsystem is going to monitor comes next:

```bash
function disk_usage() {
    df --output=pcent ${DISKUSAGE_NAME} | tail -1 | cut -d% -f1
}
```

It is important to remember that each check will have a unique key to store its last alarm status. To retrieve the check name used to source the subsystem, we can use the following function:

```bash
function get_monitor_name() {
    local -r monitor_name=$(basename "${env}")
    local -r strip_subsystem_name=${monitor_name//diskusage_/}
    echo ${strip_subsystem_name%.sh}
}
```

Finally, we define how and when the _OK_ and _CRITICAL_ alerts are generated and send to the Mender Server. 
In this case we check if the `DISKUSAGE_USAGE` exceeded the `DISKUSAGE_THRESHOLD` value, in case it does, it will send the _CRITIAL_ alert using the function `monitor_send_alert` from the `mender-monitor` library. When the `DISKUSAGE_USAGE` comes down and it is no longer exceeding, it will send the _OK_ alert.

```bash
CONNECTIVITY_MONITOR_KEY=$(get_monitor_name)

DISKUSAGE_USAGE=$(disk_usage)

if [[ ${DISKUSAGE_USAGE} -gt ${DISKUSAGE_THRESHOLD:-80} ]]; then
    log_debug "Disk storage has grown to fill more than ${DISKUSAGE_THRESHOLD:-80}% of the disk"
    if [[ $(subsystem_get "${SUBSYSTEM_NAME}" "LAST_ALARM_${CONNECTIVITY_MONITOR_KEY}") != CRITICAL ]]; then
        log_debug "Disk storage alarm ready to send CRITICAL"
        monitor_send_alert \
            CRITICAL \
            "Disk storage has grown to fill more than ${DISKUSAGE_THRESHOLD:-80}% of the disk '${DISKUSAGE_NAME}'" \
            "Disk ${DISKUSAGE_NAME} is now at ${DISKUSAGE_USAGE}% capacity, above the ${DISKUSAGE_THRESHOLD:-80}% threshold" \
            "${DISKUSAGE_NAME}" \
            DISKUSAGE_USAGE_WARNING \
            "disk"
        subsystem_set "${SUBSYSTEM_NAME}" "LAST_ALARM_${CONNECTIVITY_MONITOR_KEY}" CRITICAL
    else
        log_debug "The disk usage is too high, but the alarm CRITICAL is already sent"
    fi
else
    if [[ $(subsystem_get "${SUBSYSTEM_NAME}" "LAST_ALARM_${CONNECTIVITY_MONITOR_KEY}") == CRITICAL ]]; then
        log_debug "Disk storage alarm send OK"
        monitor_send_alert \
            OK \
            "Disk storage has grown to fill more than ${DISKUSAGE_THRESHOLD:-80}% of the disk '${DISKUSAGE_NAME}'" \
            "Disk ${DISKUSAGE_NAME} is now at ${DISKUSAGE_USAGE}% capacity, back below the ${DISKUSAGE_THRESHOLD:-80}% threshold" \
            "${DISKUSAGE_NAME}" \
            DISKUSAGE_USAGE_WARNING \
            "disk"
        subsystem_set "${SUBSYSTEM_NAME}" "LAST_ALARM_${CONNECTIVITY_MONITOR_KEY}" OK
    else
        log_debug "Disk usage is fine, and no need to send alarm OK"
    fi
fi
```

After creating the subsystem, let us proceed to create a new check named `root_space` to monitor the disk usage of the root directory mounted at `/`:

```bash
cat >/etc/mender-monitor/monitor.d/available/diskusage_root_space.sh <<EOF
# Copyright 2022 Northern.tech AS
#
#    All Rights Reserved

#
# Monitor the whole rootfs space usage
#
DISKUSAGE_NAME="/"
# Report on 3/4 full disk
DISKUSAGE_THRESHOLD=75
EOF
```

To enable the check, you can do so by running the following command:

```bash
mender-monitorctl enable diskusage root_space
```