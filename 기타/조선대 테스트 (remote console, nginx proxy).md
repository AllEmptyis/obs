- 리버스 프록시 443 아닌 다른 걸로 하면 접속 안됨
- 8888/443으로 했을 때 처음부터 443으로 접속하면 안나오고 8888로 한 번 접속하면 java 포트 열림
	- 그 후 443으로도 접속 가능
```
[root@localhost ~]# ss -lntup
Netid        State         Recv-Q        Send-Q               Local Address:Port               Peer Address:Port       Process
udp          UNCONN        0             0                        127.0.0.1:323                     0.0.0.0:*           users:(("chronyd",pid=752,fd=5))
udp          UNCONN        0             0                            [::1]:323                        [::]:*           users:(("chronyd",pid=752,fd=6))
tcp          LISTEN        0             511                        0.0.0.0:443                     0.0.0.0:*           users:(("nginx",pid=5673,fd=5),("nginx",pid=5672,fd=5),("nginx",pid=5671,fd=5))
tcp          LISTEN        0             128                        0.0.0.0:22                      0.0.0.0:*           users:(("sshd",pid=778,fd=3))
tcp          LISTEN        0             80                               *:3306                          *:*           users:(("mysqld",pid=821,fd=22))
tcp          LISTEN        0             50                               *:9300                          *:*           users:(("java",pid=1137,fd=72))
tcp          LISTEN        0             32                               *:21                            *:*           users:(("vsftpd",pid=1145,fd=3))
tcp          LISTEN        0             128                           [::]:22                         [::]:*           users:(("sshd",pid=778,fd=4))
tcp          LISTEN        0             50                               *:9200                          *:*           users:(("java",pid=1137,fd=89))
[root@localhost ~]#
[root@localhost ~]#
[root@localhost ~]# ss -lntup
Netid        State         Recv-Q        Send-Q               Local Address:Port               Peer Address:Port       Process
udp          UNCONN        0             0                        127.0.0.1:323                     0.0.0.0:*           users:(("chronyd",pid=752,fd=5))
udp          UNCONN        0             0                            [::1]:323                        [::]:*           users:(("chronyd",pid=752,fd=6))
tcp          LISTEN        0             511                        0.0.0.0:443                     0.0.0.0:*           users:(("nginx",pid=5673,fd=5),("nginx",pid=5672,fd=5),("nginx",pid=5671,fd=5))
tcp          LISTEN        0             128                        0.0.0.0:22                      0.0.0.0:*           users:(("sshd",pid=778,fd=3))
tcp          LISTEN        0             80                               *:3306                          *:*           users:(("mysqld",pid=821,fd=22))
tcp          LISTEN        0             50                               *:9300                          *:*           users:(("java",pid=1137,fd=72))
tcp          LISTEN        0             32                               *:21                            *:*           users:(("vsftpd",pid=1145,fd=3))
tcp          LISTEN        0             128                           [::]:22                         [::]:*           users:(("sshd",pid=778,fd=4))
tcp          LISTEN        0             4096                             *:9000                          *:*           users:(("java",pid=5698,fd=136))
tcp          LISTEN        0             4096                             *:9001                          *:*           users:(("java",pid=5698,fd=135))
tcp          LISTEN        0             50                               *:9200                          *:*           users:(("java",pid=1137,fd=89))
tcp          LISTEN        0             100                              *:8888                          *:*           users:(("java",pid=5698,fd=21)) ---> 생김
```
- 로그 (/var/log/nginx/error.log)
	- nginx가 업스트림 서버(로컬호스트:8888,자바)로 연결하려 했는데 8888이 포트가 안열려 있어서 거절 되었을 때 로그
```
2025/09/17 18:14:22 [error] 6399#6399: *3 connect() failed (111: Connection refused) while connecting to upstream, client: 192.168.212.174, server: localhost, request: "GET / HTTP/1.1", upstream: "https://[::1]:8888/", host: "192.168.212.225"
2025/09/17 18:14:22 [warn] 6399#6399: *3 upstream server temporarily disabled while connecting to upstream, client: 192.168.212.174, server: localhost, request: "GET / HTTP/1.1", upstream: "https://[::1]:8888/", host: "192.168.212.225"
2025/09/17 18:14:22 [error] 6399#6399: *3 connect() failed (111: Connection refused) while connecting to upstream, client: 192.168.212.174, server: localhost, request: "GET / HTTP/1.1", upstream: "https://127.0.0.1:8888/", host: "192.168.212.225"
2025/09/17 18:14:22 [warn] 6399#6399: *3 upstream server temporarily disabled while connecting to upstream, client: 192.168.212.174, server: localhost, request: "GET / HTTP/1.1", upstream: "https://127.0.0.1:8888/", host: "192.168.212.225"
2025/09/17 18:14:22 [error] 6399#6399: *3 no live upstreams while connecting to upstream, client: 192.168.212.174, server: localhost, request: "GET /favicon.ico HTTP/1.1", upstream: "https://localhost/favicon.ico", host: "192.168.212.225", referrer: "https://192.168.212.225/"
```
- **현재 상태에서 웹콘솔 안붙음**
- nginx proxy 서버로 붙었을 때 nginx error 나는 이유 > 아마 자바 포트가 up이 안 되어서 그런 거 같다
- tftp 명령어
	- `tftp -p -l ./test.pcap -r test.pcap 192.168.212.174`
- gui로 접속한 포트 = 웹콘솔로 붙는 포트
- 케이스 질문에 대한 답으로는, nginx 파일은 리버스 프록시가 활성화 되어야 변경되는데 기존에 application 설정 파일 백업이 누락되어 백업 되지 않은 것으로 보임
  아마 설정파일 백업 제대로 됐으면 nginx 파일도 동일하게 백업 됐을 것으로 추측
