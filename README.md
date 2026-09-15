# dsh-plugin-dev

DeepSeek Harness（dsh）插件开发参考 Skill（**以本机安装的 dsh 源码为唯一依据**，当前核对版本 **0.1.2-rc.1**）。

写这个 Skill 的起因：dsh 的插件机制公开文档很少，能搜到的说法普遍停留在“复制文件就算装好”的层面，
而实际上 dsh 的插件是普通的 npm 包，由宿主半侧与 Web 客户端半侧分别加载，安装判据、patch 语义、
客户端模块契约都有明确的硬性约束。照记忆动笔，写出来的插件最常见的结局是“装完毫无反应”。
本仓库里的每个字段名、函数签名与判据，都取自本机 `dsh` 安装目录中的真实源码与运行态验证；
凡与源码对不上的说法，一律以源码为准。

## 内容结构

```
SKILL.md                          总览、开发工作流八步、双半侧取舍、常见故障
references/
├── host-half.md                  宿主半侧：webServer 注册种类与优先级、llm/stream 语义、ctx.effect 与日志约定
├── client-half.md                parseDshClient 字段契约、模块加载器 bundle 格式、共享模块基座、combo URL 上限
├── profile-and-patch.md          profile 目录布局、bundle 解析顺序、patch 覆盖语义、dsh plugin 转发与热重载
└── install-and-verify.md         安装步骤、三判据、以及打真实路由的 HTTP 验证命令
```

## 安装

**先分清两个东西**：本仓库是一个 **AstrBot Skill**（一份给模型看的参考资料），它的**内容**讲的是 dsh 插件开发。
它本身不是 dsh 插件，装进 dsh 不会有任何效果；dsh 插件另有自己的位置（见下一节）。

把它作为 Skill 装进 AstrBot 的 skills 目录：

```
<AstrBot>/data/skills/dsh-plugin-dev/
```

放到该位置后，AstrBot 会把它作为一个 Skill 提供给模型；也可以直接当文本资料阅读（例如在别的终端里
打开 `SKILL.md` 边看边写），此时不需要任何安装步骤。

### 那 dsh 插件装在哪

dsh 插件是 npm 包，安装位置是 dsh 的插件目录与 profile，而不是 AstrBot：

```
<dsh>/plugins/<plugin-name>/                插件本体
<dsh>/profiles/<profile>/node_modules/...   profile 里的链接
<dsh>/profiles/<profile>/package.json       dsh.profile.bundles 与 dependencies
```

三处同时成立插件才会被加载，具体步骤与判据见 `references/install-and-verify.md`。

## 覆盖范围

- 插件声明契约（`package.json` 中的 `dsh.bundle.patch` 与 `dsh.client`，含必填的 `platform`）
- 挂载方式（`cordis.patch.yml` 的 `insert` 挂行，以及 patch 整体替换而非字段合并的语义）
- 宿主半侧（`apply` / `inject` / `ctx.effect`、`ctx.webServer.register` 的重复注册异常、日志前缀）
- Web 客户端半侧（`window.__ModuleLoader__.load` 的懒加载 CJS 工厂格式、共享模块基座与依赖限制）
- 安装与验证（插件目录、profile 链接、`dsh.profile.bundles` 三者同时成立，再打真实路由看返回）

## 使用注意

- **版本敏感**：本 Skill 核对的是 dsh `0.1.2-rc.1`。dsh 升级后请以线上源码为准，动笔前先读安装目录里的
  `README.zh.md` 并 grep `lib/index.js` 复核，旧写法会静默失效。
- **不以第三方文档为准**：本仓库刻意不引用任何第三方教程，避免把过期写法带进来。
- **退出码不等于装好**：安装器返回 0 只说明文件动过，必须按 `install-and-verify.md` 复核三判据，
  并以真实路由的返回内容与首次生成的运行态存储作为最终证据。
- 所有示例路径都用 `<dsh>` / `<AstrBot>` 占位，指安装根目录，不绑定具体机器。

## 许可

MIT License，见 [LICENSE](LICENSE)。
