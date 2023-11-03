---
title: Monitor your device
taxonomy:
  category: docs
  label: tutorial
---

!!!!! Requires the Mender Monitor add-on package.
!!!!! See [the Mender features page](https://mender.io/product/features?target=_blank)
!!!!! for an overview of all Mender plans and features.

This tutorial will walk you through how to monitor your device and its application with
Mender. We will be using the [Monitor Add-on](../../09.Add-ons/20.Monitor/docs.md) which
allows you to monitor various parts of your system.

## Prerequisites

To follow this tutorial and perform the examples, you will need to install the [Monitor
Add-on package](../../09.Add-ons/20.Monitor/10.Installation/docs.md) and the [Demo monitors package](../../10.Downloads/docs.md#demo-monitors) on your device. If
you have followed the [get started tutorial](../01.Preparation/docs.md) to prepare your
device, the Monitor Add-on and Demo monitors packages should already be
installed.

> To quickly verify the required dependencies are installed on your device, run
> the following commands:
>
> ```bash
> mender --version && mender-monitorctl --version
> ```
>>```bash
>>3.5.1	runtime: go1.17.13
>>Mender Monitoring control utility v1.3.0
>>
>>Usage: mender-monitorctl command [options]
>>Use "mender-monitorctl help" for more information about available commands and options.
>>```

> To check if the monitoring subsystems are installed, run following command:
>
>```bash
> ls -lAh /etc/mender-monitor/monitor.d | grep -v '^d'
>```
>>```bash
>> -rwxr-xr-x 1 root root 2669 Aug 17 20:09 connectivity.sh
>> -rwxr-xr-x 1 root root 2298 Mar 10 09:37 dbus.sh
>> -rwxr-xr-x 1 root root 2498 Aug 18 01:14 diskusage.sh
>> -rwxr-xr-x 1 root root 3738 Mar 10 09:37 log.sh
>> -rwxr-xr-x 1 root root 3016 Mar 10 09:37 service.sh
>>```
>
> To verify that the example check definitions are installed, please run the following command:
>
>```bash
> ls -lAh /etc/mender-monitor/monitor.d/available/ | grep -v '^d'
>```

>> ```bash
>> -rwxr-xr-x 1 root root 176 Aug 17 20:09 connectivity_example.sh
>> -rwxr-xr-x 1 root root 173 Aug 18 01:14 diskusage_root_space.sh
>> -rwxr-xr-x 1 root root 201 Aug 17 20:09 log_mender_client.sh
>> -rwxr-xr-x 1 root root 203 Aug 17 20:09 log_mender_connect.sh
>> -rwxr-xr-x 1 root root 297 Aug 17 20:09 log_usb_disconnect.sh
>> -rw-r--r-- 1 root root  43 Jun 20 20:06 service_cron.sh
>> ```

## Demo Checks

The Mender Monitor Add-on includes a collection of Demo Checks designed to showcase essential use cases, available through the [Demo monitors package](../../10.Downloads/docs.md#demo-monitors) (Debian package) or in the `examples` directory in case of the [Yocto package](../../05.Operating-System-updates-Yocto-Project/05.Customize-Mender/docs.md#monitor). This is meant as a starting point to try it out. You can also customize and define your own [Checks](../../09.Add-ons/20.Monitor/20.Concepts/docs.md#creating-custom-checks) once you are ready.

Email notifications are automatically generated in addition to UI notifications
shown in the screenshots following the examples. While going through the examples in this
tutorial, watch the email inbox of your Mender user to see that you get
notified about Alerts triggered and cleared on the device. 

! The default configuration for `mender-monitorctl` command requires
! read-write access to the `/etc/mender-monitor` directory, which on most systems
! means the need to switch to super user, or run with `sudo`. 

! For read-only filesystem, it is essential to establish a symbolic link 
! to a writable directory. This symlink is required for the purpose of 
! creating, modifying, and enabling alert check.

### USB disconnect

Peripherals such as keypads and displays are in many cases required to be
connected in order for an IoT product to function properly. This demo Alert
shows how to detect that a USB-connected peripheral gets disconnected.

First, verify an USB peripheral (e.g thumbdrive or mouse) is connected to the device.
Then, enable the `log` check called `usb_disconnect` by running the following command:

```bash
sudo mender-monitorctl enable log usb_disconnect
```

The `log` monitoring subsystem will detect (using the check definition) occurrences of
USB disconnection events within the `/var/log/kern.log` log file.

Now proceed and disconnect the USB peripheral from the device. Once you remove
the USB peripheral, the log monitoring subsystem triggers an alert which you 
can inspect in the device details in the Mender UI:

![Connectivity alarm OK](log-usb-alarm.png)

!!!!! Note: This alert will remain unless a manual [alert cleaning](../../09.Add-ons/20.Monitor/40.Advanced-use-cases/docs.md#alert-cleaning ) is performed:
!!!!! ```
!!!!! cd /usr/share/mender-monitor
!!!!! source lib/monitor-lib.sh
!!!!! monitor_send_alert OK "Log file contains \[.*\] +usb [\w\-\.]+: USB disconnect" "\[.*\] +usb [\w\-\.]+: USB disconnect present in /var/log/kern.log" "log_usb_disconnect" LOGCONTAINS "log" "\[.*\] +usb [\w\-\.]+: USB disconnect" "/var/log/kern.log" 
!!!!! ```

### Disk usage

Running low on key device resources like disk space, often due to growing
log files, can disrupt the product's functionality. These example shows you how to monitor for  high disk usage, allowing timely action to avoid downtime.

First, enable the `diskusage` check for the root space partition called
`root_space` by running the following command:

```bash
sudo mender-monitorctl enable diskusage root_space
```

Then check the current disk usage.

```bash
df -h /

# Output:
# Filesystem      Size  Used Avail Use% Mounted on
# /dev/root       3.4G  1.4G  1.9G  42% /
```

With the `diskusage` monitoring subsystem checking the disk space, it will send 
an alert as soon as the root partition goes above 75% (predefined threshold).

To trigger the alert fill up the filesystem with a large file:

```bash
fallocate -l 10G ~/large-file
```

And the disk usage goes above the 75% treshold

```bash
df -h /

# Output:
# Filesystem      Size  Used Avail Use% Mounted on
# /dev/root       3.4G  3.4G     0 100% /
```

And an alert shows up in the UI.

![Disk usage alarm triggered](diskusage-alarm.png)

An _OK_ alert will be send by the `diskusage` monitoring subsystem if this 
file is removed and the disk space is less than the threshold.

```bash
rm ~/large-file
```


### Connectivity

Ongoing connectivity issues may cause the device application to hang or
malfunction, disrupting the user experience or function of the product. This
Alert is one example how to detect connectivity issues. 

Mender stores triggered Checks on the device. Therefore, even if Mender cannot
send the Checks to the server immediately you will get notified about triggered
Checks later on, once the device regains connectivity. This means that even
during offline periods, Checks are triggered.

First, enable the `connectivity` check called `example` by running the following command:

```bash
sudo mender-monitorctl enable connectivity example
```

This will enable a Check using the demo `connectivity` monitoring subsystem, which sends HTTP HEAD
requests to `example.com`, making sure that it is responding.

To trigger the alert, let us stop the traffic to `example.com` by
redirecting the DNS resolver to localhost in `/etc/hosts`.

```bash
echo ‘127.0.0.1 example.com’ | sudo tee -a /etc/hosts
```

Which then triggers the alert:

![Connectivity alarm triggered](connectivity-alarm.png)

And when re-enabling the route to `example.com` in `/etc/hosts`:

```bash
sudo sed -i '/example/d' /etc/hosts
```

After connection to `example.com` is restored, an _OK_ Alert will show up in the UI:

![Connectivity alarm OK](connectivity-ok.png)

### Docker container restart

!!!!! In this example, Docker Engine must be installed in the device. 
!!!!! Please validate with `docker --version` that it is installed.
!!!!! If it is missing, you can get it by following [Docker's official guide](https://docs.docker.com/engine/install/?target=_blank).


Applications may restart sporadically when they encounter new situations like
intermittent connectivity. As they are often automatically started again the
root cause of this condition may be difficult to detect, and even harder to
diagnose solely based on customer reports.

However, both the situation itself and its underlying causes can be relatively
straightforward to discover by examining either the log or the output generated
by the application. This Alert is designed to identify restarts initiated by the
Docker daemon events and subsequently notify you in cases where a container has
undergone a restart.

First, launch the container we want to monitor:

```bash
sudo docker run --name demo --rm -d alpine sleep infinity
```

Secondly, create the check for the `dockerevents` monitoring subsystem by running:

```bash
sudo mender-monitorctl create dockerevents container_demo_restart demo restart 1d
```

Then, enable the `dockerevents` check called `container_demo_restart` by running 
the following command:
```bash
sudo mender-monitorctl enable dockerevents container_demo_restart
```

Finally, restart the container

```bash
sudo docker container restart demo
```

The `dockerevents` monitoring subsystem should send the alert and it will show up in the UI.

![Dockerevents alarm restarted](dockerevents-alarm.png)

For a complete description of the parameters and the `dockerevents` subsystem
please refer to the [monitoring subsystem](../../09.Add-ons/20.Monitor/20.Monitoring-subsystems/docs.md#docker-events) section.

## Check if the mender-connect systemd service is running

Assume you want to monitor the state of the `mender-connect` systemd service,
and you want to receive _CRITICAL_ alerts if the service is not running,
and _OK_ alerts when it is back up. To get this working, we need to create
a check for the systemd `service` monitoring subsystem using `mender-monitorctl`:

```bash
sudo mender-monitorctl create service mender-connect systemd
```

This will create a file in the directory `/etc/mender-monitor/monitor.d/available`
with the check definitions of the service name and the service type:

```bash
cat /etc/mender-monitor/monitor.d/available/service_mender-connect.sh
```

> ```bash
> # This file was autogenerated by Monitoring Utilities based on the configuration
> SERVICE_NAME="mender-connect"
> SERVICE_TYPE="systemd"
> ```

Then, you can enable the systemd `service` check called `mender-connect` by running
the following command:

```bash
sudo mender-monitorctl enable service mender-connect
```

This command links the file in `/etc/mender-monitor/monitor.d/available` to
`/etc/mender-monitor/monitor.d/enabled`. You can verify it by running:

```bash
readlink /etc/mender-monitor/monitor.d/enabled/service_mender-connect.sh
```

> ```bash
> /etc/mender-monitor/monitor.d/available/service_mender-connect.sh
> ```

The `mender-monitor` daemon will automatically reload the configuration files 
and start the checks.

## Monitor systemd services logs

You can trigger alerts when a given pattern shows in the logs of your systemd service
For this you can create a check definition for to the `log` monitoring subsystem that will
use the `journalctl` command as a source for log data and check for errors or any other pattern.

First, create a check for the systemd `log` monitoring subsystem called `my_service_logs`:

```bash
sudo mender-monitorctl create log my_service_logs "Exited with code \d+" "@journalctl -u my-service -f" 255
```

The above will create a check definition in the following path:

```bash
cat /etc/mender-monitor/monitor.d/available/log_my_service_logs.sh
```

> ```bash
> # This file was autogenerated by Monitoring Utilities based on the configuration
> SERVICE_NAME="my_service_logs"
> LOG_PATTERN="Exited with code \d+"
> LOG_FILE="@journalctl -u my-service -f"
> LOG_PATTERN_EXPIRATION=255
> ```

The `log` monitoring subsystem will trigger a critical alert every time _Exited with code n_
(where `n` is any decimal number) shows up in the logs for `my-service` systemd service.
In the above example, an automatic rest of the critical condition
(sending of an _OK_ alert) will happen after 255 consecutive seconds during which the
pattern does not appear in the logs.
This is what the last and optional argument to `mender-monitorctl` stands for.

For more information please refer
to the [Monitoring subsystems](../../09.Add-ons/20.Monitor/20.Monitoring-subsystems/docs.md) section.

## Receive alerts on new sessions for the root user

To receive alerts on login on the device for the `root` user, we can search for
the pattern `Started User Manager for UID 0` in the `/var/log/auth.log` file. 

First, create a check for the `log` monitoring subsystem named `auth_root_session` by running
the following command:

```bash
sudo mender-monitorctl create log auth_root_session "Started User Manager for UID 0" /var/log/auth.log
```

Executing this command generates a file within the `/etc/mender-monitor/monitor.d/available`
directory, containing the specifications for the targeted log file and the corresponding
pattern to search for. To verify the content of this check defintion, run following command:

```bash
cat /etc/mender-monitor/monitor.d/available/log_auth_root_session.sh
```

> ```bash
> # This file was autogenerated by Monitoring Utilities based on the configuration
> SERVICE_NAME="auth_root_session"
> LOG_PATTERN="Started User Manager for UID 0"
> LOG_FILE="/var/log/auth.log"
> ```

Now you can enable the `log` check called `auth_root_session` by running
the following command:

```bash
sudo mender-monitorctl enable log auth_root_session
```

This command links the file in `/etc/mender-monitor/monitor.d/available` to
`/etc/mender-monitor/monitor.d/enabled`, as you can verify running:

```bash
readlink /etc/mender-monitor/monitor.d/enabled/log_auth_root_session.sh
```

> ```bash
> /etc/mender-monitor/monitor.d/available/log_auth_root_session.sh
> ```

The `mender-monitor` daemon will automatically reload the configuration files and start the checks.

!!! You can also do Perl-compatible regular expressions (PCRE) pattern matching 
!!! for the UID to catch users other than root, using for instance:
!!! `LOG_PATTERN='Started User Manager for UID \d+'`
!!! If your device does not support PCRE, it falls back to -E if available or plain grep if not.
