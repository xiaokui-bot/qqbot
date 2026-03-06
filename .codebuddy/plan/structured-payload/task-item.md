# 实施计划：QQBot 结构化消息载荷优化

## 概述

本实施计划基于需求文档，将 QQBot 消息处理从旧的 Markdown 语法解析方式升级为统一的结构化载荷（`QQBOT_PAYLOAD:`）格式。

---

## 任务清单

- [ ] 1. 定义载荷类型接口和解析工具函数
   - 创建 `src/utils/payload.ts` 文件
   - 定义 `QQBotPayload` 联合类型，包含 `CronReminderPayload` 和 `MediaPayload`
   - 实现 `parseQQBotPayload(text: string)` 函数：检测 `QQBOT_PAYLOAD:` 前缀，提取并解析 JSON
   - 实现 `encodePayloadForCron(payload: CronReminderPayload)` 函数：将载荷编码为 Base64 并添加 `QQBOT_CRON:` 前缀
   - 实现 `decodeCronPayload(message: string)` 函数：解码 `QQBOT_CRON:` 前缀的消息
   - _需求：1.1、1.2、1.3、1.5、2.2_

- [ ] 2. 实现 AI 响应消息的载荷检测与分发处理
   - 修改 `src/gateway.ts` 中的消息处理逻辑
   - 在处理 AI 回复时，优先调用 `parseQQBotPayload()` 检测结构化载荷
   - 根据 `type` 字段分发到对应处理器：
     - `cron_reminder` → 定时提醒处理器
     - `media` → 媒体消息处理器
   - 非结构化消息作为普通文本处理
   - _需求：1.2、1.4、1.5_

- [ ] 3. 实现定时提醒载荷处理器
   - 在 `src/gateway.ts` 或单独模块中实现 `handleCronReminderPayload()` 函数
   - 接收解析后的 `CronReminderPayload` 对象
   - 调用 `encodePayloadForCron()` 生成 Base64 编码的消息
   - 构建 `openclaw cron add` 命令，使用 `--message "QQBOT_CRON:{base64}"` 参数
   - _需求：2.1、2.2_

- [ ] 4. 实现媒体消息载荷处理器
   - 在 `src/gateway.ts` 或单独模块中实现 `handleMediaPayload()` 函数
   - 接收解析后的 `MediaPayload` 对象
   - 根据 `mediaType` 分发处理：
     - `image` → 调用现有图片发送逻辑
     - `audio` → 调用音频发送 API
     - `video` → 预留，暂返回不支持提示
   - 根据 `source` 类型处理媒体来源（`url` / `file`）
   - _需求：3.1、3.2、3.3_

- [ ] 5. 实现定时提醒触发时的载荷解析
   - 修改 `src/proactive.ts` 或相关模块中处理 cron 触发消息的逻辑
   - 检测 `QQBOT_CRON:` 前缀
   - 调用 `decodeCronPayload()` 解码 Base64 获取 JSON
   - 根据 `targetType` 和 `targetAddress` 发送 `content` 内容
   - _需求：2.3_

- [ ] 6. 实现接收图片的自然语言描述生成
   - 修改 `src/gateway.ts` 中处理用户发送图片的逻辑
   - 当用户消息包含图片附件时，构建自然语言描述
   - 描述格式：包含图片地址、格式、消息ID、时间戳等信息
   - 将描述拼接到用户消息内容中传递给 AI
   - _需求：4.1、4.2、4.3_

- [ ] 7. 更新 AI 提示词配置
   - 修改 `skills/qqbot-media/SKILL.md`：
     - 移除旧的 Markdown 图片语法说明
     - 添加 `QQBOT_PAYLOAD:` + JSON 格式的发送图片/音频指南
     - 明确说明 AI 只需输出 JSON，Base64 编码由代码处理
   - 修改 `skills/qqbot-cron/SKILL.md`：
     - 添加结构化载荷格式的定时提醒设置说明
     - 更新 `--message` 参数说明，指明使用新的载荷格式
   - _需求：5.1、5.2_

- [ ] 8. 移除旧格式解析代码
   - 在 `src/gateway.ts` 中删除：
     - Markdown 图片语法解析正则 `mdImageRegex`
     - 裸 URL 图片提取正则 `bareUrlRegex`
     - 裸本地路径检测正则 `bareLocalPathRegex`
     - `collectImageUrl()` 函数（如存在）
     - 相关的图片 URL 收集和处理逻辑
   - 删除 `[[reply_to:xxx]]` 等内部标签处理逻辑（如存在）
   - _需求：6.1、6.2、6.3_

- [ ] 9. 添加错误处理和日志记录
   - 在载荷解析失败时记录错误日志
   - 在媒体发送失败时返回友好的错误提示
   - 在类型未识别时回退到普通文本处理
   - 添加关键节点的调试日志，便于问题排查
   - _需求：3.3_

- [ ] 10. 集成测试和验证
   - 测试定时提醒场景：设置提醒 → 触发 → 收到消息
   - 测试发送图片场景：AI 输出载荷 → 解析 → 发送成功
   - 测试接收图片场景：用户发送图片 → AI 收到自然语言描述
   - 测试普通文本场景：非载荷消息正常发送
   - 测试错误场景：JSON 格式错误、媒体路径无效等
   - _需求：成功标准 1-8_

---

## 依赖关系

```
任务 1 (基础工具)
    ↓
任务 2 (分发框架) ──────┬──────┐
    ↓                  ↓      ↓
任务 3 (定时提醒)   任务 4 (媒体)  任务 6 (接收图片)
    ↓
任务 5 (触发解析)
    
任务 7 (提示词更新) 可并行
任务 8 (移除旧代码) 依赖任务 2-4 完成后
任务 9 (错误处理) 贯穿全流程
任务 10 (测试验证) 最后执行
```

---

## 涉及文件

| 文件 | 修改类型 | 说明 |
|------|---------|------|
| `src/utils/payload.ts` | 新增 | 载荷类型定义和解析工具 |
| `src/gateway.ts` | 修改 | 消息处理逻辑重构 |
| `src/proactive.ts` | 修改 | 定时提醒触发处理 |
| `skills/qqbot-media/SKILL.md` | 修改 | AI 媒体发送提示词 |
| `skills/qqbot-cron/SKILL.md` | 修改 | AI 定时提醒提示词 |
| `src/utils/image-size.ts` | 可能删除部分 | 旧的 Markdown 图片格式化函数 |
