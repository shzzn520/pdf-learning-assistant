# AGENTS.md

## 项目目标

微信小程序项目：上传 PDF 后，后端解析文档并调用 AI 生成知识点、题目、解析、复习任务和问答。

## 默认技术栈

- 前端：Vue 3、uni-app、TypeScript、Wot Design Uni
- 后端：Python 3.11+、Flask
- 数据库：MySQL 8
- PDF：PyMuPDF
- 模型接口：兼容 OpenAI 格式的服务
- 异步任务：MVP 可先同步实现，后续引入 Celery

## 工作原则

1. 优先实现完整闭环，不堆叠无关功能。
2. 首版只保证文字型 PDF，不把复杂 OCR 作为核心依赖。
3. 不在前端保存 AI API Key、数据库密码或管理员密钥。
4. 所有 AI 输出先校验，再写入数据库。
5. 每个知识点保存原文页码范围。
6. 删除操作必须考虑相关文件和数据库记录。
7. 文档和代码注释使用中文，除非仓库已有英文约定。

## 禁止事项

- 不要把 PDF 二进制数据保存到 MySQL。
- 不要在浏览器或小程序端调用原始模型 API。
- 不要假设所有 PDF 都包含可提取文字。
- 不要在小程序中使用浏览器 DOM 或 React-only 组件。
- 不要使用 shadcn/ui、shadcn-vue、Flutter 或 ArkUI 替代小程序组件。
- 不要提交密钥、真实用户数据或本地数据库文件。

## 仓库结构约定

```text
PRD.md
TechDesign.md
spec.md
AGENTS.md
frontend/
backend/
```

## 前端规范

- 使用 uni-app Vue3 语法。
- 使用 `pages.json` 和 easycom 引入组件。
- 使用 Wot Design Uni 作为基础组件库。
- 页面复用 API 模块，不直接拼装服务端地址。
- 在 API 模块统一处理状态码、网络错误和登录过期。
- 页面不直接操作数据库。
- 不使用原生 DOM API。

## 后端规范

- 使用分层结构：API → Service → Model。
- 公开接口使用版本前缀 `/api/v1`。
- 校验资源归属：课程、文档、知识点和任务只能由所属用户操作。
- 统一返回格式：`code`、`message`、`data`。
- 上传文件使用 UUID 新文件名。
- 长任务优先异步处理。
- 异常不向前端泄露堆栈。

## 题目格式约定

AI 题目必须符合：

```json
{
  "question_type": "single",
  "difficulty": "medium",
  "question_text": "题干",
  "options": ["选项A", "选项B", "选项C", "选项D"],
  "answer_text": "选项A",
  "explanation": "解析",
  "page_start": 1,
  "page_end": 2
}
```

## 数据库规范

- 使用自增或统一的主键方案。
- 删除重要资源时优先软删除并保留审计状态。
- 不保存 PDF 完整二进制内容。
- JSON 字段仅用于选项等结构化但易变数据。
- 为外键和查询字段建立索引。

## 测试要求

- 新增后端接口必须提供 pytest 测试。
- PDF 解析至少覆盖正常文字和解析失败场景。
- AI 生成必须覆盖 JSON 格式错误和空内容。
- 题目答案校验覆盖选择、多选、填空和判断题。
- 前端核心流程在微信开发者工具中验证。

## 提交规范

- `feat:` 新功能
- `fix:` 修复
- `docs:` 文档
- `test:` 测试
- `refactor:` 重构
- `chore:` 构建或配置

示例：`feat: 添加 PDF 文档上传和解析流程`

## 调试规范

- 优先复现问题并定位小程序、Flask、数据库、PDF 或 AI 所在层级。
- 不在日志中输出完整 PDF 或模型密钥。
- 异步问题使用任务状态日志排查。

## 当前任务优先级

1. 项目骨架。
2. 登录和课程管理。
3. PDF 上传与解析。
4. 知识点展示。
5. AI 出题与答题。
6. 错题与复习。
7. 学习统计与打卡。
