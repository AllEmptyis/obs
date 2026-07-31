# 리눅스 -> 윈도우
- scp root@192.168.212.10:/home/test/file.txt "C:\Users\user\Downloads\"
# 윈도우 -> 리눅스
- scp  -P 2222 root@192.168.212.10:/var/log/messages C:\Users\user\Desktop\
# 리눅스
- scp `파일명` root@ip:`옮길 경로`
