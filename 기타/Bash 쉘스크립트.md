- 스크립트
```
#!/bin/bash
JAR_PATH="/home/piotac/ticontroller-license-0.0.6.jar"
TODAY=$(date +%Y%m%d)
key1=$(java -jar "$JAR_PATH" -e 00002 4 01 "$TODAY" 50 0 | grep -i "license Key:" | awk '{print $4}')
key2=$(java -jar "$JAR_PATH" -e 00002 5 01 "$TODAY" 0 1 | grep -i "License Key:" | awk '{print $4}')
echo "서버라이센스:$key1"
echo "유지보수라이센스:$key2"
```
- 권한 부여
	-  chmod 755 test.sh
	- 7: owner는 rwx 모두 가능
	  55: 그룹, other는 읽기 쓰기만 가능
