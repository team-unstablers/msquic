# macOS `datapath_kqueue.c` 조사 보고서

## 요약

`src/platform/datapath_kqueue.c`는 기본적인 UDP 송수신과 kqueue 이벤트 처리 자체는 구현되어 있지만, 현재 상태는 Linux `epoll` 경로 대비 기능과 성능 면에서 분명한 격차가 있다.

핵심 결론은 다음과 같다.

- ~~단기적으로는 충분히 개선 가능하다.~~ 핵심 인프라 개선이 완료되어 실사용 가능한 UDP 구현 수준에 도달했다.
- ~~특히 `feature` 플래그 정합성, TTL/Hop Limit 수신, 소켓 옵션 보강, receive buffer 설정은 저위험 대비 효과가 크다.~~ feature 플래그 정합성 복구 (`SEND_DSCP`, `RECV_DSCP`, `LOCAL_PORT_SHARING`), `SO_REUSEPORT`, 배치 I/O (`sendmsg_x`/`recvmsg_x`), 다중 파티션이 구현되었다.
- ~~반면 TCP parity, 다중 파티션 확장, macOS 전용 배치 I/O 도입은 별도 규모의 작업으로 봐야 한다.~~ 다중 파티션과 배치 I/O는 해소됐고, TCP parity만 별도 프로젝트로 남아 있다.
- ~~`datapath_kqueue.c` 하나만 고쳐서는 해결되지 않는 제약도 있다. Darwin 공용 계층인 `src/platform/platform_posix.c`가 현재 macOS를 사실상 single-core 플랫폼으로 취급한다.~~ `platform_posix.c`의 single-core 제약은 `pthread_cpu_number_np()` 도입으로 해소했다.
- 남은 저비용 과제: TTL feature advertising end-to-end 완성.

## 조사 범위

- `src/platform/datapath_kqueue.c`
- 비교 기준: `src/platform/datapath_epoll.c`
- 관련 공용 계층: `src/platform/datapath_unix.c`, `src/platform/platform_posix.c`
- 단위 테스트: `src/platform/unittest/DataPathTest.cpp`
- macOS SDK 헤더/로컬 manpage

## 현재 상태

### 1. kqueue 구현은 존재하지만 기능 플래그가 거의 비어 있음

`CxPlatDataPathInitialize()`는 UDP callback과 worker pool만 검사하고 초기화한 뒤, `Datapath->Features`를 별도로 채우지 않는다.

- `src/platform/datapath_kqueue.c:432`
- `src/platform/datapath_kqueue.c:475`
- `src/platform/datapath_kqueue.c:558`

반면 Linux `epoll` 경로는 초기화 중에 feature probing을 수행해서 아래 기능들을 명시적으로 설정한다.

- `CXPLAT_DATAPATH_FEATURE_LOCAL_PORT_SHARING`
- `CXPLAT_DATAPATH_FEATURE_TCP`
- `CXPLAT_DATAPATH_FEATURE_TTL`
- `CXPLAT_DATAPATH_FEATURE_SEND_DSCP`
- `CXPLAT_DATAPATH_FEATURE_RECV_DSCP`
- 조건부로 `SEND_SEGMENTATION`, `RECV_COALESCING`

근거:

- `src/platform/datapath_epoll.c:209`
- `src/platform/datapath_epoll.c:336`
- `src/platform/datapath_epoll.c:416`

즉, mac 경로는 실제로 일부 가능한 기능도 "지원 안 함"처럼 보이게 만들고 있다.

### 2. 기본 UDP 수신/송신 경로는 동작하지만 고급 메타데이터 처리가 부족함

현재 kqueue 경로는 `recvmsg()`로 ancillary data를 읽어서 다음은 처리한다.

- local address (`IP_PKTINFO` / `IP_RECVDSTADDR` / `IPV6_PKTINFO`)
- interface index (`IP_RECVIF` 또는 pktinfo 계열)
- TOS / traffic class (`IP_RECVTOS`, `IP_TOS`, `IPV6_TCLASS`)

근거:

