# Bonus: Local GitLab

Bonus는 Part 3에서 사용한 외부 Git Repository를 Local GitLab으로 바꾸는 과제입니다.

핵심은 새로운 배포 시스템을 만드는 것이 아니라, **Git Repository의 위치만 바꾸어도 기존 GitOps 흐름이 유지되는지** 확인하는 것입니다.

## 1. Mandatory와의 관계

Part 3의 기본 구조는 다음과 같습니다.

```text
GitHub
	|
	v
Argo CD
	|
	v
Kubernetes
```

Bonus에서는 GitHub가 Local GitLab으로 바뀝니다.

```text
Local GitLab
	|
	v
Argo CD
	|
	v
Kubernetes
```

Argo CD와 Kubernetes의 역할은 그대로입니다. 달라지는 것은 Argo CD가 Manifest를 가져오는 Git Server가 직접 운영하는 Local GitLab이라는 점입니다.

## 2. GitLab을 Cluster 안에서 실행하기

Bonus의 요구사항에 따라 GitLab은 `gitlab` Namespace에서 실행됩니다.

```text
Kubernetes Cluster
		|
		+-- gitlab Namespace
		|     +-- GitLab Application
		|
		+-- argocd Namespace
		|     +-- Argo CD
		|
		+-- dev Namespace
					+-- Target Application
```

이 구조는 GitLab 자체도 Kubernetes에서 실행되는 Application이라는 점을 보여줍니다. Git Repository를 제공하는 시스템과 그 Repository를 읽는 Argo CD가 같은 Local Infrastructure 안에 있습니다.

## 3. Helm을 사용하는 이유

GitLab처럼 구성요소가 많은 Application은 Deployment, Service, Storage, 설정값 등을 직접 하나씩 작성하면 관리해야 할 설정이 많아집니다.

Helm은 Kubernetes Manifest를 Template과 Values로 묶어 복잡한 Application을 설치하고 설정하는 도구입니다.

```text
Helm Chart
	+-- Template
	+-- Values
	+-- 설치/업데이트 규칙
				|
				v
Kubernetes Resources
```

Bonus에서 Helm을 사용할 수 있다는 것은 필수라는 뜻이 아니라, 복잡한 GitLab 설치를 관리하기 위한 선택지가 있다는 의미입니다.

## 4. Bonus에서 유지되어야 하는 흐름

GitLab을 설치했다고 Bonus가 끝나는 것이 아닙니다. Part 3에서 검증했던 GitOps 흐름이 Local GitLab에서도 동작해야 합니다.

```text
1. Local GitLab에 Repository 생성
2. Kubernetes Manifest Push
3. Argo CD가 Local GitLab Repository를 Source로 사용
4. Argo CD Application 생성
5. dev Namespace에 Application 배포
6. Git의 Image Tag 변경
7. Argo CD가 변경을 감지
8. Kubernetes Application 업데이트
```

외부 GitHub를 Local GitLab으로 바꾼 뒤에도 이 흐름이 유지되어야 합니다.

## 5. 네트워크에서 확인할 것

Argo CD가 GitLab Repository에 접근하려면 URL과 인증 정보가 올바르고, Cluster 내부에서 GitLab 주소에 도달할 수 있어야 합니다.

```text
Argo CD
	 |
	 +-- GitLab Service 주소로 접근 가능한가?
	 +-- Repository URL이 정확한가?
	 +-- 인증 정보가 올바른가?
	 +-- 필요한 Port가 열려 있는가?
	 v
Local GitLab Repository
```

GitLab Web UI가 브라우저에서 열리는 것과 Argo CD가 Repository를 읽을 수 있는 것은 별개의 검증입니다.

## 6. Bonus를 시작하기 전 조건

Bonus는 Mandatory Part 1, Part 2, Part 3가 정상적으로 동작한 뒤 진행하는 것이 좋습니다.

```text
Part 1: Cluster 구성 성공
				↓
Part 2: Application 배포 성공
				↓
Part 3: GitHub + Argo CD 동기화 성공
				↓
Bonus: GitHub를 Local GitLab으로 교체
```

기본 GitOps 흐름이 불안정한 상태에서 GitLab 설치까지 시작하면, 문제가 GitLab인지 Argo CD인지 Kubernetes인지 구분하기 어려워집니다.

## 7. 검증 체크리스트

- `gitlab` Namespace가 존재하는가?
- GitLab Pod가 Ready 상태인가?
- GitLab Repository를 생성하고 Push할 수 있는가?
- Cluster 내부에서 GitLab Service에 접근할 수 있는가?
- Argo CD가 Local GitLab Repository를 Source로 등록했는가?
- Argo CD Application이 `dev` Namespace를 Destination으로 사용하는가?
- Git의 변경이 실제 Application 업데이트로 이어지는가?
- Part 3에서 사용한 기능이 Local GitLab에서도 모두 유지되는가?

## Bonus의 핵심 질문

1. GitLab은 이번 구조에서 어떤 역할을 하는가?
2. GitLab과 Argo CD는 각각 무엇을 담당하는가?
3. GitLab이 `gitlab` Namespace에 있다는 것은 어떤 의미인가?
4. GitLab Web UI 접속과 Argo CD의 Repository 접근은 왜 따로 확인해야 하는가?
5. GitHub를 GitLab으로 바꾸어도 GitOps의 핵심 개념이 변하지 않는 이유는 무엇인가?
