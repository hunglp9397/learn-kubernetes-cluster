
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
- B2: Truy cập folder _[virtual_machine_config/kubernetes-centos7/master]_. Chạy lệnh: `vagrant up` (Nếu lỗi có thể xem hướng dẫn ở project vagrant_guide)
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
    + ở đây Trong lệnh khởi tạo cluster có tham số --pod-network-cidr để chọn cấu hình mạng của POD, do dự định dùng Addon calico nên chọn--pod-network-cidr=192.168.0.0/16
    + Nếu gặp lỗi khi chạy lệnh kubeadm init này thì chạy thêm 2 lệnh sau rồi chạy lại kubeadm init:
    `rm /etc/containerd/config.toml`
    `systemctl restart containerd`

- B2 : Sau khi lệnh chạy xong, chạy tiếp cụm lệnh nó yêu cầu chạy sau khi khởi tạo- để chép file cấu hình đảm bảo trình kubectl trên máy này kết nối Cluster
```shell
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

- B3: Tiếp đó, nó yêu cầu cài đặt một Plugin mạng trong các Plugin tại addon, ở đây đã chọn calico, nên chạy lệnh sau để cài nó
`kubectl apply -f https://docs.projectcalico.org/v3.10/manifests/calico.yaml`

- B4: Kiểm tra cluster:
```shell
# Thông tin cluster
kubectl cluster-info
# Các node trong cluster
kubectl get nodes
# Các pod đang chạy trong tất cả các namespace
kubectl get pods -A
```
![2](/img_guide/2.png)

- B5: Mở kết nối của máy master
    + Tại máy master, Thực hiện lệnh sau với Cluster để lấy lệnh kết nối: `kubeadm token create --print-join-command`
    + Kết quả:
        ```shell
        [root@master ~]# kubeadm token create --print-join-command
        kubeadm join 172.16.10.100:6443 --token n7olii.yro8krwj96ddnz6x --discovery-token-ca-cert-hash sha256:398a0bacced59719fab0e7eab6e12ca3af546be11923308290db96838937a21d  
        ```

- B6: Kết nối các các máy worker tới máy master
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

# 2. Gán nhãn cho node (ko quan trọng lắm có thể bỏ qua)
  - Chạy lệnh sau để gán nhãn cho kube-worker-1: `kubectl label node kube-worker-1 nodeabc=dechayungdungphp`
  - Kiểm tra : 
```shell
  PS C:\Users\hunglp> kubectl describe no kube-worker-1
  Name:               kube-worker-1
  Roles:              <none>
  Labels:             beta.kubernetes.io/arch=amd64
                      beta.kubernetes.io/os=linux
                      kubernetes.io/arch=amd64
                      kubernetes.io/hostname=kube-worker-1
                      kubernetes.io/os=linux
                      nodeabc=dechayungdungphp
```

# 3. Ví dụ về pod, Và truy cập giữa các pod với nhau, Và truy cập từ máy host tới pod
- Tạo file 1-swarm-test-node.yaml
- Apply file manifest yaml : `kubectl apply -f 1-swarm-test-node.yaml -n kube-system`, `kubectl  apply -f 2-nginx.yaml -n kube-system`, `kubectl apply -f 3-tools.yaml -n kube-system`
- Kết quả:
![kết quả 3](/img_guide/3.png)
- Giả sử truy cập vào container của pods tools : `kubectl exec -it po/tools bash`
- Tại command line của container tools, Truy cập đến cổng 8080 của container chạy pods nginxapp, Kết quả như sau:
![kết quả 3](/img_guide/4.png)
- Tương tự, tại Command line của containers tools, Truy cập đến cổng 8085 của container chạy pods ungdungnode, Kết quả như sau:
```shell
  root@tools:/# curl 10.1.1.164:8085
  Swarm serive (Node App), hostname=ungdungnoderoot@tools:/#
```
- Tại máy host, để truy cập vào container trong cluster cần lệnh chạy lệnh sau ở máy host : `kubectl proxy --accept-hosts='.*'`
- Để truy cập từ máy host vào container nginxapp(chạy cổng 8080) : `http://localhost:8001/api/v1/namespaces/default/pods/nginxapp/proxy/`
- Để truy cập từ máy host vào container ungdungnode(chạy cổng 8085) : `http://localhost:8001/api/v1/namespaces/default/pods/ungdungnode:8085/proxy/`

