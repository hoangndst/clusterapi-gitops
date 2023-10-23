# clusterapi-gitops

For Auto Upgrade Kubernetes Version

### Tạo cụm kind cho việc thử nghiệm

1. Các thành phần yêu cầu
   - Docker
   - kind
   - kubectl
   - clusterctl
2. capi-cluster.yaml
   ```yaml
   kind: Cluster
   apiVersion: kind.x-k8s.io/v1alpha4
   networking:
   ipFamily: dual
   nodes:
   - role: control-plane
   extraMounts:
       - hostPath: /var/run/docker.sock
       containerPath: /var/run/docker.sock
   ```
3. Chạy lệnh tạo cụm kind
   ```bash
   kind create cluster --config capi-cluster.yaml
   ```
4. Khởi tạo management cluster

   ```bash
   # Enable the experimental Cluster topology feature.
   export CLUSTER_TOPOLOGY=true

   # Initialize the management cluster
   clusterctl init --infrastructure docker

   ```

5. Cài đặt ArgoCD:
   ```bash
   kubectl create namespace argocd
   kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
   ```

### Upgrade k8s version manually with `cluster-api`

> Bên trên là các bước chuẩn bị cho việc thử nghiệm, bây giờ chúng ta sẽ bắt đầu thử nghiệm. Trước khi thử nghiệm ta cần hiểu được `cluster-api` sẽ upgrade version một cụm kubernetes như thế nào. Cụ thể là nó sẽ tạo ra một cụm kubernetes mới với version mới và sau đó sẽ migrate dữ liệu từ cụm cũ sang cụm mới. Vậy nên để thử nghiệm chúng ta sẽ tạo ra một cụm kubernetes với version cũ và sau đó thử nghiệm upgrade version của nó.

> Kỹ thuật nâng cấp truyền thống, được gọi là `inline upgrade`, bao gồm việc nâng cấp các thành phần Kubernetes tại chỗ. Cách tiếp cận này có một số rủi ro, trong đó có nhiều rủi ro dẫn đến việc nâng cấp không thành công trên một node, đòi hỏi phải chuyển các pods và apps sang các node khác một cách bất ngờ. Nếu các node khác cũng ở trong tình trạng tương tự, một loạt các lỗi nâng cấp có thể khiến một cụm bị sập. Nếu lỗi nâng cấp liên quan đến một loạt bản vá lỗi và cấu hình thủ công đã được áp dụng theo thời gian, thì có thể rất khó để khắc phục sự cố và đưa nó trở lại trạng thái tốt.

> Cluster API giúp nâng cấp an toàn hơn và giảm tác động của chúng đến dung lượng cụm. Cluster API thực hiện nâng cấp luân phiên, bao gồm việc cung cấp lần lượt các node mới, được nâng cấp và di chuyển các pods từ các node cũ hơn đến chúng. Cách tiếp cận này giúp duy trì nhiều dung lượng cụm nhất có thể trong quá trình nâng cấp. Trong trường hợp xấu nhất, khi node mới không được cung cấp và thêm thành công, thì workload đang chạy sẽ không bị ảnh hưởng vì các pods chỉ được di chuyển khi node mới tham gia cụm.

<div align="center">
    <img src="https://www.oreilly.com/api/v2/epubs/9781098126865/files/assets/cdkm_0401.png" width="500px">
</div>

> Bằng cách giúp việc nâng cấp cụm Kubernetes trở nên dễ dàng và an toàn hơn, Cluster API khuyến khích cập nhật thường xuyên hơn. Điều này giúp Kubernetes luôn cập nhật và an toàn hơn, đồng thời giảm nguy cơ sai lệch cấu hình, trong đó trạng thái thực của cụm khác với trạng thái mong muốn được hệ thống hóa trong bản blueprint/manifest, do các thay đổi được thực hiện trực tiếp trên cụm.

