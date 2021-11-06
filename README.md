OpenStack Dynamic IP Security Group Updater
===========================================

A utility for managing security group ingress rules from a dynamic IP address.

When the dynamic IP address changes, this utility can be run to remove
the old rules and create new rules with the new IP address.

## Example

The following will update the security group called "From Home" in the
OpenStack project specified by the _my-project-openrc.sh_ file.  And
the user's password is read from the _my-account.password_ file.

```sh
openstac-dip-sg --security-group 'From Home' --rc my-project-openrc.sh  -y my-account.password
```

It will determine the public IP address of the local computer that is
running the utility, and create ingress rules in the security group
from that IP address.

It can be run everytime the public IP address of the local computer
changes.

## Requirements

- OpenStack RC files for the projects with the security groups to update.

- [OpenStack command line
  client](https://docs.openstack.org/newton/user-guide/common/cli-install-openstack-command-line-clients.html)
  for the unified _openstack_ command.

- [Stuntman](http://www.stunprotocol.org) for the _stunclient_
  command.  (Optional: only needed for automatic public IP address
  detection.)

## Usage

* `-s | --security-group` NAME --- name of the security groups to update

* `-s | --rc PROJECT_SRC` --- OpenStack RC file(s) for the projects
   containing the named security group. Either a single filename or a
   directory containing one or more "*-openrc.sh" files.

* `-y | --password FILE` --- read OpenStack password from this file,
   instead of prompting for it.

* `-p | --port PORT_NUM` --- add ingress rules for these TCP/IP ports.

* `--no-icmp` --- do not add an ingress rule for all ICMP packets.

* `-o | --output FILE` --- output file from OpenStack commands.

* `-f | --force` --- update security groups, even if the IP address has not
  changed from the previous update.

* `-S | --stun SERVER` --- STUN server for detecting the public Internet IP address.

* `-l | --log FILE` --- log file.

* `-h | --help` display the help and exit.



## Details

### Security groups to update

#### Which OpenStack projects

The script will operate on the security group from one or more
OpenStack projects.

The project(s) are those:

- identified by the environment variables passed to the script
  (i.e. source an OpenStack RC file before running the script);

- a single OpenStack RC file by providing a file to the `--rc` option;

- multiple OpenStack RC files by providiing a directory to the `--rc`
  option. All the files in that directory that ends with "-openrc.sh"
  will be used.

The password, for the OS_USERNAME accounts in the OpenStack RC files,
is obtained in this order:

1. The `OS_PASSWORD` environment variable, if it has been set.
2. Read from the file indicated with the `--password` option.
3. Otherwise, it prompts for the password.

The same password is used with all the projects. So if there are
multiple OpenStack RC files, they should all be using the same
username.

#### Which security group in those projects

The script will only operate on OpenStack _security groups_
that satisfy **all** of these conditions:

- The name is the same as the value in the `--security-group` option.

- All rules in it are for ingress.

- All rules in it refer to the same source address.

- The source address is a single IPv4 address (i.e. it cannot be
  an IPv6 address, and must be a "/32" CIDR).

This is to avoid it accidently modifying the wrong security group.  If
the intended security group does not satisfy these conditions, modify
it so it does satisfy them.

Note: a security group with no rules in it, but matching the name,
satisfies all these conditions.

### Updates that are made to the security groups

The script will modify the rules in the selected security groups by:

1. Deleting all the existing rules.

2. Creating a new rule for all ICMP ingress from the source address
   (unless the `--no-icmp` option is used).

3. Creating new TCP/IP ingress rules to the specified ports from the
   source address.

The ports can be specified using the `--port` option. It defaults to
allow ingress to SSH (port 22), HTTP (port 80) and HTTPS (port 443).

By default, the output from the _openstack security group rule create_
command are discarded. To view them, use the `--output` option to
write them into a file.

### Source IP address for the ingress rules

#### Obtaining the source IP address

The source address for the new rules must be an IPv4 address.

If it is not explicitly provided as a command line argument, the
program tries to use the public Internet address for the machine
running the program.

The public Internet address is detected using the _stunclient_
program, from the _Stuntman_ project is.  On macOS, _Stuntman_ can
be installed using HomeBrew with _brew install stuntman_.

The detection is done using the _Session Traversal Utilities for NAT_
(STUN) protocol.  The STUN server can be set using the `--stun`
option.

#### Changes are only made if the source IP address changes

The security groups are only updated if the IP address has changed,
since the last time the utility was run.  The command can be re-run
without causing any changes, if the public IP address has not changed.

An update can be forced by using the `--force` option.

The force option should be used if the security group's name or the
set of TCP/IP ports has changed since the last run. The utility is
unable to detect those changes.

The change in IP address is detected by comparing the IP address with
the previous IP address, which is stored in the log file.  The log
file can be changed with the `--log` option. To not use a log file,
set its filename to an empty string.


