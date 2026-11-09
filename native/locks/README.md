# 原生写入器锁

`locks.c` 是一个第一方 C Node-API v3 插件，暴露不透明的 `openLock`、非阻塞 `tryLock` 和幂等 `close` 操作。POSIX 使用 `flock`；Windows 使用 `LockFileEx`。进程死亡时内核释放所有权。`src/core/persistence/native-lock.ts` 中的 TypeScript 封装增加了可取消的异步等待，默认 5 秒截止时间和 25 毫秒轮询，仅在调用方需要锁能力时加载插件。

调用方必须提供绝对、稳定的锁路径，位于宿主控制的目录中，且不在任何可替换的数据存储或工作树内。释放后保留该文件。切勿删除、轮转、覆盖，或根据其年龄或元数据推断所有权：替换已锁定的 inode 可能导致两个所有者。叶符号链接、重解析点和非常规文件会被拒绝。目录限制和文件系统的跨进程锁语义是宿主提供的前提条件；此绑定不建立跨主机的分布式所有权。插件不可用或操作系统错误会产生 `writer_lock_unavailable`。没有时间戳、PID 或 TTL 所有权回退。关闭失败必须停止发布；不能将其视为成功放弃。

Windows IPC 在 `windows-ipc.h` 中增加了两个范围狭窄的操作。命名管道从探测之前到监听器实际关闭期间持有一个非阻塞的全局内核互斥锁。Windows 不变 Unicode 大写加 CNG SHA-256 使大小写和前缀别名具有同一身份，独立于主目录和登录会话。环境注册表和一个小的共享内核所有者记录防止同一线程上通过独立插件副本递归获取；不透明的终结器和清理在所属线程上释放。终结器释放失败会保留句柄和注册表引用，直到验证清理完成。被放弃的互斥锁仅通过内核获取。命名空间权限错误会拒绝绑定。

对于可证明已死亡的 Windows AF_UNIX 监听器，清理需要仍持有的不透明文件绑定声明。它打开叶节点而不跟随重解析点，验证 `IO_REPARSE_TAG_AF_UNIX`，并将该确切句柄标记为删除。普通文件、目录、其他重解析标签和访问错误会被拒绝。这避免了将 Bun 的陈旧套接字 `lstat` 错误视为任意文件系统条目可被删除的证据。

## 分发与可复现构建

所有八个插件均已检入，因此源码安装可使用 `bun install --frozen-lockfile --ignore-scripts`。它们支持 Linux glibc（2.17 ABI 基线）、Linux musl、macOS（13.0 部署目标）和 Windows 上的 x64 与 arm64。必需的 CI 矩阵覆盖仓库最低 Bun 版本 1.4.4，它也是发布编译器。操作系统兼容性还要求所选 Bun 版本自身的平台最低要求。

Node-API 头文件及其上游许可证从 Node v22.15.0 引入。`darwin-abi.h` 声明了无 SDK 交叉编译所需的狭窄公共 Darwin LP64 ABI，常量/布局已对照 Apple XNU 标签 `xnu-11215.81.4` 检查；macOS CI 针对实际 SDK 编译 `abi-check.c`。x86_64 的 `fstat$INODE64` 符号和 arm64 的 `fstat` 符号不同。不重新分发任何 Apple SDK。Windows 通过 `windows-napi.h` 从运行的可执行文件解析其使用的 Node-API 符号，包括重命名的编译 CLI。它从不加载单独的 `node.exe`。进程级一次性守卫在任何 API 调用之前发布完整函数表；缺失导出会拒绝注册，而不借用其他运行时的环境。Windows IPC 辅助程序链接操作系统提供的 `bcrypt` CNG 库，并保持在 Node-API v3 和现有 Windows 平台最低要求之内。

## 导出发布

`export-publication.h` 增加了 `beginExport(absoluteDestination)`、`publishExportFile(handle, relativePath, Buffer)`、`publishExportFileFrom(handle, relativePath, absoluteSource)`、`completeExport(handle)` 和幂等 `closeExport(handle)`。`publishExportFileFrom` 通过 1 MiB 读取将暂存的常规文件（绝对路径，拒绝叶符号链接和重解析点）复制到同一个不替换叶发布中，因此大页面永远不会整体驻留内存。`nativeExportPublisher()` 通过相同的字面插件导入异步加载这五个同步方法。导出句柄有自己的注册表和清理钩子；它们不能传递给锁操作，反之亦然。当插件或操作系统原语不可用时，没有文件系统回退。

