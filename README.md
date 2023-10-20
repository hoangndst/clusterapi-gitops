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

> Cluster API giúp nâng cấp an toàn hơn và giảm tác động của chúng đến dung lượng cụm. Cluster API thực hiện nâng cấp luân phiên, bao gồm việc cung cấp lần lượt các node mới, được nâng cấp và di chuyển các pods từ các node cũ hơn đến chúng. Cách tiếp cận này giúp duy trì nhiều dung lượng cụm nhất có thể trong quá trình nâng cấp. Trong trường hợp xấu nhất, khi nút mới không được cung cấp và thêm thành công, thì workload đang chạy sẽ không bị ảnh hưởng vì các pods chỉ được di chuyển khi nút mới tham gia cụm.

<div align="center">
    <img src="https://www.oreilly.com/api/v2/epubs/9781098126865/files/assets/cdkm_0401.png" width="500px">
</div>

> Bằng cách giúp việc nâng cấp cụm Kubernetes trở nên dễ dàng và an toàn hơn, Cluster API khuyến khích cập nhật thường xuyên hơn. Điều này giúp Kubernetes luôn cập nhật và an toàn hơn, đồng thời giảm nguy cơ sai lệch cấu hình, trong đó trạng thái thực của cụm khác với trạng thái mong muốn được hệ thống hóa trong bản blueprint/manifest, do các thay đổi được thực hiện trực tiếp trên cụm.

1. Tạo cụm capi-quickstart manual với version 1.26.0:

   ```bash
   clusterctl generate cluster capi-quickstart --flavor development \
       --kubernetes-version v1.26.0 \
       --control-plane-machine-count=1 \
       --worker-machine-count=1 \
       > capi-quickstart.yaml
   kubectl apply -f capi-quickstart.yaml
   clusterctl get kubeconfig capi-quickstart > capi-quickstart.kubeconfig
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
   name: capi-quickstart-control-plane-v1.26.10
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
       customImage: kindest/node:v1.26.10
       extraMounts:
       - containerPath: /var/run/docker.sock
           hostPath: /var/run/docker.sock

   ```

   ```bash
   kubectl apply -f capi-quickstart-control-plane-template.yaml
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
            name: capi-quickstart-control-plane-v1.26.10
            namespace: capi-quickstart
    ...
    selector: cluster.x-k8s.io/cluster-name=capi-quickstart,cluster.x-k8s.io/control-plane
    unavailableReplicas: 0
    updatedReplicas: 1
    version: v1.26.10
   ```
7. Apply lại `KubeadmControlPlane` mới:
   ```bash
   kubectl -n capi-quickstart patch --type=merge kubeadmcontrolplane capi-quickstart-56sjp --type merge --patch-file kubeadm-control-plane-update.yaml
   ```

8. Kiểm tra trạng thái của cụm kubernetes:
    ```bash
    [3m23.302s][~/Desktop/sprint/upgrade-k8s-version]$ clusterctl describe cluster capi-quickstart -n capi-quickstart 

    NAME                                                           READY  SEVERITY  REASON                   SINCE  MESSAGE                                                       
    Cluster/capi-quickstart                                        False  Warning   RollingUpdateInProgress  51s    Rolling 1 replicas with outdated spec (1 replicas up to date)  
    ├─ClusterInfrastructure - DockerCluster/capi-quickstart-4hnhh  True                                      7h4m                                                                  
    ├─ControlPlane - KubeadmControlPlane/capi-quickstart-56sjp     False  Warning   RollingUpdateInProgress  51s    Rolling 1 replicas with outdated spec (1 replicas up to date)  
    │ └─2 Machines...                                              True                                      7h4m   See capi-quickstart-56sjp-tqg8x, capi-quickstart-56sjp-vw7nc   
    └─Workers                                                                                                                                                                      
    └─MachineDeployment/capi-quickstart-md-0-jjpfw               True                                      7h1m                                                                  
        └─Machine/capi-quickstart-md-0-jjpfw-pknr9-c9pnc           True                                      7h3m                                                                  
    [0.108s][~/Desktop/sprint/upgrade-k8s-version]$ 
    ```
