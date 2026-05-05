# 사전 설정 - 커널 로그
- shell 진입 후 아래와 같이 입력
```
/* xswitch rx되는 packet 확인 하기 위해 로그 출력 되도록 설정 */
echo 1 > /proc/layer2/debug_rx
echo 8 > /proc/layer3/debug_rx
echo 8 > /proc/layer3/debug_rxpkt

참고일감: https://redmine.piolink.com/issues/102012
```

# RTK 테스트
## 설정
- 버전: v3.2.17.5.0
- 아래와 동일한 조건으로 테스트 했을 때 ping이 나가지 않음
```
Ping 10.10.10.1 32바이트 데이터 사용:
요청 시간이 만료되었습니다.
요청 시간이 만료되었습니다.
요청 시간이 만료되었습니다.
요청 시간이 만료되었습니다.
요청 시간이 만료되었습니다.
```
- 설정
```
TiFRONT% sh vlan
 ---------------------------------------------------------------------------------
  PORT            | ge                                              | xg
                  |                   1 1 1 1 1 1 1 1 1 1 2 2 2 2 2 |
                  | 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 | 1 2 3 4
 -----------------+-------------------------------------------------+-----------
  SWITCH MODE     | T A H H H H H H H H H H H H H H H H H H H H H H | H H H H
 -----------------+-------------------------------------------------+-----------
  default  (   1) | T . U U U U U U U U U U U U U U U U U U U U U U | U U U U
  VLAN0010 (  10) | t U . . . . . . . . . . . . . . . . . . . . . . | . . . .
 ---------------------------------------------------------------------------------

----------------------
TiFRONT% sh running-config
!
hostname TiFRONT
!
!
username admin privilege 15 password 8 $6$e74c52ec3a186bbf03c36a206b7f51d7fde4b18a40c7662508cb7542d0678c32caa7a6342f01fb3b0eb90d7080c16722e2c8db64b194f5dc2cca34182d5d48cb
!
remote-reset enable
no ip forwarding
ecmp ip-src
!
timezone kst
!
spanning-tree mode rpvst+
!
vlan 10  name VLAN0010
vlan classifier rule 1 ipv4 10.10.10.10/24 vlan 10
vlan classifier group 1 add rule 1
!
spanning-tree vlan 10
!
interface ge1
 no shutdown
 switchport
 switchport mode trunk
 switchport trunk allowed vlan add 1
 switchport trunk allowed vlan add 10
 flowcontrol send off
 flowcontrol receive off
 vlan classifier activate 1
 jumbo-frame off
 set lldp enable txrx
 lldp tlv chassis-id port-id ttl system-capabilities
 spanning-tree vlan 10
```
## 로그
- ping 10.10.10.20->10.10.10.1
	- 통신 안 됨
