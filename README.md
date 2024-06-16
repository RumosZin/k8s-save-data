# k8s-save-data

[쿠버네티스 입문:90가지 예제로 배우는 컨테이너 관리 자동화 표준] 14장 데이터 저장 단원의 실습 내용입니다. 

- 책의 버전과 달라진 명령어, 결과 화면을 공유합니다.
- 교재와 다른 실습을 통해 노드 내에서 수정한 파일을 저장해봅시다.
- 준비한 퀴즈를 통해 알고 있는 내용을 확인합시다.

## ✅ hostPath

1. 현재 클러스터에 등록된 노드들의 목록을 출력한다. 만약 클러스터가 구성되어 있으면, 하나 이상의 노드가 나타난다.

```shell
$ kubectl get nodes
NAME             STATUS   ROLES           AGE   VERSION
docker-desktop   Ready    control-plane   54d   v1.27.2

$ kubectl get pods
NAME                                    READY   STATUS                 RESTARTS      AGE
...
yunjinserver 
```

2. 사용자 노드에서 `index.html`를 연다.
- 각자의 환경에서 사용자 클러스터의 단일 노드에 연결되는 shell을 연다.
- 윈도우의 경우 아래와 같은 방법으로 열여서 `index.html`을 생성한다.

```shell
$ kubectl exec -it yunjinserver -- /bin/sh

# mkdir /mnt/data
# sh -c "echo Hello from k8s storage" > /mnt/data/index.html
```

3. `index.html` 파일이 존재하는지 테스트합니다.

```shell
# cat /mnt/data/index.html

Hello from k8s storage
```

</br>


### /volume/volume-hostpath.yml

1. /volume/volume-hostpath.yml 파일을 작성하고 클러스터에 적용한다.

```shell
$ kubectl apply -f volume-hostpath.yaml
pod/kubernetes-hostpath-pod created
```

2. /test-volume 디렉터리에 test.txt 파일 생성 후 exit 한다.

```shell
$ kubectl exec kubernetes-hostpath-pod -it sh
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
~ # cd /test-volume
/test-volume # touch test.txt
/test-volume # ls
test.txt
/test-volume # exit
```

3. 컨테이너를 종료하고 호스트의 /tmp 디렉터리를 확인한다.
- `test.txt` 파일이 남아있다.
- **volume-hostpath.yml에서 설정한 내용대로, 컨테이너 안 /test-volume 디렉터리는 컨테이너 외부 호스트의 /tmp 디렉터리를 마운트 했고 데이터를 보존한다.**

```shell
$ kubectl delete pod kubernetes-hostpath-pod
pod "kubernetes-hostpath-pod" deleted

$ ls /tmp/
test.txt
```
<br>

## ✅ NFS

1. 노드 하나에 NFS 서버를 설정한 후 공유해서 사용한다.
- 외부 볼륨이 아닌 hostPath 볼륨으로 NFS 서버를 만들고, 다른 파드에서 해당 파드의 볼륨을 마운트한다.
- /volume/volume-nfsserver.yml

2. deployment를 생성하고, 실행한 컨테이너의 IP를 확인한다.
```shell
$ kubectl apply -f volume-nfsserver.yaml
deployment.apps/nfs-server created

$ kubectl get pods -o wide -l app=nfs-server
NAME                          READY   STATUS    RESTARTS   AGE   IP           NODE             NOMINATED NODE   READINESS GATES
nfs-server-84fd4c5858-bvrtc   1/1     Running   0          19s   10.1.0.172   docker-desktop   <none>           <none>
```

3. NFS 서버에 접속할 클라이언트 앱 컨테이너를 설정한다.
- /volume/volume-nfsapp.yml

4. 파드 2개가 NFS 볼륨을 사용할 수 있는 상태로 실행한다.

```shell
$ kubectl apply -f volume-nfsapp.yaml
deployment.apps/kubernetes-nfsapp-pod created

$ kubectl get pods
NAME                                     READY   STATUS                 RESTARTS        AGE
kubernetes-nfsapp-pod-85689b5fbd-fb8fh   0/1     ContainerCreating      0               62s
kubernetes-nfsapp-pod-85689b5fbd-ndtx8   0/1     ContainerCreating      0               62s
```

5. 이 파드들 중 하나에 접속해서 파일을 변경해본다.
- index.html은 nfs-server가 자동으로 생성한다.

```shell
$ kubectl exec it kubernetes-nfsapp-pod-6df5b4f6cf-m2xvp sh
~ # cd /test-nfs/
/test-nfs # ls
index.html
```

6. index.html을 vi 편집기로 열어서 간단하게 수정하고, exit 명령어로 컨테이너에서 빠져나온다.
- /tmp/index.html을 확인해보면, 내용이 수정된 것을 알 수 있다.

```shell
/test-nfs # vi index.html
/test-nfs # exit

$ ca /tmp/index.html
Hello from NFS!
modify
```

## ✅ Persistent Volume Template

### /volume/pv-hostpath.yml

1. /volume/pv-hostpath.yml을 적용한다.

```shell
[root@k8s-master **]# kubectl apply -f pv-hostname.yaml 
persistentvolume/pv-hostpath created
```

2. status를 확인한다.
- status가 available이면 잘 적용된 것이다.
- 특정 PVC에 연결된 bound, PVC는 삭제되었고 PV는 아직 초기화 되지 않은 released, 자동 초기화를 실패한 failed

```shell
[root@k8s-master **]# kubectl get pv
NAME          CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
pv-hostpath   2Gi        RWO            Delete           Available           manual                  16m
```

### 퀴즈

/volume/pv-hostpath.yml에서 storage 용량 설정을 2Gi로 하였고, 내가 현재 사용 가능한 스토리지 용량은 3Gi이다. 이때 PVC의 용량을 얼마로 해야 할까? [정답]

### /pvc/pvc-hostpath.yml

- pvc-hostpath.yaml을 클러스터에 적용한다.
```shell
$ kubectl apply -f pvc-hostpath.yaml
persistentvolumeclaim "pvc-hostpath" created

$ kubectl get pvc
NAME           STATUS    VOLUME        CAPACITY   ACCESS MODES   STORAGECLASS   AGE
pvc-hostpath   Bound     pv-hostpath   2Gi        RWO            manual         3s

$ kubectl get pv
NAME          CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS    CLAIM                  STORAGECLASS   REASON    AGE
pv-hostpath   2Gi        RWO            Delete           Bound     default/pvc-hostpath   manual                   11m
```


