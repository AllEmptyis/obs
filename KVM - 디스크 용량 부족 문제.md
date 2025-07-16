## 문제 상황
- kvm 설치 중 디스크 용량 부족 에러
```
[root@localhost /]# virt-install --name en_po_test --memory 8192 --vcpu 2 --cdrom /Rocky-10.0-x86_64-dvd1.iso --network network=bridge --os-variant generic --graphics vnc --disk path=/qcow2/en_po_test.qcow2,size=50
WARNING  볼륨이 완전히 할당되면 요청된 볼륨 용량이 사용 가능한 풀 공간을 초과합니다. (51200 요청된 용량 > 40497 사용 가능)
ERROR    볼륨이 완전히 할당되면 요청된 볼륨 용량이 사용 가능한 풀 공간을 초과합니다. (51200 요청된 용량 > 40497 사용 가능) (--check disk_size=off 또는 --check all=off를 사용하여 대체)
```
## 원인
- qcow 디렉토리로 지정한 곳의 가용 공간이 부족
	- root 현재 40G 남음
```
[root@localhost /]# df -h
Filesystem           Size  Used Avail Use% Mounted on
devtmpfs             4.0M     0  4.0M   0% /dev
tmpfs                3.8G     0  3.8G   0% /dev/shm
tmpfs                1.5G   39M  1.5G   3% /run
/dev/mapper/rl-root   70G   31G   40G  44% /
/dev/mapper/rl-home  387G  2.8G  384G   1% /home
/dev/sda1            960M  299M  662M  32% /boot
tmpfs                765M  124K  765M   1% /run/user/1000
```
## 해결
- 가용 공간이 많이 남아있는 곳을 가상 디스크 경로로 지정한다
	- ex) home
```
virt-install --name enpotest --memory 8012 --vcpu 2 --cdrom /Rocky-9.6-x86_64-dvd.iso --network bridge=br0 --os-variant rocky9 --disk /home/enpotest,size=50
```