- 특이사항 - 443으로 접속했을 때만 펌웨어 실패 표시 출력
## 패킷 덤프
- 덤프상으로는 오히려 클라이언트-서버 구간에서 tcp retransmission 발생 (웹콘솔 접속 시)
- 스위치-서버 간 9001번 통신도 리트랜스미션이 한 번 발생해서 어느 쪽이 문제인지 정확히 모르겠다
## 스위치 /var/log/messages_all 확인
- 컨트롤러-스위치 간 통신은 nginx proxy 포트 번호로 변경되는 거 같다
```
2025 Sep 17 17:40:22 KST (none) sysmgrd: [pio_sysinfo_set_tech_assist_info_message] protocol http ip ticontroller port 1443 path /api/upload/v1/files secure 1  
2025 Sep 17 17:43:06 KST (none) sysmgrd: Received C-Switch Regist status(Active) from ticontroller.  
2025 Sep 17 17:43:07 KST (none) sysmgrd: [pio_sysinfo_set_tech_assist_info_message] protocol http ip ticontroller port 443 path /api/upload/v1/files secure 1  
```
## 펌웨어 업데이트 오류 메시지 (스위치)
- 아래 에러는 스위치의 /etc/hosts에서 컨트롤러 정보를 추가해주면 해결 된다
```
2025-09-18 12:05:24 [cbroker] [send:401 bytes] {
    "msg_id": -1,
    "contents": {
        "cswitch_firmware_download_start": {
            "service": "update",
            "function": "firmware",
            "time": 1758164724,
            "filename": "/api/v1/files/PLOS-CS-V3.1.16.3.0/PLOS-CSm-V3.1.16.3.0",
            "evt_string": "Firmware download start (file=\"/api/v1/files/PLOS-CS-V3.1.16.3.0/PLOS-CSm-V3.1.16.3.0\")"
        }
    }
}
2025-09-18 12:05:24 [libsoup] starting output source
2025-09-18 12:05:24 [libsoup] queued 1 frame of len 315
2025-09-18 12:05:24 [libsoup] sent frame
2025-09-18 12:05:24 [libsoup] stopping output source
2025-09-18 12:05:24 [cbroker] [send:493 bytes] {
    "msg_id": -1,
    "contents": {
        "cswitch_firmware_download_result": {
            "service": "update",
            "function": "firmware",
            "time": 1758164724,
            "result": 0,
            "filename": "/api/v1/files/PLOS-CS-V3.1.16.3.0/PLOS-CSm-V3.1.16.3.0",
            "evt_string": "Firmware download result (result=\"fail\", file=\"/api/v1/files/PLOS-CS-V3.1.16.3.0/PLOS-CSm-V3.1.16.3.0\", error=\"Couldn't resolve host 'ticontroller'\")"
        }
    }
}
```
- success 메시지
```
2025-09-18 12:12:33 [cbroker] [send:462 bytes] {
    "msg_id": -1,
    "contents": {
        "cswitch_firmware_download_result": {
            "service": "update",
            "function": "firmware",
            "time": 1758165153,
            "result": 1,
            "filename": "/api/v1/files/PLOS-CS-V3.1.16.3.0/PLOS-CSm-V3.1.16.3.0",
            "evt_string": "Firmware download result (result=\"success\", file=\"/api/v1/files/PLOS-CS-V3.1.16.3.0/PLOS-CSm-V3.1.16.3.0\", error=\"OK\")"
        }
    }
}

bash-4.3# cat /etc/hosts
127.0.0.1                TiFRONT
192.168.212.225         ticontroller
```
## 컨트롤러 신규 설치 후 상태 확인
- 리버스 프록시 디폴트 false 상태이고, nginx 파일도 기본 파일로 되어 있음
	- applicaiton properties 설정 파일 변경으로는 nginx 파일 변경 안됨
		- 왜냐하면 nginx 데몬이 stop 되어 있음
