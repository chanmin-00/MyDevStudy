# 1. 배포 플로우

Source Build는 Naver Cloud Platform의 Source Commit 저장소에 있는 소스 코드를 기반으로, Dockerfile을 사용해 애플리케이션 이미지를 빌드한다. 빌드가 완료된 이미지는 Container Registry로 자동 푸시되어 배포에 사용할 수 있는 상태가 된다.

Source Deploy는 Container Registry에 업로드된 최신 이미지를 참조하여, 지정된 manifest 경로의 Kubernetes 설정 파일을 클러스터에 apply한다. 이를 통해 쿠버네티스 클러스터 내에서 새로운 버전의 애플리케이션이 자동으로 배포된다.

Source Pipeline은 이러한 Source Build → Source Deploy 과정을 하나의 파이프라인으로 연결해준다.

즉, 코드 변경이 발생하면 자동으로 빌드, 이미지 푸시, 그리고 쿠버네티스 배포까지 순차적으로 실행되는 **CI/CD 자동화 프로세스**를 구성할 수 있다.

<img width="592" height="138" alt="image" src="https://github.com/user-attachments/assets/205b36fb-fe40-4964-8e11-02fa06d965a8" />

# 2. 소스코드 동기화

배포를 진행하기 전에, GitHub에 있는 소스코드를 Naver Cloud Platform의 Source Commit 저장소와 동기화해야 한다. 이는 Source Deploy가 배포 시 Source Commit 저장소의 manifest 경로를 기준으로 Kubernetes에 apply 명령을 수행하기 때문이다.

즉, Source Deploy는 GitHub 저장소가 아닌 Source Commit 저장소와 직접 연동된다.

먼저 로컬 저장소에서 NCP Source Commit 저장소를 새로운 원격 저장소로 추가한다.

```bash
git remote add sourcecommit 네이버클라우드 리포지터리주소
```

그 후, 아래 명령어를 통해 develop 브랜치를 NCP 리포지터리로 푸시한다.

```bash
git push sourcecommit develop --force
```

<img width="635" height="99" alt="image" src="https://github.com/user-attachments/assets/aaa0e3ee-7f32-48e7-bea3-d2a48be73b22" />

NCP Source Commit 콘솔에서는 **외부 리포지터리 복사 기능**을 제공한다. 이 기능을 이용하면 GitHub 저장소를 직접 연결하여 코드를 자동으로 복제할 수 있다.

<img width="628" height="239" alt="image" src="https://github.com/user-attachments/assets/2efc8da2-b50d-4774-9dcb-def7db0e58ad" />

정상적으로 동기화가 완료되면, Source Commit 저장소 내에서 GitHub 코드가 그대로 반영된 것을 확인할 수 있다. 이제 해당 저장소를 기반으로 Source Build와 Source Deploy 파이프라인을 구성할 수 있다.

# 3. Source Build

Source Build는 소스 코드를 기반으로 Docker 이미지를 생성하고, 이를 Container Registry에 저장하는 역할을 한다.

이후 Source Deploy 단계에서 해당 이미지를 사용해 애플리케이션을 쿠버네티스 클러스터에 배포한다.

### 저장소 연결

<img width="463" height="407" alt="image" src="https://github.com/user-attachments/assets/9c33efc3-8328-41bb-aaf7-0bc71f5b9393" />

먼저 빌드 프로젝트를 생성하고, 빌드 대상으로 사용할 저장소와 브랜치를 연결한다. 저장소는 Naver Cloud Platform에서 제공하는 Source Commit, GitHub, Bitbucket 중 선택할 수 있다.

### 빌드 환경 설정

다음으로 빌드가 실행될 환경 정보를 지정한다. 빌드 환경에는 실행 OS, 런타임 버전, 추가 패키지 등을 설정할 수 있으며, 이번에는 Docker 이미지를 빌드할 것이므로 ‘도커 이미지 빌드’ 옵션을 활성화한다.

<img width="515" height="314" alt="image" src="https://github.com/user-attachments/assets/491863c7-744c-4715-a41a-a479fedaeaa0" />

### 빌드 명령어 설정

Source Build에서는 다음과 같은 순서로 명령어가 실행된다:

1. **빌드 전 명령어**
2. **빌드 명령어**
3. **도커 이미지 빌드**
4. **빌드 후 명령어**

즉, 도커 빌드는 빌드 명령어 단계와 함께 수행되며, 전체 프로세스는 위 순서대로 진행된다.

<img width="538" height="598" alt="image" src="https://github.com/user-attachments/assets/0aed8d62-5e38-4e8c-b171-1e5421391604" />

설정이 완료되면 빌드를 실행한다. 빌드가 정상적으로 완료되면, 생성된 Docker 이미지가 Container Registry에 저장된 것을 확인할 수 있다.

