# 개요
> Rook is an open source cloud-native storage orchestrator for Kubernetes, providing the platform, framework, and support for Ceph storage to natively integrate with Kubernetes.

# 1. Rook-Ceph Operator 설치

## 1.1 설치 전 사전 작업
### 1.1.1 Git Clone
```bash
git clone https://github.com/rook/rook
```

### 1.1.2 namespace 생성
```bash

kubectl create namespace nile-storage
```

### 1.1.3 해당 namespace에 pod security policy 적용
```bash
kubectl label ns nile-storage pod-security.kubernetes.io/enforce=privileged
```

### 1.1.4  rook-ceph Operator Values.yaml 수정
> 파일위치: rook/deploy/charts/rook-ceph/values.yaml

* Discovery Daemon 활성화
> 이 추가적으로 생성된다. device 추가 시 자동으로 이를 감지한다.

```bash
# 파일위치: rook/deploy/charts/rook-ceph/values.yaml
# 기본값: enableDiscoveryDaemon: false
# 아래 내용으로 수정
enableDiscoveryDaemon: true
```

* (싱글노드일경우만) provisionerReplicas 수정
```bash
# 기본값: provisionerReplicas: 2
# 아래 내용으로 수정
provisionerReplicas: 1
```
## 1.2 설치
```bash
cd rook/deploy/charts/rook-ceph
helm install rook-ceph . -f values.yaml -n nile-storage
```

# 2. Rook-Ceph Cluster 설치
## 2.1 설치 전 사전 작업

* osd 대상 device 세팅 
> Kubernetes worker node에 Ceph가 Storage로 사용할 Raw device가 있어야 한다. 아래와 같이 **`lsblk -f`** 명령을 수행하여 깡통 상태의 Storage가 있는지 확인한다. 아래 명령 결과에서 **vdb**가 아무도 사용하지 않는, 그리고 **formatting도 하지 않은 완전한 깡통 상태의 Storage** 이다.

```bash
$ lsblk -f
vda                                                                              
├─vda1  ext4           cloudimg-rootfs  ...  258.3G    11% /
├─vda14                                                                       
└─vda15 vfat           UEFI             ...  99.2M     5% /boot/efi
vdb ## <- 이 "vdb" 저장 장치와 같이 아무 설정도 없는 상태의 Storage가 있어야 한다.

# 또는

$ fdisk -l /dev/vdb
**Disk /dev/vdb: 1000 GiB, 1073741824000 bytes, 2097152000 sectors**
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
## 아래 파티션 정보가 없어야 한다.

## 만약 osd 대상 device가 사용하던 장비라면, 다음 명령어를 통해 device를 초기화한다.

# device의 파티션 정보 제거
sgdisk --zap-all /dev/vdb
```

### 1.1 operatorNamespace 수정
> rook-ceph의 default namespace는 rook-ceph이다. 만약, operator 설치 시 다른 namespace를 사용했다면, rook-ceph cluster에서  operatorNamespace를 operator와 동일한 namespace로 변경한다.
```bash
# 파일위치: rook/deploy/charts/rook-ceph-cluster/values.yaml
# 기본값: operatorNamespace: rook-ceph
# 아래 내용으로 수정
operatorNamespace: nile-storage
```

### 1.2 Ceph Image 변경
> ceph v17.2.6에서는 default namespace가 아닌 다른 namespace를 이용할 경우 dashboard의 일부 기능에 이슈가 발생한다. 따라서 ceph 버전을 16으로 낮춰서 설치를 진행한다.
```yaml
# 파일 위치는 위와 동일하여 생략
# 아래 내용으로 수정
toolbox:
  image: quay.io/ceph/ceph:v16.2.10

cephClusterSpec:
  cephVersion:
    image: quay.io/ceph/ceph:v16.2.10
```

### 1.3 pod resource
```text
클러스터 배포 시 하드웨어 요건 및 권장사항은 다음과 같다.

- 하드웨어 요건
    - `OSD`
        - CPU: 2GHz CPU 2 core 이상 권장
        - Memory: 최소 2GB + 1TB당 1GB 이상의 메모리 권장
            - ex) 2TB disk 기반 OSD -> 4GB(2GB+2GB) memory 이상 권장
        - Disk: 최소 10GB 이상 디스크 권장, 데이터 저장용으로 큰 용량의 디바이스를 사용하는 게 좋음
    - `MON`
        - CPU: 2GHz CPU 1 core 이상 권장
        - Memory: 2GB 메모리 이상 권장
        - Disk: 10GB 디스크 이상 권장
    - `MGR`
        - CPU: 2GHz CPU 1 core 이상 권장
        - Memory: 1GB 메모리 이상 권장
    - `MDS`
        - CPU: 2GHz CPU 4 core 이상 권장
        - Memory: 4GB 메모리 이상 권장
- 권장사항
    - 각 노드마다 OSD를 배포하도록 권장 (Taint 걸린 host 없는 걸 확인해야함)
    - 총 OSD 개수는 3개 이상으로 권장
    - CephFS 및 RBD pool 설정 시 Replication 개수 3개 권장

```