```
Enter the number of your choice: 3
[root] Input : 3
[root] LOCAL CMD || [ -d /cproject ] &>/dev/null && echo "True" || echo "False"
[root] LOCAL STDOUT || True
[root] [ TiController Status ]
[root]  - Install Type: stand_alone
[root] -----------------------------------------------
[root]  - ticontroller (pid 8327) is running
[root]  - Web Node Info
[root]   + localhost.localdomain state: MASTER
[root]   + heap size: 1g
[root] -----------------------------------------------
[root]  - mysqld (pid 8471) is running
[root] -----------------------------------------------
[root]  - elasticsearch (pid 8385) is running
[root]  - Elasticsearch node status
[root]   + localhost.localdomain heap size: 1.0g
[root] LOCAL CMD || curl -s '127.0.0.1:9200/_cat/nodes?v'
[root] LOCAL STDOUT || host                  ip        heap.percent ram.percent load node.role master name
[root] LOCAL STDOUT || localhost.localdomain 127.0.0.1           11          38 1.47 d         *      localhost.localdomain
[root]  - Elasticsearch indices status
[root] LOCAL CMD || curl -s '127.0.0.1:9200/_cat/indices?v'
[root] LOCAL STDOUT || health status index             pri rep docs.count docs.deleted store.size pri.store.size
[root] LOCAL STDOUT || green  open   host_usages         1   0          0            0       115b           115b
[root] LOCAL STDOUT || green  open   usages              1   0          3            0     10.6kb         10.6kb
[root] LOCAL STDOUT || green  open   client_usages       1   0          0            0       115b           115b
[root] LOCAL STDOUT || green  open   connectivities      1   0          0            0       115b           115b
[root] LOCAL STDOUT || green  open   statistics          1   0          0            0       115b           115b
[root] LOCAL STDOUT || green  open   host_usage_detail   1   0          0            0       115b           115b
[root] LOCAL STDOUT || green  open   event_logs          1   0          0            0       115b           115b
[root] LOCAL STDOUT || green  open   error_logs          1   0          0            0       115b           115b
[root] -----------------------------------------------
[root]  - nginx is stopped    ---->데몬 stop 확인 (초기상태)
[root] -----------------------------------------------


443 포트 안올라옴
[root@localhost operator]# ss -lntup
Netid             State              Recv-Q             Send-Q                         Local Address:Port                          Peer Address:Port            Process
udp               UNCONN             0                  0                                  127.0.0.1:323                                0.0.0.0:*                users:(("chronyd",pid=5990,fd=5))
udp               UNCONN             0                  0                                          *:54328                                    *:*                users:(("java",pid=8385,fd=74))
udp               UNCONN             0                  0                                      [::1]:323                                   [::]:*                users:(("chronyd",pid=5990,fd=6))
tcp               LISTEN             0                  128                                  0.0.0.0:22                                 0.0.0.0:*                users:(("sshd",pid=786,fd=3))
tcp               LISTEN             0                  80                                         *:3306                                     *:*                users:(("mysqld",pid=8471,fd=47))
tcp               LISTEN             0                  50                                         *:9300                                     *:*                users:(("java",pid=8385,fd=72))
tcp               LISTEN             0                  50                                         *:9200                                     *:*                users:(("java",pid=8385,fd=130))
tcp               LISTEN             0                  4096                                       *:9001                                     *:*                users:(("java",pid=8327,fd=161))
tcp               LISTEN             0                  4096                                       *:9000                                     *:*                users:(("java",pid=8327,fd=160))
tcp               LISTEN             0                  100                                        *:8443                                     *:*                users:(("java",pid=8327,fd=59))
tcp               LISTEN             0                  32                                         *:21                                       *:*                users:(("vsftpd",pid=5966,fd=3))
tcp               LISTEN             0                  128                                     [::]:22                                    [::]:*                users:(("sshd",pid=786,fd=4))
```
## 스위치 cbroker.log
- 컨트롤러 포트 8443 / 컨트롤러 연동 ip 바꿨다가 다시 붙였을 때
- gui는 현재 8443으로 붙어 있음 / nginx 러닝 중
```
        "cswitch_tech_assist_info_message": {

            "protocol": "http",

            "ticontroller_ip": "ticontroller",

            "ticontroller_port": 8443,   ----> 설정파일에서 리버스 프록시 false로 하면 서버 포트로 나옴

            "ticontroller_user_id": "api",

            "ticontroller_user_password": "api123",

            "tech_assist_upload_path": "/api/upload/v1/files",

            "ticontroller_secure": 1

        },
        
--------리버스 프록시 포트 1443 변경 후 gui 1443으로 붙었을 때 -------

        "cswitch_tech_assist_info_message": {

            "protocol": "http",

            "ticontroller_ip": "ticontroller",

            "ticontroller_port": 1443,

            "ticontroller_user_id": "api",

            "ticontroller_user_password": "api123",

            "tech_assist_upload_path": "/api/upload/v1/files",

            "ticontroller_secure": 1

        }
        
        
------- 리버스 프록시 포트 443으로 변경 / gui 8443으로 접속 -------
  "cswitch_tech_assist_info_message": {

            "protocol": "http",

            "ticontroller_ip": "ticontroller",

            "ticontroller_port": 443,

            "ticontroller_user_id": "api",

            "ticontroller_user_password": "api123",

            "tech_assist_upload_path": "/api/upload/v1/files",

            "ticontroller_secure": 1

        }
        
        
------ ngix stop 후 8443 포트로 접속했을 때 ----------

        "cswitch_tech_assist_info_message": {

            "protocol": "http",

            "ticontroller_ip": "ticontroller",

            "ticontroller_port": 443,

            "ticontroller_user_id": "api",

            "ticontroller_user_password": "api123",

            "tech_assist_upload_path": "/api/upload/v1/files",

            "ticontroller_secure": 1

        },
```
- 물어볼 것: 당시 443으로 접속했을 때 어떤 페이지 떴는지?
	- nginx stop 한 경우 443 으로 들어갔을 때 페이지 찾을 수 없다고 뜬다 (연결 거부)
	- 러닝 중인데 포트가 안올라왔다거나 그런 경우에는 nginx error 페이지 표출
- 내 생각: nginx proxy true로 된 경우 프록시 서버로 요청, 그런데 당시 사이트에서는 nginx 설정 반영이 안 되어 있었어서 펌웨어 업데이트 응답을 못한 것이 아닐까 싶음
### 재현1
- nginx stop 후 443으로 요청 / 펌웨어 다운로드 실패 확인
```
2025-09-18 18:28:16 [cbroker] [send:429 bytes] {

    "msg_id": -1,

    "contents": {

        "cswitch_firmware_download_start": {

            "service": "update",

            "function": "firmware",

            "time": 1758187696,

            "filename": "/api/v1/files/PLOS-CS-V3.1.17.2.0-rc1001/PLOS-CSm-V3.1.17.2.0-rc1001",

            "evt_string": "Firmware download start (file=\"/api/v1/files/PLOS-CS-V3.1.17.2.0-rc1001/PLOS-CSm-V3.1.17.2.0-rc1001\")"

        }

    }

}

2025-09-18 18:28:16 [libsoup] starting output source

2025-09-18 18:28:16 [libsoup] queued 1 frame of len 343

2025-09-18 18:28:16 [cbroker] firmware_download

2025-09-18 18:28:16 [cbroker]   - [Apply]  elapsed  0.02 seconds

2025-09-18 18:28:16 [libsoup] sent frame

2025-09-18 18:28:16 [libsoup] stopping output source

2025-09-18 18:28:16 [cbroker] [send:160 bytes] {

    "msg_id": 58,

    "last_msg_id": 43,

    "contents": {

        "firmware_download": {

            "code": 0,

            "message": "OK"

        }

    }

}

2025-09-18 18:28:16 [libsoup] starting output source

2025-09-18 18:28:16 [libsoup] queued 1 frame of len 104

2025-09-18 18:28:16 [libsoup] sent frame

2025-09-18 18:28:16 [libsoup] stopping output source

2025-09-18 18:28:16 [cbroker] [send:291 bytes] {

    "msg_id": -1,

    "contents": {

        "cswitch_lldp_info_message": [

            {

                "remote_mac": "00:06:c4:71:8a:2c",

                "local_ports": "ge13",

                "remote_ports": "ge8",

                "system_type": "switch"

            }

        ]

    }

}

2025-09-18 18:28:16 [libsoup] starting output source

2025-09-18 18:28:16 [libsoup] queued 1 frame of len 175

2025-09-18 18:28:16 [libsoup] sent frame

2025-09-18 18:28:16 [libsoup] stopping output source

2025-09-18 18:28:16 [cbroker] [send:547 bytes] {

    "msg_id": -1,

    "contents": {

        "cswitch_firmware_download_result": {

            "service": "update",

            "function": "firmware",

            "time": 1758187696,

            "result": 0,

            "filename": "/api/v1/files/PLOS-CS-V3.1.17.2.0-rc1001/PLOS-CSm-V3.1.17.2.0-rc1001",

            "evt_string": "Firmware download result (result=\"fail\", file=\"/api/v1/files/PLOS-CS-V3.1.17.2.0-rc1001/PLOS-CSm-V3.1.17.2.0-rc1001\", error=\"Failed to connect to ticontroller port 443: Connection refused\")"       ------> refused 메시지 / 케이스 에러 로그와 내용은 다름

        }

    }
    
    connection refused: 서버에서 reset 보냈을 때?
```
## 신규 티콘 서버에서 다운로드 시도
- 현재 티콘 서버 정보: nginx 데몬 꺼져 있고 설정파일에서는 리버스 프록시 1443 사용 (true)
	- 1443으로 요청이 온다.
