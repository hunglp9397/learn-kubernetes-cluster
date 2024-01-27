
# 1.Tạo kubernetes cluster
## 1.1. Tạo máy ảo
- Cần tạo các máy có cấu hình như sau:
```shell
Tên máy/Hostname	Thông tin hệ thống	                                            Vai trò
master.xtl          HĐH CentOS7, Docker CE, Kubernetes. Địa chỉ IP 172.16.10.100	Khởi tạo là master
worker1.xtl	        HĐH CentOS7, Docker CE, Kubernetes. Địa chỉ IP 172.16.10.101	Khởi tạo là worker
worker2.xtl	        HĐH CentOS7, Docker CE, Kubernetes. Địa chỉ IP 172.16.10.102	Khởi tạo là worker
```

- B1: Cài vagrant
- B2: Truy cập folder _[virtual_machine_config/kubernetes-centos7/master]_. Chạy lệnh: `vagrant up` 
   + Nếu lỗi thì làm như sau:
```shell
vagrant ssh
sudo yum -y update kernel
exit
vagrant reload --provision
```

- B3: Kiểm kết quả bằng cách ssh từ máy host vào máy ảo (Mật khẩu là 123). Kết quả như hình sau:
```shell
    hunglp@HungLP MINGW64 /d/Workspace/Learning/learn-kubernetes-cluster/virtual_machine_config/kubernetes-centos7/master (master)
    $ ssh root@172.16.10.100
    The authenticity of host '172.16.10.100 (172.16.10.100)' can't be established.
    ED25519 key fingerprint is SHA256:Ev96w6AWSPnGYByZRiiv3npc2m4PifjlBTRtg4HCEk8.
    This key is not known by any other names
    Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
    Warning: Permanently added '172.16.10.100' (ED25519) to the list of known hosts.
    root@172.16.10.100's password:
    [root@master ~]# docker version
    Client:
    Version:           18.06.2-ce
    API version:       1.38
    Go version:        go1.10.3
    Git commit:        6d37f41
    Built:             Sun Feb 10 03:46:03 2019
    OS/Arch:           linux/amd64
    Experimental:      false

    Server:
    Engine:
    Version:          18.06.2-ce
    API version:      1.38 (minimum version 1.12)
    Go version:       go1.10.3
    Git commit:       6d37f41
    Built:            Sun Feb 10 03:48:29 2019
    OS/Arch:          linux/amd64
    Experimental:     false
    [root@master ~]#
```
- B4 : Tạo máy ảo worker1 và worker 2 tương tự ở 2 folder worker1 và worker2


## 1.2 Tạo cluster
- B1: ssh vào máy master `ssh root@172.16.10.100`, Sau đó chạy lệnh sau: `kubeadm init --apiserver-advertise-address=172.16.10.100 --pod-network-cidr=192.168.0.0/16`

    +  ở đây Trong lệnh khởi tạo cluster có tham số --pod-network-cidr để chọn cấu hình mạng của POD, do dự định dùng Addon calico nên chọn--pod-network-cidr=192.168.0.0/16
    +  Nếu gặp lỗi khi chạy lệnh kubeadm init này thì chạy  cụm lệnh sau rồi chạy lại kubeadm init:
    `sudo yum makecache`
    `sudo yum -y install containerd`
    `systemctl start containerd`
    `systemctl enable containerd`
    `rm /etc/containerd/config.toml`
    `systemctl restart containerd`

- B2 : Sau khi lệnh chạy xong, chạy tiếp cụm lệnh nó yêu cầu chạy sau khi khởi tạo- để chép file cấu hình đảm bảo trình kubectl trên máy này kết nối Cluster
```shell
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
- B3: Tại máy master, Cài plugin calico : 
`kubectl apply -f https://docs.projectcalico.org/v3.10/manifests/calico.yaml`
  + Kiểm tra thông tin pods : `kubectl get pods -A`
  + Lỗi calico-kube-controller pending:
    + Fix bằng cách unset schedule : 
      `kubectl taint nodes --all node-role.kubernetes.io/control-plane-`
      `kubectl taint nodes --all node-role.kubernetes.io/master-`

- B4.Tại máy master, chạy tiếp lệnh sau để gen ra lệnh join : 
`kubeadm token create --print-join-command`

- B5.Tại máy worker, apply lệnh join sau: 
`kubeadm join 172.16.10.100:6443 --token mwm3oi.81l9gh6fpmafn1b4 --discovery-token-ca-cert-hash sha256:3a5f72c08505f4609165f6cd689b6b93a2cab7d50a7d4bc74053a23c85b0e451`


