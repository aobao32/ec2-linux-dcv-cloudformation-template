# Ubuntu DCV 安全增强云桌面部署操作步骤

本文档描述如何通过 CloudFormation 模版，一键拉起 EC2 虚拟机，机型为 G6.2xlarge，使用 L4 GPU（24GB显存）。机器内预先安装了面向3D工作站的 Nvidia Grid 图形驱动，安装了 DCV Server，并配置了剪切板单向复制、禁止截屏的安全策略。由此可使用 DCV Client 直接连接。

以下为客户侧操作步骤。模版文件为 `EC2-Ubuntu-DCV-existing-vpc-security-enhanced-updated-tag.yaml`，默认 AMI 为 `ami-042dc8681de073ac4`，仅在 **eu-central-1（法兰克福）** 有效，其他区域必须替换该参数值。

## 一、上传并创建堆栈

### 1、进入创建向导

1. 登录 AWS 控制台，右上角区域选择器切换到 **Europe (Frankfurt) eu-central-1**。
2. 搜索栏输入 `CloudFormation`，进入服务，左侧菜单点 **Stacks**。
3. 点右上角 **Create stack** → 下拉选 **With new resources (standard)**。

### 2、指定模版

模版地址：[https://github.com/aobao32/ec2-linux-dcv-cloudformation-template/blob/main/EC2-Ubuntu-DCV-existing-vpc-security-enhanced-updated-tag.yaml](https://github.com/aobao32/ec2-linux-dcv-cloudformation-template/blob/main/EC2-Ubuntu-DCV-existing-vpc-security-enhanced-updated-tag.yaml)

1. Prerequisite 区域选 **Choose an existing template**。
2. Specify template 区域选 **Upload a template file**，点 **Choose file**，选择本地的 `EC2-Ubuntu-DCV-existing-vpc-security-enhanced-updated-tag.yaml`。
3. 点右下角 **Next**。

## 二、填写堆栈参数

### 1、Stack name

填写堆栈名称，例如 `dcv-secure-desktop-01`。该名称会成为安全组名前缀。

### 2、EC2 configuration

| 参数 | 填写内容 |
| --- | --- |
| EC2 instance name | 保留默认 `Ubuntu 24.04 DCV Secure Desktop 01`，或自定义 |
| G6-family instance type | 保留默认 `g6.2xlarge` |
| Canonical Ubuntu 官方镜像 24.04 LTS AMI | 在 eu-central-1 保留默认值 |

### 3、Network configuration

1. **Existing VPC**：下拉选择目标 VPC。
2. **Existing subnet**：下拉选择该 VPC 内的 Private Subnet 私有子网（内网）。该子网必须具备出站网络路径（NAT Gateway），否则驱动与 DCV 安装包无法下载，堆栈会创建失败。

### 4、Remote access

**Allowed DCV client IPv4 CIDR**：默认 `10.0.0.0/8` 仅允许具备路由路径的 RFC 1918 私有地址范围访问；仍建议填写办公区的实际客户端网段，格式为 `<单个IP>/32` 或办公网段 `<网段>/24`。

### 5、Storage and driver

| 参数 | 填写内容 |
| --- | --- |
| Root volume size (GiB) | 默认 `100`，可在 50 到 16384 之间调整 |
| AWS NVIDIA GRID driver S3 prefix | 保留默认 `s3://ec2-linux-nvidia-drivers/grid-19.5/`，这里不要改动，否则安装会失败 |

填写完成后点 **Next**。

## 三、确认权限并提交

1. Configure stack options 页面无需修改，滚动到页面底部。
2. 勾选 **I acknowledge that AWS CloudFormation might create IAM resources**。该模版会创建 IAM 角色与实例配置文件，不勾选无法提交。
3. 点 **Next**，在 Review 页面核对参数，点 **Submit**。

## 四、等待部署完成

1. 堆栈进入 `CREATE_IN_PROGRESS`。引导脚本分两个阶段执行，中间包含一次实例重启，用于安装 GNOME 桌面、NVIDIA GRID 驱动与 Amazon DCV Server。
2. 在堆栈详情页的 **Events** 标签页可查看进度，模版设定的等待上限为 120 分钟。
3. 状态变为 **CREATE_COMPLETE** 后方可连接。若变为 `CREATE_FAILED`，在 Events 中查看失败原因，实例内的日志位于 `/var/log/dcv-bootstrap-phase1.log` 与 `/var/log/dcv-bootstrap-phase2.log`。

## 五、查看登录字符串

1. 在堆栈详情页点 **Outputs** 标签页。
2. 找到 **DcvNativeClientUrl**，其值形如 `dcv://dcvuser@10.0.1.23:8443`，这就是登录字符串。
3. 同页 **DesktopUserName** 为登录用户名 `dcvuser`，**PrivateIp** 为实例私有地址。

## 六、查看登录密码

### 1、通过控制台查看

1. 在 CloudFormation 控制台当前 Stack 堆栈的标签页上，进入 **Outputs** 标签页复制 **DcvCredentialSecretArn** 的值。
2. 控制台搜索栏输入 `Secrets Manager`，进入服务，左侧点 **Secrets**。
3. 在列表中点击与该 ARN 对应的密钥名称。
4. 在 **Overview** 标签页下的 Secret value 区域点 **Retrieve secret value**。
5. 页面显示 `username` 为 `dcvuser`，`password` 为随机生成的 24 位密码。

## 七、以普通用户身份连接桌面

1. 下载 DCV 客户端：[https://www.amazondcv.com/](https://www.amazondcv.com/)
2. 打开本机的 Amazon DCV 原生客户端。
3. 在连接地址栏粘贴 **DcvNativeClientUrl** 的值，或手工填写 `<PrivateIp>:8443`。
4. 用户名填 `dcvuser`，密码填第六步取得的值，连接后直接进入已自动登录的 GNOME 会话。
5. 该会话已按模版策略禁用截图、服务端到客户端的剪贴板复制、文件下载与打印，保留客户端到服务端的粘贴与文件上传。

## 八、以管理员权限登录实例、及镜像软件管理

镜像软件管理。

### 1、由专门的桌面管理员为用户安装软件和更新镜像

`dcvuser` 不在 sudo 组内，桌面会话中无法执行管理操作。需要管理员安装软件包等权限时，管理员按以下步骤从 EC2 控制台进入 Session Manager。

1. 控制台搜索栏输入 `EC2`，进入服务，左侧菜单点 **Instances**。
2. 在实例列表中勾选目标实例，实例名为参数 **EC2 instance name** 的值，也可用堆栈 **Outputs** 中的 **InstanceId** 核对。
3. 点击页面上方的 **Connect** 按钮。
4. 在 Connect to instance 页面选择 **Session Manager** 标签页。
5. 点右下角 **Connect**，浏览器会打开一个终端会话，当前身份为 `ssm-user`。
6. `ssm-user` 具备免密 sudo 权限，执行管理命令时加 `sudo`，或先执行 `sudo -i` 切换到 root。

注意：`ubuntu` 账户的密码已被锁定，不能用于 DCV 登录、GNOME 登录或 su 切换，Session Manager 是唯一的管理通道。因此应在 IAM 中把 `ssm:StartSession` 限定给运维人员使用。若 **Session Manager** 标签页的 Connect 按钮为灰色，说明实例尚未完成引导或 SSM Agent 未上线，等待堆栈状态变为 `CREATE_COMPLETE` 后重试。

用root身份装好软件后，可以把这个EC2做一个AMI镜像，然后在创建别的桌面时候，在CloudFormation模版启动的那个步骤，询问镜像ID，用你自己创建的AMI ID替换掉默认的。于是新创建出来的桌面，镜像就是刚才新镜像了。

### 2、临时下放 sudo 权限给用户、操作完毕后管理员回收

如果需要临时下放 sudo 权限给云桌面 dcvuser 用户，完毕后还要回收，并且要核验 CloudFormation 模版中配置的 DCV 服务端配置仍然保留，重点核验防截屏与单向剪贴板复制这两组参数是否被改动。所有操作均通过第 1 节介绍的 Session Manager 管理通道执行，`dcvuser` 本身无权完成这些操作。

注意：临时授予 sudo 权限存在明确的安全风险。`dcvuser` 一旦进入 sudo 组，即可改写 `/etc/dcv/default.perm`、`/etc/dcv/dcv.conf` 与 `/etc/dconf/` 下的锁定文件，从而绕过防截屏、单向剪贴板复制、禁止文件下载与打印等全部安全策略。因此本节强调"最小时间窗口授予、操作完毕立即回收、回收后逐项核验"的闭环流程，不建议长期保留该权限。

下面依次介绍授权、回收与配置核验环节。

#### (1) 临时授予 sudo 权限

在 root 身份下，将 `dcvuser` 临时加入 sudo 组。执行如下命令：

```shell
usermod -aG sudo dcvuser
id dcvuser
```

返回结果如下：

```shell
uid=1001(dcvuser) gid=1001(dcvuser) groups=1001(dcvuser),44(video),27(sudo)
```

输出中出现 `27(sudo)` 表示授权成功。组变更需要 `dcvuser` 重新登录 DCV 会话后才对新开的终端生效。授权后应立即通知用户在限定时间窗口内完成安装软件、更新系统等操作。

备注：若只需授予单条命令而非完整 sudo 权限，更安全的做法是通过 `visudo -f /etc/sudoers.d/dcvuser-temp` 写入精确到命令的白名单，例如仅允许 `apt-get`，从而缩小权限边界。操作完毕后删除该文件即可回收。

#### (2) 操作完毕后回收 sudo 权限

用户操作完毕后，管理员在 root 身份下把 `dcvuser` 移出 sudo 组。执行如下命令：

```shell
gpasswd -d dcvuser sudo
id dcvuser
```

返回结果如下：

```shell
Removing user dcvuser from group sudo
uid=1001(dcvuser) gid=1001(dcvuser) groups=1001(dcvuser),44(video)
```

输出中不再出现 `27(sudo)` 表示回收成功。如果第 (1) 步使用的是 `/etc/sudoers.d/dcvuser-temp` 白名单方式，则改为执行 `rm -f /etc/sudoers.d/dcvuser-temp` 完成回收。回收后需强制结束 `dcvuser` 当前仍持有旧权限的会话，执行如下命令：

```shell
pkill -KILL -u dcvuser
```

#### (3) 回收后核验安全配置未被篡改

回收权限后，管理员在 root 身份下直接查看两处 DCV 配置文件的内容，逐项核对防截屏与单向剪贴板复制参数是否与模版原始配置一致。先查看 DCV 服务端权限文件，执行如下命令：

```shell
cat /etc/dcv/default.perm
```

返回结果应与下方模版原始内容完全一致：

```shell
[permissions]
%any% deny screenshot clipboard-copy file-download printer
%owner% allow builtin
```

其中 `deny screenshot` 是防截屏参数，`deny clipboard-copy` 是禁止服务端向客户端复制；由于 DCV 的 `deny` 规则无法被后续语句覆盖，该规则对会话属主 `dcvuser` 同样生效。若该文件被改动，需按上述内容恢复。

接下来查看剪贴板方向控制文件 `/etc/dcv/dcv.conf`，执行如下命令：

```shell
cat /etc/dcv/dcv.conf
```

返回结果中的 `[clipboard]` 小节应与下方内容一致：

```shell
[clipboard]
enabled=true
primary-selection-copy=false
primary-selection-paste=false
max-payload-size-paste=1048576
```

配合 `default.perm` 中拒绝 `clipboard-copy`、保留 `clipboard-paste`，即构成客户端到服务端单向粘贴、服务端到客户端禁止复制的单向剪贴板行为。GNOME 层面的截屏与录屏锁定位于 `/etc/dconf/db/local.d/`，查看该目录下的锁定文件是否仍然存在，执行如下命令：

```shell
cat /etc/dconf/db/local.d/00-dcv-lockdown /etc/dconf/db/local.d/locks/dcv-lockdown
```

若该目录文件被改动或缺失，恢复后必须执行如下命令使其重新生效：

```shell
dconf update
```

核验通过后，需重启 DCV 服务端使配置文件确保加载生效。执行如下命令：

```shell
systemctl restart dcvserver.service
systemctl status dcvserver.service --no-pager
```

返回结果中 `Active: active (running)` 表示服务正常。至此临时授权、回收与核验流程闭环完成。建议管理员在完成核验后，通过 DCV 客户端以 `dcvuser` 身份实测一次截屏与复制粘贴，确认安全策略确实恢复生效。

下面转向参考文档。

## 九、参考文档

Amazon DCV 授权与权限文件（default.perm）的官方说明，包含 allow、disallow、deny 三种规则的区别，以及本文使用的 default.perm 默认路径 `/etc/dcv/default.perm`：

[https://docs.aws.amazon.com/dcv/latest/adminguide/security-authorization.html](https://docs.aws.amazon.com/dcv/latest/adminguide/security-authorization.html)

Amazon DCV 权限项详解，包含 screenshot、clipboard-copy、clipboard-paste、file-download、printer 等权限的含义，以及 screenshot 需要 DCV 2021.2 及以上版本、拒绝 screenshot 同时应禁用 clipboard-copy 的官方建议：

[https://docs.aws.amazon.com/dcv/latest/adminguide/security-authorization-file-create-permission.html](https://docs.aws.amazon.com/dcv/latest/adminguide/security-authorization-file-create-permission.html)

Linux Amazon DCV 服务端剪贴板配置官方说明，对应本文 `/etc/dcv/dcv.conf` 中 `[clipboard]` 小节的 primary-selection-copy、primary-selection-paste 等参数：

[https://docs.aws.amazon.com/dcv/latest/adminguide/manage-clipboard.html](https://docs.aws.amazon.com/dcv/latest/adminguide/manage-clipboard.html)

Amazon DCV 客户端下载页面：

[https://www.amazondcv.com/](https://www.amazondcv.com/)

AWS Systems Manager Session Manager 官方文档，本文用作实例的唯一管理通道：

[https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager.html](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager.html)