```
2025-09-18 18:45:38 [cbroker] [Recv:364 bytes] {

    "msg_id": 12,

    "contents": {

        "firmware_download": {

            "filename": "/api/v1/files/PLOS-CS-V3.1.17.2.0-rc1001/PLOS-CSm-V3.1.17.2.0-rc1001",

            "protocol": "http",

            "ip": "192.168.212.213",

            "port": 1443,

            "user_id": "api",

            "user_pw": "api123",

            "secure": 1

        }

    }

}

2025-09-18 18:45:38 [cbroker] [send:429 bytes] {

    "msg_id": -1,

    "contents": {

        "cswitch_firmware_download_start": {

            "service": "update",

            "function": "firmware",

            "time": 1758188738,

            "filename": "/api/v1/files/PLOS-CS-V3.1.17.2.0-rc1001/PLOS-CSm-V3.1.17.2.0-rc1001",

            "evt_string": "Firmware download start (file=\"/api/v1/files/PLOS-CS-V3.1.17.2.0-rc1001/PLOS-CSm-V3.1.17.2.0-rc1001\")"

        }

    }

}

2025-09-18 18:45:38 [libsoup] starting output source

2025-09-18 18:45:38 [libsoup] queued 1 frame of len 343

2025-09-18 18:45:38 [libsoup] sent frame

2025-09-18 18:45:38 [libsoup] stopping output source

2025-09-18 18:45:38 [cbroker] firmware_download

2025-09-18 18:45:38 [cbroker]   - [Apply]  elapsed  0.02 seconds

2025-09-18 18:45:38 [cbroker] [send:159 bytes] {

    "msg_id": 12,

    "last_msg_id": 7,

    "contents": {

        "firmware_download": {

            "code": 0,

            "message": "OK"

        }

    }

}

2025-09-18 18:45:38 [libsoup] starting output source

2025-09-18 18:45:38 [libsoup] queued 1 frame of len 103

2025-09-18 18:45:38 [libsoup] sent frame

2025-09-18 18:45:38 [libsoup] stopping output source

2025-09-18 18:45:38 [cbroker] [send:549 bytes] {

    "msg_id": -1,

    "contents": {

        "cswitch_firmware_download_result": {

            "service": "update",

            "function": "firmware",

            "time": 1758188738,

            "result": 0,

            "filename": "/api/v1/files/PLOS-CS-V3.1.17.2.0-rc1001/PLOS-CSm-V3.1.17.2.0-rc1001",

            "evt_string": "Firmware download result (result=\"fail\", file=\"/api/v1/files/PLOS-CS-V3.1.17.2.0-rc1001/PLOS-CSm-V3.1.17.2.0-rc1001\", error=\"Failed to connect to 192.168.212.213 port 1443: No route to host\")"

        }

    }
    
    -----> no route to host: nginx를 통해서 프록시가 안 된다는 의미 아닐지?
```
## 로그
- 티콘 데몬 재시작 로그
```
2025-09-05 14:39:32.568  INFO 224979 [main] com.piolink.cc.CControllerApplication   :48 - Starting CControllerApplication on TiCon with PID 224979 (started by root in /root)

2025-09-05 14:39:32.570  INFO 224979 [main] com.piolink.cc.CControllerApplication   :660 - No active profile set, falling back to default profiles: default

2025-09-05 14:39:32.683  INFO 224979 [main] ationConfigEmbeddedWebApplicationContext:578 - Refreshing org.springframework.boot.context.embedded.AnnotationConfigEmbeddedWebApplicationContext@6659c656: startup date [Fri Sep 05 14:39:32 KST 2025]; root of context hierarchy
```
## 웹콘솔 복구
- 어플리케이션 설정파일에서 리버스 프록시 false로 바꾼 뒤 데몬 재시작 후 콘솔 연동 됨
## 포트 확인
- 웹콘솔: 9001
```
11:27:13.454257 IP 192.168.212.213.9001 > 192.168.212.239.35426: Flags [P.], seq 1495:1662, ack 841, win 505, options [nop,nop,TS val 1319088369 ecr 167437], length 167
11:27:13.476749 IP 192.168.212.239.35426 > 192.168.212.213.9001: Flags [P.], seq 841:948, ack 1662, win 359, options [nop,nop,TS val 167443 ecr 1319088369], length 107
11:27:13.477842 IP 192.168.212.239.35426 > 192.168.212.213.9001: Flags [P.], seq 948:1010, ack 1662, win 359, options [nop,nop,TS val 167443 ecr 1319088369], length 62
11:27:13.477967 IP 192.168.212.213.9001 > 192.168.212.239.35426: Flags [.], ack 1010, win 504, options [nop,nop,TS val 1319088393 ecr 167443], length 0
11:27:13.479378 IP 192.168.212.213.9001 > 192.168.212.239.35426: Flags [P.], seq 1662:1735, ack 1010, win 504, options [nop,nop,TS val 1319088395 ecr 167443], length 73
11:27:13.520205 IP 192.168.212.239.35426 > 192.168.212.213.9001: Flags [P.], seq 1010:1085, ack 1735, win 359, options [nop,nop,TS val 167454 ecr 1319088395], length 75
11:27:13.561416 IP 192.168.212.213.9001 > 192.168.212.239.35426: Flags [.], ack 1085, win 504, options [nop,nop,TS val 1319088477 ecr 167454], length 0
11:27:17.445323 IP 192.168.212.239.43436 > 192.168.212.213.9001: Flags [.], seq 132:1580, ack 251, win 2003, options [nop,nop,TS val 168435 ecr 1319088233], length 1448
11:27:17.445412 IP 192.168.212.213.9001 > 192.168.212.239.43436: Flags [.], ack 1580, win 1181, options [nop,nop,TS val 1319092361 ecr 168435], length 0
11:27:17.445803 IP 192.168.212.239.43436 > 192.168.212.213.9001: Flags [.], seq 1580:3028, ack 251, win 2003, options [nop,nop,TS val 168435 ecr 1319088233], length 1448
```
- 기술지원도우미 티콘에서 장비요청
	- 9001로 통신하다가 8443도 사용
