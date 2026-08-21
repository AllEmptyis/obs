# 로그
- `/opt/k2/var/log`
	- AEU가 생성하는 로그
- `/var/lib/tftpboot`
	- SFU가 생성하는 로그
	- 그 외 로그는 모두 aeu가 생성한 것

- 로그 설명
```
- /opt/k2/var/log/syslog : system 동작에 대한 전반적인 이벤트가 포함되어 있는 로그, 로그 분석 시 1순위로 보게 됨
	- 원래는 데몬별로 따로 로그 남김, 
	  syslog는 전부 합쳐서 출력
	  
- /opt/k2/var/log/messages : 실제 장비에서 show log로 확인이 되는 로그 정보, syslog에 비해 생략된 부분이 많음

- /opt/k2/var/log/clicmds : 사용자가 CLI 에서 커맨드 실행한 내역이 기록된 로그, 로그뒤에 보이는 초단위 시간은 해당 명령어가 실행되기까지의 시간을 의미

- /opt/k2/var/log/k2/amss.keepsfu.log : AEU -> SFU에 대한 H/C 상태 및 Fast Recovery 이벤트가 기록된 로그
  - H/C fail 기준은 icmp 3회, telnet 3회 그 이후 status bad로 변경
    이러한 로그가 없는 경우 SFR 아님
  
- /opt/k2/var/log/k2/amss.statistics.log : 1분단위로 통계데몬 실행 결과가 기록된 로그, 해당 로그 확인으로 AEU 정상 동작 여부 확인 가능
	- pask aeu 자체적으로 실행하는 통계 데몬. (게시판 통계 등)

- /opt/k2/var/log/k2/amss.log : 설정 적용 시 설정 데몬에 대한 실행 결과가 기록된 로그, 설정 오류 등이 난다면 해당 로그에서 상세 확인이 가능

- /opt/k2/var/log/k2/amss.firmware.log : OS 업데이트 이벤트에 대한 진행 사항이 기록된 로그, 보안적합성 모드라서 상세 OS 버전 확인이 불가할 경우 해당 로그에서 확인이 가능
	- pask는 보안적합성모드, 일반모드 두 가지 지원
```

- `/opt/k2/sfu.log`
	- sfu 로그를 aeu가 받아오는 것
	- 통신 상태 등에 따라 기록이 누락될 수 있음

- `amss.statistics.log`
	- AEU가 hang 걸렸는지 확인 가능

- `ipmitool`
	- H/W 이상이 있는 경우 해당 경로에 로그가 기록됨 (안나올 수도 있음)

- `ps`
	- 기술지원도우미 취득 당시 OS에서 ps 찍은 것
	- 참고: pask는 cpu 부하가 90% 이상인 경우 자동으로 RSS top 로그 찍음, 그 외에는 ps 로그를 참조.

- `/tmp/tech-assist-system-info`
	- running config 등 확인 전부 확인 가능
# 모델
k3 :앞자리 홀수 (8620 예외)
k2 : 앞자리 짝수
k4 :신모델

# SFR(SFU Fast Recovery)
- AEU는 SFU를 icmp- telnet 순서로 실시간 모니터링
- SFU가 응답이 없을 경우 SFU를 재기동하게 됨 (SFR 동작)

- SFR 발생 이유?
	- K2에서만 발생하는 이유 
	- K3에서는 발생하지 않음
	- K2에서 발생 원인은 확인되지 않음
- SFR 원인 확인
	- `/var/lib/tftpboot/sfu_log/sfu_backup/messages_etc`
```
Aug 11 13:24:54 UTC (none) kernel: reboot reason: Watchdog

AEU가 sfu에 대한 헬스체크가 실패해서 sfu 재기동 시킨 동작이 watchdog
```

- SFR 재기동 하면 원래는 SFU OS 재부팅이 됨, 따라서 이것을 방지하고자 추후 SWR 기능 개발

SWR
->SFU 일부 데몬만 재기동, 서비스에는 영향 없음
v2.2.4 이상부터 생긴 기능

SWR 기능을 사용하기 위해서는
CPLD (메인 보드bios 관련 펌웨어) 업데이트 필요
OS는 최신인데 CPLD가 작업이 안되어 있는 경우 SWR 안된다는 로그가 찍힘

