# 数智底座 · 智能问数后端

基于自然语言查询订单数据仓库的异步服务。用户提出业务问题后，系统召回字段、指标和真实取值，组装上下文、生成并校验 SQL，再通过 SSE 返回节点进度与数据结果。

前端项目：[data-agent-fronted](https://github.com/luckyblank/data-agent-fronted)。

## 技术架构

Python 3.12、FastAPI、LangChain、LangGraph、SQLAlchemy 2、MySQL 8、Qdrant、Elasticsearch、BGE Embedding、Docker Compose。

问数链路包含 12 个节点：关键词抽取；字段、指标、取值三路并行召回；信息合并；表与指标筛选；上下文补充；SQL 生成；EXPLAIN 预校验；错误纠正；SQL 执行。SQL 校验失败时触发一次纠正，然后执行。

元数据由 `conf/meta_config.yaml` 维护，当前配置包含订单事实表与地区、商品、客户、时间四张维度表。元数据与指标存储在 MySQL，字段及指标语义索引使用 Qdrant，真实取值检索使用 Elasticsearch。

## 本地运行

需要 Python 3.12+、[uv](https://docs.astral.sh/uv/)、Docker Compose，以及能够访问所选模型服务的本地凭证。

1. 安装依赖并准备配置（在项目根目录执行）：

   ```powershell
   uv sync --locked
   Copy-Item conf/app_config.example.yaml conf/app_config.yaml
   Copy-Item docker/docker-compose.example.yaml docker/docker-compose.yaml
   Copy-Item docker/.env.example docker/.env
   ```

2. 编辑 `conf/app_config.yaml` 和 `docker/.env`。设置 MySQL 密码以及自己的模型名称、API key 和 provider base URL。配置中的 `host` 只填写主机名或 IP，不包含 `http://`。Embedding 向量维度需要与所用模型一致，模板采用 BGE large 的 1024 维。

3. 准备本地下载资产。将 BGE 模型放到 `docker/embedding/bge-large-zh-v1.5/`；将与 Dockerfile 中 Elasticsearch **8.19.10** 版本一致的 IK 插件 ZIP 放到 `docker/elasticsearch/plugins/elasticsearch-analysis-ik-8.19.10.zip`。这些下载资产不随 Git 仓库上传。

4. 启动基础服务：

   ```powershell
   docker compose --project-directory docker -f docker/docker-compose.yaml --env-file docker/.env up -d --build
   ```

   示例 Compose 将端口绑定到本机：MySQL `3307`，Elasticsearch `9200`，Qdrant `6333/6334`，Embedding `8081`。MySQL SQL 脚本初始化 `meta` 和 `dw`；初始化仅在首次创建数据卷时执行。示例 Elasticsearch 配置面向本地开发。

5. 服务就绪后构建知识库，再启动 FastAPI：

   ```powershell
   uv run python -m app.scripts.build_meta_knowledge -c conf/meta_config.yaml
   uv run python main.py
   ```

   服务默认监听 `8000`，接口文档为 `http://localhost:8000/docs`。

## 问数接口

`POST /api/query`，请求头 `Content-Type: application/json`，请求体：

```json
{"query":"统计去年各地区的销售总额，按销售额降序排列"}
```

返回 `text/event-stream`，每个事件是 `data: <JSON>`，事件之间由空行分隔：

- `progress`：`step` 表示节点名称，`status` 为 `running`、`success` 或 `error`。
- `result`：`data` 为数据库返回的对象数组，包括空结果数组。
- `error`：`message` 为本次查询的错误信息。

每次请求独立执行，服务目前不接收历史聊天上下文，也不返回生成的 SQL 文本。前端停止接收数据流不构成后端任务已经终止的保证。

## 仓库内容与配置

`app/api` 提供路由与参数结构；`app/agent` 组织状态图与处理节点；`app/services` 编排业务；`app/repositories` 对接存储；`app/clients` 管理异步连接；`prompts` 保存提示模板；`docker/mysql` 保存本地数据初始化脚本。

`conf/app_config.yaml`、`docker/.env`、本地 Compose 文件、日志、虚拟环境、模型权重与插件 ZIP 已通过 `.gitignore` 排除。仓库仅提供不含真实凭证的模板。
