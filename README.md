# tcp-nagle-lab

TCP Nagle 알고리즘 vs TCP_NODELAY가 API 서버 latency에 미치는 실제 영향을 검증한다.

## 배경

거의 모든 웹 서버/프레임워크가 `TCP_NODELAY`를 기본으로 켜는데, 왜 그런지 직접 검증해본 사람은 드물다.

- **Nagle ON (기본)**: 작은 패킷을 모아서 전송 → bandwidth 효율적이지만 latency 증가
- **Nagle OFF (TCP_NODELAY)**: 즉시 전송 → latency 감소, 소규모 패킷 다수 발생
- **Delayed ACK와의 상호작용**: Nagle + Delayed ACK가 합쳐지면 최대 200ms 지연 발생 가능

## 가설

| # | 가설 | 근거 |
|---|------|------|
| H1 | 작은 JSON 응답(<200B)에서 Nagle ON이 40-200ms latency 추가할 것 | Nagle이 ACK 대기하며 패킷 보류 |
| H2 | 큰 응답(>1KB)에서는 Nagle ON/OFF 차이가 거의 없을 것 | MSS를 채우면 Nagle이 즉시 전송 |
| H3 | Nagle OFF + 작은 응답에서 패킷 수가 2-3배 증가할 것 | 헤더 오버헤드 비율 증가 |

## 실험 설계

### 서버
- Go net/http 서버에서 `net.TCPConn.SetNoDelay(true/false)` 토글
- 응답 크기: 50B / 200B / 500B / 1KB / 1.4KB / 10KB

### 측정
- k6로 latency (avg, p95, p99) 측정
- tcpdump로 패킷 수, 패킷 크기 분포 캡처
- Wireshark으로 Nagle + Delayed ACK 상호작용 시각화
