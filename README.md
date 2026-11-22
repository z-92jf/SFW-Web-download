# SFW-Web-download

SERVICES FOR Windows 的**第三方下载站**，托管在 Cloudflare Pages 上。

---

## 一、它是干嘛的

官网 `sfw.us.ci` 托管在 GitHub Pages，而安装包放在 GitHub Releases。

问题在于 **GitHub 在国内访问很不稳定** —— 有时候能打开页面，但点下载就一直转圈，或者直接断在 90%。

所以这里另开一个下载点：

```
官网的下载弹窗
   ├── GitHub Release  →  github.com（国内时快时慢）
   └── Cloudflare      →  dl.sfw.us.ci（就是本站，国内稳定）
```

安装包本身没有变，只是**同一个文件多了一个分发入口**。

---

## 二、目录里有什么

```
SFW-Web-download/
├── index.html                       下载页（读 versions.json 渲染）
├── 404.html                         ⚠ 必须有，见第五节
├── versions.json                    ★ 版本清单，发新版改这里
├── logo.png                         站点图标（从官网复制过来的）
├── favicon.ico                      浏览器标签页图标
│
├── Setup.SERVICES.FOR.Windows.V1.0.0.exe        ┐
├── v1.0.0.zip                                    │
├── Setup.SERVICES.FOR.Windows.V1.0.1.Beta.exe    │
├── 1.0.1.Beta.zip                                │
├── Setup.SERVICES.FOR.Windows.V1.1.0-Win64.exe   │  6 个版本
├── 1.1.0-Win64.zip                               │  × 2 个文件
├── Setup-SERVICES-FOR-Windows-V1.1.1.exe         │  = 12 个
├── 1.1.1.zip                                     │  共 78 MB
├── Setup.SERVICES.FOR.Windows.1.1.2.exe          │
├── 1.1.2.zip                                     │
├── Setup.SERVICES.FOR.Windows.1.1.3.exe          │
├── SERVICES.FOR.Windows.1.1.3.zip                ┘
│
├── Windows6.1-KB2999226-x64.msu      ┐  Windows 7 / 8.1 的依赖补丁
├── Windows6.1-KB2533552-x64.msu      │  3 个文件，共 11 MB
└── Windows8.1-KB2999226-x64.msu      ┘
```

> 安装包已经全部放进来了。如果你以后重新 clone 仓库，这些文件会一起下来（约 90 MB）。
> **文件名一个字都不能改** —— `versions.json` 里写的就是它们，改名即 404。

> 三个 `.msu` 是 Windows 7 / 8.1 的 Universal C Runtime 补丁。Win10 及以上系统自带这个运行库，
> 不需要装；Win7 SP1 / 8.1 缺了会报「计算机中丢失 api-ms-win-crt-runtime-l1-1-0.dll」。
>
> ⚠ 换补丁文件时 **三处要一起改**：`versions.json` 的 `dependencies` 段、`index.html` 里的下载区和 MD5、
> 官网 `src/data/site.ts` 的 `runtimeDeps`。任一处漏改，用户下载到的就是旧文件或对不上号的 MD5。
>
> ⚠ Cloudflare Pages 单文件上限 25 MiB。Windows 8.1 的 **KB2919355 有 724 MB，放不进来**，
> 所以 8.1 用的是 KB2999226（约 1 MB），不是那个更新汇总包。

---

## 三、第一次部署：新建 GitHub 仓库

### 3.1 为什么要单独一个仓库

因为 Cloudflare Pages 是**连 Git 仓库自动构建**的，一个仓库对应一个站点。
官网那个 `SFW-Web` 仓库已经连了 GitHub Pages，再拿它连 Cloudflare 会互相干扰。

而且二进制文件单独一个仓库，以后想让官网仓库保持轻量也很方便。

### 3.2 建仓库

1. 打开 <https://github.com/new>
2. **Repository name** 填 `SFW-Web-download`
3. 选 **Public**（Pages 免费额度对公开仓库更宽松，也不涉及隐私）
4. **不要**勾选 "Add a README file"（我们本地已经有文件了，勾了还要处理冲突）
5. 点 **Create repository**

### 3.3 把文件传上去

**方法 A：用 GitHub Desktop（你之前用的这个，推荐）**

1. 在 GitHub Desktop 里 `File → Add local repository`
2. 选 `D:\我的\SERVICES FOR Windows\SFW-Web-download`
3. 它会提示 "这不是一个 Git 仓库"，点 **create a repository**
4. 填名字 `SFW-Web-download`，点 **Create repository**
5. 左下角填一句提交信息（比如 `初始版本`），点 **Commit to main**
6. 右上角点 **Publish repository**，**取消勾选** "Keep this code private"，然后 Publish

