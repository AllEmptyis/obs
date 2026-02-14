 - 기존에 있던 설치파일 제거 (ticontroller)
	 - /cporject, 설치 rpm은 그냥 둬도 됨 (rpm 변경 사항 없는 경우)
 - 설치파일 다운로드
 ```
  wget -O ticontroller-v1.2.1.5.15.tar.gz https://piolink.dooray.com/share/drive-files/by7tvsjx6qqi.uh7hgLb1QESuUKei4JfvKQ

 wget -O ticontroller-v1.2.1.5.15-local-rpm.tar.gz https://piolink.dooray.com/share/drive-files/by7tvsjx6qqi.uJCMn0QLSgOO2TZO-Ijq6g
 ```
- 압축 해제
```
tar -xvzf ticontroller-v1.2.1.5.15.tar.gz
```
- /ticontroller/operator/run.py에서 3번 컨트롤러 업그레이드 선택
```
[root@localhost operator]# python run.py
[root] [ Select Features ]
[root]   1. TiController Setting
[root]   2. Additional Setting
[root]   3. Check Status
[root]   4. EXIT
Enter the number of your choice: 1
[root] Input : 1
[root] [ Select Ticontroller features ]
[root]   1. TiController Install
[root]   2. TiController Uninstall
[root]   3. TiController Upgrade
[root]   4. EXIT
Enter the number of your choice: 3
```