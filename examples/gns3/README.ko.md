# Chisel / Ligolo-ng Multi-hop Pivoting GNS3 랩

`multi-hop-pivoting-lab.gns3`는 GNS3 2.2.37과 로컬 VirtualBox를 사용하는 실제 VM 기반 랩이다. GNS3 VM이나 Dynamips 라우터를 필요로 하지 않도록 VM 사이를 직접 연결했으며, 각 링크가 하나의 격리된 네트워크 구간을 나타낸다.

## 토폴로지

```text
Kali-Attacker ── 외부망 ── WebSrv1 ── DMZ/중간망 ── PivotHost2 ── 내부망 ── WebSrv2
```

| 노드 | 역할 | NIC |
|---|---|---:|
| `Kali-Attacker` | 외부 공격자 | 2 |
| `WebSrv1` | 외부에서 접근 가능한 시작점, 첫 번째 Pivot Host | 2 |
| `PivotHost2` | 두 번째 Pivot Host | 2 |
| `WebSrv2` | 외부에서 직접 접근하지 않는 최종 대상 | 2 |

프로젝트 파일은 다음과 같은 로컬 VirtualBox VM 이름을 참조한다.

| GNS3 노드 | VirtualBox VM |
|---|---|
| `Kali-Attacker` | `Standalone Kali New` |
| `WebSrv1` | `Standalone Ubuntu 1` |
| `PivotHost2` | `Standalone Ubuntu 2` |
| `WebSrv2` | `Standalone Ubuntu 3` |

다른 PC에서 열 때는 동일한 이름의 VirtualBox VM을 등록하거나 GNS3 VirtualBox 템플릿에서 해당 VM을 다시 선택해야 한다.

## 로그인 및 주소 계획

로그인 정보는 프로젝트 파일에 저장하지 않는다. 실제 VM 콘솔에서 Kali는 사용자가 지정한 `kali` 계정, Ubuntu VM은 `sunrin` 계정을 사용한다. 실습 후에는 기본 비밀번호를 변경한다.

| 구간 | 권장 주소 |
|---|---|
| 외부망 | Kali `192.168.1.100/24` ↔ WebSrv1 `192.168.1.10/24` |
| DMZ/중간망 | WebSrv1 `192.168.2.10/24` ↔ PivotHost2 `192.168.2.20/24` |
| 내부망 | PivotHost2 `192.168.3.20/24` ↔ WebSrv2 `192.168.3.30/24` |

`WebSrv1`과 `PivotHost2`에서만 IPv4 forwarding을 활성화하고, Kali와 WebSrv2에는 필요한 정적 경로만 추가한다. GNS3에서 직접 연결된 NIC 순서는 프로젝트의 포트 번호를 기준으로 확인한다.

## 현재 구성 및 검증 결과

현재 로컬 랩 VM에 다음 설정을 적용했다.

- WebSrv1: `192.168.1.10/24`, `192.168.2.10/24`, IPv4 forwarding 활성화
- PivotHost2: `192.168.2.20/24`, `192.168.3.20/24`, IPv4 forwarding 활성화
- WebSrv2: `192.168.3.30/24`, 테스트 HTTP 서버 `8080/tcp`
- Kali: `192.168.1.100/24`, `192.168.2.0/24` 및 `192.168.3.0/24` 정적 경로

Kali에서 WebSrv1과 WebSrv2로 각각 ICMP 및 HTTP 요청을 확인했다. WebSrv2 응답은 `Web Server 2 - internal target`이며, 세그먼트 간 라우팅과 테스트 서비스가 정상 동작한다. 이 설정은 게스트 재부팅 후에도 유지하려면 각 Ubuntu의 Netplan 또는 systemd-networkd 설정으로 옮겨야 한다.

## 실습 순서

1. GNS3에서 `multi-hop-pivoting-lab.gns3`를 연다.
2. 네 노드를 시작하고 각 Ubuntu 콘솔에서 `ip -br link`로 NIC 이름을 확인한다.
3. 위 주소 계획을 적용하고 `WebSrv1`, `PivotHost2`에서 `net.ipv4.ip_forward=1`을 설정한다.
4. WebSrv1에서 Chisel 서버 또는 reverse 터널 종단점을 구성한다. 인증 파일과 고정 fingerprint를 사용하고, 인터넷에 노출하지 않는다.
5. Ligolo-ng relay는 Kali의 격리된 랩 인터페이스에서만 열고, agent는 WebSrv1 또는 PivotHost2에서 실행한다.
6. 한 홉씩 `ping`, TCP 연결, 테스트 HTTP 요청을 확인한 뒤 두 번째 홉으로 확장한다.
7. ACL/iptables를 적용하고 동일한 검증을 반복하여 직접 접근은 실패하고 허용된 pivot 경로만 성공하는지 기록한다.

## 터널 실습 명령 예시

아래 명령은 격리된 랩에서 Chisel/Ligolo-ng 공식 릴리스 바이너리를 각 VM에 준비한 뒤 실행한다. 바이너리와 인증서, 비밀번호는 프로젝트 파일에 저장하지 않는다.

### Chisel

Kali에서 reverse 서버를 실행하고 WebSrv1에서 연결한다.

```bash
# Kali
chisel server --reverse --host 192.168.1.100 --port 8000

# WebSrv1
chisel client --fingerprint <server-fingerprint> 192.168.1.100:8000 R:18080:192.168.3.30:8080
```

그 후 Kali에서 `curl http://127.0.0.1:18080`으로 내부 WebSrv2에 대한 첫 번째 reverse forwarding을 확인한다. 두 번째 홉은 WebSrv1과 PivotHost2 사이에 별도의 인증된 Chisel 연결을 추가하고, 임시 포트는 서로 겹치지 않게 사용한다.

### Ligolo-ng

Kali에서 relay를 열고, 먼저 WebSrv1 agent를 연결한다. WebSrv1과 PivotHost2가 별도 세션으로 연결되면 relay의 TUN 인터페이스에 내부 경로를 추가한다.

```bash
# Kali
ligolo-ng relay -selfcert -laddr 192.168.1.100:11601

# WebSrv1 또는 PivotHost2
ligolo-ng agent -connect 192.168.1.100:11601 -ignore-cert
```

릴레이 콘솔에서는 승인된 세션만 선택하고 `192.168.3.0/24` 경로를 추가한다. 실습이 끝나면 relay, agent, TUN 경로를 모두 제거한다.

이 랩은 본인이 소유하거나 허가받은 격리 환경에서만 사용한다. 실제 제3자 시스템에 대한 스캔, 침투, 터널링에는 사용하지 않는다.

## 참고 자료

- [Chisel 공식 저장소](https://github.com/jpillora/chisel)
- [Ligolo-ng 공식 문서](https://docs.ligolo.ng/)
- [GNS3 파일 형식](https://gns3-server.readthedocs.io/en/latest/file_format.html)
