# 容器安全

Kubernetes 提供了多种机制来限制容器的行为，减少容器攻击面，保证系统安全性。

- Security Context：限制容器的行为，包括 Capabilities、ReadOnlyRootFilesystem、Privileged、RunAsNonRoot、RunAsUser 以及 SELinuxOptions 等
- Pod Security Policy：集群级的 Pod 安全策略，自动为集群内的 Pod 和 Volume 设置 Security Context
- Sysctls：允许容器设置内核参数，分为安全 Sysctls 和非安全 Sysctls
- AppArmor：限制应用的访问权限
- Seccomp：Secure computing mode 的缩写，限制容器应用可执行的系统调用

## Security Context 和 Pod Security Policy

请参考 [这里](../concepts/security-context.md)。

## Sysctls

Sysctls 允许容器设置内核参数，分为安全 Sysctls 和非安全 Sysctls

- 安全 Sysctls：即设置后不影响其他 Pod 的内核选项，只作用在容器 namespace 中，默认开启。包括以下几种
  - `kernel.shm_rmid_forced`
  - `net.ipv4.ip_local_port_range`
  - `net.ipv4.tcp_syncookies`
- 非安全 Sysctls：即设置好有可能影响其他 Pod 和 Node 上其他服务的内核选项，默认禁止。如果使用，需要管理员在配置 kubelet 时开启，如 `kubelet --experimental-allowed-unsafe-sysctls 'kernel.msg*,net.ipv4.route.min_pmtu'`

Sysctls 还在 alpha 阶段，需要通过 Pod annotation 设置，如：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sysctl-example
  annotations:
    security.alpha.kubernetes.io/sysctls: kernel.shm_rmid_forced=1
    security.alpha.kubernetes.io/unsafe-sysctls: net.ipv4.route.min_pmtu=1000,kernel.msgmax=1 2 3
spec:
  ...
```

## AppArmor

[AppArmor(Application Armor)](http://wiki.apparmor.net/index.php/AppArmor_Core_Policy_Reference) 是 Linux 内核的一个安全模块，允许系统管理员将每个程序与一个安全配置文件关联，从而限制程序的功能。通过它你可以指定程序可以读、写或运行哪些文件，是否可以打开网络端口等。作为对传统 Unix 的自主访问控制模块的补充，AppArmor 提供了强制访问控制机制。

在使用 AppArmor 之前需要注意

- Kubernetes 版本 >=v1.4
- apiserver 和 kubelet 已开启 AppArmor 特性，`--feature-gates=AppArmor=true`
- 已开启 apparmor 内核模块，通过 `cat /sys/module/apparmor/parameters/enabled` 查看
- 仅支持 docker container runtime
- AppArmor profile 已经加载到内核，通过 `cat /sys/kernel/security/apparmor/profiles` 查看

AppArmor 还在 alpha 阶段，需要通过 Pod annotation `container.apparmor.security.beta.kubernetes.io/<container_name>` 来设置。可选的值包括

- `runtime/default`: 使用 Container Runtime 的默认配置
- `localhost/<profile_name>`: 使用已加载到内核的 AppArmor profile

```sh
$ sudo apparmor_parser -q <<EOF
#include <tunables/global>

profile k8s-apparmor-example-deny-write flags=(attach_disconnected) {
  #include <abstractions/base>

  file,

  # Deny all file writes.
  deny /** w,
}
EOF'

$ kubectl create -f /dev/stdin <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: hello-apparmor
  annotations:
    container.apparmor.security.beta.kubernetes.io/hello: localhost/k8s-apparmor-example-deny-write
spec:
  containers:
  - name: hello
    image: busybox
    command: ["sh", "-c", "echo'Hello AppArmor!'&& sleep 1h"]
EOF
pod "hello-apparmor" created

$ kubectl exec hello-apparmor cat /proc/1/attr/current
k8s-apparmor-example-deny-write (enforce)

$ kubectl exec hello-apparmor touch /tmp/test
touch: /tmp/test: Permission denied
error: error executing remote command: command terminated with non-zero exit code: Error executing in Docker Container: 1
```

## Seccomp

[Seccomp](https://www.kernel.org/doc/Documentation/prctl/seccomp_filter.txt) 是 Secure computing mode 的缩写，它是 Linux 内核提供的一个操作，用于限制一个进程可以执行的系统调用．Seccomp 需要有一个配置文件来指明容器进程允许和禁止执行的系统调用。

在 Kubernetes 中，需要将 seccomp 配置文件放到 `/var/lib/kubelet/seccomp` 目录中（可以通过 kubelet 选项 `--seccomp-profile-root` 修改）。比如禁止 chmod 的格式为

```sh
$ cat /var/lib/kubelet/seccomp/chmod.json
{
    "defaultAction": "SCMP_ACT_ALLOW",
    "syscalls": [
        {
            "name": "chmod",
            "action": "SCMP_ACT_ERRNO"
        }
    ]
}
```

Seccomp 还在 alpha 阶段，需要通过 Pod annotation 设置，包括

- `security.alpha.kubernetes.io/seccomp/pod`：应用到该 Pod 的所有容器
- `security.alpha.kubernetes.io/seccomp/container/<container name>`：应用到指定容器

而 value 有三个选项

- `runtime/default`: 使用 Container Runtime 的默认配置
- `unconfined`: 允许所有系统调用
- `localhost/<profile-name>`: 使用 Node 本地安装的 seccomp，需要放到 `/var/lib/kubelet/seccomp` 目录中

比如使用刚才创建的 seccomp 配置：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: trustworthy-pod
  annotations:
    seccomp.security.alpha.kubernetes.io/pod: localhost/chmod
spec:
  containers:
    - name: trustworthy-container
      image: sotrustworthy:latest
```
