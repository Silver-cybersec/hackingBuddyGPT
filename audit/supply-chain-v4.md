# 供应链补查 v4

基准 SHA：`e23d918592556f71b8e3b2ee152449e82fa14794`（上游 https://github.com/ipa-lab/hackingBuddyGPT ，2026-10-24 23:28:18 香港时间）。只读。未安装依赖，未拉镜像，未跑仓库脚本或 CI。长表已在 `audit/supply-chain.md` 的不重复展开。下表每个字只表示该项结论，不表示整条供应链已证明安全。

| 项 | 结论 |
|---|---|
| 1 清单与 lock，含间接依赖 | 未发现问题 |
| 2 版本钉、校验和、git／URL 直装 | 已确认 |
| 3 仿冒名、安装钩子、非官方源 | 未核实 |
| 4 镜像来源、tag、digest | 已确认 |
| 5 子模块、下载脚本、运行时远程工具 | 已确认 |
| 6 CI：SHA 钉、密钥、危险触发 | 已确认 |
| 7 恶意包库／已知漏洞 | 已确认 |
| 8 主要依赖维护状态 | 已确认 |

## 1. 清单与 lock（未发现问题）

方法：解析基准树上的 `uv.lock` 与 `pyproject.toml`。有 lock。`uv.lock:1-3` 为 `version = 1`、`revision = 3`、`requires-python >=3.13`。包条目 136 个，其中 135 个是第三方，1 个是本项目 `hackingbuddygpt`（`uv.lock:861-863`，`source = { editable = "." }`）。直接依赖、optional extra（testing／dev／rag）和它们的间接依赖都在这 135 个名字里。没有第二种语言的 lock；也没有 `requirements.txt`。`pip install -e .` 与镜像构建不消费这份 lock，记在第 2 项，不把清单本身写成缺失。

## 2. 版本钉、校验和、直装（已确认）

方法：读声明与 lock 的 `source`／`hash` 字段，未下载文件核对哈希。

- 已确认：`pyproject.toml:24-38` 的直接依赖没有上界，只有名字。唯一下限是 `tenacity>=9.1.4`（`pyproject.toml:39`）。构建依赖是范围 `uv_build>=0.8.16,<0.9.0`（`pyproject.toml:49`）。
- 已确认：`Dockerfile:5` 与 `.github/workflows/python-app.yml:28-31` 执行 `pip install -e .`，不读 `uv.lock`。这两条路径的实际版本未核实。
- lock 内：第三方条目的源是 `https://pypi.org/simple`（示例 `uv.lock:12`），sdist／wheel 带 `sha256`（示例 `uv.lock:13`）。解析全部 135 个第三方条目都有哈希。没有 `git`、path 或任意 URL 直装；唯一非 registry 源是本项目 editable（`uv.lock:863`）。哈希未下载复核，匹配与否未核实。

## 3. 仿冒名、钩子、非官方源（未核实）

方法：对照 lock 内 135 个发行名；对不常见名字查 PyPI 项目页；读本仓库 `pyproject.toml`。查询时间 2026-11-26 11:32 香港时间。

本仓库侧未看到非官方 index（`pyproject.toml` 无 `[[tool.uv.index]]`）、无 `setup.py`、无 postinstall。`[project.scripts]`（`pyproject.toml:82` 起）只注册控制台入口。不常见名的 PyPI 页指向既有上游，而不是邻近拼写：`hf-xet` → huggingface/xet-core，`langchain-protocol`／`langchain-classic` → langchain-ai，`sspilib`／`smbprotocol` → jborean93，`fastuuid`、`pypika`、`durationpy`、`mmh3`、`pybase64`、`uuid-utils`、`overrides` 的 Homepage 均为各自既有仓库。查询 URL 形如 `https://pypi.org/pypi/<name>/json`。

整项仍是未核实：没有打开这 135 个包的 setup／构建脚本，不能判断它们自己的安装钩子。未下载 wheel。

## 4. 镜像（已确认）

方法：读 Dockerfile 与工作流，未访问注册表、未拉层。

- 已确认：`Dockerfile:1` 为 `python:3.13-slim`，无 digest，无 `USER`。来源是镜像名上的官方库标签，不是 digest。
- 已确认：`.github/workflows/publish-docker.yml:36-40` 再推 `:<时间戳>` 和 `:latest`，无 digest。层内容未核实。
- 工作流 `runs-on: ubuntu-latest`（`python-app.yml:18`，`publish-docker.yml:7`，`dependency-review.yml:28`，`jekyll-gh-pages.yml:27`）同样是浮动运行器标签，无 digest。

## 5. 子模块与远程拉取（已确认）

方法：在基准树查找 `.gitmodules`、`curl`、`wget`、运行时下载。无 `.gitmodules`。`publish-docker.yml:12` 写了 `submodules: recursive`，没有子模块可拉。