- `src/platform/datapath_kqueue.c:724`
- `src/platform/datapath_kqueue.c:774`
- `src/platform/datapath_kqueue.c:1113`
- `src/platform/datapath_kqueue.c:1122`

하지만 TTL / Hop Limit은 명시적으로 미구현이다.

- `src/platform/datapath_kqueue.c:1115`

### 3. send batching — ~~사실상 비활성 상태~~ 구현 완료 (macOS)

~~파일 시작부터 batching TODO가 남아 있고, batch size가 1로 고정되어 있다.~~

macOS에서 `CXPLAT_MAX_BATCH_SEND=16`으로 확대하고, connected 소켓에서는 `sendmsg_x`로 배치 전송, 비연결 소켓에서는 `sendmsg` 반복으로 버퍼별 데이터그램을 전송하도록 send path를 재작성했다. iOS에서는 `CXPLAT_MAX_BATCH_SEND=1`로 기존 동작을 유지한다.

### 4. TCP는 아직 포팅되지 않음

TCP client/listener 생성 API가 모두 `QUIC_STATUS_NOT_SUPPORTED`를 반환한다.

- `src/platform/datapath_kqueue.c:1488`
- `src/platform/datapath_kqueue.c:1506`

send completion 경로에도 TCP TODO가 남아 있다.

- `src/platform/datapath_kqueue.c:1879`

즉 현재 macOS kqueue datapath는 실질적으로 UDP 우선 구현이다.

### 5. route / RSS 관련 API도 대부분 placeholder 수준

- `CxPlatResolveRoute()`는 즉시 `RouteResolved`로 끝낸다.
- `CxPlatUpdateRoute()`는 no-op이다.
- `CxPlatDataPathRssConfigGet()`는 `NOT_SUPPORTED`다.

근거:

- `src/platform/datapath_kqueue.c:2144`
- `src/platform/datapath_kqueue.c:2162`
- `src/platform/datapath_kqueue.c:2173`

이건 Linux의 raw/XDP 계층처럼 고급 경로 제어를 하는 설계와는 거리가 있다.

## Linux `epoll` 대비 주요 격차

### 1. 기능 감지와 feature advertising

Linux는 실제 probing 후 feature를 공개한다.

- GSO/GRO loopback 검증 후 segmentation/coalescing 활성화
- TTL/DSCP/TCP feature 활성화
- local port sharing 활성화

근거:

- `src/platform/datapath_epoll.c:214`
- `src/platform/datapath_epoll.c:276`
- `src/platform/datapath_epoll.c:304`
- `src/platform/datapath_epoll.c:336`

macOS는 이런 단계가 전혀 없다. 결과적으로 테스트와 상위 계층이 지원 여부를 보수적으로 해석하게 된다.

### 2. TTL 수신

Linux는 socket option에서 TTL/HopLimit 수신을 활성화하고, recv 경로에서 ancillary data를 파싱한다.

- `src/platform/datapath_epoll.c:835`
- `src/platform/datapath_epoll.c:861`
- `src/platform/datapath_epoll.c:1826`
- `src/platform/datapath_epoll.c:1929`

macOS도 SDK 차원에서 필요한 옵션이 있다.

- `IP_RECVTTL`: `/Library/Developer/CommandLineTools/SDKs/MacOSX.sdk/usr/include/netinet/in.h:431`
- `IPV6_RECVHOPLIMIT`: `/Library/Developer/CommandLineTools/SDKs/MacOSX.sdk/usr/include/netinet6/in6.h:459`

즉, 이 부분은 OS 한계보다는 구현 공백에 가깝다.

### 3. receive buffer 설정 — 완료

Linux는 `SO_RCVBUF`를 크게 키운다.

- `src/platform/datapath_epoll.c:899`

macOS도 동일한 의도를 `kqueue` 경로에 복원했다.

- `src/platform/datapath_kqueue.c:920`

구현 내용:

- `CxPlatSocketContextInitialize()`에서 `SO_RCVBUF`에 `INT32_MAX`를 요청한다.
- 실패 시 `DatapathErrorStatus` trace 후 socket init을 중단하도록 `epoll`/WinUser와 동일한 에러 경로를 따른다.
- Darwin은 oversized request를 커널 최대값으로 clamp하므로 별도 `sysctl` probe 없이 parity 동작을 택했다.