1. Tạo cụm capi-quickstart manual với version 1.26.0:

   ```bash
   clusterctl -n capi-quickstart generate cluster capi-quickstart --flavor development \
       --kubernetes-version v1.26.0 \
       --control-plane-machine-count=1 \
       --worker-machine-count=1 \
       > capi-quickstart.yaml
   kubectl apply -f capi-quickstart.yaml
   clusterctl -n capi-quickstart get kubeconfig capi-quickstart > capi-quickstart.kubeconfig
   curl https://raw.githubusercontent.com/projectcalico/calico/v3.26.3/manifests/calico.yaml -O
   kubectl --kubeconfig=capi-quickstart.kubeconfig apply -f calico.yaml
   ```

   ```bash
   [136ms][~/Desktop/sprint/upgrade-k8s-version]$ clusterctl describe cluster capi-quickstart -n capi-quickstart

   NAME                                                           READY  SEVERITY  REASON  SINCE  MESSAGE
   Cluster/capi-quickstart                                        True                     3m45s
   ├─ClusterInfrastructure - DockerCluster/capi-quickstart-4hnhh  True                     4m23s
   ├─ControlPlane - KubeadmControlPlane/capi-quickstart-56sjp     True                     3m45s
   │ └─Machine/capi-quickstart-56sjp-vw7nc                        True                     3m45s
   └─Workers
   └─MachineDeployment/capi-quickstart-md-0-jjpfw               True                     85s
       └─Machine/capi-quickstart-md-0-jjpfw-pknr9-c9pnc           True                     3m9s
   ```

2. Kiểm tra version của cụm kubernetes:
   ```bash
   [128ms][~/Desktop/sprint/upgrade-k8s-version]$ k --kubeconfig=capi-quickstart.kubeconfig get nodes
   NAME                                     STATUS   ROLES           AGE     VERSION
   capi-quickstart-56sjp-vw7nc              Ready    control-plane   6m19s   v1.26.0
   capi-quickstart-md-0-jjpfw-pknr9-c9pnc   Ready    <none>          5m42s   v1.26.0
   ```
3. Deploy demo app nginx
   ```bash
   kubectl --kubeconfig=capi-quickstart.kubeconfig apply -f clusterapi-gitops/workload
   ```
4. Lấy file `MachineTemplate` của `control-plane`:
   ```bash
   [88ms][~/Desktop/sprint/upgrade-k8s-version]$ k -n capi-quickstart get dockermachinetemplate capi-quickstart-control-plane-mr5sc -o yaml
   apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
   kind: DockerMachineTemplate
   metadata:
   annotations:
       cluster.x-k8s.io/cloned-from-groupkind: DockerMachineTemplate.infrastructure.cluster.x-k8s.io
       cluster.x-k8s.io/cloned-from-name: quick-start-control-plane
   creationTimestamp: "2023-10-19T18:29:41Z"
   generation: 1
   labels:
       cluster.x-k8s.io/cluster-name: capi-quickstart
       topology.cluster.x-k8s.io/owned: ""
   name: capi-quickstart-control-plane-mr5sc
   namespace: capi-quickstart
   ownerReferences:
   - apiVersion: cluster.x-k8s.io/v1beta1
       kind: Cluster
       name: capi-quickstart
       uid: 38d363c9-e667-4bc4-bb6b-29985e26f0ad
   resourceVersion: "20086"
   uid: a5161b1f-6571-4059-ac81-0cd2dd6e5417
   spec:
   template:
       spec:
       customImage: kindest/node:v1.26.0
       extraMounts:
       - containerPath: /var/run/docker.sock
           hostPath: /var/run/docker.sock
   ```
   Ta sẽ lưu lại và sửa lại file này để tạo ra một `MachineTemplate` mới với version mới hơn. `1.26.0` -> `1.26.10`