```
1970 Jan 01 09:13:52 KST (none) kernel: [l3_pkt_vif_rx_deferred:1432] RX vlan 1, devname vlan1
1970 Jan 01 09:13:52 KST (none) kernel: [XSW][RXPKT][INFO][l3_pkt_vif_rx_deferred:1435] PKT PASS TO KERNEL, RX vlan 1, devname vlan1, protocol 806
1970 Jan 01 09:13:53 KST (none) kernel: [XSW][RXPKT][WARN][oprt_rtk_l2pkt_rx_cb:206] RX RSN FILTER 0x5000 0x0
1970 Jan 01 09:13:53 KST (none) kernel: [XSW][RXPKT][INFO][oprt_rtk_l2pkt_rx_cb:208] LPORT(0x08000001)
1970 Jan 01 09:13:53 KST (none) kernel: [XSW][RXPKT][INFO][opr_rtk_common_pkt_dump_cputag:654] === [NIC RX Debug - CPU Rx Tag Information] ============
1970 Jan 01 09:13:53 KST (none) kernel: [XSW][RXPKT][INFO][opr_rtk_common_pkt_dump_cputag:700]  RPN : 1
1970 Jan 01 09:13:53 KST (none) kernel: [XSW][RXPKT][INFO][opr_rtk_common_pkt_dump_cputag:701]  UNIT : 0
1970 Jan 01 09:13:53 KST (none) kernel: [XSW][RXPKT][INFO][opr_rtk_common_pkt_dump_cputag:702]  SPN : 1
1970 Jan 01 09:13:53 KST (none) kernel: [XSW][RXPKT][INFO][opr_rtk_common_pkt_dump_cputag:703]  OF_LU_MIS_TBL_ID : 0
1970 Jan 01 09:13:53 KST (none) kernel: [XSW][RXPKT][INFO][opr_rtk_common_pkt_dump_cputag:704]  MIR_HIT : 0
1970 Jan 01 09:13:53 KST (none) kernel: [XSW][RXPKT][INFO][opr_rtk_common_pkt_dump_cputag:710]  SFLOW_HIT : 0
1970 Jan 01 09:13:53 KST (none) kernel: [XSW][RXPKT][INFO][opr_rtk_common_pkt_dump_cputag:711]  ACL_HIT : 1
1970 Jan 01 09:13:53 KST (none) kernel: [XSW][RXPKT][INFO][opr_rtk_common_pkt_dump_cputag:712]  OF_HIT : 0
1970 Jan 01 09:13:53 KST (none) kernel: [XSW][RXPKT][INFO][opr_rtk_common_pkt_dump_cputag:713]  IDX : 255
1970 Jan 01 09:13:53 KST (none) kernel: [XSW][RXPKT][INFO][opr_rtk_common_pkt_dump_cputag:714]  TT_HIT : 0
1970 Jan 01 09:13:53 KST (none) kernel: [XSW][RXPKT][INFO][opr_rtk_common_pkt_dump_cputag:715]  TT_IDX : 0
1970 Jan 01 09:13:53 KST (none) kernel: [XSW][RXPKT][INFO][opr_rtk_common_pkt_dump_cputag:716]  OF_TBL_ID : 0
1970 Jan 01 09:13:53 KST (none) kernel: [XSW][RXPKT][INFO][opr_rtk_common_pkt_dump_cputag:717]  IS_TRK : 0
1970 Jan 01 09:13:53 KST (none) kernel: [XSW][RXPKT][INFO][opr_rtk_common_pkt_dump_cputag:718]  TRK_ID : 0
1970 Jan 01 09:13:53 KST (none) kernel: [XSW][RXPKT][INFO][opr_rtk_common_pkt_dump_cputag:719]  OTAGIF : 0
1970 Jan 01 09:13:53 KST (none) kernel: [XSW][RXPKT][INFO][opr_rtk_common_pkt_dump_cputag:720]  ITAGIF : 0
1970 Jan 01 09:13:53 KST (none) kernel: [XSW][RXPKT][INFO][opr_rtk_common_pkt_dump_cputag:721]  OVID : 0
1970 Jan 01 09:13:53 KST (none) kernel: [XSW][RXPKT][INFO][opr_rtk_common_pkt_dump_cputag:722]  IVID : 0
1970 Jan 01 09:13:53 KST (none) kernel: [XSW][RXPKT][INFO][opr_rtk_common_pkt_dump_cputag:723]  FWD_VID_SEL : 0
1970 Jan 01 09:13:53 KST (none) kernel: [XSW][RXPKT][INFO][opr_rtk_common_pkt_dump_cputag:724]  FVID : 1
1970 Jan 01 09:13:53 KST (none) kernel: [XSW][RXPKT][INFO][opr_rtk_common_pkt_dump_cputag:725]  QID : 3
1970 Jan 01 09:13:53 KST (none) kernel: [XSW][RXPKT][INFO][opr_rtk_common_pkt_dump_cputag:726]  ATK_TYPE : 0
1970 Jan 01 09:13:53 KST (none) kernel: [XSW][RXPKT][INFO][opr_rtk_common_pkt_dump_cputag:727]  MAC_CST : 0
1970 Jan 01 09:13:53 KST (none) kernel: [XSW][RXPKT][INFO][opr_rtk_common_pkt_dump_cputag:728]  DM_RXIDX : 63
1970 Jan 01 09:13:53 KST (none) kernel: [XSW][RXPKT][INFO][opr_rtk_common_pkt_dump_cputag:729]  NEW_SA : 0
1970 Jan 01 09:13:53 KST (none) kernel: [XSW][RXPKT][INFO][opr_rtk_common_pkt_dump_cputag:730]  PMV_FORBID : 0
1970 Jan 01 09:13:53 KST (none) kernel: [XSW][RXPKT][INFO][opr_rtk_common_pkt_dump_cputag:731]  L2_STTC_PMV : 0
1970 Jan 01 09:13:53 KST (none) kernel: [XSW][RXPKT][INFO][opr_rtk_common_pkt_dump_cputag:732]  L2_DYN_PMV : 0
1970 Jan 01 09:13:53 KST (none) kernel: [XSW][RXPKT][INFO][opr_rtk_common_pkt_dump_cputag:733]  L2_ERR_PKT : 0
1970 Jan 01 09:13:53 KST (none) kernel: [XSW][RXPKT][INFO][opr_rtk_common_pkt_dump_cputag:734]  L3_ERR_PKT : 0
1970 Jan 01 09:13:53 KST (none) kernel: [XSW][RXPKT][INFO][opr_rtk_common_pkt_dump_cputag:735]  OVERSIZE : 0
1970 Jan 01 09:13:53 KST (none) kernel: [XSW][RXPKT][INFO][opr_rtk_common_pkt_dump_cputag:740]  ext_unit : 0
1970 Jan 01 09:13:53 KST (none) kernel: [XSW][RXPKT][INFO][opr_rtk_common_pkt_dump_cputag:741]  ext_source_port : 1
1970 Jan 01 09:13:53 KST (none) kernel: [XSW][RXPKT][INFO][opr_rtk_common_pkt_dump_cputag:743]  HASH_FULL : 0
1970 Jan 01 09:13:53 KST (none) kernel: [XSW][RXPKT][INFO][opr_rtk_common_pkt_dump_cputag:744]  INVALID_SA : 0
1970 Jan 01 09:13:53 KST (none) kernel: [XSW][RXPKT][INFO][opr_rtk_common_pkt_dump_cputag:745]  REASON : 0
1970 Jan 01 09:13:53 KST (none) kernel: [RX PKT][24:f5:aa:ce:4f:b0->ff:ff:ff:ff:ff:ff][UNTAGGED][ARP][(1:2048:1540:1)(24:f5:aa:ce:4f:b0->00:00:00:00:00:00)(10.10.10.20/32->10.10.10.10/32)]
1970 Jan 01 09:13:53 KST (none) kernel: [XSW][RXPKT][INFO][oprt_rtk_l2pkt_rx_cb:212] LAYER2 RX_REASON(0x00000000)
1970 Jan 01 09:13:53 KST (none) kernel: [XSW][RXPKT][INFO][oprt_rtk_l2pkt_rx_cb:217] reasons: NIC_RX_REASON_ACL_HIT(91)
1970 Jan 01 09:13:53 KST (none) kernel: [0x02] l2_pkt_rx_cb() : PORT: 8000001, LEN: 64, VID: 1, REASON: 0x5000, COS: 3
1970 Jan 01 09:13:53 KST (none) kernel: [0x02] l2_pkt_rx_cb() : MAC_DST ff:ff:ff:ff:ff:ff MAC_SRC 24:f5:aa:ce:4f:b0
1970 Jan 01 09:13:53 KST (none) kernel: [XSW][RXPKT][DBG0][l2_if_arp_rx:981] CHECK RX DEV 0x08000001 80b8d000 80b8d000
1970 Jan 01 09:13:53 KST (none) kernel: [XSW][RXPKT][DBG0][l2_if_arp_rx:982] CHECK RX DEV ge1
1970 Jan 01 09:13:53 KST (none) kernel: [l2_if_arp_rx] RX: port 134217729, pktlen 64
1970 Jan 01 09:13:53 KST (none) kernel:
1970 Jan 01 09:13:53 KST (none) kernel:    0 : ff ff ff ff ff ff 24 f5 aa ce 4f b0 81 00 00 01
1970 Jan 01 09:13:53 KST (none) kernel:   10 : 08 06 00 01 08 00 06 04 00 01 24 f5 aa ce 4f b0
1970 Jan 01 09:13:53 KST (none) kernel:   20 : 0a 0a 0a 14 00 00 00 00 00 00 0a 0a 0a 0a 00 00
1970 Jan 01 09:13:53 KST (none) kernel:   30 : 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
1970 Jan 01 09:13:53 KST (none) kernel: [0x02] br_handle_frame() : Frame recvd on ifindex ge1
1970 Jan 01 09:13:53 KST (none) kernel: [0x02] br_handle_frame() : applied ingress rules and the frames vid(1) instance (0)
1970 Jan 01 09:13:53 KST (none) kernel: [0x02] br_handle_frame() : learn 1 and forward 1 for instance 0
1970 Jan 01 09:13:53 KST (none) kernel: [0x02] br_handle_frame() : XXX ff ff ff ff ff ff : 245 : 3
1970 Jan 01 09:13:53 KST (none) kernel: [0x02] br_handle_frame() : GARP Broadcast PDU received  3
1970 Jan 01 09:13:53 KST (none) kernel: [XSW][RXPKT][INFO][opr_rtk_common_pkt_rsn_check:342] =========> COMPARE Pkt reasons: 14
1970 Jan 01 09:13:53 KST (none) kernel: [XSW][RXPKT][INFO][oprt_rtk_l3pkt_rx_cb:162] PORT(0x08000001), ge1
1970 Jan 01 09:13:53 KST (none) kernel: [XSW][RXPKT][INFO][oprt_rtk_l3pkt_rx_cb:164] LAYER3 RX_REASON(0x00000000)
1970 Jan 01 09:13:53 KST (none) kernel: [XSW][RXPKT][INFO][oprt_rtk_l3pkt_rx_cb:170] reasons: NIC_RX_REASON_ACL_HIT(91)
1970 Jan 01 09:13:53 KST (none) kernel: [XSW][RXPKT][DBG0][l3_pkt_rx:1606] l2if(ge1) l3if(vlan1) PTYPE(0x00000806)
1970 Jan 01 09:13:53 KST (none) kernel: [XSW][RXPKT][INFO][l3_pkt_vif_handle_check:1176] MATCH(0) BE PASSED TO THE KERNEL. RX_CTAG(0)
1970 Jan 01 09:13:53 KST (none) kernel: [l3_pkt_vif_rx_deferred:1339] Port 8000001, vid 1, reason 0x0, size 64PKT TYPE(0x806:ARP)
```

