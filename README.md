# 开发报告 — AI旅行规划师

## 一、项目概述

本项目是一个基于 Spring Boot + Vue3 + 硅基流动大模型的全栈 AI 旅行规划师应用。用户输入目的地、天数、预算、偏好，AI 自动生成按天拆分的行程单，并支持对话式修改。支持流式输出（SSE）和10轮对话记忆。

---

## 二、附录：提示词协作记录

以下摘录开发过程中 8 次关键 AI 提示词交互及其要点，标注了哪次交互带来了关键突破。

---

### 交互 1：项目初始化 — 从零构建全栈项目

**提示词**：
> 你是一位拥有10年经验的资深Java全栈工程师兼架构师。现在，我需要你帮我从零构建一个完整的【AI旅行规划师】全栈项目。核心功能：用户输入目的地、天数、预算、偏好，AI生成按天拆分的行程单（含交通、餐饮），并支持对话式修改。后端Java，前端Vue3+Vite+Element Plus，AI对接使用硅基流动智能体。

**AI 回答要点**：
- 设计了完整项目结构：后端 `agent/` 目录（Spring Boot），前端 `agent-vue/` 目录（Vue3+Vite+Element Plus）
- 后端分为 models（6个数据类）、agent（AI调用服务）、controller（REST API）三层
- 前端包含 App.vue（主布局）、api.js（Axios封装）、ItineraryCard.vue + ActivityDetail.vue（行程卡片渲染）
- System Prompt 强制 AI 输出严格 JSON 格式，定义了完整的行程 JSON Schema
- 后端开启 CORS，前端配置 Vite 代理解决跨域

**标记：关键突破** — 确定了整体架构和技术选型，是项目的基础框架搭建。

---

### 交互 2：数据模型设计 — 定义请求参数和行程数据结构

**提示词**：
> 后端项目代码文件放入agent。1. models：定义请求参数和AI返回的行程数据结构。2. agent：封装大模型调用逻辑，包含System Prompt，强制AI输出JSON。3. main：FastAPI主程序，包含生成行程和对话修改的API接口，需开启CORS。

**AI 回答要点**：
- `TravelRequest`：destination、days、budget、preferences、transportMode，使用 `@NotBlank`/`@Min` 校验
- `Itinerary`：title、destination、totalDays、budget、days列表、summary
- `DayPlan`：day、date、theme、morning/afternoon/evening（各为Activity对象）、transport、dining、estimatedCost
- `Activity`：name、description、duration、location、transport、cost、dining
- `ChatRequest`：message、currentItinerary、chatHistory，支持对话式修改
- `ChatMessage`：role、content

**标记：关键突破** — 数据模型直接决定了前后端交互契约和 AI Prompt 的 JSON Schema。

---

### 交互 3：前端组件设计 — 行程卡片可视化渲染

**提示词**：
> 前端项目代码文件放入agent-vue。1. App.vue：主布局，包含左侧表单/对话区，右侧行程展示区。2. api.js：封装调用后端接口的Axios请求。3. ItineraryCard.vue：解析JSON数据，按天、按时段（上午/下午/晚上）渲染行程卡片，包含交通和餐饮建议。

**AI 回答要点**：
- App.vue 采用左右分栏布局：左侧 420px 表单+对话区，右侧 flex:1 行程展示区
- 表单使用 Element Plus 的 el-form、el-input、el-input-number、el-select（多选偏好）、el-radio-group
- 对话区支持多轮对话历史展示，用户消息右对齐蓝色气泡，AI 消息左对齐白色气泡
- ItineraryCard.vue 按天渲染 el-card，每天内分上午/下午/晚上三个时段，使用 Element Plus 图标（Sunny/Sunrise/Moon）
- ActivityDetail.vue 渲染单个活动详情，包含地点、时长、交通、费用、餐饮
- 响应式设计：768px 以下切换为上下布局

**标记：关键突破** — 前端组件设计决定了用户交互体验和可视化效果。

---

### 交互 4：引入流式输出和10轮记忆

**提示词**：
> 为项目引入大模型，硅基流动 `https://siliconflow.cn/zh-cn/siliconc`，流式输出，记忆轮数为10。

**AI 回答要点**：
- 后端新增 `callAiStream()` 方法，使用 `HttpURLConnection` 读取 SSE 流式响应
- `SseEmitter` 用于向前端推送流式数据，120秒超时
- `ConcurrentHashMap<String, List<ChatMessage>>` 按 sessionId 存储对话记忆
- `getMemory()` 方法最多保留10轮（20条消息），超出自动截断最早的
- `addMemory()` 在流式结束后将本轮 user+assistant 消息存入
- Controller 新增 `/generate/stream` 和 `/modify/stream` 两个 SSE 接口
- 前端使用 `fetch + ReadableStream` 读取 SSE 流，逐字渲染带光标动画
- 前端显示记忆轮数 `x/10`，支持清空记忆按钮

**标记：关键突破** — 这是最核心的技术升级，将项目从简单请求-响应模式升级为支持流式输出和多轮记忆的智能对话系统。

---

### 交互 5：ChatMessage 构造方法报错修复

**提示词**：
> 出现报错，修改代码。

**AI 回答要点**：
- 问题定位：`ChatMessage` 使用 Lombok `@Data`，但代码中调用了 `new ChatMessage("user", prompt)` 带参构造方法
- Lombok `@Data` 默认只生成无参构造器和 getter/setter，不生成全参构造器
- 修复：添加 `@NoArgsConstructor` 和 `@AllArgsConstructor` 注解
- 同时验证其他 model 类（TravelRequest、ChatRequest、Itinerary、DayPlan、Activity）都有隐式无参构造器，无此问题

