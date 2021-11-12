# AKS NGINX Ingress Controller
![image](https://user-images.githubusercontent.com/50107548/141392546-2fe7eed7-1b85-4ad1-934e-2a74e9914cdc.png)

## 1. 시작하기
### Ingress란 무엇인가?
- Ingress는 외부로부터 서버 내부로 유입되는 네트워크 트래픽을 의미한다.
- k8s의 Ingress는 외부에서 클러스터 내부의 애플리케이션에 접근하기 위한 일종의 관문 역할을 한다.
- Ingress는 Layer 7에서의 요청을 처리할 수 있다. 외부로부터 들어오는 요청에 대한 로드밸런싱, TLS/SSL 인증서 처리, 특정 HTTP 경로의 라우팅 등을 Ingress를 통해 자세하게 정의할 수 있다.
- Ingress를 사용하지 않으면 해당 역할을 NodePort나 ExternalIP로 할 수 있는데, 이들은 Layer 4에서의 요청을 처리하기 때문에 네트워크 요청에 대한 세부적인 로직 처리를 구현하는 데에는 한계가 있다.
- Ingress 요청을 처리하기 위한 서비스는 일반적으로 Load Balancer 타입의 서비스를 사용한다.

### Ingress Controller는 왜 필요한가?
- k8s에서 Ingress를 사용하기 위해서는 Ingress 오브젝트 파일과 Ingress Controller이 필요하다.
- Ingress는 단지 규칙을 정의하는 선언적인 오브젝트이며 외부 요청을 받아들이는 실제 서버가 아니다.
- Ingress 규칙에 기반하여 이 요청을 어떻게 처리할 지 결정하고 Ingress에 적용된 규칙을 활성화하는 역할을 하는 것이 **Ingress Controller** 이다.
- Ingress Controller 구동 방법은 두 가지가 있다.
  - 직접 구동하려면 NGINX 웹 서버 기반의 NGINX Ingress Controller를 사용한다. helm을 통해 repository를 설치할 수 있다.
  - 클라우드 플랫폼에 위임한다. (ex. AGIC)

## 2. NGINX Ingress Controller 설치
- helm으로 설치한다.
```
# Ingress 리소스를 위한 namespace 생성
kubectl create namespace ingress-basic

# Ingress-nginx Repository 추가
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx

# helm으로 NGINX Ingress Controller 배포
helm install nginx-ingress ingress-nginx/ingress-nginx `
  --namespace ingress-basic `
  --set controller.replicaCount=2 `
  --set controller.nodeSelector."beta\.kubernetes\.io/os"=linux `
  --set defaultBackend.nodeSelector."beta\.kubernetes\.io/os"=linux `
  --set controller.admissionWebhooks.patch.nodeSelector."beta\.kubernetes\.io/os"=linux
# -- set controller.admissionWebhooks.enabled=false
```

- 설치하는 동안 Azure Public IP가 Ingress Controler에 생성된다. 서비스를 확인하여 Public IP를 확인한다.


## 3. 애플리케이션 배포
- Ingress Controller 작동 여부를 확인하기 위해 애플리케이션을 배포한다.

### 3-1) HTTP를 통해 ASK 서비스 노출
- 애플리케이션 생성
```
kubectl apply -f aks-helloworld-one.yaml
kubectl apply -f aks-helloworld-two.yaml
```

- Ingress 생성
  - 애플리케이션이 실행되어도 ClusterIP 형식의 서비스로 구성되며 인터넷에 액세스할 수 없다.
  - 애플리케이션이 공개적으로 노출될 수 있도록 Ingress 리소스를 생성한다.
  - Ingress는 두 애플리케이션 중 하나로 트래픽을 라우팅하는 규칙을 구성한다.
  ```
  kubectl apply -f ingress-http.yaml -n ingress-basic
  ```
  
- 애플리케이션이 정상적으로 작동하는지 Public IP를 브라우저에 입력하여 확인한다.

### 3-2) HTTPS를 통해 ASK 서비스 노출
- DNS 영역에 A 레코드 추가
  - 먼저 Azure DNS 리소스를 생성한다.
  - NGINX 서비스의 Public IP 주소를 사용하여 DNS 영역에 A 레코드를 추가한다.
  ```
  az network dns record-set a add-record `
    --resource-group myResourceGroup `
    --zone-name MY_CUSTOM_DNS `
    --record-set-name * `
    --ipv4-address MY_EXTERNAL_IP
  ```
- Host PC의 Hosts 파일을 관리자 권한으로 수정한다.(C:\Windows\System32\drivers\etc)
```
MY_EXTERNAL_IP   hello-world-ingress.MY_CUSTOM_DNS
```
- Cert-manager 설치
  - cert-manager는 기본 k8s 인증서를 관리하는 컨트롤러이다.
  - Let 's Encrypt , HashiCorp Vault , Venafi, 간단한 서명 키 쌍 또는 자체 서명과 같은 다양한 소스에서 인증서를 발급하는 데 도움이 된다.
  ```
  kubectl label namespace ingress-basic cert-manager.io/disable-validation=true
  # cert-manager namespace 생성
  kubectl create namespace cert-manager

  # Jetstack Helm Repository 추가
  helm repo add jetstack https://charts.jetstack.io
  helm repo update

  # cert-manager helm chart 설치
  helm install cert-manager jetstack/cert-manager `
    --namespace cert-manager `
    --set installCRDs=true `
    --set nodeSelector."kubernetes\.io/os"=linux `
    --set webhook.nodeSelector."kubernetes\.io/os"=linux `
    --set cainjector.nodeSelector."kubernetes\.io/os"=linux
  ```
- CA 클러스터 발급자 만들기
  - 인증서가 발급되기 전에 cert-manager에게는 Issuer 또는 ClusterIssuer 리소스가 필요하다.
  - Issuer은 단일 네임스페이스에서 작동하고 ClusterIssuer은 모든 네임스페이스에서 작동한다.
  ```
  kubectl apply -f cluster-issuer.yaml
  ```

- 애플리케이션 생성
```
kubectl apply -f aks-helloworld-one.yaml
kubectl apply -f aks-helloworld-two.yaml
```

- Ingress 생성
```
kubectl apply -f ingress-https.yaml -n ingress-basic
```

- cert-manager가 자동으로 인증서를 만든다.