- 已确认：`src/hackingBuddyGPT/usecases/call_usecase_from_usecase.py:36` 在 SSH 目标上 `wget` GitHub `releases/latest` 的 `lse.sh`，接着 `chmod 700` 并执行。无版本号、无校验和。本单未下载该脚本。
- 仓库安装文档和 CI 里没有 `curl|bash`。没有 `ollama pull` 或模型权重 URL。
- 运行时还会把提示词发给 `litellm`（`src/hackingBuddyGPT/utils/llm.py:138`），RAG 路径会调用未写 `base_url` 的嵌入接口（`src/hackingBuddyGPT/utils/rag.py:28`）。具体主机未在本轮重查，标未核实。这不是安装期下载。

## 6. CI（已确认）

方法：通读 `.github/workflows/` 下四个文件。仓库内没有 `dependabot.yml`。GitHub 网站上的 Dependabot 开关未读，标未核实。

- 已确认：动作都是浮动主版本标签，没有 commit SHA。`actions/checkout@v4`（`python-app.yml:21`，`publish-docker.yml:10`，`dependency-review.yml:31`，`jekyll-gh-pages.yml:30`）、`actions/setup-python@v3`（`python-app.yml:23`）、`actions/dependency-review-action@v4`（`dependency-review.yml:33`）、`docker/login-action@v3`（`publish-docker.yml:15`）、`actions/configure-pages@v4`（`jekyll-gh-pages.yml:32`）、`actions/jekyll-build-pages@v1`（`jekyll-gh-pages.yml:34`）、`actions/upload-pages-artifact@v3`（`jekyll-gh-pages.yml:39`）、`actions/deploy-pages@v4`（`jekyll-gh-pages.yml:51`）。这些标签当时指向的提交未核实。
- 四个文件都没有 `pull_request_target` 或 `workflow_run`。触发是 `push`、`pull_request`（`python-app.yml:6-10`，`dependency-review.yml:12-13`）和 Pages 的 `workflow_dispatch`（`jekyll-gh-pages.yml:10`）。
- 密钥：文件里没有令牌字面量。唯一引用是 `secrets.GITHUB_TOKEN`（`publish-docker.yml:19`），用作 GHCR 登录，没有写进 `echo`。`python-app.yml:12-13` 只给 `contents: read`。`dependency-review.yml:21-24` 另有 `pull-requests: write`。`jekyll-gh-pages.yml:13-16` 有 `pages: write` 与 `id-token: write`。`publish-docker.yml` 没有 `permissions` 字段，令牌默认权限取决于仓库设置，本单未读该设置，标未核实。

## 7. 漏洞库与恶意包库

方法：沿用 2026-11-26 10:28 香港时间对锁内 135 个包的 OSV `https://api.osv.dev/v1/querybatch`（带版本，ecosystem `PyPI`）。详情当时取自 `https://api.osv.dev/v1/vulns/<id>`。本轮不重复请求。PYSEC 编号是同一次结果里的别名，下面只列 GHSA。调用路径是否可被利用，除公告区间外均未核实。

### 依赖是否恶意（未发现问题）

该次 `querybatch` 的返回编号只有 GHSA 与 PYSEC，没有 `MAL-`。这只说明这次查询没有恶意包条目。未读包源码，未下载制品，不能写成已经证明无恶意代码。

### 已知漏洞（已确认）

锁内版本落在下列区间。影响版本按公告 `affected.ranges`。

