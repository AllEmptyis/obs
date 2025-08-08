## 문제 
- 컨트롤러 ip 변경 후 ACL 설정에서 변경 된 컨트롤러 ip가 반영되지 않는 현상
	- 허용주소에 컨트롤러 ip가 기본 허용되어야 하는데 다른 ip로 적용
## 원인
- `node.cc.server.ip` 항목에서 ip가 변경되지 않았음
	- ACL 등 일부 기능과 연결 된 항목일 것으로 예상
- `/cproject/conf/application.properties`
```
# node settings
node.cc.enable=true
node.cc.name=localhost.localdomain
node.cc.server.vip=ticontroller
node.cc.server.ip=192.168.212.244
node.cc.server.port=9001
node.cc.server.ssl=true
acl.allow.ip=
```
## 해결
- ip 변경 후 컨트롤러 데몬 재시작
	- `systemctl restart ticontroller.service`
