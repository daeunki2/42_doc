# Part 3: K3d, Argo CD and GitOps

Part 2까지는 `kubectl`이나 Kubernetes 설정 파일을 사용해 Cluster에 Application을 배포했습니다. Part 3에서는 배포의 기준을 개발자의 명령어에서 **Git Repository**로 옮깁니다.

핵심 질문은 다음과 같습니다.

> Application의 원하는 상태를 Git에 기록하고, Cluster가 그 상태를 자동으로 따라가게 만들 수 있는가?

## 1. Part 2와 무엇이 다른가

Part 2의 흐름은 사람이 Cluster에 직접 명령하는 방식입니다.

```text
Developer
		|
		+-- kubectl apply
		v
Kubernetes Cluster
```

Part 3에서는 Git에 설정을 Push하고 Argo CD가 Cluster에 반영합니다.

```text
Developer
		|
		+-- git push
		v
Git Repository
		|
		v
Argo CD
		|
		v
Kubernetes Cluster
```

이 차이를 이해하는 것이 Part 3의 중심입니다. Argo CD는 단순히 YAML을 실행하는 도구가 아니라, Git의 원하는 상태와 Cluster의 실제 상태를 계속 비교하는 시스템입니다.

## 2. K3s와 K3d 비교

Part 1과 Part 2에서는 K3s를 사용했고, Part 3에서는 K3d를 사용합니다.

| 항목 | K3s | K3d |
|---|---|---|
| 정체 | 경량 Kubernetes Distribution | Docker 안에서 K3s Cluster를 실행하는 도구 |
| 실행 환경 | VM이나 일반 Linux Host | Docker Container 기반 |
| 주요 역할 | Kubernetes Server/Agent 제공 | 개발·실습용 K3s Cluster 생성 및 관리 |
| 과제에서의 위치 | Part 1, Part 2 | Part 3 |

K3d는 Kubernetes 그 자체가 아닙니다. K3s를 Docker Container 안에서 쉽게 실행하고 관리하기 위한 Wrapper에 가깝습니다.

```text
Docker
	|
	v
K3d
	|
	v
K3s Server/Agent Container
	|
	v
Kubernetes Cluster
```

따라서 Part 3에서 Docker가 필요한 이유는 K3d가 K3s Node를 Container로 실행하기 때문입니다.

## 3. Namespace

하나의 Kubernetes Cluster 안에서 리소스를 논리적으로 분리하려면 Namespace를 사용합니다.

과제에서는 적어도 다음 두 영역을 구분합니다.

```text
Kubernetes Cluster
		|
		+-- argocd Namespace
		|     +-- Argo CD Components
		|
		+-- dev Namespace
					+-- Application
```

`argocd`는 배포를 관리하는 Argo CD가 위치하는 공간이고, `dev`는 Argo CD가 배포할 Application의 공간입니다.

Namespace를 사용한다고 Cluster가 두 개로 나뉘는 것은 아닙니다. 같은 Cluster 안의 리소스를 이름과 권한 관점에서 구분하는 것입니다.

## 4. Argo CD가 관리하는 두 상태

Argo CD는 다음 두 상태를 비교합니다.

```text
Desired State
Git Repository에 기록된 Kubernetes 설정

Actual State
현재 Kubernetes Cluster에서 실행 중인 리소스
```

두 상태가 같으면 `Synced`, 다르면 `OutOfSync` 상태가 됩니다.

```text
Git:     image v1
Cluster: image v1
				 ↓
			 Synced
```

Git의 Image Tag를 `v1`에서 `v2`로 바꾸면 상태가 달라집니다.

```text
Git:     image v2
Cluster: image v1
				 ↓
			 OutOfSync
				 ↓
Argo CD가 변경을 감지하고 Sync
				 ↓
Cluster: image v2
```

## 5. GitOps란 무엇인가

