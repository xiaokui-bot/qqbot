# 需求文档

## 引言

本文档描述了对 QQBot 插件**消息载荷格式**的统一优化方案。通过设计一套 Base64 编码的 JSON 结构化消息格式，解决以下场景中的问题：

### 场景一：定时提醒

当前实现中，`openclaw cron add` 的 `--message` 参数只支持纯文本内容，当 cron 任务触发时，OpenClaw 会启动一个新的 Agent session 并将 message 作为用户输入传递给 AI。这种方式存在以下问题：

1. **上下文丢失**：AI 在处理定时任务时没有原始请求的上下文（如原始消息ID等）
2. **行为不确定**：AI 可能不理解应该如何处理收到的消息，导致返回 `HEARTBEAT_OK` 等无意义响应
3. **无法引用原消息**：虽然 `cron add` 不支持 `--reply-to` 参数，但可以通过其他方式实现消息关联

### 场景二：图片发送

当前实现中，图片发送依赖 Markdown 语法 `![](path)` 进行识别和解析。这种方式存在以下问题：

1. **解析歧义**：Markdown 图片语法可能与正常文本内容混淆，特别是当用户讨论代码示例时
2. **元数据缺失**：无法携带图片的附加信息（如来源、描述、压缩选项等）
3. **格式不统一**：与其他结构化数据的处理方式不一致，增加维护成本
4. **自定义标签依赖**：原有实现依赖 `[[reply_to: xxx]]` 等自定义标签，容易泄露到用户可见的消息中

### 优化思路

设计一套**统一的结构化消息载荷格式**：
- 使用 `QQBOT_PAYLOAD:` 前缀标识结构化消息
- 消息体为 Base64 编码的 JSON 结构
- 通过 `type` 字段区分不同的消息类型（定时提醒、图片、富文本等）
- QQBot 插件统一解析和处理，无需 AI 二次处理
- **不再支持旧的 Markdown 图片语法**，全部采用新格式

---

## 需求

### 需求 1：设计统一的结构化消息载荷格式

**用户故事**：作为 QQBot 插件开发者，我希望定义一种统一的结构化消息载荷格式，以便在各种场景下传递完整的上下文信息。

#### 验收标准

1. WHEN 构建结构化消息 THEN 系统 SHALL 使用统一的前缀 `QQBOT_PAYLOAD:` 标识

2. WHEN 编码消息载荷 THEN 系统 SHALL 将以下基础字段编码为 JSON：
   - `type`: 消息类型标识（`"cron_reminder"` | `"media"`）
   - `version`: 载荷格式版本号（如 `"1.0"`）
   - `timestamp`: 消息创建时间戳（UTC 毫秒）

3. WHEN Base64 编码完成 THEN 系统 SHALL 生成形如 `QQBOT_PAYLOAD:eyJ0eXBlIjoi...` 的消息格式

4. IF JSON 编码失败 THEN 系统 SHALL 记录错误日志

### 需求 2：定时提醒载荷格式（type: cron_reminder）

**用户故事**：作为 AI 助手，我希望在创建定时提醒时能够正确生成结构化的消息载荷，以便提醒能够可靠地送达用户。

#### 验收标准

1. WHEN 类型为 `cron_reminder` THEN 载荷 SHALL 包含以下字段：
   ```typescript
   {
     type: "cron_reminder",
     version: "1.0",
     content: string,              // 提醒文本内容
     targetType: "c2c" | "group",  // 目标类型
     targetAddress: string,        // 目标地址（user_openid 或 group_openid）
     originalMessageId?: string,   // 原始消息 ID（可选，用于上下文关联）
     timestamp: number             // 创建时间戳
   }
   ```

2. WHEN 解析成功且类型为 `cron_reminder` THEN 系统 SHALL 直接调用消息发送 API，将 `content` 发送给目标用户

3. WHEN 投递 C2C 消息 THEN 系统 SHALL 使用主动消息 API（不依赖 msg_id 回复）

4. WHEN 投递群组消息 THEN 系统 SHALL 根据 `targetType` 选择正确的发送接口

### 需求 3：媒体消息载荷格式（type: media）

