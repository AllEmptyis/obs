```
#!/bin/bash
JAR_PATH="/home/piotac/ticontroller-license-0.0.6.jar"
TODAY=$(date +%Y%m%d)
key1=$(java -jar "$JAR_PATH" -e 00002 4 01 "$TODAY" 50 0 | grep "License Key:" | awk '{print $4}')
key2=$(java -jar "$JAR_PATH" -e 00002 5 01 "$TODAY" 0 1 | grep "License Key:" | awk '{print $4}')
echo "서버 라이센스: $key1"
echo "유지보수 라이센스: $key2" 
```