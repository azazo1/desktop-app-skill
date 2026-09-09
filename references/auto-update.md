# 桌面应用自动更新机制 (客户端侧)

本文只覆盖应用内部的更新实现, 与具体语言和 GUI 框架无关. 发布侧 (CI 打包, 产物命名, SHA256SUMS 生成, tag 流程) 由 create-github-release-flow skill 保证, 本文不重复; 实现更新功能时需与其产物契约对齐.

推荐将更新实现为独立模块, 按职责分四层:

- 更新运行时: 状态机, 启动静默检查, 设置持久化.
- 发布查询: latest release 查询, 版本比较, 平台资产匹配.
- 下载校验: SHA256SUMS 解析, 流式下载与校验.
- 安装: 平台安装策略, 二进制替换, 重启与备份清理.

## 状态机

`Idle -> Checking -> (UpToDate | Available | Failed)`, `Available -> Downloading -> (ReadyToRestart | DmgOpened | Failed)`.

- 更新状态集中存放并加锁, UI 读取不可变快照渲染; 检查, 下载, 安装在后台任务中执行, 状态变化后通知 UI 刷新.
- `Failed` 携带用户可读的错误信息; 手动检查失败记 warn, 静默检查失败只记 debug, 不打扰用户.

## 检查更新

- 检查接口: `https://api.github.com/repos/<owner>/<repo>/releases/latest`, 请求头 `Accept: application/vnd.github+json`. 该接口自动排除 draft 与 prerelease.
- HTTP 403 与 429 识别为 GitHub API rate limit, 给出明确错误信息.
- 版本比较: 当前显示版本与候选 tag 都按运行时版本号约定归一化 (strip 可选 `v` 前缀, 截断 `-<commit>` 与 `^<commit>` 后缀) 后解析 semver, 候选严格大于当前版本才提示; 同版本视为不更新, 覆盖本地带 commit 后缀重构建的场景.
- `dev-build` 等无法解析出 semver 的构建永远不提示更新, 保证开发构建安静.

## 检查时机

- 启动静默检查: 启动后延迟 5 秒执行一次, 仅当自动检查开启时执行; 该设置默认开启, 以 `update.auto_check` 为键持久化, 更新窗口开关与托盘勾选项双向同步.
- 静默检查发现新版本且该版本未被用户跳过时, 发系统通知引导到主界面; 网络失败只记日志.
- 手动检查入口: 托盘菜单 "检查更新" (触发检查并打开更新窗口), 更新窗口内 "重新检查".

## 资产匹配与下载校验

- 客户端按精确文件名匹配 release 资产: `<app>-<tag>-<platform>-<arch>.<ext>`, 其中 ext 随平台为 linux `tar.gz`, windows `zip`, macos `dmg`; 同一 release 必须存在 `SHA256SUMS` 资产. 匹配不到直接报错.
- 先下载 SHA256SUMS, 按归档文件名提取期望摘要; 解析容忍 `*` 二进制标记与 CRLF 行尾, 缺少对应行时报错.
- 归档流式下载到 `<data_dir>/update/<name>.part`, 边写边计算 sha256, 每个数据块触发进度回调 (received 与 total, total 来自 Content-Length, 可能为 None).
- 摘要大小写不敏感比对, 不匹配则删除 `.part` 并报错; 匹配后 rename 为正式文件名落盘.

## 安装策略 (平台差异)

- linux / windows, 安装版与便携版通用: 解包归档 (tar.gz / zip), 替换运行中的自身二进制.
  - 运行中的二进制允许 rename 不允许覆盖删除: 先把当前可执行文件 rename 成 `<exe>.old` 让位, 再把新二进制 move 到原路径; 中途失败把 `.old` 移回来回滚, 保证应用始终可用.
  - rename 因跨文件系统失败时, 退回复制后删除源文件.
  - unix 下设置 0o755 可执行权限.
  - 替换成功进入 ReadyToRestart; 用户点击 "重启应用" 时以新二进制启动新进程接管, 当前进程随后走正常退出路径.
  - 每次启动时清理上次更新遗留的 `.old` 备份, 文件被占用则静默留待下次.
- macos: 产物是 dmg, 程序内无法静默替换已安装的 `.app` (权限与签名限制), 只能用系统方式挂载并打开 dmg, 引导用户手动拖拽安装, 状态进入 DmgOpened, UI 提示拖入 Applications 后重启. 便携版同样走 dmg 引导.
- windows 的 release 构建是 GUI 子系统, 没有控制台, 全程错误必须落到日志与 UI, 不能依赖标准错误输出.

## UI 与托盘集成

- 状态栏: 无更新时显示当前版本号; Available / ReadyToRestart / DmgOpened 状态时替换为加粗链接, 引导打开更新窗口.
- 更新窗口: 当前版本; 各状态对应文案与控件; release notes 滚动区; 下载进度条与已下载字节数; 操作按钮 "立即更新", "跳过此版本" (以 `update.skipped_version` 为键持久化, 静默检查不再提示该版本), "查看 Release 页" 外链, "重启应用", 失败信息与重试.
- 托盘菜单: "检查更新" 项 + "启动时自动检查更新" 勾选项.