K3에서 sfu 재기동 watchdog 발생하면 별도의 분석 필요.

SFU 발생 시 봐야 하는 것
- /opt/k2/var/log/syslog, /opt/k2/var/log/k2/amss.keepsfu.log (aeu)
- messages (sfu)



# 기타
- SFU와 AEU는 OS, H/W 모두 다르게 사용
- SFU가 부팅 중 cpu 부하가 일시적으로 많이 찰 수 있다 (문제인지 보려면 지속적인지 확인해야 함)
- `/var/lib/tftpboot/messages_all` 전체 부팅 로그
```
Aug 11 22:25:16 KST (none) kernel: [K2-M04/SYS:HISTORY] Switching port link UP!! (port="ge22")
Aug 11 22:25:17 KST (none) hwmon: [K2-M04/SYS:HISTORY] Event alarm generate (type="CPU",threshold="80 %",used_rate="93 %")
Aug 11 22:25:17 KST (none) login[1258]: [K2-M04/USER:HISTORY] Log In (user="root", from="pts/2")
Aug 11 22:25:17 KST (none) kernel: [K2-M04/SYS:HISTORY] Switching port link UP!! (port="ge18")
Aug 11 22:25:18 KST (none) kernel: [K2-M04/SYS:HISTORY] Switching port link UP!! (port="ge23")
Aug 11 22:25:18 KST (none) kernel: [K2-M04/SYS:HISTORY] Switching port link UP!! (port="ge19")
Aug 11 22:25:19 KST (none) kernel: [K2-M04/SYS:HISTORY] Switching port link UP!! (port="ge24"
```

- SFU는 재부팅 되면서 시간이 UTC로 변경, AEU는 KST 유지
- SFU는 재부팅 후에 AEU에 NTP를 맞추게 되는데 재부팅 되는 동안 시간 오차가 생길 수 있음
	- sfu.log 확인
```
2026/07/31 04:26:05 sfu (notice) keepaeu[1048]: [K3-K3L01ADC/SYS:HISTORY] Keepaeud port state (port="sy5(32)",state="FORWARDING")
2026/07/31 04:26:05 sfu (notice) keepaeu[1048]: [K3-K3L01ADC/SYS:HISTORY] Keepaeud port state (port="sy6(33)",state="FORWARDING")
2026/07/31 04:26:15 sfu (notice) login[2711]: [K3-K3L01ADC/USER:HISTORY] Log In (user="root", from="203.0.113.1")
2026/07/31 04:26:15 sfu (warning) IMISH[2711]: [USER:root]Enabled privilege mode 
2026/07/31 04:26:15 sfu (notice) IMISH[2711]: [K3-K3L01ADC/CONF:CONFIG] Enter Force configuration Mode (user="root",by="vty")
2026/07/31 04:26:24 sfu (notice) ntp_client_daemon: [K3-K3L01ADC/SYS:HISTORY] NTP Client (msg="Time adjusted offset -1708.482671 sec ")
//시간 보정 확인
2026/07/31 04:26:40 sfu (debug) xinetd[1026]: [K3-K3L01ADC/SYS:HISTORY] Xinetd Debug (msg="[server_end] telnet server ends")
2026/07/31 04:36:25 sfu (notice) ntp_client_daemon: [K3-K3L01ADC/SYS:HISTORY] NTP Client (msg="Time adjusted offset -0.046819 sec ")
```

- sy 인터페이스
	- aeu와 sfu간 통신하는 인터페이스
	- 해당 포트가 link up 될 때 sfu가 부팅 되어 연동이 다시 시작된 것
```
2026/07/31 04:11:40 sfu (notice) ntp_client_daemon: [K3-K3L01ADC/SYS:HISTORY] NTP Client (msg="Time adjusted offset -0.027462 sec ")
2026/07/31 04:21:40 sfu (notice) ntp_client_daemon: [K3-K3L01ADC/SYS:HISTORY] NTP Client (msg="Time adjusted offset -0.027044 sec ")

2026/07/31 04:24:30 sfu (err) kernel: [K3-K3L01ADC/SYS:HISTORY] Switching port link UP!! (port="sy2")
2026/07/31 04:24:30 sfu (warning) RMON[965]: Port up notification received for port sy2 <---확인
2026/07/31 04:24:30 sfu (warning) ONMD[996]: Port up notification received for port sy2.(flags : up 1, running 1) 
```

