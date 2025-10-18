# 인그레스 컨트롤러 만들기

인그레스(Ingress)는 클러스터 외부에서 쿠버네티스 내부로 들어오는 요청을 어떻게 처리할지를 정의해둔 규칙이자 리소스 오브젝트이다. **한마디로 외부 요청이 내부 서비스로 접근하기 위한 관문(Gateway) 역할**을 한다.

인그레스는 L7(애플리케이션 계층)에서 작동하며, 다음과 같은 기능을 설정할 수 있다.

- 외부 요청에 대한 로드 밸런싱
- TLS/SSL 인증서 처리
- HTTP 경로 기반 라우팅

쿠버네티스에서는 NodePort나 LoadBalancer 타입의 서비스로도 외부 접근이 가능하지만, **인그레스를 사용하지 않을 경우 각 서비스마다 라우팅 규칙과 TLS 설정을 개별적으로 관리해야 한다.**

따라서, 외부 요청을 효율적으로 관리하고 일관된 방식으로 라우팅하기 위해 **인그레스(ingress)** 가 필요하다.

![Ingress Controller](https://static.us-east-1.prod.workshops.aws/public/0254fda7-64f0-4645-b9b1-6df053e6cf38/static/images/60-ingress-controller/ingress-controller.svg)

인그레스는 단순히 규칙을 정의한 리소스이며, 이 규칙이 실제로 동작하기 위해서는 이를 해석하고 적용하는 **인그레스 컨트롤러**가 필요하다. 

인그레스 컨트롤러는 **kube-controller-manager**의 일부로 자동 생성되지 않기 때문에, 클러스터 구성 후 **직접 구현 및 배포해야 한다.**

# AWS Load Balancer 컨트롤러 만들기

Amazon EKS의 Application Load Balancing이란 클러스터에 Ingress 자원이 생성될 때, **ALB(Application Load Balancer) 및 필요한 AWS 리소스들이 자동으로 생성되도록 트리거하는 컨트롤러**이다.

Ingress 자원은 ALB를 구성하여 HTTP 또는 HTTPS 트래픽을 클러스터 내부의 파드로 라우팅한다.

### **AWS Load Balancer 컨트롤러의 주요 동작 방식**

1. **쿠버네티스의 Ingress는 Application Load Balancer로 프로비저닝된다.**
    
    사용자가 Ingress 리소스를 생성하면, AWS Load Balancer Controller가 해당 설정을 읽고 실제 AWS 상에서 Application Load Balancer(ALB) 를 자동으로 생성한다.
    
    이렇게 만들어진 **ALB는 HTTP/HTTPS 등 L7(애플리케이션 계층) 트래픽을 처리하며, 정의된 경로나 도메인 규칙에 따라 요청을 적절한 서비스로 라우팅**한다.
    
    즉, Ingress는 어떤 요청을 어떤 서비스로 보낼지를 정의하고, ALB는 그 정의를 실제로 수행하는 물리적인 진입점이 된다.
    
2. **쿠버네티스의 Service는 Network Load Balancer로 프로비저닝된다.**
    
    Service 리소스를 type: LoadBalancer로 생성하면, AWS Load Balancer Controller가 이를 감지하고 Network Load Balancer(NLB) 를 생성한다.
    
    NLB는 TCP/UDP 등 L4(전송 계층) 트래픽을 처리하며, 클러스터 외부에서 내부 파드로 네트워크 연결을 직접 전달한다.
    
    **HTTP 수준의 라우팅이나 인증서 처리는 수행하지 않지만, 고성능 저지연 트래픽 처리가 필요한 경우에 적합하다.**
    

AWS Load Balancer 컨트롤러는 아래의 두 가지 트래픽 모드를 지원한다.

| **모드** | **설명** |
| --- | --- |
| **Instance (기본)** | 클러스터 내 노드를 ALB의 대상으로 등록한다. ALB 트래픽은 NodePort를 통해 파드로 전달된다. |
| **IP** | 파드를 ALB 대상으로 직접 등록한다. 트래픽이 파드로 바로 전달된다. 이 모드를 사용하려면 ingress.yaml 파일에 주석(annotation)으로 명시해야 한다. |

![](https://static.us-east-1.prod.workshops.aws/public/0254fda7-64f0-4645-b9b1-6df053e6cf38/static/images/60-ingress-controller/alb-ingress-controller-traffic-mode.svg)

---

### 1. 매니페스트 디렉터리 생성

앞으로 관리할 매니페스트 파일들을 정리하기 위해 루트 폴더(/home/ec2-user/environment) 아래에 manifests 폴더를 만들고, 그 안에 alb-ingress-controller 폴더를 생성한다.

```bash
cd ~/environment
mkdir -p manifests/alb-ingress-controller && cd manifests/alb-ingress-controller
```

```bash
# 최종 경로
/home/ec2-user/environment/manifests/alb-ingress-controller
```

AWS Load Balancer 컨트롤러를 배포하기 전에는 컨트롤러가 워커 노드에서 동작할 수 있도록 IAM permissions 설정이 필요하다.

IAM 권한은 다음 두 가지 방식 중 하나로 부여할 수 있다.

1. Service Account용 IAM Role 생성
2. 워커 노드의 IAM Role에 직접 권한 추가

### **2. IAM OIDC Provider 생성**

AWS Load Balancer 컨트롤러를 배포하기 전, 클러스터에 대한 IAM OIDC(OpenID Connect) identity provider 를 생성한다. 이는 쿠버네티스의 ServiceAccount 가 IAM Role을 사용할 수 있게 해준다.

```bash
eksctl utils associate-iam-oidc-provider \
    --region ${AWS_REGION} \
    --cluster eks-demo \
    --approve
```

OIDC Provider는 IAM 콘솔의 Identity providers 메뉴 또는 아래 명령어를 통해 확인할 수 있다.

```bash
aws eks describe-cluster --name eks-demo --query "cluster.identity.oidc.issuer" --output text
```

출력 예시는 다음과 같다

```bash
https://oidc.eks.ap-northeast-2.amazonaws.com/id/8A6E78112D7F1C4DC352B1B511DD13CF
```

### **3. IAM Policy 생성**

AWS Load Balancer 컨트롤러에 부여할 IAM Policy를 생성한다.

```bash
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.13.3/docs/install/iam_policy.json
```

```bash
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json
```

### 4. ServiceAccount 생성

IAM Policy를 ServiceAccount에 연결하여 컨트롤러가 해당 권한을 사용할 수 있게 한다.

```bash
eksctl create iamserviceaccount \
    --cluster eks-demo \
    --namespace kube-system \
    --name aws-load-balancer-controller \
    --attach-policy-arn arn:aws:iam::$ACCOUNT_ID:policy/AWSLoadBalancerControllerIAMPolicy \
    --override-existing-serviceaccounts \
    --region ${AWS_REGION} \
    --approve
```

### 5. 클러스터에 컨트롤러 추가

1. **cert-manager 설치**
    
    웹훅 인증서 구성을 삽입할 수 있도록 cert-manager를 설치한다. cert-manager는 쿠버네티스 내에서 TLS 인증서를 자동 관리하는 오픈소스다.
    
    ```bash
    kubectl apply --validate=false -f https://github.com/cert-manager/cert-manager/releases/download/v1.12.3/cert-manager.yaml
    ```
    
2. **컨트롤러 매니페스트 다운로드**
    
    ```bash
    curl -Lo v2_13_3_full.yaml https://github.com/kubernetes-sigs/aws-load-balancer-controller/releases/download/v2.13.3/v2_13_3_full.yaml
    ```
    
    ServiceAccount 설정이 이미 존재하므로, 매니페스트 내의 ServiceAccount 섹션(730~738행) 을 제거한다.
    
    ```bash
    sed -i.bak -e '730,738d' ./v2_13_3_full.yaml
    ```
    
    Deployment 섹션의 your-cluster-name을 현재 클러스터 이름(eks-demo)으로 변경한다.
    
    ```bash
    sed -i.bak -e 's|your-cluster-name|eks-demo|' ./v2_13_3_full.yaml
    ```
    
3. **컨트롤러 배포**
    
    ```bash
    kubectl apply -f v2_13_3_full.yaml
    ```
    
    IngressClass 및 IngressClassParams 매니페스트를 다운로드하여 클러스터에 적용한다.
    
    ```bash
    curl -Lo v2_13_3_ingclass.yaml https://github.com/kubernetes-sigs/aws-load-balancer-controller/releases/download/v2.13.3/v2_13_3_ingclass.yaml
    kubectl apply -f v2_13_3_ingclass.yaml
    ```
    

### **6. 배포 확인**

컨트롤러가 정상적으로 배포되었는지 확인한다.

```bash
kubectl get deployment -n kube-system aws-load-balancer-controller
```

ServiceAccount 생성 여부를 확인한다.

```bash
kubectl get sa aws-load-balancer-controller -n kube-system -o yaml
```

클러스터 내부에서 필요한 기능을 수행하는 파드들은 애드온이라고 한다. 이러한 애드온 파드들은 Deployment나 ReplicationController 등에 의해 관리되며, 보통 kube-system 네임스페이스에서 실행된다.

따라서, 위 명령어로 파드 이름이 정상적으로 출력되면 컨트롤러가 올바르게 배포된 것이다.

### **7. 로그 및 상세 정보 확인**

컨트롤러의 로그를 확인한다.

```bash
kubectl logs -n kube-system $(kubectl get po -n kube-system | egrep -o "aws-load-balancer[a-zA-Z0-9-]+")
```

파드의 상세 속성을 확인한다.

```bash
ALBPOD=$(kubectl get pod -n kube-system | egrep -o "aws-load-balancer[a-zA-Z0-9-]+")
kubectl describe pod -n kube-system ${ALBPOD}
```
