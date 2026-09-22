# Week 2 Lab 2A — systemd Disk Report

## 1. Final Unit Files

### `/etc/systemd/system/disk-report.service`

[Service]
Type=oneshot
User=reports
ExecStart=/usr/local/bin/disk-report.sh
StandardOutput=append:/var/log/disk-report.log
StandardError=append:/var/log/disk-report.log

/etc/systemd/system/disk-report.timer

[Unit]
Description=Run disk report periodically

[Timer]
OnCalendar=daily
Unit=disk-report.service

[Install]
WantedBy=timers.target


The timer was initially configured with OnCalendar=*:0/5 for testing. It was changed to OnCalendar=daily as required for the final configuration.

2. Initial Journal Error
The first version of the service attempted to write directly to /var/log/disk-report.log from the script while the service was running as the reports user.

The initial journal showed:

/usr/local/bin/disk-report.sh: line 2:
/var/log/disk-report.log: Permission denied

disk-report.service: Main process exited, code=exited, status=1/FAILURE

What the error told me
The important part was:

Permission denied

This showed that the reports user did not have permission to write to /var/log/disk-report.log. The service was configured with User=reports, while the log file had originally been created by root when the script was
tested manually with sudo.

The status=1/FAILURE confirmed that the service failed because the script could not write to the log file.

3. Why Option B is better than Option A
Option B, using StandardOutput=append:/var/log/disk-report.log, is better than Option A, using chown, because the script itself does not need permission to write directly to the log file or the /var/log directory.
Systemd handles opening the log file while the actual service continues running as the unprivileged reports user. This follows the principle of least privilege and also avoids the service breaking if the log file is
deleted and recreated with different ownership. It also keeps the logging destination in the service configuration rather than hard-coding it into the script.

5. Timer
Command:

systemctl list-timers disk-report.timer

Output:

NEXT                        LEFT LAST PASSED UNIT              ACTIVATES
Wed 2026-09-23 00:00:00 UTC  17h -         - disk-report.timer disk-report.service

1 timers listed.
Pass --all to see loaded but inactive timers, too.

5. Successful Service Runs
The lab instructions use:

journalctl -u disk-report.service --no-pager | grep Succeeded

On my Ubuntu system, this produced no output because systemd records successful completion of this Type=oneshot service as Deactivated successfully.

The service status confirms successful execution:

Process: 2304 ExecStart=/usr/local/bin/disk-report.sh (code=exited, status=0/SUCCESS)
Main PID: 2304 (code=exited, status=0/SUCCESS)

Two successful runs from the journal were:

Sep 22 06:01:56 ubuntuserver systemd[1]: disk-report.service: Deactivated successfully.
Sep 22 06:02:03 ubuntuserver systemd[1]: disk-report.service: Deactivated successfully.

These show that the service completed successfully twice.

6. Service Account Security
The reports user was created with --no-create-home and --shell /usr/sbin/nologin because it only needs to run the disk-report service and does not need a home directory or interactive login shell. If those restrictions
were omitted, an attacker who gained control of the service account could potentially use an unnecessary home directory and interactive shell to establish persistence or gain easier interactive access.

verification




oliver@ubuntuserver:~$ systemctl list-timers disk-report.timer
NEXT                        LEFT LAST PASSED UNIT              ACTIVATES
Wed 2026-09-23 00:00:00 UTC  17h -         - disk-report.timer disk-report.service

1 timers listed.
Pass --all to see loaded but inactive timers, too.


oliver@ubuntuserver:~$ systemctl list-timers disk-report.timer
NEXT                        LEFT LAST PASSED UNIT              ACTIVATES
Wed 2026-09-23 00:00:00 UTC  17h -         - disk-report.timer disk-report.service

1 timers listed.
Pass --all to see loaded but inactive timers, too.
oliver@ubuntuserver:~$ systemctl list-timers disk-report.timer
NEXT                        LEFT LAST PASSED UNIT              ACTIVATES
Wed 2026-09-23 00:00:00 UTC  17h -         - disk-report.timer disk-report.service

1 timers listed.
Pass --all to see loaded but inactive timers, too.





oliver@ubuntuserver:~$ sudo cat /etc/systemd/system/disk-report.service
[sudo: authenticate] Password:
[Service]
Type=oneshot
User=reports
ExecStart=/usr/local/bin/disk-report.sh
StandardOutput=append:/var/log/disk-report.log
StandardError=append:/var/log/disk-report.log




oliver@ubuntuserver:~$ sudo cat /etc/systemd/system/disk-report.timer
[Unit]
Description=Run disk report periodically

[Timer]
OnCalendar=daily
Unit=disk-report.service

[Install]
WantedBy=timers.target



oliver@ubuntuserver:~$ 
systemctl is-enabled disk-report.service
systemctl is-enabled disk-report.timer
static
enabled




