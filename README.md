
# 1.Tạo kubernetes cluster
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