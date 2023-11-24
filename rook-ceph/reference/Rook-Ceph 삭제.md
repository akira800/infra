## 환경
- namesapce : nile-storage
- 설치 방법 : helm
- 설치 정보 (namespace 예시 : `nile-storage`)
```shell
helm list -n nile-storage

NAME                NAMESPACE   
rook-ceph           nile-storage ## operator
rook-ceph-cluster   nile-storage
```
- ceph storage class (storageclass명에 포함된 정규식 또는 명칭 예시 : `ceph`)
```shell
kubectl get sc | grep ceph

ceph-block (default)   nile-storage.rbd.csi.ceph.com   
ceph-bucket            nile-storage.ceph.rook.io/bucket
ceph-filesystem        nile-storage.cephfs.csi.ceph.com
```
- osd 구성 (namespace 예시 : `nile-storage`)
```shell
kubectl get po -n nile-storage -o name | grep tool | xargs -I {} kubectl exec {} -n nile-storage -i -- ceph osd status

ID  HOST                 USED  AVAIL ...
 0  haiqv-dev-worker01  28.1G   971G 
 1  haiqv-dev-worker02  28.1G   971G 
 2  haiqv-dev-worker03  28.1G   971G
```
## 사전 작업
ceph storage class를 사용하는 어플리케이션을 미리 종료한다. 

### PVC 삭제
- ceph storage class 를 사용하는 pvc list 출력.
```shell
STORAGE_CLASS_NAMES=$(kubectl get sc -o jsonpath="{range .items[*].metadata}{'\''}{.name}{'\' '}{end}" | grep ceph)

kubectl get pvc -A -o jsonpath="{range .items[?(@.spec.storageClassName in $STORAGE_CLASS_NAMES)]}{.metadata.name}{' -n '}{.metadata.namespace}{'\n'}{end}"
```
- 위 결과 pvc list 삭제.
```shell
kubectl delete pvc [pvc명] -n [namespace명]
```

- [참고] pvc 전체 삭제 스크립트
```shell
#!/bin/sh

STORAGE_CLASS_NAMES=$(kubectl get sc -o jsonpath="{range .items[*].metadata}{'\''}{.name}{'\' '}{end}" | grep ceph)

kubectl get pvc -A -o jsonpath="{range .items[?(@.spec.storageClassName in $STORAGE_CLASS_NAMES)]}{.metadata.name}{' -n '}{.metadata.namespace}{'\n'}{end}"|xargs kubectl delete pvc 
```
### PV 삭제
- ceph storage class 를 사용하는 pv list 출력.
```shell
STORAGE_CLASS_NAMES=$(kubectl get sc -o jsonpath="{range .items[*].metadata}{'\''}{.name}{'\' '}{end}" | grep ceph)

kubectl get pv -A -o jsonpath="{range .items[?(@.spec.storageClassName in $STORAGE_CLASS_NAMES)]}{.metadata.name}{'\n'}{end}"
```
- 위 결과 pv list 삭제.
```shell
kubectl delete pv [pv명] -n [namespace명]
```
- [참고] pv 전체 삭제 스크립트
```shell
#!/bin/sh

STORAGE_CLASS_NAMES=$(kubectl get sc -o jsonpath="{range .items[*].metadata}{'\''}{.name}{'\' '}{end}" | grep ceph)

kubectl get pv -A -o jsonpath="{range .items[?(@.spec.storageClassName in $STORAGE_CLASS_NAMES)]}{.metadata.name}{'\n'}{end}" | xargs kubectl delete pv
```

## rook-ceph 삭제
### helm chart 삭제
- rook-ceph Cluster와 Operator helm을 삭제한다.
```shell
helm uninstall -n [namespace명] [rook-ceph Cluster명]
helm uninstall -n [namespace명] [rook-ceph Ooperator명]

ex)
helm uninstall -n nile-storage rook-ceph-cluster
helm uninstall -n nile-storage rook-ceph
```

###  CRD 삭제
- helm을 이용해 삭제를 진행해도 rook-ceph에서 사용되는 CRD는 삭제되지 않으며, 직접 해당하는 CRD를 삭제를 진행한다.
- CRD 확인
```shell
kubectl get crd -oname | grep -E 'rook|ceph|objectbucket'
```
- CRD를 이용한 자원 확인
```shell
kubectl get crd | grep -E 'rook|ceph|objectbucket' | awk '{print $1}' | xargs -I {} kubectl get {} -A -o jsonpath="{range .items[*]}{.metadata.finalizers[*]}{' '}{.metadata.name}{' -n '}{.metadata.namespace}{'\n'}{end}"
```
- 자원 삭제
```shell
kubectl get crd | grep -E 'rook|ceph|objectbucket' | awk '{print $1}' | xargs -I {} kubectl get {} -A -o jsonpath="{range .items[*]}{.metadata.finalizers[*]}{' '}{.metadata.name}{' -n '}{.metadata.namespace}{'\n'}{end}" | xargs -n 4 kubectl delete

ctrl+c

kubectl get crd | grep -E 'rook|ceph|objectbucket' | awk '{print $1}' | xargs -I {} kubectl get {} -A -o jsonpath="{range .items[*]}{.metadata.finalizers[*]}{' '}{.metadata.name}{' -n '}{.metadata.namespace}{'\n'}{end}" | xargs -n 4 kubectl patch -p '{"metadata":{"finalizers":[]}}' --type=merge
```
- CRD 삭제
```shell
kubectl get crd -oname | grep -E 'rook|ceph|objectbucket' | xargs kubectl delete
```
### 확인
CRD가 정상적으로 삭제될 때 까지 대기한다. CRD가 삭제되지 않은 상태에서 rook-ceph 재설치 시 아래와 같은 이슈가 발생한다.
- 이슈 내용
```
기존 설치에서 osd 넘버링이 0-2이고, crd를 제대로 삭제하지 않은상태에서 rook-ceph 재설치 시 osd 넘버링이 3-5로 설정된다.
```
- crd 확인
```shell
while [ "$(kubectl get crd -o name | grep -E 'rook|ceph|objectbucket' | wc -l)" -gt 0 ]; do echo "Count: $(kubectl get crd -o name | grep -E 'rook|ceph|objectbucket' | wc -l)"; sleep 5; done
```

## 서버 작업
Uninstall 작업 후, 초기화를 위해 다음 작업들을 수행해야합니다. 
### directory 정리
k8s Cluster의 **모든 노드**에서 `/var/lib/rook` directory를 삭제합니다. haiqv-dev-master01~03, haiqv-dev-worker01~03
```shell
rm -rf /var/lib/rook/
```
### device 정리
해당 작업은 osd 볼륨으로 사용된 서버에서 작업을 진행한다. osd를 사용한 호스트 정보는 haiqv-dev-worker01~03이다.
- osd device 확인
```shell
ceph osd metadata <OSD_ID>

ex)
ceph osd metadata osd.0 | grep "/dev"
ceph osd metadata osd.1 | grep "/dev"
ceph osd metadata osd.2 | grep "/dev"
```
- device 초기화 (device 예시: `vdb`)
```shell
# device의 파티션 정보 제거
sgdisk --zap-all /dev/vdb

# device mapper에 남아있는 ceph-volume 정보 제거 (각 노드당 한 번씩만 수행하면 됨)
ls /dev/mapper/ceph-* | xargs -I% -- dmsetup remove %

# /dev에 남아있는 찌꺼기 파일 제거
rm -rf /dev/ceph-*
```