- B6: Tiếp đó, nó yêu cầu cài đặt một Plugin mạng trong các Plugin tại addon, ở đây đã chọn calico, nên chạy lệnh sau để cài nó
`kubectl apply -f https://docs.projectcalico.org/v3.10/manifests/calico.yaml`

- B7: Kiểm tra cluster:
```shell
# Thông tin cluster
kubectl cluster-info
# Các node trong cluster
kubectl get nodes
# Các pod đang chạy trong tất cả các namespace
kubectl get pods -A
```
![2](/img_guide/2.png)

- B8: Mở kết nối của máy master
    + Tại máy master, Thực hiện lệnh sau với Cluster để lấy lệnh kết nối: `kubeadm token create --print-join-command`
    + Kết quả:
        ```shell
        [root@master ~]# kubeadm token create --print-join-command
        kubeadm join 172.16.10.100:6443 --token n7olii.yro8krwj96ddnz6x --discovery-token-ca-cert-hash sha256:398a0bacced59719fab0e7eab6e12ca3af546be11923308290db96838937a21d  
        ```

- B9: Kết nối các các máy worker tới máy master
    + Tại máy worker1, worker2, Thực hiện lệnh vừa lấy được:`kubeadm join 172.16.10.100:6443 --token n7olii.yro8krwj96ddnz6x --discovery-token-ca-cert-hash sha256:398a0bacced59719fab0e7eab6e12ca3af546be11923308290db96838937a21d `
    + Nếu gặp lỗi khi chạy lệnh kubeadm join này thì chạy thêm 2 lệnh sau rồi chạy lại kubeadm join:
        `rm /etc/containerd/config.toml`
        `systemctl restart containerd`
    + Kết quả trên máy worker1, worker2:
        ```shell
            [root@worker1 ~]# kubeadm join 172.16.10.100:6443 --token n7olii.yro8krwj96ddnz6x --discovery-token-ca-cert-hash sha256:398a0bacced59719fab0e7eab6e12ca3af546be11923308290db96838937a21d 
            [preflight] Running pre-flight checks
            [preflight] Reading configuration from the cluster...
            [preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
            [kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
            [kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
            [kubelet-start] Starting the kubelet
            [kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...
            This node has joined the cluster:
            * Certificate signing request was sent to apiserver and a response was received.
            * The Kubelet was informed of the new secure connection details.
            Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
            [root@worker1 ~]#
        ```
    +  Kết quả trên máy master:
    ```shell
        [root@master ~]# kubectl get nodes
        NAME          STATUS   ROLES           AGE     VERSION
        master.xtl    Ready    control-plane   29h     v1.28.2
        worker1.xtl   Ready    <none>          7m40s   v1.28.2
        worker2.xtl   Ready    <none>          34s     v1.28.2
    ```

## 1.3 Thêm cluster vừa tạo vào cluster kubernetes ở máy host
- B1: Kiểm tra cluster ở máy host:
```shell
    C:\Users\hunglp>kubectl cluster-info
    Kubernetes control plane is running at https://kubernetes.docker.internal:6443
    CoreDNS is running at https://kubernetes.docker.internal:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

    To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.

    C:\Users\hunglp>
```
- B2: Lấy thông tin kubectl ở máy master tại đường dẫn file sau:
`[root@master /]# cd etc/kubernetes/admin.conf`
- B3: Lấy thông tin kubectl ở máy host tại đường dẫn sau:
 `C:\Users\hunglp\.kube\config`
- B4. Thực hiện copy các mục : context, cluster, user ở file `[root@master /]# cd etc/kubernetes/admin.conf` vào file `C:\Users\hunglp\.kube\config`
  (Lưu ý chỗ này nên lưu file cũ lại để sau còn backup)
