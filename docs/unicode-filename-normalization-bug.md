# Unicode 文件名规范化导致的 Git 分支切换故障

## 摘要

在 macOS 上从 `play/my-simulation` 切换到
`reform/materialist-system-optimization` 时，Git 中止了 checkout，并报告：

```text
error: The following untracked working tree files would be overwritten by checkout
```

故障涉及 16 个包含 `Étienne Bellamy` 或 `Lite Arévalo` 的 persona、profile
及 character scratchpad 路径。文件内容没有损坏，也没有真正的未跟踪副本；根因是两个分支使用了
不同的 Unicode 文件名规范化形式。

## 技术背景

Unicode 允许同一个可见字符使用不同的码点序列表示。例如 `é` 可以表示为：

- NFC：单一码点 `U+00E9`。
- NFD：普通字母 `e`（`U+0065`）加组合重音符（`U+0301`）。

两种形式显示结果相同，但 UTF-8 字节不同。Git 将路径视为不透明的字节序列，因此会把 NFC
和 NFD 路径视为不同路径。macOS 的 APFS 则对这两种形式不敏感：使用 NFC 路径访问 NFD
文件名时，仍可能命中同一个磁盘文件。

## 故障来源

Git 历史检查确认，原始 main 数据使用的是 NFC，异常路径首次出现在提交
`cb66d09`（提交信息 `test`，2026-07-16）中。该提交把 16 条路径从 NFC 改成了
NFD，Git 将其识别为内容相同的 `100% rename`。

当时仓库的本地配置为：

```text
core.precomposeunicode=false
```

因此，macOS 文件系统或相关 API 暴露出的 NFD 名称没有在进入 Git 索引前重新组合为 NFC。
执行 `git add` 并创建 `cb66d09` 后，这些名称差异被正式记录进
`play/my-simulation` 的历史。后续提交 `6031930` 继续继承了这些路径。

这不是原始 main 数据主动产生的错误，也没有证据表明业务代码显式执行了 NFC 到 NFD 的
转换。直接触发因素是 macOS 文件名行为、Git 配置以及一次提交操作的组合。

不过，源代码当时存在健壮性缺口：程序直接使用 `Path.name` 及模型或历史数据提供的角色名
拼接 persona、profile、contact 和 scratchpad 路径，没有统一执行 NFC 规范化。这使 NFD
字符串一旦进入运行时，就可能继续传播并创建更多 NFD 路径。

## checkout 失败机制

`play/my-simulation` 的 Git 索引记录 NFD 路径，而
`reform/materialist-system-optimization` 记录 NFC 路径。切换分支时，Git 准备写入
目标分支的 NFC 文件，并先检查目标路径是否已经存在。

APFS 使用 NFC 路径查询时命中了磁盘上的 NFD 文件；Git 再用 NFC 字节串查询当前索引时，
却找不到对应条目，因为索引里保存的是 NFD 字节串。Git 因而把该文件误判为可能被覆盖的
未跟踪文件，并出于数据保护目的中止 checkout。

当前分支执行 `git status` 仍显示干净，是因为该分支的索引和工作区当时都使用 NFD。只有在
切换到采用 NFC 路径的分支时，这个差异才会暴露。

## 修复过程

首先扫描了全部本地分支的已跟踪路径：

- `main`：0 条非 NFC 路径。
- `reform/materialist-system-optimization`：0 条非 NFC 路径。
- `play/my-simulation`：16 条 NFD 路径，无 NFC 路径碰撞。

随后比较了 16 个文件在两个分支中的 Git blob 哈希。全部哈希一致，证明差异仅存在于路径
编码，文件内容完全相同。

历史数据修复在独立 Git worktree 中完成。由于 APFS 无法可靠地直接区分仅规范化形式不同
的两个名称，每条路径先改为唯一临时名称，再从临时名称改为 NFC。最终 Git 将所有改动识别
为 100% rename，统计结果为 0 行新增、0 行删除。

为防止问题再次出现，运行时增加了 NFC 规范化防线：

- `src/world/world.py`：从 persona 目录读取角色名时转换为 NFC。
- `src/agents/role_agent.py`：构造 `RoleAgent` 时规范化角色名。
- `src/agents/data_manager.py`：在角色、世界、scratchpad、profile、联系人及消息参与者等动态
  路径参数进入文件系统前统一转换为 NFC。

## 修复提交

- `play/my-simulation`：`8e6ea96 Normalize Unicode filenames to NFC`
- `play/my-simulation`：`08ef30e Normalize dynamic path names to NFC`
- `main`：`1536de9 Normalize dynamic path names to NFC`
- `reform/materialist-system-optimization`：`1ddb1e2 Normalize dynamic path names to NFC`

以上提交创建时均为本地提交，是否推送远程应根据后续仓库操作确认。

## 验证结果

修复后完成了以下验证：

- 三个本地分支的 Git tree 均为 0 条非 NFC 路径。
- 三个分支均通过 `python3 -m compileall -q src`。
- `git diff --check` 未发现空白符错误。
- 实际执行 `reform -> play -> main -> reform` 连续切换成功。
- 未修改或删除工作区中的未跟踪模拟输出目录。

## 后续约束

所有会进入文件路径的外部字符串都应先使用以下方式规范化：

```python
import unicodedata

normalized = unicodedata.normalize("NFC", value)
```

尤其需要关注从 `Path.name`、`os.listdir()`、模型工具参数、JSON 历史记录以及用户输入中取得
的角色名。以后若再次发现肉眼相同但 Git 判断不同的路径，应优先检查 Unicode 规范化形式，
不要直接使用强制 checkout 或删除未确认内容的文件来绕过保护。
