# Azure Kubernetes(AKS)에서의 PV/PVC

## PV/PVC란 무엇인가?
- PV(Persistent Volume) : 쿠버네티스 클러스터 외부 스토리지와의 연결을 담당하는 리소스
- PVC(Persistent Volume Claim) : PV와 파드를 연결하기 위한 리소스
- PV 리소스의 네 가지 상태
  - Available : 다른 PVC에 연결되지 않은 상태, 사용 가능 상태
  - Bound : 특정 PVC에 연결됨
  - Released : 연결되었던 PVC가 해제, 리소스를 회수하지 않은 상태
  - Failed : 회수 실패


## Azure Kubernetes에서의 Static Provisioning
- Azure Disk 혹은 Azure Files(Storage Account) 리소스 먼저 생성한다.
- 먼저 생성한 외부 스토리지 리소스를 PV에 연결해야 하므로 PV도 Azure Storage 개수만큼 먼저 생성한다.
- PVC를 생성해 PV를 연결하여 파드에 볼륨을 마운트한다.

### 1. Azure Disk
- Azure Disk는 단일 파드에만 연결 가능
#### 1-1) 파드 생성할 때 Azure Disk 직접 붙여 볼륨 마운트 하는 방법
- 'volumes' 부분에 미리 생성한 Disk 정보를 기입한다.
```
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
  - image: mcr.microsoft.com/oss/nginx/nginx:1.15.5-alpine
    name: mypod
    resources:
      requests:
        cpu: 100m
        memory: 128Mi
      limits:
        cpu: 250m
        memory: 256Mi
    volumeMounts:
      - name: azure
        mountPath: /mnt/azure
  volumes: 
    - name: pv-disk
      azureDisk:
        kind: Managed
        diskName: myAKSDisk
        diskURI: <생성한 Disk URI 주소>
  ```
  
#### 1-2) PV/PVC를 생성하여 볼륨을 마운트 하는 방법
- PV 생성
  - 'azureDisk' 부분에 미리 생성한 Azure Disk 정보를 기입한다.
  ```
  azureDisk: 
    kind: Managed
    diskName: pv-disk
    diskURI: <생성한 Disk URI 주소>
  ```
  ```
  kubectl apply -f pv-disk.yaml
  ```
- PVC 생성
  - 'volumeName' 부분에 PV의 이름을 기입한다.
  ```
  kubectl apply -f pvc-disk.yaml
  ```
- Pod 생성
  - 'persistentVolumeClaim' 부분에 PVC의 이름을 기입한다.
  - PVC를 통해 파드와 PV가 연결되면서 파드는 PV에 붙어있는 Azure Disk를 마운트하여 사용할 수 있게 된다.
```
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
  - image: mcr.microsoft.com/oss/nginx/nginx:1.15.5-alpine
    name: mypod
    resources:
      requests:
        cpu: 100m
        memory: 128Mi
      limits:
        cpu: 250m
        memory: 256Mi
    volumeMounts:
      - name: azure
        mountPath: /mnt/azure
  volumes:
  - name: azure
    persistentVolumeClaim:
      claimName: pvc-disk
```
  
### 2. Azure Files (Storage Account)
- Azure Files는 여러 파드에 접근 가능하다.
- Pod Replica 개수에 따라 Azure Files를 생성한다.
- Secret을 생성한다.
```
kubectl create secret generic azure-secret --from-literal=azurestorageaccountname=$AKS_PERS_STORAGE_ACCOUNT_NAME --from-literal=azurestorageaccountkey=$STORAGE_KEY
```

#### 2-1) 파드 생성할 때 Azure Files를 직접 붙여 볼륨 마운트 하는 방법
- 'volumes' 부분에 미리 생성한 Storage Account Secret과 Share 이름을 기입한다.
```
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
  - image: mcr.microsoft.com/oss/nginx/nginx:1.15.5-alpine
    name: mypod
    resources:
      requests:
        cpu: 100m
        memory: 128Mi
      limits:
        cpu: 250m
        memory: 256Mi
    volumeMounts:
      - name: azure
        mountPath: /mnt/azure
  volumes: 
  - name: azure
    azureFile:
      secretName: azure-secret
      shareName: aksshare
      readOnly: false
```
#### 2-2) PV/PVC를 생성하여 볼륨을 마운트 하는 방법 - Deployment
- Deployment는 파드들이 한 개의 PV/PVC를 공유하므로 하나의 Azure Storage Account를 생성한다.
- PV 생성
  - 'azureFile' 부분에 미리 생성한 Azure Files 정보를 기입한다.
  ```
  azureFile: 
    secretName: azure-secret
    secretNamespace: default
    shareName: jyefileshare1
    readOnly: false
  ```
  ```
  kubectl apply -f pv-file.yaml
  ```
