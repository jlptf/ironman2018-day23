## Day 23 - 常見問題與建議 (4)

### 本日共賞

* ConfigMap 或 Secret 遺失
* 掛載磁碟錯誤

### 希望你知道

* [與 k8s 溝通: kubectl](https://ithelp.ithome.com.tw/articles/10193502)
* [yaml](https://ithelp.ithome.com.tw/articles/10193509)

<br/>

#### ConfigMap 或 Secret 遺失

在 k8s 中會使用 ConfigMap 或 Secret 物件來儲存系統運行時需要的設定檔。一個常見的錯誤是：指定使用某個 ConfigMap 或 Secret 物件，但是指定的物件卻不存在。

> 有可能是放在不同的命名空間 (namespace)

底下內容為 error-missing-configmap.yaml

```
# error-missing-configmap.yaml

---
apiVersion: v1
kind: Pod
metadata:
  name: err-misconfigmap-nginx
spec:
  containers:
  - name: nginx
    image: nginx
    env:
    - name: MY_KEY    		  <=== 設定一個環境變數 MY_KEY
      valueFrom:
        configMapKeyRef:
          name: my-config     <=== 參考名為 my-config 的 ConfigMap 物件 
          key: mykey          <=== 參考值
```

部署到 k8s 

```bash
$ kubectl get po
NAME                     READY     STATUS                       RESTARTS   AGE
err-misconfigmap-nginx   0/1       CreateContainerConfigError   0          7m
```

這裡就會看到 `CreateContainerConfigError` 的錯誤，接著檢查詳細內容

```bash
$ kubectl describe pods err-misconfigmap-nginx
...
Events:
  Type     Reason                 Age              From               Message
  ----     ------                 ----             ----               -------
  Normal   Scheduled              6m               default-scheduler  Successfully assigned err-misconfigmap-nginx to minikube
  Normal   SuccessfulMountVolume  6m               kubelet, minikube  MountVolume.SetUp succeeded for volume "default-token-ld9jt"
  Normal   Pulling                6m (x4 over 6m)  kubelet, minikube  pulling image "alpine"
  Normal   Pulled                 6m (x4 over 6m)  kubelet, minikube  Successfully pulled image "alpine"
  Warning  Failed                 6m (x4 over 6m)  kubelet, minikube  Error: configmaps "my-config" not found   <=== 找不到 ConfigMap 物件
  Warning  FailedSync             6m (x4 over 6m)  kubelet, minikube  Error syncing pod
```

`configmaps "my-config" not found` 會顯示在 Event 區域內。

*建議方法*

解決辦法很簡單，把需要用到的物件補上，如下 missing-configmap.yaml

```
# missing-configmap.yaml

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-config     <=== "my-config" 與 error-missing-configmap.yaml 內指定的名稱要一致
data:
  mykey: thisismykey  <=== "mykey" 與 error-missing-configmap.yaml 內指定的名稱要一致
```

然後再把 ConfigMap 部署到 k8s 

```bash
$ kubectl apply -f missing-configmap.yaml
configmap "my-config" created
```

接著再檢查一次 Pod

```bash
$ kubectl get pods
NAME                     READY     STATUS    RESTARTS   AGE
err-misconfigmap-nginx   1/1       Running   0          6m
```

這時候你就會發現本來無法運行的 Pod 現在可以成功運作了。

> Secret 遺失的狀況與 ConfigMap 的情況類似

上面的例子是將物件套用在環境變數上，接下來看看如果以 Volume 的形式使用 Secret 會出現什麼錯誤。底下是 error-missing-secret.yaml 的內容

```
# error-missing-secret.yaml

---
apiVersion: v1
kind: Pod
metadata:
  name: err-missecret-nginx
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - mountPath: /etc/secret/   <=== 將 mysecret 掛載到 /etc/secret
      name: mysecret
  volumes:            
  - name: mysecret  
    secret:
      secretName: mysecret  <=== 指定 Volume 參考 mysecret 物件
```

部署到 k8s 並檢視狀態

```bash
$ kubectl apply -f error-missing-secret.yaml 
pod "err-missecret-nginx" created

$ kubectl get pods
NAME                  READY     STATUS              RESTARTS   AGE
err-missecret-nginx   0/1       ContainerCreating   0          6m
```

你會看到狀態一直都維持在 `ContainerCreating`，讓我們瞧瞧它發生什麼事

```bash
$ kubectl describe pods err-missecret-nginx
...
Events:
  Type     Reason                 Age               From               Message
  ----     ------                 ----              ----               -------
  Normal   Scheduled              10m               default-scheduler  Successfully assigned err-missecret-nginx to minikube
  Normal   SuccessfulMountVolume  10m               kubelet, minikube  MountVolume.SetUp succeeded for volume "default-token-ld9jt"
  Warning  FailedMount            7m                kubelet, minikube  Unable to mount volumes for pod "err-missecret-nginx_default(42ee0ef4-d4e7-11e7-b5ae-0800279a8379)": timeout expired waiting for volumes to attach/mount for pod "default"/"err-missecret-nginx". list of unattached/unmounted volumes=[mysecret]
  Warning  FailedSync             7m                kubelet, minikube  Error syncing pod
  Warning  FailedMount            7m (x9 over 10m)  kubelet, minikube  MountVolume.SetUp failed for volume "mysecret" : secrets "mysecret" not found  <=== 錯誤訊息在這裡！
```

`failed for volume "mysecret" : secrets "mysecret" not found` 毫無懸念的，因為找不到 `mysecret` 這個 Volume 物件，因此 Pod 無法正常運作。

*建議方法*

修正的方法與之前一樣，只要將 Secret 物件補上後，Pod 就可以正確運作。下面是 missing-secret.yaml 內容

```
# missing-secret.yaml

---
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  username: YWRtaW4=
  password: MWYyZDFlMmU2N2Rm
```

> 這裡就不附上後續操作，大家可以試著部署到 k8s 看看錯誤是否被修正
> 
> 如果有問題，請再留言告訴我

#### 掛載磁碟錯誤

掛載磁碟錯誤也是常發生的問題，剔除打錯磁碟名稱，磁碟本身並不存在也是造成掛載錯誤的原因。底下示範掛載一個不存在的 Google Compute Engine (GCE) 磁碟。

```
# error-unmount.yaml

---
apiVersion: v1
kind: Pod
metadata:
  name: err-unmount-nginx
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - mountPath: /test   <=== 掛載目錄
      name: test
  volumes:
  - name: test
    gcePersistentDisk:   <=== 指定掛載一個 GCE 的磁碟
      pdName: my-disk    <=== 指定磁碟名稱
      fsType: ext4       <=== 磁碟格式
```

查看一下 Pod 的狀態

```bash
$ kubectl get pods
NAME                READY     STATUS              RESTARTS   AGE
err-unmount-nginx   0/1       ContainerCreating   0          1m
```

你會發現 k8s 在嘗試建立 Container，於是查看詳細內容

```bash
$ kubectl describe pods err-unmount-nginx
Volumes:
  test:  <=== 以下為掛載磁碟的內容
    Type:       GCEPersistentDisk (a Persistent Disk resource in Google Compute Engine)
    PDName:     my-disk
    FSType:     ext4
    Partition:  0
    ReadOnly:   false
  default-token-ld9jt:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-ld9jt
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     <none>
Events:
  Type     Reason                 Age   From               Message
  ----     ------                 ----  ----               -------
  Normal   Scheduled              2m    default-scheduler  Successfully assigned err-unmount-nginx to minikube
  Normal   SuccessfulMountVolume  2m    kubelet, minikube  MountVolume.SetUp succeeded for volume "default-token-ld9jt"
  Warning  FailedMount            1m    kubelet, minikube  MountVolume.SetUp failed for volume "test" : mount failed: exit status 32   <=== 錯誤在這裡！
...
```

`"test" : mount failed`，因為指定的 disk 並不存在，所以無法順利掛載。

*建議方法*

要處理這類的問題，請確認需要用到的 Volume 都已經被正確配置即可。

> 由於是使用 minikube 當做示範，如果是使用雲端商提供的服務例如： GKE (Google Kubernetes Engine)，則會看到更詳細的錯誤提示。
> 
> 一般來說 minikube 不會直接使用 GCE 的 Disk，上述例子僅供示範用。

本文與部署檔案同步發表於 [https://jlptf.github.io/ironman2018-day23/](https://jlptf.github.io/ironman2018-day23/)