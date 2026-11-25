# PDF 智能学习助手 — Technical Design

## 1. 系统概述

采用前后端分离架构：

```text
微信小程序（Vue3 + uni-app + Wot Design Uni）
                  |
                  | HTTPS REST API
                  v
Flask 后端（认证 / PDF任务 / AI生成 / 学习算法）
                  |
       +----------+----------+
       |                     |
     MySQL              本地文件或对象存储
 业务与学习数据             PDF 文件
```

## 2. 技术栈

- 前端：Vue 3、uni-app、TypeScript、Wot Design Uni、Pinia
- 后端：Python 3.11+、Flask、Flask-SQLAlchemy、Pydantic、Celery（可选）
- 数据库：MySQL 8
- PDF：PyMuPDF 或 pypdf
- OCR：Tesseract 或云端 OCR（扩展项）
- AI：OpenAI-compatible REST API
- 文件存储：开发环境本地目录，生产环境腾讯云 COS
- 部署：Gunicorn、Nginx、HTTPS

## 3. 前端结构

```text
frontend/
├── src/
│   ├── api/
│   ├── components/
│   ├── pages/
│   ├── stores/
│   ├── styles/
│   └── utils/
├── src/App.vue
├── src/main.js
├── src/manifest.json
└── src/pages.json
```

页面包括：首页、课程、文档详情、练习、错题本、复习、个人中心。

## 4. 后端结构

```text
backend/
├── app/
│   ├── api/
│   ├── services/
│   ├── models.py
│   ├── schemas.py
│   └── extensions.py
├── migrations/
├── tests/
└── app.py
```

## 5. 数据模型

### users

- id BIGINT PK
- openid VARCHAR(64) UNIQUE NULL
- nickname VARCHAR(100)
- avatar_url VARCHAR(500)
- created_at DATETIME
- updated_at DATETIME

### courses

- id BIGINT PK
- user_id BIGINT FK
- name VARCHAR(150)
- description TEXT
- deleted_at DATETIME NULL

### documents

- id BIGINT PK
- course_id BIGINT FK
- filename VARCHAR(255)
- file_path VARCHAR(500)
- file_size BIGINT
- status ENUM(uploaded, parsing, parsed, failed)
- page_count INT
- error_message TEXT
- deleted_at DATETIME NULL

### knowledge_points

- id BIGINT PK
- document_id BIGINT FK
- title VARCHAR(255)
- content TEXT
- page_start INT
- page_end INT
- section_title VARCHAR(255)

### questions

- id BIGINT PK
- knowledge_point_id BIGINT FK
- question_type ENUM(single, multiple, boolean, fill)
- difficulty ENUM(easy, medium, hard)
- question_text TEXT
- options_json JSON NULL
- answer_text TEXT
- explanation TEXT

### answer_records

- id BIGINT PK
- user_id BIGINT FK
- question_id BIGINT FK
- user_answer TEXT
- is_correct BOOLEAN
- answered_at DATETIME

### review_tasks

- id BIGINT PK
- user_id BIGINT FK
- knowledge_point_id BIGINT FK
- due_at DATETIME
- status ENUM(pending, completed, skipped)
- ease_factor DECIMAL
- review_count INT

### favorites

- id BIGINT PK
- user_id BIGINT FK
- knowledge_point_id BIGINT FK
- created_at DATETIME

## 6. 核心流程

### PDF 处理流程

```text
上传 PDF
  -> 校验格式和��小
  -> 保存文件
  -> 创建 document(status=uploaded)
  -> 异步解析
  -> 按页提取文本
  -> 清洗和分段
  -> 调用 AI 生成知识点
  -> 保存页码范围
  -> status=parsed
```

解析失败时保留错误信息并返回 `status=failed`。

### AI 题目生成流程

```text
选择知识点
  -> 构造含页码上下文的提示词
  -> 限定输出 JSON Schema
  -> 调用模型
  -> 校验题型、选项、答案和解析
  -> 保存题目
```

禁止把 MySQL 密钥、AI 密钥或管理员凭据下发到小程序。

## 7. API 设计

统一成功响应：

```json
{
  "code": 0,
  "message": "success",
  "data": {}
}
```

### 认证

- POST `/api/v1/auth/test-login`
- POST `/api/v1/auth/wechat-login`

### 课程与文档

- GET `/api/v1/courses`
- POST `/api/v1/courses`
- GET `/api/v1/courses/{course_id}`
- DELETE `/api/v1/courses/{course_id}`
- POST `/api/v1/documents/upload`
- GET `/api/v1/documents/{document_id}`
- DELETE `/api/v1/documents/{document_id}`

### 知识与题目

- GET `/api/v1/documents/{document_id}/knowledge`
- POST `/api/v1/knowledge/{knowledge_id}/generate-questions`
- GET `/api/v1/questions/{knowledge_id}`
- POST `/api/v1/questions/{question_id}/answer`
- POST `/api/v1/knowledge/{knowledge_id}/favorite`
- POST `/api/v1/knowledge/{knowledge_id}/ask`

### 复习与统计

- GET `/api/v1/review/today`
- POST `/api/v1/review/{task_id}/complete`
- GET `/api/v1/mistakes`
- GET `/api/v1/statistics/summary`
- POST `/api/v1/checkin`

## 8. 复习算法

首版使用简化 SM-2：

- 答对且连续答对：间隔乘 ease_factor。
- 答错：重置为当天或下一日复习。
- 初始 interval：0 天。
- 初始 ease_factor：2.5。

示例：

```text
第1次答对：1天
第2次答对：3天
第3次答对：7天
第4次答对：15天
答错：下一天
```

## 9. 安全与隐私

- 使用 JWT 或服务端登录会话。
- 校验用户对课程、文档和知识点的资源归属。
- 校验 PDF 扩展名、MIME、大小和文件结构。
- 使用 UUID 文件名，不使用用户原始文件名。
- AI 请求不附带其他用户数据。
- 删除文档时删除文件、知识点、题目和相关记录。
- 使用 HTTPS 和环境变量管理密钥。

## 10. 部署

开发环境：Flask、MySQL Docker、本地文件目录、微信开发者工具。

生产环境：Nginx → Gunicorn → Flask，Redis + Celery（可选），MySQL、腾讯云 COS、HTTPS。

## 11. 测试策略

- 后端：pytest 与接口测试。
- 数据库：测试数据库或事务回滚。
- PDF：文字型、乱码、加密和空文档用例。
- AI：固定 Mock 响应验证格式和重试。
- 前端：微信开发者工具真机预览。
- 核心流程：上传 → 解析 → 生成题目 → 答题 → 错题复习。
