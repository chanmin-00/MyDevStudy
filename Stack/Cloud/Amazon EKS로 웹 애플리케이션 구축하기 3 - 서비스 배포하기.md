# 서비스 배포하기

이번 실습에서는 백엔드와 프론트엔드로 구성된 웹 애플리케이션을 Amazon EKS 클러스터에 배포하는 과정을 진행한다.

### 전체 배포 흐름

전체적인 배포 과정은 크게 4단계로 나눌 수 있다.

![Pod Deploy Steps](https://static.us-east-1.prod.workshops.aws/public/0254fda7-64f0-4645-b9b1-6df053e6cf38/static/images/70-deploy-service/service-deploy-step.svg)

- 소스 코드 준비
- ECR 리포지토리 구성 및 컨테이너 이미지 업로드
- Kubernetes 매니페스트 파일 작성 (Deployment, Service, Ingress)
- 클러스터에 리소스 배포

사용자는 ALB → Ingress → Service → Pod 순서로 애플리케이션에 접근하게 된다.

![Service Access Sequence](https://static.us-east-1.prod.workshops.aws/public/0254fda7-64f0-4645-b9b1-6df053e6cf38/static/images/70-deploy-service/service-access.svg)

### Flask 백엔드 구성

ECR에 이미지를 업로드하는 작업이 먼저 완료되어야 한다.

1. **작업 디렉토리 이동**
    
    ```bash
    cd ~/environment/manifests/
    ```
    
2. **Deployment 리소스 정의**
    
    Flask 애플리케이션을 3개의 레플리카로 실행하도록 설정한다.
    
    ```bash
    cat <<EOF> flask-deployment.yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: demo-flask-backend
      namespace: default
    spec:
      replicas: 3
      selector:
        matchLabels:
          app: demo-flask-backend
      template:
        metadata:
          labels:
            app: demo-flask-backend
        spec:
          containers:
          - name: demo-flask-backend
            image: $ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/demo-flask-backend:latest
            imagePullPolicy: Always
            ports:
            - containerPort: 8080
    EOF
    ```
    
3. **Service 리소스 정의**
    
    Pod들을 하나의 엔드포인트로 묶어주는 Service를 생성한다.
    
    ```bash
    cat <<EOF> flask-service.yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: demo-flask-backend
      annotations:
        alb.ingress.kubernetes.io/healthcheck-path: "/contents/aws"
    spec:
      selector:
        app: demo-flask-backend
      type: NodePort
      ports:
      - port: 8080
        targetPort: 8080
        protocol: TCP
    EOF
    ```
    
4. **Ingress 리소스 정의**
    
    외부에서 접근 가능하도록 ALB를 프로비저닝하는 Ingress를 구성한다.
    
    ```bash
    cat <<EOF> flask-ingress.yaml
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: "flask-backend-ingress"
      namespace: default
      annotations:
        alb.ingress.kubernetes.io/scheme: internet-facing
        alb.ingress.kubernetes.io/target-type: ip
        alb.ingress.kubernetes.io/group.name: eks-demo-group
        alb.ingress.kubernetes.io/group.order: '1'
    spec:
      ingressClassName: alb
      rules:
      - http:
          paths:
          - path: /contents
            pathType: Prefix
            backend:
              service:
                name: "demo-flask-backend"
                port:
                  number: 8080
    EOF
    ```
    
5. **클러스터에 배포**
    
    작성한 매니페스트 파일들을 순차적으로 적용한다.
    
    ```bash
    kubectl apply -f flask-deployment.yaml
    kubectl apply -f flask-service.yaml
    kubectl apply -f flask-ingress.yaml
    ```
    
    Ingress 리소스가 생성되면 AWS에서 자동으로 ALB가 프로비저닝된다.
    
6. 동작 확인
    
    아래 명령어로 생성된 ALB의 엔드포인트를 확인하고, 브라우저나 Postman 등에서 테스트한다.
    
    ```bash
    echo http://$(kubectl get ingress/flask-backend-ingress -o jsonpath='{.status.loadBalancer.ingress[*].hostname}')/contents/aws
    ```
    
    ALB가 완전히 프로비저닝되기까지 수 분 정도 소요될 수 있다. EC2 콘솔의 Load Balancer 메뉴에서 상태를 확인할 수 있다.
    
    <img width="821" height="370" alt="image" src="https://github.com/user-attachments/assets/8acbd6a0-7c78-4f31-81a7-01a5e658645a" />

    

현재까지의 아키텍처는 아래와 같다.

![](https://static.us-east-1.prod.workshops.aws/public/0254fda7-64f0-4645-b9b1-6df053e6cf38/static/images/70-deploy-service/first-service-deploy.svg)

# 두번째 백앤드 배포하기

Flask와 동일한 방식으로 Express 백엔드도 EKS 클러스터에 배포한다. 이번 실습에서는 시간 단축을 위해 미리 빌드된 퍼블릭 이미지를 사용한다. 실제 환경에서는 직접 빌드한 이미지를 ECR에 푸시하여 사용하면 된다.

1. **Deployment 작성**
    
    작업 디렉토리로 이동 후 Node.js 애플리케이션을 배포할 Deployment를 정의한다.
    
    ```bash
    cd ~/environment/manifests/
    ```
    
    ```bash
    cat <<EOF> nodejs-deployment.yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: demo-nodejs-backend
      namespace: default
    spec:
      replicas: 3
      selector:
        matchLabels:
          app: demo-nodejs-backend
      template:
        metadata:
          labels:
            app: demo-nodejs-backend
        spec:
          containers:
          - name: demo-nodejs-backend
            image: public.ecr.aws/y7c9e1d2/joozero-repo:latest
            imagePullPolicy: Always
            ports:
            - containerPort: 3000
    EOF
    ```
    
2. **Service 작성**
    
    Express 애플리케이션의 3000번 포트를 8080번으로 매핑하는 Service를 생성한다.
    
    ```bash
    cat <<EOF> nodejs-service.yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: demo-nodejs-backend
      annotations:
        alb.ingress.kubernetes.io/healthcheck-path: "/services/all"
    spec:
      selector:
        app: demo-nodejs-backend
      type: NodePort
      ports:
      - port: 8080
        targetPort: 3000
        protocol: TCP
    EOF
    ```
    
3. **Ingress 작성**
    
    `/services` 경로로 들어오는 요청을 Express 백엔드로 라우팅하는 Ingress를 구성한다.
    
    ```bash
    cat <<EOF> nodejs-ingress.yaml
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: "nodejs-backend-ingress"
      namespace: default
      annotations:
        alb.ingress.kubernetes.io/scheme: internet-facing
        alb.ingress.kubernetes.io/target-type: ip
        alb.ingress.kubernetes.io/group.name: eks-demo-group
        alb.ingress.kubernetes.io/group.order: '2'
    spec:
      ingressClassName: alb
      rules:
      - http:
          paths:
          - path: /services
            pathType: Prefix
            backend:
              service:
                name: "demo-nodejs-backend"
                port:
                  number: 8080
    EOF
    ```
    
    여기서 주목할 점은 `alb.ingress.kubernetes.io/group.name: eks-demo-group` 어노테이션이다. 
    
    Flask Ingress에서도 **동일한 그룹명을 사용했기 때문에, 두 Ingress가 하나의 ALB를 공유하게 된다.** 하나의 ALB가 경로 기반 라우팅을 할 수 있다. ALB의 경로 기반 라우팅 기능을 활용하여 `/contents`는 Flask로, `/services`는 Express로 트래픽을 분산한다.
    
    group.order 값은 라우팅 규칙의 우선순위를 결정한다. 숫자가 작을수록 먼저 평가된다.
    
4. **리소스 배포**
    
    작성한 매니페스트들을 클러스터에 적용한다.
    
    ```bash
    kubectl apply -f nodejs-deployment.yaml
    kubectl apply -f nodejs-service.yaml
    kubectl apply -f nodejs-ingress.yaml
    ```
    
5. **엔드포인트 확인**
    
    생성된 ALB 주소로 API 요청을 보내 정상 동작 여부를 확인한다.
    
    ```bash
    echo http://$(kubectl get ingress/nodejs-backend-ingress -o jsonpath='{.status.loadBalancer.ingress[*].hostname}')/services/all
    ```
    
    브라우저나 API 테스트 도구에서 위 URL로 접속하여 응답을 확인할 수 있다.
    
    ![](https://static.us-east-1.prod.workshops.aws/public/0254fda7-64f0-4645-b9b1-6df053e6cf38/static/images/70-deploy-service/second-service-deploy.svg)
    

# React 프론트엔드 구성

두 개의 백엔드 배포를 완료했다면, 이제 사용자가 실제로 보게 될 웹 페이지를 구성하는 프론트엔드를 배포한다.

1. **소스 코드 다운로드**
    
    프론트엔드 애플리케이션 소스 코드를 클론한다.
    
    ```bash
    cd /home/ec2-user/environment
    git clone https://github.com/joozero/amazon-eks-frontend.git
    ```
    
2. **ECR 리포지터리 생성**
    
    프론트엔드 이미지를 저장할 리포지토리를 생성한다.
    
    ```bash
    aws ecr create-repository \
    --repository-name demo-frontend \
    --image-scanning-configuration scanOnPush=true \
    --region ${AWS_REGION}
    ```
    
3. **API 엔드포인트 설정**
    
    백엔드 API를 호출하기 위해 소스 코드의 URL을 수정해야 한다.
    
    수정할 파일 위치는 다음과 같다.
    
    - `/home/ec2-user/environment/amazon-eks-frontend/src/App.js`
    - `/home/ec2-user/environment/amazon-eks-frontend/src/page/UpperPage.js`
    
    Flask 백엔드 엔드포인트를 확인하는 명령어이다.
    
    ```bash
    echo http://$(kubectl get ingress/flask-backend-ingress -o jsonpath='{.status.loadBalancer.ingress[*].hostname}')/contents/'${search}'
    ```
    
    - 위 명령어 결과를 `App.js`의 해당 URL 부분에 붙여넣는다.
    
    Express 백엔드 엔드포인트를 확인하는 명령어이다.
    
    ```bash
    echo http://$(kubectl get ingress/nodejs-backend-ingress -o jsonpath='{.status.loadBalancer.ingress[*].hostname}')/services/all
    ```
    
    - 위 명령어 결과를 `UpperPage.js`의 해당 URL 부분에 붙여넣는다.
4. **프론트엔드 빌드**
    
    React 애플리케이션을 빌드한다.
    
    ```bash
    cd /home/ec2-user/environment/amazon-eks-frontend
    npm install
    npm run build
    ```
    
    - npm install 실행 후 security vulnerability 경고가 나타나면, npm audit fix 명령어를 먼저 실행한 뒤 npm run build를 진행한다.
5. **Docker 이미지 빌드 및 푸시**
    
    빌드된 애플리케이션을 컨테이너 이미지로 만들어 ECR에 업로드한다.
    
    ```bash
    docker build -t demo-frontend .
    
    docker tag demo-frontend:latest $ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/demo-frontend:latest
    
    docker push $ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/demo-frontend:latest
    ```
    
    ```bash
    aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com
    ```
    
    - `denied: Your authorization token has expired` 에러가 발생하면 위 명령어로 재인증 후 다시 푸시한다.
6. **Kubernetes 매니페스트 작성**
    
    manifests 폴더로 이동하여 프론트엔드 리소스를 정의한다.
    
    ```bash
    cd /home/ec2-user/environment/manifests
    ```
    
    ```bash
    cat <<EOF> frontend-deployment.yaml
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: demo-frontend
      namespace: default
    spec:
      replicas: 3
      selector:
        matchLabels:
          app: demo-frontend
      template:
        metadata:
          labels:
            app: demo-frontend
        spec:
          containers:
            - name: demo-frontend
              image: $ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/demo-frontend:latest
              imagePullPolicy: Always
              ports:
                - containerPort: 80
    EOF
    ```
    
    ```bash
    cat <<EOF> frontend-service.yaml
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: demo-frontend
      annotations:
        alb.ingress.kubernetes.io/healthcheck-path: "/"
    spec:
      selector:
        app: demo-frontend
      type: NodePort
      ports:
        - protocol: TCP
          port: 80
          targetPort: 80
    EOF
    ```
    
    ```bash
    cat <<EOF> frontend-ingress.yaml
    ---
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: "frontend-ingress"
      namespace: default
      annotations:
        alb.ingress.kubernetes.io/scheme: internet-facing
        alb.ingress.kubernetes.io/target-type: ip
        alb.ingress.kubernetes.io/group.name: eks-demo-group
        alb.ingress.kubernetes.io/group.order: '3'
    spec:
      ingressClassName: alb
      rules:
        - http:
            paths:
              - path: /
                pathType: Prefix
                backend:
                  service:
                    name: "demo-frontend"
                    port:
                      number: 80
    EOF
    ```
    
7. **리소스 배포**
    
    작성한 매니페스트를 클러스터에 적용한다.
    
    ```bash
    kubectl apply -f frontend-deployment.yaml
    kubectl apply -f frontend-service.yaml
    kubectl apply -f frontend-ingress.yaml
    ```
    
8. **웹 페이지 접속**
    
    프론트엔드 엔드포인트를 확인하고 브라우저로 접속한다.
    
    ```bash
    echo http://$(kubectl get ingress/frontend-ingress -o jsonpath='{.status.loadBalancer.ingress[*].hostname}')
    ```
    
    정상적으로 배포되었다면 React 애플리케이션 화면이 표시된다.
    
    <img width="783" height="385" alt="image" src="https://github.com/user-attachments/assets/4c55d921-d5cb-4b14-b004-436360bde3ea" />


프론트엔드까지 배포를 완료한 전체 시스템 아키텍처는 다음과 같다. **하나의 ALB가 세 개의 서비스를 경로별로 라우팅한다.**

- `/` → React 프론트엔드
- `/contents` → Flask 백엔드
- `/services` → Express 백엔드

사용자는 단일 엔드포인트를 통해 프론트엔드에 접속하고, 프론트엔드에서 백엔드 API를 호출하여 데이터를 화면에 표시하는 구조로 동작한다.
