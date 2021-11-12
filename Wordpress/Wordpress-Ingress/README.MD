![Wordpress_Architecture](https://user-images.githubusercontent.com/50107548/141408380-0fea52a2-fe0d-4980-83dd-b60e13d3c53e.png)

# Configuration
## 1. Ingress
- 클러스터 외부의 네트워크로부터 액세스 받기 위한 서비스로 L7 로드밸런서에 해당한다.
- Ingress Controller는 외부로부터 네트워크 요청을 수신했을 때 Ingress 규칙에 기반하여 이 요청을 어떻게 처리할지 결정하고 이를 통해 Ingress에 적용된 규칙이 활성화된다.
- External Public IP가 붙어 서비스된다.


## 2. Wordpress (Frontend - Web)
- Service
  - ClusterIP 사용 이유 : Ingress를 통해 외부에서 내부로 접속하기 때문에 Web Service에 내부 IP인 ClusterIP를 사용한다.
- Deployment
  - Wordpress가 배포된 것을 확인할 수 있는 Web Layer 이다.
  - Deployment는 Stateless의 속성을 가지고 있다.
  - Stateless는 파드 간 볼륨을 공유하기 때문에 쓰기 작업 시 충돌이 일어날 수 있다.
  - 하지만 Web Layer의 데이터들은 컨텐츠 파일이기 때문에 읽기 작업만 수행하면 되므로 쓰기 작업에 따른 충돌이 큰 문제가 되지는 않는다.
- Storage
  - Deployment는 모든 파드들이 하나의 PVC와 PV를 공유한다. 따라서 파드끼리 데이터들을 서로 공유할 수 있다.
  - Deployment의 경우 Stateless한 성격을 가지고 있기 때문에 Azure Disk를 사용하면 단일 파드에만 볼륨이 붙는다.
  - 따라서 Azure Files를 사용하여 모든 파드에 볼륨이 붙을 수 있도록 한다.

## 3. MySQL (Backend - Database)
- Service
  - Headless인 이유 : Headless는 각각의 파드에 내부 IP를 할당할 수 있기 때문에 파드 개개인의 관리가 용이하다.
- StatefulSet
  - Wordpress 배포 시 필요한 데이터를 포함한 Database Layer 이다.
  - Statefulset은 파드를 재실행해도 상태 값이 유지되며 PV/PVC를 각각의 파드에 1:1로 배치할 수 있다.
  - 따라서 파드 간 볼륨 공유에 따른 충돌 발생을 막을 수 있다. (만약 Stateless라면 볼륨이 공유되어 쓰기 작업 시 충돌이 일어날 수 있다.)
- Storage
  - StatefulSet을 사용하면 파드 하나 당 각각의 PV/PVC가 생성된다.
  - 따라서 담일 파드에만 붙는 성격을 가진 Azure Disk를 사용해도 무방핟.
  - Azure에서는 기본적으로 4개의 Storage Class를 제공한다. -> "default" 사용
  - Storage Class가 "default"인 경우 Azure Disk가 managed 된다.