5. Tạo 1 MachineTemplate mới với version mới hơn:

   ```yaml
   apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
   kind: DockerMachineTemplate
   metadata:
   annotations:
       cluster.x-k8s.io/cloned-from-groupkind: DockerMachineTemplate.infrastructure.cluster.x-k8s.io
       cluster.x-k8s.io/cloned-from-name: quick-start-control-plane
   creationTimestamp: "2023-10-19T18:29:41Z"
   generation: 1
   labels:
       cluster.x-k8s.io/cluster-name: capi-quickstart
       topology.cluster.x-k8s.io/owned: ""
   name: capi-quickstart-control-plane-v1.26.6
   namespace: capi-quickstart
   ownerReferences:
   - apiVersion: cluster.x-k8s.io/v1beta1
       kind: Cluster
       name: capi-quickstart
       uid: 38d363c9-e667-4bc4-bb6b-29985e26f0ad
   resourceVersion: "20086"
   uid: a5161b1f-6571-4059-ac81-0cd2dd6e5417
   spec:
   template:
       spec:
       customImage: kindest/node:v1.26.6
       extraMounts:
       - containerPath: /var/run/docker.sock
           hostPath: /var/run/docker.sock

   ```

   ```bash
   kubectl apply -f control-plane-template.yaml
   ```

6. Lấy ra `KubeadmControlPlane` của cụm capi-quickstart và sửa đổi thành phiên bản mới hơn và `MachineTemplate` mới tạo ở trên:
   ```bash
   [63ms][~/Desktop/sprint/upgrade-k8s-version]$ kubectl -n capi-quickstart get kubeadmcontrolplane capi-quickstart-56sjp -o yaml > kubeadm-control-plane-update.yaml
   ```
   ```yaml
    machineTemplate:
        infrastructureRef:
            apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
            kind: DockerMachineTemplate
            name: capi-quickstart-control-plane-v1.26.6
            namespace: capi-quickstart
    rolloutStrategy:
        rollingUpdate:
            maxSurge: 1
        type: RollingUpdate
    version: v1.26.6
    ...
    selector: cluster.x-k8s.io/cluster-name=capi-quickstart,cluster.x-k8s.io/control-plane
    unavailableReplicas: 0
    updatedReplicas: 1
    version: v1.26.6
   ```
7. Apply lại `KubeadmControlPlane` mới:
   ```bash
   kubectl -n capi-quickstart patch --type=merge kubeadmcontrolplane capi-quickstart-56sjp --type merge --patch-file kubeadm-control-plane-update.yaml
   ```

8. Kiểm tra trạng thái của cụm kubernetes:
    ```bash
    [0.115s][master][~/Desktop/sprint/upgrade-k8s-version/clusterapi-gitops]$ clusterctl describe cluster capi-quickstart -n capi-quickstart
    NAME                                                           READY  SEVERITY  REASON                   SINCE  MESSAGE                                                       
    Cluster/capi-quickstart                                        False  Warning   RollingUpdateInProgress  6s     Rolling 1 replicas with outdated spec (1 replicas up to date)  
    ├─ClusterInfrastructure - DockerCluster/capi-quickstart-f2wrr  True                                      10m                                                                   
    ├─ControlPlane - KubeadmControlPlane/capi-quickstart-n4rtj     False  Warning   RollingUpdateInProgress  6s     Rolling 1 replicas with outdated spec (1 replicas up to date)  
    │ ├─Machine/capi-quickstart-n4rtj-jmr9g                        True                                      10m                                                                   
    │ └─Machine/capi-quickstart-n4rtj-m6g2f                        False  Info      WaitingForBootstrapData  8s     1 of 2 completed                                               
    └─Workers                                                                                                                                                                      
    └─MachineDeployment/capi-quickstart-md-0-l5z9b               True                                      8m53s                                                                 
        └─Machine/capi-quickstart-md-0-l5z9b-p7hvd-rvp2c           True                                      9m37s 
    ```