검증 메모:

- `msquicplatformtest`의 기본 UDP 송수신 케이스 (`UdpBind`, `UdpData`, `UdpDataECT0`)를 IPv4/IPv6에서 통과시켰다.
- 로컬 Darwin UDP socket probe에서 `SO_RCVBUF=INT32_MAX` 요청이 오류 없이 더 큰 수신 버퍼로 반영되고, 커널 상한으로 clamp되는 것을 확인했다.

### 4. local port sharing / multi-socket fanout — ~~격차~~ 해소됨

~~그러나 kqueue 경로에는 해당 설정이 없다. 게다가 datapath partition 수도 1로 고정되어 있다.~~

epoll 경로의 패턴을 이식하여 `SO_REUSEPORT`를 도입했다. 서버 소켓 또는 `CXPLAT_SOCKET_FLAG_SHARE` 플래그가 설정된 소켓에서 `PartitionCount > 1`이면 자동 설정된다. `CXPLAT_DATAPATH_FEATURE_LOCAL_PORT_SHARING` feature 플래그도 활성화했다.

### 5. 다중 파티션 활용 — ~~불가~~ 구현 완료

~~macOS 공용 계층은 현재 프로세서 수를 1로 강제하고, 현재 processor index도 항상 0을 돌려준다.~~

`platform_posix.c`에서 `CxPlatProcessorCount`를 `sysconf(_SC_NPROCESSORS_ONLN)`으로 복원하고, `CxPlatProcCurrentNumber()`에 `pthread_cpu_number_np()` (macOS 11.0+)를 도입했다. `datapath_kqueue.c`에서도 `PartitionCount`를 `CxPlatWorkerPoolGetCount(WorkerPool)`로 복원하고, `SO_REUSEPORT` 기반 다중 소켓 모델을 활성화했다.

## macOS에서 실제로 활용 가능한 API 단서

### 1. `IP_RECVTTL`, `IPV6_RECVHOPLIMIT`

SDK 헤더에 정의가 있다.

- `/Library/Developer/CommandLineTools/SDKs/MacOSX.sdk/usr/include/netinet/in.h:431`
- `/Library/Developer/CommandLineTools/SDKs/MacOSX.sdk/usr/include/netinet6/in6.h:459`

따라서 TTL/Hop Limit 수신은 충분히 구현 가능성이 높다.

### 2. `SO_REUSEPORT`, `SO_NOSIGPIPE`

둘 다 macOS socket option으로 공식 정의돼 있다.

- `/Library/Developer/CommandLineTools/SDKs/MacOSX.sdk/usr/include/sys/socket.h:137`
- `/Library/Developer/CommandLineTools/SDKs/MacOSX.sdk/usr/include/sys/socket.h:169`

특히 `SO_NOSIGPIPE`는 TCP를 포팅할 경우 사실상 필수에 가깝다. 현재 로컬 브랜치에는 이를 `datapath_kqueue.c`에 추가한 커밋도 있다.

- commit `c91fbe4b3`: `platform/datapath_kqueue: set SO_NOSIGPIPE to suppress SIGPIPE on sends`

### 3. `connectx()`

macOS에는 `connectx()`가 public API로 존재한다.

- `/Library/Developer/CommandLineTools/SDKs/MacOSX.sdk/usr/include/sys/socket.h:740`
- `man 2 connectx`

이 API는 source address/interface 지정과 bind+connect 결합에 유리해서, 현재 kqueue 코드가 `bind()` 후 `connect()`를 따로 수행하는 부분을 정리하는 데 활용 여지가 있다.

현재 코드:

- `src/platform/datapath_kqueue.c:816`
- `src/platform/datapath_kqueue.c:860`

다만 이건 correctness fix보다는 구조 개선에 가깝다.

### 4. `sendmsg_x` / `recvmsg_x`

상태:

- [x] 구현 완료 (macOS 전용, iOS 비활성)

