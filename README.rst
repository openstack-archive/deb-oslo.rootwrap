-------------
Oslo Rootwrap
-------------

The Oslo Rootwrap allows fine filtering of shell commands to run as `root`
from OpenStack services.

Rootwrap should be used as a separate Python process calling the
oslo.rootwrap.cmd:main function. You can set up a specific console_script
calling into oslo.rootwrap.cmd:main, called for example `nova-rootwrap`.
To keep things simple, this document will consider that your console_script
is called `/usr/bin/nova-rootwrap`.

The rootwrap command line should be called under `sudo`. It's first parameter
is the configuration file to use, and the remainder of the parameters are the
command line to execute:

`sudo nova-rootwrap ROOTWRAP_CONFIG COMMAND_LINE`


How rootwrap works
==================

OpenStack services generally run under a specific, unprivileged user. However,
sometimes they need to run a command as `root`. Instead of just calling
`sudo make me a sandwich` and have a blanket `sudoers` permission to always
escalate rights from their unprivileged users to `root`, those services can
call `sudo nova-rootwrap /etc/nova/rootwrap.conf make me a sandwich`.

A sudoers entry lets the unprivileged user run `nova-rootwrap` as `root`.
`nova-rootwrap` looks for filter definition directories in its configuration
file, and loads command filters from them. Then it checks if the command
requested by the OpenStack service matches one of those filters, in which
case it executes the command (as `root`). If no filter matches, it denies
the request. This allows for complex filtering of allowed commands, as well
as shipping filter definitions together with the OpenStack code that needs
them.

Security model
==============

The escalation path is fully controlled by the `root` user. A `sudoers` entry
(owned by `root`) allows the unprivileged user to run (as `root`) a specific
rootwrap executable, and only with a specific configuration file (which should
be owned by `root`) as its first parameter.

`nova-rootwrap` imports the Python modules it needs from a cleaned (and
system-default) `PYTHONPATH`. The configuration file points to root-owned
filter definition directories, which contain root-owned filters definition
files. This chain ensures that the unprivileged user itself is never in
control of the configuration or modules used by the `nova-rootwrap` executable.

Installation
============

All nodes wishing to run `nova-rootwrap` should contain a `sudoers` entry that
lets the unprivileged user run `nova-rootwrap` as `root`, pointing to the
root-owned `rootwrap.conf` configuration file and allowing any parameter
after that. For example, Nova nodes should have this line in their `sudoers`
file, to allow the `nova` user to call `sudo nova-rootwrap`:

``nova ALL = (root) NOPASSWD: /usr/bin/nova-rootwrap /etc/nova/rootwrap.conf *``

Then the node also should ship the filter definitions corresponding to its
usage of `nova-rootwrap`. You should not install any other filters file on
that node, otherwise you would allow extra unneeded commands to be run as
`root`.

The filter file(s) corresponding to the node must be installed in one of the
filters_path directories. For example, on Nova compute nodes, you should only
have `compute.filters` installed. The file should be owned and writeable only
by the `root` user.

Rootwrap configuration
======================

The `rootwrap.conf` file is used to influence how `nova-rootwrap` works. Since
it's in the trusted security path, it needs to be owned and writeable only by
the `root` user. Its location is specified in the `sudoers` entry, and must be
provided on `nova-rootwrap` command line as its first argument.

`rootwrap.conf` uses an *INI* file format with the following sections and
parameters:

[DEFAULT] section
-----------------

filters_path
    Comma-separated list of directories containing filter definition files.
    All directories listed must be owned and only writeable by `root`.
    This is the only mandatory parameter.
    Example:
    ``filters_path=/etc/nova/rootwrap.d,/usr/share/nova/rootwrap``

exec_dirs
    Comma-separated list of directories to search executables in, in case
    filters do not explicitely specify a full path. If not specified, defaults
    to the system `PATH` environment variable. All directories listed must be
    owned and only writeable by `root`. Example:
    ``exec_dirs=/sbin,/usr/sbin,/bin,/usr/bin``

use_syslog
    Enable logging to syslog. Default value is False. Example:
    ``use_syslog=True``

use_syslog_rfc_format
    Enable RFC5424 compliant format for syslog (add APP-NAME before MSG part).
    Default value is False. Example:
    ``use_syslog_rfc_format=True``

syslog_log_facility
    Which syslog facility to use for syslog logging. Valid values include
    `auth`, `authpriv`, `syslog`, `user0`, `user1`...
    Default value is `syslog`. Example:
    ``syslog_log_facility=syslog``

syslog_log_level
    Which messages to log. `INFO` means log all usage, `ERROR` means only log
    unsuccessful attempts. Example:
    ``syslog_log_level=ERROR``