**方法 B：网页上传**

1. 建好的仓库页面里点 **uploading an existing file**
2. 把整个目录的文件拖进去（**注意别拖文件夹，只拖文件**）
3. 底下填提交信息，点 **Commit changes**

### 3.4 一个必须注意的点

⚠️ **`.exe` 文件可能会被 Git 当成二进制**，这没问题，能正常提交。
但如果文件超过 **100 MB**，GitHub 会拒绝推送 —— 现在的安装包 6 MB 左右，远没到。

---

## 四、连 Cloudflare Pages

### 4.1 创建项目

1. 登录 Cloudflare 控制台，左侧选 **「计算」（Compute）** → **「Workers 和 Pages」**
2. 点 **「创建应用程序」**
3. 选 **「Pages」** 标签页 → **「连接到 Git」**
4. 第一次用会要求授权 GitHub，点 **「连接 GitHub」**，在弹出的窗口里选
   **Only select repositories** → 只勾 `SFW-Web-download` → **Install & Authorize**
   （只给这一个仓库的权限，别的仓库它看不到，更安全）
5. 回到 Cloudflare，选中刚授权的仓库，点 **「开始设置」**

### 4.2 构建配置

这一步最容易填错。**这是一个纯静态站，不需要任何构建**：

| 项目 | 填什么 |
|---|---|
| 项目名称 | `sfw-download`（会变成 `sfw-download.pages.dev`，随便起） |
| 生产分支 | `main` |
| 框架预设 | **无 / None** |
| 构建命令 | **留空** |
| 构建输出目录 | **留空**（或者填 `/`） |
| 根目录 | 留空 |

点 **「保存并部署」**。

> 构建命令留空是**故意的**：这里没有 npm、没有打包，GitHub 上的文件是什么，
> 部署出来就是什么。填了构建命令反而会失败。

等一两分钟，会给你一个 `xxx.pages.dev` 的地址。先打开试试：

```
https://sfw-download.pages.dev/
https://sfw-download.pages.dev/Setup.SERVICES.FOR.Windows.1.1.3.exe
```

第二条应该**直接开始下载**。如果不是，先看第五节。

### 4.3 绑自定义域名（`dl.sfw.us.ci`）

⚠️ **顺序很重要：先在 Pages 里加域名，再去 DNS 加记录。**

1. 进入这个 Pages 项目 → **「自定义域」**（Custom domains）→ **「设置自定义域」**
2. 填 `dl.sfw.us.ci`，点 **「继续」**
3. Cloudflare 会提示「因为 `sfw.us.ci` 也在这个账号下，可以自动添加 DNS 记录」，
   点 **「激活域」** —— 它会自己加一条 CNAME，不用手动去 DNS 页面添加
4. 等状态从「初始化中」变成 **「有效」**（一般 1~3 分钟）

**为什么必须先加域名再加记录**：如果反过来，Node 去找不到目标，
Cloudflare 会以为你要手动托管，容易配出一条重复或冲突的记录。

**如果你的域名不在 Cloudflare 账号下**（比如在别家注册、只是 NS 指向 Cloudflare，
那也算在账号下，没关系），那就手动去 DNS 加：

| 类型 | 名称 | 内容 | 代理状态 |
|---|---|---|---|
| CNAME | `dl` | `sfw-download.pages.dev` | **DNS only（灰色云）** |

### 4.4 验证

```bash
# 1. 域名解析
nslookup dl.sfw.us.ci

# 2. 首页
curl -I https://dl.sfw.us.ci/

# 3. 安装包能否下载（应该返回 200 和正确的 content-length）
curl -I https://dl.sfw.us.ci/Setup.SERVICES.FOR.Windows.1.1.3.exe

# 4. 不存在的文件应该返回 404，而不是首页
curl -I https://dl.sfw.us.ci/不存在的文件.exe
```

第 3 步的 `content-length` 应该是 **6360513**（1.1.3 的安装包大小），
和 GitHub 上一致 —— 对得上就说明文件没传坏。

---

## 五、为什么必须有 404.html

这一条很容易被忽略，但**没有它后果很严重**。

Cloudflare Pages 对「找不到的路径」有一个默认行为：如果站点里没有 `404.html`，
它会**返回 `index.html`**（很多单页应用靠这个做路由）。

对普通网站这没什么，但对下载站是灾难：

