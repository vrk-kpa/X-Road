# X-Road: External Load Balancer Installation Guide

Version: 1.0  
Doc. ID: IG-XLB


| Date       | Version     | Description                                             | Author             |
|------------|-------------|---------------------------------------------------------|--------------------|
| 3.3.2017   | 1.0         | Initial version                                         | Olli Lindgren      |


## Table of Contents

<!-- toc -->

  * [License](#license)
- [1. Introduction](#1-introduction)
  * [1.1 Target Audience](#11-target-audience)
  * [1.2 References](#12-references)
- [2. Installation](#2-installation)
  * [2.1 Prerequisites to Installation](#21-prerequisites-to-installation)
  * [2.2 Reference Data](#22-reference-data)
  * [2.3 Requirements to the Central Server](#23-requirements-to-the-central-server)
  * [2.4 Preparing OS](#24-preparing-os)
  * [2.5 Installation](#25-installation)
  * [2.6 Installing the Support for Hardware Tokens](#26-installing-the-support-for-hardware-tokens)
  * [2.7 Installing the Support for Monitoring](#27-installing-the-support-for-monitoring)
- [3 Initial Configuration](#3-initial-configuration)
  * [3.1 Reference Data](#31-reference-data)
  * [3.2 Initializing the Central Server](#32-initializing-the-central-server)
  * [3.3 Configuring the Central Server and the Management Services' Security Server](#33-configuring-the-central-server-and-the-management-services-security-server)
- [4 Additional configuration](#4-additional-configuration)
  * [4.1 Adding support for V1 global configuration](#41-adding-support-for-v1-global-configuration)
- [5 Installation Error Handling](#5-installation-error-handling)
  * [5.1 Cannot Set LC_ALL to Default Locale](#51-cannot-set-lc_all-to-default-locale)
  * [5.2 PostgreSQL Is Not UTF8 Compatible](#52-postgresql-is-not-utf8-compatible)
  * [5.3 Could Not Create Default Cluster](#53-could-not-create-default-cluster)
  * [5.4 Is Postgres Running on Port 5432?](#54-is-postgres-running-on-port-5432)

<!-- tocstop -->


## License

This document is licensed under the Creative Commons Attribution-ShareAlike 3.0 Unported License. To view a copy of this license, visit http://creativecommons.org/licenses/by-sa/3.0/.

# 1. Introduction

## 1.1 Target Audience

The intended audience of this installation guide are the X-Road security server administrators responsible for installing and configuring the X-Road security server software.
The document is intended for readers with a good knowledge of Linux server management, computer networks, database administration and the X-Road functioning principles.

## 1.2 References

1. [UG-CS] Cybernetica AS. X-Road 6. Central Server User Guide
2. [IG-SS] Cybernetica AS. X-Road 6. Security Server Installation Guide
3. [UG-SS] Cybernetica AS. X-Road 6. Security Server User Guide
4. [IG-CSHA] Cybernetica AS. X-Road 6. Central Server High Availability Installation Guide

# X. Overview

This document describes the external load balancing support features implemented by X-Road. The supported setup consists
of security servers in a cluster having an identical configuration, including their keys and certificates.
X-Road security server configuration changes are handled by a single master server and one or more slave servers (nodes).

The primary goal of the load balancing support is, as the name suggests, load balancing, not fault tolerance.
A clustered environment increases fault tolerance but some X-Road messages can still be lost if a security server node fails.

The implementation does not include a load balancer component. It should be possible to use any external load balancer
component that supports HTTP-based heath checks and load balancing at the TCP level (eg. haproxy, nginx, AWS ELB or
Classic Load Balancing, or a hardware appliance).

The load balancing support is implemented with a few assumptions about the environment that users should be aware of. Carefully consider
these assumptions before deciding if the supported features are suitable for your needs.

__Assumptions about the environment:__
* Adding or removing nodes to or from the cluster is infrequent. **FIXME:** Mihin vaikuttaa?
* Configuration changes are relatively infrequent and some downtime in ability to change configuration can be tolerated.
  Therefore, the cluster uses a master-slave model and the configuration master is not replicated.
  
__Consequences of the selected implementation model:__  
* Changes to the ServerConf database, authorization and signing keys are applied via the configuration master, which is
  a member of the cluster. The replication is one-way from master to slaves and the slaves should treat the configuration as read-only.
* The cluster nodes can continue operation if the master fails but the configuration can not be changed until:
  - the master becomes back on-line, or
  - some other node is manually promoted to the master. 
* If a node fails, the messages being processed by that node are lost.
  - It is the responsibility of the load balancer component to detect the failure and route further messages to other nodes.
    Because there potentially is some delay before the failure is noticed, some messages might be lost due to the delay.
  - Recovering any lost messages is currently out-of-scope of the support implementation.
* Configuration updates are asynchronous and the cluster state is eventually consistent.
* If the master node fails or communication is interrupted during a configuration update, each slave should have a valid configuration,
  but the cluster state can be inconsistent (some members might have the old configuration while some might have received all the changes).
  
## Communication with external servers -- The Cluster from the point of view of a SS client

![alt-text](load_balancing_traffic.png)

![alt-text](load_balancing_traffic-2.png)

![alt-text](load_balancing_state_replication.png)





# Security server cluster setup -- REFACTOR

This ansible playbook configures a master (1) - slave (n) security server cluster. In addition, setting up a load balancer (out of scope) is needed.

The playbook has been tested in AWS EC2 using stock RHEL 7 and Ubuntu 14.04 AMIs running default X-Road security server installation. Other environments might require modifications to the playbook.

## Prerequisites

* One security server that acts as master
* One or more slave security servers.
* The slave server(s) have network access to master ssh port (tcp/22)
* The slave server(s) have network access to master serverconf database (default: tcp/5433)
* X-Road security server packages have been installed on each server
    * It is not necessary to configure the servers
    * The master server configuration is preserved, so it is possible to create a cluster using an existing security server that is already attached to an X-Road instance.
* The control host executing this playbook has ssh access with sudo privileges on all the hosts.
    * Ansible version >2.1 required
    * The control host can be one of the cluster servers (e.g. the master node), but a separate control host is recommended.
* Decide names for the cluster members and configure the nodes in the ansible inventory. 
    * See hosts/cluster-example.txt for an example (nodename parameter)
    * Node names are related to certificate DN's, see "Set up SSL keys" for specifics
* Change the serverconf_password in group_vars/all and preferably encrypt the file using ansible vault. 
    * The serverconf_password is used to authenticate the local connections to the serverconf database. The default is 'serverconf'.

All the servers in a cluster should have the same operating system (Ubuntu 14.04 or RHEL 7). The setup also assumes that the servers are in the same subnet. If that is not the case, one needs to modify master's pg_hba.nconf so that it accepts replication configurations from the correct network(s).

## Set up SSL keys certificates for PostgreSQL replication connections

Create a CA certificate and store it in PEM format as ca.crt in the "ca" folder. Create TLS key and certificate (PEM) signed by the CA for each node and store those as ca/"nodename"/server.key and ca/"nodename"/server.crt. The server keys must not have a passphrase, but one can and should use ansible-vault to protect
the keys.

Note that the common name (CN) part of the certificate subject's DN must be the *nodename* defined in the host inventory file.

The ca directory contains two scripts that can be used to generate the keys and certificates.
* init.sh creates a CA key and self-signed certificate.
* add-node.sh creates a key and a certificate signed by the CA.


# 2. Installation

## 2.1 Prerequisites to setting up an external load balancer

The nodes that will be set up as slaves  


The central server software assumes an existing installation of the Ubuntu 14.04 operating system, on an x86-64bit platform.
To provide management services, a security server is installed alongside the central server.
The central server’s software can be installed both on physical and virtualized hardware (of the latter, Xen and Oracle VirtualBox have been tested).
Note: If the central server is a part of a cluster for achieving high availability, the database cluster must be installed and configured before the central server itself can be installed. Please refer to the Central Server High Availability Installation Guide [IG-CSHA] for details.

## 2.2 Reference Data


## 2.3 Requirements to the Central Server



## 2.4 Preparing OS



## 2.5 Installation


## 2.6 Installing the Support for Hardware Tokens



## 2.7 Installing the Support for Monitoring



# 3 Initial Configuration

## 3.1 Reference Data


## 3.2 Initializing the Central Server


## 3.3 Configuring the Central Server and the Management Services' Security Server


# 4 Additional configuration

## 4.1 Adding support for V1 global configuration



# 5 Installation Error Handling

## 5.1 Cannot Set LC_ALL to Default Locale

## 5.2 PostgreSQL Is Not UTF8 Compatible


## 5.3 Could Not Create Default Cluster


## 5.4 Is Postgres Running on Port 5432?
