# 供应链

基准：https://github.com/ipa-lab/hackingBuddyGPT @ `e23d918592556f71b8e3b2ee152449e82fa14794`（2026-10-24 23:28:18 香港时间）。

结论：锁文件把 135 个第三方包固定在 PyPI，并写了 sha256；本单没有发现恶意包编号，也没有发现仿冒包名、额外索引或安装期下载执行。同一份锁里有已知漏洞公告。镜像和 GitHub Actions 只用浮动标签。未下载制品、未拉镜像，不能写成依赖或镜像已确认安全，也不能写成已确认无恶意代码。

## 依赖是否恶意

结论：只读查询下没有恶意投毒证据；这不等于已经证明依赖干净。

- 2026-11-26 10:28 香港时间（UTC 2026-11-26T02:28:23Z）对 `uv.lock` 里除本项目 editable 项以外的 135 个包调用 OSV `https://api.osv.dev/v1/querybatch`（ecosystem `PyPI`，带锁内版本）。返回的编号是 GHSA 与 PYSEC，没有 `MAL-` 编号。
- 包来源只有 `https://pypi.org/simple`。本项目自身是 `{ editable = "." }`。没有 git 源、path 源或自建 index。`pyproject.toml` 没有 `[[tool.uv.index]]`。
- 直接依赖名与常见正牌发行名一致：`fabric`、`asyncssh`、`Mako`（锁内名 `mako`）、`requests`、`rich`、`litellm`、`PyYAML`（`pyyaml`）、`python-dotenv`、`pypsexec`、`pydantic`、`BeautifulSoup4`（`beautifulsoup4`）、`faker`、`tenacity`。未见把这些名字换成邻近拼写的依赖。
- 安装钩子：没有 `setup.py`，构建后端是 `uv_build`。`[project.scripts]` 只注册控制台入口 `wintermute`、`hackingBuddyGPT`、`hackingbuddygpt-log-view`、`hackingbuddygpt-log-analyze`，安装时不执行用例。仓库安装说明里没有 `curl|sh` 或 `wget|sh`。
- 运行期远程脚本只有 `ExPrivEscLinuxLSE`：在 SSH 目标上下载并执行 `lse.sh` 的 `releases/latest`，无校验和。见 `src/hackingBuddyGPT/usecases/call_usecase_from_usecase.py:36`。这不是安装钩子。本单未下载该文件。
- 未阅读这 135 个包的源码，未下载 wheel 与锁内 sha256 对照，未检查维护者账号是否被接管。因此不写「已确认无恶意」。

## 锁文件与直接依赖

`uv.lock`：`version = 1`，`revision = 3`，`requires-python = ">=3.13"`。135 个第三方包的 sdist 与 wheel 条目都带 `sha256`（本环境解析锁文件统计，未下载文件）。

| 依赖 | 锁内版本 | 组别 |
|---|---|---|
| asyncssh | 2.24.0 | 默认 |
| beautifulsoup4 | 4.15.0 | 默认 |
| fabric | 3.2.3 | 默认 |
| faker | 40.36.0 | 默认，测试 extra 也列 |
| litellm | 1.97.0 | 默认 |
| mako | 1.4.3 | 默认 |
| pydantic | 2.13.4 | 默认 |
| pypsexec | 0.3.0 | 默认 |
| python-dotenv | 1.2.3 | 默认 |
| pyyaml | 6.0.3 | 默认 |
| requests | 2.34.2 | 默认 |
| rich | 15.0.0 | 默认 |
| tenacity | 9.1.4 | 默认（`pyproject.toml` 要求 `>=9.1.4`） |
| ruff | 0.16.4 | dev |
| pytest / pytest-mock | 9.1.1 / 3.15.1 | testing |
| langchain-core / community / chroma / openai | 1.6.0 / 0.4.2 / 1.1.0 / 1.6.0 | rag |
| chromadb | 1.5.9 | rag |
| markdown | 3.10.3 | rag |

默认树里和出站相关的传递包：`httpx` 0.28.1、`openai` 2.54.0、`certifi` 2026.7.22、`urllib3` 2.8.0、`aiohttp` 3.14.3（`litellm` 直接依赖）、`tokenizers` 0.23.1 → `huggingface-hub` 1.28.0 → `filelock` 3.32.3。`chromadb` 1.5.9 再拉 `kubernetes` 36.0.3，只在 rag extra。

`Dockerfile` 与 `.github/workflows/python-app.yml` 使用 `pip install -e .`，不读 `uv.lock`。这两条路径装到的版本本单未解析，未核实。

## 已知漏洞（漏洞公告，不是恶意包结论）

OSV 命中 9 个包名。下面按与本仓库调用方式的关系压缩，PYSEC 与 GHSA 的重复编号不逐条展开。版本区间来自公告的 `affected.ranges`，查询时间同上。

