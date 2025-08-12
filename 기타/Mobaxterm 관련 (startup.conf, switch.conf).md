- 유선 문의 (모바엑스텀 관련)
	- switch.conf: 모바엑스텀으로 접근 시 초기화 되는 설정 파일. 부팅 시 해당 파일로 startup.conf 덮어 씌움
	- startup.conf: running config와 같은 파일. wr 하면 startup.conf 내용이 switch.conf로 저장
```
부팅할 때 참고하는 파일: switch.conf
wr하면 startup.conf파일이 switch.conf 파일에 저장 됨

모바텀으로 접속했을 때 switch.conf파일이 초기화 됨
wr 안한 경우-> 초기화 된 switch.conf 파일 내용을 startup.conf가 가져오면서 초기화
모바텀 접속 후 wr한 경우 -> switch.conf 파일이 초기화 되었으나 다시 기존 파일로 원상복구 되어 재부팅 후에도 문제 없음

재부팅 전
-rw-r-----    1 nobody   root         10314 Aug 11 16:13 startup.conf
-rw-r-----    1 nobody   root         10314 Aug 11 16:13 switch.conf

재부팅 후 (모바텀 붙은 상태에서)
-rw-r-----    1 nobody   root         10285 Aug 11 16:16 startup.conf
-rw-r-----    1 nobody   root         10314 Aug 11 16:13 switch.conf

```
- 개선 버전
	- 3.2.17.2.0 (RTK)
	- 3.1.17.1.0 (BCM)