9. Kiếm tra Machine (ở đây là giả lập container) ta thấy đã tạo ra một container control-plane mới:
    ```bash
    [4m1.817s][~/Desktop/sprint/upgrade-k8s-version]$ docker ps
    CONTAINER ID   IMAGE                                COMMAND                  CREATED              STATUS              PORTS                              NAMES
    00cca00a34a4   kindest/node:v1.26.0                 "/usr/local/bin/entr…"   About a minute ago   Up About a minute   0/tcp, 127.0.0.1:32770->6443/tcp   capi-quickstart-56sjp-tqg8x
    4a53a5ce967d   kindest/node:v1.26.0                 "/usr/local/bin/entr…"   7 hours ago          Up 7 hours                                             capi-quickstart-md-0-jjpfw-pknr9-c9pnc
    936c030ace0c   kindest/node:v1.26.0                 "/usr/local/bin/entr…"   7 hours ago          Up 7 hours          0/tcp, 127.0.0.1:32769->6443/tcp   capi-quickstart-56sjp-vw7nc
    eeacab420c3a   kindest/haproxy:v20230510-486859a6   "haproxy -W -db -f /…"   7 hours ago          Up 7 hours          0/tcp, 0.0.0.0:32768->6443/tcp     capi-quickstart-lb
    71133f48c69a   kindest/node:v1.27.1                 "/usr/local/bin/entr…"   18 hours ago         Up 7 hours          127.0.0.1:40603->6443/tcp          kind-control-plane
    [62ms][~/Desktop/sprint/upgrade-k8s-version]$ 
    ```
    Check các nodes của cụm:
    ```bash
    [4m11.790s][~/Desktop/sprint/upgrade-k8s-version]$ kubectl --kubeconfig=capi-quickstart.kubeconfig get nodes 
    NAME                                     STATUS     ROLES           AGE    VERSION
    capi-quickstart-56sjp-tqg8x              NotReady   control-plane   61s    v1.26.0
    capi-quickstart-56sjp-vw7nc              Ready      control-plane   7h4m   v1.26.0
    capi-quickstart-md-0-jjpfw-pknr9-c9pnc   Ready      <none>          7h3m   v1.26.0
    [98ms][~/Desktop/sprint/upgrade-k8s-version]$ 
    ```
10. Sau khi upgrade xong:
    ```bash
    [1m7.394s][~/Desktop/sprint/upgrade-k8s-version]$ clusterctl describe cluster capi-quickstart -n capi-quickstart 
    NAME                                                           READY  SEVERITY  REASON  SINCE  MESSAGE 
    Cluster/capi-quickstart                                        True                     10s             
    ├─ClusterInfrastructure - DockerCluster/capi-quickstart-4hnhh  True                     7h6m            
    ├─ControlPlane - KubeadmControlPlane/capi-quickstart-56sjp     True                     10s             
    │ └─Machine/capi-quickstart-56sjp-tqg8x                        True                     2m13s           
    └─Workers                                                                                               
    └─MachineDeployment/capi-quickstart-md-0-jjpfw               True                     20s             
        └─Machine/capi-quickstart-md-0-jjpfw-pknr9-c9pnc           True                     7h5m            
    [116ms][~/Desktop/sprint/upgrade-k8s-version]$ 


    [82ms][~/Desktop/sprint/upgrade-k8s-version]$ docker ps
    CONTAINER ID   IMAGE                                COMMAND                  CREATED         STATUS         PORTS                              NAMES
    00cca00a34a4   kindest/node:v1.26.0                 "/usr/local/bin/entr…"   3 minutes ago   Up 3 minutes   0/tcp, 127.0.0.1:32770->6443/tcp   capi-quickstart-56sjp-tqg8x
    4a53a5ce967d   kindest/node:v1.26.0                 "/usr/local/bin/entr…"   7 hours ago     Up 7 hours                                        capi-quickstart-md-0-jjpfw-pknr9-c9pnc
    eeacab420c3a   kindest/haproxy:v20230510-486859a6   "haproxy -W -db -f /…"   7 hours ago     Up 7 hours     0/tcp, 0.0.0.0:32768->6443/tcp     capi-quickstart-lb
    71133f48c69a   kindest/node:v1.27.1                 "/usr/local/bin/entr…"   18 hours ago    Up 7 hours     127.0.0.1:40603->6443/tcp          kind-control-plane
    [38ms][~/Desktop/sprint/upgrade-k8s-version]$ 
    ```