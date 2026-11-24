# iloader（修复版）

> 本仓库是 [nab138/iloader](https://github.com/nab138/iloader) 的 fork，仅用于修复 Apple ID 无法登录的问题。
> 只从上游拉取代码，不向上游提交任何改动。

---

## 一、为什么登录会失败

2026 年 11 月起，Apple 的 GSA（GrandSlam）服务端收紧了校验，导致 iloader / SideStore 这类第三方工具无法登录。整个过程分三个阶段，报错会依次变化：

### 阶段 1：HTTP 503

Apple 的边缘节点会拒绝任何 `X-MMe-Client-Info` 请求头中包含 `macOS;` 的请求。

```
Failed to log in to Apple ID
```

### 阶段 2：-23782

官方 2.4.2 的临时方案是把身份字符串回滚成 macOS 10.12.6：

```
<MacBookPro13,3> <Mac OS X;10.12.6;16G29> <com.apple.AuthKit/1 (com.apple.akd/1.0)>
```

这样能绕过 503，但 Apple 的账户层随后会以版本过旧为由拒绝：

```
Auth error -23782: A Mac with macOS version 10.13 or later is required to sign in with this Apple Account.
```

注意：**这个错误发生在密码验证之前**，所以无论密码是否正确都会失败。

### 阶段 3：-22410

把身份字符串换成当前版本的 macOS 后，密码验证、2FA 都能通过，但会卡在 2FA 之后的第二次登录：

```
Failed to login again
Auth error -22410: This action could not be completed. Try again.
```

原因是 **anisette 状态残留**。anisette 状态（`adi.pb`）是在旧身份下生成的，和新的身份字符串绑定不一致，Apple 会拒绝。日志里能看到关键区别：

| 日志 | 含义 |
|---|---|
| `Loaded existing anisette state` | 复用了旧状态 → 会报 -22410 |
| `No existing anisette state found` → `Provisioning required...` | 重新生成了状态 → 登录成功 |

---

## 二、本仓库做了什么修改

只改了一个文件：`src-tauri/src/account.rs`

```rust
// 修改前（官方 2.4.2）
client_info: "<MacBookPro13,3> <Mac OS X;10.12.6;16G29> <com.apple.AuthKit/1 (com.apple.akd/1.0)>"
user_agent:  "akd/1.0 CFNetwork/811.5.4 Darwin/16.7.0"

// 修改后
client_info: "<MacBookAir10,1> <Mac OS X;27.3;26C5035r> <com.apple.AuthKit/1 (com.apple.akd/1.0)>"
user_agent:  "akd/1.0 CFNetwork/3896.300.31 Darwin/27.3.0"
```

两个要点：

1. 必须写 `Mac OS X;`，**不能**写 `macOS;`，否则直接 503。
2. 版本号必须是当前版本（10.13 以上），否则 -23782 / -22410。

这套身份对应上游已验证的方案：[#1203](https://github.com/nab138/iloader/issues/1203)、[#1205](https://github.com/nab138/iloader/pull/1205)。

---

## 三、使用方法

### 1. 下载

到本仓库的 **Actions** 页面，打开最新一次成功的构建，在页面底部下载 `iloader-fixed-windows-x64` 压缩包，解压后得到安装包。

也可以自己编译（见第四节）。

### 2. 安装前先卸载旧版

如果之前装过官方 iloader，建议先卸载，避免残留配置干扰。

### 3. 关键一步：重置 Anisette 状态

**这一步不能跳过。** 如果跳过，会在 2FA 之后收到 `-22410`。

打开 iloader → 左下角 **Settings（设置）** → 找到 **Reset Anisette State** 按钮 → 点击确认。

> 提示文字会说明：重置后需要重新输入 2FA 验证码。这是正常的。

### 4. 确认 Anisette 服务器

同样在 **Settings** 页面，顶部有 **Anisette Server** 下拉框。推荐选择：

```
mac-ani.jkcoxson.com
```

如果登录失败，可以换列表里的其他服务器逐个尝试。选择后**重启 iloader** 再登录。

### 5. 登录

回到主界面输入 Apple ID 和密码。如果账号开了双重认证，会要求输入受信任设备上显示的 6 位验证码。

登录成功后即可正常安装应用。

---

## 四、常见问题

### 还是报 -22410

说明 anisette 状态没有真正重置。检查方法：

1. 完全退出 iloader（任务管理器里确认进程已结束，不能只是关窗口）
2. 重新打开，进 Settings 再点一次 **Reset Anisette State**
3. 再登录

日志位置（Windows）：`%APPDATA%\me.nabdev.iloader\logs\`，可以搜索 `anisette` 确认是否出现 `No existing anisette state found`。

### 报 -45061（not provisioned）

anisette 服务器那边的问题，换一个服务器再试。

### 报 -23782

说明身份字符串没生效，可能装成了官方版本。确认下载的是本仓库的构建产物。

### 不要做的事

- **不要反复重试登录。** 每次失败都是一次真实请求，容易触发 Apple 风控。
- **不要关闭双重认证或「高级数据保护」。** 失败发生在密码验证之前，和这些设置无关。
- **不要反复卸载重装。** 问题在 anisette 状态，不在安装本身。

---

## 五、自己编译

需要 Rust、Node.js 和 [Bun](https://bun.sh)。

```bash
git clone https://github.com/NANYUA/iloader.git
cd iloader
git checkout fix/login-client-info
bun i --frozen-lockfile
bunx tauri build --config src-tauri/ci.conf.json
```

产物在 `src-tauri/target/release/bundle/`。

用 `ci.conf.json` 是为了跳过 updater 签名密钥校验，本仓库没有上游的签名私钥。

也可以通过 GitHub Actions 云端编译：进入 **Actions** → **Build fork (fixed login)** → **Run workflow**，可选是否同时构建 Linux / macOS。

---

## 六、说明

- 本构建**未做代码签名**，Windows SmartScreen 可能提示「未知发布者」，选择「仍要运行」即可。
- 本构建**关闭了自动更新**（因为没有上游签名密钥）。等官方发布正式修复后，建议换回官方版本。
- 身份字符串是硬编码的。Apple 以后若再次提高版本要求，需要同步修改 `account.rs` 并重新构建，**且必须再次重置 anisette 状态**。

---

## 附：SideStore 上的同类问题处理（原文参考）

以下内容来自 [SideStore#1745 的评论](https://github.com/SideStore/SideStore/issues/1745#issuecomment-7065875033)，是同一问题的另一种解法。**SideStore 的界面和 iloader 不同**，它有 Client Info、Remote Server Sync 等设置项，而 iloader 没有这些选项（身份字符串已在代码里写死）。仅作背景参考。

> **Setup:** LiveContainer 3.8.14 nightly with built-in SideStore 0.7.0-20261107 nightly, iOS 27.0.1, free Apple ID.
>
> **What I went through:**
> 1. Refresh failed with "Your session has expired". Signing in again returned **HTTP 503**.
> 2. Changed `macOS;` to `Mac OS X;` in Client Info. The 503 was gone, but got **-22410**.
> 3. **Reset adi.pb** with On-Device Anisette on then failed with **"HTTP error (401): Apple PTM/TK endpoint returned HTTP 401"**.
>
> **What finally worked:**
> 1. Settings → User Customizations → turn **On-Device Anisette OFF**, restart SideStore.
> 2. Settings → Anisette Servers → select `https://mac-ani.jkcoxson.com`.
> 3. User Customizations → Anisette Client Configuration → Remote Server Sync → tap **Fetch**, then **Save as Overrides**.
> 4. In **Client Info**, change only `10.12.6;16G29` to `26.6;25F84`:
>    ```
>    <MacBookPro13,3> <Mac OS X;26.6;25F84> <com.apple.AuthKit/1 (com.apple.akd/1.0)>
>    ```
>    Keep the fetched User Agent, `akd/1.0 CFNetwork/811.5.4 Darwin/16.7.0`, and tap **Save Overrides**.
> 5. Force-quit SideStore, reopen it and sign in.
>
> If you get **-45061** or **-22410**: User Customizations → Reset adi.pb → Reset & Sign Out with header customizations kept. Restart and sign in again.
>
> **Why each step is needed:**
> - Since 2026-11-20 Apple's GSA edge returns 503 for any `X-MMe-Client-Info` containing `macOS;`. It must say `Mac OS X;`.
> - The old `Mac OS X;10.12.6` identity gets past the edge, but Apple then rejects it with -22410 or -23782 (nab138/iloader#1205).
> - On-Device Anisette couldn't get a new identity registered (401), so a remote server was needed.
>
> **Tips:**
> - Don't tap **Reset to Defaults** in Anisette Client Configuration. It puts `macOS;` back.
> - Don't retry over and over. Each failed attempt is another request to Apple.

---

## 许可

继承上游 [LICENSE](LICENSE) 与 [LICENSE-BRANDING](LICENSE-BRANDING)。本项目与 Apple 无关联。