Darwin private syscall (`SYS_recvmsg_x=480`, `SYS_sendmsg_x=481`)로 공개 SDK에 프로토타입이 없다. `struct msghdr_x`와 함수 프로토타입을 `datapath_kqueue.c` 상단에 직접 선언해서 사용한다.

**`recvmsg_x` (배치 수신)**:
- `CXPLAT_MAX_BATCH_RECV=16`개 데이터그램을 1회 syscall로 수신.
- ancillary data (pktinfo, TOS, TTL 등)를 정상 지원하므로 제약 없음.
- `CxPlatSocketContextRecvComplete`를 리팩터링하여 명시적 `struct msghdr*`과 `DATAPATH_RX_IO_BLOCK*`을 받도록 변경.
- 기존 4회 `recvmsg` 루프는 fallback 경로로 유지.

**`sendmsg_x` (배치 송신)**:
- `msg_name`과 `msg_control`을 모두 NULL로 강제하는 제약이 있다.
- 따라서 **connected 소켓에서만** 사용 가능. TOS/DSCP는 `setsockopt`로 사전 설정.
- 비연결 소켓은 기존 `sendmsg()` 반복으로 처리.

**`ENOSYS` fallback**: 첫 호출에서 `ENOSYS`가 반환되면 `Datapath->HasBatchIo = FALSE`로 전환하여 이후 모든 호출에서 `sendmsg`/`recvmsg` 반복으로 폴백한다.

**iOS 가드**: `#if !TARGET_OS_IPHONE`으로 전체 배치 I/O 경로를 비활성화. iOS에서는 `CXPLAT_MAX_BATCH_SEND=1`, `CXPLAT_MAX_BATCH_RECV=1`로 기존 동작을 유지한다.

## 실질적인 개선 우선순위

### 1단계: 바로 손댈 가치가 큰 항목

#### A. feature 플래그 정합성 복구

상태:

- [x] `CXPLAT_DATAPATH_FEATURE_SEND_DSCP`
- [x] `CXPLAT_DATAPATH_FEATURE_RECV_DSCP`
- [ ] `CXPLAT_DATAPATH_FEATURE_TTL`
- [x] `CXPLAT_DATAPATH_FEATURE_LOCAL_PORT_SHARING`

메모:

- `SEND_DSCP` / `RECV_DSCP`는 `Datapath->Features` 계산 경로를 추가해서 실제 지원 상태와 맞췄다.
- `TTL`은 ancillary parsing helper와 관련 guard는 들어갔지만, dual-mode wildcard listener가 IPv4 TTL metadata를 truthfully 보장하지 못해서 feature advertising은 보류했다.
- 이 작업을 진행하면서 pure IPv4 socket과 dual-mode listener 사이의 `sendmsg()` destination family 정합성 문제도 함께 수정했다.

권장 항목:

- `CXPLAT_DATAPATH_FEATURE_TTL`
- `CXPLAT_DATAPATH_FEATURE_SEND_DSCP`
- `CXPLAT_DATAPATH_FEATURE_RECV_DSCP`

조건부 검토:

- `CXPLAT_DATAPATH_FEATURE_LOCAL_PORT_SHARING`

이 단계는 상위 계층과 테스트가 macOS 능력을 더 정확히 반영하게 만든다.

관련 테스트 근거:

- TTL 검사: `src/platform/unittest/DataPathTest.cpp:301`
- DSCP 검사: `src/platform/unittest/DataPathTest.cpp:307`
- sharing feature gate: `src/platform/unittest/DataPathTest.cpp:1001`
- TCP feature gate: `src/platform/unittest/DataPathTest.cpp:1101`

#### B. TTL/Hop Limit 수신 구현

상태:

- [ ] end-to-end 완료 아님
- [x] IPv4 / IPv6 TTL ancillary parsing helper 추가
- [ ] socket init에서 `IP_RECVTTL` / `IPV6_RECVHOPLIMIT`를 실제 enable하고 feature까지 advertise

작업 포인트:

