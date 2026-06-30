# GitHub Actions 自动构建与签名指南

本指南说明如何在你 Fork 的 Asspp 仓库中，使用 GitHub Actions 自动构建、签名并发布 iOS 应用。

完成后你可以：

1. **自动跟进上游更新**：工作流会定期检测上游仓库的新提交并自动构建。
2. **OTA 安装**：在 iPhone 上通过网页链接直接安装，无需电脑。
3. **自动签名**：使用你自己的 Apple 开发者证书完成签名。

---

## 0. 选择构建方案

| 方案 | 工作流 | 是否需要证书 | 能否 OTA 安装 | 适用场景 |
|------|--------|-------------|--------------|----------|
| **签名 + OTA（推荐）** | `Upstream Signed iOS Build` | 需要 | 可以 | 个人长期使用，手机直接安装更新 |
| **无签名构建** | `Build App` | 不需要 | 不可以 | 仅验证能否编译，需自行签名 |
| **正式发布** | `Release` | 不需要 | 不可以 | 打 tag 时发布未签名 IPA/ZIP |

本文重点介绍**方案一（签名 + OTA）**。如果你没有 Mac，请直接看 **[没有 Mac 怎么办？](#没有-mac-怎么办)**。

---

## 没有 Mac 怎么办？

**好消息：编译不需要你的 Mac。** GitHub Actions 会在云端 macOS 机器上完成构建，你本地只需要浏览器 + 任意电脑（Windows / Linux 均可）。

根据你的情况，选一条路径：

| 你的情况 | 推荐方案 | 难度 |
|---------|---------|------|
| 不想付费开发者账号 | 直接下载 [官方 Releases](https://github.com/Lakr233/Asspp/releases) 的 IPA，用巨魔/ESign 等安装 | ⭐ 最简单 |
| 有付费开发者账号，想 OTA 自动安装 | 浏览器创建证书 + Linux/Windows 生成 Secrets + GitHub 自动构建 | ⭐⭐⭐ |
| 没有开发者账号，但想自己构建 | 触发 **Build App** 工作流，下载未签名 IPA 后自行签名 | ⭐⭐ |

### 路径 A：最简单（无需 GitHub Actions）

1. iPhone Safari 打开 [Releases](https://github.com/Lakr233/Asspp/releases) 下载 `.ipa`
2. 用 **TrollStore（巨魔）**、**ESign**、**Sideloadly（Windows）** 等工具安装

> 若设备支持巨魔，安装后无需反复签名，最省心。

#### 路径 A+：没有巨魔、没有开发者账号（大多数人适用）

这是**最现实的方案**：用**免费 Apple ID** 签名，应用 **7 天后过期**，需重新签名。

**你需要：** Windows 电脑 + 数据线 + 一个**备用 Apple ID**（不要用主力账号）

**第 1 步：下载 IPA**

iPhone 或电脑浏览器打开：

https://github.com/Lakr233/Asspp/releases/latest

下载 `Asspp.ipa`（当前最新版约 15 MB）

**第 2 步：选一种 Windows 签名工具**

| 工具 | 下载 | 特点 |
|------|------|------|
| **Sideloadly** | [sideloadly.io](https://sideloadly.io) | 英文界面，步骤简单，推荐 |
| **爱思助手** | [i4.cn](https://www.i4.cn) | 中文界面，国内用户熟悉 |
| **AltStore** | [altstore.io](https://altstore.io) | 需装 AltServer，最多 3 个 App，可 WiFi 续签 |

**第 3 步：Sideloadly 安装步骤（推荐）**

1. Windows 安装 [iTunes](https://www.apple.com/itunes/) 或 [Apple 设备支持驱动](https://support.apple.com/downloads)（用于识别 iPhone）
2. iPhone 数据线连接电脑 → 手机上点「信任此电脑」
3. 打开 Sideloadly → 拖入 `Asspp.ipa`
4. 输入**备用 Apple ID** 和密码（不要用主力账号）
5. 若开启双重认证，去 [appleid.apple.com](https://appleid.apple.com) 生成**专用密码**
6. 点 Start → 等待安装完成
7. iPhone：**设置 → 通用 → VPN 与设备管理** → 信任该开发者

**第 4 步：7 天后续签**

免费签名有效期 **7 天**。到期后 App 打不开，重新用 Sideloadly 装一次即可（不用删，直接覆盖安装）。

> **GitHub Actions 对你不是必须的** — 官方 Releases 的 IPA 和云端构建的未签名 IPA 本质相同，直接下载更省事。

### 路径 B：无 Mac + 有开发者账号 → OTA 自动安装

你需要在 **浏览器** 完成 Apple Developer 操作，在 **Windows/Linux** 生成证书文件。

#### B1. 浏览器操作（developer.apple.com）

1. 创建 App ID：`wiki.qaq.Asspp`
2. 注册设备 UDID（iPhone Safari 打开 [udid.tech](https://udid.tech) 获取，无需 Mac）
3. 创建 **Apple Distribution** 证书时，需要上传 CSR 文件（见 B2）
4. 下载 `.cer` 证书文件
5. 创建 **Ad Hoc** 描述文件，勾选你的设备，下载 `.mobileprovision`

#### B2. 在 Windows / Linux 生成 CSR 和 .p12

**Linux / macOS / Git Bash：**

```bash
# 1. 生成私钥和 CSR
openssl genrsa -out private.key 2048
openssl req -new -key private.key -out request.csr \
  -subj "/emailAddress=你的邮箱/CN=你的名字/C=CN"

# 2. 把 request.csr 上传到 Apple Developer 网站，下载 distribution.cer

# 3. 转成 .p12（把密码换成你想要的）
openssl x509 -in distribution.cer -inform DER -out certificate.pem -outform PEM
openssl pkcs12 -export -out certificate.p12 \
  -inkey private.key -in certificate.pem \
  -password pass:你的p12密码
```

**Windows PowerShell** 需先安装 [OpenSSL](https://slproweb.com/products/Win32OpenSSL.html)，命令同上。

#### B3. 上传 GitHub Secrets（跨平台脚本）

```bash
git clone https://github.com/<你的用户名>/Asspp.git
cd Asspp

# 此脚本支持 Linux / Windows Git Bash，不依赖 Mac
./Resources/Scripts/generate.github.action.inputs.cross.sh \
  --p12 ./certificate.p12 \
  --p12-password '你的p12密码' \
  --mobileprovision ./Asspp_AdHoc.mobileprovision

export GITHUB_REPOSITORY="<你的用户名>/Asspp"
/tmp/asspp-gha-inputs-xxxx/apply-with-gh.sh
```

没有 `gh` CLI 也可以：把 `secrets/*.txt` 里的内容手动粘贴到 GitHub → Settings → Secrets。

然后按本文第 3–6 步启用 Actions、触发 **Upstream Signed iOS Build**、Safari 打开 OTA 链接安装。

### 路径 C：无 Mac + 无开发者账号 → 云端构建未签名 IPA

1. Fork 仓库，在 **Actions** 页面启用工作流
2. 打开 **Actions** → **Build App** → **Run workflow**
3. 构建完成后，在 Artifacts 下载 `Asspp.ipa`（未签名）
4. 用以下工具之一签名安装：
   - **TrollStore**：无需签名，直接安装（需设备支持）
   - **ESign**：在 iPhone 上直接签名安装
   - **Sideloadly**（Windows）：用免费 Apple ID 签名，7 天有效
   - **AltStore** + Windows 版 AltServer

---

## 1. 前置条件

### 账号与设备

- **付费 Apple Developer 账号**（个人或公司均可）
- **GitHub 账号**，且已 Fork [Asspp 仓库](https://github.com/Lakr233/Asspp)
- **iPhone UDID**：Ad Hoc 描述文件必须包含你要安装设备的 UDID

### 获取 iPhone UDID

任选一种方式：

- 用数据线连接 Mac，打开 **Finder** → 选中设备 → 点击序列号区域切换显示 **UDID**，右键复制
- 访问 [https://udid.tech](https://udid.tech) 等网页获取（需在 Safari 中打开）

### 签名文件

你需要准备两个文件：

| 文件 | 说明 |
|------|------|
| `certificate.p12` | 从「钥匙串访问」导出的分发证书 |
| `Asspp_AdHoc.mobileprovision` | Ad Hoc 描述文件，包含你的设备 UDID |

---

## 2. 在 Apple Developer 创建签名资源

### 2.1 创建 App ID

1. 登录 [Apple Developer](https://developer.apple.com/account)
2. 进入 **Certificates, Identifiers & Profiles** → **Identifiers**
3. 点击 **+** → 选择 **App IDs** → **App**
4. 填写：
   - **Description**：`Asspp`（任意名称）
   - **Bundle ID**：选 **Explicit**，填写 `wiki.qaq.Asspp`（或与项目一致的其他 Bundle ID）
5. 保存

> 若你打算使用自定义 Bundle ID，后续在 GitHub Variables 中设置 `IOS_BUNDLE_ID` 即可，但必须与描述文件中的 App ID 一致。

### 2.2 注册设备 UDID

1. 进入 **Devices** → 点击 **+**
2. 输入设备名称和 UDID
3. 保存

### 2.3 创建分发证书

1. 进入 **Certificates** → 点击 **+**
2. 选择 **Apple Distribution**（用于 Ad Hoc 分发）
3. 按提示在 Mac 上生成 CSR 并上传
4. 下载 `.cer` 文件，双击安装到钥匙串

### 2.4 导出 .p12 证书

1. 打开 Mac **钥匙串访问**
2. 在「我的证书」中找到 **Apple Distribution: …**
3. 右键 → **导出** → 格式选 **个人信息交换 (.p12)**
4. 设置导出密码（记住它，后面要用）

### 2.5 创建 Ad Hoc 描述文件

1. 进入 **Profiles** → 点击 **+**
2. 选择 **Ad Hoc**
3. 选择刚创建的 App ID
4. 选择分发证书
5. **勾选你要安装的设备**
6. 命名（如 `Asspp AdHoc`）并生成
7. 下载 `.mobileprovision` 文件

---

## 3. 配置 Fork 仓库

你的 Fork 地址示例：`https://github.com/<你的用户名>/Asspp`

### 3.1 启用 GitHub Actions

Fork 后 Actions **默认可能是关闭的**，需要手动开启：

1. 打开 Fork 仓库 → **Actions** 标签页
2. 若提示 *Workflows aren't being run on this forked repository*，点击 **I understand my workflows, go ahead and enable them**

### 3.2 设置 Actions 权限

1. **Settings** → **Actions** → **General**
2. **Workflow permissions** 选择 **Read and write permissions**
3. 点击 **Save**

### 3.3 启用 GitHub Pages

1. **Settings** → **Pages**
2. **Build and deployment** → **Source** 选 **GitHub Actions**

---

## 4. 配置 Secrets 与 Variables

GitHub 需要你的签名文件才能在云端完成签名。

### 方式 A：使用项目脚本（推荐，在 Mac 上执行）

```bash
# 克隆你的 Fork
git clone https://github.com/<你的用户名>/Asspp.git
cd Asspp

# 生成 Secrets / Variables
./Resources/Scripts/generate.github.action.inputs.sh \
  --p12 /path/to/certificate.p12 \
  --p12-password '你的p12密码' \
  --mobileprovision /path/to/Asspp_AdHoc.mobileprovision
```

脚本会输出一个目录（如 `/tmp/asspp-gha-inputs-...`），其中包含：

- `secrets/*.txt` — 需要上传到 GitHub Secrets 的内容
- `variables/*.txt` — 需要上传到 GitHub Variables 的内容
- `apply-with-gh.sh` — 一键上传脚本（需安装 `gh` CLI）

**一键上传（需先登录 gh）：**

```bash
export GITHUB_REPOSITORY="<你的用户名>/Asspp"
/tmp/asspp-gha-inputs-xxxx/apply-with-gh.sh
```

### 方式 B：在 GitHub 网页手动填写

进入 **Settings** → **Secrets and variables** → **Actions**

#### Secrets（Repository secrets）

| 名称 | 值 |
|------|-----|
| `IOS_CERT_P12_BASE64` | `.p12` 文件的 Base64 编码 |
| `IOS_CERT_PASSWORD` | 导出 `.p12` 时设置的密码 |
| `IOS_PROVISIONING_PROFILE_BASE64` | `.mobileprovision` 文件的 Base64 编码 |

macOS 生成 Base64：

```bash
base64 -i certificate.p12 | pbcopy
base64 -i Asspp_AdHoc.mobileprovision | pbcopy
```

#### Variables（Repository variables）

| 名称 | 值 | 说明 |
|------|-----|------|
| `IOS_BUNDLE_ID` | `wiki.qaq.Asspp` | 必须与描述文件中的 App ID 一致 |
| `IOS_EXPORT_METHOD` | `ad-hoc` | 一般保持 `ad-hoc` |
| `IOS_OTA_BASE_URL` | _(留空)_ | 可选，自定义 OTA 域名时填写 |

---

## 5. 触发构建

### 手动触发（首次推荐）

1. 打开 Fork 仓库 → **Actions**
2. 左侧选择 **Upstream Signed iOS Build**
3. 点击 **Run workflow**
4. 参数保持默认即可：
   - `source_kind`: `upstream`（从上游 Lakr233/Asspp 拉取最新代码）
   - `force_build`: `true`
5. 点击绿色 **Run workflow** 按钮

### 自动触发

配置完成后，该工作流会：

- **每 30 分钟**检测上游 `main` 分支是否有新提交
- 有新提交时自动构建并发布

---

## 6. 安装应用

构建成功后（约 5–10 分钟）：

### 方式一：GitHub Pages OTA 安装（推荐）

在 iPhone 的 **Safari** 中打开：

```
https://<你的用户名>.github.io/Asspp/ios/latest/install.html
```

例如 Fork 为 `KAIMA04/Asspp` 时：

```
https://kaima04.github.io/Asspp/ios/latest/install.html
```

点击页面上的 **Install** 链接即可安装。

### 方式二：从 Releases 下载

1. 打开仓库 **Releases** 页面
2. 找到类似 `upstream-main-xxxxxxx` 的 Release
3. 下载 `.ipa` 或使用 Release 附带的安装页面

### 首次安装后信任证书

若提示「无法验证 App」：

**设置** → **通用** → **VPN 与设备管理** → 信任你的开发者证书

---

## 7. 常见问题

### Actions 标签页看不到工作流

Fork 后需先在 Actions 页面点击启用。若仍无工作流，确认 `.github/workflows/` 目录已同步到你的 Fork。

### 构建失败：Missing required secrets

未配置 `IOS_CERT_P12_BASE64`、`IOS_CERT_PASSWORD` 或 `IOS_PROVISIONING_PROFILE_BASE64`，请按第 4 步补全。

### 构建失败：Bundle ID 不匹配

`IOS_BUNDLE_ID` 变量值必须与描述文件中的 App ID 完全一致。

### 安装一直转圈 / 无法安装

- 确认 iPhone UDID 已加入 Ad Hoc 描述文件
- 若更新了描述文件，需重新下载 `.mobileprovision` 并更新 GitHub Secret
- 重新触发一次构建

### 证书过期

在 Apple Developer 重新创建证书和描述文件，更新 GitHub Secrets 后重新构建。

### 无签名构建（不需要证书）

若只想验证编译是否通过，推送代码到 `main` 分支即可触发 **Build App** 工作流，产物为未签名的 `Asspp.ipa`，需自行用 AltStore、SideStore 等工具签名安装。

---

## 8. 工作流说明

| 工作流文件 | 名称 | 触发条件 |
|-----------|------|---------|
| `upstream-signed-ios.yml` | Upstream Signed iOS Build | 定时 + 手动；**仅在 Fork 仓库运行** |
| `build.yml` | Build App | push/PR 到 `main` |
| `release.yml` | Release | 推送任意 tag |

---

[返回中文 README](./README.md) | [English Guide](../../../Resources/Document/FORK_AUTOBUILD_GUIDE.md)
