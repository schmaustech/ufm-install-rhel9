# NVIDIA UFM on Red Hat Enterprise Linux 9

**Goal**: The goal of this document is to provide an easy to consume procedure for install UFM on Red Hat Enterprise Linux 9.

## Workflow Sections

- [Environment](#environment)
- [Configure Repos](#configure-repos)
- [Set Firewall Rules](#set-firewall-rules)
- [Disable SeLinux](#disable-selinux)
- [Install Software Dependencies](#install-software-dependencies)
- [Install UFM Software](#install-ufm-software)
- [Configure UFM](#configure-ufm)
- [Start UFM Services](#start-ufm-services)

## Environment

## Configure Repos

~~~bash
# subscription-manager repos --enable codeready-builder-for-rhel-9-$(arch)-rpms
Repository 'codeready-builder-for-rhel-9-x86_64-rpms' is enabled for this system.

[root@nvd-srv-26 ~]# dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm -y
Updating Subscription Management repositories.
Red Hat CodeReady Linux Builder for RHEL 9 x86_64 (RPMs)                                                                                                                            47 MB/s |  15 MB     00:00    
epel-release-latest-9.noarch.rpm                                                                                                                                                   1.1 MB/s |  19 kB     00:00    
Dependencies resolved.
===================================================================================================================================================================================================================
 Package                                              Architecture                                   Version                                            Repository                                            Size
===================================================================================================================================================================================================================
Installing:
 epel-release                                         noarch                                         9-10.el9                                           @commandline                                          19 k

Transaction Summary
===================================================================================================================================================================================================================
Install  1 Package

Total size: 19 k
Installed size: 26 k
Downloading Packages:
Running transaction check
Transaction check succeeded.
Running transaction test
Transaction test succeeded.
Running transaction
  Preparing        :                                                                                                                                                                                           1/1 
  Installing       : epel-release-9-10.el9.noarch                                                                                                                                                              1/1 
  Running scriptlet: epel-release-9-10.el9.noarch                                                                                                                                                              1/1 
Many EPEL packages require the CodeReady Builder (CRB) repository.
It is recommended that you run /usr/bin/crb enable to enable the CRB repository.

  Verifying        : epel-release-9-10.el9.noarch                                                                                                                                                              1/1 
Installed products updated.

Installed:
  epel-release-9-10.el9.noarch                                                                                                                                                                                     

Complete!

#

# yum repolist
Updating Subscription Management repositories.
repo id                                                                                     repo name
codeready-builder-for-rhel-9-x86_64-rpms                                                    Red Hat CodeReady Linux Builder for RHEL 9 x86_64 (RPMs)
epel                                                                                        Extra Packages for Enterprise Linux 9 - x86_64
epel-cisco-openh264                                                                         Extra Packages for Enterprise Linux 9 openh264 (From Cisco) - x86_64
rhel-9-for-x86_64-appstream-rpms                                                            Red Hat Enterprise Linux 9 for x86_64 - AppStream (RPMs)
rhel-9-for-x86_64-baseos-rpms                                                               Red Hat Enterprise Linux 9 for x86_64 - BaseOS (RPMs)
~~~

## Set Firewall Rules

~~~bash
# firewall-cmd --get-active-zones
public
  interfaces: eno12399 enp55s0np0

# firewall-cmd --zone=public --add-service=http
success

# firewall-cmd --zone=public --add-service=https
success

#  firewall-cmd --permanent --zone=public --add-service=http
success

# firewall-cmd --permanent --zone=public --add-service=https
success

# firewall-cmd --reload
success

# firewall-cmd --zone=public --list-services
cockpit dhcpv6-client http https ssh
~~~

## Disable SeLinux

hi
