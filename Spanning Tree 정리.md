Loop 도는 원인:
LAN 구간에서 목적지로 가는 길이 2개 이상 있으면 (A->B->C or A->C)
A에서 발생한 브로드캐스트 패킷이 C->B->다시 A 이런 식으로 루프가 발생하게 됨.

STP

루트 스위치:
루트 스위치 기준으로 stp 형성

Designated 스위치:
루트 스위치에게 패킷을 전달하는 스위치
root port: desig스위치에서 루트 스위치로 패킷을 전달할 때 사용하는 포트
desig port: 하위(다른) desig 스위치와 연결되는 포트


RSTP
alterante port: 다른 장비로부터 높은 우선순위의 bpdu를 받아서 블락 된 포트
backup port: 같은 장비의 다른 포트로부터 높은 우선순위의 bpdu를 받아서 블락
-> 두 포트는 discarding(차단) 상태

