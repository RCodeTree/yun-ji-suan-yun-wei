# 在 Debian 上通过官方 APT 源安装 kubectl（deb 源配置）

> 适用系统：Debian 11（bullseye）/ Debian 12（bookworm）及更新版本
> 本文档以**配置 deb 源**为核心，使用 Kubernetes 官方软件包仓库 `pkgs.k8s.io`，并按最新的 APT 规范（`signed-by` + `/etc/apt/keyrings`，废弃 `apt-key`）编写。

---

## 1. 概述

自 Kubernetes 1.24 起，官方 Debian/Ubuntu 软件包仓库由旧的 `apt.kubernetes.io` 迁移到新的 `pkgs.k8s.io`。新的仓库采用**按次要版本（minor version）分隔的路径**，例如：

```
https://pkgs.k8s.io/core:/stable:/v1.37/deb/
```

本文档中的 `v1.37` 为示例版本号，请按需替换为你的目标版本（如 `v1.36`、`v1.37` 等）。

> **版本兼容性**：kubectl 客户端与集群版本之间的差异必须在一个小版本号以内。例如 `v1.37` 的客户端可与 `v1.36`、`v1.37`、`v1.38` 的控制面通信。使用最新兼容版的 kubectl 有助于避免不可预见的问题。

同一个 deb 源同时提供 `kubectl`、`kubeadm`、`kubelet` 三个包，本文以安装 `kubectl` 为例。

---

## 2. 前置要求

- 具有 `sudo` 权限的非 root 用户或 root 用户
- 能够访问 `https://pkgs.k8s.io`
- Debian 12 及以上默认已存在 `/etc/apt/keyrings` 目录；Debian 11 需手动创建（见步骤 2）

---

## 3. 步骤 1：更新软件包索引并安装依赖

```bash
sudo apt-get update

# apt-transport-https 在较新版本中可能为虚拟包，可安全跳过
sudo apt-get install -y apt-transport-https ca-certificates curl gnupg
```

- `ca-certificates`：提供 HTTPS 证书校验能力
- `curl`：用于下载 GPG 密钥
- `gnupg`：提供 `gpg` 命令（Debian 12 中 `gpg` 通常已默认安装，`gnupg` 为完整套件）

---

## 4. 步骤 2：准备 `keyrings` 目录

最新的 APT 规范要求第三方仓库的签名密钥存放在 `/etc/apt/keyrings/` 目录，而不是旧的 `apt-key` 全局密钥环。

```bash
# Debian 12 及以上默认已存在，可跳过；Debian 11 需执行：
sudo mkdir -p -m 755 /etc/apt/keyrings
```

- 目录权限建议为 `0755`
- 密钥环文件权限建议为 `0644`（允许非特权 APT 进程读取）

---

## 5. 步骤 3：下载并导入 GPG 签名密钥

> **最新规范**：不再使用已废弃的 `apt-key add`，改用 `gpg --dearmor` 生成独立的密钥环文件，并在 deb 源中以 `signed-by` 指定。

```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.37/deb/Release.key \
  | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

sudo chmod 644 /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

说明：

- 所有次要版本共用同一个签名密钥，因此 URL 中的 `v1.37` 可以替换为任意版本，密钥相同。
- `gpg --dearmor` 把 ASCII 文本格式的公钥（`Release.key`）转换为二进制密钥环（`.gpg`）。

可选：查看导入密钥的指纹以进行核对

```bash
gpg --show-keys --with-fingerprint /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

---

## 6. 步骤 4：编写 deb 源（核心）

以下提供两种书写方式，均为**不依赖 `apt-key` 的最新规范**。官方文档推荐使用「方法 A」。

### 方法 A：经典单行 `.list` 格式（官方推荐）

写入 `/etc/apt/sources.list.d/kubernetes.list`：

```bash
# 这会覆盖该文件中的任何现有配置
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.37/deb/ /' \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo chmod 644 /etc/apt/sources.list.d/kubernetes.list
```

deb 源内容解析：

```text
deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.37/deb/ /
    └─────────────────────────────────────────┘ └──────────────────────────────────────────────┘ └
                  signed-by 选项指定密钥环文件                  仓库地址（扁平仓库）              发行版/套件
```

- `signed-by=`：指定用于校验该仓库签名的密钥环，替代已废弃的 `apt-key`。
- 仓库地址为**扁平仓库（flat repository）**，其 `Release` 与 `Packages` 文件直接位于 `deb/` 根目录下。
- 末尾的 `/` 表示发行版套件为根路径（扁平仓库的标准写法）。

### 方法 B：deb822 `.sources` 新格式（Debian 12+）

写入 `/etc/apt/sources.list.d/kubernetes.sources`：

```bash
sudo tee /etc/apt/sources.list.d/kubernetes.sources <<'EOF'
Types: deb
URIs: https://pkgs.k8s.io/core:/stable:/v1.37/deb/
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

> 两种方法二选一即可，不要同时写入同一个仓库。方法 B 为 Debian 官方正在推进的新格式，但 Kubernetes 官方文档仍以方法 A 为准。

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

> 更详细的仓库切换步骤参见官方文档「更改 Kubernetes 软件包存储库」。

---

## 11. 常见问题

- **`apt-key` 已废弃**：Debian 11 起 `apt-key` 被标记废弃，Debian 12 已移除。请始终使用 `signed-by` + `/etc/apt/keyrings` 方式。
- **`apt-transport-https` 提示为虚拟包**：自 apt 1.5 起 HTTPS 支持已内置，该包仅为过渡兼容，可安全忽略。
- **`/etc/apt/keyrings` 不存在**：Debian 12 及以上默认存在；Debian 11 需执行 `sudo mkdir -p -m 755 /etc/apt/keyrings`。
- **下载密钥失败**：确认网络可访问 `https://pkgs.k8s.io`，或考虑使用国内镜像源（如清华 TUNA 镜像）。

---

## 参考链接

- Kubernetes 官方安装文档：<https://kubernetes.io/zh-cn/docs/tasks/tools/install-kubectl-linux/>
- Kubernetes 官方软件包仓库：<https://pkgs.k8s.io/>