```
用户点了 dl.sfw.us.ci/Setup.SERVICES.FOR.Windows.1.1.4.exe
                                    ↑ 这个版本还没传

预期：404，用户知道文件不存在
实际：返回 index.html（一个 HTML 页面），浏览器存成 .exe
     → 用户双击，提示「不是有效的 Win32 程序」
```

所以 `404.html` **必须存在**，它会让未匹配的路径老老实实返回 404 状态码。

---

## 六、发新版怎么更新

每次发新版，按这个顺序走（**顺序错了用户会下到旧文件**）：

1. **先传文件**：把新的 exe / zip 复制进这个目录（**旧版的文件不要删**）
2. **再改 `versions.json`**：在 `versions` 数组最前面加一条（新的在最前）
3. 提交并推送，Cloudflare 会自动重新部署（约 1 分钟）
4. **去官网改 `src/data/site.ts`**：
   - `links.download` 换成新文件名
   - `releases` 数组最前面加一个新版本，**两个文件都要带 `md5`**
     （弹窗底部的「文件MD5值」就是读这里）
5. 算 MD5 的办法：`Get-FileHash .\文件名 -Algorithm MD5`

`versions.json` 的格式（**`md5` 字段不能省，页面会显示它**）：

```json
{
	"updated": "2026-11-20",
	"site": "https://sfw.us.ci",
	"versions": [
		{
			"version": "1.1.4",
			"date": "2026-11-XX",
			"stage": "正式版",
			"note": "一句话说明这版改了什么",
			"files": [
				{
					"label": "安装版",
					"file": "Setup.SERVICES.FOR.Windows.1.1.4.exe",
					"size": "6.1 MB",
					"md5": "填这个文件真实的 32 位 MD5"
				},
				{
					"label": "免安装包",
					"file": "SERVICES.FOR.Windows.1.1.4.zip",
					"size": "6.6 MB",
					"md5": "填这个文件真实的 32 位 MD5"
				}
			]
		}
	]
}
```

> 页面是纯前端读这个 JSON 渲染的，所以**改完必须提交**，不提交页面上看不到。

---

## 七、限制和要注意的

### 7.1 Cloudflare Pages 免费额度

| 项目 | 限制 | 现在 | 够不够 |
|---|---|---|---|
| 单文件大小 | **25 MiB** | 最大 8.74 MB | ✓ 够 |
| 文件总数 | **20,000** | 12 个 | ✓ 完全够 |
| 每月构建次数 | 500 | 每次发版 1 次 | ✓ 够 |
| 流量 / 带宽 | **不限** | — | 这是它比 GitHub Pages 强的地方 |
| **站点总大小** | **没有硬性限制** | 78 MB | ✓ |

**结论：放全部 6 个版本、12 个文件完全没问题。**

### 7.1.1 和计数器 Worker 会不会抢额度？

**不会。** 官方文档原文：

> On both free and paid plans, requests to static assets are **free and unlimited**.
> A request is considered static when it does not invoke Functions.

翻译成实际影响：

| | 计数器 Worker（times.sfw.us.ci） | 本站（纯静态） |
|---|---|---|
| 请求计数 | 10 万/天 | **不计量，无限** |
| 占谁的额度 | Workers 自己的 | 谁的都不占 |
| 带宽 | — | 无限 |

**只有 Pages Functions 才共享 Workers 那 10 万/天**，而本站没有 `functions/` 目录，
所有请求都是直接返回文件，所以**一个请求都不计**。

真正共享的只有这些（都和访问量无关）：

| 资源 | 免费额度 | 现状 |
|---|---|---|
| Workers 脚本数 | 100 个/账号 | 2 个 |
| Pages 项目数 | 100 个/账号 | 1 个 |
| Pages 构建次数 | **500 次/月**（账号内共享） | 每次 push 用 1 次 |
| D1 存储 | 5 GB | 只有计数器在用 |

⚠️ **以后如果给本站也加统计功能**，那部分 D1 写入会和计数器**共用** 10 万行/天。
现在没有，不用管。

⚠️ 构建的小技巧：本站 78 MB，每次构建都要重传全部文件。
如果某次提交**不需要重新部署**（比如只改 README），在 commit 信息里写
`[CF-Pages-Skip]`，Cloudflare 会跳过这次构建，省一次额度。

真正会先撞上的不是 Pages，而是 **GitHub 仓库**（见 7.2）和**单文件 25 MiB**。

⚠️ **25 MiB 是硬限制**，单个文件超了部署直接失败（不是整个站失败，是那个文件传不上去）。
以后安装包如果涨到 25MB 以上，就得换方案（比如 R2 对象存储，免费 10 GB）。