- PVC 생성
  - 'volumeName' 부분에 PV의 이름을 기입한다.
  ```
  kubectl apply -f pvc-file.yaml
  ```
- Deployment 생성
  - 'persistentVolumeClaim' 부분에 PVC의 이름을 기입한다.
  - PVC를 통해 파드와 PV가 연결되면서 파드는 PV에 붙어있는 Azure Files를 마운트하여 사용할 수 있게 된다.
  ```
  kubectl apply -f deployment-file.yaml
  ```

#### 2-3. PV/PVC를 생성하여 볼륨을 마운트 하는 방법 - Statefulset
- Statefulset은 파드들이 각각의 PV/PVC와 연결될 수 있으므로 Azure Files와 PV/PVC가 모두 Statefulset의 Replicas 개수만큼 생성되어야 한다.

- Azure Files 개수만큼 PV 생성
```
kubectl apply -f pv-file.yaml
```
- Statefulset 생성
  - volumeClaimTemplate을 넣어서 생성하면 PVC가 Replica 개수만큼 자동 생성되면서 미리 생성한 PV에 각각 연결된다.
  ```
  kubectl apply -f statefulset-file.yaml
  ```


## Azure Kubernetes에서의 Dynamic Provisioning
- Azure Disk 혹은 Azure Files(Storage Account) 리소스를 먼저 생성하지 않아도 동적으로 생성된다.
- 미리 여러 스토리지를 PV에 등록하고 스토리지 기능 및 분류 정보를 담고 있는 Storage Class만 정의한다.
- 원하는 스토리지 정보를 담고 있는 Storage Class를 선택하여 PVC를 생성하면 PV는 자동으로 프로비저닝 된다.
- Stateless한 파드를 배포할 경우, Disk는 단일 파드를 사용할 때, Files는 여러 파드에서 동시에 접근할 때 사용한다.

- Storage Class
  - PV로 사용할 스토리지의 클래스(분류, 정책, 종류)에 대한 정보를 정의할 수 있는 리소스이다.
  - PVC 생성할 때 혹은 파드 생성할 때 Storage Class Name을 선언하면, 선언한 Storage Class 조건에 맞는 PV가 자동으로 생성된다.
  - Azure에서는 기본적으로 4개의 Storage Class를 제공한다. (azurefile, azurefile-premium, default/azuredisk, managed-premium)

### 1. Azure Disk
#### 1-1. PVC 생성할 때 Storage Class Name 명시
- PVC 이름은 지정된대로 생성되지만 PV 이름은 랜덤하게 생성된다.
- PVC는 아직 PVC에 연결되지 않은, 가장 먼저 생성된 PV를 찾아 연결한다.
```
kubectl apply -f pvc-disk-sc.yaml
```
#### 1-2. 파드 생성할 떄 volumeClaimTemplates 만들어 Storage Class Name 명시 -> Statefulset만 가능
- PV/PVC 모두 랜덤하게 생성됨
```
kubectl apply -f statefulset-disk-sc.yaml
```

### 2. Azure Files (Storage Account)
#### 2-1. PVC 생성할 때 Storage Class Name 명시
- PVC 이름은 지정된대로 생성되지만 PV 이름은 랜덤하게 생성된다.
- PVC는 아직 PVC에 연결되지 않은, 가장 먼저 생성된 PV를 찾아 연결된다.
```
kubectl apply -f pvc-file-sc.yaml
```
#### 2-2. 파드 생성할 때 volumeClaimTemplates 만들어 Storage Class Name 명시 -> Statefulset만 가능
- PV/PVC 이름 모두 랜덤하게 생성
```
kubectl apply -f statefulset-file-sc.yaml
```