- socket init에서 `IP_RECVTTL` / `IPV6_RECVHOPLIMIT` 설정
- recv complete에서 `IP_TTL` / `IPV6_HOPLIMIT` cmsg 파싱
- `RecvPacket->HopLimitTTL` 설정

이건 구현 난이도에 비해 확실한 품질 개선이다.

#### C. `SO_RCVBUF` 복구

상태:

- [x] 완료

`CxPlatSocketContextInitialize()`에서 `setsockopt(SO_RCVBUF, INT32_MAX)`를 복원했다. Darwin이 oversized request를 커널 최대값으로 clamp하므로 별도 상한 탐색 없이 Linux/Windows와 동일한 요청값을 사용한다. 실패 시에는 기존 datapath 초기화 패턴대로 trace 후 즉시 실패 처리한다.

### 2단계: 중간 규모 개선

#### A. `SO_REUSEPORT` 도입

상태:

- [x] 완료

epoll 경로의 패턴을 이식했다. 서버 소켓 또는 `CXPLAT_SOCKET_FLAG_SHARE` 플래그가 설정된 소켓에서 `PartitionCount > 1`이면 `SO_REUSEPORT`를 설정한다. `CXPLAT_SOCKET` 구조체에 `SharedBinding` 비트필드를 추가해서 `Config->Flags`의 정보를 `CxPlatSocketContextInitialize`까지 전달한다.

#### B. send batching 개선

상태:

- [x] 완료 (macOS), iOS에서는 기존 동작 유지

`CXPLAT_MAX_BATCH_SEND`를 16으로 확대하고 `CxPlatSocketSendInternal`의 send 경로를 재작성했다.

- **connected 소켓**: `sendmsg_x`로 다중 데이터그램을 1회 syscall로 전송. TOS/DSCP는 `setsockopt`로 사전 설정.
- **비연결 소켓**: 버퍼별 `sendmsg()` 반복 (ancillary data 필요).
- **EAGAIN 시**: `CurrentIndex`로 부분 전송 상태를 추적하여 pended send에서 이어서 전송.
- **`ENOSYS` fallback**: `sendmsg_x` 미지원 시 `HasBatchIo = FALSE`로 전환하여 이후 `sendmsg` 반복으로 폴백.
- **iOS**: `#if TARGET_OS_IPHONE`으로 배치 경로를 비활성화하고, `CXPLAT_MAX_BATCH_SEND=1`로 기존 단일 버퍼 동작을 유지.

### 3단계: 별도 프로젝트로 봐야 할 항목

#### A. TCP 포팅

이건 사실상 `epoll` TCP path를 kqueue semantics에 맞게 다시 옮기는 작업이다.

필수 범위:

- listener 생성
- accept 처리
- passive socket 관리
- TCP send / recv completion
- `SO_NOSIGPIPE` 처리
- 통계 API

참고 기준:

- `src/platform/datapath_epoll.c:1545`
- `src/platform/datapath_epoll.c:1566`
- `src/platform/datapath_epoll.c:1667`
- `src/platform/datapath_epoll.c:2615`

#### B. multi-partition / per-processor scaling

상태:

- [x] 완료

다음 세 곳을 수정해서 macOS에서 다중 파티션을 활성화했다.

1. **`src/platform/platform_posix.c` — 프로세서 카운트**: macOS 분기의 `CxPlatProcessorCount = 1` 하드코딩을 제거하고, Linux와 동일하게 `sysconf(_SC_NPROCESSORS_ONLN)`을 사용하도록 통합했다.
2. **`src/platform/platform_posix.c` — 현재 CPU 번호**: `CxPlatProcCurrentNumber()`의 macOS 분기에서 `return 0` 대신 `pthread_cpu_number_np()` (macOS 11.0+)를 사용한다. 이 API는 approximate하지만 파티션 배정 힌트로만 쓰이므로 correctness에 영향 없다.
3. **`src/platform/datapath_kqueue.c` — 파티션 카운트**: `Datapath->PartitionCount = 1` 하드코딩을 `CxPlatWorkerPoolGetCount(WorkerPool)`로 복원했다.