.filters files
==============

Filters definition files contain lists of filters that `nova-rootwrap` will
use to allow or deny a specific command. They are generally suffixed by
`.filters`. Since they are in the trusted security path, they need to be
owned and writeable only by the `root` user. Their location is specified
in the `rootwrap.conf` file.

It uses an *INI* file format with a `[Filters]` section and several lines,
each with a unique parameter name (different for each filter you define):

[Filters] section
-----------------

filter_name (different for each filter)
    Comma-separated list containing first the Filter class to use, followed
    by that Filter arguments (which vary depending on the Filter class
    selected). Example:
    ``kpartx: CommandFilter, /sbin/kpartx, root``


Available filter classes
========================

CommandFilter
-------------

Basic filter that only checks the executable called. Parameters are:

1. Executable allowed
2. User to run the command under

Example: allow to run kpartx as the root user, with any parameters:

``kpartx: CommandFilter, kpartx, root``

RegExpFilter
------------

Generic filter that checks the executable called, then uses a list of regular
expressions to check all subsequent arguments. Parameters are:

1. Executable allowed
2. User to run the command under
3. (and following) Regular expressions to use to match first (and subsequent)
   command arguments

Example: allow to run `/usr/sbin/tunctl`, but only with three parameters with
the first two being -b and -t:

``tunctl: /usr/sbin/tunctl, root, tunctl, -b, -t, .*``

PathFilter
----------

Generic filter that lets you check that paths provided as parameters fall
under a given directory. Parameters are:

1. Executable allowed
2. User to run the command under
3. (and following) Command arguments.

There are three types of command arguments: `pass` will accept any parameter
value, a string will only accept the corresponding string as a parameter,
except if the string starts with '/' in which case it will accept any path
that resolves under the corresponding directory.

Example: allow to chown to the 'nova' user any file under /var/lib/images:

``chown: PathFilter, /bin/chown, root, nova, /var/lib/images``

EnvFilter
---------

Filter allowing extra environment variables to be set by the calling code.
Parameters are:

1. `env`
2. User to run the command under
3. (and following) name of the environment variables that can be set,
   suffixed by `=`
4. Executable allowed

Example: allow to run `CONFIG_FILE=foo NETWORK_ID=bar dnsmasq ...` as root:

``dnsmasq: EnvFilter, env, root, CONFIG_FILE=, NETWORK_ID=, dnsmasq``

ReadFileFilter
--------------

Specific filter that lets you read files as `root` using `cat`.
Parameters are:

1. Path to the file that you want to read as the `root` user.

Example: allow to run `cat /etc/iscsi/initiatorname.iscsi` as `root`:

``read_initiator: ReadFileFilter, /etc/iscsi/initiatorname.iscsi``

KillFilter
----------

Kill-specific filter that checks the affected process and the signal sent
before allowing the command. Parameters are:

1. User to run `kill` under
2. Only affect processes running that executable
3. (and following) Signals you're allowed to send

Example: allow to send `-9` or `-HUP` signals to `/usr/sbin/dnsmasq` processes:

``kill_dnsmasq: KillFilter, root, /usr/sbin/dnsmasq, -9, -HUP``

IpFilter
--------

ip-specific filter that allows to run any `ip` command, except for `ip netns`
(in which case it only allows the list, add and delete subcommands).
Parameters are:

1. `ip`
2. User to run `ip` under

Example: allow to run any `ip` command except `ip netns exec` and
`ip netns monitor`:

``ip: IpFilter, ip, root``

IpNetnsExecFilter
-----------------

ip-specific filter that allows to run any otherwise-allowed command under
`ip netns exec`. The command specified to `ip netns exec` must match another
filter for this filter to accept it. Parameters are:

1. `ip`
2. User to run `ip` under

Example: allow to run `ip netns exec <namespace> <command>` as long as
`<command>` matches another filter:

``ip: IpNetnsExecFilter, ip, root``


Calling rootwrap from OpenStack services
=============================================

The `oslo.processutils` library ships with a convenience `execute()` function
that can be used to call shell commands as `root`, if you call it with the
following parameters:

``run_as_root=True``

``root_helper='sudo nova-rootwrap /etc/nova/rootwrap.conf``

NB: Some services ship with a `utils.execute()` convenience function that
automatically sets `root_helper` based on the value of a `rootwrap_config`
parameter, so only `run_as_root=True` needs to be set.

If you want to call as `root` a previously-unauthorized command, you will also
need to modify the filters (generally shipped in the source tree under
`etc/rootwrap.d` so that the command you want to run as `root` will actually
be allowed by `nova-rootwrap`.