| 锁内包 | 版本 | 编号 | 区间 |
|---|---|---|---|
| certifi | 2026.7.22 | GHSA-6q48-6c7w-9gw2 | 2019.03.09 起，修复 2026.10.18 |
| litellm | 1.97.0 | GHSA-7hp6-4w63-5g45 | 1.91.0 起，1.100.4 修复；公告另有更高版本段 |
| aiohttp | 3.14.3 | GHSA-24c4-3p7v-29cm、GHSA-28qf-cpwq-vmc2、GHSA-2g87-pp6c-29x5、GHSA-3rp8-rxj8-mvjw、GHSA-494j-54xm-rwp3、GHSA-5pv8-vvhc-h7jv、GHSA-5rmc-48h2-hvcm、GHSA-5wr8-gcqf-gqf3、GHSA-6cjc-53gc-5rqm、GHSA-9xw5-w5jj-pvcg、GHSA-f9xw-jh8m-3mwq、GHSA-gx76-c99x-2c96、GHSA-h4cc-hm5h-fmm9、GHSA-jm3w-gw2p-xjfj、GHSA-pqxq-w44r-73xw、GHSA-pw86-h8fm-j323、GHSA-xgw8-8ccx-rmqh | 均在 3.14.4 修复；GHSA-9xw5-w5jj-pvcg 从 2.0.0 起，其余从 0 起 |
| multidict | 6.7.1 | GHSA-54p9-h82j-f925 | 6.7.0 起，6.9.1 修复 |
| filelock | 3.32.3 | GHSA-567q-5qvw-pf92、GHSA-6627-m3xw-93mp、GHSA-69hj-f68j-2879、GHSA-8mx5-7m5m-r77g、GHSA-9c4x-gm8v-wgc4、GHSA-9vhv-jxww-xfm5、GHSA-cwqr-hh33-gh44、GHSA-fq56-3cx3-fjxp、GHSA-j8f7-rjxc-mr56、GHSA-m462-7rxw-wvpg、GHSA-rfpc-qcw5-3pc4、GHSA-v4x8-3wr5-3wq8 | 修复版本 4.2.1，GHSA-j8f7-rjxc-mr56 为 4.0.9；introduced 各条不同，锁内 3.32.3 落在区间内 |
| chromadb | 1.5.9 | GHSA-2wm9-hf6c-p5cr、GHSA-36p7-vc44-83pf、GHSA-f4j7-r4q5-qw2c、GHSA-xph7-9rjv-w5fr | `last_affected` 含 1.5.9。仅 rag extra（`pyproject.toml:73-80`） |
| oauthlib | 4.0.0 | GHSA-3jp2-3mrg-2vxc | 3.3.0 起，4.0.1 修复。随 chromadb |
| opentelemetry-api | 1.44.0 | GHSA-mf5v-w3w3-f6cg、GHSA-xcq7-p3pf-p88h | 修复于 1.45.2、1.45.1。随 chromadb |
| soupsieve | 2.9.2 | GHSA-4j63-6c4x-wfg4、GHSA-6655-6qjp-jx7x、GHSA-7g44-g9rc-x53w、GHSA-mmhr-3c5m-f64f | 修复版本分别为 2.10.0、3.0.2、3.0.2、4.0.0 |

其余 126 个锁内包在同一次查询中没有漏洞条目。

## 8. 维护状态（已确认）

方法：2026-11-26 11:32 香港时间读取 `https://pypi.org/pypi/<name>/json` 与对应 `https://api.github.com/repos/<owner>/<repo>`。抽查锁内版本的 `yanked`。未覆盖全部 135 个包的归档状态。

- 已确认：默认直接依赖 `pypsexec`（`pyproject.toml:35`，锁内 0.3.0）在 PyPI 的最近上传是 2021-10-21 20:30:45Z，即 2021-10-22 04:30 香港时间。`https://pypi.org/pypi/pypsexec/json` 的 latest 仍是 0.3.0。`https://api.github.com/repos/jborean93/pypsexec` 的 `archived` 为 false，最近 push 为 2025-10-17 13:40 香港时间。没有弃用分类，也没有新发版。
- 已确认：rag extra 的 `langchain-community`（`pyproject.toml:75`，锁内 0.4.2）上游 `https://api.github.com/repos/langchain-ai/langchain-community` 的 `archived` 为 true。最近 push 2026-06-20 06:21 香港时间。PyPI latest 仍是 0.4.2，上传 2026-05-22 19:42:57Z（2026-05-23 03:42 香港时间）。
- 同一次抽查里，锁内版本 `yanked` 均为 false：fabric 3.2.3、asyncssh 2.24.0、mako 1.4.3、requests 2.34.2、rich 15.0.0、litellm 1.97.0、pyyaml 6.0.3、python-dotenv 1.2.3、pypsexec 0.3.0、pydantic 2.13.4、beautifulsoup4 4.15.0、faker 40.36.0、tenacity 9.1.4、chromadb 1.5.9、langchain-core 1.6.0、langchain-community 0.4.2、aiohttp 3.14.3、certifi 2026.7.22、httpx 0.28.1、openai 2.54.0。其余包是否被 yank，未核实。
- 这些名字里，除上面两条外，PyPI 在 2026 年仍有发版或 GitHub `archived=false` 且 2026 年仍有 push：pydantic 仓 `pydantic/pydantic` 未归档（2026-11-26 02:20 香港时间仍有 push）；litellm latest 1.111.0 上传于 2026-11-25 09:41 香港时间；certifi latest 2026.11.20。httpx 锁内即 latest 0.28.1，上传 2024-12-06 23:37 香港时间，仓库 `encode/httpx` 未归档。beautifulsoup4 的项目页是 `https://www.crummy.com/software/BeautifulSoup/bs4/`，不是 GitHub；PyPI 最近上传 2026-06-07 16:44:20Z（2026-06-08 00:44 香港时间），分类为 Production/Stable。其源码仓是否归档，未核实。