- SFU와 AEU가 따로 재부팅 발생할 수 있음
	- SFU만 재기동 된 경우 - cli상으로는 정상이나 link가 다운됨
	- aeu만 재기동 된 경우 - cli상 부팅 로그가 올라온다

## mbc
- AEU 재부팅 로그
	- PI is unhealty: AEU와 SFU간 물리적인 통신 인터페이스 자체가 문제 생긴 것
```
2026/08/10 00:50:01 SABIIDLB4824B_08FM (info) rsyslogd: [origin software="rsyslogd" swVersion="4.6.4" x-pid="8491" x-info="http://www.rsyslog.com"] rsyslogd was HUPed, type 'lightweight'.
2026/08/10 04:24:07 SABIIDLB4824B_08FM (err) hwmon: PI is unhealthy (ixg4(3) )
2026/08/10 04:24:07 SABIIDLB4824B_08FM (notice) hwmon: state_check (AE, 1)
2026/08/10 04:24:16 SABIIDLB4824B_08FM (err) hwmon: PI is unhealthy (ixg4(4) )
2026/08/10 04:24:26 SABIIDLB4824B_08FM (err) hwmon: PI is unhealthy (ixg4(5) )
2026/08/10 04:24:37 SABIIDLB4824B_08FM (err) hwmon: PI is unhealthy (ixg4(6) )
2026/08/10 04:24:47 SABIIDLB4824B_08FM (err) hwmon: PI is unhealthy (ixg4(7) )
2026/08/10 04:24:57 SABIIDLB4824B_08FM (err) hwmon: PI is unhealthy (ixg4(8) )
2026/08/10 04:25:07 SABIIDLB4824B_08FM (err) hwmon: PI is unhealthy (ixg4(9) )
2026/08/10 04:25:17 SABIIDLB4824B_08FM (err) hwmon: PI is unhealthy (ixg4(10) )
2026/08/10 04:25:27 SABIIDLB4824B_08FM (err) hwmon: PI is unhealthy (ixg4(11) )
2026/08/10 04:25:37 SABIIDLB4824B_08FM (err) hwmon: PI is unhealthy (ixg4(12) )
```

- AEU 커널 패닉으로 인해 SFU도 같이 재부팅 시킨 것
```
2026/08/10 04:28:40 SABIIDLB4824B_08FM (notice) [amss.driver.reboot_driver] reboot sfu
2026/08/10 04:28:40 SABIIDLB4824B_08FM (err) hwmon: failed to read socket
2026/08/10 04:28:40 SABIIDLB4824B_08FM (notice) [amss.driver.reboot_driver] reboot aeu
2026/08/10 04:28:40 SABIIDLB4824B_08FM (notice) [amss.driver.reboot_driver] Soft reset booting!
```

- SFU 로그 확인
	- reboot reason - s/w command : AEU가 강제 재부팅 시켰기 때문 (reboot sfu)
	- 즉 AEU가 먼저 커널 패닉이 발생하여 SFU도 같이 재기동, 단 로그 상 SFU는 전체 재부팅이 아닌 OS만 재기동 된 것으로 보임
		- AEU는 보통 전체 재부팅되고, SFU는 OS만 재부팅 되는 경우가 있음