```
[root@localhost ~]# tcpdump -nn host 192.168.212.239
dropped privs to tcpdump
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on ens160, link-type EN10MB (Ethernet), snapshot length 262144 bytes
12:15:16.203985 IP 192.168.212.213.9001 > 192.168.212.239.43436: Flags [P.], seq 2384974204:2384974330, ack 1863656338, win 1181, options [nop,nop,TS val 1321971119 ecr 886013], length 126
12:15:16.212427 IP 192.168.212.239.43436 > 192.168.212.213.9001: Flags [.], ack 126, win 2003, options [nop,nop,TS val 888124 ecr 1321971119], length 0
12:15:16.233184 IP 192.168.212.239.43436 > 192.168.212.213.9001: Flags [P.], seq 1:143, ack 126, win 2003, options [nop,nop,TS val 888130 ecr 1321971119], length 142
12:15:16.233222 IP 192.168.212.213.9001 > 192.168.212.239.43436: Flags [.], ack 143, win 1181, options [nop,nop,TS val 1321971148 ecr 888130], length 0


12:18:26.848178 IP 192.168.212.239.60119 > 192.168.212.213.8443: Flags [.], seq 23484243:23485691, ack 16256, win 1245, options [nop,nop,TS val 935781 ecr 1322161298], length 1448
12:18:26.848597 IP 192.168.212.239.60119 > 192.168.212.213.8443: Flags [.], seq 23485691:23487139, ack 16256, win 1245, options [nop,nop,TS val 935781 ecr 1322161298], length 1448
12:18:26.848610 IP 192.168.212.213.8443 > 192.168.212.239.60119: Flags [.], ack 23487139, win 14376, options [nop,nop,TS val 1322161764 ecr 935781], length 0
12:18:26.849159 IP 192.168.212.239.60119 > 192.168.212.213.8443: Flags [.], seq 23487139:23488587, ack 16256, win 1245, options [nop,nop,TS val 935781 ecr 1322161298], length 1448
12:18:26.849524 IP 192.168.212.239.60119 > 192.168.212.213.8443: Flags [.], seq 23488587:23490035, ack 16256, win 1245, options [nop,nop,TS val 935781 ecr 1322161298], length 1448
12:18:26.849536 IP 192.168.212.213.8443 > 192.168.212.239.60119: Flags [.], ack 23490035, win 14376, options [nop,nop,TS val 1322161765 ecr 935781], length 0
12:18:26.850229 IP 192.168.212.239.60119 > 192.168.212.213.8443: Flags [.], seq 23490035:23491483, ack 16256, win 1245, options [nop,nop,TS val 935781 ecr 1322161298], length 1448
12:18:26.850401 IP 192.168.212.239.60119 > 192.168.212.213.8443: Flags [.], seq 23491483:23492931, ack 16256, win 1245, options [nop,nop,TS val 935781 ecr 1322161298], length 1448
12:18:26.850449 IP 192.168.212.213.8443 > 192.168.212.239.60119: Flags [.], ack 23492931, win 14376, options [nop,nop,TS val 1322161766 ecr 935781], length 0
12:18:26.850961 IP 192.168.212.239.60119 > 192.168.212.213.8443: Flags [.], seq 23492931:23494379, ack 16256, win 1245, options [nop,nop,TS val 935781 ecr 1322161298], length 1448
12:18:26.851356 IP 192.168.212.239.60119 > 192.168.212.213.8443: Flags [P.], seq 23494379:23494857, ack 16256, win 1245, options [nop,nop,TS val 935781 ecr 1322161298], length 478
12:18:26.851384 IP 192.168.212.213.8443 > 192.168.212.239.60119: Flags [.], ack 23494857, win 14376, options [nop,nop,TS val 1322161767 ecr 935781], length 0
12:18:26.858371 IP 192.168.212.239.60119 > 192.168.212.213.8443: Flags [.], seq 23494857:23496305, ack 16256, win 1245, options [nop,nop,TS val 935786 ecr 1322161328], length 1448
12:18:26.858809 IP 192.168.212.239.60119 > 192.168.212.213.8443: Flags [.], seq 23496305:23497753, ack 16256, win 1245, options [nop,nop,TS val 935786 ecr 1322161328], length 1448
```
# nginx error.log 확인 (/var/log/nginx)
- 아래 로그가 error occured 발생 시 나오는 로그 
	- nginx 포트 1443으로 접속했으나 백엔드(자바)가 아직 올라오지 않아서 프록시가 안 되는 경우
	- 8888은 서버 포트