目标必须由操作员控制。现有组件一次打开一个，不跟随符号链接或 Windows 重解析点。缺失目录在 POSIX 上私有创建，在 Windows 上使用继承权限创建。Windows 保留祖先句柄而不共享删除，并接受绝对驱动器路径，而非 UNC/设备命名空间。POSIX 接受绝对路径；macOS 惯用的 `/tmp` 和 `/var` 符号链接别名必须使用其真实目录路径提供。相对发布路径使用 `/`，不包含空/点/父组件，并在所有平台上拒绝 Windows 设备名、备用流、反斜杠、控制字符和尾随点/空格。组件限制为 255 个 UTF-8 字节，遍历限制为 256 个组件。

Begin 独占创建 `.gbrain-export-status` 并写入和刷新 `GBRAIN EXPORT INCOMPLETE\n`。现有标记永远不会被移除或覆盖。每个文件在发布前完全写入并刷新到同目录独占临时文件。POSIX 使用 `linkat(..., 0)` 然后移除临时名称并 `fsync` 目录；Windows 使用 `SetFileInformationByHandle` 配合 `FileRenameInfo`、`ReplaceIfExists=FALSE`、空 `RootDirectory` 和绝对目标路径。保留的祖先句柄在整个重命名过程中拒绝删除共享，因此目标组件无法被替换。现有文件、硬链接别名、目录和符号链接无法被替换。Windows 文件使用 `FILE_FLAG_WRITE_THROUGH` 和 `FlushFileBuffers`；这并非声称具备 POSIX 目录 fsync 或整卷断电持久性。Microsoft 的契约记录在 [FILE_RENAME_INFO](https://learn.microsoft.com/en-us/windows/win32/api/winbase/ns-winbase-file_rename_info) 和 [FlushFileBuffers](https://learn.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-flushfilebuffers) 中。

发布错误会永久阻止该句柄完成。Complete 追加并刷新 `COMPLETE\n`；完成失败会尝试截断该追加并刷新原始不完整标记，然后拒绝进一步使用。与所有文件系统持久性一样，同时拒绝恢复写入的设备无法提供保证的磁盘回滚。关闭、终结、进程退出和错误会保留标记和任何已发布输出；它们永远不会擦除未知或预先存在的文件。成功完成不会隐式关闭。

在每次 POSIX 发布和完成之前，插件重新打开保留的目标和每个祖先组件而不跟随符号链接，并将其设备/inode 身份与保留的句柄进行比较。它还验证当前标记名称仍标识保留的常规文件。发布在暂存后重复这些检查，包括其输出父链。缺失或被替换的组件或标记会在进一步发布或完成之前毒化句柄，而不修改替换树或标记。这检测调用之间的拓扑变化；它不能消除针对敌对同等特权进程的最后身份检查后的竞态。

暂存名称为 `.gbrain-export-<128-bit-random-run-id>.tmp`。普通失败仅移除该调用刚创建的暂存文件。同步发布允许每个导出最多一个未完成暂存文件；崩溃可能留下该文件（可能部分写入），加上已完成输出和不完整标记。失败的导出不会自动重试或回收其他运行的遗留物。操作员可以在重试前检查并移除其失败的目标。这是针对操作员控制目录的锚定限制，而非针对同等特权敌对进程移动或编辑导出树或其保留祖先的安全声明。

`bun test test/native-export-publication.test.ts test/native-export-faults.test.ts` 检查真实不替换行为、标记所有权、句柄分离、组件拒绝、竞争导出器、生命周期和权限。仅 Linux/glibc 的故障套件编译一个仅测试用的插入器，以针对真实插件执行写入、刷新、终端刷新和 SIGKILL 失败；它不链接到发布的二进制文件中。原生 macOS 和 Windows 执行以及 Darwin SDK ABI 检查需单独要求；仅交叉编译不能验证运行时行为。

使用固定的 Zig 0.14.1 编译器。归档 URL、SHA-256 哈希和大小在 `scripts/native/toolchain.json` 中；设置在解压前验证它们。

```sh
bun scripts/native/setup-toolchain.ts --dir .context/native-toolchain
ZIG=/absolute/path/to/the/downloaded/zig bun scripts/native/build.ts --target all --write-manifest
bun scripts/native/verify.ts
ZIG=/absolute/path/to/the/downloaded/zig bun scripts/native/build.ts --output /tmp/native-rebuilt
bun scripts/native/verify.ts --rebuilt /tmp/native-rebuilt
```

清单哈希每个插件以及构建输入、编译器配方和工具链元数据。一起提交源码、所有八个重新生成的二进制文件和清单。构建固定源时间戳、macOS 安装名称和 Linux 构建 ID 行为，以允许跨输出目录进行逐字节比较。CI 在其平台上重建每个目标并比较字节。

## 运行时验证

`bun test test/native-lock.test.ts` 运行真实的竞争 Bun 进程、保留 inode 检查、取消、截止时间清理、活跃持有者陈旧性、SIGKILL 恢复以及失败关闭的缺失插件/无效路径情况。`test/scripts/native-lock-prebuilds.test.ts` 证明源码/二进制篡改会导致验证失败，并检查必需的 CI 矩阵覆盖所有二十四个目标/运行时对。原生 CI 还在原生 musl 用户空间中运行测试。

`bun scripts/native/compiled-smoke.ts` 构建一个导入确切生产封装的重点可执行文件，证明两个编译进程互相排斥，杀死持有者并证明立即交接。发布 CI 额外传递 `--binary bin/<artifact>` 以要求实际 CLI 可执行文件嵌入确切的当前平台插件。该断言验证打包；重点可执行文件验证编译锁执行。

`bun scripts/native/cli-persistence-smoke.ts --binary bin/<artifact>` 将实际发布可执行文件复制到源码检出之外，并演练无密钥磁盘 PGLite 初始化、原生所有权激活、规范发布、带类型化回执的修订冲突、精确重放、常驻 stdio/CLI IPC 以及关闭/重新打开。发布 CI 对已发布的 Linux x64 和 macOS arm64 产物运行它。子主目录和凭据被隔离；脚本从不加载仓库 TypeScript 或相邻原生文件来满足可执行文件。

## 测试与 CI

`bun test test/native-lock.test.ts test/scripts/native-lock-prebuilds.test.ts` 检查真实进程排斥、崩溃交接、保留文件、取消、缺失插件失败和源码/二进制清单完整性。测试使用隔离的临时路径，从不打开操作员数据存储。必需的 `native-locks.yml` 通道在 Bun 1.4.4 上重建并执行所有八个操作系统/架构/libc 目标，包括原生 musl Docker 用户空间。每对还运行 `bun scripts/native/compiled-smoke.ts` 以证明编译进程锁定。发布 CI 验证发布的 CLI 嵌入匹配的插件，并在其两个发布平台上运行编译冒烟测试。重建说明和精确的打包/运行时区别如上。发布编译使用 Bun 1.4.4；严格的 Darwin 代码签名验证必须在发布前通过。原生 macOS 26.2 冒烟测试不是 macOS 27 认证，Linux 故障注入不是完整的原生 Windows 备份创建/恢复测试。

OpenClaw 2026.9.4 / Node 24.18.0 原生宿主夹具使用确定性回环提供者证明插件启动、重启轮次保存页面指针检索和同 slug 源隔离：

```bash
GBRAIN_TEST_OPENCLAW_BIN=<absolute-installed-cli> \
GBRAIN_TEST_OPENCLAW_DATABASE_URL=<isolated-postgres-test-db> \
bun test test/openclaw-context-engine-native.serial.test.ts
```

数据库用户需要 `CREATEDB`；夹具创建/删除唯一数据库，而非截断共享行。真实提供者召回和 macOS 27 行为仍未验证。

此目录中的 `package.json` 是私有包边界，而非包：从 `.node` 文件上方最近的 `package.json` 派生原生插件命名空间的插件宿主否则会将整个检出视为一个包并缓慢加载（#6026）。它不声明入口点，不是预构建输入，且绝不能移入 `prebuilds/`，后者恰好存放八个插件。