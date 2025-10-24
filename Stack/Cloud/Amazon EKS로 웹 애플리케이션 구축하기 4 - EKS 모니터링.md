# Container Insights로 EKS 모니터링하기

### Container Insights

**Container Insights는 AWS에서 제공하는 컨테이너 모니터링 서비스**이다. EKS 클러스터의 CPU, 메모리, 네트워크 같은 메트릭을 자동으로 수집해주며, CloudWatch 대시보드에서 한눈에 확인할 수 있다.

주요 장점은 다음과 같다:

- 별도 설정 없이 인프라 메트릭이 자동 수집된다
- crashloop 같은 문제 상황을 빠르게 파악할 수 있다
- 로그와 메트릭을 한 곳에서 통합 관리할 수 있다

![](https://static.us-east-1.prod.workshops.aws/public/0254fda7-64f0-4645-b9b1-6df053e6cf38/static/images/80-monitoring/container-insight-resource-02.png)

### 구성 요소

Container Insights는 두 가지 핵심 컴포넌트로 구성된다

- **CloudWatch Agent**: 클러스터의 메트릭 데이터를 수집하는 역할을 담당한다
- **Fluent Bit**: 애플리케이션 로그를 CloudWatch Logs로 전송하는 역할을 한다

두 컴포넌트 모두 DaemonSet 형태로 배포되어 모든 노드에서 자동으로 실행된다.

![](https://static.us-east-1.prod.workshops.aws/public/0254fda7-64f0-4645-b9b1-6df053e6cf38/static/images/80-monitoring/container-insight-overview.png)

### 설치 과정

1. **네임스페이스 생성**
    
    ```bash
    kubectl create ns amazon-cloudwatch
    kubectl get ns  # 생성 확인
    ```
    
2. **환경변수 설정**
    
    ```bash
    ClusterName=eks-demo
    RegionName=$AWS_REGION
    FluentBitHttpPort='2020'
    FluentBitReadFromHead='Off'
    [[ ${FluentBitReadFromHead} = 'On' ]] && FluentBitReadFromTail='Off'|| FluentBitReadFromTail='On'
    [[ -z ${FluentBitHttpPort} ]] && FluentBitHttpServer='Off' || FluentBitHttpServer='On'
    ```
    
    - RegionName이 실제 리전(예: ap-northeast-2)으로 올바르게 설정되었는지 반드시 확인해야 한다.
3. **매니페스트 다운로드 및 배포**
    
    ```bash
    # YAML 파일 다운로드
    wget https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-monitoring/quickstart/cwagent-fluent-bit-quickstart.yaml
    
    # 환경 변수 적용
    sed -i 's/{{cluster_name}}/'${ClusterName}'/;s/{{region_name}}/'${RegionName}'/;s/{{http_server_toggle}}/"'${FluentBitHttpServer}'"/;s/{{http_server_port}}/"'${FluentBitHttpPort}'"/;s/{{read_from_head}}/"'${FluentBitReadFromHead}'"/;s/{{read_from_tail}}/"'${FluentBitReadFromTail}'"/' cwagent-fluent-bit-quickstart.yaml
    
    # 배포 실행
    kubectl apply -f cwagent-fluent-bit-quickstart.yaml
    ```
    
4. **설치 확인**
    
    ```bash
    # Pod 확인 (cloudwatch-agent와 fluent-bit 각각 3개씩)
    kubectl get po -n amazon-cloudwatch
    
    # DaemonSet 확인 (총 2개)
    kubectl get daemonsets -n amazon-cloudwatch
    ```
    

## CloudWatch 콘솔에서 모니터링하기

CloudWatch 콘솔에서 Container Insights → Resources로 이동하면 다양한 뷰를 활용할 수 있다

- **List View**: 모든 리소스를 목록 형태로 확인할 수 있다
    
    ![](https://static.us-east-1.prod.workshops.aws/public/0254fda7-64f0-4645-b9b1-6df053e6cf38/static/images/80-monitoring/container-insight-resource-01.png)
    
- **Map View**: 클러스터 리소스를 트리 구조로 시각화하여 보여준다. 특정 오브젝트를 클릭하면 관련 메트릭도 함께 확인 가능하다
    
    ![](https://static.us-east-1.prod.workshops.aws/public/0254fda7-64f0-4645-b9b1-6df053e6cf38/static/images/80-monitoring/container-insight-resource-03.png)
    
- **Pod Performance**: 각 파드의 성능 메트릭을 상세하게 볼 수 있다
    
    ![](https://static.us-east-1.prod.workshops.aws/public/0254fda7-64f0-4645-b9b1-6df053e6cf38/static/images/80-monitoring/container-insight-pod-performance.png)
    

특정 파드를 선택한 후 Actions → View performance logs를 클릭하면 CloudWatch Logs Insights로 이동하여 로그를 쿼리할 수 있다.
