# GitHub 版本管理指南

> 2026-10-28 更新：网站维护源统一为 `code/`。`scripts/publish_dashboard.ps1` 使用 `../output/site-publish-checkout/` 作为独立发布检出，并在生成前检查该检出干净且与远端 `main` 一致；不一致时停止并先核对合并。`-PrepareOnly` 将预览生成到 `../output/site-preview/`，不会更改发布检出。下文旧 `public-site/` 路径为历史说明，不能直接用于发布。

适用于 [QuPath Agentic Pathology 设计](frame.md)，面向 Windows PowerShell、个人开发和导师协作。网站部署与维护约定核对日期：2026-10-21；其余操作指南保留原有验证范围。

当前目录已有本地 Git 仓库，主项目的提交与远程同步状态须单独核对。实现位于 `qupath-agent/`，分阶段测试见 [TESTING_GUIDE.md](TESTING_GUIDE.md)。独立网站仓库 `public-site/` 已连接 [KLEIBER404/cancer-harness-dashboard](https://github.com/KLEIBER404/cancer-harness-dashboard)，2026-10-21 首次 Pages 部署成功，[公网主页](https://kleiber404.github.io/cancer-harness-dashboard/)已验证可访问。网站发布不等于主项目源码已提交或推送。下文首次建库命令只用于尚未建立的仓库，已有仓库不得重复初始化；网站维护必须遵守第 7 节。

## 1. 管理哪些内容

Git 保存本地修改历史；GitHub 保存推送后的远程历史并提供 Pull Request（PR）审查。建议采用一个 `main` 主分支加短期任务分支；每完成一项可解释、可验证的改动就提交一次。

| 纳入版本管理 | 保留在受控本地存储 |
|---|---|
| 设计文档、源码、构建配置、依赖锁文件 | API key、token、`.env`、私钥 |
| JSON Schema、指纹规格、合成 fixtures、golden 用例 | 真实病理图像、患者信息、未获许可的数据 |
| 不含敏感信息的版本化 SemanticsProfile | `runtime.json`、WAL、SQLite 数据库、原始日志 |
| 脱敏的实验协议、manifest、复现命令、汇总报告 | 原始 detection 表、完整 LLM 交互、容器输出 |

manifest、类别名、路径、截图和汇总报告也须检查内容。Private 仓库和 Git LFS 都不是数据使用许可；可公开的合成 CSV/JSON 不应被通配规则误排除。

## 2. 第一次建立仓库

### 2.1 检查位置与身份

在本项目目录打开 PowerShell，先执行：

```powershell
Get-Location
git --version
git rev-parse --show-toplevel
```

最后一条若返回已有仓库根目录，先确认路径并跳过初始化；仅在确认目标目录尚无仓库时才执行 `git init`。`public-site/` 是按发布方案明确隔离的独立网站仓库，其余目录不要随意嵌套建库。未安装 Git 时先安装 Git for Windows。

确认在项目根目录后初始化；将姓名和邮箱替换为自己的值，邮箱可使用 GitHub 提供的 noreply 地址：

```powershell
git init -b main
git config user.name "YOUR_NAME"
git config user.email "YOUR_VERIFIED_OR_NOREPLY_EMAIL"
```

这里的身份设置只作用于当前仓库。提交作者信息会进入历史，提交前核对邮箱。

### 2.2 先配置忽略，再提交

根目录已有 `.gitignore`，覆盖当前实现的虚拟环境、缓存和 `qupath-agent/local-data/`；先保留现有规则，再按以下示例补充实际新增的数据目录：

```gitignore
.env
.env.*
!.env.example
*.pem
*.key
runtime.json
.qupath-agent/
/data/
/local-data/
/artifacts/
/experiment-outputs/
*.sqlite
*.sqlite-*
*.db
*.db-*
*.log
.venv/
__pycache__/
*.py[cod]
.pytest_cache/
.gradle/
build/
dist/
.idea/
.DS_Store
Thumbs.db
```

将真实数据、实验运行输出统一放在仓库外或这些明确排除的目录中；合成 fixtures 放在 `contracts/` 等受版本管理目录。`.env.example` 只能包含空值或占位符。不要忽略所有 CSV/JSON、所有 JAR 或整个 `experiments/`，否则会漏掉测试输入、Gradle wrapper 或实验协议。

只暂存确认要提交的文件，然后查看暂存内容：

```powershell
git status --short
git add -- frame.md cancer-harness-architecture.md docs GITHUB_VERSION_CONTROL.md TESTING_GUIDE.md .gitignore qupath-agent .github/workflows/phase1.yml
git diff --cached --check
git diff --cached
git commit -m "docs: establish architecture and version control guide"
```

`.gitignore` 对已经被跟踪的文件不起作用；以后需要停止跟踪时使用 `git rm --cached -- PATH`，本地文件保留，但旧提交中的内容仍在。[Git 官方忽略规则说明](https://git-scm.com/docs/gitignore)

### 2.3 连接 GitHub 并首次推送

在 GitHub 创建项目仓库，初始建议 Private；创建时不要预置 README、许可证或 `.gitignore`，避免与本地首次提交产生两套历史。复制真实远程地址并替换下列占位值：

```powershell
git remote add origin https://github.com/YOUR_ACCOUNT/YOUR_REPOSITORY.git
git remote -v
git push -u origin main
```

若已经存在 origin，先用 `git remote -v` 核对；确需更换时用 `git remote set-url origin NEW_URL`。若远程已有有效提交，应先 clone 到另一个目录再复制需要的本地文件，不能通过强制推送覆盖远程历史。[GitHub 导入本地仓库指南](https://docs.github.com/en/migrations/importing-source-code/using-the-command-line-to-import-source-code/adding-locally-hosted-code-to-github)

HTTPS 使用 Git Credential Manager、GitHub CLI 登录或 personal access token；SSH 使用已添加至 GitHub 的 SSH key。GitHub 账户密码不能用于 Git HTTPS 认证，凭据不要写进远程 URL、文档或脚本。[GitHub 身份认证说明](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/about-authentication-to-github)

## 3. 日常修改：分支 → 提交 → PR

工作区干净时从最新主分支开始一项任务：

```powershell
git status --short
git switch main
git pull --ff-only origin main
git switch -c docs/refine-writeback-contract
```

分支名称按任务取名，例如 `feat/detection-upload`、`fix/reconcile-race`、`docs/github-guide`。无需额外长期维护 `develop` 分支。

修改后检查差异，运行实际存在且相关的检查，再提交指定文件：

```powershell
git diff --check
git diff
git add -- frame.md
git diff --cached
git commit -m "docs: clarify writeback recovery contract"
git push -u origin docs/refine-writeback-contract
```

提交信息建议采用 `docs:`、`feat:`、`fix:`、`test:`、`refactor:`、`chore:`，后接具体行为。不要把无关功能、依赖升级和文档改写混成一个提交。

在 GitHub 创建 PR，目标为 `main`，写明问题、改动、验证结果以及尚未验证的事项。涉及科学语义时附 S 编号、profile 版本与确认记录。单人项目可以自行检查 PR；需要导师审查的决策由真实审查者确认。[GitHub flow 工作流](https://docs.github.com/en/get-started/using-github/github-flow)

合并方式优先选择 **Create a merge commit**，便于保留提交关系。合并后回本地：

```powershell
git switch main
git pull --ff-only origin main
git branch -d docs/refine-writeback-contract
```

`-d` 会拒绝删除 Git 无法证明已合并的分支。若使用了 squash/rebase merge，先核对 GitHub 的合并结果和本地差异，可以暂时保留分支，不要直接改用 `-D`。

### 同步分支与解决冲突

在任务分支且工作区干净时，将主分支的新内容合入：

```powershell
git fetch origin
git merge origin/main
```

有冲突时执行 `git status` 查看文件，逐处编辑 `<<<<<<<`、`=======`、`>>>>>>>` 之间的内容，保留正确结果并删除标记；不要整文件盲选某一方。修好后检查、暂存并完成合并：

```powershell
git diff --check
git add -- frame.md
git commit
git push
```

文件名按实际冲突替换；所有冲突都解决后才能提交。不继续本次合并时用 `git merge --abort`；合并前保持工作区干净，避免已有修改难以恢复。[Git merge 官方文档](https://git-scm.com/docs/git-merge)

## 4. 撤销与回退

先用 `git status` 和 `git diff` 确认目标，`PATH` 与 `COMMIT_SHA` 均须替换为真实值。

| 情况 | 操作 | 对工作文件/历史的影响 |
|---|---|---|
| 文件误暂存，已有至少一次提交 | `git restore --staged -- PATH` | 取消暂存，保留编辑 |
| 首次提交前误暂存新文件 | `git rm --cached -- PATH` | 从索引移除，保留本地文件 |
| 要放弃一个已跟踪文件的未暂存编辑 | `git restore -- PATH` | 用暂存区版本覆盖工作文件，未暂存修改会丢失；执行前先保存需要的内容 |
| 已共享的普通提交有误 | `git revert COMMIT_SHA` | 新增反向提交，保留历史；随后测试、推送和 PR |
| 想查看历史版本 | `git show COMMIT_SHA:frame.md` | 只读，不改变工作区 |

对已共享提交优先 revert，不把 `reset --hard`、`clean -fd` 或 force push 当作日常恢复步骤。merge commit 的 revert 需要选择主线，不能直接照搬普通提交命令；可先查看 `git show --no-patch --pretty=raw COMMIT_SHA` 的父提交关系。[Git revert 官方文档](https://git-scm.com/docs/git-revert)

若误提交密钥，先撤销或轮换凭据；仅删除文件、加入 `.gitignore` 或 revert 不会清除历史。已推送敏感数据时按实际权限和数据要求协调历史清理与其他副本处理。[GitHub 敏感数据移除指南](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/removing-sensitive-data-from-a-repository)

## 5. 里程碑、标签与实验复现

文档修订号、协议版本和软件版本分别管理：`frame.md` 的“修订 3”不意味着软件已发布 v0.3。首次设计冻结可用 `design-v0.3-r3`；实现达到出口标准后再使用软件版本标签。

在最新 `main`、工作区干净且已完成对应检查时创建带注释的标签：

```powershell
git switch main
git pull --ff-only origin main
git status --short
git rev-parse HEAD
git tag -a design-v0.3-r3 -m "Freeze architecture draft v0.3 revision 3"
git push origin design-v0.3-r3
```

已经发布的标签不移动、不复用；修改后创建新标签。GitHub Release 可选择该标签并附变更说明、验证范围和允许公开的产物，不上传真实数据、运行日志或含凭据的配置。

正式实验另建 `experiment-<protocol-version>` 标签，并在 manifest/报告记录：源码 commit、协议版本与摘要、Schema/profile/输入/提示模板摘要、依赖锁文件、容器 digest、模型标识、运行参数、seed（如支持）、命令和结果存放位置。标签只固定源码，不会固定远程模型或未入库数据；这些需要独立记录和按许可保存。

已冻结用例或 SemanticsProfile 不原地覆盖。科学规则修改后递增 profile 版本，保留旧版和变更原因，重新冻结实验协议；失败实验也保留原记录。

## 6. 主分支保护与检查

仓库支持时，为 `main` 配置分支保护或 ruleset：要求 PR、禁止强制推送和删除，并要求已实际运行的必要检查。单人项目不必设置无人能够完成的强制审查人数；有人参与时再要求至少一名审查者。

GitHub Free 对公开仓库提供 protected branches；私有仓库是否可用取决于账户或组织方案。以仓库设置中的可用能力为准，不为设置保护而擅自公开研究项目。[GitHub 分支保护说明](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches)

当前已有 `.github/workflows/phase1.yml`，运行 Python 3.11/3.12 与 Windows/Linux 矩阵、JDK 21 指纹互验、共享契约/离线测试及 CLI 合成用例；首次推送后确认 Actions 实际通过，再选择对应 required check。Phase 0 的 QuPath GUI、保存/崩溃和隔离不由该 CI 证明。正式实验和 GitHub Actions 都默认使用合成数据，真实数据与生产密钥不进入 PR 检查。

每次提交前确认：改动只包含本次任务；没有敏感信息和运行产物；文档与实际状态相符；完成了相关检查；待确认的科学问题仍明确标为待确认。

## 7. 网站同步是每次变更的强制交付条件

**强制规则：项目功能、架构、验证结果、阶段进度，以及网页、文档或发布配置发生新增、删除、修改、重命名或回退后，负责的开发者或 Agent 必须在同一次交付中同步网站。不得等用户再次提醒，也不得把网页维护留作下一次任务。** 架构中的同等要求见 [frame.md §17.6](frame.md#176-增删改后的强制同步与交付条件)。

1. 先完成源码与原始文档，清理被删除或改名内容的入口、正文、目录、单份下载和 ZIP 下载；根目录 Markdown 是文档页的来源，不直接手改其生成快照。`public-site/` 仅存生成副本，不在其中手写文档。
2. 同步根目录 `项目进度看板.html` 的已有能力、缺项、下一步、证据、实际核对日期和最近活动；涉及能力变化时更新动画说明与终点。若只改网站或文档，记录这次维护，不提升研究阶段或延伸能力边界。
3. 刷新并检查文档与发布副本，按改动运行相关检查；只有本地生成通过不能算网站更新完成。

```powershell
python scripts/update_dashboard_docs.py
python scripts/update_dashboard_docs.py --check
python scripts/update_code_structure.py
python scripts/update_code_structure.py --check
python scripts/build_public_site.py
python scripts/build_public_site.py --check
.\scripts\publish_dashboard.ps1
```

4. 远程与授权可用时必须运行发布脚本，推送独立网站仓库的 `main`。核对[部署工作流](https://github.com/KLEIBER404/cancer-harness-dashboard/actions)中**本次提交**部署成功，并实际请求[公网主页](https://kleiber404.github.io/cancer-harness-dashboard/)、文档页和受影响的入口；确认线上新增/修改已出现、删除内容不再被列出或下载。交付时记录网站提交、部署链接、核对时间与访问结果。`git push` 成功不能代替部署成功或线上内容核对。
5. 源码项目与网站仓库分别提交、分别检查。禁止用源码工程根目录作发布源，禁止上传切片、患者信息、本地运行数据或凭据。文档删除或改名时，生成器清理对应的旧 Markdown 副本，发布脚本必须同时提交删除，防止旧文件继续留在线上。

仅在网络、授权或部署故障等实际阻碍下可留下待发布状态：先完成本地同步与检查，在看板和交付中写清失败步骤、已完成范围及恢复命令，不得报告已完成本次线上更新。阻碍消除后仍须补齐发布及验证。`-Watch` 只在进程运行时辅助同步，不替代开发者或 Agent 的这项交付责任。

代码结构预览也属于这项强制同步范围。新增、删除或修改源码文件、类/函数、导入或依赖后，必须刷新 `代码结构预览.html`，同时核对模块职责、依赖图和调用路线；新增源码目录要更新生成器的明确目录名单。发布脚本会自动运行结构生成器，`-Watch` 同时检查源码内容摘要。结构生成通过不表示执行、隔离或模型验收通过。维护规则见 [frame.md §17.7](frame.md#177-独立代码结构预览)。
