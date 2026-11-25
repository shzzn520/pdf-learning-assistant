# Specification

## 1. 范围

本规格定义 PDF 智能学习助手 MVP 的页面、接口、数据和验收规则。

## 2. 术语

- Document：用户上传的 PDF 文档。
- Knowledge Point：由文档切片生成的知识点。
- Question：单选、多选、判断或填空题。
- Answer Record：一次答题记录。
- Review Task：需要在指定日期复习的知识点任务。

## 3. 页面规格

### 登录页

- 首版提供测试登录。
- 微信环境提供微信登录入口。
- 登录成功后跳转课程首页。

### 首页

展示：今日复习任务数量、今日打卡状态、最近课程、累计答题量和正确率。

### 课程详情页

展示课程名称、简介、文档列表和上传 PDF 按钮；支持删除文档和进入文档详情。

### 文档详情页

展示文档摘要、知识点列表、页码范围和收藏状态。

支持：生成题目、收藏知识点、知识点提问、查看错题。

### 练习页

配置题量、难度和题型��答题后展示是否正确、正确答案和解释，并保存答题记录。

### 错题本

展示错题、题型、知识点和错误次数；支持筛选、重新练习和移除。

### 复习页

展示今日待复习任务和逾期任务；支持开始复习、完成或跳过任务。

### 个人中心

展示累计学习天数、连续打卡天数、收藏数量及课程、文档和错题入口。

## 4. 接口规格

### 通用约定

请求头：

```text
Authorization: Bearer <token>
Content-Type: application/json
```

成功响应：

```json
{
  "code": 0,
  "message": "success",
  "data": {}
}
```

错误响应：

```json
{
  "code": 1001,
  "message": "课程不存在",
  "data": null
}
```

### 上传文档

```text
POST /api/v1/documents/upload
Content-Type: multipart/form-data
```

字段：

- course_id
- file

限制：仅接受 PDF；单文件默认不超过 20MB；原始文件名不得作为存储文件名。

响应：

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "document_id": 123,
    "status": "uploaded"
  }
}
```

### 创建题目

请求：

```json
{
  "knowledge_point_id": 10,
  "count": 10,
  "difficulty": "medium",
  "types": ["single", "multiple", "boolean"]
}
```

限制：count 为 1 到 50，类型数组不得为空，只能操作用户自己的知识点。

### 提交答案

请求：

```json
{
  "answer": "A"
}
```

响应：

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "is_correct": true,
    "correct_answer": "A",
    "explanation": "答案解析"
  }
}
```

### 提问

请求：

```json
{
  "question": "什么是缓存一致性？",
  "knowledge_point_id": 10
}
```

响应：

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "answer": "缓存一致性是...",
    "sources": [
      { "page_start": 12, "page_end": 13 }
    ]
  }
}
```

## 5. 数据规格

主要关系：

```text
User 1:N Course
Course 1:N Document
Document 1:N KnowledgePoint
KnowledgePoint 1:N Question
User 1:N AnswerRecord
User 1:N ReviewTask
User 1:N Favorite
```

必填约束：

- 用户、课程、文档和知识点名称不能为空��
- 文档状态必须是约定枚举值。
- 题型和难度必须是约定枚举值。
- 答题记录必须关联有效用户和题目。
- 复习任务的到期时间不能为空。

## 6. 非功能规格

- 单次正常接口响应建议不超过 3 秒；AI 和 PDF 任务允许异步。
- 并发上传使用任务队列，避免同步阻塞。
- 小程序限制每次上传 20MB。
- 不保存用户无关文件。
- MySQL 使用 UTF-8 字符集。
- 所有日期以 UTC 保存并在前端本地化。

## 7. 验收用例

| ID | 操作 | 预期结果 |
|---|---|---|
| AC-01 | 用户创建课程 | 课程出现在课程列表 |
| AC-02 | 上传文字型 PDF | 文档进入解析并最终解析成功 |
| AC-03 | 查看知识点 | 展示摘要、标题和页码范围 |
| AC-04 | 生成 10 道题 | 题目数量符合要求且格式有效 |
| AC-05 | 提交正确答案 | 返回正确并保存答题记录 |
| AC-06 | 提交错误答案 | 错题进入错题本 |
| AC-07 | 查看今日复习 | 展示待复习任务 |
| AC-08 | 完成复习 | 下一次到期时间按规则更新 |
| AC-09 | 删除文档 | 文件和关联数据不可再访问 |
| AC-10 | 提问知识点 | 回答包含可用页码来源 |

## 8. 交付物

- 微信小程序前端源码。
- Flask 后端源码。
- MySQL 建表脚本。
- API 文档。
- 部署说明。
- 测试结果。
- PRD、技术设计、AGENTS 和规格文档。
