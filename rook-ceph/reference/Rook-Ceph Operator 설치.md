## 전제 조건
- Kubernetes 1.22+
- Helm 3.x
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
Helm : v3.10.3
```
- rook-ceph
```
Repo : https://github.com/rook/rook
Branch : release-1.12 
Chart Version : 0.0.1
App Version : 0.0.1
```
- osd device
```
haiqv-dev-worker01 : /dev/vdb
haiqv-dev-worker02 : /dev/vdb
haiqv-dev-worker03 : /dev/vdb
```

## 사전 작업
- git clone
```
wget https://github.com/rook/rook/tree/release-1.12
```
- namespace 생성 (예시 : `nile-storage`)
```
kubectl create namespace nile-storage
```
- 라벨 추가
```
kubectl label ns nile-storage pod-security.kubernetes.io/enforce=privileged
```

## 설정
### Discovery Daemon
Discovery Daemon이 추가적으로 생성된다. device 추가 시 자동으로 이를 감지한다.
```yaml
## values.yaml
enableDiscoveryDaemon: true
```

## 설치
- rook-ceph operator helm chart 위치 : rook/deploy/charts/rook-ceph
- rook-ceph operator 설치
```shell
cd rook/deploy/charts/rook-ceph
helm install rook-ceph . -f values.yaml -n nile-storage
```

## 확인
Istio sidecar가 설정되어 있어 READY 상태가 다를 수 있다.
```shell
kubectl get po -n nile-storage

NAME                                         READY   STATUS     RESTARTS      AGE
rook-ceph-operator-786489648f-qnvc9          2/2     Running    1 (13d ago)   13d
```

## 참고
- [Ceph Operator Helm Chart](https://rook.io/docs/rook/latest/Helm-Charts/operator-chart/)
- [HAiQV NILE DEV Cluster 인프라 현황](https://wiki.hanwhasystems.com/confluence/pages/viewpage.action?pageId=109748249)