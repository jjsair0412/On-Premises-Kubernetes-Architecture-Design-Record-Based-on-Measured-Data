

## Verified Facts

문서와 실제 구현이 어긋나는 지점들입니다. 각 항목은 상위 프로젝트의 문서 또는 소스 코드에서 확인했습니다.
> **English :** Points where documentation and actual behaviour diverge. Each item was checked against upstream documentation or source code.

|항목(Item)|확인된 사실(Verified fact)|설계 반영(Applied as)|
|--|--|--|
|RKE2 v1.35 default ingress|여전히 `ingress-nginx`. Traefik이 기본이 되는 것은 v1.36, nginx 제거는 v1.37<br>Still `ingress-nginx`. Traefik becomes the default in v1.36; nginx is removed in v1.37|`ingress-controller: traefik` 명시<br>set explicitly|
|ingress-nginx EOL|2026-03 은퇴. 후속으로 계획됐던 InGate 프로젝트도 취소됨<br>Retired 2026-03. The planned successor, InGate, was also cancelled|Traefik 표준화<br>standardize on Traefik|
|Cilium 1.19 BGPv1|`CiliumBGPPeeringPolicy` **제거됨**<br>**removed**|`cilium.io/v2` 4종 CRD<br>the four v2 CRDs|
|Cilium 1.19 new|`advertisementType: Interface` — 로컬 인터페이스의 임의 IP를 /32로 광고<br>advertises arbitrary IPs on a local interface as /32|Option A의 핵심 메커니즘<br>the core mechanism of Option A|
|Egress GW failover|**없음.** 게이트웨이 선택 로직에 readiness·taint·unschedulable 검사가 없음 ([#18230](https://github.com/cilium/cilium/issues/18230), open)<br>**None.** The gateway selection path contains no readiness, taint, or unschedulable check|Option B에 라벨 컨트롤러 필수<br>label controller is mandatory in Option B|
|`egressGateways` list (1.18+)|부하 분산이지 페일오버가 아님. 파드는 CiliumEndpoint UID로 배정되고, 노드가 NotReady여도 목록은 그대로<br>Load distribution, not failover. Pods are assigned by CiliumEndpoint UID and the list is unchanged when a node goes NotReady|Option A의 한계로 명시<br>documented as Option A's limit|
|Egress GW and MTU|켜면 라우팅 모드와 무관하게 터널 디바이스가 생성되어 파드 MTU가 내려감<br>Enabling it creates a tunnel device regardless of routing mode, lowering pod MTU|`MTU` 고정 후 실측<br>pin `MTU`, then measure|
|Egress GW prerequisites|`bpf.masquerade` · `kubeProxyReplacement` · identity `crd` · CiliumEndpointSlice 비활성. 하나라도 어긋나면 에이전트가 기동 자체를 거부<br>If any of these is wrong the agent refuses to start|Helm 값에 고정<br>pinned in Helm values|
|`egressIP` format|단일 IPv4만. CIDR 불가. `interface`와 동시 지정 시 정책 전체가 **조용히 무시됨**<br>Single IPv4 only, no CIDR. Setting it together with `interface` makes the policy **silently ignored**|둘 중 하나만 사용<br>use exactly one|
|SNAT connection limit|`{egressIP, dst IP, dst port}` 당 약 32,768<br>~32,768 per tuple|검증 항목에 포함<br>added to the verification checklist|
|RKE2 + Cilium API access|`k8sServiceHost: localhost` — agent가 127.0.0.1:6443에 클라이언트 사이드 LB를 띄움<br>the RKE2 agent runs a client-side load balancer on 127.0.0.1:6443|단일 서버 주소를 박지 않음<br>never hardcode one server address|
|RKE2 secrets encryption|FIPS는 `aescbc` 한정. `secretbox`는 불가<br>FIPS applies to `aescbc` only, not `secretbox`|`aescbc` 고정|
|etcd encryption key|복호화 키가 같은 노드에 평문으로 존재<br>The decryption key sits in plaintext on the same node|Vault를 2계층으로 병행<br>Vault added as a second tier|
|CIS profile value|v1.29+ 는 `cis` (버전 표기 없음)<br>plain `cis` since v1.29|`profile: cis`|
|SL Micro install method|`install.sh`는 tarball 기본 → `/usr/local` 또는 `/opt/rke2`, **OS 스냅샷 밖**. RPM은 `INSTALL_RKE2_METHOD=rpm`<br>tarball is the default and lands **outside** the OS snapshot|RPM 강제 (롤백 일관성)<br>force RPM for rollback consistency|
|Traefik chart 41.x|`service.type` 키 **삭제됨** → `service.spec.type`. 구 표기는 무시되고 ClusterIP로 뜸<br>`service.type` is **gone**; the old key is silently ignored and you get a ClusterIP|신 표기 사용<br>use the new key|
|Split-role etcd|쿼럼 수식은 동일. 분리 근거는 장애 격리와 운영 표준화<br>Quorum arithmetic is unchanged; the justification is fault isolation and runbook uniformity|근거를 문서에 명시<br>stated explicitly|

---