# RTK 패치 버전 업데이트 후 동작 확인
- 버전
	- PLOS-CSrSD-V1.0.0-rc1002
- 설정 및 구성 - 동일
## 로그
### classifier 설정X
```
1970 Jan 01 09:06:52 KST (none) kernel: [0x02] br_handle_frame() : Frame recvd on ifindex ge1
1970 Jan 01 09:06:52 KST (none) kernel: [0x02] br_handle_frame() : applied ingress rules and the frames vid(1) instance (0)
1970 Jan 01 09:06:52 KST (none) kernel: [0x02] br_handle_frame() : learn 1 and forward 1 for instance 0
1970 Jan 01 09:06:52 KST (none) kernel: [0x02] br_handle_frame() : XXX ff ff ff ff ff ff : 245 : 3
1970 Jan 01 09:06:52 KST (none) kernel: [0x02] br_handle_frame() : GARP Broadcast PDU received  3
1970 Jan 01 09:06:52 KST (none) kernel: [XSW][RXPKT][INFO][opr_rtk_common_pkt_rsn_check:342] =========> COMPARE Pkt reasons: 14
1970 Jan 01 09:06:52 KST (none) kernel: [XSW][RXPKT][INFO][oprt_rtk_l3pkt_rx_cb:161] PORT(0x08000001), ge1
1970 Jan 01 09:06:52 KST (none) kernel: [XSW][RXPKT][INFO][oprt_rtk_l3pkt_rx_cb:163] LAYER3 RX_REASON(0x00000000)
1970 Jan 01 09:06:52 KST (none) kernel: [XSW][RXPKT][INFO][oprt_rtk_l3pkt_rx_cb:169] reasons: NIC_RX_REASON_ACL_HIT(91)
1970 Jan 01 09:06:52 KST (none) kernel: [XSW][RXPKT][DBG0][l3_pkt_rx:1541] l2if(ge1) l3if(vlan1) PTYPE(0x00000806)
1970 Jan 01 09:06:52 KST (none) kernel: [XSW][RXPKT][INFO][l3_pkt_vif_handle_check:1111] MATCH(0) BE PASSED TO THE KERNEL. RX_CTAG(0)
1970 Jan 01 09:06:52 KST (none) kernel: [l3_pkt_vif_rx_deferred:1274] Port 8000001, vid 1, reason 0x0, size 64PKT TYPE(0x806:ARP)
1970 Jan 01 09:06:52 KST (none) kernel: -----------------
1970 Jan 01 09:06:52 KST (none) kernel: ff ff ff ff ff ff 24 f5 aa ce
1970 Jan 01 09:06:52 KST (none) kernel: 4f b0 81 00 00 01 08 06 00 01
1970 Jan 01 09:06:52 KST (none) kernel: 08 00 06 04 00 01 24 f5 aa ce
1970 Jan 01 09:06:52 KST (none) kernel: 4f b0 0a 0a 0a 14 00 00 00 00
1970 Jan 01 09:06:52 KST (none) kernel: 00 00 0a 0a 0a 01 00 00 00 00
1970 Jan 01 09:06:52 KST (none) kernel: 00 00 00 00 00 00 00 00 00 00
1970 Jan 01 09:06:52 KST (none) kernel: 00 00 00 00
1970 Jan 01 09:06:52 KST (none) kernel: [l3_pkt_vif_rx_deferred:1367] RX vlan 1, devname vlan1
1970 Jan 01 09:06:52 KST (none) kernel: [XSW][RXPKT][INFO][l3_pkt_vif_rx_deferred:1370] PKT PASS TO KERNEL, RX vlan 1, devname vlan1, protocol 806
1970 Jan 01 09:06:53 KST (none) kernel: [XSW][RXPKT][WARN][oprt_rtk_l2pkt_rx_cb:204] RX RSN FILTER 0x5000 0x0
1970 Jan 01 09:06:53 KST (none) kernel: [XSW][RXPKT][INFO][oprt_rtk_l2pkt_rx_cb:206] LPORT(0x08000001)
1970 Jan 01 09:06:53 KST (none) kernel: [XSW][RXPKT][INFO][opr_rtk_common_pkt_dump_cputag:654] === [NIC RX Debug - CPU Rx Tag Information] ============
1970 Jan 01 09:06:53 KST (none) kernel: [XSW][RXPKT][INFO][opr_rtk_common_pkt_dump_cputag:700]  RPN : 1
1970 Jan 01 09:06:53 KST (none) kernel: [XSW][RXPKT][INFO][opr_rtk_common_pkt_dump_cputag:701]  UNIT : 0
1970 Jan 01 09:06:53 KST (none) kernel: [XSW][RXPKT][INFO][opr_rtk_common_pkt_dump_cputag:702]  SPN : 1
1970 Jan 01 09:06:53 KST (none) kernel: [XSW][RXPKT][INFO][opr_rtk_common_pkt_dump_cputag:703]  OF_LU_MIS_TBL_ID : 0
1970 Jan 01 09:06:53 KST (none) kernel: [XSW][RXPKT][INFO][opr_rtk_common_pkt_dump_cputag:704]  MIR_HIT : 0
1970 Jan 01 09:06:53 KST (none) kernel: [XSW][RXPKT][INFO][opr_rtk_common_pkt_dump_cputag:710]  SFLOW_HIT : 0
1970 Jan 01 09:06:53 KST (none) kernel: [XSW][RXPKT][INFO][opr_rtk_common_pkt_dump_cputag:711]  ACL_HIT : 1
1970 Jan 01 09:06:53 KST (none) kernel: [XSW][RXPKT][INFO][opr_rtk_common_pkt_dump_cputag:712]  OF_HIT : 0
1970 Jan 01 09:06:53 KST (none) kernel: [XSW][RXPKT][INFO][opr_rtk_common_pkt_dump_cputag:713]  IDX : 383
1970 Jan 01 09:06:53 KST (none) kernel: [XSW][RXPKT][INFO][opr_rtk_common_pkt_dump_cputag:714]  TT_HIT : 0
1970 Jan 01 09:06:53 KST (none) kernel: [XSW][RXPKT][INFO][opr_rtk_common_pkt_dump_cputag:715]  TT_IDX : 0
1970 Jan 01 09:06:53 KST (none) kernel: [XSW][RXPKT][INFO][opr_rtk_common_pkt_dump_cputag:716]  OF_TBL_ID : 0
1970 Jan 01 09:06:53 KST (none) kernel: [XSW][RXPKT][INFO][opr_rtk_common_pkt_dump_cputag:717]  IS_TRK : 0
1970 Jan 01 09:06:53 KST (none) kernel: [XSW][RXPKT][INFO][opr_rtk_common_pkt_dump_cputag:718]  TRK_ID : 0
1970 Jan 01 09:06:53 KST (none) kernel: [XSW][RXPKT][INFO][opr_rtk_common_pkt_dump_cputag:719]  OTAGIF : 0
1970 Jan 01 09:06:53 KST (none) kernel: [XSW][RXPKT][INFO][opr_rtk_common_pkt_dump_cputag:720]  ITAGIF : 0
1970 Jan 01 09:06:53 KST (none) kernel: [XSW][RXPKT][INFO][opr_rtk_common_pkt_dump_cputag:721]  OVID : 0
1970 Jan 01 09:06:53 KST (none) kernel: [XSW][RXPKT][INFO][opr_rtk_common_pkt_dump_cputag:722]  IVID : 0
1970 Jan 01 09:06:53 KST (none) kernel: [XSW][RXPKT][INFO][opr_rtk_common_pkt_dump_cputag:723]  FWD_VID_SEL : 0
1970 Jan 01 09:06:53 KST (none) kernel: [XSW][RXPKT][INFO][opr_rtk_common_pkt_dump_cputag:724]  FVID : 1
1970 Jan 01 09:06:53 KST (none) kernel: [XSW][RXPKT][INFO][opr_rtk_common_pkt_dump_cputag:725]  QID : 3
1970 Jan 01 09:06:53 KST (none) kernel: [XSW][RXPKT][INFO][opr_rtk_common_pkt_dump_cputag:726]  ATK_TYPE : 0
1970 Jan 01 09:06:53 KST (none) kernel: [XSW][RXPKT][INFO][opr_rtk_common_pkt_dump_cputag:727]  MAC_CST : 0
1970 Jan 01 09:06:53 KST (none) kernel: [XSW][RXPKT][INFO][opr_rtk_common_pkt_dump_cputag:728]  DM_RXIDX : 63
1970 Jan 01 09:06:53 KST (none) kernel: [XSW][RXPKT][INFO][opr_rtk_common_pkt_dump_cputag:729]  NEW_SA : 0
1970 Jan 01 09:06:53 KST (none) kernel: [XSW][RXPKT][INFO][opr_rtk_common_pkt_dump_cputag:730]  PMV_FORBID : 0
1970 Jan 01 09:06:53 KST (none) kernel: [XSW][RXPKT][INFO][opr_rtk_common_pkt_dump_cputag:731]  L2_STTC_PMV : 0
1970 Jan 01 09:06:53 KST (none) kernel: [XSW][RXPKT][INFO][opr_rtk_common_pkt_dump_cputag:732]  L2_DYN_PMV : 0
1970 Jan 01 09:06:53 KST (none) kernel: [XSW][RXPKT][INFO][opr_rtk_common_pkt_dump_cputag:733]  L2_ERR_PKT : 0
1970 Jan 01 09:06:53 KST (none) kernel: [XSW][RXPKT][INFO][opr_rtk_common_pkt_dump_cputag:734]  L3_ERR_PKT : 0
1970 Jan 01 09:06:53 KST (none) kernel: [XSW][RXPKT][INFO][opr_rtk_common_pkt_dump_cputag:735]  OVERSIZE : 0
1970 Jan 01 09:06:53 KST (none) kernel: [XSW][RXPKT][INFO][opr_rtk_common_pkt_dump_cputag:740]  ext_unit : 0
1970 Jan 01 09:06:53 KST (none) kernel: [XSW][RXPKT][INFO][opr_rtk_common_pkt_dump_cputag:741]  ext_source_port : 1
1970 Jan 01 09:06:53 KST (none) kernel: [XSW][RXPKT][INFO][opr_rtk_common_pkt_dump_cputag:743]  HASH_FULL : 0
1970 Jan 01 09:06:53 KST (none) kernel: [XSW][RXPKT][INFO][opr_rtk_common_pkt_dump_cputag:744]  INVALID_SA : 0
1970 Jan 01 09:06:53 KST (none) kernel: [XSW][RXPKT][INFO][opr_rtk_common_pkt_dump_cputag:745]  REASON : 0
1970 Jan 01 09:06:53 KST (none) kernel: [RX PKT][24:f5:aa:ce:4f:b0->ff:ff:ff:ff:ff:ff][UNTAGGED][ARP][(1:2048:1540:1)(24:f5:aa:ce:4f:b0->00:00:00:00:00:00)(10.10.10.20/32->10.10.10.1/32)]
1970 Jan 01 09:06:53 KST (none) kernel: [XSW][RXPKT][INFO][oprt_rtk_l2pkt_rx_cb:210] LAYER2 RX_REASON(0x00000000)
1970 Jan 01 09:06:53 KST (none) kernel: [XSW][RXPKT][INFO][oprt_rtk_l2pkt_rx_cb:215] reasons: NIC_RX_REASON_ACL_HIT(91)
1970 Jan 01 09:06:53 KST (none) kernel: [0x02] l2_pkt_rx_cb() : PORT: 8000001, LEN: 64, VID: 1, REASON: 0x5000, COS: 3
1970 Jan 01 09:06:53 KST (none) kernel: [0x02] l2_pkt_rx_cb() : MAC_DST ff:ff:ff:ff:ff:ff MAC_SRC 24:f5:aa:ce:4f:b0
1970 Jan 01 09:06:53 KST (none) kernel: [XSW][RXPKT][DBG0][l2_if_arp_rx:981] CHECK RX DEV 0x08000001 80d2c800 80d2c800
1970 Jan 01 09:06:53 KST (none) kernel: [XSW][RXPKT][DBG0][l2_if_arp_rx:982] CHECK RX DEV ge1
1970 Jan 01 09:06:53 KST (none) kernel: [l2_if_arp_rx] RX: port 134217729, pktlen 64
```
### classifier 설정O
- arp 관련 로그 나오지 않아서 와이어샤크에서 arp 패킷 전달되는 것으로 대체 확인