결과적으로 macOS에서도 CPU 코어 수만큼 워커 스레드와 datapath 파티션이 생성되며, 서버 소켓은 `SO_REUSEPORT`로 파티션별 소켓을 공유한다. 스레드 어피니티(`thread_policy_set`)는 macOS에서 커널이 무시할 수 있어 별도 후속 작업으로 분리했다.

## 가능/불가능 요약

| 가능 | 불가능 또는 제약 큼 |
|---|---|
| feature 플래그 정확하게 설정 | Linux GSO/GRO parity |
| TTL/HopLimit 수신 파싱 | `SO_ATTACH_REUSEPORT_CBPF` 류 RSS |
| DSCP 송수신 feature 정합성 복구 | QEO 하드웨어 오프로드 |
| `SO_RCVBUF` 복구 완료 | Linux와 동일한 커널 fanout 모델 |
| `SO_REUSEPORT` 기반 제한적 확장 | |
| `sendmsg()` 반복 기반 batching 개선 | |
| TCP 지원 추가 | |

## 주의할 점

### 1. 기존 코드의 dual-stack 회피 로직은 실제 macOS 이슈의 흔적이다

현재 코드에는 다음 주석이 있다.

- `There is problem with receiving PKTINFO on dual-mode when binded and connect to IP4 endpoints. For that case we use AF_INET.`

위치는 다음과 같다.

- `src/platform/datapath_kqueue.c:623`

초기 macOS 지원 커밋 메시지도 datapath test 실패 원인으로 dual-mode socket 문제를 직접 언급한다.

- commit `50683f5b3`

따라서 multi-socket이나 local port sharing을 도입할 때, dual-stack 정합성부터 다시 검증해야 한다.

### 2. `IP_DONTFRAG`는 헤더에는 있지만 runtime probe가 안전하다

현재 코드 주석은 macOS에서 미지원이라고 적고 있다.

- `src/platform/datapath_kqueue.c:716`

하지만 최신 SDK 헤더에는 `IP_DONTFRAG` 정의가 있다.

- `/Library/Developer/CommandLineTools/SDKs/MacOSX.sdk/usr/include/netinet/in.h:436`

즉, "절대 불가능"으로 박기보다 runtime `setsockopt()` probe 후 feature 여부를 정하는 쪽이 더 맞다.

## 최종 결론

`datapath_kqueue.c`는 ~~당장 폐기할 정도로 빈약한 파일은 아니다. UDP 기반의 기본 기능은 갖췄다. 다만 현재는 "최소 동작 구현"에 더 가깝고,~~ 다중 파티션, 배치 I/O, `SO_REUSEPORT` 등 핵심 인프라가 구현된 상태다. Linux `epoll` 경로와 비교하면 다음 층위의 차이가 남아 있다.

- 저비용 누락: ~~feature flags,~~ TTL feature advertising
- ~~중간 규모 격차: `SO_REUSEPORT`, batching~~ → 해소됨
- ~~대규모 미구현: TCP, multi-partition scaling~~ → multi-partition 해소, TCP만 잔존
- 잔존 대규모 미구현: TCP 포팅

가장 현실적인 개선 전략은 아래 순서다.

1. ~~feature advertising 정리~~ → 완료 (`SEND_DSCP`, `RECV_DSCP`, `LOCAL_PORT_SHARING`)
2. TTL/HopLimit 수신 end-to-end 완성 (feature advertising 포함)
3. ~~`SO_RCVBUF` 복구~~ → 완료 (`setsockopt(SO_RCVBUF, INT32_MAX)`, Darwin clamp)
4. ~~`SO_REUSEPORT` 실험적 도입~~ → 완료
5. ~~batching 개선~~ → 완료 (`sendmsg_x`/`recvmsg_x` + `sendmsg` 반복 폴백)
6. ~~multi-partition 재설계~~ → 완료 (`pthread_cpu_number_np` + 파티션 카운트 복원)
7. TCP 포팅

macOS datapath는 "실사용 가능한 UDP 구현" 수준에 도달했다. 남은 핵심 과제는 TTL feature advertising 완성과 TCP 포팅이다.
