```
[root@localhost scripts]# ./all_config.sh
-- Welcome to ticontroller configurator --

Select to configure
1) Config
2) Load
3) Save
4) Apply
5) Help
6) Quit
#? 1
[Step 1] Select to configuration mode
7) config             3) config_controller  5) config_es
8) config_ha          4) config_data
#? 4
 - Select mode : config_data

[Step 2] Network setup.
 - Select network interface.
1) ens33
2) lo
#? 1
 - Interface : ens33
 - Hostname : localhost.localdomain
  + Is the set hostname correct? (y/n) : y
 - Current settings network information.
  + IPADDR  : 192.168.212.63
  + NETMASK : 255.255.255.0
  + GATEWAY : 192.168.212.1
  + DNS1    : 8.8.8.8
  + DNS2    : 8.8.8.8
  + Is the network information correct? (y/n) : y
 - Set DB-HA node information.
Enter the total number of nodes : 1
  + Input check: 1
[ 1 Node ] Enter the node IP : 192.168.212.63
[ 1 Node ] Enter the node Hostname : 192.168.212.63
  + Input check: 192.168.212.63
  + Select DB-HA Web Node ip.
1) 192.168.212.63
#? 1
  + select node ip : 192.168.212.63
  + Select DB-HA Master Node ip.
1) 192.168.212.63
#? 1
  + select node ip : 192.168.212.63
  + Select Current Node ip.
1) 192.168.212.63
#? 1
  + current node ip : 192.168.212.63
 - Enter ticontroller domain name or service ip : 192.168.212.63
  + Input check: 192.168.212.63
 - Set smtp information.
  + SMTP Server :
  + SMTP Port :
  + SMTP User :
  + SMTP Password :

[Step 3] MariaDB setup.
 - MariaDB root account password
  + Enter password:

```