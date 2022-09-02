OpenStack Dynamic IP Security Group Updater
===========================================

A utility for managing security group ingress rules from a dynamic IP
address.

When the dynamic IP address changes, this utility can be run to remove
the old rules and create new rules with the new IP address.

For example, it can be used to maintain security groups to allow
directly access from a home ISP connection that uses a dynamic IP
address.

## Example

The following will update the security group called "From Home" in the
OpenStack project specified by the _my-project-openrc.sh_ file.  And
the user's password is read from the _my-account.password_ file.

```sh
$ openstac-dip-sg --security-group 'From Home' \
                  --rc my-project-openrc.sh  -y my-account.password
```

It will determine the public IP address of the local computer that is
running the utility, and create ingress rules in the security group
from that IP address.

The utility is designed to be run on a semi-regular basis, when the
public IP address is suspected to have changed. To use it, there must
be a separate _security group_ in the OpenStack project, created for
ingress from the computer with the changing public IP address.

## Installation and setup

1. Install the dependencies.

2. Install the utility.

3. Obtain the OpenStack RC files.

4. Configure the security groups.

### Dependencies

- OpenStackClient
- Stuntman (optional)

#### OpenStackClient

Querying and modifying of the security groups is performed using the
unified _openstack_ command line client command, which is known as the
[OpenStackClient](https://docs.openstack.org/newton/user-guide/common/cli-overview.html).

It can be
[installed](https://docs.openstack.org/newton/user-guide/common/cli-install-openstack-command-line-clients.html)
using Python's _pip_ command. And that is recommended, since
distribution packages may contain older versions.

#### Stuntman

Optional: this is only needed for automatic public IP address
detection. If not available, the IP address must always be provided as
a command line argument.

Public Internet address detection is done using the _stunclient_
command from the [Stuntman](http://www.stunprotocol.org) project.

On macOS, _Stuntman_ can be installed using HomeBrew with _brew
install stuntman_.

### Installing the utility

Download the _openstack-dip-sg_ file and put it somewhere on your
PATH.

### OpenStack RC files

Obtain the OpenStack RC files for the projects with the security
groups the utility is going to manage.

These can be downloaded from the OpenStack Horizon Web dashboard.

1. Select the project (from the top left of the page).
2. Select _Project > API Access_ (from the left navigation column).
3. Press the "Download OpenStack RC File" dropdown menu (top right).
4. Select "OpenStack RC File".

### Configure security groups

Create or rename the security groups the utility is going to manage.
See the "Which security group in those projects" section below for
details.

## Usage

* `-s | --security-group NAME` --- name of the security groups to
  update.

* `-s | --rc PROJECT_SRC` --- OpenStack RC file(s) for the projects
   containing the named security group. Either a single filename or a
   directory containing one or more files with names that end in
   "-openrc.sh".

* `-y | --password FILE` --- read OpenStack password from this file,
   instead of prompting for it.

* `-p | --port PORT_NUM` --- add ingress rules for these TCP/IP ports.
   This option can be repeated, or multiple comma separated port
   numbers can be provided in the value.

* `--no-icmp` --- do not add an ingress rule for all ICMP packets.

* `-o | --output FILE` --- output file from OpenStack commands.

* `-f | --force` --- update the security groups, even if the IP
  address has not changed from the previous update.

* `-S | --stun SERVER` --- STUN server for detecting the public
  Internet IP address.

* `-l | --log FILE` --- log file for saving the most recent IP address
  that was used. Set to an empty string to disable this feature.

* `--version` --- display version information and exit.

* `-h | --help` display the help and exit.

## Details

### Security groups to update

#### Which OpenStack projects

The utility will operate on the security group from one or more
OpenStack projects.

The project(s) are those:

- identified by environment variables passed to the utility
  (i.e. source an OpenStack RC file before running the utility);

- a single OpenStack RC file by providing a file to the `--rc` option; or

- multiple OpenStack RC files by providing a directory to the `--rc`
  option. All the files in that directory that ends with "-openrc.sh"
  will be used.

The password, for the OS_USERNAME accounts in the OpenStack RC files,
is obtained in this order:

1. The `OS_PASSWORD` environment variable, if it has been set.
2. from the first line in the file from the `--password` option.
3. Otherwise, it prompts for the password.

The same password is used with all the projects. So if there are
multiple OpenStack RC files, they should all be using the same
username.

#### Which security group in those projects

The utility will only operate on OpenStack _security groups_
that satisfy **all** of these conditions:

- The name is the same as the value in the `--security-group` option.

- All rules in it are for ingress.

- All rules in it refer to the same source address.

- The source address is a single IPv4 address (i.e. it cannot be
  an IPv6 address, and must be a "/32" CIDR).

This is to avoid it accidentily modifying the wrong security group.  If
the intended security group does not satisfy these conditions, modify
it so it does satisfy them.

Note: a security group with no rules in it satisfies all these
conditions, as long as its name matches.

### Updates that are made to the security groups

The utility will modify the rules in the selected security groups by:

1. Deleting all the existing rules.

2. Creating a new rule for all ICMP ingress from the source IP address
   (unless the `--no-icmp` option is used).

3. Creating new TCP/IP ingress rules to the specified ports from the
   source IP address.

The ports can be specified using the `--port` option. It defaults to
allow ingress to SSH (port 22), HTTP (port 80) and HTTPS (port 443).

By default, the output from the _openstack security group rule create_
command are discarded. To view them, use the `--output` option to
write them into a file.

### Source IP address for the ingress rules

#### Obtaining the source IP address

The source address for the new rules must be an IPv4 address.

If it is not explicitly provided as a command line argument, the
utility tries to use the public Internet address for the machine
running the utility.  This detection is done using the _Session
Traversal Utilities for NAT_ (STUN) protocol.  The STUN server can be
set using the `--stun` option.

If the value of "none" is used for the IP address, all the rules are
deleted but no rules are created.

#### Changes are only made if the source IP address changes

The security groups are only updated if the IP address has changed,
since the last time the utility had updated them.  This allows the
utility to be run without causing any changes, if the public IP
address has not changed.

The change in IP address is detected locally. It does not query
OpenStack to detect if it has changed.It compares the IP address with
the previous IP address stored in the log file.  The log file can be
changed with the `--log` option.

If the utility is used to manage security groups in different projects
with different ports, use a different log file for each. Otherwise,
the first project will update the log file, and subsequent projects
will not get updated (since the IP address would be the same).

### Forcing updates

An update can be forced by using the `--force` option.

The force option should be used if the security group's name or the
set of TCP/IP ports has changed since the last run. Otherwise, the
results may be unexpected. For example, if the IP address has not
changed and the utility is run with a new port number, a new rule will
not be created unless the force option is used.

## Licence

This utility is distributed in the hope that they will be useful,
but without any warranty; without even the implied warranty of
merchantability or fitness for a particular purpose. See the GNU
General Public License for more details.

## Reporting issues

Please submit issues using the repository's [GitHub
issues](https://github.com/qcif/openstack-dip-sg/issues)

