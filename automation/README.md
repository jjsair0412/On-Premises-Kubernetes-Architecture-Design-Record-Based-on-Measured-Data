# automation
자동화는 GitOps를 따르며, 아래 기준으로 작성합니다.
> **English :** The automation follows GitOps principles and is structured based on the criteria below.

## 1. Infra
>**Stack :** Jenkins + Ansible

## ***Benefit List***
### 1.1 자동화와 멱등성 (Automation and Idempotency)
- sysctl 등의 노드 구성 혹은 신규노드 추가 시, 구축단계에 들어갈 공수를 자동화할 수 있습니다.

    > **English :** Automates the effort required during the setup phase, such as node configuration (e.g., sysctl) or adding new nodes.
- 노드의 상태를 유지하면서, 특정 Config 등의 구성을 변경할 수 있습니다. (ex: RKE2 Node Label 변경 등)

    > **English :** Allows updating specific configurations while maintaining node state (e.g., changing RKE2 Node Labels).

### 1.2 Ansible Apply는 Pipeline을 통해 (Ansible Apply via Pipeline)

- Jenkins Pipeline에 Ansible Apply 단계를 포함함으로써 실패 시 로그를 자동 저장하여 원인 파악에 용이합니다.

    > **English :** Including the Ansible Apply step in the Jenkins Pipeline automatically saves logs upon failure, making root-cause analysis easier.
- 유저 계정 별 파이프라인 권한을 차등으로 부여하여 무단 변경을 방지합니다.

    > **English :** Prevents unauthorized changes by granting differential pipeline permissions based on user accounts.

## 2. Application
>**Stack :** ArgoCD + Helm

## ***Benefit List***
### 2.1 ArgoCD의 상태관리 유연성 (State Management Flexibility in ArgoCD)

- Sycn 된 K8s 리소스들의 변화를 예측할 수 있습니다.

    또한 배포된 리소스 상태를 ArgoCD 상에서 수정하여 실제 Helm 변경 전 상태를 점검해 볼 수 있습니다.
    개발자 혹은 타 엔지니어가, 특정 K8s 리소스들의 상태를 자유롭게 변화시킬 수 있습니다.

    이는 RBAC과 직접 연관성이 있습니다. 만약 권한이 없는 사용자가 특정 리소스를 수정하려 한다면, ArgoCD 상에서 제어가 가능합니다.

    Helm을 관리하는 Git과, ArgoCD 별로 RBAC을 관리함으로써 권한관리의 유연성을 확보할 수 있습니다. 
    
    > **English :** Predicts changes in synced K8s resources.
    >
    >Additionally, resource states can be modified directly within ArgoCD to inspect conditions prior to actual Helm changes.
    >Developers or other engineers can freely adjust the state of specific K8s resources.
    >
    >This directly connects to RBAC. If an unauthorized user attempts to modify a specific resource, access can be controlled through ArgoCD.
    >
    >Flexibility in access control is achieved by managing RBAC separately for ArgoCD and the Git repository that manages Helm.

### 2.2 변경 추적 (Change Tracking)

- Kubectl 혹은 Helm Command를 이용해 직접 Kube-API Server로 요청을 최소화시킴으로써, 감사 추적에 용이합니다. 
    
    또한 문제 발생 시 롤백이 단순하고 빠르게 이루어질 수 있습니다. (직접 수정 혹은 강제 Sync)

    > **English :** Minimizes direct requests to the Kube-API Server via Kubectl or Helm commands, improving audit traceability.
    >
    >In addition, rollbacks can be performed quickly and simply when issues occur (via direct modification or forced Sync).

## 3. Helm Pattern
**Apps Of Apps Pattern**, **Rendered Manifests Pattern** 두가지를 사용합니다.

> **English :** Uses both the Apps of Apps Pattern and the Rendered Manifests Pattern.

### 3.1 Apps Of Apps Pattern
    
- 배포 대상이 될 여러 오픈소스 솔루션패키지를 묶어 관리합니다.

    Cluster 별로 배포된 Application이 어떤 상태를 가지고있는지 한번에 파악할 수 있으며, 수많은 value 파일을 묶어 관리할 수 있습니다.

    > **English :** Bundles and manages multiple open-source solution packages to be deployed.
    >
    >Provides a single-pane-of-glass view into the status of deployed applications across clusters and enables bundled management of numerous value files.

### 3.2 Rendered Manifests Pattern

- Application이 배포되기 전, 실제 Helm 구조를 Template 화 하여 배포 이후를 예측할 수 있습니다.

    템플릿 로직 (if, Range 등)에 의해 실제 배포상태를 예측하기 어려운 구조를 탈피할 수 있습니다.

    > **English :**Templates the actual Helm structure before applications are deployed, allowing clear prediction of the post-deployment state.
    >
    >Eliminates complex template logic (e.g., if, range) that makes predicting actual deployment states difficult.