9. Kiếm tra Machine (ở đây là giả lập container) ta thấy đã tạo ra một container control-plane mới:
    ```bash
    [40ms][master][~/Desktop/sprint/upgrade-k8s-version/clusterapi-gitops]$ docker ps
    CONTAINER ID   IMAGE                                COMMAND                  CREATED          STATUS          PORTS                              NAMES
    fcbf443a183c   kindest/node:v1.26.6                 "/usr/local/bin/entr…"   19 seconds ago   Up 11 seconds   0/tcp, 127.0.0.1:32772->6443/tcp   capi-quickstart-n4rtj-m6g2f
    513e013a5cfa   kindest/node:v1.26.0                 "/usr/local/bin/entr…"   9 minutes ago    Up 9 minutes                                       capi-quickstart-md-0-l5z9b-p7hvd-rvp2c
    b22f67405369   kindest/node:v1.26.0                 "/usr/local/bin/entr…"   10 minutes ago   Up 10 minutes   0/tcp, 127.0.0.1:32771->6443/tcp   capi-quickstart-n4rtj-jmr9g
    21f147f1e106   kindest/haproxy:v20230510-486859a6   "haproxy -W -db -f /…"   10 minutes ago   Up 10 minutes   0/tcp, 0.0.0.0:32768->6443/tcp     capi-quickstart-lb
    59231b74a366   kindest/node:v1.27.1                 "/usr/local/bin/entr…"   29 minutes ago   Up 29 minutes   127.0.0.1:33867->6443/tcp          kind-control-plane
    ```
    Check các nodes của cụm:
    ```bash
    [0.158s][master][~/Desktop/sprint/upgrade-k8s-version/clusterapi-gitops]$ k --kubeconfig=capi-quickstart.kubeconfig get nodes   
    NAME                                     STATUS     ROLES           AGE   VERSION
    capi-quickstart-md-0-l5z9b-p7hvd-rvp2c   Ready      <none>          11m   v1.26.0
    capi-quickstart-n4rtj-jmr9g              Ready      control-plane   11m   v1.26.0
    capi-quickstart-n4rtj-m6g2f              NotReady   control-plane   93s   v1.26.6
    
    ```
10. Sau khi upgrade xong:
    ```bash
    [1.018s][master][~/Desktop/sprint/upgrade-k8s-version/clusterapi-gitops]$ clusterctl describe cluster capi-quickstart -n capi-quickstart      
    NAME                                                           READY  SEVERITY  REASON  SINCE  MESSAGE 
    Cluster/capi-quickstart                                        True                     9s              
    ├─ClusterInfrastructure - DockerCluster/capi-quickstart-f2wrr  True                     17m             
    ├─ControlPlane - KubeadmControlPlane/capi-quickstart-n4rtj     True                     9s              
    │ └─Machine/capi-quickstart-n4rtj-g4t7z                        True                     2m27s           
    └─Workers                                                                                               
    └─MachineDeployment/capi-quickstart-md-0-l5z9b               True                     3m40s           
        └─Machine/capi-quickstart-md-0-l5z9b-p7hvd-rvp2c           True                     15m   

    [121ms][master][~/Desktop/sprint/upgrade-k8s-version/clusterapi-gitops]$ docker ps
    CONTAINER ID   IMAGE                                COMMAND                  CREATED          STATUS          PORTS                              NAMES
    7b3efefdaa71   kindest/node:v1.26.0                 "/usr/local/bin/entr…"   4 minutes ago    Up 4 minutes    0/tcp, 127.0.0.1:32773->6443/tcp   capi-quickstart-n4rtj-g4t7z
    513e013a5cfa   kindest/node:v1.26.0                 "/usr/local/bin/entr…"   16 minutes ago   Up 16 minutes                                      capi-quickstart-md-0-l5z9b-p7hvd-rvp2c
    21f147f1e106   kindest/haproxy:v20230510-486859a6   "haproxy -W -db -f /…"   17 minutes ago   Up 17 minutes   0/tcp, 0.0.0.0:32768->6443/tcp     capi-quickstart-lb
    59231b74a366   kindest/node:v1.27.1                 "/usr/local/bin/entr…"   36 minutes ago   Up 36 minutes   127.0.0.1:33867->6443/tcp          kind-control-plane
    ```

