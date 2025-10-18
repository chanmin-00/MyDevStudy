[Workshop Studio](https://catalog.us-east-1.prod.workshops.aws/workshops/9c0aa9ab-90a9-44a6-abe1-8dff360ae428/ko-KR)

Workshop Studio에서 진행했던 실습을 복습한 내용입니다.

# **Amazon EKS로 웹 애플리케이션 구축하기**

### **프로젝트 배경**

국내 한 유명 기업의 DevOps 팀에서 일하는 A씨는 새로운 웹 애플리케이션을 개발하는 프로젝트를 맡는다. 해당 애플리케이션은 다음과 같은 조건을 충족해야 한다.

- 새로운 요청이 들어왔을 때 변경 사항을 빠르게 반영할 수 있어야 한다.
- 확장이 용이해야 한다.
- 소수 인원으로 운영과 개발을 병행할 수 있어야 한다.

이러한 요구사항을 바탕으로 A씨는 모던 애플리케이션 아키텍처를 구축하기로 한다. 팀원들과 논의한 끝에 MSA와 컨테이너 기반으로 서비스를 설계하고, 오케스트레이션 도구로는 Kubernetes를 선택한다.

직접 쿠버네티스를 구축하려면 설치, 구성, 인프라 확장 등 많은 리소스가 필요하다. 그러나 팀 인원이 많지 않고, 쿠버네티스를 잘 모르는 구성원도 있어 모든 과정을 오픈소스로 직접 관리하는 것은 부담이 된다.

이때 A씨는 AWS에서 제공하는 관리형 쿠버네티스 서비스인 **Amazon EKS (Elastic Kubernetes Service)** 를 알게 된다. EKS를 사용하면 클러스터 운영의 복잡도를 줄이고, 애플리케이션 배포에 집중할 수 있다. 

# **Kubernetes (k8s) 란?**

쿠버네티스는 컨테이너화된 워크로드와 서비스를 관리하기 위한 이식성 있고 확장 가능한 오픈소스 플랫폼이다. 쿠버네티스는 선언적 구성과 자동화를 모두 지원하여, **컨테이너 환경을 효율적으로 운영할 수 있게 해주는 컨테이너 오케스트레이션 도구**이다.

컨테이너가 수십 개, 수백 개로 늘어나면 도커 단독으로는 관리가 매우 어려워진다. 이때 쿠버네티스는 여러 컨테이너를 자동으로 배치하고, 상태를 관리하며, 확장, 복구 작업까지 수행한다.

<img width="524" height="274" alt="image" src="https://github.com/user-attachments/assets/73b9cc5f-6cfa-4a7d-8812-04bd3e2c10a8" />

쿠버네티스를 배포하면 클러스터를 얻게 된다. 클러스터는 여러 개의 노드로 구성되어 있으며, 이 노드들은 크게 컨트롤 플레인(Control Plane) 과 데이터 플레인(Data Plane) 으로 구분된다.

- 컨트롤 플레인은 워커 노드와 클러스터 내의 파드를 관리하고 제어한다.
- 데이터 플레인은 실제 워커 노드들로 구성되며, 컨테이너화된 애플리케이션의 구성 요소인 파드를 호스팅한다.

또한, 쿠버네티스는 필수 요소 외에도 애드온 형태의 컴포넌트를 추가로 사용할 수 있다. 이 애드온은 클러스터의 모니터링, 네트워크 관리, 로깅 등 기능을 확장할 때 활용된다.

### **Kubernetes Objects**

**쿠버네티스 오브젝트는 바라는 상태를 정의한 레코드이다**. 사용자가 오브젝트를 생성하면, 컨트롤 플레인이 이를 지속적으로 감시하며 현재 상태가 바라는 상태와 일치하도록 자동으로 조정한다.

주요 오브젝트에는 다음이 포함된다.

- `Pod` : 컨테이너가 실제로 실행되는 최소 단위
- `Service` : 네트워크 트래픽을 Pod로 연결해주는 역할
- `Deployment` : 애플리케이션 배포와 업데이트를 관리

이러한 구조 덕분에 쿠버네티스는 **컨테이너의 상태를 자동으로 조율하고 유지하는 오케스트레이션 시스템**으로 불린다.

# Amazon EKS

Amazon EKS는 Kubernetes를 손쉽게 실행할 수 있는 AWS의 관리형 서비스이다. Amazon EKS를 사용하면 AWS 환경에서 Kubernetes 컨트롤 플레인또는 노드를 직접 설치하거나 운영, 유지 관리할 필요가 없다.

<img width="409" height="295" alt="image" src="https://github.com/user-attachments/assets/b7f7ca38-6c4b-4d02-8562-720f28c43518" />

Amazon EKS는 여러 가용 영역에서 컨트롤 플레인 인스턴스를 실행하여 고가용성을 보장한다. 또한 비정상 컨트롤 플레인 인스턴스를 자동으로 감지하고 교체하며, 버전 업그레이드와 보안 패치를 자동으로 수행한다.

Amazon EKS는 다양한 AWS 서비스와 연동되어 애플리케이션의 확장성과 보안을 강화한다.

- 컨테이너 이미지 저장소: Amazon ECR
- 트래픽 로드 분산: AWS ELB
- 인증 및 권한 관리: AWS IAM
- 네트워크 격리: Amazon VPC

이러한 통합을 통해 개발자는 운영 부담을 줄이고, 안정적이고 확장 가능한 환경에서 애플리케이션을 실행할 수 있다.

Amazon EKS는 오픈소스 Kubernetes의 최신 버전을 그대로 실행한다. 따라서 Kubernetes 커뮤니티에서 사용되는 플러그인, 애드온, 툴을 그대로 사용할 수 있다.

EKS에서 실행되는 애플리케이션은 온프레미스 환경이든 퍼블릭 클라우드 환경이든 표준 Kubernetes 환경과 완벽하게 호환된다. 즉, 코드 수정 없이 기존 Kubernetes 애플리케이션을 EKS로 그대로 마이그레이션할 수 있다.

# 실습 환경 구축

<img width="737" height="297" alt="image" src="https://github.com/user-attachments/assets/3a70ef4f-cb2e-4420-811e-5b9742b76856" />

kubectl은 쿠버네티스 클러스터에 명령을 전달하는 CLI도구이다. 쿠버네티스는 오브젝트 생성, 수정, 삭제 등의 동작을 수행하기 위해 Kubernetes API를 사용한다.

**kubectl을 통해 사용자가 명령어를 입력하면, 해당 명령이 쿠버네티스 API를 호출하여 동작을 수행한다.**

배포할 Amazon EKS 버전에 맞는 kubectl을 설치한다. 이번 실습에서는 2025년 7월 기준 최신 버전을 설치한다.

```bash
sudo curl -o /usr/local/bin/kubectl  \
   https://s3.us-west-2.amazonaws.com/amazon-eks/1.33.0/2025-05-01/bin/linux/amd64/kubectl
```

```bash
sudo chmod +x /usr/local/bin/kubectl
```

```bash
kubectl version --client=true
```

위 명령을 실행하여 설치된 kubectl의 버전을 확인한다. 이로써 Amazon EKS 환경에서 클러스터를 제어하기 위한 kubectl 설치 과정이 완료된다.

# eksctl 설치하기

Amazon EKS 클러스터를 배포하는 방법에는 여러 가지가 있다. 대표적으로 AWS Management Console, CloudFormation, AWS CDK, eksctl, Terraform 등을 사용할 수 있다.

**이 중 eksctl 은 EKS 클러스터를 손쉽게 생성하고 관리할 수 있는 CLI 도구이다.** Go 언어로 작성되어 있으며, 내부적으로 CloudFormation 템플릿을 사용해 클러스터를 배포한다.

아래 명령어를 실행하여 최신 eksctl 바이너리를 다운로드한다.

```bash
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
```

다운로드된 바이너리를 /usr/local/bin 디렉터리로 이동한다.

```bash
sudo mv -v /tmp/eksctl /usr/local/bin
```

설치가 완료되면 아래 명령어를 통해 버전 정보를 확인한다.

```bash
eksctl version
```

# 도커 컨테이너 이미지 만들기

Docker는 컨테이너화된 애플리케이션을 구축, 테스트, 배포할 수 있는 플랫폼이다. Docker는 애플리케이션을 컨테이너라는 표준화된 단위로 패키징하며, 이 안에는 코드, 런타임, 시스템 도구, 라이브러리 등 실행에 필요한 모든 요소가 포함된다.

**컨테이너 이미지는 컨테이너 실행에 필요한 파일과 설정 값을 묶은 것**이다. 이미지는 저장소에 업로드하거나 다운로드할 수 있으며, **이를 실행한 상태가 바로 컨테이너**이다.

이미지는 Amazon ECR Public Gallery나 Docker Hub 같은 공식 저장소에서 가져올 수도 있고, 직접 빌드할 수도 있다.

# 컨테이너 이미지 만들기

<img width="568" height="243" alt="image" src="https://github.com/user-attachments/assets/be28c435-0e05-4db5-ad5d-4098d0b7760f" />

### 1. Dockerfile 생성하기

Dockerfile은 컨테이너 이미지를 빌드하기 위한 설정 파일이다. 즉, **이미지의 구조와 구동 방식을 정의한 청사진**으로 볼 수 있다. **Dockerfile을 기반으로 생성된 이미지는 컨테이너가 되어 실제 애플리케이션이 구동**된다.

1. 루트 폴더(/home/ec2-user/environment)로 이동한다.
    
    ```bash
    cd ~/environment/
    ```
    
2. 아래 명령어를 입력하여 Dockerfile을 생성한다.
    
    ```bash
    cat << EOF > Dockerfile
    FROM nginx:latest
    RUN  echo '<h1> test nginx web page </h1>'  >> index.html
    RUN cp /index.html /usr/share/nginx/html
    EOF
    ```
    
    | 명령어 | 설명 |
    | --- | --- |
    | FROM | Base Image 지정(OS 및 버전 명시, Base Image에서 시작해서 커스텀 이미지를 추가) |
    | RUN | shell command를 해당 docker image에 실행시킬 때 사용함 |
    | WORKDIR | Docker File에 있는 RUN, CMD, ENTRYPOINT, COPY, ADD 등의 지시를 수행할 곳 |
    | EXPOSE | 호스트와 연결할 포트 번호를 지정 |
    | CMD | application을 실행하기 위한 명령어 |

### 2. 이미지 빌드 및 실행

1. docker build 명령어로 이미지를 빌드한다. 이름(name)은 이미지의 이름을, 태그(tag)는 버전을 의미한다. 태그를 지정하지 않으면 기본값은 latest이다.
    
    ```bash
    docker build -t test-image .
    ```
    
2. docker images 명령어로 빌드된 이미지를 확인한다.
    
    ```bash
    docker images
    ```
    
3. docker run 명령어로 이미지를 컨테이너로 실행한다.
    
    ```bash
    docker run -p 8080:80 --name test-nginx test-image
    ```
    
    위 명령은 호스트의 8080 포트를 컨테이너의 80 포트에 매핑한다. 즉, 브라우저에서 localhost:8080으로 접근하면 컨테이너 내부 Nginx 웹 페이지를 확인할 수 있다
    

### 3. 컨테이너 관리

1. 현재 실행 중인 컨테이너 목록을 확인한다.
    
    ```bash
    docker ps
    ```
    
2. 컨테이너 로그를 확인한다.
    
    ```bash
    docker logs -f test-nginx
    ```
    
3. 컨테이너 내부 쉘 환경에 접속한다.
    
    ```bash
    docker exec -it test-nginx /bin/bash
    ```
    
4. 실행 중인 컨테이너를 중지한다.
    
    ```bash
    docker stop test-nginx
    ```
    
    중지 후 docker ps를 실행하면 해당 컨테이너가 목록에서 사라진 것을 확인할 수 있다.
    
5. 정지된 컨테이너를 삭제한다.
    
    ```bash
    docker rm test-nginx
    ```
    
6. 컨테이너 이미지를 삭제한다.
    
    ```bash
    docker rmi test-image
    ```
    

# Amazon ECR에 이미지 올리기

Amazon ECR(Elastic Container Registry)에 리포지토리를 생성하고 컨테이너 이미지를 업로드하는 과정을 진행한다. **Amazon ECR은 Docker 컨테이너 레지스트리 서비스로, 컨테이너 이미지를 안전하고 확장 가능한 아키텍처에 호스팅**하여 애플리케이션을 안정적으로 배포할 수 있게 해준다.

Amazon ECR은 사용자가 직접 이미지 리포지토리를 구축하거나 관리할 필요 없이 완전관리형 이미지 저장소를 제공한다. 또한 AWS IAM을 통해 접근 권한을 제어하고, 이미지 취약점 스캔 기능을 활성화할 수 있다. 리포지토리는 프라이빗 혹은 퍼블릭으로 설정할 수 있다.

### 1. 소스 코드 다운로드

컨테이너화할 예제 소스 코드를 GitHub에서 클론한다.

```bash
git clone https://github.com/joozero/amazon-eks-flask.git
```

### 2. Amzon ECR 리포지터리 생성

AWS CLI를 이용해 리포지토리를 생성한다. 본 실습에서는 리포지토리 이름을 demo-flask-backend로 설정하고, 리전에는 EKS 클러스터를 배포할 리전 코드(예: ap-northeast-2)를 지정한다.

```bash
aws ecr create-repository \
--repository-name demo-flask-backend \
--image-scanning-configuration scanOnPush=true \
--region ${AWS_REGION}
```

명령 실행 후 리포지토리 정보가 출력되며, AWS ECR 콘솔에서도 생성된 리포지토리를 확인할 수 있다.

<img width="667" height="473" alt="image" src="https://github.com/user-attachments/assets/14416fac-cce4-4627-a94e-28076974fbce" />

### 3. ECR 인증 및 로그인

ECR에 이미지를 푸시하기 위해 인증 토큰을 발급받고 로그인한다. 사용자 이름은 AWS, 인증 URI는 ECR 레지스트리 주소를 사용한다.

```bash
aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin $ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com
```

### 4. Docker Image 빌드

앞서 클론한 소스 코드 폴더로 이동하여 Docker 이미지를 빌드한다.

```bash
cd ~/environment/amazon-eks-flask
docker build -t demo-flask-backend .
```

### **5. 이미지 태그 설정**

빌드한 이미지를 ECR 리포지토리에 푸시할 수 있도록 태그를 설정한다.

```bash
docker tag demo-flask-backend:latest $ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/demo-flask-backend:latest
```

### **6. 이미지 푸시**

아래 명령어를 실행하여 이미지를 ECR 리포지토리에 업로드한다.

```bash
docker push $ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/demo-flask-backend:latest
```

푸시가 완료되면 ECR 콘솔에서 업로드된 이미지를 확인할 수 있다.

<img width="737" height="331" alt="image" src="https://github.com/user-attachments/assets/cc82419a-5253-4cdf-8aa0-2b613453db1d" />

# EKS 클러스터 생성하기

Amazon EKS 클러스터는 다양한 방식으로 배포할 수 있다.

- AWS 콘솔에서 클릭으로 배포
- AWS CloudFormation 또는 AWS CDK 같은 IaC 도구로 배포
- EKS의 공식 CLI인 eksctl을 사용하여 배포
- Terraform, Pulumi, Rancher 등을 이용하여 배포

# eksctl로 클러스터 생성하기

eksctl create cluster 명령어를 설정 없이 실행하면 기본 파라미터로 클러스터가 생성된다. 그러나 실습에서는 **몇 가지 설정 값을 직접 지정하기 위해 구성 파일(ClusterConfig) 을 작성하여 배포**한다. 

이는 쿠버네티스 오브젝트를 생성할 때 CLI 명령어 대신 YAML 구성 파일로 정의하는 방식과 같다. 이 방식을 사용하면 사용자가 명시한 바라는 상태를 명확하게 관리할 수 있다.

### 1. 구성 파일 작성

루트 디렉터리(/home/ec2-user/environment)에서 아래 명령어를 입력한다.

```bash
cd ~/environment

cat << EOF > eks-demo-cluster.yaml
---
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: eks-demo
  region: ${AWS_REGION}
  version: "1.33"

vpc:
  cidr: "10.0.0.0/16"
  nat:
    gateway: HighlyAvailable

managedNodeGroups:
  - name: node-group
    instanceType: m5.large
    desiredCapacity: 3
    volumeSize: 20
    privateNetworking: true
    iam:
      withAddonPolicies:
        imageBuilder: true
        awsLoadBalancerController: true
        cloudWatch: true
        autoScaler: true
        ebs: true

cloudWatch:
  clusterLogging:
    enableTypes: ["*"]
EOF
```

구성 파일을 보면,

- iam.attachPolicyARNs를 통해 직접 정책(Policy)을 지정할 수 있고,
- iam.withAddonPolicies를 통해 Add-on 관련 권한을 부여할 수 있다.

EKS 클러스터가 배포된 후 EC2 콘솔에서 워커 노드의 IAM Role을 확인하면 자동으로 추가된 권한을 확인할 수 있다.

### 2. 클러스터 배포

아래 명령어를 입력해 클러스터를 배포한다.

```bash
eksctl create cluster -f eks-demo-cluster.yaml
```

클러스터 생성에는 약 15~20분 정도 소요된다. 터미널 창에서 진행 상황을 확인할 수 있으며, AWS CloudFormation 콘솔에서도 이벤트와 리소스 상태를 함께 확인할 수 있다.

<img width="727" height="300" alt="image" src="https://github.com/user-attachments/assets/ae8ab734-093c-4df2-8b16-c1826db51ecb" />

### 3. 노드 확인

배포가 완료되면 아래 명령어로 노드가 정상적으로 배포되었는지 확인한다.

```bash
kubectl get nodes
```

~/.kube/config 파일에 새로 생성된 클러스터의 자격 증명이 자동으로 추가된 것을 확인할 수 있다.

eksctl로 EKS 클러스터를 생성한 후, 현재까지 구성된 서비스의 전체 아키텍처는 다음과 같다.

![current architecture](https://static.us-east-1.prod.workshops.aws/public/0254fda7-64f0-4645-b9b1-6df053e6cf38/static/images/50-eks-cluster/eks-cluster.svg)

# **Amazon EKS Access Entry 설정하기**

EKS 콘솔에 접속하면, 기본적으로 현재 IAM 사용자는 Kubernetes 오브젝트에 접근할 수 없다. 따라서, 접근 권한을 명시적으로 부여하기 위해 EKS IAM Access Entry를 생성해야 한다.

**EKS Access Entry 는 사용자에게 Kubernetes API 접근 권한을 부여하는 권장 방식이다. 이를 통해 개발자에게 kubectl 명령을 사용할 수 있는 권한**을 줄 수 있다.

### **Access Entry 생성 절차**

Amazon EKS 콘솔에서 생성한 eks-demo 클러스터를 클릭하고, 상세 화면에서 Create access entry 버튼을 선택한다.

![](https://static.us-east-1.prod.workshops.aws/public/0254fda7-64f0-4645-b9b1-6df053e6cf38/static/images/50-eks-cluster/eks-console-no-shown.png)

IAM principal를 확인하고 다음 버튼을 누른다.

![](https://static.us-east-1.prod.workshops.aws/public/0254fda7-64f0-4645-b9b1-6df053e6cf38/static/images/50-eks-cluster/eks-console-01.png)

AmazonEKSClusterAdminPolicy를 검색한 뒤 Add policy 버튼을 클릭한다. 정책이 적용된 것을 아래 화면에서 확인할 수 있다.

![](https://static.us-east-1.prod.workshops.aws/public/0254fda7-64f0-4645-b9b1-6df053e6cf38/static/images/50-eks-cluster/eks-console-02.png)

Access Entry 생성이 완료되면, 콘솔에서 클러스터 노드들의 상세 정보를 정상적으로 확인할 수 있다.

![](https://static.us-east-1.prod.workshops.aws/public/0254fda7-64f0-4645-b9b1-6df053e6cf38/static/images/50-eks-cluster/eks-console-03.png)