```
Aug 09 19:31:34 UTC (none) syslogd: sendto: Network is unreachable
Aug 09 19:31:34 UTC (none) sfu: (Switch Log) Boot Time: 2026/08/ 9 19:31:34 UTC
Aug 09 19:31:34 UTC (none) kernel: klogd started: BusyBox v1.2.1 (2011.06.17-01:36+0000)
Aug 09 19:31:34 UTC (none) kernel: Linux version 2.6.27.39 (releaser@ss-comp-k2) (gcc version 4.3.2 (Wind River Linux Sourcery G++ 4.3-237) ) #PLOS-PASK-SFU-V30.9.5.0.0-rc0 Wed Oct 18 19:01:16 KST 2023
Aug 09 19:31:34 UTC (none) kernel: Kernel command line: 
Aug 09 19:31:34 UTC (none) kernel: RTC get time 2026.8.9 19:31.24.-2142787632
Aug 09 19:31:34 UTC (none) kernel: registering PCI controller with io_map_base unset
Aug 09 19:31:34 UTC (none) kernel: registering PCI controller with io_map_base unset
Aug 09 19:31:34 UTC (none) kernel: Physically mapped flash: CFI does not contain boot bank location. Assuming top.
Aug 09 19:31:34 UTC (none) kernel: number of CFI chips: 1
Aug 09 19:31:34 UTC (none) kernel: cfi_cmdset_0002: Disabling erase-suspend-program due to code brokenness.
Aug 09 19:31:34 UTC (none) kernel: Flash device: 0x4000000 at 0x20000000
Aug 09 19:31:34 UTC (none) kernel: ically mapped flash":
Aug 09 19:31:34 UTC (none) kernel: 0x00000000-0x000a0000 : "boot"
Aug 09 19:31:34 UTC (none) kernel: 0x000a0000-0x028a0000 : "os"
Aug 09 19:31:34 UTC (none) kernel: 0x028a0000-0x028c0000 : "nvram"
Aug 09 19:31:34 UTC (none) kernel: 0x028c0000-0x028e0000 : "xqc_env"
Aug 09 19:31:34 UTC (none) kernel: 0x028e0000-0x02920000 : "nmi_log"
Aug 09 19:31:34 UTC (none) kernel: 0x02920000-0x03640000 : "cfg"
Aug 09 19:31:34 UTC (none) kernel: 0x03640000-0x04000000 : "backup"
Aug 09 19:31:34 UTC (none) kernel: slram: not enough parameters.

Aug 09 19:31:34 UTC (none) kernel: (Switch Log) reboot reason: S/W Reboot Command

Aug 09 19:31:34 UTC (none) kernel: CONFIG_NF_CT_ACCT is deprecated and will be removed soon. Plase use
Aug 09 19:31:34 UTC (none) kernel: nf_conntrack.acct=1 kernel paramater, acct=1 nf_conntrack module option or
Aug 09 19:31:34 UTC (none) kernel: sysctl net.netfilter.nf_conntrack_acct=1 to enable it.
```

- 커널 크래시 로그
	- 아래 로그가 있는 경우 하드웨어 문제
	- 커널 크래시 로그 파일은 업타임 기준 마지막으로 부팅 된 날짜로 기록됨.
```
<0>[29859259.818021] No human readable MCE decoding support on this CPU type.
<0>[29859259.818366] Run the message through 'mcelog --ascii' to decode.

<0>[29859259.818688] This is not a software problem!

<0>[29859259.818924] Machine check: Processor context corrupt
<0>[29859259.819201] Kernel panic - not syncing: Fatal Machine check
```

- AEU-> SFU 헬스체크도 있듯이 SFU>AEU 헬스체크도 존재함
	- 마찬가지로 일정 수치 이상 실패하면 AEU 복구 재부팅 가능
	- 이런 경우 sfu log에서 확인 가능
	- 아래 로그가 SFU>AEU 헬스체크 실패