- B5: Kết quả copy các mục như sau:
```shell
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURCVENDQWUyZ0F3SUJBZ0lJS0tqcmh6b3NhQW93RFFZSktvWklodmNOQVFFTEJRQXdGVEVUTUJFR0ExVUUKQXhNS2EzVmlaWEp1WlhSbGN6QWVGdzB5TkRBeE1UUXlNalV5TVROYUZ3MHpOREF4TVRFeU1qVTNNVE5hTUJVeApFekFSQmdOVkJBTVRDbXQxWW1WeWJtVjBaWE13Z2dFaU1BMEdDU3FHU0liM0RRRUJBUVVBQTRJQkR3QXdnZ0VLCkFvSUJBUUNZb0VOT0hLVURzUlNYdENUZXJUQkFRR1Y4Z2hPVFFPVGQ1cDVDTEVtMStyajZOWG9BdXd3ZHR5dGIKU3lkc0RoRzJGNDB1cXRHbHlOeDBYV2Jjang1cTEyN0lId2J1ZFdUYnY0aXh4V3dQSTZzM3k5aE9KUzkzRzg2YQpXRUhiSXIvNWlPTzZqQk10aTBEalExZHIvdTFMUFJvd0hoTEVjZ1RxWENZMy9CUHRkWlZxczFONzRtUGJhcnJJCnZUa0NVeFh6Z2lsbEl0ckJoWktnQkZmMHc3RmFVT3luK3FPSXltWEJONnU0ZWcyQXZUQ204QlZMS3grQWpBak0Kdk1tV3RTSFNCK0dTRXZuOStKVmRDclVrRHI0aVJ3eTRjRktGWk5KcUhIZFVyOUhzV0ZSOEtFQmJBRGplRkFHbwowRXEwL0RNanF2bTNyQ0twbCtIeU1zdzBSUDhsQWdNQkFBR2pXVEJYTUE0R0ExVWREd0VCL3dRRUF3SUNwREFQCkJnTlZIUk1CQWY4RUJUQURBUUgvTUIwR0ExVWREZ1FXQkJUNWZuOG1jNW5yOWJuRHFCVTY2OHBRY2tJekJUQVYKQmdOVkhSRUVEakFNZ2dwcmRXSmxjbTVsZEdWek1BMEdDU3FHU0liM0RRRUJDd1VBQTRJQkFRQkZTMnBGSFFQTwoybzFtUkZRVzdZUHBwa1hVeDBjZFVQVVhzck8vWHJwZ3ZpVzlpMmNPUWp6bHM2dllBZWVUd0ZCbm1JN09JLzhZCmFlZEtLNkhvOTNoTlYycW9BSVhiUTlaSHowdkhoa1dGK280Z1d0UkFOZFMyQkhvYWI1TzRoTnlrZks0R3dPS2YKZnZmT3ZlTzZ1NWRnR3NqM08rWXhnMzloL1lpcmJ4VHo3alZDN0Q4SUtEZmFJWHVtSVM0Ykc5dWVES2pkUkQyRgp4NXBaNjRVRnA0bldhV25FU00ycDJORTRZdWU3N0kvMmMxc1F5R2VMd0I0WHp6K0lIVlJJc0lVc1VhRjhzUmJjCmo0OHRUOWk2V3J5QjlCa3d2Wnd5OHdzbGQ0YnNFOWRXNUN3cGw5UHRBUHlkd1BoeTc2eVFGaXRzaGxBZHlMdkIKekxqQTRqcGVoMDRXCi0tLS0tRU5EIENFUlRJRklDQVRFLS0tLS0K
    server: https://kubernetes.docker.internal:6443
  name: docker-desktop
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURCVENDQWUyZ0F3SUJBZ0lJRjhPUmpXb2FCSll3RFFZSktvWklodmNOQVFFTEJRQXdGVEVUTUJFR0ExVUUKQXhNS2EzVmlaWEp1WlhSbGN6QWVGdzB5TkRBeE1qQXdNekF6TVRGYUZ3MHpOREF4TVRjd016QTRNVEZhTUJVeApFekFSQmdOVkJBTVRDbXQxWW1WeWJtVjBaWE13Z2dFaU1BMEdDU3FHU0liM0RRRUJBUVVBQTRJQkR3QXdnZ0VLCkFvSUJBUUMrcXBIS2xJQ1lDdGVDWXM5ZzlkZUFmK1gxN25mUXB0dklXWEN4Y2RtUWVWVVlkS3NOZXFDZEZQZWwKb0E3bHFFR29rQTRsQzYvVGVtdUJaOWdmKzMyTEJWK1VNMEFSM1dLUzFaMGdqTzlsbTFaMzBNc0lNY1Y4bXo4YwpnRG4zcVJaVHE0VGVMd0tmc0dnYjJrQ1M0V2ZSdmNISm1WbVBLQndxeW9sVWJOQ0hxcWhod2dWZEJsamtQdmN4CmdXcWRVemk4VzJ0MDdVT1NPbmxYSklxam5XamQ1Q0lWR2JyZCs1b001cGVEN3dzWStEQXhzUlovbGU1NDU1dlcKM0hKWjNTYjZjS1JFRFVNcmozVlllKzdCWllZSEpmL3JsQk5USTdETFdoWW83Si9ocy81TkpvZzlkbHAya2RZWgpEelhjSXdKMGY5OVp2L21HMWpyQXBWdEVoQmMxQWdNQkFBR2pXVEJYTUE0R0ExVWREd0VCL3dRRUF3SUNwREFQCkJnTlZIUk1CQWY4RUJUQURBUUgvTUIwR0ExVWREZ1FXQkJRalhwK1hZanBwTExHQVZSUzg1dE9uZ0t0TlpqQVYKQmdOVkhSRUVEakFNZ2dwcmRXSmxjbTVsZEdWek1BMEdDU3FHU0liM0RRRUJDd1VBQTRJQkFRQ21ZTEozY0NhaQovWUtUY1g4N2pzSnZXbWdqOWJjeHRycFNwTDh0Zk9PMEJ5STYwREd6Q1R1Mi81ZlV3RmZwZHJhNmpaZ1pBUlVQCjhNVFo0Z1NnUDNyOUlaVHlSOW9tZmZTd2RSVThmSEdYUktWWHgzV0tHS2FvVEtpR1drdHk1OU53L1hER2hpbHYKOHI2VHNsWHNIRnl4YzZLMzY2YWFKT3N6L0wybEhWM3ovL3d4SjNsYTZqVXdlUWZVVndsK2dGeXlFQ0p0S2xZRAorTmFGTHBtRXJhQ04ybjNobHJiVUpDTmdnZmxmeFAxZGtJU1JHb0hYZ1dZV0VUQlhsRjJpWVZpaXlyWVdXVk1GCmJ6ckVKVjJpdnF4blQrYUp5cTgxYUtNQXUzZmV6K0dYTHlhNmlJTXUyUDN0S1NCWFhBZGVpckJ2REt6bU43aFMKemxDNUhNTytPNGtPCi0tLS0tRU5EIENFUlRJRklDQVRFLS0tLS0K
    server: https://172.16.10.100:6443
  name: kubernetes
- cluster:
    certificate-authority: C:\Users\hunglp\.minikube\ca.crt
    extensions:
    - extension:
        last-update: Sun, 08 Oct 2023 08:58:34 +07
        provider: minikube.sigs.k8s.io
        version: v1.30.1
      name: cluster_info
    server: https://172.17.185.142:8443
  name: minikube
contexts:
- context:
    cluster: docker-desktop
    user: docker-desktop
  name: docker-desktop
- context:
    cluster: kubernetes
    user: kubernetes-admin
  name: kubernetes-admin@kubernetes
- context:
    cluster: minikube
    extensions:
    - extension:
        last-update: Sun, 08 Oct 2023 08:58:34 +07
        provider: minikube.sigs.k8s.io
        version: v1.30.1
      name: context_info
    namespace: default
    user: minikube
  name: minikube
current-context: docker-desktop
kind: Config
preferences: {}
users:
- name: docker-desktop
  user:
    client-certificate-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURRakNDQWlxZ0F3SUJBZ0lJZXlUZ1R0NTJGNGt3RFFZSktvWklodmNOQVFFTEJRQXdGVEVUTUJFR0ExVUUKQXhNS2EzVmlaWEp1WlhSbGN6QWVGdzB5TkRBeE1UUXlNalV5TVROYUZ3MHlOVEF4TWpBd056RTFOVFZhTURZeApGekFWQmdOVkJBb1REbk41YzNSbGJUcHRZWE4wWlhKek1Sc3dHUVlEVlFRREV4SmtiMk5yWlhJdFptOXlMV1JsCmMydDBiM0F3Z2dFaU1BMEdDU3FHU0liM0RRRUJBUVVBQTRJQkR3QXdnZ0VLQW9JQkFRQzUrWERVZllKZ2ovT3IKME11RFNPNnUyckxmN3dRVUorR0lENTZPL1RQQWpTcjh1a1pOTU4xTjJhQ0J1aTlScUJheng5UW8yZ0dYZzBRKwpJS1ZIRTNkMjR0QlhZTUNHTWVtRnh4em9mdkJtdForbTNHRm1GVWprQ1RpOVc1eGV1VnJWU2FyNzJtekFnRzRECnRrUlVUMlJtbmd1N3NVbXB0cmFPMFlJT1NBUDJxN1BSbnFxUkVJQjZ1MTQ1a2NrLytXcEdSU0p1RmkyWEpSaUkKNnFuZ3lhcmVwQk1jcU5SR2QzQWFXT0N4ek8xVUNpMmN4cml2N3BwUk1GVDFkcWd0akFGbS9iUTdtVDM4dy90WQpGN3ZCOGxrQlZoM0E1WnpIYU5vcnhoUjRlcTNEaDV1dElaVzdvZkl0TnFRM2FrRHhxUnN2cUdJeTFnYXFwLzlSCm9CWnhROERSQWdNQkFBR2pkVEJ6TUE0R0ExVWREd0VCL3dRRUF3SUZvREFUQmdOVkhTVUVEREFLQmdnckJnRUYKQlFjREFqQU1CZ05WSFJNQkFmOEVBakFBTUI4R0ExVWRJd1FZTUJhQUZQbCtmeVp6bWV2MXVjT29GVHJyeWxCeQpRak1GTUIwR0ExVWRFUVFXTUJTQ0VtUnZZMnRsY2kxbWIzSXRaR1Z6YTNSdmNEQU5CZ2txaGtpRzl3MEJBUXNGCkFBT0NBUUVBQW5EL2p2NHI1ek1GcWRXMmpGa2RMVHVwRTVadTByanlZSUJLTmRNTXpyN1VpeEpHN1g1S3VnNTIKOUtNS1JVVU13czlVTUtnYzA5RnNnS1hweEkwUWRaZUpEc2pGNnJwdWF5YXI2ODNmb1NlSk5QVEJaRG5qUGxyaAozNTVKekZoakZpUDhna0FJTjR5WE9lUUFWTDNlK3RHZ1EybWNaVEEwOXhCak9MdTBvV1NQMkxoY1R1WlZOU0ZhClhOZ3dFcWhYeUhoNnh4QW9pZlVJUExGUHZZTDlqbWJBN295UG8xTkdjUitlSjRoM256cWw5czRRR3Y3YVc3YkoKVVk4cFJVU0c5YlJSd05WOE01NzdCbTFTdkt2ZXdFWHMrY0hIRUJqa0JvbGE5aHBrVXBUSDdHY1ozcUNRZFpTSApOZ0ZSSmprM2dmWnN0NTRxanY1U0NMbHpiUlhncHc9PQotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==
    client-key-data: LS0tLS1CRUdJTiBSU0EgUFJJVkFURSBLRVktLS0tLQpNSUlFcEFJQkFBS0NBUUVBdWZsdzFIMkNZSS96cTlETGcwanVydHF5Mys4RUZDZmhpQStlanYwendJMHEvTHBHClRURGRUZG1nZ2JvdlVhZ1dzOGZVS05vQmw0TkVQaUNsUnhOM2R1TFFWMkRBaGpIcGhjY2M2SDd3WnJXZnB0eGgKWmhWSTVBazR2VnVjWHJsYTFVbXErOXBzd0lCdUE3WkVWRTlrWnA0THU3RkpxYmEyanRHQ0RrZ0Q5cXV6MFo2cQprUkNBZXJ0ZU9aSEpQL2xxUmtVaWJoWXRseVVZaU9xcDRNbXEzcVFUSEtqVVJuZHdHbGpnc2N6dFZBb3RuTWE0CnIrNmFVVEJVOVhhb0xZd0JadjIwTzVrOS9NUDdXQmU3d2ZKWkFWWWR3T1djeDJqYUs4WVVlSHF0dzRlYnJTR1YKdTZIeUxUYWtOMnBBOGFrYkw2aGlNdFlHcXFmL1VhQVdjVVBBMFFJREFRQUJBb0lCQUFOeDZYTW1PQW9ONEplbwpNSHpvRnZQS1BWSUVuWEM2SkdWZTFMTVZZYVlKZDJoakV2WlBGMnBmdzZkamlZamJzai8yVGFuTUVBZDhlUUVsCm5hb3BaQ2Nob0haZDVuTVY3WnQ2eXNCTHlhdzlaUTIwTzJHbXQwanlHc2ozTDNoWnVxTUUwRlFHQWNtM0YxS2UKUjduQUZyNEg0M1BBbnZxejFjSGpnNk04RmthMWMzQlhTRjlCeUl2Y0RsQldwTUJSOWFPUVBHQlFZMUVyME1yMwoyTy9RakFJUDRWNnZkczg2cUJOS3hyaVdNVFJCOSt0V2tZMHU3SFpvdWFpbUlXdE0wWG81SXJLWDZ0eWd6K09VCmRPSUpnWE5GcGMyeWdjTE1mdmpjMDN1RDYreWtHdUwyb3RiSlU0alBKdzhGWmU1SS9TeW9MN2F5M1ZmYWg5a0sKV0VtYVA3a0NnWUVBNXo1WDdmU2I4c3BOME1HZ0JGd0hqSC80MDlKTFdIa05LNE53SkNnTTI5ajFLRU1pbGNncAoxOHZyTGRPd0U4Ym5oUFZ4MUhvUEdyM1RRZnA0SEt0WmVxSkpTZkZRNlF3R1d3eVpoTUR3UTdSS0ZBUWxzcGYyCkFaa1ZuL1JGT0s1K3F3NThUcVl0SzR0NDdvUVJuWjFLS2VPVTBIaHVySyttMG1DUU1hcWJjRDhDZ1lFQXplSnAKbDJJNHRQeWpFK3drL2d0d1pobG9rNmkwanNkbGlPYjZWTjdOdDBhbjNoUDRwTlRmcW5QSXl3cWdEcGl5YjkwMgpWbzVXN1Y3UkNrRjRhaytKcWZtM0xGbkI2TnU0NEwxS3FnbGNjZXZRV0ZIN2ZzbFU4MGM5UHVIMlVFV0hrRHg2CnVSTzcrNzVHWWloKzBQSG85N0t5RVZTa0V2bkVGZFlIbWJkY2l1OENnWUVBeDYyVzRmdzUrWUhsaGVEY20wY1kKb2FNVHExMUpBSUd1OUtjUDI3alZ1YlZ6cEt1c0hxaDBNVXA5cnRtL2pxUlA0UWpNblV3MDVNT0x1OHBiazI0RwoyeFZ0c2JMMlNmYS9Pam44Q3AxTUd6cUFTUjUzcXVyN1c4L2owM1pybTVGYUFiMkZhNmlsRXBmaCtod0MxaFl2CkoxTEVldXV6cmR3VGNsQTkweFZlR2FNQ2dZRUF2ZW05SnhSR1pNU3FGVVY5OWcxTk9CRDJBMGJhanQzbGpmd3EKTEVGOWx6TUl3L1MrSmlYcXo0dVFTNkxZYzc1czBuMUdrMThuVmp4aExVbXBMcitCcUJZZDNqNUpmV2U0eVM0egpBbGd5T3krZjl1aGd5ZG9qajJsR1dJd05Mb3lFZVFzZzFUb2I3Q0xmUDhwRStLNDlETWQ4TkRwVVF1QzcvTHg2Ck5GUU1mR2NDZ1lCMnpxUm41OUJLSkNXbUZmK0drZ1E3YnRacWppV3c5dXF5NDFvbm1EYTBFUHlvYmpBTXhjc0UKTnFrNGdVeUpMbm85Y0FSL0lzdnJmcGVnaW9lZkFEdUZKeXhmTlFtaVREOXlyQlU2WG9QaWVsS0g3SEE2dzgxbApMaHQ4Qzd0NmQ0eUNmWGVuSmFhbEloWG81UUtJNkR4K2pyWjgvK085K0Yrc01ld25GM0tVenc9PQotLS0tLUVORCBSU0EgUFJJVkFURSBLRVktLS0tLQo=
- name: kubernetes-admin
  user:
    client-certificate-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURJVENDQWdtZ0F3SUJBZ0lJQVlubWs2SlhWL1F3RFFZSktvWklodmNOQVFFTEJRQXdGVEVUTUJFR0ExVUUKQXhNS2EzVmlaWEp1WlhSbGN6QWVGdzB5TkRBeE1qQXdNekF6TVRGYUZ3MHlOVEF4TVRrd016QTRNVEphTURReApGekFWQmdOVkJBb1REbk41YzNSbGJUcHRZWE4wWlhKek1Sa3dGd1lEVlFRREV4QnJkV0psY201bGRHVnpMV0ZrCmJXbHVNSUlCSWpBTkJna3Foa2lHOXcwQkFRRUZBQU9DQVE4QU1JSUJDZ0tDQVFFQTJFcFVkcDNscEhXcWFNalYKNE4xaldUS2xQZTVHU1NNeHZlazY5UngrMU5lTkhOMHl3cVVZZlpGNG5zZzQ2TzRhSXJLOEVnVGlLVmxzRHBBeQpvL2pCNExjVzRjSm5KTkttUG8wUmRkazRWc2hLZDhCdWludmdGMVdQSXpiZ0UwNmV5NFdYU3NqK3piays2eHFMCmRUWlZ0UmtMUDE1VEtnazNEWkVZb1ZZcit4dnk3L2hpd2IrMmtqYmkwUm8vZTBWTVBqRjQyRFhRdWV3eTFMSUEKK3JXRkNlekhEUno4NmtSZXZLSlVkdElaQmh3ekpHd0lNSmhHMHdTdU9QeDRXSXN6UEJ1MzNaMCtNbGk3NWFYZQpJS1lvSS9BVEVna3JGVC9VVWN3UjFzV2EvN1JkeUl4TVJlazBQaCt1UEVGMzhnbmVqQXZzay91aVdXcVFEZzVRCmZDOE1ZUUlEQVFBQm8xWXdWREFPQmdOVkhROEJBZjhFQkFNQ0JhQXdFd1lEVlIwbEJBd3dDZ1lJS3dZQkJRVUgKQXdJd0RBWURWUjBUQVFIL0JBSXdBREFmQmdOVkhTTUVHREFXZ0JRalhwK1hZanBwTExHQVZSUzg1dE9uZ0t0TgpaakFOQmdrcWhraUc5dzBCQVFzRkFBT0NBUUVBYm9BMUJQZ1VEN2hJSDNWeXpwMjFOcmVTTHk0em5pNXlMWWNHCnlGYXd3ZU5GWTJRY0s4dzMybjF3RnduWWl4MndWRUNGT0czQW1Td3JkTkpTSDRwdjdyUk16SHYrUzh3aDFiKzEKVXREN3BVU2lWL2JDdEo1WnFzeWp5VXY3UzNydFVjK1l1YjZlZjk2UC9XWWtNSEJVN3gwRmF6MmpUeW9JbXp2SApHNTg3TkJkV002MEx1dDZOaWJxOWRrVUY4d0F0UWpERG14aXh4aW9Jc0tuaEpZQzJmWnVHTzM0c3VNemhQSnl2CmhsVU1TZ1RFblNPMXE2NDZ5d3V5MmhFM0dQQUUwMU1IU0VuVkUvK3pwT25UZTM3YjBsbzRIdSsyZEx2emtMeFAKNUdxN2Q3Z09uaTBHOGV6VWJDOUFLR2ZyYWFVYTVuT2pQZ1RMRmNRc05sS0w5MTlCSkE9PQotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==
    client-key-data: LS0tLS1CRUdJTiBSU0EgUFJJVkFURSBLRVktLS0tLQpNSUlFb3dJQkFBS0NBUUVBMkVwVWRwM2xwSFdxYU1qVjROMWpXVEtsUGU1R1NTTXh2ZWs2OVJ4KzFOZU5ITjB5CndxVVlmWkY0bnNnNDZPNGFJcks4RWdUaUtWbHNEcEF5by9qQjRMY1c0Y0puSk5LbVBvMFJkZGs0VnNoS2Q4QnUKaW52Z0YxV1BJemJnRTA2ZXk0V1hTc2oremJrKzZ4cUxkVFpWdFJrTFAxNVRLZ2szRFpFWW9WWXIreHZ5Ny9oaQp3Yisya2piaTBSby9lMFZNUGpGNDJEWFF1ZXd5MUxJQStyV0ZDZXpIRFJ6ODZrUmV2S0pVZHRJWkJod3pKR3dJCk1KaEcwd1N1T1B4NFdJc3pQQnUzM1owK01saTc1YVhlSUtZb0kvQVRFZ2tyRlQvVVVjd1Ixc1dhLzdSZHlJeE0KUmVrMFBoK3VQRUYzOGduZWpBdnNrL3VpV1dxUURnNVFmQzhNWVFJREFRQUJBb0lCQUhaRnB3dzU2WUpWNlhwbApJRGRYT0dWbFFXQ3RNL2Y0YTlIYWdLZmFEaXpiTmNucjF6OEN3bktmb3FMSS8vMjNmY2t0alpRWTRZY1U0L2JPCnVUSmE2OEd5dkt0MC82dnVHSVFwNWJ0WXJlc2VtVUlFa3kzYzhUd3hTQlZNZzVscks0QkZLK2IwSkFsZzI5djUKNXZxUVhLdXI1eStlcDhGYnlxUDdqTWxrY3FaYnIyVWZFQ096cXFRSTN0SjBqVGZmdmt5dEZZNFRSRWVoU0J5SQpkOFNtVVJldnR6S3R3Q0ZWMTlIVjV3ZWhUWmovU3VCaTFybGFwaTQrUHBEeERLdFNBMWRWOEFVUUxmNTJvMitaClUwc2x0TnFGcmRtSlpvZ21qU2ltU1NMRFVXcjBoRjlGZDVwYUM3b3pKc0d5VVZxUUFWOUhVT0pHVDlpZ3p4ZHAKMCsxVUdVVUNnWUVBM2grU2tRbFlPMHo4N2R6c01UTUFSWExxMklBK3BJN0pHQ0Fyd1RhUEd1aThwNVlPWkdBeQpQbm1sbEh6R1VIWDJ6QXBxVHNzSE1Fak1WM2oxZFdwSlBIS0VQWHB3NHUvQTFQTkJUWWZDVmI2eUl3U1pOaUNqCmJVUDVlZmxyeGlpbXN3S1AvU3pNeTFwVFpwVFJlMElIbVdaamI3MVEzNHlNeldJVDlPOGczOE1DZ1lFQStVY0UKeDltSmRlTHFIWS9pRkpERTBsdXJ3d0NsSzVjekJaTisvdzlaUGZuV0I4MlpyL3ZKbytramo2N2svaXdzbXEvUQpiU3JsRjF5NGFrem4vWDNzQmdXZmthNmtkMVJOKzMxZTFpZ2xFQ0o5WEozL1paNCsrSm0zaFdRdUxTeVBubFM4CkFTWm1lc29oNVA4emoybzZvZWNnWjFiMGQxdlEvMEQ3M3A0MDVRc0NnWUJ0bWtLbUVtaFpDb29iak5GM0RXVnEKMzJPR1pQR0VIWGlZMFBjR0piZkRYV2dKZ1gra2c5c0cvTnQ1UTRCUG40V2g5Tm16KzNhV21yVkp6RVBDSmlueApDOGk0MVR2eW5yOFYxTm82T1d6cEJtbTc0YjcvK0dicnVZaldhUDZIRHZRQ2pKY2tKQUVCcnBaTW5jNG45ZEx1CkhKbWdQMWd5bHBXN21sT2lub1FvSlFLQmdHeE8waUh2UDgyTHdVTUU4Q3NWVjU4Nm0xK0gyVHdlWHRuT1kwQjUKTDhKQTJpRGIwU25va1l6NVVDMHV4V28yVVU4SWt0dkw1bXdIS2sxdGl1TFdJb1hmVFp5anIrdjFJa2ppQ1NHdApvYVRvQjJZRmRDRjM1MDVtbzVsK2xKMm1IZVNpVm1sOWdNdGJKZXowZ1RlUDVWZlJMNEFYQlBNVFhyUjVUTFpHCk1SOVBBb0dCQUtCcStaUGZ4em55ZFlESHJ0cXhlWjR3UzVRS1BySGduSU43T3ZzTlFZNEFjUjJ0RUw3dHFjb1gKTWU4b09oQTcxVkJXV25EL3Y5NktaZDB2SU9ONmZva21UL3dIckxCTElaVXRMTzVES1J4NlJnUjRrMDI2TUtTdwoyeHd4U21EYThCNi9RWjJnVkJpZmhCank1SGQzM1MrMHBMRHVNSGYvenRvY2hnK3EyVmQ2Ci0tLS0tRU5EIFJTQSBQUklWQVRFIEtFWS0tLS0tCg==
- name: minikube
  user:
    client-certificate: C:\Users\hunglp\.minikube\profiles\minikube\client.crt
    client-key: C:\Users\hunglp\.minikube\profiles\minikube\client.key

```

