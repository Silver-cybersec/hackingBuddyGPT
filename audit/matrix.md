# 验收矩阵

审计对象：上游 https://github.com/ipa-lab/hackingBuddyGPT ，基准 `e23d918592556f71b8e3b2ee152449e82fa14794`（提交时间 2026-10-24 23:28:18 香港时间）。本环境检出的 HEAD 与该 SHA 一致。查询 OSV 的时间是 2026-11-26 10:28 香港时间。未运行产品代码。

| ID | 需求 | 检查项 | 证据 | 结论 | 未覆盖 |
|---|---|---|---|---|---|
| R1 | 对用户指定的本仓、本单固定基准 SHA 做只读静态重审（09:07 原话） | 记录上游 URL、默认分支、基准 SHA、提交时间（香港时间）；只读源码、清单与锁文件 | 本环境 `git rev-parse HEAD` = `e23d918592556f71b8e3b2ee152449e82fa14794`；`git log -1` 作者 Andreas Happe，AuthorDate 2026-10-24 17:28:18 +0200（即 15:28:18Z，香港时间 23:28:18）。fork `Silver-cybersec/hackingBuddyGPT` 的 `main` 与该 SHA 相同 | 已按该树阅读。默认分支 `main`。来源日期：基准 SHA 由任务书记载为 2026-11-26 09:29 香港时间的 `git ls-remote`／GitHub API | 无。SHA 已检出 |
| R2 | 代码安全 | 故意漏洞与对部署者／宿主机的真实风险分开；危险执行、默认凭据、硬编码密钥、数据外传、隔离、暴露面、容器／宿主挂载；高／中／低 | `audit/findings.json` 的 HB-01–HB-12；报告第 3 节。链接均指向该 SHA | 高 2、中 7、低 3。仓库内没有独立靶场关卡源码；危险行为是代理能力。真实风险见 HB-01–HB-12 | 未动态利用、未对真实目标发包、未确认 Fabric/paramiko 的默认主机密钥策略、未确认 pypsexec 是否残留服务 |
| R3 | 架构（含与学习目标相称的功能阅读） | 主要组件、信任边界、数据与权限流；结合 README 做相称阅读，不做完整宣称对照全表（全表见 F1，总调度附注已启用） | `src/hackingBuddyGPT/cli/wintermute.py`、`usecases/`、`utils/connectors/`、`utils/llm.py`、`utils/limits.py` | 单一 CLI 进程：配置 → 用例循环 → litellm；执行面是 SSH、PSExec 或本机 HTTP 客户端。无入站监听 | `docs.hackingbuddy.ai` 正文未抓取。`utils/prompt_generation/information/` 的大段测试步骤按文案阅读，未逐段当漏洞条目 |
| R4 | 供应链，且必须单列「依赖是否恶意」 | 包与锁、镜像、子模块、curl/wget、版本与校验和、仿冒名、安装钩子、非官方源、OSV／GHSA／恶意包库 | `audit/supply-chain.md`；`uv.lock` 135 个 PyPI 包；OSV `querybatch` 2026-11-26 10:28 香港时间 | 未见恶意包编号（无 MAL-）。不能写成已证明无恶意。默认树有 certifi 与 litellm 的已知公告，可利用性未按调用路径证实 | 未下载 wheel 复核哈希；未拉镜像层；未把 Actions tag 解析到 SHA；未打开依赖库源码核对遥测默认值；`pip install -e .` 不走锁文件时的解析版本未核实 |
| R5 | 产出精练中文报告 | 四个交付文件；每节先结论 | `audit/RERUN-20261126.md`、`audit/matrix.md`、`audit/findings.json`、`audit/supply-chain.md` | 已写。功能与编译两章在主报告末尾，并回写本表 F1、D1 | 无模板要求而写不出的附件 |
| R6 | 查是否存在可对照的旧仓库安全审计基准 | 任务书写明本课题没有已完成的仓库安全审计基准 | 任务书「与旧基准的差异」：总报告 §2.22 当时列为未审计 | 无旧仓库安全审计基准。不编造差异章 | 无。这不是缺口 |
| R7 | 不新建私库、不改产品代码、不把静态阅读写成已运行验证 | 交付物只在 `audit/`；表述带未实测 | 本 PR 只新增 `audit/` 下四个文件 | 未改产品代码，未新建仓库，未合并。功能与部署两章每条标未实测 | 无 |
| F1 | 功能与使用（总调度附注：用户 2026-11-26 10:14 香港时间同意，覆盖「功能不作为独立模块」） | README／仓内文档宣称对照实现；入口、配置、一条典型工作流 | 主报告「功能与使用（未实测）」；实现位置为该 SHA 的文件与行号 | 入口、十个用例名、限额、日志工具与代码一致。`--conn=ssh`／`--conn=local_shell`、`llm.api_base` 能切换端点、策略类默认 100 轮，与代码不符或部分相符。行为未实测 | 官网文档未读；未执行 `wintermute` |
| D1 | 编译与部署要求（同一附注启用） | Dockerfile、CI、包管理器、语言与外部服务、硬件、外网、文档与代码不符 | `Dockerfile`、`pyproject.toml`、`.python-version`、`.github/workflows/`、`uv.lock` | 推荐路径是 Python ≥3.13 加 `uv sync`。另有未使用锁文件的 pip 与 Docker 路径。无 compose、无 Makefile、无数据库。文档未写 GPU／内存。全部未实测，不能称为已验证可部署 | 未构建镜像、未 `uv sync`、未跑 CI、未访问 GHCR 与 Docker Hub 层 |
