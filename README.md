# LS-align 测试框架

LS-align（2020, *Bioinformatics*）的只读测试框架：白盒单测、黑盒 CLI 测试、
golden 快照与输出级逐字节回归。**不修改被测源码。**

被测源码是**外部代码树**，不在本仓库内。框架通过 `LSALIGN_SRC` 指向它的 `src/`：

```bash
make test                              # 默认源码树
make test LSALIGN_SRC=/path/to/src     # 指向其他源码树
```

默认值 `../LSalign_src20180916/src` 与本仓库拆分出来时的目录布局一致：

```
<工作区>/
├── LSalign_src20180916/              # 被测源码
└── LSalign-refactor-tests-framework/ # 本框架（含全部测试数据）
```

`LSALIGN_SRC` 是构建身份的一部分：切换它会使已编译的二进制失效并强制重编，
不会悄悄复用来自另一棵源码树的旧二进制。

## 快速开始

```bash
make test                    # unit + cli（CI 默认目标；不需要数据集，不跑 ASan）
make unit                    # 白盒单测：124 项
make cli                     # 黑盒 CLI：75 项（8 个脚本）
make golden                  # 重新生成 golden 快照
make check-golden            # 对比 golden 快照（2 个）
make output-diff             # 输出级回归：11 用例（regression_cases.txt）
make output-diff CASES=baseline
```

## 测试数据布局

每个测试类别自带一份完整的所需数据，可单独运行或拷贝；被多个类别共用的
文件（如 `self.mol2`）在每个类别目录下都有一份，这份重复是有意为之：

```
.
├── unit/data/        # unit 套件所需的全部数据（23 个）
├── cli/data/         # cli 套件所需的全部数据，含坏输入（14 个）
├── golden/           # golden 快照 + 校验脚本 check_golden.sh
│   └── data/         #   生成快照所用的输入（4 个）
└── output_diff/data/ # output_diff 全部用例所需数据（10 个）
```

修改共用输入时需同步修改所有使用它的类别目录，否则各套件会漂移。
所有数据路径都相对于仓库根，测试二进制始终从仓库根运行。

## 两份用例清单

`output_diff/` 下有两份格式相同的用例清单：

| 文件 | 用途 |
|---|---|
| `baseline_cases.txt` | 原始上游代码（冻结的 `master` 基线） |
| `regression_cases.txt` | 修复后的回归线（默认） |

选择方式：`--cases baseline` / `--cases regression` / `--cases <文件>`，
或 `make output-diff CASES=baseline`。

两者当前内容相同，作为共同起点；后续若两侧用例分叉（新增用例、切
`expected:` 模式），各改各的文件即可。

## 环境说明（本机）

- 动态链接 `libstdc++` 在此机器上不可用（collect2 退出 116）：必须
  `-static-libstdc++ -static-libgcc`（Makefile 已处理）。
- `make` 的执行环境里 `TMP`/`TEMP`/`TMPDIR` 为空，Makefile 显式导出了一个
  可写的临时目录，否则 g++ 会退到 `C:\Windows\`。
- ASan/UBSan 不可用（MinGW 无 libasan 运行时）；泄漏检查改用可移植的
  `leak_probe.h` 计数器。
- 退出码：成功为 0，失败为 1（B28）。数值断言仍解析 stdout，因为表格即契约；
  `cli/test_cli_args.sh` 专门覆盖退出码、非法选项值与 stdout 纯净性。

## xfail 约定

`XFAIL(name)` 表达"缺陷未修时**正确**的行为"：CHECK 今天失败 → XFAIL（绿），
缺陷修复后该用例变为 XPASS（红），迫使维护者把它提升为 `TEST`。

第 1 批缺陷已全部修复，因此**当前没有 `XFAIL` 用例**：`unit/test_xfail.cpp`
与 `unit/test_delete_mismatch.cpp` 里原本的期望失败用例已经是普通 `TEST`
回归锁。XPASS 仍是硬失败（说明有锁没提升）；`TEST` 失败说明缺陷回归。
新缺陷用例请直接写成 `TEST`。

## 不可达路径（R5）

R5 移除了无参 `LigRotamerDetector::generateRotamers()`（N=5，无对齐键过滤）
与三个未使用的 `FlexibleLSalign` 构造函数，只剩六参数构造函数和带参数的
`generateRotamers(RigidLSalign*)`（N=3）。因此
`generateRotamers(RigidLSalign*)` 里 `> 5000` 的确定性抽稀分支**当前从任何
公开调用者都不可达**（11^3 = 1331 上限）。`unit/test_rotamer.cpp` 保留
`x1_deterministic_decimation`，用测试专用的扩展角度集（20 个角度 → 8000 条记录）
锁定公开路径的可复现性并覆盖抽稀算术；它不声称该分支在生产中可达。
`NRS_LEGACY_UNPARAM_BOND_NUM`（=5）仅因采样器测试仍显式传入而保留。

## 可选的 DUD-E 数据集检查

需要 ~6.9 GB 数据集（默认 `../dataset`，可用 `LSALIGN_DATASET` 覆盖；
引用而非复制）：

```bash
bash dataset_tools/audit_ace.sh ace 200
```