```
Jun 15 01:19:59 UTC (none) keepaeud[920]: [ICMP] Error.. not ICMP_ECHOREPLY (8) len 172 
Jun 15 01:19:59 UTC (none) keepaeud[920]: [ICMP] failed to recv_icmp 
Jun 16 23:54:00 UTC (none) keepaeud[920]: [ICMP] Error.. not ICMP_ECHOREPLY (8) len 172 
Jun 16 23:54:00 UTC (none) keepaeud[920]: [ICMP] failed to recv_icmp 
Jun 25 04:26:00 UTC (none) keepaeud[920]: [ICMP] Error.. not ICMP_ECHOREPLY (8) len 172 
Jun 25 04:26:00 UTC (none) keepaeud[920]: [ICMP] failed to recv_icmp 
Jul 01 15:37:08 UTC (none) keepaeud[920]: [ICMP] Error.. not ICMP_ECHOREPLY (8) len 172 
Jul 01 15:37:08 UTC (none) keepaeud[920]: [ICMP] failed to recv_icmp 
Jul 18 14:34:15 UTC (none) keepaeud[920]: [ICMP] Error.. not ICMP_ECHOREPLY (8) len 172 
Jul 18 14:34:15 UTC (none) keepaeud[920]: [ICMP] failed to recv_icmp 
Jul 20 07:49:59 UTC (none) keepaeud[920]: [ICMP] Error.. not ICMP_ECHOREPLY (8) len 172 
Jul 20 07:49:59 UTC (none) keepaeud[920]: [ICMP] failed to recv_icmp 
Aug 02 06:54:56 UTC (none) keepaeud[920]: [ICMP] Error.. not ICMP_ECHOREPLY (8) len 172 
Aug 02 06:54:56 UTC (none) keepaeud[920]: [ICMP] failed to recv_icmp 
```

- keep sfu 확인
	- H/C 실패 로그 없음 - 즉 AEU에서 그냥 커널 패닉 발생해서 다운 된 것
```
Mon Aug 10 04:29:53 UTC-9 2026
2026-08-10 04:29:53,933 140383740573472 amss.keepsfu                        INFO     run:222 - KeepSFU Start with reset=True
2026-08-10 04:29:53,940 140383740573472 amss.keepsfu                        INFO     check_icmp:457 - success to check SFU(icmp)
2026-08-10 04:29:55,530 140383740573472 amss.keepsfu                        INFO     check_telnet:511 - success to check SFU(telnet)
2026-08-10 04:29:55,531 140383740573472 amss.keepsfu                        INFO     check_telnet:518 - set flag for available statistics status
2026-08-10 04:29:55,636 140383740573472 amss.common.sysinfo                 INFO     load_product_info:1147 - init board_info
2026-08-10 04:29:55,817 140383740573472 amss.common.sysinfo                 INFO     load_product_info:1151 - init product_name
2026-08-10 04:29:55,970 140383740573472 amss.common.sysinfo                 INFO     load_product_info:1154 - init product_type
2026-08-10 04:29:56,142 140383740573472 amss.common.sysinfo                 INFO     load_product_info:1157 - init vendor
2026-08-10 04:29:57,440 140383740573472 amss.keepsfu                        NOTICE   restore_system:705 - SFU booted!!!
2026-08-10 04:29:57,470 140383740573472 amss.keepsfu                        NOTICE   restore_system:710 - init AEU
2026-08-10 04:29:57,471 140383740573472 amss.keepsfu                        INFO     reload_aeu:659 - start k2_start
2026-08-10 04:30:42,929 140383740573472 amss.keepsfu                        NOTICE   restore_system:727 - init SFU
2026-08-10 04:31:51,399 140383740573472 amss.keepsfu                        INFO     _check_sfu_warm_ctrl:624 - Check warm_support: No
2026-08-10 04:31:51,400 140383740573472 amss.common.sysmng                  INFO     ctrl_amss_init_done:48 - ---> CTRL AMSS INIT STAT(True)
2026-08-10 04:31:51,413 140383740573472 amss.keepsfu                        INFO     _sfu_config_set_restore:560 - send config-set-restore completed
2026-08-10 04:32:00,254 140383740573472 amss.keepsfu                        INFO     run:322 - set flag for sfu booting done
2026-08-10 04:32:00,255 140383740573472 amss.keepsfu                        NOTICE   run:324 - SFU State goes to GOOD in 0:01:57.466692
```

