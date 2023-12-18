# grafana-selinux

Grafana selinux policy module for CentOS 7 and RHEL 7. Recently updated to CentOS Stream 9 - Grafana 10.0.3-1.

At present, from my testing this should be working for all basic functions of Grafana. This hasn't been extensively tested(at present. I will change this when I do) by me and should be considered in an early beta state.

The policy assumes that you used the rpm from Grafana to install it. Thus all the file locations should adhere to the rpm or make sure to change them as needed.


### Untested:
* Anything outside the scope of using it at the most basic level - keep an eye on that AVC log!

### Future considerations:
* Add a grafana_plugin_t label to contain plugins.


## Prerequisites (Needed before installing)
Ensure you have the `selinux-policy-devel` package installed.
```sh
# Ensure you have the devel packages
sudo dnf install git selinux-policy-devel setools-console

# make sure we're at home for this
cd ~

# clone the repo
git clone https://github.com/alexwoellhaf/grafana-selinux.git

# cd to the source directory
cd grafana-selinux
```

## Installation
```sh
# Copy relevant .if interface file to /usr/share/selinux/devel/include to expose them when building and for future modules.
# May need to use full path for grafana.if if not working.
sudo install -Dp -m 0664 -o root -g root grafana.if /usr/share/selinux/devel/include/myapplications/grafana.if

# Compile the selinux module
sudo make -f /usr/share/selinux/devel/Makefile grafana.pp

# Install the SELinux policy module. Compile it before hand to ensure proper compatibility 
semodule -i grafana.pp

# Add grafana port
sudo semanage port -a -t grafana_port_t -p tcp 3000

# Add influxdb port
sudo semanage port -a -t influxdb_port_t -p tcp 8086

# Restore all the correct context labels
sudo restorecon -RvF /usr/sbin/grafana-* \
		/etc/grafana \
		/var/log/grafana \
		/var/lib/grafana \
		/usr/share/grafana
```

## Debugging and Troubleshooting

* If you're getting permission errors, uncomment permissive in the .te file and try again. Re-check logs for any issues. Or `semanage permissive -a grafana_t`
* Easy way to add in allow rules is the below command, then copy or redirect into the .te module. Rebuild and re-install:
* Don't forget to actually look at what is suggested. audit2allow will most likely go for a coarse grained permission!

```sh
ausearch -m avc,user_avc,selinux_err -ts recent | audit2allow -R
```
If you get a could not open interface info [/var/lib/sepolgen/interface_info] error. 
Ensure policycoreutils-devel is installed and/or run: `sepolgen-ifgen`

## Compatibility Notes
Tested with CentOS Stream release 9 running on an AWS EC2 Instance on 2023-12-17:
```
selinux-policy.noarch 38.1.27-1.el9
selinux-policy-devel.noarch 38.1.27-1.el9
selinux-policy-targeted.noarch 38.1.27-1.el9

setools-console.x86_64 4.4.3-1.el9

kernel: Linux 5.14.0-229.el9.x86_64
```
### A Bug
My box exhibits the following from ausearch - im not sure what needs to be added to fix this at the moment, and I don't use redis currently so this will be on the back burner for the meantime
```
$ sudo ausearch -m avc,user_avc,selinux_err -ts recent
----
..
..
..
arch=c000003e syscall=59 success=no exit=-13 a0=c000e87e60 a1=c000c76dc0 a2=c000d7afc0 a3=0 items=0 ppid=74260 pid=74267 auid=4294967295 uid=991 gid=991 euid=991 suid=991 fsuid=991 egid=991 sgid=991 fsgid=991 tty=(none) ses=4294967295 comm="grafana-server" exe="/usr/sbin/grafana-server" subj=system_u:system_r:grafana_t:s0 key=(null)
type=AVC msg=audit(1702877077.493:2565): avc:  denied  { execute } for  pid=74267 comm="grafana-server" name="pcp_redis_datasource_linux_amd64" dev="nvme0n1p2" ino=54562018 scontext=system_u:system_r:grafana_t:s0 tcontext=system_u:object_r:usr_t:s0 tclass=file permissive=0
[ec2-user@ip-172-26-1-226 grafana-selinux]$
```
