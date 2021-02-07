# Cuckoo/CAPE data collection
## Documentation
The goal of this documentation is to introduce the project and its infrastructure to the potential user, not all the details are included.

## Summary for users
If you want access collected data, send your public key to Dominik Kouba (!! to access the datasets you have to be on University network!!). If you want to change something regarding infrastructure (windows software, sandbox configuration, network configuration, additional network capture...) drop a message to Domnik Kouba. Any other issues --> Dominik Kouba.

## General
Goal of our experiment is to perform dynamic analysis of portable executable files using sandbox CAPEv2. Those data are going to be collected and further analysed. Data are collected under three different setups of the sandbox host machine.
1. No internet connection for the machine (_none_)
2. Internet connection (_internet_)
3. KCL University modified version of sandbox monitor software (dll file used during analyses on windows machines) (_modified_)

Due to the fact that dynamic analysis is much more time-consuming than the statical one, the task distribution is involved. The architecture used for this purpose is Master-Slave.

Samples are fetched from [abuse](abuse.ch). To compare analysis results the public instance of [cape sandbox](https://capesandbox.com/) or for instance [any.run](https://app.any.run/submissions/) could be used.

## Machines
Each machine is running ssh client on port _Will be shared after providing the public key_.

**Network:** _Will be shared after providing the public key_

**Master:** .191 (Public key auth ssh)

**Slaves:**
- .230
- .199
- .170
- .177
- .204
- .237
- .207

**VPN Router:** .179 (Vbox Virtual machine on master machine)

Each slave is running 4 KVM windows virtual machines for sandbox use. As noticed we have three modes in witch slave machine could be configured - internet, none and modified (all described below).
  
## Sandbox
There is running the Capev2 sandbox on each slave machine  - [repo](https://github.com/kevoreilly/CAPEv2). This sandbox is running (almost) out of box on our slave machines. Everything is set to run after the system boot, so in case of some changes it is better to reboot host machine. Sandbox home directory is `/opt/CAPEv2`, all its scripts are there.

Important (not all) sandbox running processes are:
- `/usr/bin/python3 cuckoo.py` - sandbox itself (has to run as _cape_ user!)
- `/usr/bin/python3 manage.py runserver 0.0.0.0:8000` - sandbox web interface
- `/usr/bin/python3 process.py -p7 auto -pt 900` - processing script

Useful:
- `sudo su cape` - sandbox should be managed only as this user (especially not as root)
- `ps afx | grep cuckoo`, `kill -9 <pid>`
- `python3 cuckoo.py -d` - run in debug mode (logging to console, tmux is handy in this case)
- `/opt/CAPEv2/config` - configuration files of sandbox
- `/opt/CAPEv2/log` - logs of sandbox


## Slave modes
### None
This mode implies that machine has no internet access. There is only isolated network between the host where the cape result server (port 2042) is running and guest where cape agent is running (port 8000). Nothing else.
````
+------------------------------------------------------------------------+
| SLAVE MACHINE                                                          |
|                                                                        |
|                                                                        |
|                                                                        |
|                                                                        |
|         +-------------+                   +--------------+             |
|         |             |    Isolated net   |              |             |
|         | KVM GUEST   +-------------------+    HOST OS   |             |
|         +-------------+ 192.168.100.0/24  +--------------+             |
|                                                                        |
|                                                                        |
|                                                                        |
+------------------------------------------------------------------------+
````
### Internet
In this case architecture is a little more complex. All windows's traffic is redirected through dirty lab (safe place to mess). Machines see each other on the same network 172.20.1.0/24 and even host machines (because of the result server) has to be assigned with ip address. This network is created for the purpose of L2 VPN between the router and each machine. All the traffic is centralized through the router machine and public ipv4 address of all KVM guests is the outgoing interface of dirty lab.


```
+----------------------------------------------------------------------------+
|SLAVE MACHINE                                                               |
|                                                                            |
|                                         +------------------------------+   |
|  +-----------------+                    |HOST OS                       |   |
|  |KVM GUEST        |             +----------------------------------+  |   |
|  |                 |             |      |        BR0                |  |   |
|  |            +----+------+      | +----+-------+    +-----------+  |  |   |
|  |            |INTERFACE  |      | | VNET       |    | TAP       |  |  |   |
|  |            |           +--------+            +----+           |  |  |   |
|  |            +----+------+      | +----+-------+    +-----+-----+  |  |   |
|  |                 |             |      |                  |        |  |   |
|  |                 |             +----------------------------------+  |   |
|  +-----------------+                    +------------------------------+   |
|                                                            |               |
|                                                            |               |
+----------------------------------------------------------------------------+
                                                             |
                                                             |L2 VPN
                               +----------------------------------------------+
                               | MASTER MACHINE              |                |
                               |                             |                |
                               |                       +-----+------+         |
                               |      +----------------+   TAP      +------+  |
                               |      |ROUTER VM       |            |      |  |
                               |      |                +------+-----+      |  |
                               |      |                       |            |  |
                               |      |                       | NAT        |  |
                               |      |                       |            |  |
                               |      |                +------+------+     |  |
                               |      |                |             |     |  |
                               |      +----------------+    TUN       -----+  |
                               |                       +------+------+        |
                               |                              |               |
                               +----------------------------------------------+
                                                              | L3 VPN
                                                              |
                                                       +------+-------+
                                                       |              |
                                                       |  Dirty LAB   |
                                                       |              |
                                                       +------+-------+
                                                              | OUTGOING
                                                              |
                                                       +------+-------+
                                                       |              |
                                                       |   INTERNET   |
                                                       |              |
                                                       +--------------+

```

_Used http://asciiflow.com/_


### Modified capemon
This change is in ``/opt/CAPEv2/analyzer/windows/dll/``, where is just different version of `capemon_x64.dll` and `capemon.dll` which is modified by KCL guys.


## Data collection
Main steps:
1. Download samples from [abuse](abuse.ch)
2. Filter samples by file type (PE) - filtered directly pe files and then extracted zip files and filtered again
3. Add filtered samples to database
4. Get additional metadata for files ([VT](https://www.virustotal.com/gui/))  
5. Distribute to slaves (each file should be analysed in each mode, for distribution REST API of cape sandbox is used)
6. Collect and store results

### Data storage
All the data are stored on NAS which is accessible from master machine in `/mnt/nfs`. For user of the data should be sufficient to know `/collecting` and maybe `/download2/samples`. Details below.  
- `/collecting` - collected results (subdirectories `none, internet, modified`)
- `/download2/archives` - downloaded archives
- `/download2/files` - all extracted files (including all types)
- `/download2/filtered` - filtered samples (exe, zip...)
- `/download2/samples` - reports from VT 

### Result structure
The result of analysis is the original directory created by sandbox, zipped.  

For extraction could be used something like:
```
for f in $source_dir/*/*/*.zip; do  unzip -p "$f" opt/CAPEv2/storage/analyses/*/reports/report.json >  "$extract_dir/"$(basename $f .zip)".json";done
```
The analysis directory can have kB but even hundreds of MB, for example compression of json files helps a lot.

## Useful commands
- `curl -F file=@/path/to/file http://<ip_address_of_slave>:8000/api/tasks/create/file/` - send single job for specific slave sandbox, result could be seen in browser on the same machine on port _8000_ or in `/opt/CAPEv2/storage/analyses/<analysis_id>` (be careful that the results of the analysis on the slave do not remain forever (they are periodically sent to the master))
