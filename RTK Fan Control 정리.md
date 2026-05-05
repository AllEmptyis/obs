# 기본 설명
- RTK 스위치는 팬 컨트롤이 가능/불가한 모델이 있음
	- 불가한 모델은 show fan 하면 not supported on this model 라고 출력 된다
- cloud / cli 설정 값은 기본적으로 동일
	- cloud
		- manual(수동) 설정 불가
		- 컨트롤러에서 팬 제어 설정하게 되면 auto-low, 그 전에는 auto-normal
		- 컨트롤러에서 직접 설정하는 값은 실제로는 적용 안 됨 (내장된 auto-low 값으로 적용)
- cloud mode에서는 show fan 명령어 동작X
## rpm/pwm 차이
- pwm
	- 듀티 사이클, 즉 fan 속도를 제어하는 값
- RPM
	- 실제 분당 회전수
	- 같은 pwm 이라도 rpm 값은 달라질 수 있음
## 개발 일감
- https://redmine.piolink.com/issues/126072
# CS2254GX
- /tmp/.pwm_threshold_table
	- 기준 온도: temp2
```
cloud, cloud-cli mode 동일

bash-4.4# cat .pwm_threshold_table

 == FAN PWM mode (manual/auto-low/auto-normal) ==
  - auto-normal

 == FAN PWM valid ranges ==   //RPM 범위표
  - fan1
  state : 0 RPM
  level 0 : 0 ~ 13500 RPM *
  level 1 : 3000 ~ 13500 RPM
  - fan2
  state : 0 RPM
  level 0 : 0 ~ 13500 RPM *
  level 1 : 3000 ~ 13500 RPM
  - fan3
  state : 0 RPM
  level 0 : 0 ~ 13500 RPM *
  level 1 : 3000 ~ 13500 RPM

 == FAN PWM threshold table ==

 - status
 PWM : 0% (0/255) (level 0)
 Temp : 27'C (level 0)
 Watt : 0W (level 0)

 - auto-normal *
 -- going up table --
  watt \ temp | ~ 40      'C | 40'C ~  50'C |       50'C ~ |
         0W ~ |           0% |          39% |          50% |
 -- going down table --
  watt \ temp | ~ 30      'C | 30'C ~  40'C |       40'C ~ |
         0W ~ |          *0% |          39% |          50% |

 - auto-low
 -- going up table --
  watt \ temp | ~ 40      'C | 40'C ~  50'C |       50'C ~ |
         0W ~ |           0% |           0% |          50% |
 -- going down table --
  watt \ temp | ~ 30      'C | 30'C ~  40'C |       40'C ~ |
         0W ~ |           0% |           0% |          50% |
```
- /tmp/hwmon_sensors
```
bash-4.4# cat /tmp/hwmon_sensors
temp1: 25 C(83, 3, 0) => OK
temp2: 25 C(86, 5, 0) => OK
temp3: 35 C(89, 9, 0) => OK
fan1: 0 RPM(13500, 0, 0) => OK  //level 0, rpm range 0-13500
fan2: 0 RPM(13500, 0, 0) => OK
fan3: 0 RPM(13500, 0, 0) => OK
```
- fan control 설정 (CLI)
```
TiFRONT(config)# fan-control ?
  auto-low     Set PWM to automatic low mode
  auto-normal  Set PWM to automatic normal mode
  manual       Set PWM to manual mode
```
- 현재 상태
```
TiFRONT# sh hardware-status
---------------------------------------------
Hardware status
---------------------------------------------
Temperature 1: 28 'C
Temperature 2: 28 'C
Temperature 3: 41 'C
---------------------------------------------
Fan 1 Status: 0 RPM
Fan 2 Status: 0 RPM
Fan 3 Status: 0 RPM
---------------------------------------------
TiFRONT#
TiFRONT# show fan
 Fan PWM : auto normal mode
```
## CS2254GXP fan rpm level 상세 내역
- https://redmine.piolink.com/issues/83215#note-2
```
- CS2254GXP FAN RPM valid range  
    level 0: 2800 ~ 6000  
    level 1: 6000 ~ 9200  
    level 2: 7500 ~ 10500  
    level 3: 8500 ~ 11500

부하가 없는 경우: 50도 이상부터 level 1
```

# FAN PWM range 확인 방법
- cat /tmp/hwmon_sensors