### Khảo sát cách upgrade version của Google Cloud
Upgrade Cluster: những điều sẽ xảy ra khi Google tự động nâng cấp cụm của hoặc khi nâng cấp thủ công. 
- Zonal clusters chỉ có một `control-plane` duy nhất. Trong quá trình nâng cấp, khối lượng công việc của bạn tiếp tục chạy nhưng bạn không thể triển khai khối lượng công việc mới, sửa đổi khối lượng công việc hiện có hoặc thực hiện các thay đổi khác đối với cấu hình của cụm cho đến khi quá trình nâng cấp hoàn tất. 
- Regional clusters có nhiều bản sao của `control-plane` và mỗi lần chỉ có một bản sao được nâng cấp theo thứ tự không xác định. Trong quá trình nâng cấp, cụm vẫn có tính khả dụng cao và mỗi bản sao `control-plane` chỉ không khả dụng khi nó đang được nâng cấp.
#### Node pool upgrades
- Quá trình khi Google tự động nâng cấp node hoặc khi nâng cấp nhóm node thủ công.
    - Với nâng cấp node pool GKE, ta có thể chọn giữa hai cách nâng cấp tích hợp, có thể định cấu hình, trong đó có thể điều chỉnh quy trình nâng cấp dựa trên nhu cầu của môi trường cụm của mình. Để tìm hiểu thêm về chiến lược nâng cấp `surge` và `blue-green`, hãy xem []()
    - Trong quá trình nâng cấp node pool, không thể thay đổi cấu hình cụm trừ khi hủy nâng cấp.
- Trong quá trình nâng cấp node pool, cách nâng cấp các node tùy thuộc vào option và cách bạn định cấu hình nó. Tuy nhiên, các bước cơ bản vẫn giữ nguyên. Để nâng cấp một node, GKE sẽ xóa Pod khỏi node đó để có thể nâng cấp node đó.
- Khi một node được nâng cấp, điều sau đây sẽ xảy ra với Pod:
    1. Node được `cordoned` để Kubernetes không lên lịch cho các Pod mới trên đó.
    2. Nút sau đó sẽ bị `drained`, nghĩa là các Pod sẽ bị loại bỏ. Đối với các bản nâng cấp `surge`, GKE tôn trọng cài đặt `PodDisruptionBudget` và `GracefulTerminationPeriod` của Pod trong tối đa một giờ. Với các nâng cấp `blue-green`, thời gian này có thể được kéo dài nếu định cấu hình thời gian lâu hơn.
    3. Control-plane sắp xếp lại các Pod do bộ điều khiển quản lý lên các nút khác. Các Pods không thể lên lịch lại sẽ ở trong giai đoạn `Pending` cho đến khi chúng có thể được lên lịch lại.
Quá trình nâng cấp node pool có thể mất tới vài giờ tùy thuộc vào cách nâng cấp, số lượng nút và cấu hình khối lượng công việc của chúng.

- Surge upgrades
    - Theo mặc định, Surge upgrades được sử dụng để nâng cấp node pool. Nó sử dụng phương pháp cuộn để nâng cấp các nút. Cách này phù hợp nhất cho các ứng dụng có thể xử lý các thay đổi gia tăng, không gây gián đoạn. Với chiến lược này, các nút được nâng cấp trong một cửa sổ cuộn. Với cài đặt này, bạn có thể thay đổi số lượng nút có thể được nâng cấp cùng một lúc và mức độ gián đoạn của việc nâng cấp, tìm ra sự cân bằng tối ưu giữa tốc độ và sự gián đoạn cho nhu cầu của môi trường của bạn.
    