- aeu statistics 로그
```
2026-08-10 04:24:34,491 140660983060224 statistics.base.databasewriter      INFO     _run:329 - Statistical information has been cleaned up successfully.(0.00315499305725 duration)
2026-08-10 04:25:35,570 140660983060224 statistics.base.databasewriter      INFO     _run:329 - Statistical information has been cleaned up successfully.(0.00215101242065 duration)
2026-08-10 04:26:35,642 140660983060224 statistics.base.databasewriter      INFO     _run:329 - Statistical information has been cleaned up successfully.(0.00302791595459 duration)
2026-08-10 04:27:35,713 140660983060224 statistics.base.databasewriter      INFO     _run:329 - Statistical information has been cleaned up successfully.(0.00367593765259 duration)
2026-08-10 04:28:35,783 140660983060224 statistics.base.databasewriter      INFO     _run:329 - Statistical information has been cleaned up successfully.(0.00297784805298 duration)
Mon Aug 10 04:29:58 UTC-9 2026
2026-08-10 04:30:04,522 140508196009760 amss.statistics.apploader           INFO     load_apps:70 - imported statistics.app.advl7service_app //데몬 재시작 발생
2026-08-10 04:30:04,523 140508196009760 amss.statistics.apploader           INFO     load_apps:70 - imported statistics.app.l7service_app
2026-08-10 04:30:04,530 140508196009760 amss.statistics.apploader           INFO     load_apps:70 - imported statistics.app.layer7cache_app
2026-08-10 04:30:04,553 140508196009760 statistics.app.watchstorage_app     INFO     _set_goal_interval:104 - set_goal_interval(type: 3s, goal: None)
2026-08-10 04:30:04,553 140508196009760 statistics.app.watchstorage_app     INFO     _set_goal_interval:104 - set_goal_interval(type: 5m, goal: None)
2026-08-10 04:30:04,553 140508196009760 statistics.app.watchstorage_app     INFO     _set_goal_interval:104 - set_goal_interval(type: 10m, goal: None)
2026-08-10 04:30:04,561 140508196009760 amss.statistics.apploader           INFO     load_apps:70 - imported statistics.app.watchstorage_app
2026-08-10 04:30:04,595 140508196009760 amss.statistics.apploader           INFO     load_apps:70 - imported statistics.app.advl4service_app
2026-08-10 04:30:04,595 140508196009760 amss.statistics.apploader           ERROR    load_apps:73 - import statistics.app.vlanmonitoring_app error(vlan monitoring_app use only in MII type)
2026-08-10 04:30:04,596 140508196009760 amss.statistics.apploader           ERROR    load_apps:73 - import statistics.app.portmonitoring_app error(vlan monitoring_app use only in VA type)
2026-08-10 04:30:04,597 140508196009760 amss.statistics.apploader           INFO     load_apps:70 - imported statistics.app.gslbservice_app
2026-08-10 04:30:04,622 140508196009760 amss.statistics.apploader           INFO     load_apps:70 - imported statistics.app.resource_app
2026-08-10 04:30:04,623 140508196009760 amss.statistics.apploader           INFO     load_apps:70 - imported statistics.app.sfu_app
2026-08-10 04:30:04,624 140508196009760 amss.statistics.apploader           INFO     load_apps:70 - imported statistics.app.l4service_app
2026-08-10 04:30:04,649 140508196009760 amss.statistics.apploader           INFO     <module>:114 - Running Statistics Daemon.
2026-08-10 04:30:09,689 140507992180480 statistics.base.databasewriter      INFO     _run:329 - Statistical information has been cleaned up successfully.(0.00899505615234 duration)
2026-08-10 04:31:09,752 140507992180480 statistics.base.databasewriter      INFO     _run:329 - Statistical information has been cleaned up successfully.(0.0029628276825 duration
```



# 일감 작성
- K2 사이트
- 담당자- 커널1팀
- 범주- 사이트명 있는 경우 선택
- 버전, S/N 기재
- 템플릿에서 필요한 내용 위주 기재
	- -단순 A/S 인 경우 템플릿 문의-H/W
	- 그 외에는 S/W
- 로그 설명을 쓸 필요 없음

- SFR 이슈는 일감, case open 둘 다 장애로 처리
	- 일감은 H/W 문의 말고 S/W 문의로 등록
	- SFR 이슈는 증적 필요하여 이렇게 처리함

- 기능 요구
	- 기능 요구의 경우 일정 확인이 중요 (몇 분기까지 개발 완료되어야 하는지)
	- 최대한 자세하게 기록
	- 사이트와 연구소 간 일정 조율 필요

- 일감 유형
	- 문의 - 기능문의
	- S/W - 소프트웨어 버그 등, 장애 , 분석요청
	- 요구 - 요구 사항