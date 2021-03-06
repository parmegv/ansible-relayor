ansible-relayor
----------------
This is an ansible role for tor relay operators.

The main focus of this role is to automate as many steps as possible for a tor relay
operator including key management (OfflineMasterKey).
Deploying a new tor server is as easy as adding a new host to the inventory,
no further manual configuration is required.

This ansible role does not aim to support tor bridges.

Main benefits for a tor relay operator
--------------------------------------
- security: **offline Ed25519 master keys** are generated on the ansible host and are never exposed to the relay ([OfflineMasterKey](https://trac.torproject.org/projects/tor/wiki/doc/TorRelaySecurity/OfflineKeys))
- **easy Ed25519 signing key renewal** (valid for 30 days by default - configurable)
- security: compartmentalization: every tor instance is run with a distinct user
- automatically makes use of IPv6 IPs (if available)
- **automatic MyFamily management**
- automatic tor instance generation (two by default - configurable)
- easily choose between exit relay/non-exit relay mode using a single boolean
- easily restore a relay setup (the ansible host becomes a backup location for all keys out of the box)

Installation
------------

This ansible role is available on galaxy https://galaxy.ansible.com/nusenu/relayor/

```ansible-galaxy install nusenu.relayor```

Requirements
------------
Control Machine Requirements

- do **not** run this role with `become: yes`
- tor >= 0.2.7
- python-netaddr package must be installed
- required commands: openssl, sort, uniq, wc, cut, sed, xargs
- ansible >= 2.1.4 or >= 2.2.1

Managed Node Requirements

- a non-root user with sudo permissions
- python 2 under /usr/bin/python
- static IPv4 address(es)
    - we can use multiple public IPs
    - if you have no public IP we will use a single private IP (and assume NAT)

Supported Operating Systems
---------------------------

- Debian 8 / Debian Testing
- CentOS 7 (incl. SELinux support)
- OpenBSD 6.0
- FreeBSD 10.3, 11.0
- Ubuntu 16.04
- Fedora 25

Supported Tor Releases
-----------------------
- tor >= 0.2.8.x
- on OpenBSD: tor >= 0.2.7.x
- on FreeBSD: >= 0.2.8.9**_2**


Role Variables
--------------
All variables mentioned here are optional.

* `tor_ContactInfo` string
    - Sets the relay's ContactInfo field.

* `tor_signingkeylifetime_days` integer
    - all tor instances created by relayor run in [OfflineMasterKey](https://www.torproject.org/docs/tor-manual.html.en#OfflineMasterKey) mode
    - this setting defines the lifetime of Ed25519 signing keys in days
    - indirectly defines **how often you have to run your ansible playbook to ensure your relay keys do not expire**
    - **a tor instance in OfflineMasterKey mode automatically stops when his key/cert expires, so this is a crucial setting!**
    - lower values (eg. 7) are better from a security point of view but require more frequent playbook runs
    - default: 30

* `tor_ports`
    - This var allows you to
        - select tor's ORPort and DirPort
        - reduce the number of Tor instances created per IP address
    - disable DirPorts by setting them to 0
    - HINT: choose them wisely and *never* change them again ;)
    - NOTE: on SELinux-enabled systems you must choose from the following ports:
    - 80, 81, 443, 488, 6969, 8008, 8009, 8443, 9000, 9001, 9030, 9050, 9051, 9150
    - default:
        - instance 1: ORPort 9000, DirPort 9001
        - instance 2: ORPort 9100, DirPort 9101

* `tor_offline_masterkey_dir`
    - default: ~/.tor/offlinemasterkeys
    - Defines the location where on the ansible control machine we store relay keys (Ed25519 and RSA)
    - Within that folder ansible will create a subfolder for every tor instance.
    - see the [documentation](https://github.com/nusenu/ansible-relayor/wiki/How-to-migrate-all-tor-instances-of-one-server-to-another) if you want to migrate instances to a new server
    - **note**: do not manually mangle file and/or foldernames/content in these tor DataDirs

* `tor_nickname` string
    - defines the nickname tor instances will use
    - up to 19 chars long, must contain only the characters [a-zA-Z0-9]
    - all tor instances on a host will get the same nickname
    - to use the server's hostname as the nickname set it to {{ ansible_hostname }}
    - tor_nicknamefile overrules this setting
    - default: none

* `tor_nicknamefile` /path/to/file.csv
    - this is a simple comma separated csv file stored on the ansible control machine specifying nicknames
    - first column: instance identifier (inventory_hostname-ip_orport)
    - second column: nickname
    - one instance per line
    - all instances MUST be present in the csv file
    - default: not set

* `tor_alpha` boolean
    - Set to True if you want to enable the Tor alpha version repository.
    - Note: This setting does not ensure an installed tor is upgraded to the alpha release.
    - This setting is supported on Debian/Ubuntu only (ignored on other platforms).
    - default: False

* `tor_ExitRelay` boolean
    - You will want to set this to True if you want to run exit relays.
    - default: False

* `tor_ExitNoticePage` boolean
    - specifies whether we display the default tor exit notice [html page](https://gitweb.torproject.org/tor.git/plain/contrib/operator-tools/tor-exit-notice.html) on the DirPort
    - only relevant if we are an exit
    - default: True

* `tor_AbuseEmailAddress` email-address
    - if set this email address is used on the tor exit notice [html page](https://gitweb.torproject.org/tor.git/plain/contrib/operator-tools/tor-exit-notice.html) published on the DirPort
    - you are encouraged to set it if you run an exit
    - only relevant if `tor_ExitRelay` is True
    - default: not set

* `tor_ExitPolicy`
    - specify your custom exit policy
    - only relevant if `tor_ExitRelay` is True
    - default: reduced exit policy (https://trac.torproject.org/projects/tor/wiki/doc/ReducedExitPolicy)

* `tor_maxPublicIPs`
    - Limits the amount of public IPs we will use to generate instances on a single host.
    - Indirectly limits the amount of instances we generate per host.
    - default: 1

* `tor_IPv6` boolean
    - autodetects if you have IPv6 IPs and enables an IPv6 ORPort accordingly
    - you can opt-out by setting it to False
    - default: True

* `tor_IPv6Exit` boolean
    - enables IPv6 exit traffic
    - only relevant if `tor_ExitRelay` and `tor_IPv6` are True and we have an IPv6 address
    - default: True (unlike tor's default)

* `tor_enableControlSocket`
    - will create a ControlSocket file named 'controlsocket' in every instance's datadir
    - authentication relies on filesystem permissions
    - default: False

* `freebsd_somaxconn`
    - configure kern.ipc.somaxconn on FreeBSD
    - by default we increase this value to at least 1024
    - if the value is higher than that we do not touch it

* `freebsd_nmbclusters`
    - configure kern.ipc.nmbclusters on FreeBSD
    - by default we increase this value to at least 30000
    - if the value is higher than that we do not touch it

This role supports most torrc options documented in the 'SERVER OPTIONS'
section of tor's manual. Set them via 'tor_OptionName'.
Have a look at templates/torrc if you want to have list of supported
options.

Available Role Tags
--------------------

Using ansible tags is optional but allows you to speed up playbook runs if
you are managing many servers.

There are OS specific tags:

* debian (includes ubuntu)
* centos
* fedora
* freebsd
* openbsd

Task oriented tags:

* **renewkey** - takes care of renewing online Ed25519 keys only (assumes that tor instances are fully configured and running already)
* install - installs tor but does not start or enable it
* createdir - creates (empty) directories on the ansible host only, useful for migration
* reconfigure - regenerates torrc files and reloads tor (requires previously configured tor instances)

So if you have a big family and you are about to add an OpenBSD host you typically
make two steps

1. install the new server by running only against the new server (-l) and only the os specific tag (openbsd)

    `ansible-playbook yourplaybook.yml -l newserver --tags openbsd`

2. then reconfigure all servers (MyFamily) by running the 'reconfigure' tag against all servers.

    `ansible-playbook yourplaybook.yml --tags reconfigure`

Security Considerations
------------------------
This ansible role makes use of tor's OfflineMasterKey feature without requiring any manual configuration.

The offline master key feature exposes only a temporary signing key to the relay (valid for 30 days by default).
This allows to recover from a complete server compromise without losing a relay's reputation (no need to bootstrap a new permanent master key from scratch).

Every tor instance is run with a distinct system user. A per-instance user has only access to his own (temporary) keys, but not to those of other instances.
We do not ultimately trust every tor relay we operate (we try to perform input validation when we use relay provided data on the ansible host or another relay).

**Be aware that the ansible control machine stores ALL your relay keys (RSA and Ed25519) - apply security measures accordingly.**


Reporting Security Bugs
-----------------------

Feel free to submit them in the public issue tracker,
or if you like via GPG encrypted email.

Origins
-------
https://github.com/david415/ansible-tor (changed significantly since then)