GitOps는 Git을 시스템의 원하는 상태를 기록하는 **Source of Truth**로 사용하는 운영 방식입니다.

일반적인 수동 배포에서는 Cluster가 실제 상태의 기준이 되기 쉽습니다.

```text
사람 -> kubectl -> Cluster
```

GitOps에서는 다음 흐름이 기준이 됩니다.

```text
사람 -> Git 변경 -> Argo CD 감지 -> Cluster 동기화
```

따라서 변경 이력이 Git commit으로 남고, 누가 언제 어떤 배포 상태를 요청했는지 확인할 수 있습니다.

## 6. Application 정의

Argo CD에서 관리할 Application은 보통 다음 정보를 가집니다.

```text
Source
	- 어떤 Git Repository인가?
	- 어떤 경로의 Manifest를 읽는가?

Destination
	- 어느 Kubernetes Cluster인가?
	- 어느 Namespace에 배포하는가?

Sync Policy
	- 자동으로 동기화할 것인가?
	- 수동으로 동기화할 것인가?
```

이 정보가 연결되어야 Argo CD가 “어디의 설정을 어디에 반영할지” 알 수 있습니다.

## 7. Image Tag 변경을 배포로 연결하기

과제에서 중요한 검증은 Git의 변경이 실제 Application 업데이트로 이어지는지 확인하는 것입니다.

```text
1. Git Repository의 Manifest 확인
2. Image Tag를 v1에서 v2로 변경
3. Commit과 Push
4. Argo CD가 Repository 변경을 확인
5. Application이 OutOfSync가 됨
6. Sync가 수행됨
7. Deployment가 새 Image를 사용
8. 새로운 Pod가 생성됨
9. 실제 응답이 v2로 바뀜
```

여기서 Git 파일만 바뀌고 Cluster의 Pod가 바뀌지 않는다면, Repository 연결·Sync Policy·Image 존재 여부·Pod 상태를 각각 확인해야 합니다.

## 8. 자동 동기화와 수동 동기화

Argo CD는 변경을 감지한 뒤 자동으로 Sync할 수도 있고, 사용자가 승인한 뒤 수동으로 Sync할 수도 있습니다.

```text
자동 Sync
Git 변경 -> 감지 -> 즉시 Cluster 반영

수동 Sync
Git 변경 -> OutOfSync 표시 -> 사람이 Sync 실행 -> 반영
```

과제에서 요구하는 방식이 무엇인지 확인해야 하며, 중요한 것은 “Argo CD가 Git 변경을 인식하고 Application 상태를 바꿀 수 있는가”를 검증하는 것입니다.

## 9. 문제를 계층별로 좁히기

```text
Git Repository에 Manifest가 존재하는가?
				↓
Argo CD Application이 Repository를 바라보는가?
				↓
Application의 Source Path가 맞는가?
				↓
Destination Namespace가 맞는가?
				↓
Application이 Synced 상태인가?
				↓
Deployment가 새 Image를 사용했는가?
				↓
새 Pod가 Ready 상태인가?
```

Argo CD 화면에 Application이 보인다는 것만으로 배포가 끝난 것은 아닙니다. Sync 상태, Health 상태, 실제 Pod와 Application 응답까지 확인해야 합니다.

## 10. 검증 질문

1. K3d는 K3s와 어떤 관계인가?
2. K3d가 실행되기 위해 Docker가 필요한 이유는 무엇인가?
3. `argocd`와 `dev` Namespace를 분리하는 이유는 무엇인가?
4. Argo CD의 Desired State와 Actual State는 각각 어디에 존재하는가?
5. Git이 Source of Truth라는 것은 실제 운영 흐름에서 무엇을 의미하는가?
6. Image Tag를 바꾼 뒤 Application이 업데이트되기까지 어떤 단계가 필요한가?
7. Argo CD가 Synced인데 Application이 동작하지 않는다면 무엇을 추가로 확인해야 하는가?