<img width="494" height="360" alt="image" src="https://github.com/user-attachments/assets/2c45a156-8db7-4cf2-954c-20b57531598a" />

# 4. Source Deploy

Source Deploy는 실제 배포를 수행하는 단계로, 여기서는 Kubernetes(K8s)를 대상으로 배포를 진행한다. 

기본적으로 Source Deploy Agent가 설치된 서버나 Auto Scaling Group에도 배포가 가능하지만, 이번 구성에서는 Kubernetes 클러스터를 대상으로 한다.

배포 대상이 Kubernetes인 경우, NCP의 Source Deploy 대신 ArgoCD를 사용하는 것도 권장된다. 이유는 Source Deploy가 NCP에 종속적인 기능인 반면, ArgoCD는 오픈소스이기 때문에 추후 클라우드 이전 등의 상황에서도 유연하게 대응할 수 있다

### 배포 프로젝트 설정

먼저 배포 프로젝트 이름을 지정한다. 이 프로젝트는 이후 여러 배포 스테이지(dev, test, real 등)를 관리하는 단위가 된다.

<img width="524" height="287" alt="image" src="https://github.com/user-attachments/assets/ff3f0765-18f4-42b6-889a-2df0989d4b8c" />

### 배포 환경 설정

NCP의 기본 배포 스테이지(Stage)는 dev, test, real로 구성되어 있으며, 필요에 따라 ‘+’ 버튼을 눌러 새로운 스테이지를 생성할 수 있다.

<img width="462" height="246" alt="image" src="https://github.com/user-attachments/assets/92a9096b-ab4c-4ef2-83a4-a11cd2005cab" />

사용하지 않을 스테이지는 “설정 안함”으로 비활성화할 수 있다. 본 예시에서는 dev, test 두 가지 스테이지만 사용한다.

Source Deploy는 Kubernetes뿐 아니라 Deploy Agent가 설치된 일반 서버 또는 Auto Scaling Group에도 배포할 수 있다. 이번에는 Kubernetes 서비스를 선택하고, 배포할 클러스터를 지정하여 진행한다.

### **배포 명령어 설정: 스테이지별 시나리오**

Source Deploy에서는 각 스테이지마다 배포 시나리오를 별도로 설정해야 한다. 스테이지 옆의 체크박스를 선택하면 하단에 시나리오 설정 화면이 나타난다.

<img width="545" height="251" alt="image" src="https://github.com/user-attachments/assets/2184cad3-44f8-4319-802b-427fd779b9ba" />

배포 시나리오 생성 버튼을 누르면 아래와 같은 상세 설정 창이 열린다.

<img width="452" height="150" alt="image" src="https://github.com/user-attachments/assets/d97da5af-3219-4112-863b-c146133d79b3" />

### **배포 시나리오 생성 - 매니페스트 설정**

여기서 Kubernetes에 apply할 매니페스트 파일 경로를 지정한다. 이때 Source Commit 저장소에 존재하는 레포지토리 목록만 선택할 수 있다.

<img width="547" height="261" alt="image" src="https://github.com/user-attachments/assets/4261f370-8d0c-461b-b10b-95449148a077" />

### **배포 시나리오 생성 - 배포 전략 설정**

배포 전략은 다음 중 하나를 선택할 수 있다.

- Rolling Update
- Blue/Green
- Canary

이번 예시에서는 Rolling Update를 사용한다.

<img width="603" height="167" alt="image" src="https://github.com/user-attachments/assets/d1446131-3ec7-4420-9e46-3a53606e7d43" />

### 배포 수행

프로젝트 목록에서 배포 프로젝트 이름을 클릭하면, 생성된 시나리오 목록을 확인할 수 있다. 각 시나리오를 선택한 뒤 ‘배포 시작하기’ 버튼을 클릭하면 배포가 진행된다. test 스테이지 역시 dev와 동일한 방식으로 시나리오를 생성한다.

<img width="568" height="422" alt="image" src="https://github.com/user-attachments/assets/5fa047cf-2ba0-4001-92e4-da1682fc499c" />

배포를 수행하면, 쿠버네티스 클러스터 내에서 Pods가 새롭게 생성된 것을 확인할 수 있다. 서비스 타입이 LoadBalancer로 설정되어 있다면, 자동으로 새로운 LoadBalancer 리소스도 함께 생성된다.

<img width="607" height="117" alt="image" src="https://github.com/user-attachments/assets/34ab6ac4-ff7a-4b4e-808a-4a2901ef88f5" />

# 5. Source PipeLine

Source Pipeline은 앞서 생성한 Source Build와 Source Deploy를 하나의 파이프라인으로 연결하여 코드 커밋 → 빌드 → 배포 과정을 자동으로 수행하도록 구성하는 기능이다.