```yaml
# 아래 내용으로 수정
cephClusterSpec:
...(중략)
  resources:
    mgr:
      limits:
        cpu: "1000m"
        memory: "1Gi"
      requests:
        cpu: "1000m"
        memory: "1Gi"
    mon:
      limits:
        cpu: "1000m"
        memory: "2Gi"
      requests:
        cpu: "1000m"
        memory: "2Gi"
    osd:
      limits:
        cpu: "2000m"
        memory: "6Gi"
      requests:
        cpu: "2000m"
        memory: "6Gi"
    prepareosd:
      # limits: It is not recommended to set limits on the OSD prepare job
      #         since it's a one-time burst for memory that must be allowed to
      #         complete without an OOM kill.  Note however that if a k8s
      #         limitRange guardrail is defined external to Rook, the lack of
      #         a limit here may result in a sync failure, in which case a
      #         limit should be added.  1200Mi may suffice for up to 15Ti
      #         OSDs ; for larger devices 2Gi may be required.
      #         cf. https://github.com/rook/rook/pull/11103
      requests:
        cpu: "500m"
        memory: "50Mi"
    mgr-sidecar:
      limits:
        cpu: "500m"
        memory: "100Mi"
      requests:
        cpu: "100m"
        memory: "40Mi"
    crashcollector:
      limits:
        cpu: "500m"
        memory: "60Mi"
      requests:
        cpu: "100m"
        memory: "60Mi"
    logcollector:
      limits:
        cpu: "500m"
        memory: "1Gi"
      requests:
        cpu: "100m"
        memory: "100Mi"
    cleanup:
      limits:
        cpu: "500m"
        memory: "1Gi"
      requests:
        cpu: "500m"
        memory: "100Mi"
    exporter:
      limits:
        cpu: "250m"
        memory: "128Mi"
      requests:
        cpu: "50m"
        memory: "50Mi"
```

### 1.4 dashboard 활성화
> 대시보드는 기본적으로 비활성되어 있으며, 아래와 같은 설정으로 활성화한다.
```yaml
# 아래 내용으로 수정
cephClusterSpec:
...(중략)
  dashboard:
    enabled: true
    ssl: false
```

### 1.5-1 rook 모듈 추가
> rook 모듈이 기본적으로 비활성화 되어 있으며, 대시보드에서 일부 기능을 사용할 수 없다. (host 정보 등) 이를 활성화하여 기능을 사용할 수 있도록 값을 변경한다.

```yaml
# 아래 내용으로 추가
cephClusterSpec:
...(중략)
  mgr:
    ...(중략)
    modules:
      - name: rook
        enabled: true
```

### 1.5-2 rook 모듈 추가 (2.5-1 Alternative)
> rook-ceph tool로 rook 모듈 활성화 하는 방법
```bash
kubectl exec -n nile-storage $(kubectl get po -n nile-storage | grep tool | awk '{print $1}') -it -- bash

(ceps tool)$ ceph mgr module enable rook
(ceps tool)$ ceph orch set backend rook
(ceps tool)$ ceph orch status

Backend: rook
Available: Yes
```

### 1.6 OSD device 식별

- 2.6.1 device 자동 식별
```yaml
cephClusterSpec:
...(중략)
  storage:
    useAllNodes: true
    useAllDevices: true
```
- 2.6.2 device 조건 식별
```yaml
cephClusterSpec:
...(중략)
  storage:
    useAllNodes: false
    useAllDevices: false
    nodes:
      - name: "haiqv-dev-worker01"
        devices: 
          - name: "vdb"
      - name: "haiqv-dev-worker02"
        devices: 
          - name: "vdb"
      - name: "haiqv-dev-worker03"
        devices: 
          - name: "vdb"		  
```
- 2.6.3 single node, single osd
  single osd인 경우 모든 replicated size를 3 -> 1로 변경한다.
```yaml


cephClusterSpec:
	mon:
    # Set the number of mons to be started. Generally recommended to be 3.
    # For highest availability, an odd number of mons should be specified.
    count: 1
    # The mons should be on unique nodes. For production, at least 3 nodes are recommended for this reason.
    # Mons should only be allowed on the same node for test environments where data loss is acceptable.
    allowMultiplePerNode: false

  mgr:
    # When higher availability of the mgr is needed, increase the count to 2.
    # In that case, one mgr will be active and one in standby. When Ceph updates which
    # mgr is active, Rook will update the mgr services to match the active mgr.
    count: 1
    
...(중략)

  storage:
    useAllNodes: false
    useAllDevices: false
    nodes:
      - name: "haiqv-dev-worker01"
        devices: 
          - name: "vdb"
...(중략)
cephBlockPools:
  - name: ceph-blockpool
    spec:
      failureDomain: host
      replicated:
        size: 1
...(중략)
cephFileSystems:
  - name: ceph-filesystem
    spec:
      metadataPool:
        replicated:
          size: 1
      dataPools:
        - failureDomain: host
          replicated:
            size: 1		
...(중략)
cephObjectStores:
  - name: ceph-objectstore
    spec:
      metadataPool:
        failureDomain: host
        replicated:
          size: 1 			
```

