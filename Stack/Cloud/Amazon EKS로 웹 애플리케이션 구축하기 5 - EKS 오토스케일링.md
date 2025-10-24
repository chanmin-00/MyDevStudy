# EKS 오토스케일링

쿠버네티스 환경에서는 두 가지 주요 오토스케일링 방식이 존재한다

- **HPA (Horizontal Pod Autoscaler)** : Pod의 개수를 자동으로 조정한다. CPU 사용률이나 커스텀 메트릭을 기반으로 동작한다
- **Cluster Autoscaler** : 워커 노드 자체를 증설하거나 축소한다. HPA로 파드를 늘렸지만 노드 리소스가 부족한 경우에 필요하다

이 두 가지 메커니즘을 조합하면 트래픽 변화에 자동으로 대응하는 탄력적인 인프라를 구성할 수 있다.

# EKS에서 Auto Scaling 구성하기

## HPA를 통한 Pod Auto Scaling

**HPA(Horizontal Pod Autoscaler)는 메트릭 기반으로 파드 수를 자동 조절하는 컨트롤러**다. 효과적인 스케일링을 위해서는 컨테이너의 리소스 요구사항을 명확히 정의하고, 스케일링 조건을 설정해야 한다.

![hpa scaling](https://static.us-east-1.prod.workshops.aws/public/0254fda7-64f0-4645-b9b1-6df053e6cf38/static/images/90-scaling/k8s-scaling.svg)

1. **Metrics Server 확인**
    
    Metrics Server는 클러스터의 리소스 사용량을 수집하는 역할을 한다. 각 워커 노드의 kubelet을 통해 컨테이너의 CPU, 메모리 등의 메트릭을 집계한다.
    
    ```java
    kubectl get deployment metrics-server -n kube-system
    ```
    
    - 메트릭 서버가 정상적으로 있는지 확인한다.
2. **Deployment 리소스 설정**
    
    기존 flask deployment를 수정하여 레플리카 수를 1로 설정하고, 컨테이너 리소스를 정의한다.
    
    ```java
    cd /home/ec2-user/environment/manifests
    
    cat <<EOF> flask-deployment.yaml
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: demo-flask-backend
      namespace: default
    spec:
      replicas: 1
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
              resources:
                requests:
                  cpu: 250m
                limits:
                  cpu: 500m
    EOF
    ```
    
3. **변경사항 적용**
    
    ```java
    kubectl apply -f flask-deployment.yaml
    ```
    
4. **HPA 설정**
    
    HPA 리소스를 생성하여 오토스케일링 정책을 정의한다.
    
    ```java
    cat <<EOF> flask-hpa.yaml
    ---
    apiVersion: autoscaling/v1
    kind: HorizontalPodAutoscaler
    metadata:
      name: demo-flask-backend-hpa
      namespace: default
    spec:
      scaleTargetRef:
        apiVersion: apps/v1
        kind: Deployment
        name: demo-flask-backend
      minReplicas: 1
      maxReplicas: 5
      targetCPUUtilizationPercentage: 30
    EOF
    
    kubectl apply -f flask-hpa.yaml
    ```
    
    또는 kubectl 명령어로 간단히 설정 가능하다
    
    ```java
    kubectl autoscale deployment demo-flask-backend --cpu-percent=30 --min=1 --max=5
    ```
    
5. **HPA 상태 모니터링**
    
    ```java
    kubectl get hpa
    ```
    
    CPU 사용률이 unknown으로 표시되면 잠시 기다린 후 재확인한다.
    
6. 부하 테스트로 동작 확인
    - 터미널 1: HPA 모니터링
        
        ```java
        kubectl get hpa -w
        ```
        
    - 터미널 2: 부하 생성
        
        hey 도구를 설치하고 부하 테스트를 실행한다.
        
        ```java
        curl -LO https://hey-release.s3.us-east-2.amazonaws.com/hey_linux_amd64
        chmod +x hey_linux_amd64
        sudo mv hey_linux_amd64 /usr/local/bin/hey
        
        export flask_api=$(kubectl get ingress/flask-backend-ingress -o jsonpath='{.status.loadBalancer.ingress[*].hostname}')/contents/aws
        
        hey -n 20000 -c 1000 http://$flask_api
        ```
        
    
    부하가 증가하면 REPLICAS 값이 최대 5까지 늘어나는 것을 확인할 수 있다.
    
<img width="854" height="304" alt="image" src="https://github.com/user-attachments/assets/a07a5412-ac82-475c-9d19-2e07cd6b923f" />


## Cluster Autoscaler를 이용한 노드 Auto Scaling

파드 스케일링만으로는 한계가 있다. 워커 노드의 리소스가 부족하면 파드가 Pending 상태로 대기하게 되는데, 이때 Cluster Autoscaler(CA)가 필요하다.

![CA scaling](https://static.us-east-1.prod.workshops.aws/public/0254fda7-64f0-4645-b9b1-6df053e6cf38/static/images/90-scaling/k8s-ca-scaling.svg)

CA는 스케줄링되지 못한 파드가 있을 때 워커 노드를 자동으로 추가하고, 사용률이 낮아지면 노드를 제거한다. AWS 환경에서는 Auto Scaling Group과 연동하여 동작한다.

1. ASG 정보 확인
    
    ```bash
    export ASG_NAME=$(aws autoscaling describe-auto-scaling-groups \
      --query "AutoScalingGroups[?Tags[?Key=='eks:cluster-name' && Value=='eks-demo']].AutoScalingGroupName | [0]" \
      --output text)
    
    echo ${ASG_NAME}
    ```
    
2. 현재 ASG 설정 확인
    
    ```bash
    aws autoscaling describe-auto-scaling-groups \
      --auto-scaling-group-names "$ASG_NAME" \
      --query "AutoScalingGroups[].[AutoScalingGroupName, MinSize, MaxSize, DesiredCapacity]" \
      --output table
    ```
    
    - 본 실습 환경에서는 IAM 정책이 사전 구성되어 있다. 만약 별도 환경이라면 Auto Scaling 관련 IAM 정책을 생성하고 워커 노드 역할에 연결해야 한다.
3. ASG 최대 크기 조정
    
    ```bash
    aws autoscaling update-auto-scaling-group \
      --auto-scaling-group-name $ASG_NAME \
      --max-size 5
    ```
    
4. Cluster Autoscaler 배포
    
    공식 예제 파일을 다운로드하고 클러스터명을 변경한 후 배포한다.
    
    ```bash
    cd /home/ec2-user/environment/manifests
    
    wget https://raw.githubusercontent.com/kubernetes/autoscaler/master/cluster-autoscaler/cloudprovider/aws/examples/cluster-autoscaler-autodiscover.yaml
    
    sed -i 's|<YOUR CLUSTER NAME>|eks-demo|g' cluster-autoscaler-autodiscover.yaml
    
    kubectl apply -f cluster-autoscaler-autodiscover.yaml
    ```
    
5. 노드 스케일 아웃 테스트
    - 터미널 1: 노드 모니터링
        
        ```bash
        kubectl get nodes -w
        ```
        
    - 터미널 2: 대량 파드 생성
        
        ```bash
        kubectl create deployment autoscaler-demo --image=nginx
        kubectl scale deployment autoscaler-demo --replicas=100
        ```
        
    - 배포 진행 상태 확인
        
        ```bash
        kubectl get deployment autoscaler-demo --watch
        ```
        
        리소스 부족으로 스케줄링되지 못한 파드들이 생기면, CA가 자동으로 노드를 추가하는 것을 확인할 수 있다.
        
6. 스케일 인 확인
    
    테스트용 deployment를 삭제하면 노드가 자동으로 축소된다.
    
    ```bash
    kubectl delete deployment autoscaler-demo
    ```
    

<img width="701" height="426" alt="image" src="https://github.com/user-attachments/assets/8376d477-bc01-43e9-8b79-082c10a33c0b" />

1. 아래의 명령어로 이전에 생성한 파드를 삭제하면 워커 노드가 스케일인 되는 것을 확인할 수 있습니다.

`kubectl delete deployment autoscaler-demo`

<img width="842" height="224" alt="image" src="https://github.com/user-attachments/assets/1e536cb3-e2bb-4394-9e75-36166d886f17" />