# 4. Ví dụ về pod chạy nhiều containers & Pod with volumes
## 4.1 Pod chạy nhiều container
- Tạo file 4-nginx-swarmtest.yaml và apply 
- Trong container nginx-swarmtest vừa mới được tạo, thì có 2 container là n1 và s1
- Nếu truy cập vào container chạy pods này, Vì ta có 2 container chạy nên nó mặc định exec truy cập container đầu tiên trừ trên xuống dưới : `kubectl exec nginx-swarmtest ls / `
- Trong trường hợp muốn chạy lệnh ls (lệnh list file ) và chỉ rõ container nào, thì cần chạy lệnh sau : `kubectl exec nginx-swarmtest ls / -c s1`
- Tương tự để vào được container nào đó, cần chỉ rõ là container nào muốn truy cập vào, Ví dụ : `kubectl exec -it nginx-swarmtest bash -c n1`
- Để truy cập từ máy host thì cần chạy `kubectl proxy --accept-hosts='.*'` 
  + Truy cập container s1 chạy cổng 8085 : `http://localhost:8001/api/v1/namespaces/default/pods/nginx-swarmtest:8085/proxy/`
  + Truy cập container n1 chạy cổng 8080 : `http://localhost:8001/api/v1/namespaces/default/pods/nginx-swarmtest/proxy/`


## 4.2 Pod with Volumes
- Tạo file 5-nginx-swamtest-vol.yaml và apply 
- Để truy cập từ máy host vào container nginx-swarmtest-vol : `http://localhost:8001/api/v1/namespaces/default/pods/nginx-swarmtest-vol/proxy/`
- Nếu báo lỗi 403 thì do trong config khai báo volume chưa có file html :
```shell
    # Định nghĩa một volume - ánh xạ thư mục /home/www máy host
    - name: "myvol"
      hostPath:
          path: "/home/html"
```

- Do vậy cần truy cập vào máy worker chạy container này, và tạo file index.html tại đường dẫn /home/html/index.html:
```shell
ssh generic@192.168.177.3
generic@kube-worker-1:~$ cd /home/html
generic@kube-worker-1:/home/html$ ls
index.html
```

# 5. Replicaset 
## 5.1 Ví dụ Replicaset
- Tạo file 1.app.yaml trong folder rs
- Tạo file 2.rs.yaml trong folder rs
- Apply 2 file yaml  
- Lý thuyết về Replicaset:
  + Về cơ bản thì replicaset chỉ quản lý các pod dựa trên labels, ví dụ trong file 2.rs.yaml có khai báo như sau:
```shell
spec:
  replicas: 3
  selector:
    matchLabels:
      app: rsapp
```

- Để kiểm tra replicaset : `kubectl get rs -o wide`
- Để xuất file manifest của replicaset : `kubectl get rs -o yaml`
- Để xem thông tin chi tiết của replicaset : `kubectl describe rs/rsapp`
- Để hiển thị các pods của 1 labels nào đó : `kubectl get pod -l "app=rsapp"`

