## History
|||||
|---|---|---|---|
|2023-11-24일|로컬 Vagrant로 7 cpus, 10GB memory로 구성했으나 자원 부족으로 컴포넌트 전체 설치 실패함|||


> 최초 OSD's 영역을 위해 raw device 단위로 필요 ex) vdb
> (할당 후 FSTYPE이 ceph_bluestore 로 등록? or 변경?
> Storage Class 'ceph-block' 으로 할당 받은 k8S PV 는 실제 물리장치의 **disk type** 으로 생성됨

* ceph-OSD
> Object Storage Daemon, 데이터 저장하는 곳으로, OSD Pod 로 Rook-Ceph을 통해 PVC가 Bound 됨