**用户故事**：作为 AI 助手，我希望在发送图片时能够使用结构化载荷，以便携带更丰富的元数据并避免解析歧义。

#### 验收标准

1. WHEN 类型为 `media` THEN 载荷 SHALL 包含以下字段：
   ```typescript
   {
     type: "media",
     version: "1.0",
     mediaType: "image" | "file",   // 媒体类型
     source: "local" | "url",       // 来源类型
     path: string,                  // 本地路径或 URL
     caption?: string,              // 图片描述/说明文字（可选）
     timestamp: number              // 创建时间戳
   }
   ```

2. WHEN 解析成功且类型为 `media` THEN 系统 SHALL：
   - IF `source` 为 `"local"` THEN 读取本地文件并转换为 Base64
   - IF `source` 为 `"url"` THEN 直接使用 URL 发送

3. WHEN `caption` 字段存在 THEN 系统 SHALL 将说明文字与图片一起发送

4. IF 文件不存在或读取失败 THEN 系统 SHALL 返回错误信息给用户

### 需求 4：AI 提示词更新支持结构化消息

**用户故事**：作为 AI 助手使用者，我希望 AI 能够正确生成结构化载荷，以便消息能够可靠地处理。

#### 验收标准

1. WHEN AI 需要发送图片 THEN 系统 SHALL 在输出中生成结构化载荷

2. WHEN AI 生成图片消息 THEN 输出格式 SHALL 为：
   ```
   QQBOT_PAYLOAD:eyJ0eXBlIjoibWVkaWEiLC...
   ```

3. WHEN AI 创建定时提醒 THEN 系统 SHALL 生成包含完整上下文的载荷

4. WHEN 更新 gateway.ts 提示词 THEN 系统 SHALL 提供：
   - 载荷 JSON 结构说明
   - 编码方法示例（使用 shell 的 base64 命令或直接输出预编码字符串）
   - 各字段的取值说明

### 需求 5：消息解析与分发处理

**用户故事**：作为 QQBot 插件，我希望能够统一解析结构化消息载荷，并根据类型分发到不同的处理器。

#### 验收标准

1. WHEN 收到以 `QQBOT_PAYLOAD:` 开头的消息 THEN 系统 SHALL 识别为结构化消息

2. WHEN 解析 Base64 编码的消息 THEN 系统 SHALL：
   - 移除 `QQBOT_PAYLOAD:` 前缀
   - 解码 Base64 字符串
   - 解析 JSON 结构
   - 验证 `type` 和 `version` 字段

3. WHEN 解析成功 THEN 系统 SHALL 根据 `type` 字段分发到对应处理器：
   - `"cron_reminder"` → `handleCronReminder()`
   - `"media"` → `handleMediaMessage()`
   - 其他类型 → 记录警告

4. IF 解析失败 THEN 系统 SHALL：
   - 记录错误日志（包含原始消息前 100 字符、错误原因）

### 需求 6：移除旧格式支持

**用户故事**：作为插件维护者，我希望移除旧的 Markdown 图片语法解析，以简化代码并避免歧义。

#### 验收标准

1. WHEN 收到 Markdown 图片语法 `![](path)` THEN 系统 SHALL NOT 解析为图片消息

2. WHEN 收到不以 `QQBOT_PAYLOAD:` 开头的消息 THEN 系统 SHALL 作为普通文本处理

3. WHEN 移除旧代码 THEN 系统 SHALL 删除以下功能：
   - `collectImageUrl()` 函数中的 Markdown 语法解析逻辑
   - `filterInternalMarkers()` 等标签过滤函数
   - 相关的正则表达式和解析代码

4. WHEN 代码清理完成 THEN 系统 SHALL 确保所有图片发送都使用新的载荷格式

### 需求 7：SKILL.md 文档更新

**用户故事**：作为 AI 助手使用者，我希望 SKILL.md 文档清晰说明新的消息格式，以便 AI 能正确生成结构化消息。

#### 验收标准

1. WHEN 更新 qqbot-media SKILL.md 文档 THEN 系统 SHALL 包含：
   - 新的结构化载荷格式说明
   - 各字段含义和取值范围
   - 完整的编码示例

