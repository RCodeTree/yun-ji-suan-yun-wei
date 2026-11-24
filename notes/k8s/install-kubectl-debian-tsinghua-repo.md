# 在 Debian 上通过清华 TUNA 镜像源安装 kubectl（deb 源配置）

> 适用系统：Debian 11（bullseye）/ Debian 12（bookworm）及更新版本
> 本文档以**配置 deb 源**为核心，使用清华大学 TUNA 镜像站提供的 Kubernetes 镜像，并按最新的 APT 规范（`signed-by` + `/etc/apt/keyrings`，废弃 `apt-key`）编写。

---

## 1. 概述

清华 TUNA 镜像站的 Kubernetes 目录镜像了 Kubernetes 官方软件包仓库 `pkgs.k8s.io`，路径结构与官方完全一致。镜像地址为：

```
https://mirrors.tuna.tsinghua.edu.cn/kubernetes/
```

对应官方 `pkgs.k8s.io` 的 `core:/stable:/v1.37/deb/` 路径，在清华镜像上即为：

```
https://mirrors.tuna.tsinghua.edu.cn/kubernetes/core:/stable:/v1.37/deb/
```

本文档中的 `v1.37` 为示例版本号，请按需替换为你的目标版本（如 `v1.36`、`v1.37` 等）。

> **版本兼容性**：kubectl 客户端与集群版本之间的差异必须在一个小版本号以内。例如 `v1.37` 的客户端可与 `v1.36`、`v1.37`、`v1.38` 的控制面通信。

同一个 deb 源同时提供 `kubectl`、`kubeadm`、`kubelet` 三个包，本文以安装 `kubectl` 为例。使用清华镜像可显著提升国内网络环境下的下载速度。

---

## 2. 前置要求

- 具有 `sudo` 权限的非 root 用户或 root 用户
- 能够访问 `https://mirrors.tuna.tsinghua.edu.cn`
- Debian 12 及以上默认已存在 `/etc/apt/keyrings` 目录；Debian 11 需手动创建（见步骤 2）

---

## 3. 步骤 1：更新软件包索引并安装依赖

```bash
sudo apt-get update

# apt-transport-https 在较新版本中可能为虚拟包，可安全跳过
sudo apt-get install -y apt-transport-https ca-certificates curl gnupg
```

---

## 4. 步骤 2：准备 `keyrings` 目录

```bash
# Debian 12 及以上默认已存在，可跳过；Debian 11 需执行：
sudo mkdir -p -m 755 /etc/apt/keyrings
```

---

## 5. 步骤 3：下载并导入 GPG 签名密钥

> **最新规范**：不再使用已废弃的 `apt-key add`，改用 `gpg --dearmor` 生成独立密钥环，并在 deb 源中以 `signed-by` 指定。

```bash
curl -fsSL https://mirrors.tuna.tsinghua.edu.cn/kubernetes/core:/stable:/v1.37/deb/Release.key \
  | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

sudo chmod 644 /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

说明：

- 清华镜像的密钥与官方 `pkgs.k8s.io` 完全一致（镜像的是同一份 `Release.key`）。
- 所有次要版本共用同一个签名密钥，因此 URL 中的 `v1.37` 可以替换为任意版本，密钥相同。

可选：查看导入密钥的指纹以进行核对

```bash
gpg --show-keys --with-fingerprint /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

---

## 6. 步骤 4：编写 deb 源（核心）

以下提供两种书写方式，均为**不依赖 `apt-key` 的最新规范**。清华镜像站推荐使用「方法 A」。

### 方法 A：经典单行 `.list` 格式（推荐）

写入 `/etc/apt/sources.list.d/kubernetes.list`：

```bash
# 这会覆盖该文件中的任何现有配置
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://mirrors.tuna.tsinghua.edu.cn/kubernetes/core:/stable:/v1.37/deb/ /' \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo chmod 644 /etc/apt/sources.list.d/kubernetes.list
```

deb 源内容解析：

