# grafana-selinux

Grafana selinux policy module for CentOS 7 and RHEL 7.

At present, from my testing this should be working for all basic functions of Grafana. This hasn't been extensively tested(at present. I will change this when I do) by me and should be considered in an early beta state.

The policy assumes that you used the rpm from Grafana to install it. Thus all the file locations should adhere to the rpm or make sure to change them as needed.


### Untested:
* Anything outside the scope of using it at the most basic level - keep an eye on that AVC log!

### Future considerations:
* Add a grafanad_plugin_t label to contain plugins.


## Installation
```sh
# Clone the repo
git clone https://github.com/georou/grafana-selinux.git

# Copy relevant .if interface file to /usr/share/selinux/devel/include to expose them when building and for future modules
install -Dp -m 0664 -o root -g root grafanad.if /usr/share/selinux/devel/include/myapplications/grafanad.if

# Compile the selinux module (see below)

# Install the SELinux policy module. Compile it before hand to ensure proper compatibility (see below)
semodule -i grafanad.pp

# Add grafana ports
semanage port -a -t grafanad_port_t -p tcp 3000

# Restore all the correct context labels
restorecon -RvF /usr/sbin/grafana-* \
		/etc/grafana \
		/var/log/grafana \
		/var/lib/grafana

# Start grafanad
systemctl start grafana-server.service

# Ensure it's working in the proper confinement
ps -eZ | grep grafana
```

## How To Compile The Module Locally (Needed before installing)
Ensure you have the `selinux-policy-devel` package installed.
```sh
# Ensure you have the devel packages
yum install selinux-policy-devel setools-console
# Change to the directory containing the .if, .fc & .te files
cd grafana-selinux
make -f /usr/share/selinux/devel/Makefile grafanad.pp
semodule -i grafanad.pp
```

## Debugging and Troubleshooting

* If you're getting permission errors, uncomment permissive in the .te file and try again. Re-check logs for any issues. Or `semanage permissive -a grafanad_t`
* Easy way to add in allow rules is the below command, then copy or redirect into the .te module. Rebuild and re-install:
* Don't forget to actually look at what is suggested. audit2allow will most likely go for a coarse grained permission!

```sh
ausearch -m avc,user_avc,selinux_err -ts recent | audit2allow -R
```
If you get a could not open interface info [/var/lib/sepolgen/interface_info] error. 
Ensure policycoreutils-devel is installed and/or run: `sepolgen-ifgen`

## Compatibility Notes
Built on CentOS 7.4 at the time with:
```
selinux-policy-targeted-3.13.1-166.el7_4.7.noarch
selinux-policy-3.13.1-166.el7_4.7.noarch
selinux-policy-devel-3.13.1-166.el7_4.7.noarch
```