2. WHEN 更新 qqbot-cron SKILL.md 文档 THEN 系统 SHALL 包含：
   - 新的 cron_reminder 载荷格式
   - Base64 编码示例
   - 完整的使用流程

3. WHEN 文档更新完成 THEN 系统 SHALL 移除所有关于旧 Markdown 语法的说明

### 需求 8：错误处理与日志增强

**用户故事**：作为运维人员，我希望在消息处理出错时能够快速定位问题，以便及时修复。

#### 验收标准

1. WHEN Base64 解码失败 THEN 系统 SHALL 记录：
   - 错误类型：`PAYLOAD_DECODE_ERROR`
   - 原始消息前 100 字符
   - 错误原因

2. WHEN JSON 解析失败 THEN 系统 SHALL 记录：
   - 错误类型：`PAYLOAD_PARSE_ERROR`
   - 解码后的字符串
   - JSON 解析错误详情

3. WHEN 字段验证失败 THEN 系统 SHALL 记录：
   - 错误类型：`PAYLOAD_VALIDATION_ERROR`
   - 缺失或无效的字段名称
   - 期望的字段类型

4. WHEN 媒体文件不存在 THEN 系统 SHALL 记录：
   - 错误类型：`MEDIA_NOT_FOUND`
   - 请求的文件路径

5. WHEN 成功处理结构化消息 THEN 系统 SHALL 记录：
   - `[qqbot] Payload processed: type={type}, success=true`

---

## 技术设计建议

### 统一载荷基础结构

```typescript
// 基础载荷接口
interface BasePayload {
  type: string;                    // 消息类型标识
  version: string;                 // 载荷格式版本
  timestamp: number;               // 创建时间戳（UTC 毫秒）
}

// 定时提醒载荷（只包含实际可获得的字段）
interface CronReminderPayload extends BasePayload {
  type: "cron_reminder";
  content: string;                 // 提醒文本内容
  targetType: "c2c" | "group";     // 目标类型
  targetAddress: string;           // 目标地址（user_openid 或 group_openid）
  originalMessageId?: string;      // 原始消息 ID（可选）
}

// 媒体消息载荷
interface MediaPayload extends BasePayload {
  type: "media";
  mediaType: "image" | "file";     // 媒体类型
  source: "local" | "url";         // 来源类型
  path: string;                    // 本地路径或 URL
  caption?: string;                // 说明文字
}

// 载荷联合类型
type QQBotPayload = CronReminderPayload | MediaPayload;
```

### 消息格式示例

#### 定时提醒

**JSON 原文**:
```json
{
  "type": "cron_reminder",
  "version": "1.0",
  "content": "💧 喝水时间到！",
  "targetType": "c2c",
  "targetAddress": "207A5B8339D01F6582911C014668B77B",
  "originalMessageId": "ROBOT1.0_abc123",
  "timestamp": 1706860800000
}
```

**编码后**:
```
QQBOT_PAYLOAD:eyJ0eXBlIjoiY3Jvbl9yZW1pbmRlciIsInZlcnNpb24iOiIxLjAiLCJjb250ZW50Ijoi8J+SqyDllp3msLTml7bpl7TliLDvvIEiLCJ0YXJnZXRUeXBlIjoiYzJjIiwidGFyZ2V0QWRkcmVzcyI6IkIzRUE5QTFkLTJEM2MtNUNCRC1DNDIyLWUzYjQ3NkIyM2ExYiIsInRpbWVzdGFtcCI6MTcwNjg2MDgwMDAwMH0=
```

#### 图片发送

**JSON 原文**:
```json
{
  "type": "media",
  "version": "1.0",
  "mediaType": "image",
  "source": "local",
  "path": "/tmp/screenshot.png",
  "caption": "这是截图",
  "timestamp": 1706860800000
}
```