```text
deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://mirrors.tuna.tsinghua.edu.cn/kubernetes/core:/stable:/v1.37/deb/ /
    └─────────────────────────────────────────┘ └──────────────────────────────────────────────────────────────────┘ └
                  signed-by 选项指定密钥环文件                          仓库地址（扁平仓库）                            发行版/套件
```

- `signed-by=`：指定用于校验该仓库签名的密钥环，替代已废弃的 `apt-key`。
- 仓库地址为**扁平仓库（flat repository）**，其 `Release` 与 `Packages` 文件直接位于 `deb/` 根目录下。
- 末尾的 `/` 表示发行版套件为根路径（扁平仓库的标准写法）。

### 方法 B：deb822 `.sources` 新格式（Debian 12+）

写入 `/etc/apt/sources.list.d/kubernetes.sources`：

```bash
sudo tee /etc/apt/sources.list.d/kubernetes.sources <<'EOF'
Types: deb
URIs: https://mirrors.tuna.tsinghua.edu.cn/kubernetes/core:/stable:/v1.37/deb/
Suites: /
Components:
Signed-By: /etc/apt/keyrings/kubernetes-apt-keyring.gpg
EOF

sudo chmod 644 /etc/apt/sources.list.d/kubernetes.sources
```

字段说明：

| 字段 | 值 | 说明 |
| --- | --- | --- |
| `Types` | `deb` | 二进制包仓库 |
| `URIs` | 仓库地址 | 扁平仓库根路径 |
| `Suites` | `/` | 扁平仓库的发行版套件 |
| `Components` | （空） | 扁平仓库无组件概念，保持空 |
| `Signed-By` | 密钥环路径 | 对应单行格式中的 `signed-by=` |

> 两种方法二选一即可，不要同时写入同一个仓库。

---

## 7. 步骤 5：更新软件包索引

```bash
sudo apt-get update
```

若出现 `NO_PUBKEY` 或签名校验失败，请确认步骤 3 的密钥已正确导入，且 deb 源中的 `signed-by` 路径与密钥环文件一致。

---

## 8. 步骤 6：安装 kubectl

```bash
sudo apt-get install -y kubectl
```

如需同时安装完整的 Kubernetes 节点组件，可执行：

```bash
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl   # 可选：锁定版本，防止意外升级
```

---

## 9. 步骤 7：验证安装

```bash
kubectl version --client
```

或查看更详细的版本信息：

```bash
kubectl version --client --output=yaml
```

---

## 10. 切换 / 升级次要版本

若要升级到其它次要版本，需要先更新 deb 源中的版本号，再执行更新：

```bash
# 1. 编辑源文件，将 v1.37 替换为目标版本，例如 v1.36
sudo sed -i 's/v1\.37/v1.36/g' /etc/apt/sources.list.d/kubernetes.list

# 2. 更新索引并升级
sudo apt-get update
sudo apt-get upgrade -y
```

---

## 11. 常见问题

- **`apt-key` 已废弃**：Debian 11 起 `apt-key` 被标记废弃，Debian 12 已移除。请始终使用 `signed-by` + `/etc/apt/keyrings` 方式。
- **镜像站访问被拒（反爬虫拦截）**：清华镜像站会拦截非常用软件或异常网段请求。使用 APT/curl 等标准工具通常不受影响；若被拦截，可尝试更换网络环境或切换到其它镜像站。
- **`/etc/apt/keyrings` 不存在**：Debian 12 及以上默认存在；Debian 11 需执行 `sudo mkdir -p -m 755 /etc/apt/keyrings`。
- **镜像同步延迟**：清华镜像与官方仓库存在同步延迟，若某版本尚不可用，请稍后重试或降级到已同步的版本。

---

## 参考链接

- 清华 TUNA Kubernetes 镜像：<https://mirrors.tuna.tsinghua.edu.cn/kubernetes/>
- Kubernetes 官方安装文档：<https://kubernetes.io/zh-cn/docs/tasks/tools/install-kubectl-linux/>