**标记：关键突破** — 解决了编译错误，确保项目可正常启动运行。Lombok 注解使用不当是常见陷阱。

---

### 交互 6：@Value 属性占位符错误修复

**提示词**：
> 代码报错，修改代码。

**AI 回答要点**：
- 问题定位：`@Value` 注解的属性占位符被错误替换成了实际值
  - `@Value("${sk-hrpozomtyzrjqikdihzosdkluipyihkhemzmpxypuqfibgmr}")` 应为 `@Value("${siliconflow.api-key}")`
  - `@Value("${https://api.siliconflow.cn}")` 应为 `@Value("${siliconflow.base-url}")`
  - `@Value("${Qwen/Qwen2.5-7B-Instruct}")` 应为 `@Value("${siliconflow.model}")`
- Spring `@Value` 的 `${}` 内必须填写属性名（key），而非直接写入值
- 修复：Java 代码恢复为属性占位符，实际值写入 `application.properties`
- 修复后 application.properties：
  ```properties
  siliconflow.api-key=sk-hrpozomtyzrjqikdihzosdkluipyihkhemzmpxypuqfibgmr
  siliconflow.base-url=https://api.siliconflow.cn/v1
  siliconflow.model=Qwen/Qwen2.5-7B-Instruct
  ```

**标记：关键突破** — 解决了 Spring 依赖注入失败导致的服务启动崩溃，确保后端可正常启动。

---

### 交互 7：项目提交准备 — Git仓库与开发宪法

**提示词**：
> 项目作业提交，源码仓库GitHub仓库，提交含.git目录的zip压缩包，须保留全部git提交记录；CLAUDE.md：项目根目录的开发宪法文件，内容覆盖项目简介、技术栈、规范与禁止事项；提示词协作记录：作为开发报告附录，摘录6条以上关键提示词与AI回答要点，标注哪次交互带来了关键突破。

**AI 回答要点**：
- 创建 CLAUDE.md 作为开发宪法文件，覆盖：
  - 项目简介（核心功能、5大特性）
  - 技术栈（后端7项、前端5项、AI服务2项，含版本和用途）
  - 开发规范（代码规范4条、Git提交规范、AI协作规范3条）
  - 禁止事项8条（硬编码Key、提交node_modules等）
- 创建 DEV_REPORT.md 开发报告，摘录8条关键提示词交互
- 初始化 Git 仓库，分阶段提交保留完整提交记录
- 配置 .gitignore 排除 node_modules、target 等无关文件
- 打包为含 .git 目录的 zip 压缩包

**标记：关键突破** — 完成项目交付物的规范化整理，满足作业提交的全部要求。

---

### 交互 8：System Prompt 设计 — 强制 JSON 输出

**提示词**：（隐含在交互1和交互4中）
> AI 必须严格输出 JSON 格式，前端负责解析并渲染为可视化的行程卡片。

**AI 回答要点**：
- System Prompt 明确角色定位："你是一位专业的旅行规划师"
- 严格输出约束："你必须只输出纯 JSON 格式，不要包含任何 markdown 代码块标记、解释文字或多余内容"
- 定义完整 JSON Schema，包含 title、destination、totalDays、budget、days数组（每天含 morning/afternoon/evening 三个 Activity 对象）
- 内容要求6条：行程合理可行、每天三时段活动、交通具体、餐饮含特色、费用符合预算、修改保持结构
- 后端 `extractJson()` 方法作为兜底：自动去除 AI 可能返回的 ```` ```json ```` 代码块标记

**标记：关键突破** — System Prompt 的 JSON Schema 设计是前后端数据契约的核心，确保 AI 输出可被程序解析。

---

## 三、技术亮点总结

| 技术点 | 实现方式 |
|--------|----------|
| 流式输出（SSE） | 后端 `SseEmitter` + `HttpURLConnection` 读取硅基流动 stream 响应；前端 `fetch + ReadableStream` 逐字渲染 |
| 10轮对话记忆 | `ConcurrentHashMap<sessionId, List<ChatMessage>>`，每次调用自动截断保留最近20条消息 |
| JSON 强制输出 | System Prompt 定义严格 JSON Schema + 后端 `extractJson()` 兜底去除 markdown 标记 |
| 可视化行程卡片 | Vue3 组件递归渲染，按天→时段→活动三级结构，Element Plus 图标区分上午/下午/晚上 |
| 跨域处理 | 后端 `@CrossOrigin(origins = "*")` + 前端 Vite proxy 双重保障 |
| 参数校验 | Spring Validation `@NotBlank`/`@Min` + Element Plus 表单验证 |

---

## 四、运行说明

### 后端启动
```powershell
cd c:\tccourse\code\agent
mvn spring-boot:run
```
服务运行在 `http://localhost:8000`

### 前端启动
```powershell
cd c:\tccourse\code\agent-vue
npm install
npm run dev
```
前端运行在 `http://localhost:3000`

### API 接口

| 接口 | 方法 | 说明 |
|------|------|------|
| `/api/travel/generate` | POST | 生成行程（非流式） |
| `/api/travel/generate/stream` | POST | 生成行程（SSE流式） |
| `/api/travel/modify` | POST | 修改行程（非流式） |
| `/api/travel/modify/stream` | POST | 修改行程（SSE流式） |
| `/api/travel/memory/clear` | POST | 清空对话记忆 |
| `/api/travel/health` | GET | 健康检查 |