## 2.2 설치
```bash
cd rook/deploy/charts/rook-ceph-cluster
helm install rook-ceph-cluster . -f values.yaml -n nile-storage


# Upgrade
helm upgrade -f values.yaml rook-ceph-cluster . -n nile-storage

```

## 2.3 확인
> Istio sidecar가 설정되어 있어 READY 상태가 다를 수 있다. rook-discover 데몬은 operator 설정에서 `enableDiscoveryDaemon` 설정을 `true` 로 했을 때 실행된다.
```shell
kubectl get po -n nile-storage

NAME                                                           READY   STATUS  
csi-cephfsplugin-646fv                                         2/2     Running 
csi-cephfsplugin-mdgk6                                         2/2     Running 
csi-cephfsplugin-njmhz                                         2/2     Running 
csi-cephfsplugin-nkrwq                                         2/2     Running 
csi-cephfsplugin-provisioner-574df79445-cmfkk                  6/6     Running 
csi-cephfsplugin-provisioner-574df79445-n6m9b                  6/6     Running 
csi-cephfsplugin-rgdft                                         2/2     Running 
csi-cephfsplugin-sv6rn                                         2/2     Running 
csi-rbdplugin-4nhm2                                            2/2     Running 
csi-rbdplugin-8xvr6                                            2/2     Running 
csi-rbdplugin-provisioner-5444dff597-dgmv4                     6/6     Running 
csi-rbdplugin-provisioner-5444dff597-vpw5k                     6/6     Running 
csi-rbdplugin-q7ldt                                            2/2     Running 
csi-rbdplugin-s87cd                                            2/2     Running 
csi-rbdplugin-x2wrj                                            2/2     Running 
csi-rbdplugin-xxsvm                                            2/2     Running 
rook-ceph-crashcollector-haiqv-dev-worker01-84589d86d8-m4tzm   2/2     Running 
rook-ceph-crashcollector-haiqv-dev-worker02-5498dfd9dc-7zjff   2/2     Running 
rook-ceph-crashcollector-haiqv-dev-worker03-54b5f94778-8nfxl   2/2     Running 
rook-ceph-mds-ceph-filesystem-a-584c84569c-t8dg5               3/3     Running 
rook-ceph-mds-ceph-filesystem-b-75cff5c9d5-jww24               3/3     Running 
rook-ceph-mgr-a-d57cd7c5-zl24s                                 4/4     Running 
rook-ceph-mgr-b-66d8854498-bdfjw                               4/4     Running 
rook-ceph-mon-a-77bf4cc8c-qg48b                                3/3     Running 
rook-ceph-mon-b-7d87b9849f-wlv5d                               3/3     Running 
rook-ceph-mon-c-7466966cd7-lsvdw                               3/3     Running 
rook-ceph-operator-786489648f-qnvc9                            2/2     Running 
rook-ceph-osd-0-7d744fd9c6-k828m                               3/3     Running 
rook-ceph-osd-1-d46ccb6df-cmjdh                                3/3     Running 
rook-ceph-osd-2-57fcc4c5ff-6q6nn                               3/3     Running 
rook-ceph-osd-prepare-haiqv-dev-master01-lm972                 1/2     NotReady
rook-ceph-osd-prepare-haiqv-dev-master02-sqrs5                 1/2     NotReady
rook-ceph-osd-prepare-haiqv-dev-master03-t6l5c                 1/2     NotReady
rook-ceph-osd-prepare-haiqv-dev-worker01-jpvzv                 1/2     NotReady
rook-ceph-osd-prepare-haiqv-dev-worker02-4j99c                 1/2     NotReady
rook-ceph-osd-prepare-haiqv-dev-worker03-vwvwb                 1/2     NotReady
rook-ceph-rgw-ceph-objectstore-a-5884bf96cd-4s47k              2/2     Running  
rook-ceph-rgw-ceph-objectstore-a-5884bf96cd-sknvv              2/2     Running   
rook-ceph-tools-979b48995-8wnxc                                2/2     Running 
rook-discover-lqgch                                            2/2     Running 
rook-discover-lrjmz                                            2/2     Running 
```

# 3. 참고
- [Ceph Cluster Helm Chart](https://rook.io/docs/rook/latest/Helm-Charts/ceph-cluster-chart/#configuration)
- [HAiQV NILE Storage 구성 1안 참고](https://wiki.hanwhasystems.com/confluence/display/AILAB/HAiQV+NILE+Storage+Architecture)
- [하드웨어 요건 및 권장사항](https://github.com/tmax-cloud/hypercloud-sds/blob/master/docs/ceph-cluster-setting.md)
- [HAiQV NILE DEV Cluster 인프라 현황](https://wiki.hanwhasystems.com/confluence/pages/viewpage.action?pageId=109748249)

