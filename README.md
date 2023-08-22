## Installation

# Clone the repo
git clone https://github.com/sfeifer/grafana-selinux.git

# Copy relevant .if interface file to /usr/share/selinux/devel/include to expose them when building and for future modules. May need to use full path for grafana.if if not working.
install -Dp -m 0664 -o root -g root grafana.if /usr/share/selinux/devel/include/myapplications/grafana.if

# Compile the selinux module (see below)
Ensure you have the `selinux-policy-devel` package installed.
```sh
# Ensure you have the devel packages
yum install selinux-policy-devel setools-console
# Change to the directory containing the .if, .fc & .te files
cd grafana-selinux
make -f /usr/share/selinux/devel/Makefile grafana.pp

# Install the SELinux policy module.
semodule -i grafana.pp

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