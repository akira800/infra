## 환경
- server info
```shell
kubectl get node -o wide

NAME                 INTERNAL-IP    OS-IMAGE          
haiqv-dev-master01   172.16.57.60   Ubuntu 20.04.6 LTS
haiqv-dev-master02   172.16.57.61   Ubuntu 20.04.6 LTS
haiqv-dev-master03   172.16.57.62   Ubuntu 20.04.6 LTS
haiqv-dev-worker01   172.16.57.63   Ubuntu 20.04.2 LTS
haiqv-dev-worker02   172.16.57.64   Ubuntu 20.04.2 LTS
haiqv-dev-worker03   172.16.57.65   Ubuntu 20.04.2 LTS
```
- version info
```shell
Kubernetes : v1.25.6
rook-ceph : v16.2.10
```
- ceph storage class (storageclass명에 포함된 정규식 또는 명칭 예시 : `ceph`)
```shell
kubectl get sc | grep ceph

ceph-block (default)   nile-storage.rbd.csi.ceph.com   
ceph-bucket            nile-storage.ceph.rook.io/bucket
ceph-filesystem        nile-storage.cephfs.csi.ceph.com
```
- ceph 대시보드 url
```

```