```
[root@localhost nginx]# tail -f error.log
2025/09/24 13:27:03 [error] 19687#19687: *4 connect() failed (111: Connection refused) while connecting to upstream, client: 192.168.212.217, server: localhost, request: "GET / HTTP/1.1", upstream: "https://[::1]:8888/", host: "192.168.212.225:1443"
2025/09/24 13:27:03 [warn] 19687#19687: *4 upstream server temporarily disabled while connecting to upstream, client: 192.168.212.217, server: localhost, request: "GET / HTTP/1.1", upstream: "https://[::1]:8888/", host: "192.168.212.225:1443"
2025/09/24 13:27:03 [error] 19687#19687: *4 connect() failed (111: Connection refused) while connecting to upstream, client: 192.168.212.217, server: localhost, request: "GET / HTTP/1.1", upstream: "https://127.0.0.1:8888/", host: "192.168.212.225:1443"
2025/09/24 13:27:03 [warn] 19687#19687: *4 upstream server temporarily disabled while connecting to upstream, client: 192.168.212.217, server: localhost, request: "GET / HTTP/1.1", upstream: "https://127.0.0.1:8888/", host: "192.168.212.225:1443"
2025/09/24 13:27:03 [error] 19687#19687: *4 no live upstreams while connecting to upstream, client: 192.168.212.217, server: localhost, request: "GET /favicon.ico HTTP/1.1", upstream: "https://localhost/favicon.ico", host: "192.168.212.225:1443", referrer: "https://192.168.212.225:1443/"
```
- 400 bad request error
	- nginx, server port 동일하게 설정한 경우 발생하는 것으로 확인
```
2025/09/24 13:39:39 [error] 19985#19985: *4 connect() failed (111: Connection refused) while connecting to upstream, client: 192.168.212.217, server: localhost, request: "GET / HTTP/1.1", upstream: "https://[::1]:443/", host: "192.168.212.225"
2025/09/24 13:39:39 [warn] 19985#19985: *4 upstream server temporarily disabled while connecting to upstream, client: 192.168.212.217, server: localhost, request: "GET / HTTP/1.1", upstream: "https://[::1]:443/", host: "192.168.212.225"
2025/09/24 13:39:39 [error] 19986#19986: *25 connect() failed (111: Connection refused) while connecting to upstream, client: 127.0.0.1, server: localhost, request: "GET / HTTP/1.0", upstream: "https://[::1]:443/", host: "192.168.212.225:443"
2025/09/24 13:39:39 [warn] 19986#19986: *25 upstream server temporarily disabled while connecting to upstream, client: 127.0.0.1, server: localhost, request: "GET / HTTP/1.0", upstream: "https://[::1]:443/", host: "192.168.212.225:443"
```
- 해당 경우에는 서버포트(자바)가 활성화 되지 않는 거 같다
```
[root@localhost nginx]# ss -lntup
Netid       State        Recv-Q       Send-Q               Local Address:Port               Peer Address:Port       Process
udp         UNCONN       0            0                        127.0.0.1:323                     0.0.0.0:*           users:(("chronyd",pid=753,fd=5))
udp         UNCONN       0            0                            [::1]:323                        [::]:*           users:(("chronyd",pid=753,fd=6))
tcp         LISTEN       0            511                        0.0.0.0:443                     0.0.0.0:*           users:(("nginx",pid=19986,fd=5),("nginx",pid=19985,fd=5),("nginx",pid=19984,fd=5))
tcp         LISTEN       0            128                        0.0.0.0:22                      0.0.0.0:*           users:(("sshd",pid=780,fd=3))
tcp         LISTEN       0            50                               *:9300                          *:*           users:(("java",pid=1140,fd=72))
tcp         LISTEN       0            50                               *:9200                          *:*           users:(("java",pid=1140,fd=94))
tcp         LISTEN       0            128                           [::]:22                         [::]:*           users:(("sshd",pid=780,fd=4))
tcp         LISTEN       0            32                               *:21                            *:*           users:(("vsftpd",pid=1145,fd=3))
tcp         LISTEN       0            80                               *:3306                          *:*           users:(("mysqld",pid=824,fd=18))
```

# server port 443으로 변경 시 포트 안올라오는 원인
- nignx 1443 / server port 443으로 변경 시 443 java port가 안올라옴
- 컨트롤러 java log 확인 방법
	- `tail -f /var/log/elasticsearch/ticontroller.log`
- 경로
	- `/etc/systemd/system/ticontroller.service.d/`
## 원인
- 리눅스에서는 1024 이하의 포트는 root 권한이 있어야만 프로세스가 바인딩 가능
- 즉 443 뿐만 아니라 1024 포트는 권한 없으면 다 안 됨
## 443 포트로만 접속할 수 있도록 하는 방법1
- 오버라이드 파일 만들기 (override.conf 파일을 먼저 읽음)
	- edit로 바로 수정하면 저장이 안 돼서 파일을 직접 생성
```
mkdir -p /etc/systemd/system/ticontroller.service.d
vi override.conf

[Service]
AmbientCapabilities=CAP_NET_BIND_SERVICE   -> 비루트 사용자라도 1024 이하 포트에 바인딩 허용
CapabilityBoundingSet=CAP_NET_BIND_SERVICE  -> 바인드만 허용하고 나머지 다른 권한은 갖지 못하도록 설정
NoNewPrivileges=false    ----> 해당 프로세스 및 하위 프로세스들이 root 추가적인 권한 상승을 할 수 있도록 설정 

sytstemctl daemon-reload
systemctl restart ticontroller.service
```
- 설정 확인
```
[root@localhost ticontroller.service.d]# systemctl cat ticontroller.service
# /usr/lib/systemd/system/ticontroller.service
[Unit]
Description=TiController
Documentation=http://www.piolink.com
Wants=network-online.target
After=network-online.target

[Service]

User=ticontroller
Group=controller
EnvironmentFile=/cproject/conf/restore.properties
ExecStart=/bin/bash /cproject/scripts/ticontroller.sh $MODE

# Connects standard output to /dev/null
StandardOutput=null

# Connects standard error to journal
StandardError=journal

# When a JVM receives a SIGTERM signal it exits with code 143
SuccessExitStatus=143

# Specifies the maximum file descriptor number that can be opened by this process
LimitNOFILE=65535


# Shutdown delay in seconds, before process is tried to be killed with KILL (if configured)
TimeoutStopSec=20

[Install]
WantedBy=multi-user.target

# /etc/systemd/system/ticontroller.service.d/override.conf
[Service]
AmbientCapabilities=CAP_NET_BIND_SERVICE
CapabilityBoundingSet=CAP_NET_BIND_SERVICE
NoNewPrivileges=false
```
- 추가
	- systemctl edit 기본 편집기 vi로 설정 방법
