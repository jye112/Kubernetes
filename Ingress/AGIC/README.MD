# Application Gateway를 Ingress Controller로 사용

## 1. 시작하기
![image](https://user-images.githubusercontent.com/50107548/141387471-2434e5e7-325c-4f7a-84e3-cbf6189ecac4.png)
### AGIC(Application Gateway Ingress Controller)는 무엇인가?
- Kubernetes 응용 프로그램으로, Azure의 L7 Load Balancer인 Application Gateway를 AKS 클러스터의 Ingress Controller로 활용하여 클라우드 소프트웨어를 인터넷에 노출할 수 있도록 한다.
- AGIC는 NGINX Ingress Controller와는 다르게 Ingress Controller가 모두 자체 Pod 형태로만 생성되고 실행된다. 
- AGIC는 Application Gateway가 AKS 클러스터 외부에 위치해 있기 때문에 AKS 클러스터 앞에 따로 Load Balancer나 Public IP가 필요하지 않다.
- AGIC는 Application Gateway만의 기능을 사용할 수 있다는 점에서 장점이 있다.
  - URL 라우팅
  - 쿠키 기반 선호도
  - SSL/TLS 암호화
  - 공용/개인 및 하이브리드 웹 사이트 지원
  - 방화벽 (WAF)
  - 리디렉션(Redirection)
- AGIC는 In-cluster Ingress Controller에 비해 네트워크 latency가 낮고 트래픽 로드의 증가/감소에 대한 대응에 용이하다.
![image](https://user-images.githubusercontent.com/50107548/141387503-aa0b2ccf-c23a-40ec-a203-827e8b4184b8.png)



## 2. AKS와 Azure Application Gateway 생성
- 먼저 AKS와 Azure Application Gateway를 각각 생성한다.
- AKS의 네트워크는 Azure CNI로 한다.
- 두 리소스 생성 후, AKS에서 Azure Application Gateway로 접근하기 위해 AKS의 Managed Identity에 Application Gateway에 대한 기여자 권한을 부여한다.
```
./agic.ps1
```

## 3. Pod-aad 설치
### AAD-Pod-Identity는 무엇인가?
- Kubernetes에서 실행되는 애플리케이션이 Azure Active Directory(AAD)를 통해 클라우드 리소스에 안전하게 액세스 할 수 있다.
- AKS 클러스터 내에서 생성되는 워크로드들과 Azure 리소스인 Application Gateway 간 연동을 위해 필요한 작업이다.
  - Ingress Controller나 애플리케이션이 모두 Pod 형태로 배포되기 때문
```
$ kubectl create ns pod-aad

$ helm repo add aad-pod-identity https://raw.githubusercontent.com/Azure/aad-pod-identity/master/charts
$ helm repo update

# Helm 3
$ helm install aad-pod-identity aad-pod-identity/aad-pod-identity -n pod-aad
```

## 4. Application Gateway Ingress Controller 설치
```
$ helm repo add application-gateway-kubernetes-ingress https://appgwingress.blob.core.windows.net/ingress-azure-helm-package/
$ helm repo update

$ kubectl create namespace agic

$ helm install agic application-gateway-kubernetes-ingress/ingress-azure `
     --namespace agic `
     --set appgw.name=$AGW_NAME `
     --set appgw.resourceGroup=$AGW_RG `
     --set appgw.subscriptionId=$SUB_ID `
     --set appgw.usePrivateIP=false `
     --set appgw.shared=false `
     --set armAuth.type=aadPodIdentity `
     --set armAuth.identityResourceID=$MID_ID `
     --set armAuth.identityClientID=$MID_CLIENT_ID `
     --set rbac.enabled=true `
     --set verbosityLevel=3 `
     --set kubernetes.ingressClass=$INGRESS_CLASS
```

## 5. 애플리케이션 배포
- Ingress Controller을 생성했으면 서비스를 노출할 애플리케이션을 배포한다.
- 이후 애플리케이션을 공개적으로 사용할 수 있도록 Ingress 리소스를 만든다.
- Ingress는 트래픽이 어떻게 노출될지를 결정하는 일종의 규칙이며, Ingress Controller는 해당 규칙이 애플리케이션에 적용되도록 하는 컨트롤러이다.

### 5-1) HTTP를 통해 ASK 서비스 노출
- 애플리케이션 생성
```
kubectl apply -f aks-helloworld-one.yaml -n agic
```
- Ingress 생성
```
kubectl apply -f agic-ingress-http.yaml -n agic
```

### 5-2) HTTPS를 통해 AKS 서비스 노출
- DNS 영역에 A 레코드 추가
  - 먼저 Azure DNS 리소스를 생성한다.
  - Application Gateway의 Public IP 주소를 사용하여 DNS 영역에 A 레코드를 추가한다.
  ```
  az network dns record-set a add-record \
    --resource-group myResourceGroup \
    --zone-name MY_CUSTOM_DNS \
    --record-set-name agic \
    --ipv4-address MY_EXTERNAL_IP
  ```
  - Host PC의 Hosts 파일을 관리자 권한으로 수정한다.(C:\Windows\System32\drivers\etc)
  ```
  MY_EXTERNAL_IP   MY_CUSTOM_DNS
  ```

- Cert-manager 설치
  - cert-manager은 기본 k8s 인증서를 관리하는 컨트롤러이다.
  - Let 's Encrypt , HashiCorp Vault , Venafi, 간단한 서명 키 쌍 또는 자체 서명과 같은 다양한 소스에서 인증서를 발급하는 데 도움이 된다.
  ```
  kubectl label namespace agic cert-manager.io/disable-validation=true

  # Jetstack Helm Repository 추가
  helm repo add jetstack https://charts.jetstack.io
  helm repo update

  # cert-manager helm chart 설치
  helm install cert-manager jetstack/cert-manager `
  --namespace agic `
  --version v0.16.1 `
  --set installCRDs=true `
  --set nodeSelector."beta\.kubernetes\.io/os"=linux
  ```

- CA 클러스터 발급자 만들기
  - 인증서가 발급되기 전에 cert-manager에게는 Issuer 또는 ClusterIssuer 리소스가 필요하다.
  - Issuer은 단일 네임스페이스에서 작동하고 ClusterIssuer은 모든 네임스페이스에서 작동한다.
  ```
  kubectl apply -f agic-cluster-issuer.yaml -n agic
  ```

- 애플리케이션 배포
```
kubectl apply -f aks-helloworld-one.yaml -n agic
kubectl apply -f aks-helloworld-two.yaml -n agic
```

- Ingress 생성
```
kubectl apply -f agic-ingress-https.yaml -n agic
```

- cert-manager가 자동으로 인증서를 만든다.

