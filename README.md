# LLM 数据集生成器 - 部署与使用指南

**本软件旨在帮助用户从各种文档中提取内容，并利用大型语言模型（LLM）自动生成结构化的问答（QA）数据集，包括思维链（CoT）和领域标签，以简化数据标注和模型训练数据的准备过程。**

## 简介

本指南旨在说明如何在你的本地计算机上安装、配置并运行“LLM 数据集生成器”应用程序。该应用允许你上传文档、解析内容、分割文本，并利用多种大型语言模型（LLM），如 Google Gemini、Ollama（本地模型）、DeepSeek（深度求索）、SiliconFlow（硅基流动）等，来生成问答对（QA）、思维链（CoT）和领域标签（Domain Tag）。生成的数据将存储在本地的 SQLite 数据库中。

本指南假定你对使用命令行或终端有基本的了解。

## 系统要求与准备

在开始之前，请确保你的系统已安装以下软件和资源：

1.  **Python:** 推荐使用 3.10 或更新版本。你可以从 [python.org](https://www.python.org/) 下载。打开终端运行 `python --version` 或 `python3 --version` 来验证安装。
2.  **Git:** 用于从代码仓库克隆项目。你可以从 [git-scm.com](https://git-scm.com/) 下载。运行 `git --version` 验证安装。
3.  **LLM API 密钥 (必需，除非只用 Ollama):** 你需要为你打算使用的**云服务** LLM 提供商获取 API 密钥：
    *   **Google Gemini:** 从 [Google AI Studio](https://aistudio.google.com/) 或 Google Cloud Console 获取。
    *   **DeepSeek (深度求索):** 从 [DeepSeek 开放平台](https://platform.deepseek.com/) 获取。(*注意：请确保我们代码中的客户端实现与他们的官方 API 文档一致*)
    *   **SiliconFlow (硅基流动):** 从 [硅基流动官网](https://siliconflow.cn/) 获取。(*注意：请确保我们代码中的客户端实现与他们的官方 API 文档一致*)
    *   **OpenAI:** (如果你重新添加了支持) 从 OpenAI 平台获取。
4.  **(可选) Ollama:** 如果你想通过 Ollama 使用本地运行的 LLM，你需要先安装并运行 Ollama 服务，并且已经通过 `ollama pull <模型名>` (例如 `ollama pull llama3`) 拉取了所需的模型。你可以从 [ollama.ai](https://ollama.ai/) 下载 Ollama。
5.  **(非 Windows 系统) `libmagic`:** `python-magic` 库（用于识别文件 MIME 类型）在 Linux 和 macOS 上需要系统级的 `libmagic` 库支持。
    *   **Debian/Ubuntu:** `sudo apt-get update && sudo apt-get install -y libmagic1`
    *   **macOS (使用 Homebrew):** `brew install libmagic`
（此版本推荐使用Gemini相关模型）
## 安装步骤

请按照以下步骤安装应用程序：
1. **下载项目**
   推荐直接下载zip
2.  **创建并激活 Python 虚拟环境:**
    强烈建议使用虚拟环境来隔离项目依赖，避免与其他 Python 项目冲突。
    *   **创建虚拟环境** (在项目根目录 `llm-dataset-generator` 下运行):
        ```bash
        # Windows
        python -m venv venv
        # macOS/Linux
        python3 -m venv venv
        ```
    *   **激活虚拟环境:**
        ```bash
        # Windows (命令提示符 / PowerShell)
        .\venv\Scripts\activate
        # macOS/Linux (Bash / Zsh 等)
        source venv/bin/activate
        ```
        激活成功后，你的终端提示符前面应该会出现 `(venv)` 字样。**之后的所有命令都应在这个激活的虚拟环境中执行。**

3.  **安装项目依赖:**
    本项目依赖多个 Python 包。使用 pip 安装它们。
    *   **(推荐) 如果项目包含 `requirements.txt` 文件:**
        ```bash
        pip install -r requirements.txt
        ```
    *   **(备选) 如果没有 `requirements.txt` 文件，手动安装核心依赖:**
        ```bash
        pip install fastapi uvicorn[standard] sqlalchemy aiosqlite pydantic pydantic-settings python-dotenv google-generativeai httpx Pillow python-magic-bin jinja2 Markdown beautifulsoup4 openpyxl PyMuPDF PySocks
        # 注意：在 Linux/macOS 上，如果已安装系统 libmagic，请使用 'python-magic' 替换 'python-magic-bin'
        # 注意：PySocks 是在使用 SOCKS5 代理时才需要安装
        # 注意：如果你之前移除了 OpenAI，就不需要装 openai 库
        ```

4.  **配置应用程序 (`.env` 文件):**
    应用程序使用项目根目录下的 `.env` 文件来存储配置信息，特别是敏感的 API 密钥。
    *   **创建文件:** 如果项目提供了一个 `.env.example` 文件，先复制它：
        ```bash
        # Windows
        copy .env.example .env
        # macOS/Linux
        cp .env.example .env
        ```
        如果没有示例文件，请在项目根目录 (`llm-dataset-generator/`) 下手动创建一个名为 `.env` 的文件。
    *   **编辑 `.env` 文件:** 使用文本编辑器打开 `.env` 文件，并填入你需要使用的服务的配置：
        ```dotenv
        # .env 文件示例内容 - 请填入你自己的真实值

        PROJECT_NAME="我的 LLM 数据集生成器" # 可选：修改项目名称
        DEBUG_MODE=False # 设置为 True 会在开发时看到更详细的日志输出

        # --- LLM API 密钥和 URL (按需填写) ---
        GOOGLE_API_KEY="这里粘贴你的GEMINI_API密钥"
        # OPENAI_API_KEY="这里粘贴你的OPENAI_API密钥" # 如果你添加了 OpenAI 支持
        DEEPSEEK_API_KEY="这里粘贴你的DEEPSEEK_API密钥"
        SILICONFLOW_API_KEY="这里粘贴你的SILICONFLOW_API密钥"

        # 可选的基础 URL (通常用于代理或非官方端点)
        # OPENAI_BASE_URL=
        # DEEPSEEK_BASE_URL=
        # SILICONFLOW_BASE_URL=

        # Ollama 的 URL (如果你的 Ollama 服务不在默认地址或端口)
        OLLAMA_BASE_URL="http://localhost:11434"

        # --- 数据库 (通常保持默认) ---
        # 数据库文件 'app.db' 会在项目根目录自动创建
        ```
    *   **安全警告:** **绝对不要**将包含你的 API 密钥的 `.env` 文件提交到 Git 仓库或公开分享。项目自带的 `.gitignore` 文件应该已经包含了 `.env`，但这仍需注意。

5.  **(可选) 配置网络代理:**
    如果你的网络环境需要通过代理才能访问互联网（特别是访问 Google、OpenAI 等服务），你需要在**运行应用之前**配置好代理。在将要运行 Uvicorn 的**同一个终端窗口**中设置 `HTTP_PROXY` 和 `HTTPS_PROXY` 环境变量。
    *   **对于 HTTP 代理:**
        ```bash
        # Windows
        set HTTP_PROXY=http://你的代理地址:端口
        set HTTPS_PROXY=http://你的代理地址:端口
        # macOS/Linux
        export HTTP_PROXY="http://你的代理地址:端口"
        export HTTPS_PROXY="http://你的代理地址:端口"
        ```
    *   **对于 SOCKS5 代理:** (需要已安装 `PySocks`: `pip install PySocks`)
        ```bash
        # Windows
        set HTTP_PROXY=socks5://你的代理地址:端口
        set HTTPS_PROXY=socks5://你的代理地址:端口
        # macOS/Linux
        export HTTP_PROXY="socks5://你的代理地址:端口"
        export HTTPS_PROXY="socks5://你的代理地址:端口"
        ```
    请将 `你的代理地址:端口` 替换为你的实际代理服务器信息。

## 运行应用程序

1.  确保你的 Python **虚拟环境已经被激活** (终端提示符前有 `(venv)` )。
2.  确保必要的**网络代理环境变量已经设置** (如果你需要代理的话)。
3.  使用 Uvicorn 启动 FastAPI 服务器。在项目根目录 (`llm-dataset-generator`) 下运行以下命令：
    ```bash
    uvicorn app.main:app --reload
    ```
    *   `--reload` 参数使得在你修改代码后服务器会自动重启，方便开发。如果用于稳定运行，可以去掉 `--reload`。

    服务器成功启动后，你会看到类似以下的输出：
    ```
    INFO:     Uvicorn running on http://127.0.0.1:8000 (Press CTRL+C to quit)
    INFO:     Started reloader process [...] using StatReload
    INFO:     Started server process [...]
    INFO:     Waiting for application startup.
    INFO:     Application startup complete.
    ```

## 使用应用程序

应用程序启动后，你可以通过以下方式使用它：

1.  **Web 用户界面 (UI):**
    *   打开你的网页浏览器，访问 `http://127.0.0.1:8000/`。
    *   你会看到一个简单的表单：
        *   **选择 LLM 提供商:** 从下拉列表中选择你想使用的 LLM 服务（如 gemini, ollama 等）。
        *   **选择或输入模型名称:** 根据选择的提供商，可以从预设模型下拉列表中选择，或者手动输入一个模型名称。如果留空，将使用该提供商的默认模型。
        *   **输入文本块:** 将你想要生成 QA 数据的文本内容粘贴到这里。
        *   **点击 "生成数据" 按钮:** 应用会将你的请求发送到后端进行处理。
    *   处理完成后，生成的 QA 对、CoT、标签等信息会以 JSON 格式显示在页面下方。如果出错，也会显示错误信息。

2.  **API 交互文档 (Swagger UI):**
    *   打开浏览器，访问 `http://127.0.0.1:8000/api/docs`。
    *   这里列出了所有可用的后端 API 接口，你可以：
        *   查看每个接口的详细说明、请求参数、响应格式。
        *   直接在页面上测试每个接口（点击 "Try it out"）。例如，你可以测试文件上传、文本分割、直接调用 QA 生成 API 等。

**基本工作流程:**

1.  （可选）使用 `/api/documents/upload` (通过 `/api/docs` 测试) 上传你的原始文档。
2.  （可选）使用 `/api/documents/{filename}/parse` (通过 `/api/docs` 测试) 解析上传的文档，获取文本内容或图片元数据。
3.  （可选）使用 `/api/tasks/text-split` (通过 `/api/docs` 测试) 将长文本分割成合适的文本块 (Chunks)。
4.  访问主 UI 页面 (`http://127.0.0.1:8000/`)：
    *   选择你想用的 LLM 提供商和模型。
    *   将第二步或第三步得到的**文本块**粘贴到输入框中。
    *   点击生成按钮。
5.  查看页面上显示的生成结果。生成的数据已自动保存到本地的 `app.db` 数据库文件中。
6.  （可选）使用 `/api/datasets/qa-pairs` (通过 `/api/docs` 测试) 查看数据库中存储的所有 QA 对摘要。
7.  （可选）使用 `/api/datasets/export` (通过 `/api/docs` 测试) 将数据库中的数据导出为 JSON, JSONL 或 CSV 文件。

## 停止应用程序

在运行 Uvicorn 的终端窗口中，按下 `Ctrl + C` 组合键即可停止服务器。

## 故障排除

*   **`ModuleNotFoundError`:** 通常意味着依赖库没有安装在当前的虚拟环境中。请确保虚拟环境已激活，并重新运行 `pip install -r requirements.txt` 或手动安装缺失的库。
*   **`ImportError: cannot import name ...`:** 可能是代码文件不完整或存在语法错误，或者导入路径配置有问题。请检查报错信息中提到的文件。
*   **API 调用超时或连接错误:** 大概率是网络问题或代理设置问题。请检查你的网络连接，并确认代理（如果需要）已正确配置并运行。可以尝试使用 `ping` 和 `curl` 命令测试目标 API 服务器的可达性。
*   **`503 Service Unavailable` (来自 `/generate-qa`):** 通常表示后端的 LLM API 调用失败。检查 Uvicorn 的终端日志，查找来自 LLM 客户端（如 gemini.py, openai_client.py）的更详细错误信息，可能是 API Key 无效、模型名称错误、内容安全策略阻止等。
*   **`422 Unprocessable Entity`:** 表示你发送给 API 的请求体数据格式不正确（例如 JSON 格式错误，或者字段值不符合要求）。检查 API 文档 (`/api/docs`) 中对应接口的请求体 Schema。