```
echo 'export SYSTEMD_EDITOR=vi' >> ~/.bashrc
source ~/.bashrc
```
- 포트 확인
```
[root@localhost ticontroller.service.d]# ss -lntup
Netid       State        Recv-Q       Send-Q             Local Address:Port                Peer Address:Port       Process
udp         UNCONN       0            0                      127.0.0.1:323                      0.0.0.0:*           users:(("chronyd",pid=755,fd=5))
udp         UNCONN       0            0                          [::1]:323                         [::]:*           users:(("chronyd",pid=755,fd=6))
udp         UNCONN       0            0                              *:54328                          *:*           users:(("java",pid=1129,fd=74))
tcp         LISTEN       0            128                      0.0.0.0:22                       0.0.0.0:*           users:(("sshd",pid=786,fd=3))
tcp         LISTEN       0            511                      0.0.0.0:1443                     0.0.0.0:*           users:(("nginx",pid=4216,fd=5),("nginx",pid=4215,fd=5),("nginx",pid=4214,fd=5))
tcp         LISTEN       0            4096                           *:9000                           *:*           users:(("java",pid=4464,fd=144))
tcp         LISTEN       0            4096                           *:9001                           *:*           users:(("java",pid=4464,fd=145))
tcp         LISTEN       0            50                             *:9200                           *:*           users:(("java",pid=1129,fd=120))
tcp         LISTEN       0            100                            *:443                            *:*           users:(("java",pid=4464,fd=18))
tcp         LISTEN       0            32                             *:21                             *:*           users:(("vsftpd",pid=1141,fd=3))
tcp         LISTEN       0            128                         [::]:22                          [::]:*           users:(("sshd",pid=786,fd=4))
tcp         LISTEN       0            50                             *:9300                           *:*           users:(("java",pid=1129,fd=72))
tcp         LISTEN       0            80                             *:3306                           *:*           users:(("mysqld",pid=831,fd=19))
```
## 443 포트만으로 접속할 수 있는 방법2
- reverse proxy 포트를 443으로 두고 백엔드 포트를 로컬호스트 리슨으로 돌리기
- /cproject/conf/application.properties 파일 수정
```
# tomcat
server.port=8443
server.address=127.0.0.1 <---- 해당 부분 추가
server.session.timeout=1800
server.tomcat.accesslog.enabled=false
server.tomcat.protocol-header=x-forwarded-proto
server.tomcat.remote-ip-header=x-forwarded-for
server.tomcat.max-threads=300
server.tomcat.uri-encoding=UTF-8
```
- 데몬 재시작 후 포트 리스닝 확인
```
[root@localhost ~]# ss -lntup
Netid      State        Recv-Q       Send-Q                  Local Address:Port              Peer Address:Port      Process
udp        UNCONN       0            0                           127.0.0.1:323                    0.0.0.0:*          users:(("chronyd",pid=758,fd=5))
udp        UNCONN       0            0                               [::1]:323                       [::]:*          users:(("chronyd",pid=758,fd=6))
udp        UNCONN       0            0                                   *:54328                        *:*          users:(("java",pid=1142,fd=74))
tcp        LISTEN       0            128                           0.0.0.0:22                     0.0.0.0:*          users:(("sshd",pid=781,fd=3))
tcp        LISTEN       0            511                           0.0.0.0:443                    0.0.0.0:*          users:(("nginx",pid=2055,fd=5),("nginx",pid=2054,fd=5),("nginx",pid=2053,fd=5))
tcp        LISTEN       0            80                                  *:3306                         *:*          users:(("mysqld",pid=825,fd=23))
tcp        LISTEN       0            128                              [::]:22                        [::]:*          users:(("sshd",pid=781,fd=4))
tcp        LISTEN       0            32                                  *:21                           *:*          users:(("vsftpd",pid=1149,fd=3))
tcp        LISTEN       0            50                                  *:9200                         *:*          users:(("java",pid=1142,fd=118))
tcp        LISTEN       0            50                                  *:9300                         *:*          users:(("java",pid=1142,fd=72))
tcp        LISTEN       0            100                [::ffff:127.0.0.1]:8443                         *:*          users:(("java",pid=3408,fd=58))

----> 로컬 호스트만 리스닝 하도록 됨
```
- 443 포트로만 접속 가능 확인
# server/reverse proxy 동일하게 443으로 설정
- 포트 변경
```
[root@localhost scripts]# ./config_reverse_proxy.sh
Install reverse proxy
httpd_can_network_connect, httpd_setrlimit enabling...
./config_reverse_proxy.sh: line 41: chkconfig: command not found
Configure reverse proxy start

Server port (default: 8443) : 443
Reverse proxy port (default: 443) : 443
http_port_t                    tcp      8888, 1443, 222, 2222, 1111, 5555, 80, 81, 443, 488, 8008, 8009, 8443, 9000
Port 443 already exists in http_port_t
Redirecting to /bin/systemctl restart nginx.service
```
- 데몬 재시작 후 프로세스 확인
	- 백엔드 안올라오고 443으로 접속하면 nginx 400 bad request 발생
```
[root@localhost conf.d]# ss -lntup
Netid       State        Recv-Q       Send-Q             Local Address:Port                Peer Address:Port       Process
udp         UNCONN       0            0                      127.0.0.1:323                      0.0.0.0:*           users:(("chronyd",pid=729,fd=5))
udp         UNCONN       0            0                              *:54328                          *:*           users:(("java",pid=1133,fd=74))
udp         UNCONN       0            0                          [::1]:323                         [::]:*           users:(("chronyd",pid=729,fd=6))
tcp         LISTEN       0            511                      0.0.0.0:443                      0.0.0.0:*           users:(("nginx",pid=8582,fd=5),("nginx",pid=8581,fd=5),("nginx",pid=8580,fd=5))
tcp         LISTEN       0            128                      0.0.0.0:22                       0.0.0.0:*           users:(("sshd",pid=763,fd=3))
tcp         LISTEN       0            80                             *:3306                           *:*           users:(("mysqld",pid=820,fd=18))
tcp         LISTEN       0            50                             *:9300                           *:*           users:(("java",pid=1133,fd=72))
tcp         LISTEN       0            50                             *:9200                           *:*           users:(("java",pid=1133,fd=116))
tcp         LISTEN       0            32                             *:21                             *:*           users:(("vsftpd",pid=1143,fd=3))
tcp         LISTEN       0            128                         [::]:22                          [::]:*           users:(("sshd",pid=763,fd=4))
```
- nginx 설정파일 (/etc/nignx/conf.d/default.conf)
```
server {
    listen       443 ssl;
    server_name  localhost;

    #charset koi8-r;
    #access_log  /var/log/nginx/host.access.log  main;

    ssl_certificate    /etc/nginx/ticontroller.crt;
    ssl_certificate_key    /etc/nginx/ticontroller.key;
    ssl_protocols TLSv1.2 TLSv1.3;

    location / {
        proxy_pass https://localhost:443;  ---> gui에서 처리하는 부분

        proxy_set_header Host $host:$server_port;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Port $server_port;
```
- defualt.conf 파일에서 프록시 하는 포트를 8443으로 변경
	- 연결할 백엔드가 활성화 되지 않아서 연결 불가
