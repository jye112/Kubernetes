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
    - name: test-pv-1
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
    diskName: test-pv-disk
    diskURI: <생성한 Disk URI 주소>
  ```
  ```
  kubectl apply -f pv-disk-1.yaml
  ```
- PVC 생성
  - 'volumeName' 부분에 PV의 이름을 기입한다.
  ```
  kubectl apply -f pvc-disk-1.yaml
  ```
- Pod 생성
  - 'persistentVolumeClaim' 부분에 PVC의 이름을 기입한다.
  - PVC를 통해 파드와 PV가 연결되면서 파드는 PV에 붙어있는 Azure Disk를 마운트하여 사용할 수 있게 된다.
  ```
  kubectl apply -f pod-disk.yaml
  ```
  
### 2. Azure Files (Storage Account)
- Azure Files는 여러 파드에 접근 가능하다.
- Azure Storage Account 먼저 생성하고 Pod Replica 개수에 따라 File Share를 생성한다.
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
  kubectl apply -f pv-file-1.yaml
  ```
- PVC 생성
  - 'volumeName' 부분에 PV의 이름을 기입한다.
  ```
  kubectl apply -f pvc-file-1.yaml
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
kubectl apply -f pv-file-1.yaml
```
- Statefulset 생성
  - volumeClaimTemplate을 넣어서 생성하면 PVC가 Replica 개수만큼 자동 생성되면서 미리 생성한 PV에 각각 연결된다.

  
