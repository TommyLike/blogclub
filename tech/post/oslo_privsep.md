# Oslo.privsep Introduce

This library helps applications perform actions which require more or less privileges than they were started with in a safe, easy to code and easy to use manner.

We used to use *rootwrap* to perform the privileged commands, but now Nova and Cinder are inclined to replace it with oslo.privsep for security and efficiency reasons. Let's compare them here and find out the pros and cons.

## Rootwrap
Rootwrap is introduced for the goal of allowing a service-specific unprivileged user to run a number of actions as the root user, it's widely used in the OpenStack services now. This process below shows
how a service can configure and use the rootwrap for their privileged
commands.

### Setup

1. Rootwrap always runs in a seperated process as a root user and only
exposes the command line for any service who wants to communicate with.
It's esay to find the rootwraps' entrance in the *setup.cfg*:
```
[entry_points]
    cinder-rootwrap = oslo_rootwrap.cmd:main
```

2. In the service itself, once we need to execute priveleged commands we have to ask rootwrap to perform and return. For instance, when Cinder
needs to setup a target in it's lvm driver, this is one of the commands
that needs privelege:
```
tgtadm --lld iscsi --op show --mode target
```
After appending the root helper's prefix the command will change into:
```
sudo cinder-rootwrap [rootwrap_config_file] tgtadm --lld iscsi --op....
```
The ``rootwrap_config_file`` is used for the rootwrap's generic config and filter invalid commands.
```
[DEFAULT]
#Comma-separated list of directories containing filter definition files
filters_path=/etc/cinder/rootwrap.d,/usr/share/cinder/rootwrap
#Comma-separated list of directories to search executables in
exec_dirs=/sbin,/usr/sbin,/bin,/usr/bin,/usr/local/bin,/usr/local/sbin,/usr/lpp/mmfs/b
#Enable logging to syslog. Default value is False
use_syslog=False
#Which syslog facility to use for syslog logging
syslog_log_facility=syslog
#Which messages to log
syslog_log_level=ERROR
```

### Filter

Rootwrap added command filter support in case of unexpected or unauthorized command are been used, it would deny the commands that don't match any filter. Some of the filters are:

1. **CommandFilter**: Basic filter that only checks the executable called
```
ploop: CommandFilter, ploop, root
[name]: CommandFilter, [command], [user]
```
2. **RegExpFilter**: Generic filter that checks the executable called, then uses a list of regular expressions to check all subsequent arguments
```
tunctl: RegExpFilter, /usr/sbin/tunctl, root, tunctl, -b, -t, .*
[name]: RegExpFilter, [command], [user], [command_regex, [option],...]
```

3. **PathFilter**: Generic filter that lets you check that paths provided as parameters fall under a given directory
```
chown: PathFilter, /bin/chown, root, nova, /var/lib/images
[name]: PathFilter, [command], [user], [path]
```

4. **EnvFilter**: Filter allowing extra environment variables to be set by the calling code
```
dnsmasq: EnvFilter, env, root, CONFIG_FILE=, NETWORK_ID=, dnsmasq
[name]: EnvFilter, env, [user], [env_config1, [env_config2]...] [command]
```