# BCM 테스트
## 설정
- ge2: 삼성노트북 (10.10.10.1)
- ge1: 테스트노트북(10.10.10.20)
	- ge1에 vlan classifier (ipv4 10.10.10.0/24  vlan10 추가)
- vlan10 인터페이스 ip (10.10.10.10)
- 여기서 classifier 룰 빼면 통신 안됨
- untag 패킷이기 때문에 PVID 1로 처리됨 (룰 없는 경우)
```
TiFRONT% show vlan
 -------------------------------------------------------------------------------
  PORT            | ge
                  |                   1 1 1 1 1 1 1 1 1 1 2 2 2 2 2 2 2 2 2
                  | 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8
 -----------------+-----------------------------------------------------------
  SWITCH MODE     | T A H H H H H H H H H H H H H H H H H H H H H H H H H H
 -----------------+-----------------------------------------------------------
  default  (   1) | T . U U U U U U U U U U U U U U U U U U U U U U U U U U
  VLAN0010 (  10) | t U . . . . . . . . . . . . . . . . . . . . . . . . . .
 -------------------------------------------------------------------------------
```
- running conf
```
TiFRONT% sh running-config
!
hostname TiFRONT
!
!
username admin privilege 15 password 8 $6$227a45183462aa2a7ce0affd2e7c40b153391e72ea1b296a
faa264d0ada699bfce3692ce557fc319b8b2a3b70007583e992b7b3b326689f35d2b4b9a2f35bb17
!
remote-reset enable
ip forwarding
ecmp ip-src
!
timezone kst
!
spanning-tree mode rpvst+
!
vlan 10  name VLAN0010
vlan classifier rule 1 ipv4 10.10.10.0/24 vlan 10
vlan classifier group 1 add rule 1
!
spanning-tree vlan 10
!
interface ge1
 no shutdown
 switchport
 switchport mode trunk
 switchport trunk allowed vlan add 1
 switchport trunk allowed vlan add 10
 flowcontrol send off
 flowcontrol receive off
 vlan classifier activate 1
 jumbo-frame off
 set lldp enable txrx
 lldp tlv chassis-id port-id ttl system-capabilities
 spanning-tree vlan 10
!
```
## 로그
- 10.10.10.20->10.10.10.1 ping test
```
2025 Dec 30 11:29:41 KST iProc kernel: [0x02] br_handle_frame() : Frame recvd on ifindex ge1
2025 Dec 30 11:29:41 KST iProc kernel: [0x02] br_handle_frame() : applied ingress rules and the frames vid(10) instance (2)
2025 Dec 30 11:29:41 KST iProc kernel: [0x02] br_handle_frame() : learn 1 and forward 1 for instance 2
2025 Dec 30 11:29:41 KST iProc kernel: [0x02] br_handle_frame() : XXX c8 a3 62 15 a0 6 : 245 : 3
2025 Dec 30 11:29:41 KST iProc kernel: [0x02] br_handle_frame() : GARP Broadcast PDU received  3
2025 Dec 30 11:29:41 KST iProc kernel: [XSW][RXPKT][INFO][switch_l2pkt_rx_reasons_check:487] =========> COMPARE Pkt reasons: FilterMatch
2025 Dec 30 11:29:41 KST iProc kernel: [XSW][RXPKT][INFO][switch_l3pkt_rx_cb:203] RX_LPORT(0x08000003), SRC LPORT(0x08000003) ge1
2025 Dec 30 11:29:41 KST iProc kernel: [XSW][RXPKT][INFO][switch_l3pkt_rx_cb:205] LAYER3 RX_REASON(0x00001000) CLASSIFICATION(0x00000000)
2025 Dec 30 11:29:41 KST iProc kernel: [XSW][RXPKT][INFO][switch_l3pkt_rx_cb:208] reasons: FilterMatch
2025 Dec 30 11:29:41 KST iProc kernel: [XSW][RXPKT][DBG0][xsw_l3_rx:1877] l2if(ge1) l3if(vlan10) PTYPE(0x00000806)
2025 Dec 30 11:29:41 KST iProc kernel: [XSW][RXPKT][INFO][xsw_l3_rx:1979] DROP PKT ERR(-5)
2025 Dec 30 11:29:45 KST iProc kernel: [0x02] switch_l2pkt_rx_cb() :   Pkt reasons: FilterMatch
2025 Dec 30 11:29:45 KST iProc kernel: [0x02] switch_l2pkt_rx_cb() :   Pkt reasons: Protocol
2025 Dec 30 11:29:45 KST iProc kernel: [XSW][RXPKT][WARN][switch_l2pkt_rx_cb:209] RX RSN FILTER 0x5000 0x5000
2025 Dec 30 11:29:45 KST iProc kernel: [XSW][RXPKT][INFO][switch_l2pkt_rx_cb:213] RX_LPORT(0x08000003), SRC LPORT(0x08000003)
2025 Dec 30 11:29:45 KST iProc kernel: [BCMPKT][VLAN(V:10 P:0: C:0)][RX_P(U:0 P:3 C:0)][SRC_P(M:0 P:3 T:-1)][DST_P(M:0 P:2)][RSN:0x5000 (13)FilterMatch (58)Protocol
2025 Dec 30 11:29:45 KST iProc kernel: [RX PKT][24:f5:aa:ce:4f:b0->c8:a3:62:15:a0:06][T(8021Q)(P:0 C:0 V:10)][ARP][(256:8:1030:256)(24:f5:aa:ce:4f:b0->c8:a3:62:15:a0:06)(10.10.10.20/32->10.10.10.1/32)]
2025 Dec 30 11:29:45 KST iProc kernel: [XSW][RXPKT][INFO][switch_l2pkt_rx_cb:226] LAYER2 RX_REASON(0x00005000)
2025 Dec 30 11:29:45 KST iProc kernel: [XSW][RXPKT][INFO][switch_l2pkt_rx_cb:229] reasons: FilterMatch
2025 Dec 30 11:29:45 KST iProc kernel: [XSW][RXPKT][INFO][switch_l2pkt_rx_cb:229] reasons: Protocol
2025 Dec 30 11:29:45 KST iProc kernel: [0x02] l2_pkt_rx_cb() : PORT: 8000003, LEN: 68, VID: 10, REASON: 0x5000, COS: 3
2025 Dec 30 11:29:45 KST iProc kernel: [0x02] l2_pkt_rx_cb() : MAC_DST c8:a3:62:15:a0:06 MAC_SRC 24:f5:aa:ce:4f:b0
2025 Dec 30 11:29:45 KST iProc kernel: [XSW][RXPKT][DBG0][l2_if_arp_rx:979] CHECK RX DEV 0x08000003 d8523000 d8523000
2025 Dec 30 11:29:45 KST iProc kernel: [XSW][RXPKT][DBG0][l2_if_arp_rx:980] CHECK RX DEV ge1
2025 Dec 30 11:29:45 KST iProc kernel: [l2_if_arp_rx] RX: port 134217731, pktlen 68
2025 Dec 30 11:29:45 KST iProc kernel:
2025 Dec 30 11:29:45 KST iProc kernel:    0 : c8 a3 62 15 a0 06 24 f5 aa ce 4f b0 81 00 00 0a
2025 Dec 30 11:29:45 KST iProc kernel:   10 : 08 06 00 01 08 00 06 04 00 01 24 f5 aa ce 4f b0
2025 Dec 30 11:29:45 KST iProc kernel:   20 : 0a 0a 0a 14 c8 a3 62 15 a0 06 0a 0a 0a 01 00 00
2025 Dec 30 11:29:45 KST iProc kernel:   30 : 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
2025 Dec 30 11:29:45 KST iProc kernel:   40 : 5f f9 49 11
2025 Dec 30 11:29:45 KST iProc kernel: [0x02] br_handle_frame() : Frame recvd on ifindex ge1
2025 Dec 30 11:29:45 KST iProc kernel: [0x02] br_handle_frame() : applied ingress rules and the frames vid(10) instance (2)
2025 Dec 30 11:29:45 KST iProc kernel: [0x02] br_handle_frame() : learn 1 and forward 1 for instance 2
2025 Dec 30 11:29:45 KST iProc kernel: [0x02] br_handle_frame() : XXX c8 a3 62 15 a0 6 : 245 : 3
2025 Dec 30 11:29:45 KST iProc kernel: [0x02] br_handle_frame() : GARP Broadcast PDU received  3
2025 Dec 30 11:29:45 KST iProc kernel: [XSW][RXPKT][INFO][switch_l2pkt_rx_reasons_check:487] =========> COMPARE Pkt reasons: FilterMatch
2025 Dec 30 11:29:45 KST iProc kernel: [XSW][RXPKT][INFO][switch_l3pkt_rx_cb:203] RX_LPORT(0x08000003), SRC LPORT(0x08000003) ge1
2025 Dec 30 11:29:45 KST iProc kernel: [XSW][RXPKT][INFO][switch_l3pkt_rx_cb:205] LAYER3 RX_REASON(0x00005000) CLASSIFICATION(0x00000000)
2025 Dec 30 11:29:45 KST iProc kernel: [XSW][RXPKT][INFO][switch_l3pkt_rx_cb:208] reasons: FilterMatch
2025 Dec 30 11:29:45 KST iProc kernel: [XSW][RXPKT][INFO][switch_l3pkt_rx_cb:208] reasons: Protocol
2025 Dec 30 11:29:45 KST iProc kernel: [XSW][RXPKT][DBG0][xsw_l3_rx:1877] l2if(ge1) l3if(vlan10) PTYPE(0x00000806)
2025 Dec 30 11:29:45 KST iProc kernel: [XSW][RXPKT][INFO][xsw_l3_rx:1979] DROP PKT ERR(-5)
2025 Dec 30 11:29:45 KST iProc kernel: [0x02] switch_l2pkt_rx_cb() :   Pkt reasons: FilterMatch
2025 Dec 30 11:29:45 KST iProc kernel: [XSW][RXPKT][WARN][switch_l2pkt_rx_cb:209] RX RSN FILTER 0x1000 0x1000
2025 Dec 30 11:29:45 KST iProc kernel: [XSW][RXPKT][INFO][switch_l2pkt_rx_cb:213] RX_LPORT(0x08000002), SRC LPORT(0x08000002)
2025 Dec 30 11:29:45 KST iProc kernel: [BCMPKT][VLAN(V:10 P:0: C:0)][RX_P(U:0 P:2 C:0)][SRC_P(M:0 P:2 T:-1)][DST_P(M:0 P:3)][RSN:0x1000 (13)FilterMatch
2025 Dec 30 11:29:45 KST iProc kernel: [RX PKT][c8:a3:62:15:a0:06->24:f5:aa:ce:4f:b0][T(8021Q)(P:0 C:0 V:10)][ARP][(256:8:1030:512)(c8:a3:62:15:a0:06->24:f5:aa:ce:4f:b0)(10.10.10.1/32->10.10.10.20/32)]
2025 Dec 30 11:29:45 KST iProc kernel: [XSW][RXPKT][INFO][switch_l2pkt_rx_cb:226] LAYER2 RX_REASON(0x00001000)
2025 Dec 30 11:29:45 KST iProc kernel: [XSW][RXPKT][INFO][switch_l2pkt_rx_cb:229] reasons: FilterMatch
```