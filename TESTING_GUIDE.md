# 测试指南：照着做就可以

这份指南面向第一次运行项目的人。你暂时不需要理解架构里的缩写，也不需要先安装 Docker、打开 QuPath 或准备病理切片。

**今天先完成第 2–4 节：检查程序、跑统计例子、看代码版本记录。** 第 0 阶段的实机检查放在后面，有需要再做。

原来较详细的技术内容已保留在 [测试技术参考](docs/testing-reference.md)，不是删除了验收要求。

**2026-10-24 新入口：** 本周单 ROI 计数与初步性状程序已完成首版；75 项 Python 回归、安装版 QuPath 合成图链路及 16 项安装包 CLI 检查通过。现在可按 [第 9 节](#9-本周-roi-计数与初步性状首版2026-10-24) 准备固定脚本。真实切片按用户选择稍后验收，不把合成图通过称为真实染色规则已确认。

## 1. 先弄清楚：现在能测试什么

| 测试 | 通俗解释 | 现在能不能做 |
|---|---|---|
| 自动检查 | 让程序自己做一套练习题，检查结果是否符合预期 | 可以 |
| 统计例子 | 给程序一份人工编好的“细胞类别名单”，让它算数量和比例 | 可以 |
| 任务管理例子 | 检查一次统计任务是否完成、结果是否保存 | 可以 |
| 代码版本例子 | 看第一版代码、修改意见、第二版代码能不能对应起来 | 可以；只是保存和检查代码，不执行它 |
| QuPath 单 ROI 固定脚本 | 安装版检测合成细胞、测量五 marker，接 Harness 生成报告 | 首版及合成图链路通过；真实 ROI 与交互 GUI 待验收 |
| 在 QuPath 中点按钮分析真实图像 | 真正连接软件和分析流程 | 尚未完成 |
| 让大模型写代码并运行 | 研究目标中的完整流程 | 尚未完成 |

所有现有例子都用人工构造的数据，不涉及患者信息。测试通过不代表已经能诊断癌症。

## 2. 准备：打开正确的文件夹

以下步骤针对你当前这台 Windows 电脑，使用 PowerShell。

### 第一步：打开 PowerShell

打开 Windows 开始菜单，搜索 **PowerShell** 并打开。普通权限即可，不需要“以管理员身份运行”。

把下面这行复制进去，按回车：

~~~powershell
Set-Location 'E:\XJTLU\大四\FYP\code\qupath-agent'
~~~

这一步只是进入代码文件夹，不会修改文件。后面的命令都在这个窗口里运行。

### 第二步：确认运行工具在不在

一行一行运行：

~~~powershell
.\.venv\Scripts\python.exe --version
java -version
~~~

你应该看到：

- 第一条显示 `Python 3.12...`。当前电脑已经准备好了这个 Python，不需要重新安装。
- 第二条显示 Java `21...`。版本信息可能显示成红色文字；只要显示了版本、没有“找不到命令”，不一定是出错。

如果提示找不到文件或命令，先看第 5 节，不要继续盲目运行后面的步骤。

**不用输入 `python` 代替上面的长路径。** 当前电脑默认的 Python 可能是旧版；直接复制这条长路径最稳妥。也不用先“激活环境”。

## 3. 先做一次自动检查

复制下面这一行，回车后等它运行结束：

~~~powershell
.\.venv\Scripts\python.exe -m unittest discover -s tests -q
~~~

### 怎么判断成功？

看最后几行，当前版本应类似：

~~~text
Ran 71 tests in ...s

OK
~~~

意思是：本版本的 71 个检查都通过了，耗时多少秒不重要；以后数量可能增加。

| 最后看到什么 | 应该怎么理解 |
|---|---|
| `OK`，没有 skipped | 本次自动检查全部通过 |
| `OK (skipped=...)` | 有检查没运行，不能算全部通过；先检查 Java 是否可用 |
| `FAILED`、`ERROR` 或 `FAIL` | 有问题；保留错误内容，先不要继续演示 |

**这些是软件检查，不是病理切片数量，也不是模型准确率。** 关键是本次全部通过且没有跳过。

## 4. 再看三个可以展示的例子

建议按 4.2 → 4.3 → 4.5 的顺序操作。4.1 先解释一个所有例子都会用到的小规则。

### 4.1 为什么每次要生成一个新文件夹？

程序不会覆盖旧结果，避免把上一次的记录弄丢。

下面的命令会给输出文件夹加上当前时间，例如 `stats-20261018-153000-123`。你不需要自己填日期，整段复制即可。

`$statsDir`、`$taskDir`、`$researchDir` 可以理解成给文件夹地址起的临时名字。**在同一个 PowerShell 窗口按顺序操作，不要中途关闭。** 想重跑某个例子时，重新运行该例子的整个命令块即可。

### 4.2 例子一：让程序算一份细胞类别名单

**你在测什么？** 检查程序是否按照给定规则正确计算数量和比例。不是让程序识别图像。

复制运行：

~~~powershell
$statsDir = 'local-data/stats-' + (Get-Date -Format 'yyyyMMdd-HHmmss-fff')
.\.venv\Scripts\python.exe -m agent_core --case contracts/golden/marker_positivity_summary/basic --output $statsDir
~~~

看到 `PASS L2` 就说明这份结果通过了程序的一致性检查。

再运行下面这一行，用记事本打开统计结果：

~~~powershell
notepad "$statsDir/results.json"
~~~

文件里的英文不必全部理解。用记事本的“查找”（Ctrl+F）搜索 `"metric_id": "count"`，第一处应是下面这一小段内容：

~~~json
"metric_id": "count",
"marker": "CD8",
"value": 3,
"unit": "cells"
~~~

意思是：“CD8 的数量是 3 个”。`metric_id` 是指标名称，`marker` 是统计对象，`value` 是结果，`unit` 是单位。不要把文件里所有 `value` 都当成同一个指标。

对照下面这些数字即可：

| 指标 | 应该看到的值 |
|---|---|
| CD8 数量（count） | 3 |
| PD-L1 数量（count） | 2 |
| CD8 比例（positive_percent） | 60，表示 60% |
| PD-L1 比例（positive_percent） | 40，表示 40% |
| 共表达数量（coexpression_count） | 1 |
| 共表达比例（coexpression_percent） | 20，表示 20% |

为什么是这些数？输入里有 7 个对象，按这份演示规则排除 2 个后，统计分母为 5。CD8 有 3 个，所以是 3 ÷ 5 = 60%。

想打开结果所在文件夹，运行：

~~~powershell
Invoke-Item $statsDir
~~~

里面的 `results.json` 是结果，`report_L2.json` 是检查记录。JSON 只是文本形式的数据文件，用记事本就能看。**不要修改原始测试名单或用程序结果覆盖参考答案。**

### 4.3 例子二：看任务有没有被管理和记录

**你在测什么？** 不是换一种算法，而是让“任务管理底座”安排同一类统计工作并保存记录。这套底座叫 Harness，你可以先把它理解成任务管家。

复制运行：

~~~powershell
$taskDir = 'local-data/task-' + (Get-Date -Format 'yyyyMMdd-HHmmss-fff')
.\.venv\Scripts\python.exe -m harness_core --data-dir $taskDir
~~~

终端会出现一大段文字，暂时只找这一行：

~~~text
"state": "REPORT_READY"
~~~

它的意思是：**这次任务完成了，结果已准备好。** 不需要读懂那些很长的编号。

想看保存位置：

~~~powershell
Invoke-Item $taskDir
~~~

里面有任务记录数据库和结果文件。数据库不是普通文本文件，不需要用记事本打开，也不要手动修改。

注意：这里调用的是提前写好的统计工具，不是大模型自主完成了分析；也没有修改 QuPath。

### 4.4 三个例子的“成功”不是一回事

| 例子 | 成功意味着什么 | 不意味着什么 |
|---|---|---|
| 统计例子 | 固定规则的计算和检查完成 | 从真实切片识别出了细胞 |
| 任务管理例子 | 固定任务完成，记录和结果已保存 | 已实现完整智能 Agent |
| 下一节的代码版本例子 | 代码文本、修改意见和版本关系保存并检查完成 | 已执行代码或验证了医学效果 |

### 4.5 新增研究主线：规格、源码候选与反馈修订预检

通俗地说，这个例子演示的是：**保存第一版代码 → 记录修改意见 → 保存第二版代码 → 检查这些记录有没有对应错。**

现在两版代码和修改意见都是人工准备的演示材料，不是大模型现场生成，也不是真实医生反馈。

复制运行：

~~~powershell
$researchDir = 'local-data/research-' + (Get-Date -Format 'yyyyMMdd-HHmmss-fff')
.\.venv\Scripts\python.exe -m agent_lab --output $researchDir
~~~

终端会输出两份检查记录。重点找下面这两个词：

~~~text
"state": "AWAITING_ISOLATION"
"execution_status": "NOT_EXECUTED"
~~~

它们不是报错，而是预期结果：

- `AWAITING_ISOLATION`：只做了预先检查，还没到允许执行代码的阶段。
- `NOT_EXECUTED`：候选代码没有运行。当前就是故意不运行，避免在隔离条件未验证前执行实验代码。

打开生成的文件夹：

~~~powershell
Invoke-Item $researchDir
~~~

先用记事本打开以下文件，不必全部看：

| 文件 | 通俗解释 | 重点看哪里 |
|---|---|---|
| `spec.json` | 这次要求怎么算 | 阈值为 128，要求“大于或等于” |
| `candidate-v1.json` | 第一版代码记录 | `source` 里的比较符号为 `>` |
| `feedback-v2.json` | 对第一版的修改意见 | 指出没有包含“等于阈值”的情况 |
| `candidate-v2.json` | 第二版代码记录 | 比较符号改成 `>=` |
| `preflight-v2.json` | 第二版的预先检查记录 | 仍然是 `NOT_EXECUTED` |
| `reference.json` | 固定参考程序算出的答案 | 4 个前景像素、共 8 个像素、比例 0.5 |

第一版和第二版都可能显示 `PASSED`。这只是说明代码符合当前检查的结构要求，**不说明算法正确**。初版故意写错比较符号，就是为了展示这个区别。

再强调一次：`reference.json` 的 0.5 是固定参考程序的答案，不是运行第二版候选代码得到的结果。这里的“像素”也不是“细胞”或“癌症阳性”。

**做到这里，你已经完成了当前最适合演示的三组例子。** 不需要为了这次演示继续做 Docker 或 QuPath 实机测试。

### 4.6 导入外部代码文本，再检查保存记录

现在可以自己指定三份文件：任务要求 `spec.json`、合成像素 `input.json`、UTF-8 代码文本。第一次可用刚才的演示材料练习；下面会生成新目录，不修改旧记录，也不会运行代码文本。

~~~powershell
$stamp = Get-Date -Format 'yyyyMMdd-HHmmss-fff'
$sampleDir = "local-data/import-sample-$stamp"
$sourceFile = "local-data/candidate-$stamp.txt"
$externalDir = "local-data/external-$stamp"
.\.venv\Scripts\python.exe -m agent_lab --output $sampleDir
.\.venv\Scripts\python.exe -X utf8 -c "from pathlib import Path; from agent_core.contracts import read_json; candidate = read_json(Path(r'$sampleDir') / 'candidate-v2.json'); Path(r'$sourceFile').write_bytes(candidate['source'].encode('utf-8'))"
.\.venv\Scripts\python.exe -m agent_lab --spec "$sampleDir/spec.json" --input "$sampleDir/input.json" --source $sourceFile --output $externalDir
~~~

最后一条就是外部导入入口。换用自己的材料时，只替换三份输入文件的路径；只允许明确获准的合成数据。示例中代码仍来自人工演示材料，`external_text` 只表示通过外部文本入口导入，不表示模型生成。

正常结果仍是 `AWAITING_ISOLATION / NOT_EXECUTED`。不支持或危险的源码会保存 `REJECTED` 报告并返回退出码 1；输入不合法则返回 2，不创建成功产物。命令后可用 `$LASTEXITCODE` 查看退出码。

再检查刚生成的目录：

~~~powershell
.\.venv\Scripts\python.exe -m agent_lab --check-bundle $externalDir
~~~

看到 `"integrity": "VERIFIED"` 表示文件、摘要和绑定关系一致。缺文件、额外文件、损坏或错绑会拒绝并返回 2；不会自动修复。**记录完整不表示候选正确、安全或获准执行**；保存了 `REJECTED` 的完整包也可以通过完整性检查。

### 4.7 准备一份最小 Coder 输入

~~~powershell
$coderFile = "local-data/coder-input-$stamp.json"
.\.venv\Scripts\python.exe -m agent_lab --prepare-coder $externalDir --output $coderFile
notepad $coderFile
~~~

看到 `PREPARED` 后，文件中只有必要任务字段、合成像素和函数接口；没有参考答案、候选、反馈、凭据或本机路径。输出文件必须是新文件，并放在研究产物包外。这个操作只准备材料，没有向模型发送请求，也没有形成 Agent Loop。

安装包复测方法与大小限制见 [测试技术参考 §8](docs/testing-reference.md#8-外部文本产物检查和安装验证2026-10-19)；本次完整交接见 [开发交接](docs/gpt-five-hour-handoff.md)。

## 5. 遇到问题，先看这里

| 看到的问题 | 先这样处理 |
|---|---|
| 找不到 `.venv\Scripts\python.exe` | 重新执行第 2 节的 `Set-Location`；仍找不到，说明 Python 环境缺失，不要删除项目 |
| `No module named agent_core` 或 `agent_lab` | 通常是所在文件夹不对；重新执行第 2 节的 `Set-Location` |
| `No module named pydantic` 等依赖错误 | 在代码文件夹运行下面的依赖安装命令，再重新测试 |
| `java` 不是可识别的命令 | 需要配置项目使用的 JDK 21；没有 Java 时，跨语言检查可能被跳过 |
| 提示文件夹已存在、`FileExistsError` | 旧结果被保护了；重新运行对应例子的整个命令块，生成新名字，不要删除旧结果 |
| `FAILED` 或一段错误堆栈 | 停在这里，复制从报错开始到最后一行的内容；不要只截图“失败”两个字 |
| `NOT_EXECUTED` | 研究预检的正常状态，不需要想办法把它改成“已执行” |
| 外部导入返回 1 / `REJECTED` | 源码不符合窄结构白名单；查看 `preflight-v1.json` 中的 issues，不执行源码 |
| 检查或导入返回 2 | 输入、产物、链接、大小或输出位置不合规；保留错误和旧目录，修正输入后换新输出路径 |

缺少依赖且环境有 pip 时使用：

~~~powershell
.\.venv\Scripts\python.exe -m pip install -r requirements.lock
~~~

当前环境的 pip 不存在，但依赖已就绪；不要为此重建 `.venv`。确实缺依赖时，可用已安装的 uv：`uv pip install --python .venv/Scripts/python.exe --cache-dir .cache/uv -r requirements.lock`。

如果换了电脑，或者 `.venv` 确实不存在，需要先安装 Python 3.12，再在代码文件夹运行：

~~~powershell
py -3.12 -m venv .venv
.\.venv\Scripts\python.exe -m pip install -r requirements.lock
~~~

如果 `py` 也找不到，先处理 Python 安装问题，不要照抄后面的命令继续试。**现有环境能用，就不要重建。**

## 6. 第 0 阶段：电脑和 QuPath 的实机检查（现在可先不做）

第 0 阶段检查的是“真实环境是否支持后面的功能”。它和刚才的软件自动测试不同：刚才测试通过，不等于这些实机项目也通过。

### 6.1 先生成一份电脑环境记录

仍在刚才的 PowerShell 窗口运行：

~~~powershell
$envFile = 'local-data/environment-' + (Get-Date -Format 'yyyyMMdd-HHmmss-fff') + '.json'
.\.venv\Scripts\python.exe -m phase0.check_environment --output $envFile
notepad $envFile
~~~

暂时只看：

- `python_supported` 是不是 `true`：表示 Python 版本够用。
- Java、Gradle、Docker 是否被找到：缺少哪个，记录哪个，不需要为跑刚才的例子把它们全部装上。
- `phase0_complete` 仍然是 `false`：这是正常的，因为还有人工操作没做。
- `pending` 的意思是“等待检查”，不是这个检查已经失败，也不是已经通过。

### 6.2 真正进入 QuPath 测试时再做什么

只用专门的测试项目和合成图像，**不要拿真实研究项目做保存、崩溃或恢复实验**。

| 顺序 | 要做的事 | 主要看什么 |
|---|---|---|
| 1 | 打开测试图片 | QuPath 能否正常显示 |
| 2 | 构建、安装最小扩展 | 重启后能否看到测试菜单 |
| 3 | 运行项目提供的检测脚本 | 能否读取数据，是否出现方法不存在等错误 |
| 4 | 在测试项目做保存、重开和故障检查 | 哪些修改保存了，失败后能否确认状态 |
| 5 | 检查文件权限和研究隔离环境 | 其他账号/实验代码能否越权访问 |

这些操作依赖实际 QuPath 安装位置和环境，详细命令及判断条件放在 [测试技术参考](docs/testing-reference.md) 第 3 节。建议需要集成时再逐项做，不用现在一次完成。

尤其注意：装了 Docker 不等于隔离合格。研究隔离检查没通过，就不能运行候选代码；源码预检通过也不能代替这一步。

## 7. 以后每个阶段怎么测（功能做好后再用）

| 阶段 | 到时你需要做的操作 | 什么才算通过 |
|---|---|---|
| Phase 2：QuPath 只读 | 打开测试图、选区域、发起固定分析、查看结果 | 结果与参考一致，原图像对象和标注不被修改 |
| Phase 2R：最小代码生成研究 | 通过隔离检查后，让 Agent 写/改代码，执行并根据反馈再改 | 候选在隔离环境执行；结果有参考可比较；版本和反馈可追溯 |
| Phase 3a：批准后写回 | 先看修改预览，再审查、授权、写回和读回 | 未授权不改；出错能准确记录，不能把不确定情况说成成功 |
| Phase 3b：自然语言入口 | 用不同说法提出需求，检查系统理解的任务 | 理解错误能发现并拒绝；没有许可不外传数据 |
| Phase 3c：正式研究实验 | 用相同任务和合理预算比较不同 Agent 方案 | 保留所有尝试与失败；比较结果误差、时间和成本，不预设多 Agent 更好 |
| Phase 4：新增任务 | 老师确认新问题后增加测试 | 新指标有自己的参考和验证方法，旧测试仍通过 |

这些功能没有全部实现，现在没有可照抄的完整启动命令。不要把本节当成已经可以操作的功能清单。正式的技术验收条件见 [实施路线](docs/implementation-roadmap.md)。

## 8. 测完留一份简单记录

每次记下面几项就够了，可以复制到自己的会议笔记：

~~~text
测试日期：
自动检查：运行了多少项？最后是 OK、跳过，还是失败？
统计例子：是否看到 PASS L2？CD8 是否为 3、60%？
任务管理例子：是否看到 REPORT_READY？
代码版本例子：是否有两版记录？是否仍为 NOT_EXECUTED？
结果文件夹：
遇到的问题：
~~~

最近一次已记录的本地结果（2026-10-19）：Python 3.11.15 和 3.12.13 各 71 项全部通过，无跳过；干净安装后的三个工具与新入口也有独立进程检查。研究候选没有执行，真实 QuPath、研究隔离和远程 GitHub 自动检查仍未完成验收。命令、证据和遗留问题见 [开发交接](docs/gpt-five-hour-handoff.md)。

之前的运行证据可以直接查看，不必为了看成果重新跑：

| 内容 | 已有位置（相对于 `qupath-agent`） |
|---|---|
| 固定统计结果 | `local-data/phase1-r4-regression/` |
| 任务管理记录 | `local-data/harness-r4-regression/` |
| 两版候选和反馈 | `local-data/research-preflight-r4-final-py312/` |

更完整的历史记录见 [测试技术参考](docs/testing-reference.md)；给老师的介绍见 [阶段成果与导师沟通指南](阶段成果与导师沟通指南.md)。

## 9. 本周 ROI 计数与初步性状首版（2026-10-24）

目前已经能运行固定脚本与报告链路，暂未接模型。按本轮选择，真实切片稍后验收；下面步骤为下一次人工测试准备。

### 9.1 准备一个新的本地会话

先在 PowerShell 进入 `E:\XJTLU\大四\FYP\code\qupath-agent`，运行：

~~~powershell
.\.venv\Scripts\python.exe -X utf8 -m qupath_bridge prepare --output local-data/my-first-roi
~~~

本机已预备一个未确认规则的会话：`qupath-agent/local-data/roi-mvp-20261024/user-session/`，也可直接使用。

生成 `config.json` 和 `run_in_qupath.groovy`。后者是项目可信固定脚本，会调用当前 Python 环境；目录已存在则拒绝覆盖。项目移动位置或更换 Python 后重新准备会话，不使用旧启动脚本。

1. 打开 `F:\QuPath-0.7.0\QuPath-0.7.0.exe`，载入测试图像，显式设置为 Fluorescence，并核对像素标定。
2. 人工框选并只选中一个 annotation，初次选小区域：包围框最多 400 万像素、最多 1000 个细胞。
3. 在 **Automate → Show script editor** 打开生成的 `run_in_qupath.groovy`。先核对配置，再用 **Run → Run** 执行，不用 Run for project。

首次配置未确认时程序会停下，并在会话的 `capture-.../context.json` 中保存实际通道名称和顺序。将 `config.json` 的七个 `channels` 与该图像逐个对应；`PDL1 → PD-L1` 是显式别名，`name` 必须写实际完整名称。初始模板来自用户提供的面板；实际文件的 Opal 顺序可能不同，不能直接套用。核对后再把 `panel_confirmed` 改为 `true`；DAPI 本版本仍要求在 0，自发荧光不参与五 marker 的阳性判断。

### 9.2 检测细胞和设定判据

**已有细胞时：** 保持 `detection.method="existing"`。程序只统计真正的 CellObject，不把所有 detection 或 StarDist 核对象当细胞；需要核对检测方法、参数和质量。在确实完成检测后填写 `review_note`，按实际情况设 `completed` 和 `confirmed`。未完成检测的空对象集不会报告零细胞。

**需要由脚本检测时：** 将 `method` 改为 `watershed`，填好 `parameters` 对象并明确 `confirmed=true`、`review_note`；`completed` 保持 false，由本次执行结果决定。参数必须来自对照图／QuPath Cell detection 设置和人工核对，不能照搬合成测试数值：

| parameters 的数值字段 | 含义 |
|---|---|
| requestedPixelSizeMicrons | 分析像素大小；至少为该图像原像素大小，本版不做上采样 |
| backgroundRadiusMicrons、medianRadiusMicrons、sigmaMicrons | 背景、滤波和平滑参数 |
| minAreaMicrons、maxAreaMicrons | 核面积范围；这里的数值单位是 μm² |
| threshold | DAPI 检测阈值，和五 marker 阳性阈值不同 |
| cellExpansionMicrons | 从核向外扩展近似细胞，必须为正；这不是经过验证的真实细胞边界 |

脚本固定使用绑定的 DAPI 通道，保留核、创建细胞对象；有子对象或相交 detection 的 ROI 会拒绝重复检测。检测仅在当前 QuPath 内存中创建对象，异常可能留下部分对象，需要人工检查；不会自动删除、重试或保存项目。

检测完成后需要修改阈值再分析时，将方法切回 `existing`、`parameters` 设为 null，并按实际检测结果填写完成记录；不要再次运行 watershed 覆盖旧对象。

**初步定性规则：** 在 `rules` 中分别核对 CD20、PD-L1、CD8、CD23、PANCK 的 `compartment`（Cell／Nucleus／Cytoplasm／Membrane）、`threshold` 和 `review_note`；仅在人工确认后设 `confirmed=true`。模板 Cell 只便于准备，不表示所有 marker 都应测全细胞。当前只支持 Mean 和 `>=`，保留原始强度单位；不自动套用 Otsu、统一阈值或“信号越亮功能越强”。

阈值留空／未确认时仍可获得细胞数量和测量值，定性为 unknown，报告状态是 `PARTIAL_REVIEW_REQUIRED`。不能把它作为本周完整定性验收通过。

### 9.3 查看结果

每次执行创建新的 `capture-...` 子目录，不覆盖上次结果：

| 文件 | 用途 |
|---|---|
| context.json、config.json、snapshot.json | 捕获目标、实际配置和逐细胞测量；后续改配置需重新捕获 |
| report/report.txt | 中文区域汇总与初步性状说明，也会打印到 QuPath 日志 |
| report/cells.csv | UTF-8 CSV：对象 ID、像素坐标、五 marker 测量／状态及性状标签，可回到原图定位 |
| report/results.json、task.json、harness/ | 完整结果、任务状态、SQLite 审计及内容摘要产物 |
| capture-failed.json、config-check.txt、analysis-log.txt | 拒绝或错误说明；无报告不能当成功，失败后先检查对象，不盲目重跑 |

计数按**细胞轮廓质心**位于 ROI 内或外边界纳入，空洞内部排除，包含嵌套细胞。五 marker 的群体可重叠，阳性数量不能相加充当总细胞数；每个比例以阳性＋阴性为有效分母，同时显示未知数量。

性状说明包含 B 细胞相关、上皮相关、CD8 相关、CD23／PD-L1 表达等线索，逐细胞保留依据。没有 Ki-67 等增殖指标时不推断高增殖。`PRELIMINARY_REVIEW_READY` 表示程序满足已确认规则，仍须人工检查检测质量和生物学解释。

### 9.4 重现本轮合成测试

~~~powershell
.\.venv\Scripts\python.exe -X utf8 -m unittest discover -s tests -v
.\.venv\Scripts\python.exe -X utf8 tests/run_roi_qupath_smoke.py --qupath 'F:\QuPath-0.7.0\QuPath-0.7.0 (console).exe' --output local-data/my-new-roi-smoke
~~~

本轮结果：75 项 Python 测试全部通过；安装版 0.7.0／Java 25.0.2 通过生产固定脚本对合成七通道像素做实际测量，watershed 三核团得到 3 个 CellObject；既有对象场景得到 4 个纳入细胞，边界／嵌套／空洞与已知参考一致。无 ROI、多 ROI、错误通道、泛核对象和重复检测均拒绝，未确认阈值保持 unknown。wheel 已离线构建，在源码目录外以 `-I` 完成 16 项安装包 CLI 检查。

证据保留于 `qupath-agent/local-data/roi-mvp-20261024/`，不发布到公网。Windows 沙箱曾因临时目录 ACL 使测试失败，在正常本机权限下以相同测试命令通过；不要因此修改 Harness 权限或把合成图结果当真实切片准确率。交互 GUI、真实染色、PathPilot、模型及生成代码隔离仍待独立验收。