## 5.2 Horizontal Pod AutoScaler
- Trong folder rs. Tạo file 2.hpa.yaml
- Apply file này : `kubectl apply -f 2.hpa.yaml`
- Kết quả:
```shell
PS C:\Users\hunglp> kubectl get all -o wide
NAME                      READY   STATUS    RESTARTS      AGE     IP            NODE            NOMINATED NODE   READINESS GATES
pod/nginx-swarmtest-vol   2/2     Running   4 (24h ago)   10d     10.0.180.20   kube-worker-1   <none>           <none>
pod/rsapp-59h8v           1/1     Running   0             91m     10.0.180.21   kube-worker-1   <none>           <none>
pod/rsapp-bl5hw           0/1     Pending   0             3m24s   <none>        <none>          <none>           <none>
pod/rsapp-fw9jx           1/1     Running   0             29m     10.0.180.23   kube-worker-1   <none>           <none>
pod/rsapp-hkw6p           1/1     Running   0             3m24s   10.0.180.24   kube-worker-1   <none>           <none>
pod/rsapp-jdgll           1/1     Running   0             91m     10.0.180.22   kube-worker-1   <none>           <none>

NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE   SELECTOR
service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   12d   <none>

NAME                    DESIRED   CURRENT   READY   AGE   CONTAINERS   IMAGES                 SELECTOR
replicaset.apps/rsapp   5         5         4       91m   app          ichte/swarmtest:node   app=rsapp

NAME                                               REFERENCE          TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
horizontalpodautoscaler.autoscaling/rsapp-scaler   ReplicaSet/rsapp   <unknown>/50%   5         10        5          3m39s
```

# 6. Deployment trong K8S
## 6.1 Định nghĩa
- Deployment cũng tương tự như replicaset có chức năng sau:
  +  quản lý các pods, nhân bản pods, tự động thay thế pods bị lỗi
  + Đảm bảo các pods ở trong tình trạng đang chạy mặc dù đang sửa đổi, cập nhật, scale
  + Về cơ bản thì deployment tạo ra các replicaset, Replicaset lại quản lí pods
## 6.2 Ví dụ
- Tạo thư mục deploy trong folder exams
- Tạo file 1.myapp-deploy.yaml và apply
- Lệnh kiểm tra các deployment: `kubectl get deploy`

## 6.3 Thực thi các lệnh liên quan đến deployment
- Trong file yaml có khai báo spec là sử dụng image: ichte/swarmtest:node
- Giả sử muốn đổi sang sử dụng image: node thì replicaset (được tạo bởi deployment) sẽ như sau:
```shell
NAME                   DESIRED   CURRENT   READY   AGE     CONTAINERS   IMAGES                 SELECTOR
deployapp-5455b4d455   3         3         3       62s     node         nginx                  app=deployapp,pod-template-hash=5455b4d455
deployapp-6576b6b897   0         0         0       8m25s   node         ichte/swarmtest:node   app=deployapp,pod-template-hash=6576b6b897
```
- Có thể thấy replicaset cũ vẫn giữ nguyên, ko bị xóa, để có thể rollback lại 
- Ngoài cách sửa deployment, thì có thể sửa trong manifest của deployment bằng lệnh: `kubectl edit deploy/deployapp`
- Lệnh get thông tin manifest của deployment : `kubectl get deploy -o yaml`
- Lấy thông tin chi tiết của deployment: `kubectl describe deploy/deployapp`
- Lệnh xem lịch sử cập nhật của deployment: `kubectl rollout history deploy/deployapp`
```shell
Kết quả:
PS C:\Users\hunglp> kubectl rollout history deploy/deployapp
deployment.apps/deployapp
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
3         <none>
4         <none>
```
- Xem chi tiết lần cập nhật thứ 2: `kubectl rollout history deploy/deployapp --revision=2`
```shell
Kết quả:
deployment.apps/deployapp with revision #2
Pod Template:
  Labels:       app=deployapp
        pod-template-hash=5455b4d455
  Containers:
   node:
    Image:      nginx
    Port:       8085/TCP
    Host Port:  0/TCP
    Limits:
      cpu:      100m
      memory:   128Mi
    Environment:        <none>
    Mounts:     <none>
  Volumes:      <none>
```
- Để rollback lại lần cập nhật thứ 2: `kubectl rollout undo deploy/deployapp --to-revision=2`

