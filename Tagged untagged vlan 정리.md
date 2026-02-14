Ingress
1. untag 프레임이 들어온 경우: pvid 삽입 (pvid: 물리 포트별 vid)
	- T: pvid tagged
	- t: 일반적인 tagged
2. tagged 프레임이 들어온 경우: 프레임의 vid가 속한 vlan으로 전달

Egress
1. access mode(untagged port): 프레임에 태그를 붙이지 않음
2. trunk mode: 프레임에 태그를 붙임
3. hybrid mode: 송신할 프레임을 tagged로 송신할지 untag로 송신할지 선택 가능

--------------
관련 일감: show vlan 에서 출력 되는 설명문의 상세 정보 확인 문의
 https://redmine.piolink.com/issues/88130 