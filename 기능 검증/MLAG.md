# MLAG
- Multi Chassis Link Aggregation
	- 정확한 표준안은 없음
- LACP가 두 개의 스위치에서 동작하도록 함 (LACP 대상이 1:2)
- 기존 lacp는 상대편 장비가 죽으면 곧바로 죽게 되지만, MLAG는 상대편 장비가 2개이기 때문에 하나가 죽어도 통신 유지
## 용어
- MLAGL
	- 각 MLAG 장비에서 상대편 장비와 연결하기 위한 2개의 링크
- ISL
	- Inter Switch Link / Mlag 스위치 간 정보 주고받는 링크
- MLAGPDU
	- MLAG Protocol Data Unit
- SF: Signal Fail
	- 연결 끊김
- NR: No Request
	- 정상
## 동작 설명
- system id를 입력 받아서 isl로 통신
- loop 발생 방지를 위해 isl 간 통신에는 egress mask 적용
	- egress mask란?
```
하드웨어 레벨에서 설정되는 마스크
isl간 mlagpdu 등 상태정보 패킷 제외 트래픽은 통신되지 않도록 차단
```
- MLAGL이 단절된 경우 반대편 MLAGL로 패킷 전송이 되어야 하므로 egress mask 해제
- ISL 단절 시에는 egress mask 유지
	- 기존 mac table로 동작
- ISL과 MLAGL간 mac learning은 MLAGL이 우선시하도록 설정됨
## MLAG H/C
- ISL을 통해 heartbeat를 1-2초마다 전송
- 5초 이상 수신 되지 않는 경우 ISL 끊김으로 판단
- ISL 링크 다운인 경우 대기 없이 바로 ISL SF로 인식
## MLAG 상태 정보
- 초기: enable
## 미지원 장비
```
F26/P/M, G24/P,SM 
```
