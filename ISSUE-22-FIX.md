# Issue #22 修复说明（QQ 主动私聊会话类型错判）

## 问题现象
当模型在 QQ 会话里主动给另一个 QQ 号发消息时，消息可以成功发到私聊小窗，但会话被记录为 `group` 前缀，导致后续回复落到 `direct` 会话，出现上下文断裂。

原始 issue：
- https://github.com/constansino/openclaw_qq/issues/22

## 根因
QQ 目标写成纯数字时（例如 `7440`），上游路由容易把目标当成“群类型”来建会话键；而 QQ 插件发送层默认把无 `group:` 前缀目标按私聊发送。

结果就是：
- 发送层：按私聊发出（成功）
- 会话层：按 group 建键（错误）

## 本次修复
已在 `src/channel.ts` 做以下改动：

1. 统一目标标准化：
- 纯数字、`user:`/`u:`/`dm:`/`direct:` 统一规范为 `user:<QQ号>`
- `group:<群号>` 保持群目标
- `guild:<guildId>:<channelId>` 保持频道目标

2. 发送层支持 `user:` 私聊目标：
- `sendOneBotMessageWithAck()` 新增私聊目标解析，避免 `parseInt("user:123")` 失败。

3. 收紧目标识别规则与提示：
- `looksLikeId` 基于规范化结果识别 `user:`/`group:`/`guild:`
- 更新 hint 文案，明确“不要只写纯数字”
- 新增 `agentPrompt.messageToolHints`，明确要求：
  - 私聊：`user:<QQ号>`
  - 群聊：`group:<群号>`
  - 频道：`guild:<guildId>:<channelId>`

## 使用建议
后续模型调用消息工具时，请使用显式前缀目标，避免歧义：

- 私聊：`user:123456789`
- 群聊：`group:987654321`
- 频道：`guild:guild_id:channel_id`

## 影响
- 不改变既有群聊/频道发送行为。
- 私聊目标解析更稳健。
- 可显著降低“私聊发出但会话建成 group”的概率，避免对话断裂。