- B6: Tại máy host, kiểm tra thông tin config
```shell
        C:\Users\hunglp\.kube> kubectl config get-contexts
          CURRENT   NAME                          CLUSTER          AUTHINFO           NAMESPACE
          docker-desktop                docker-desktop   docker-desktop
*         kubernetes-admin@kubernetes   kubernetes       kubernetes-admin
          minikube                      minikube         minikube           default
```
- B7: Context hiện tại đang là : kubernetes-admin@kubernetes là context kubenetes ở máy ảo master
    +  lệnh swith context : `kubectl config use-context docker-desktop`

# 2. Kubernetes Dashboard [Code demo ở nhánh : part2-k8s-dashboard]
- B1. Tại folder dashboard: tạo file dashboard-v2-beta6.yaml
- B2. Sau đó chạy lệnh sau: `kubectl apply -f dashboard-v2-beta6.yaml`
- B3. Xác thực SSL
```shell
openssl req -nodes -newkey rsa:2048 -keyout certs/dashboard.key -out certs/dashboard.csr -subj
openssl x509 -req -sha256 -days 365 -in certs/dashboard.csr -signkey certs/dashboard.key -out certs/dashboard.crt
```
- B4. Tạo secret từ các file trong thư mục certs, và trong namespace là kubernetes-dashboard
`kubectl create secret generic kubernetes-dashboard-certs --from-file=certs -n kubernetes-dashboard`
- B5. Kiểm tra: 
```shell
$ kubectl get secret -n kubernetes-dashboard
NAME                              TYPE     DATA   AGE
kubernetes-dashboard-certs        Opaque   3      54s
kubernetes-dashboard-csrf         Opaque   1      15m
kubernetes-dashboard-key-holder   Opaque   0      15m
```