**编码后**:
```
QQBOT_PAYLOAD:eyJ0eXBlIjoibWVkaWEiLCJ2ZXJzaW9uIjoiMS4wIiwibWVkaWFUeXBlIjoiaW1hZ2UiLCJzb3VyY2UiOiJsb2NhbCIsInBhdGgiOiIvdG1wL3NjcmVlbnNob3QucG5nIiwiY2FwdGlvbiI6Iui/meaYr+aIquWbviIsInRpbWVzdGFtcCI6MTcwNjg2MDgwMDAwMH0=
```

### 实际可获得字段说明

根据 QQBot API，以下是各消息类型中**实际可获得**的字段：

| 消息类型 | 可获得字段 | 不可获得字段 |
|---------|-----------|------------|
| C2C 消息 | `author.user_openid`, `content`, `id`, `timestamp` | 用户昵称 |
| 群消息 | `author.member_openid`, `group_openid`, `content`, `id`, `timestamp` | 用户昵称 |
| 频道消息 | `author.id`, `author.username`, `member.nick`, `channel_id`, `guild_id` | - |

> ⚠️ **注意**：C2C（私聊）消息中无法获取用户昵称，因此 `senderName` 字段已从载荷设计中移除。

### 处理流程

```
┌────────────────────────────────────────────────────────────┐
│                    收到 AI 输出消息                         │
└──────────────────────────┬─────────────────────────────────┘
                           ↓
┌────────────────────────────────────────────────────────────┐
│ 检测消息是否以 "QQBOT_PAYLOAD:" 开头                        │
└──────────────────────────┬─────────────────────────────────┘
                           ↓
            ┌──────────────┴──────────────┐
            │                             │
           是                            否
            ↓                             ↓
┌───────────────────┐          ┌───────────────────┐
│ 解析 Base64 + JSON │          │ 作为普通文本处理   │
└─────────┬─────────┘          └───────────────────┘
          ↓
┌───────────────────┐
│ 验证 type + version│
└─────────┬─────────┘
          ↓
    ┌─────┴─────┐
    │           │
  成功        失败
    ↓           ↓
┌──────────────────┐  ┌───────────────────┐
│ 根据 type 分发    │  │ 记录错误日志      │
│ - cron_reminder  │  └───────────────────┘
│ - media          │
└────────┬─────────┘
         ↓
┌───────────────────┐
│ 调用对应处理器     │
│ 发送消息给用户     │
└───────────────────┘
```

---

## 需要删除的旧代码

以下是需要从 `gateway.ts` 中移除的旧代码：

1. **`collectImageUrl()` 函数** - Markdown 图片语法解析
2. **`filterInternalMarkers()` 函数** - 内部标签过滤
3. **相关正则表达式** - 如 `/!\[.*?\]\((.*?)\)/g` 等
4. **旧的图片处理逻辑** - `classifyImageSources()` 等相关代码

---

## 边界情况与约束

1. **消息大小限制**：Base64 编码会增加约 33% 的数据量，需确保编码后的消息不超过限制
2. **特殊字符处理**：`content` 和 `caption` 字段可能包含 emoji 和换行符，需确保 JSON 序列化正确
3. **时区一致性**：`timestamp` 使用 UTC 毫秒数，避免时区问题
4. **安全性**：Base64 不是加密，敏感信息不应放在 payload 中
5. **图片大小**：本地图片建议不超过 10MB，超大文件可能导致 Base64 编码后内存占用过高
6. **版本兼容**：`version` 字段用于未来格式升级时的兼容性处理
7. **字段精简**：只包含实际可获得的字段，避免 AI 产生幻觉数据

---

## 成功标准

1. ✅ 定时提醒触发后，用户能在 5 秒内收到提醒消息
2. ✅ 提醒内容与创建时设置的 `content` 完全一致
3. ✅ 不再出现 `HEARTBEAT_OK` 等无意义响应
4. ✅ 图片发送使用新的结构化载荷格式
5. ✅ AI 输出中不再出现泄露的内部标签（如 `[[reply_to:...]]`）
6. ✅ 旧的 Markdown 图片解析代码已完全移除
7. ✅ 日志能清晰追踪消息的创建、解析、发送全过程
8. ✅ SKILL.md 文档已更新为新格式
9. ✅ 载荷中不包含无法获取的字段，避免 AI 产生幻觉数据