<img width="564" height="144" alt="image" src="https://github.com/user-attachments/assets/2f7e8a4c-a06e-41e9-a5ac-d0cfe7917165" />

### 기본 설정

파이프라인을 생성한 후, 이름과 설명 등을 입력한다. 이 단계에서는 파이프라인의 전체 흐름을 정의할 준비를 한다.

<img width="564" height="211" alt="image" src="https://github.com/user-attachments/assets/9fe03186-f78b-4a72-b8db-101799f424c1" />

### 작업 추가

이제 앞에서 만든 Source Build와 Source Deploy를 파이프라인에 추가한다. 작업 추가 버튼을 클릭하면 기존에 생성한 빌드 작업과 배포 작업을 가져올 수 있다.

<img width="554" height="216" alt="image" src="https://github.com/user-attachments/assets/33450837-da36-4278-b99f-fb3b41d2788b" />

1. **source build**
    
    빌드 단계에서는 이전에 구성한 Source Build 프로젝트를 선택한다. 이 단계에서 빌드할 브랜치를 지정할 수도 있다.
    

<img width="578" height="539" alt="image" src="https://github.com/user-attachments/assets/d8c1cae5-ae8e-44ed-98b2-22a2aeb03477" />

1. **source deploy**
    
    Source Deploy 작업을 추가할 때는 먼저 프로젝트를 선택한 후, 해당 프로젝트 내의 스테이지와 시나리오를 지정한다.
    
<img width="463" height="608" alt="image" src="https://github.com/user-attachments/assets/52c7985a-1ed3-4711-bfcc-7f4913f34ad7" />


필요하다면 추가 스테이지와 시나리오를 더 연결해 다중 환경 배포 파이프라인을 구성할 수도 있다.

### 그래프 구성

좌측의 Source Build, Source Deploy 작업 항목에서 ‘+’ 버튼을 눌러 파이프라인의 실행 순서를 그래프 형태로 구성한다.

1. Source Build 작업의 플러스 버튼을 눌러 시작점을 만든다. 이 작업은 선행 작업이 없으므로 ‘선행 작업 없음’을 선택한다.
2. 이후 Source Deploy 작업은 빌드 작업 완료 후 실행되도록 설정한다. 여러 개의 작업을 병렬로 실행하는 것도 가능하다.

예를 들어, 작업1 완료 후 작업2와 작업3이 동시에 실행되게 할 수 있으며, 작업2와 작업3 모두 완료된 후 작업4가 실행되도록 구성할 수도 있다. 

즉, 선행 작업을 여러 개 지정함으로써 병렬 및 직렬 실행 구조를 자유롭게 만들 수 있다.

<img width="537" height="170" alt="image" src="https://github.com/user-attachments/assets/1c4b6c3f-d542-4af9-8642-1b6155b5b69e" />

## 트리거 설정

파이프라인의 기본 트리거는 수동 실행이지만, NCP에서는 다음과 같은 자동 트리거 옵션 3가지를 제공한다.

예를 들어, GitHub 또는 Source Commit의 특정 브랜치에 코드가 푸시되면 자동으로 빌드와 배포가 진행되도록 설정할 수 있다.

<img width="477" height="408" alt="image" src="https://github.com/user-attachments/assets/886d9a62-8a50-4f1e-a58d-cae1caadffc5" />

# 멀티 환경 파이프라인 구성

운영 환경(dev → staging → production)별로 파이프라인을 분리해 구성하는 것이 좋다.

즉, dev용 파이프라인, staging용 파이프라인, production용 파이프라인을 각각 만들어 관리하면 안정적인 배포 프로세스를 유지할 수 있다.

이번 예시에서는 하나의 모듈 단위 파이프라인을 구성했지만, 멀티모듈 구조라면 모듈별로 독립된 파이프라인을 만들어 전체 프로젝트의 CI/CD 파이프라인을 체계적으로 관리할 수 있다

# 결론

이번 과정에서는 Naver Cloud Platform(NCP) 환경에서 Kubernetes 기반 CI/CD 파이프라인을 구축하며, Source Commit으로 소스 코드를 관리하고 Source Build를 통해 Docker 이미지를 생성, Container Registry에 저장한 뒤, Source Deploy를 통해 해당 이미지를 Kubernetes 클러스터에 자동 배포하는 전 과정을 완성했다. 

이를 Source Pipeline으로 연결함으로써 코드 푸시 한 번으로 빌드 → 이미지 업로드 → 배포 → Pod 재생성 및 서비스 갱신이 자동화되는 클라우드 네이티브 CI/CD 시스템을 구현할 수 있었다.
