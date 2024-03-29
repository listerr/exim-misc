# Debian-exim custom build

### Why?

Debian ships two flavours of the exim4 package, `exim4-daemon-light` and `exim4-daemon-heavy`.

* exim4-daemon-light - Sufficient for most simple installations (e.g. where the host sends everything to a smarthost, or only local mail etc.)
* exim4-daemon-heavy - Is intended for mail server deployments, and is complied with more runtime features enabled, including extra database lookup capabilities, integrated spam scanning/scoring with spamd, integrated malware/content scanning ACL for use with a variety of scanners e.g. clamd. 
* ACL integration makes it easy to configure and allows Exim to perform scanning in one pass, rejecting or accepting messages at SMTP time, rather than first accepting the message, _then_ pipe it to an external program, only for it to be rejected.

 * Some features are required which are not built in `exim4-daemon-heavy` on bullseye, or are considered "experimental" and therefore not enabled by default.
("Experimental" in Exim means a feature which works, but the exact configuration may change, or it will relate to a draft, or fairly recent RFC that not everybody will want to implement yet). This is mostly to do with SPF, DKIM and ARC features and the way the Debian-exim package maintainers chose to implement these features in the different releases of Debian.

Bullseye `exim4-daemon-heavy` (and before) did not enable Exim's SPF checking, but instead relied on an external tool (spfquery) and appropriate (and a bit ugly!) configuration to make it work.

Bookworm `exim4-daemon-heavy` now has Exim's SPF checking enabled, but not ARC. "spfquery" and the ugly config is gone, but we still need ARC for mailing list support etc.

Also worth reading: 

