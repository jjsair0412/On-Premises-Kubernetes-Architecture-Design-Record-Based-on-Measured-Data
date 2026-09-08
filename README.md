# On-Premises Kubernetes Architecture Design Record Based on Measured Data
온프레미스 K8s Cluster 아키텍처를 선정하며, 각 계층 별 실측치를 기록합니다.

## 목차 (Contents)
1. [OverView](#overview)
2. [Design 정책 (Design Principles)](#design-principles)
3. [Environment](#environment)
4. [Architecture Diagram](#architecture-diagram)
5. [실측 필요 항목 (Items Requiring On-Site Measurement)](#실측-필요-항목-items-requiring-on-site-measurement)
6. [Contributing](#contributing)
7. [References](#references)

## OverView

해당 레포지토리는 온프레미스 환경에 적합한 K8s Cluster를 구성하기 위해 참고할 정보와, 솔루션 별 산재된 Best Practice를 모아 아키텍처로 설계한 내용을 작성하였습니다.
> **English :** This repository documents an architecture designed by collecting reference information and solution-specific best practices for building a K8s cluster suitable for on-premises environments.

Ansible 기반으로 코드화 하였으며, PR, Issue, Fork 모두 허용하며 수정요청또한 고마운 마음으로 받도록 하겠습니다.
> **English :** It is codified with Ansible. Pull requests, issues, and forks are all welcome, and suggestions for improvements are also greatly appreciated.

특히 아래 **Verified Facts** 섹션은 문서와 실제 구현이 어긋나는 지점을 소스 코드까지 확인해 정리한 것입니다. 틀린 부분이 있다면 Issue로 알려주시면 감사하겠습니다.
> **English :** In particular, the **Verified Facts** section below documents the points where official docs and actual implementation diverge, cross-checked against upstream source code. If anything here is wrong, please open an issue — corrections are the main thing this repository is asking for.

---

## Design 정책(Principles)
### **0. 물리 배치를 파악하고, 고 가용성에 사용한다.**

모든 솔루션의 고 가용성 확보를 위해, 실제 상면 Rack 별 Work Node 위치를 파악합니다.
공통 Rack에 모든 Ingress Traffic이 관리되거나, harbor와 같은 Image Registry가 위치할 경우 문제 발생 시 전체 장애로 이어질 수 있기 때문입니다.


전 노드 NodePort를 타겟으로 잡으면 워커를 늘릴 때마다 L4 정책이 늘어납니다. LoadBalancer + LB IPAM + BGP 광고로 바꾸면 backend가 VIP 개수로 고정됩니다.
> **English :** Decouple the upstream L4 backend count from the node count. Targeting NodePorts on every node means L4 config grows every time you add a worker. Using `LoadBalancer` + Cilium LB IPAM + BGP advertisement pins the backend list to the number of VIPs instead.

### **2. 인그레스는 성격에 따라 나눈다.**
트래픽 스위칭이 필요한 경로에만 서비스 메시를 붙입니다. 사내 도구 30개에 사이드카를 붙이면 자원만 쓰고 얻는 것이 없습니다.
> **English :** Split ingress by workload character. Attach the service mesh only to paths that actually need traffic switching — putting a sidecar on thirty internal tools costs resources and buys nothing.

### **3. 역할 노드를 분리하고 테인트로 격리한다.**
etcd / control-plane / ingress / egress / worker를 나눕니다. etcd 분리의 근거는 쿼럼이 아니라 **장애 격리와 운영 표준화**입니다.
> **English :** Separate node roles and isolate them with taints — etcd / control-plane / ingress / egress / worker. Note that splitting etcd onto dedicated nodes does **not** change quorum arithmetic; the real justification is fault isolation and a single operational runbook across clusters.

### **4. Immutable OS의 전제를 코드가 지킨다.**
패키지 변경은 `transactional-update`로 모아서 한 번, 재기동도 한 번. 노드에서 직접 `sysctl`을 고치지 않습니다.
> **English :** Respect the immutable OS contract in code. Package changes go through a single `transactional-update` transaction and a single reboot; nothing is mutated on the node by hand.

### **5. Application 및 인프라의 모든 코드는 GitOps 기준으로 작성된다.**
SSoT(Single Source of Truth, 단일 진실 공급원) 아키텍처 원칙을 고수하여 Git 상태만 보더라도 실제 클러스터 상태를 확인할 수 있도록 합니다.
> **English :** Adheres to the SSoT (Single Source of Truth) architectural principles so that the actual cluster state can be verified directly from the Git state alone.

---

## Environment

|구분(Category)|솔루션(Solution)|버전(Version)|비고(Notes)|
|--|--|--|--|
|OS|SUSE Linux Micro (SL Micro)|6.2|Immutable · `transactional-update` · btrfs snapshot|
|K8s|RKE2|v1.35.7+rke2r1|Kubernetes 1.35.7. RPM 설치 권장(RPM install recommended — see Verified Facts)|
|CNI|Cilium|1.19.x|kube-proxy replacement · BGP Control Plane v2 · Egress Gateway|
|Management|Rancher|v2.15.1|RKE2 v1.36 / v1.35 / v1.34 지원(supports)|
|Ingress (non-critical)|Traefik|v3.7.12 / chart 41.4.0|RKE2 패키지 컴포넌트(packaged component)|
|Ingress (business) · Mesh|Istio|1.31.0|k8s 1.32–1.36 지원(supported)|
|Egress HA|kube-vip|v1.2.3|Option B 전용(Option B only)|
|CI|Jenkins (LTS)|2.568.3|Tekton v1.15.1 LTS 도 대안(alternative)|
|CD|Argo CD|v3.5.2|`selfHeal` 기본 비활성(disabled by default — change control)|
|Registry|Harbor|v2.15.2 / chart 1.19.2|폐쇄망 미러 · OCI 차트 저장소(air-gapped mirror + OCI chart repo)|
|Authentication / Authorization|Keycloak|26.7.3|OIDC → Rancher · Argo CD · Grafana|
|Policy|Kyverno|v1.19.0 / chart 3.9.0|OpenShift SCC 대체(SCC replacement)|
|Runtime Security|NeuVector|v5.6.1 / chart 2.11.1|프로세스·파일 무결성, 이미지 CVE|
|Secrets|HashiCorp Vault|v2.1.0|+ External Secrets Operator v2.10.0|
|Certificates|cert-manager|v1.21.1|Ingress TLS|
|Monitoring (metrics)|Prometheus|v3.14.0 (LTS 3.13.0)|chart `kube-prometheus-stack` 89.2.2|
|Visualization|Grafana|13.2.1|Keycloak OIDC 연동|
|Logging|Grafana Loki|v3.7.7|chart 6.51.x|
|Backup|Velero|v1.18.2 / chart 12.1.0|etcd 스냅샷으로 못 하는 네임스페이스·PV 단위 복구|

버전은 2026-09-06 기준 각 프로젝트의 최신 안정 릴리스입니다. 사전 릴리스(RC)는 포함하지 않았습니다.
> **English :** Versions are the latest stable releases as of 2026-09-06. No pre-releases (RCs) are included.

> **Note on the original draft table :** 초안에서 Monitoring을 Grafana, Logging을 Prometheus로 적었는데 이는 뒤바뀐 표기입니다. Prometheus는 메트릭 수집, Grafana는 시각화, 로그는 Loki(또는 OpenSearch)가 담당합니다.
> An earlier draft listed Grafana under *Monitoring* and Prometheus under *Logging*. That is inverted — Prometheus collects metrics, Grafana visualizes, and Loki (or OpenSearch) handles logs.

---
## Architecture Diagram

    2026_09_06 In progress...

---

## 실측 필요 항목 (Items Requiring On-Site Measurement)

1. **파드 MTU** — `kubectl exec <pod> -- ping -M do -s 1422 <external IP>`
   egress gateway 활성 시 터널이 생기므로 이론값과 다를 수 있습니다.
   > Pod MTU. A tunnel appears once the egress gateway is enabled, so the effective value may differ from the calculated one.
2. **이그레스 노드 장애 시 동작** — 게이트웨이 노드를 `NotReady`로 만들고 관찰하십시오. **Option A는 끊기는 것이 정상 동작입니다.**
   > Behaviour on egress node failure. Force a gateway node to `NotReady` and watch. For Option A, traffic stopping *is* the expected behaviour.
3. **Option B의 전환 총 단절 시간** — 리스 만료 + 라벨 반영 + Cilium 재수렴.
   > Total switchover gap for Option B.
4. **BGP 수렴 시간** — 노드 재기동 시 경로 광고/철회에 걸리는 시간.
   > BGP convergence time on node reboot.
5. **cilium-agent 재시작 영향** — `kubeProxyReplacement` + socketLB + hostFirewall 조합에서 호스트 egress가 수 분간 끊긴 사례가 보고되어 있습니다 ([#45077](https://github.com/cilium/cilium/issues/45077), closed as not planned). **롤링 업그레이드 전에 반드시 확인하십시오.**
   > Impact of a cilium-agent restart. Multi-minute host egress outages have been reported for the `kubeProxyReplacement` + socketLB + hostFirewall combination. Verify this before any rolling upgrade.
6. **NeuVector L7 DPI 지연** — Istio mTLS 구간 밖에서만 유효한 측정입니다.
   > NeuVector L7 DPI latency — only measurable outside the Istio mTLS path.

---

## Contributing

이 저장소는 **틀린 부분을 지적받는 것**을 목적으로 공개합니다. 특히 다음 항목에 대한 반론을 환영합니다.
> **English :** This repository is published specifically to have its mistakes pointed out. Pushback on the following is especially welcome.

- Verified Facts 표의 각 항목 — 버전이 올라가면서 사실이 바뀐 것이 있는지
  > Each row of the Verified Facts table — anything that has changed with a newer release
- Egress Option A / B 외에 더 나은 접근이 있는지 (예: 노드 밖 프록시 계층, MetalLB + 별도 SNAT)
  > Better approaches than Options A and B — an off-cluster proxy tier, MetalLB with separate SNAT, and so on
- Traefik / Istio 이원화가 과설계인지, 아니면 더 나눠야 하는지
  > Whether the Traefik/Istio split is over-engineering, or should be split further
- Immutable OS 위에서 RPM vs tarball 선택의 실제 운영 경험
  > Real operational experience with RPM vs tarball on an immutable OS

PR, Issue, Fork 모두 환영합니다.
> **English :** PRs, issues and forks are all welcome.

---

## References

- [Cilium — Egress Gateway](https://docs.cilium.io/en/stable/network/egress-gateway/egress-gateway/)
- [Cilium — BGP Control Plane](https://docs.cilium.io/en/stable/network/bgp-control-plane/bgp-control-plane/)
- [Cilium — Kubernetes without kube-proxy](https://docs.cilium.io/en/stable/network/kubernetes/kubeproxy-free/)
- [cilium#18230 — Egress gateway does not tolerate single node failure](https://github.com/cilium/cilium/issues/18230)
- [cilium#39245 — Maintain existing connections when modifying egress gateways](https://github.com/cilium/cilium/discussions/39245)
- [cilium#45077 — Host loses egress connectivity on agent restart](https://github.com/cilium/cilium/issues/45077)
- [RKE2 — Ingress migration](https://docs.rke2.io/reference/ingress_migration)
- [RKE2 — Server roles](https://docs.rke2.io/install/server_roles)
- [RKE2 — Network options](https://docs.rke2.io/networking/basic_network_options)
- [RKE2 — Hardening guide](https://docs.rke2.io/security/hardening_guide)
- [RKE2 — Secrets encryption](https://docs.rke2.io/security/secrets_encryption)
- [Kubernetes — Ingress NGINX retirement](https://www.kubernetes.dev/blog/2025/11/12/ingress-nginx-retirement/)
- [Istio — Supported releases](https://istio.io/latest/docs/releases/supported-releases/)
- [traefik-helm-chart](https://github.com/traefik/traefik-helm-chart)
- [kube-vip](https://kube-vip.io/)

---

## License

Apache-2.0