## 6.4 Scale pods của deployment
- Scale thủ công(ko hay dùng): `kubectl scale deploy/deployapp --replicas=3`
- Scale tự động dựa trên trafic của pods (chính là Horizontal pod autoscaler ở mục 5)
  + `kubectl autoscale deploy/deployapp --min=2 --max=4`

# 7. Metrix server
- Metric server trong k8s giúp giám sát tài nguyên sử dụng trên cluster, Cung cấp các API để các thành phần khác truy vấn đến biết được và mức độ sử dụng tài nguyên (CPU, Memory) của pod, node, 
- Horizontal pod autoscaler cần đến metric server để hoạt động chính xác

# 8. Service
## 8.1 Tổng quan
- Service là một hệt thống cân bằng tải, 1 proxy, một hệ thống cân bằng tải
- Service có một IP
- Khi service nhận được các request, sẽ chuyển hướng các request đó tới các pod mà nó quản lý

## 8.1 Ví dụ 1
### Tạo service
- Tạo folder svc, tạo file 1.svc1.yaml
- Apply file yaml này `kubeclt apply -f 1.svc1.yaml`
- Xuất file yaml của service: `kubectl get svc/svc1 -o yaml`
- Xem chi tiết service : `kubectl describe svc/svc1`

### Tạo endpoint cho service svc1 ở trên
- Tạo file 2.endpoint.yaml
- Apply file yaml này: `kubectl apply -f 2.endpoint.yaml`
- Hiển thị tất cả endpoints: `kubectl get endpoints`
- Xem lại chi tiết serices svc1, Có thể thấy svc1 đã có endpoints: 
```shell
PS C:\Users\hunglp> kubectl describe svc/svc1
Name:              svc1
Namespace:         default
Labels:            <none>
Annotations:       <none>
Selector:          <none>
Type:              ClusterIP
IP Family Policy:  SingleStack
IP Families:       IPv4
IP:                10.98.75.82
IPs:               10.98.75.82
Port:              <unset>  80/TCP
TargetPort:        80/TCP
Endpoints:         216.58.220.195:80
Session Affinity:  None
Events:            <none>
PS C:\Users\hunglp>
```
### Sau khi có endpoint thì từ 1 pods tools (đang ko chạy được) có thể curl tới service svc1 này
- Truy cập command line pods tools : `kubeclt exec -it tools bash`
- Từ cmdline của pods tools, truy cập tới service svc1 : `curl <địa chỉ IP của service svc1>:80` -> Thì sẽ trả ra html của google
## Ví dụ 2 : Thực hành tạo Service có Selector, chọn các Pod là Endpoint của Service
- Tạo file 3.pods.yaml rồi apply ta được 2 pods sau:
```shell
 NAMESPACE↑            NAME        PF         READY                       RESTARTS STATUS                       IP                      NODE                   AGE      
 default              myapp1       ●          1/1                          0 Running                         10.0.180.52             kube-worker-1          13s           
 default              myapp2       ●          1/1                          0 Running                         10.0.180.53             kube-worker-1          13s
```
- Tạo service: 2.svc2.yaml và apply
 + Tại file service này có khai báo selector, selector khai báo nhãn của những pods mà chỉ định làm endpoint của service này
- Kiểm tra service svc2 này : `kubectl describe svc/svc2`
```shell
PS C:\Users\hunglp> kubectl describe svc/svc2
Name:              svc2
Namespace:         default
Labels:            <none>
Annotations:       <none>
Selector:          app=app1
Type:              ClusterIP
IP Family Policy:  SingleStack
IP Families:       IPv4
IP:                10.99.180.209
IPs:               10.99.180.209
Port:              port1  80/TCP
TargetPort:        80/TCP
Endpoints:         10.0.180.52:80,10.0.180.53:80
Session Affinity:  None
Events:            <none>
PS C:\Users\hunglp>
```
- Ta thấy có 2 địa chỉ endpoints này chính là IP của 2 pods myapp1 và myapp2