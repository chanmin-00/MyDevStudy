# Ingress

Kubernetes에서 **Ingress는 클러스터 외부의 HTTP/HTTPS 요청을 클러스터 내부의 서비스로 라우팅하는 API 리소스**이다.  

Ingress 리소스에 정의된 규칙에 따라 트래픽을 적절한 서비스로 연결한다.

**Ingress가 동작하려면 Ingress Controller가 필요하다.** Ingress Controller는 **Ingress 리소스의 규칙을 실제로 구현하고 트래픽을 관리하는 컴포넌트**이다. 이 가이드에서는 NGINX Ingress Controller를 사용한다.

ingress-nginx는 NGINX 웹 서버를 기반으로 하는 Kubernetes용 Ingress Controller 구현체이다. 오픈소스이며 Kubernetes 커뮤니티에서 공식적으로 관리한다.

# NGINX Ingress Controller 설치

클러스터의 Kubernetes 버전에 맞는 ingress-nginx를 설치한다:

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.13.3/deploy/static/provider/cloud/deploy.yaml
```

Ingress Controller Pod의 상태를 확인한다

```bash
kubectl get pods -n ingress-nginx
```

<img width="644" height="82" alt="image" src="https://github.com/user-attachments/assets/2dbd1d44-06be-4fc7-b1d9-d0a2fd87cf77" />

LoadBalancer의 생성 여부를 확인한다.

```bash
kubectl get svc -n ingress-nginx
```

- `EXTERNAL-IP`가 실제 IP 주소 또는 도메인으로 변경될 때까지 기다린다.

<img width="652" height="67" alt="image" src="https://github.com/user-attachments/assets/95de5d8b-a238-475e-b672-053486318a09" />

# 경로 기반 라우팅 설정

## 로드밸런서 접속 정보 저장

Ingress Controller의 외부 접속 주소를 환경 변수에 저장한다

```bash
export nginx_address=$(kubectl get svc -n ingress-nginx ingress-nginx-controller -o jsonpath='{.status.loadBalancer.ingress[*].hostname}')
```

```bash
echo $nginx_address
```

## Ingress 리소스 작성

생성한 Ingress Service의 EXTERNAL-IP로 들어오는 요청을 **path별로 라우팅하여 각각 다른 서비스로 연결되도록 설정할 수 있다.**

아래와 같이 Ingress 리소스를 각 path에 따라 다른 backend를 갖도록 설정하여 파일로 저장한다.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress

metadata:
  name: paaspaas-ingress
  annotations:
    # NGINX Ingress Controller 설정
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
    # TLS 설정이 필요한 경우
    # cert-manager.io/cluster-issuer: "letsencrypt-prod"
    # nginx.ingress.kubernetes.io/force-ssl-redirect: "true"

spec:
  ingressClassName: nginx

  # TLS/HTTPS 설정 (인증서 준비 후 활성화)
  # tls:
  #   - hosts:
  #       - paaspaas.kr
  #       - www.paaspaas.kr
  #     secretName: paaspaas-tls-secret

  rules:
    # 경로 기반 라우팅
    - http:
        paths:
          # Swagger UI → Gateway
          - path: /swagger-ui
            pathType: Prefix
            backend:
              service:
                name: gateway-service
                port:
                  number: 8080

          # Swagger API Docs → Gateway
          - path: /v3/api-docs
            pathType: Prefix
            backend:
              service:
                name: gateway-service
                port:
                  number: 8080

          # API 요청 → Gateway
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: gateway-service
                port:
                  number: 8080

          # 웹 UI (기본) → Front Server
          - path: /
            pathType: Prefix
            backend:
              service:
                name: front-service
                port:
                  number: 8080

```

- `pathType: Prefix`, 지정된 경로로 시작하는 모든 요청을 매칭한다
- `nginx.ingress.kubernetes.io/ssl-redirect: "false"`: TLS 설정을 하지 않았기 때문에 강제 리다이렉션을 막기 위해 추가한다. TLS를 설정할 시에는 해당 어노테이션은 제거해야 한다.

## Ingress 생성

작성한 YAML 파일로 Ingress를 생성한다.

```bash
kubectl apply -f ./path-ingress.yaml
```

생성된 Ingress를 확인한다.

```bash
kubectl get ingress
```

<img width="754" height="82" alt="image" src="https://github.com/user-attachments/assets/302bffd6-d1c9-4531-a643-79217925977d" />

## 동작 확인

각 경로로 요청을 보내 정상적으로 라우팅되는지 확인한다

```bash
# Frontend 서비스
curl http://$nginx_address/

# API Gateway
curl http://$nginx_address/api/

# Swagger UI
curl http://$nginx_address/swagger-ui/
```