```
[root@localhost conf.d]# cat default.conf
server {
    listen       443 ssl;
    server_name  localhost;

    #charset koi8-r;
    #access_log  /var/log/nginx/host.access.log  main;

    ssl_certificate    /etc/nginx/ticontroller.crt;
    ssl_certificate_key    /etc/nginx/ticontroller.key;
    ssl_protocols TLSv1.2 TLSv1.3;

    location / {
        proxy_pass https://localhost:8443;
```
- 1022 이하 포트로 설정 시 -> 백엔드 프로세스 안올라옴
- 프록시, 서버 포트 동일하게 설정 시 -> 백엔드 프로세스 안올라옴
```
/var/log/nginx/error.log

2025/10/15 14:22:40 [warn] 9792#9792: *7450 upstream server temporarily disabled while connecting to upstream, client: 127.0.0.1, server: localhost, request: "POST /tiscreen/getAllEventSummaryDataByNetworkAll.json HTTP/1.0", upstream: "https://[::1]:8443/tiscreen/getAllEventSummaryDataByNetworkAll.json", host: "192.168.212.225:8443", referrer: "https://192.168.212.225:8443/configure/cswitch_settings.do"
2025/10/15 14:22:40 [error] 9791#9791: *7453 connect() failed (111: Connection refused) while connecting to upstream, client: 127.0.0.1, server: localhost, request: "POST /tiscreen/getAllEventSummaryDataByNetworkAll.json HTTP/1.0", upstream: "https://[::1]:8443/tiscreen/getAllEventSummaryDataByNetworkAll.json", host: "192.168.212.225:8443", referrer: "https://192.168.212.225:8443/configure/cswitch_settings.do"
2025/10/15 14:22:40 [warn] 9791#9791: *7453 upstream server temporarily disabled while connecting to upstream, client: 127.0.0.1, server: localhost, request: "POST /tiscreen/getAllEventSummaryDataByNetworkAll.json HTTP/1.0", upstream: "https://[::1]:8443/tiscreen/getAllEventSummaryDataByNetworkAll.json", host: "192.168.212.225:8443", referrer: "https://192.168.212.225:8443/configure/cswitch_settings.do"
^C
```
# 설정파일을 통해서만 포트 변경
- 서버 포트 변경 가능, 리버스 프록시 포트는 적용 안됨
	- 리버스 프록시 설정은 스크립트를 거쳐야 하는 거 같음 (nginx 설정 파일 변경도 안됨)
- well known 포트로 서버 포트 변경 불가
# CAP_NET_BIND_SERVICE 설명
- 컨트롤러 데몬에 CAP_NET_BIND_SERVICE 권한을 직접 부여해주는 방식
- 경로
	- `/usr/lib/systemd/system/ticontroller.service`
		- rpm(패키지)에서 제공하는 기본 유닛 파일
	- `/etc/systemd/system/ticontroller.service.d/override.conf`
		- 관리자가 데몬 유닛을 수정할 수 있도록 하는 용도
## systemd
- 데몬
	- 백그라운드에서 계속 동작하는 프로그램
- systemd
	- 서비스 전체의 서비스/프로세스를 관리하는 데몬
		- pid 1으로 실행된다
		- systemctl  start, stop 같은 명령을 제어할 수 있도록 함
	- 여러 unit으로 나뉘어져 있음
		- 각 서비스는 unit 파일로 정의되어 있다
```
unit 예시

service: 프로그램/데몬 실행   ---> 컨트롤러도 systemd가 관리하는 service 유닛
socket: 소켓(포트) 활성화 관리
target: 여러 유닛을 묶는 그룹
mount: 파일 시스템 마운트 관리
timer: 스케줄러, 예약 실행
device: 커널 디바이스 트리거
```
- 즉 systemd 유닛인 ticontroller 데몬 서비스에 직접 root 권한 추가 설정을 하는 것
## 권한 설명
- 리눅스에는 root 권한을 세분화한 capability 라는 개념이 있음 (약 30여가지)
	- cap_net_bind_service: 1024 미만 포트 열 수 있도록 하는 권한
	- cap_sys_admin: 시스템 관리
	- cap_chown: 파일 소유자 변경 등등
- CAP_NET_BIND_SERVICE
	- 1024 미만의 포트를 열 수 있도록 하는 세분화된 권한
- AmbientCapabilities
	- 서비스 프로세스에 특정 linux capability를 환경처럼 상속 시켜주는 기능
- CapabilityBoundingSet
	- systemd 서비스가 가질 수 있는 capability 허용 범위 제한
	- 즉 컨트롤러가 가질 수 있는 cap 권한을 cap_net_bind로 제한
- NoNewPrivileges=false
	- 커널이 새로운 권한을 부여할 수 있도록 함
```
AmbientCapabilities=CAP_NET_BIND_SERVICE
CapabilityBoundingSet=CAP_NET_BIND_SERVICE
NoNewPrivileges=false
```
---------------
# 참고 일감
-  https://redmine.piolink.com/issues/49212 리모트 콘솔
- https://redmine.piolink.com/issues/49696 nginx proxy