* [Debian-Exim FAQ](https://wiki.debian.org/PkgExim4UserFAQ)
* [Exim Packages for Debian](https://wiki.debian.org/PkgExim4)
* [Debian Exim Overview](https://wiki.debian.org/Exim)

Checking what you have enabled:-

The following shows what I have enabled/disabled in the custom build:

```
root@:~# exim -bV
Exim version 4.96 #2 built 29-Sep-2023 20:38:02
Copyright (c) University of Cambridge, 1995 - 2018
(c) The Exim Maintainers and contributors in ACKNOWLEDGMENTS file, 2007 - 2022
Berkeley DB: Berkeley DB 5.3.28: (September  9, 2013)
Support for: crypteq iconv() IPv6 GnuTLS TLS_resume move_frozen_messages Content_Scanning DANE DKIM DNSSEC Event I18N OCSP PIPECONNECT PRDR Queue_Ramp SOCKS SPF SRS TCP_Fast_Open Experimental_ARC
Lookups (built-in): lsearch wildlsearch nwildlsearch iplsearch cdb dbm dbmjz dbmnz dnsdb dsearch passwd
Authenticators: plaintext
Routers: accept dnslookup ipliteral manualroute queryprogram redirect
Transports: appendfile/maildir/mailstore autoreply lmtp pipe smtp
Malware: clamd sock cmdline
Fixed never_users: 0
Configure owner: 0:0
Size of off_t: 8
Configuration file search path is /etc/exim4/exim4.conf:/var/lib/exim4/config.autogenerated
Configuration file is /etc/exim4/exim4.conf
```

Debian-exim also provides a custom source package (aka `exim4-daemon-custom`) to configure 
runtime build options as required and still benefit from the Debian-exim patches, 
apt and dependencies system without having to compile Exim from scratch from exim.org 
source (and all its dependencies.)

There is perhaps another benefit to using a custom build: Unneeded features 
can be "compiled out", allowing a "hardened" server version to be built. 

To exploit many security vulnerabilities, exploitable features usually need to be enabled in the 
configuration, and/or be configured in a certain way. If a feature is not used, 
it won't be in the config, unless an attacker has the ability to modify the 
config itself, or trick the server into loading an alternative configuration, in 
which case we'd be in even bigger trouble!

A pure "mx" or relay only server never does any local deliveries 
(i.e. there are no actual user mailboxes on the server itself.)

The following is from the [Exim documentation - Security Considerations](https://exim.org/exim-html-current/doc/html/spec_html/ch-security_considerations.html)

----
The Exim binary is normally setuid to root, which means that it 
gains root privilege (runs as root) when it starts execution.

In most installations, root privilege is required for two things:

 * To set up a socket connected to the standard SMTP port (25) when initialising the listening daemon. If Exim is run from inetd, this privileged action is not required.
 * To be able to change uid and gid in order to read users’ .forward files and perform local deliveries as the receiving user or as specified in the configuration.
----

As this is never required on an mx server, its ability to run anything as root or 
setuid can be disabled (from systemd), so it never runs as a privileged user. 

This makes any exploit relying on privileged execution considerably more difficult, or maybe impossible. 

If affected features were not compiled in exim in the first place, this may add an extra
layer of protection (e.g. if exim could be "tricked" into using an alternative 
configuration file which _did_ enable the vulnerable feature.) 

Bear in mind if systemd won't allow exim to run as root, an attacker would 
probably need to be able to change systemd's configuration too, and restart it, 
or exploit multiple vulnerabilities.

The downside being: if you actually want to use a feature that's not compiled in, 
it's obviously more work to rebuild exim rather than just adding the relevant config.

I digress...

## How to build custom Debian-exim:

### Debian Sources

If not already there, add the appropriate ``deb-src`` repositories are in ``/etc/apt/sources.list``

The important bit here is to add `deb-src` lines.

**Your sources.list will vary, So do not just copy paste these!**

- Bullseye example:
  
```
deb http://mirror.mythic-beasts.com/debian/ bullseye main
deb-src http://mirror.mythic-beasts.com/debian/ bullseye main

deb http://security.debian.org/debian-security bullseye-security main
deb-src http://security.debian.org/debian-security bullseye-security main

deb http://mirror.mythic-beasts.com/debian/ bullseye-updates main
deb-src http://mirror.mythic-beasts.com/debian/ bullseye-updates main
```

* Bookworm example:

```
deb http://deb.debian.org/debian bookworm main contrib non-free non-free-firmware
deb-src http://deb.debian.org/debian bookworm main contrib non-free non-free-firmware

deb http://deb.debian.org/debian-security/ bookworm-security main contrib non-free non-free-firmware
deb-src http://deb.debian.org/debian-security/ bookworm-security main contrib non-free non-free-firmware

deb http://deb.debian.org/debian bookworm-updates main contrib non-free non-free-firmware
deb-src http://deb.debian.org/debian bookworm-updates main contrib non-free non-free-firmware
```

Now run

```
apt-get update
```

### Install build dependencies

```
apt-get -y install pbuilder devscripts grep-dctrl debhelper
```

### Custom build:

* Bullseye ships with exim 4.94  (4.94.2)
* Bookworm ships with exim 4.96

```
mkdir exim
cd exim
apt-get source exim4

# cd to appropriate directory:
# cd exim4-4.94.2
cd exim4-4.96

fakeroot debian/rules unpack-configs

export  my_pkg_name=custom

cp -a exim_monitor/EDITME EDITME.eximon
cp EDITME.exim4-light EDITME.exim4-$my_pkg_name
```

Edit the EDITME.exim4-custom file and add/remove features as required:

```
vi EDITME.exim4-$my_pkg_name
```

By way of example, I did:

```
WITH_CONTENT_SCAN=yes
DISABLE_MAL_FFROTD=yes
DISABLE_MAL_FFROT6D=yes
DISABLE_MAL_DRWEB=yes
DISABLE_MAL_FSECURE=yes
DISABLE_MAL_SOPHIE=yes
DISABLE_MAL_AVAST=yes
EXPERIMENTAL_ARC=yes
SUPPORT_SPF=yes
LDFLAGS += -lspf2

# AUTH_CRAM_MD5=yes
# AUTH_EXTERNAL=yes
```

Note there are slight variations between different versions of Exim.

Now pack the config up again:

```
fakeroot debian/rules pack-configs
```

- In `debian/control` Uncomment the block which defines the `exim4-daemon-custom` packages:

```
vi debian/control
```

* In `debian/rules` uncomment `customdaemon = exim4-daemon-custom`:

``vi debian/rules``

* If using opendmarc:  (I'm not yet!)
  
```
apt-get source opendmarc
apt-get -y install libopendmarc-dev
```

* If building in SPF:
  
```
apt-get source libspf2-2
apt-get -y install libspf2-dev
```

### Build package deps

```
apt-get build-dep exim4

# cd to appropriate directory:
cd exim4-4.96

dpkg-source --commit . exim4-daemon-custom
debuild -us -uc
```

Note: The compiler/linter will spew a lot of warnings while it works, but you should end up with a series of .deb files.

### For a first install/testing:

```
# remove exim4 package if already installed:
apt-get -y remove exim4-daemon-heavy exim4-daemon-light

dpkg -i ../exim4-daemon-custom_4.96-15+deb12u2_amd64.deb
apt-get -y install bsd-mailx
apt -y autoremove

service exim4 restart
```

### Remove build dependencies (Optional):
```
apt-get -y remove pbuilder devscripts grep-dctrl debhelper 
apt -y autoremove
```

### Creating your own debian package (optional)

If you have many servers, you may want to create a Debian package to save 
in your own local repository, or distribute via automation:

  1. Copy all the resulting .deb files to a new directory:

  ```
  mkdir -p /tmp/debian-exim/bookworm
  cp -p /home/robl/exim/*.deb /tmp/debian-exim/bookworm
  ```

  2. Create Release:

  ```
  cd /tmp/debian-exim
  dpkg-scanpackages /tmp/debian-exim/bookworm /dev/null > /tmp/debian-exim/bookworm/Release
  ```

  3. Create Packages.gz:

  ```
  cd /tmp/debian-exim
  dpkg-scanpackages /tmp/debian-exim/bookworm | gzip > /tmp/debian-exim/bookworm/Packages.gz
  ```

  4. Upload the .deb, Release and Packages files to the relevant repository directory.


### Alternative repository creation

Alternative method (Sometimes the above didn't generate a working path, 
apt wouldn't install with newer versions of apt)

```
# vi /etc/apt/sources.list.d/exim.list
```
```
deb [trusted=yes] file:/opt/debian-exim/bookworm/debs/ /

```

Create apt package:

```
mkdir -p /opt/debian-exim/bookworm/debs
cp -pr /root/exim/*.deb /opt/debian-exim/bookworm/debs

cd /opt/debian-exim/bookworm/debs
dpkg-scanpackages -m . > Packages
dpkg-scanpackages -m . | gzip > Packages.gz

# For some reason it's picky about the owner of apt sources:
chown -R _apt /opt/debian-exim/bookworm

apt-get update && apt-get upgrade

```
