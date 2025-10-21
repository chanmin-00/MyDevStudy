# 1. Object Storage 버킷 생성

Container Registry를 사용하기 위해서는 먼저 Object Storage에 버킷을 생성해야 한다. Object Storage는 AWS의 S3와 유사한 역할을 수행한다.

생성 절차는 다음과 같다.

1. NCP 콘솔에서 Object Storage 메뉴로 이동
    
    <img width="539" height="141" alt="image" src="https://github.com/user-attachments/assets/d1105e01-47b5-4af4-87f1-4b383bd35af9" />
    
2. 버킷 생성 버튼 클릭
3. 버킷 이름 및 설정 입력 후 생성 완료

<img width="751" height="523" alt="image" src="https://github.com/user-attachments/assets/74f27946-50c1-48f8-85cb-57a0f1dda8b6" />

# 2. Container Registry 생성

버킷 생성 후 Container Registry를 생성한다. Container Registry는 Docker Hub나 GitHub Container Registry와 같은 이미지 저장소 역할을 한다.

### Login

Docker CLI를 통해 레지스트리에 로그인한다.

```bash
docker login paaspaas.kr.ncr.ntruss.com
```

- 요구사항: Docker Engine 1.10 이상
- Username: NCP Access Key ID
- Password: NCP Secret Key

### Push

이미지를 레지스트리에 업로드한다.

```bash
# 이미지 태깅
docker tag <IMAGE_NAME> paaspaas.kr.ncr.ntruss.com/<TARGET_IMAGE[:TAG]>

# 이미지 푸시
docker push paaspaas.kr.ncr.ntruss.com/<TARGET_IMAGE[:TAG]>
```

### Pull

레지스트리에서 이미지를 다운로드한다.

```bash
docker pull paaspaas.kr.ncr.ntruss.com/<IMAGE_NAME[:TAG]>
```

Public Endpoint가 비활성화된 경우 위 명령어가 동작하지 않는다. Private Endpoint를 사용하면 모든 트래픽을 NCP VPC 네트워크 내부로 제한하여 보안을 강화할 수 있다.

# 3. 이미지 빌드 및 푸시

### 이미지 빌드

<img width="727" height="96" alt="image" src="https://github.com/user-attachments/assets/be4012d4-5d2d-4b74-b85e-768bdeeeb60b" />

Container Registry의 Endpoint를 prefix로 지정하여 이미지를 빌드한다.

```bash
docker buildx build --platform linux/amd64 \
  -t paaspaas.kr.ncr.ntruss.com/document-server:latest \
  --load .
```

### 이미지 푸시

```bash
docker push paaspaas.kr.ncr.ntruss.com/document-server:latest
```

- 처음 푸시 시 `denied: you don't have the permissions` 오류가 발생할 수 있다.
- 이 경우 `docker login` 명령으로 먼저 로그인해야 한다.
    
    <img width="697" height="212" alt="image" src="https://github.com/user-attachments/assets/ed59a92e-e456-4914-b51d-19bc4066745d" />
    

푸시가 정상적으로 완료되면 Object Storage에 디렉터리가 자동으로 생성된다.

# 4. Kubernetes Secret 설정

Kubernetes 클러스터가 프라이빗 Container Registry에서 이미지를 가져올 수 있도록 인증 정보를 Secret으로 등록한다.

### Secret 생성

```bash
kubectl create secret docker-registry <SECRET_NAME> \
  --docker-server=paaspaas.kr.ncr.ntruss.com \
  --docker-username=<NCP_ACCESS_KEY> \
  --docker-password=<NCP_SECRET_KEY>
```

- `--docker-server`에는 Registry의 Public Endpoint 또는 Private Endpoint를 명시한다.

# 5. Deployment 설정

Deployment YAML 파일에서 프라이빗 레지스트리의 이미지를 사용하도록 설정한다.

```bash
apiVersion: apps/v1
kind: Deployment

metadata:
  name: chat-deployment

spec:
  replicas: 5
  selector:
    matchLabels:
      app: chat-app

  template:
    metadata:
      labels:
        app: chat-app
    spec:
      containers:
        - name: chat-container
          image: paaspaas.kr.ncr.ntruss.com/chat-server:latest
          imagePullPolicy: Always
          ports:
            - containerPort: 8080
          env:
            - name: DB_HOST
              valueFrom:
                configMapKeyRef:
                  name: chat-config
                  key: db-host
            - name: DB_PORT
              valueFrom:
                configMapKeyRef:
                  name: chat-config
                  key: db-port
            - name: DB_NAME
              valueFrom:
                configMapKeyRef:
                  name: chat-config
                  key: db-name
            - name: DB_USERNAME
              valueFrom:
                secretKeyRef:
                  name: chat-secret
                  key: db-username
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: chat-secret
                  key: db-password
      imagePullSecrets:
        - name: chat-server-image-secret
```

- `image`: Container Registry의 전체 경로로 변경
- `imagePullSecrets`: 앞서 생성한 Secret 이름 지정

### Deployment 적용

```bash
kubectl apply -f chat-deployment.yaml
```

적용 후 Pod가 정상적으로 Running 상태인지 확인한다.

```bash
kubectl get pods
```
