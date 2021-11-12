# Azure Kubernetes with Azure Key vault

![AKS with Azure Key vault](https://i.stack.imgur.com/83ujN.png)

### 1. Azure Key vault 생성 및 액세스 정책 설정
- Azure 포털에서 Key Vault 리소스를 생성합니다.
- Azure Key vault에 Certificate, Secret, Key 등을 추가하기 위해서는 액세스 정책에 사용자를 추가해야 합니다.
- AKS에서 Azure Key vault에 접근하기 위해 액세스 정책에 AKS Managed Identity를 추가합니다.
- 액세스 정책 설정 후 Key vault에서 Secret을 생성합니다.
- Secret에는 DB의 Username, Password와 같은 시크릿한 값들을 추가합니다.


### 2. 클러스터에 Azure Key Vault Controller 설치
- Azure Key vault의 Secret 값을 AKS에 가져오기 위한 Azure Key vault Controller을 설치합니다.
```
kubectl create ns akv2k8s 
helm repo add spv-charts http://charts.spvapi.no
helm repo update
helm install akv2k8s spv-charts/akv2k8s --namespace akv2k8s
```

### 3. 클러스터에 AzureKeyVaultSecret 생성
- Azure Key vault에 추가한 Secret 값을 클러스터 내에 Secret이라는 쿠버네티스 리소스로 생성하기 위한 과정입니다.
- 위에서 설치한 Controller가 Azure Key vault에서 Secret 값을 가져와 클러스터 내에 배포합니다.
```
kubectl apply -f akvs.yaml
```

### 4. 애플리케이션 배포
- PaaS DB를 연결하는 애플리케이션을 배포합니다.
- 'env' 부분에 DB 정보를 기입하게 되는데, 이때 Username과 Password와 같은 시크릿한 값들은 이미 클러스터 내에 Secret 리소스로 저장해두었기 때문에 해당 값들을 직접 기입하는 것이 아니라 각각의 Secret 이름만 적어주면 됩니다.
```
kubectl apply -f db-app.yaml 
```

```
- name: db_user
  valueFrom: 
    secretKeyRef:
      name: akv-secret-un  # 클러스터 내에 생성된 Secret 이름
      key: username        # Secret에서 가져올 값
- name: db_passwd
  valueFrom:
    secretKeyRef:
      name: akv-secret-pw  # 클러스터 내에 생성된 Secret 이름
      key: password        # Secret에서 가져올 값
```

