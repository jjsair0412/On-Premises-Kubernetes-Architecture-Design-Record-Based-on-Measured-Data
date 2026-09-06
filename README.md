# On-Premises K8s Cluster Best Architecture

## OverView
해당 레포지토리는 온프레미스 환경에 적합한 K8s Cluster를 구성하기 위해 참고할 정보와, 솔루션 별 산재된 Best Practice를 모아 아키텍처로 설계한 내용을 작성하였습니다.
> **English :** This repository documents an architecture designed by collecting reference information and solution-specific best practices for building a K8s cluster suitable for on-premises environments.

Ansible 기반으로 코드화 하였으며, PR, Issue, Fork 모두 허용하며 수정요청또한 고마운 마음으로 받도록 하겠습니다.
> **English :** It is codified with Ansible. Pull requests, issues, and forks are all welcome, and suggestions for improvements are also greatly appreciated.

## Environment
|구분(Category)|솔루션(Solution)|버전(Version)|비고(Notes)|
|--|--|--|--|
|K8s|RKE2|v1.35.x|Treafik 공식 지원(Officially supported by Traefik)|
|CNI|Cilium|1.19||
|CI|Jenkins|||
|CD|ArgoCD|||
|Authenticate / Authorization |KeyCloak|||
|Monitoring|Grafana|||
|Logging|Prometheus|||