| 包与锁内版本 | 公告（代表） | 区间 | 和本仓库的关系 |
|---|---|---|---|
| certifi 2026.7.22 | GHSA-6q48-6c7w-9gw2、PYSEC-2026-4320 | 2019.03.09 起，修复 2026.10.18。摘要：捆绑的 emSign 根所涉中间证书密钥受损 | 经 `requests`、`httpx` 进入默认安装。影响 HTTPS 信任库。未替换、未实测 TLS |
| litellm 1.97.0 | GHSA-7hp6-4w63-5g45 | 1.91.0 起，1.100.4 修复（另有更高版本段）。摘要：跨域重用 salt，可把权限提到 Proxy 管理员。CVSS 3.1 向量含 C:H/I:H/A:H | 本仓库只调用 `litellm.completion`（`utils/llm.py:138`），没有启动 LiteLLM Proxy。公告场景与这条调用是否重合，未核实 |
| aiohttp 3.14.3 | 17 个 GHSA，均在 3.14.4 修复；另有 PYSEC 别名。内容主要是解析、WebSocket、表单与 Cookie 的资源耗尽或请求走私 | `< 3.14.4` | `litellm` 的直接依赖。本仓库 Python 源码没有 `import aiohttp`，也没有自建 aiohttp 服务。可利用性未核实 |
| multidict 6.7.1 | GHSA-54p9-h82j-f925 | 6.7.0–6.9.1 之前。引用泄漏 | 随 aiohttp。未核实调用 |
| filelock 3.32.3 | 12 个 GHSA，本地锁文件竞争，修复多在 4.2.1 | 见各条 introduced | 默认树：litellm → tokenizers → huggingface-hub。攻击前提是本机其他用户。未核实 |
| chromadb 1.5.9 | GHSA-f4j7-r4q5-qw2c（预认证代码注入）、GHSA-36p7-vc44-83pf、GHSA-2wm9-hf6c-p5cr、GHSA-xph7-9rjv-w5fr | `last_affected` 含 1.5.9 | 仅 rag extra。代码用进程内 `Chroma.from_documents`（`utils/rag.py:31`），是否监听端口未核实 |
| oauthlib 4.0.0 | GHSA-3jp2-3mrg-2vxc | 3.3.0–4.0.1 之前。设备码流程 | 经 chromadb → kubernetes → requests-oauthlib。本仓库未写设备码流程 |
| opentelemetry-api 1.44.0 | GHSA-mf5v-w3w3-f6cg、GHSA-xcq7-p3pf-p88h | 修复于 1.45.x。traceparent / tracestate 解析 | chromadb 的依赖。本仓库自己写 JSONL，不解析外来 traceparent |
| soupsieve 2.9.2 | 4 个 GHSA，选择器编译的复杂度与递归 | 修复版本 2.10.0 / 3.0.2 / 4.0.0 不等 | `beautifulsoup4` 的依赖。本仓库调用是固定标签 `find_all("a")` 等（`utils/web_api/target_surface.py:115`），不是把不可信字符串当选择器 |

## 镜像、CI 动作、子模块

- 子模块：无 `.gitmodules`。`publish-docker.yml` 的 checkout 写了 `submodules: recursive`，没有子模块可拉。
- `Dockerfile`：`FROM python:3.13-slim`，无 digest，无 `USER`。构建命令不使用锁文件。
- 发布：`.github/workflows/publish-docker.yml` 推 `ghcr.io/<仓库名小写>:<时间戳>` 和 `:latest`，无 digest。层未拉取，不写已确认安全。README 给出的运行示例是 `ghcr.io/ipa-lab/hackingbuddygpt:latest`。该仓库是否仍公开这个标签，本单未查询注册表。
- Actions 均无 commit SHA：`actions/checkout@v4`、`actions/setup-python@v3`、`actions/dependency-review-action@v4`、`docker/login-action@v3`、`actions/configure-pages@v4`、`actions/jekyll-build-pages@v1`、`actions/upload-pages-artifact@v3`、`actions/deploy-pages@v4`。这些 tag 指向的提交及其公告未核实。
- `dependency-review.yml` 会在 PR 上跑依赖审查，`fail-on-severity` 被注释掉，不阻断合并。

## 未覆盖

未下载任何 wheel 或 sdist。未拉取 `python:3.13-slim` 与 GHCR 镜像。未核对 litellm、langchain、langsmith 的默认遥测开关（本仓库源码没有 `telemetry = False` 或等价设置）。未把 Actions 浮动标签解析成 SHA。`pip install -e .` 与镜像构建不使用 `uv.lock`，这两条路径的实际版本未核实。
