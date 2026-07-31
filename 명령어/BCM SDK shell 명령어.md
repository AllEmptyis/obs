## tty 깨진 경우
- bcm.user.proxy에서 빠져나올 때 Ctrl C로 나오면서 터미널이 깨진 경우
- 방법
```
Ctrl J or Ctrl M
stty sane
Ctrl J
```