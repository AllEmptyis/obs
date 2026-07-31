nmcli conn show
nmcli conn up `인터페이스명`

ip 설정
nmcli conn modify `인터페이스` +ipv4.addr `ip` 
nmcli conn modify `인터페이스` -ipv4.addr `ip` 

nmcli conn modify ens160 +ipv4.gateway 192.168.212.1
nmcli conn modify ens160 +ipv4.dns 8.8.8.8

nmcli conn modify `인터페이스` ipv4.method manaul