### 7.2 Git 仓库会越来越大

Git 有个特性：**删掉文件，历史里依然留着**。所以仓库只会越来越大，不会因为删除而变小。

现在的账：

```
6 个版本 × 约 13 MB = 78 MB
```

按每版 +13 MB 估算：

| 版本数 | 仓库约 | GitHub 状态 |
|---|---|---|
| 6（现在） | 78 MB | 很轻 |
| 20 | 260 MB | 正常 |
| 50 | 650 MB | 接近建议上限（1 GB） |
| 80+ | 1 GB+ | 会变慢，建议迁移 |

**所以现在可以放心全放**，但要有心理准备：**等版本攒到 50 个左右要清理一次**。

到时候的办法（现在不用做）：
1. 把旧版移到 **Cloudflare R2**（免费 10 GB，且下载不计流量费）
2. 或者只留最近 10 个版本，更老的从目录删掉 + 只在 GitHub Releases 保留

⚠️ 注意「从目录删掉」也不能回收已占的仓库空间 —— 要真正减重得重写历史（`git filter-repo`），
比较麻烦。所以**如果确定要控制体积，宁可不放，而不是放进去再删**。

### 7.3 下载统计不经过这里

官网的下载次数统计（`times.sfw.us.ci`）是在**用户点下载通道时**由浏览器上报的，
跟文件从哪条通道下、下没下成功都没关系。

所以：走 Cloudflare 通道的下载**也会被正常计入**，不用额外配置。

### 7.4 所有版本都在这里

镜像站上放了**全部 6 个版本 × 2 个文件**，所以官网下载弹窗里的 Cloudflare 通道
**对历史版本也一路可用**，和 GitHub 那条通道完全对等。

这也意味着：**每发一版，除了改 `versions.json`，还要把新文件复制进来**（见第六节）。

---

## 八、和官网的联动

官网这边有两个地方跟本站有关，都在 `SFW-Web/src/data/site.ts`：

```ts
// ① 镜像域名。改成空字符串的话，弹窗里的 Cloudflare 通道会自动禁用
export const downloadMirror = 'https://dl.sfw.us.ci';

// ② 每个版本的每个文件都带一个 md5，弹窗底部的「文件MD5值」读的是这里
export const releases = [
	{
		version: '1.1.3',
		assets: [
			{ label: '安装版', file: '...exe', size: '6.07 MB', md5: '1e2fb65a...' },
			{ label: '免安装包', file: '...zip', size: '6.6 MB', md5: 'd5324554...' },
		],
	},
	// …其余版本
];
```

**弹窗里的 MD5 是按版本切换的**：点 1.1.3 显示 1.1.3 那两个文件的值，
点 1.1.2 就换成 1.1.2 的。所以**每个版本的每个文件都要填**，
填错了用户照着核对永远对不上。

算 MD5（Windows 自带，不用装东西）：

```powershell
Get-FileHash .\Setup.SERVICES.FOR.Windows.1.1.3.exe -Algorithm MD5
Get-FileHash .\SERVICES.FOR.Windows.1.1.3.zip -Algorithm MD5
```

或者用软件自带的 **「5.MD5 校验工具」** → `1.计算文件 MD5`。

---

## 九、常见问题

**Q：部署成功了，但访问 `dl.sfw.us.ci` 显示 522 / 无法连接**

自定义域还在「初始化中」。去 Pages 项目 → 自定义域，看状态是不是「有效」。
不是的话等几分钟，或者删掉域名重新加一次。

**Q：安装包能打开页面但下载下来是 0 字节**

文件没真正传上去（Git LFS 指针、或者用了网页上传但中断了）。
去仓库里看看那个文件的实际大小，应该是 6360513 字节。

**Q：修改了 versions.json，页面没变**

浏览器缓存。`Ctrl + F5`。还不行就看 Cloudflare 的部署记录，
是不是这次提交没触发部署。

**Q：能不能把这个仓库也接上 GitHub Pages 做双活**

可以，但没必要 —— 两个站点的域名和证书要各配一遍，收益不大。
Cloudflare 这一路的带宽是不限的，本身就比 GitHub Pages 稳。

**Q：Cloudflare 那边要不要开代理（橙色云）？**

**不用。** Pages 自己会签证书，域名状态设成「有效」就自动有 HTTPS。
如果你在 DNS 页面看到这条记录是橙色云，改成灰色（DNS only）更省事，
否则可能出现「重定向次数过多」。
