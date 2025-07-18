## 정의
vnic
- QEMU가 제공하는 가상 nic
virtio-net 드라이버
- 게스트가 vnic을 인식/사용할 수 있도록 해주는 드라이버


- 기본 구조
```text
[게스트 OS]
  └─ VirtIO 드라이버 (virtio-blk, virtio-net 등)
        ↓
[QEMU 사용자 공간]
  └─ VirtIO 장치 모델 (virtio-blk-device.c 등)
        ↓
[호스트 자원]
  └─ 실제 파일, 네트워크 인터페이스 등
```

- 게스트 입장에서는 virtio 디스크, 네트워크 장치를 실제 장치처럼 인식하지만, 사실 내부적으로는 virtio 드라이버 임
- 게스트가 보내는 요청은 VirtQueue라는 공유 메모리 큐에 전달
- 해당 메모리 큐를 qemu가 감지하고, 실제 하드웨어 장치로 처리

```
게스트 OS
┌────────────┐
│ VirtIO 드라이버 │◀─ I/O 요청 (virtqueue)
└────────────┘
        ↓ 공유 메모리
QEMU
┌────────────┐
│ VirtIO 장치 모델 │─▶ 실제 호스트 자원 접근
└────────────┘
```