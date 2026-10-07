# 踩坑陷阱:来龙去脉与实测证据

[CLAUDE.md](../CLAUDE.md)「高频踩坑陷阱」每条只留红线与真相源,这里是同名条目的长版:为什么这样定、实测看到了什么、怎么复跑。标题与 CLAUDE.md 一一对应,改标题两处同步。证据数据以代码头注释与守卫测试为准;这里只保留代码里没有归宿的部分(`test-results/` 不进仓库)。

## 请求转换 · 工具(Claude→Kiro)

### 注入文本

Kiro `conversationState` 只有 user/assistant,`userInputMessageContext.additionalContext`(Smithy 模型里唯一像上下文通道的结构化字段)上游 200 但**静默丢弃**(2026-09-10 直连实测:塞进去的秘密码模型答 UNKNOWN、指令被无视、input token 不变)。所以 system 文本 = `buildSystemPrefix`(客户端 system + 可选身份指令,不含任何 thinking 前缀)→ `foldSystemIntoFirstUserMessage` 在 **Kiro 消息层**拼到首条 user 消息最前(首轮 = currentMessage,之后 = history 首条 user;图例之后拼,字符串 / 块数组 content 字节一致);后续 user 轮次(含纯 tool_result)原文不动;末尾 user 连串整体 = 当前轮(Anthropic 语义),history 必以 assistant 收尾。

**网关不造合成轮次**:不加开场 `user: system / assistant: "I will follow these instructions."`(kiro-cli 自己的形态),也不在末尾补 `assistant: "OK"`。前者会被模型当真实历史逐字引用(零注入基线答 NONE),后者让同一段对话在下一轮被 `mergeUserMessages` 合并、形态随轮次漂移。长上下文 + 真实工具执行的 A/B 证明两者零收益,数据在 `foldSystemIntoFirstUserMessage` / `buildHistory` 头注释。**身份覆写与注入方式无关地不可靠**(user 级权重压不过上游系统提示)→ `KIRO2CLAUDE_IDENTITY_OVERRIDE` 默认关,数据在 `IDENTITY_OVERRIDE_DIRECTIVE` 头注释。

**中途插入的内容**(2026-09-11 专项):录得的 5923 条真实 Claude Code 请求里 2023 条含 `role:system` 消息,**永远紧跟一条 user 之后**(中途 3730 条是字符串、收尾 176 条是文本块数组;内容是权限模式指令 / 「文件已在磁盘上变更」/ `<total_tokens>`)。只走 `foldSystemMessages` 折进相邻 user 轮,重放全部零丢失、收尾的落 currentMessage、中途的落对应 history user;7 种形态(收尾 / 中途 system、tool_result 后排队的用户文本、ESC 打断工具 / 文本、首条 reminder、assistant 起手)真实上游 nonce 7/7 回,**assistant 起手的 history 上游接受**(Kiro 不要求 user 起手)。同根的三条规则:① system 前缀在 **Kiro 消息层**拼,不在 ClaudeMessage 层——在后者折,字符串 content 得 `SYS\n\nx` 而块数组只得 `SYS\nx`(CC / Codex 发的都是块数组),≥2 图时图例还会排到 system 前面;② 非 user/assistant/system 的 role → **400 `InvalidRole`**,不静默丢——静默丢就是丢用户内容,还会让 2.5 步的 continuation 文案融进用户自己的末条消息;③ OpenAI Chat / Responses 只把**开头**那段 system/developer 提升成 system,对话开始后出现的原位保留为 `role:system`——一律提到开头会让「从现在起…」类中途指令读起来像开场规则。复跑 `test/manual/replay-content-preservation.ts`(免费)/ `inserted-content-live.mjs`(计费)。

查 wire 字段名看 `aws/amazon-q-developer-cli` 的 Smithy 客户端类型,别对 kiro-cli 二进制 `strings`(连 `toolResults` 都搜不到)。

Smithy 新命名空间 `com.amazon.kiro.runtimeservice` 有顶层 `systemPrompt` 字段,但两个 target 直打都 400 `REQUEST_BODY_INVALID`,服务端未开;证据与 KAS 的做法见「原生 reasoning / effort / system 的 wire 真相」。

**2026-09-11 真实 Docker CLI 验收**(CC 2.1.263 / Codex 0.153.4):`tools/claude-code/test.sh` 8/8;CC 三阶段真实编码 107 次调用三段验收全过;Codex 三阶段 35 次调用全过;AskUserQuestion、WebSearch、WebFetch、`/simplify`、`/code-review` 全过;合成探针 CC 10 场景 / Codex 11 / 空响应 9+9 / subagent 生命周期 8 与 09-07 基线逐项一致;CC 六图 2/2 全对,Codex 六图归属抖动经受控 A/B 证明是模型侧既有问题(改前 0/3、改后 3/5,图片轮 wire 逐字节相同)。复跑要点:长会话阶段超时放到 60 分钟(opus-5 extend 约 47 次调用,上游慢时 20/30 分钟窗口的超时不是转换层问题);headless 验 AskUserQuestion 走 `--permission-prompt-tool stdio` 的 `can_use_tool` 控制消息;CC headless 的 init 工具清单本身不含 Glob / Grep / TodoWrite,网关照收全转发,别当网关问题。

### 工具调用文本泄漏

上游解析偶发失败 → 工具调用块以纯文本掉进响应,留在历史会被模型模仿 → 同会话确定性复发。`KIRO2CLAUDE_TOOL_CALL_TEXT_RESCUE`(默认开)双向兜底:响应侧解析回真 tool_use、请求侧剥历史泄漏块。全部红线在 `claude/tool-call-text.ts` 头注释。别用「大文件分块写入」类 prompt 指令治它(已证伪)。

### tool description cap

Kiro 对**单个** description 无字符硬上限,真限制是 **context window**(多 tool + history + system 撑爆报 400 "Context window is full")。`KIRO2CLAUDE_TOOL_DESCRIPTION_MAX_LEN`(默认 32768)截住畸形超大 description——覆盖已知最大合法工具 Workflow 且留余量;cap 只管单工具,总量保护交给 Kiro 的 400。

### 多图归属只靠顺序

Kiro wire 只有消息级 `images[]`——`toolResults[].content` 塞 Bedrock 风格 `{image}` 上游 200 但**静默丢弃**(2026-09-09 直连实测:模型说结果为空、输入 token 恰好少掉图片量),user 正文是纯字符串也放不进图。所以 tool_result 里的图提升到 `images[]` 后,归属**只剩位置**。三层真实上游实测(`multi-image-attribution-probe.mjs` + `multi-image-cli-probe.mjs`):

1. 两次并行图片工具、回执**反序**时 Claude opus-5 与 GPT-5.6 都按 **tool_use 顺序**对应 `images[i]`,答案整体对调 → `canonicalizeToolResultOrder` 按 tool_use 顺序重排 tool_result 块(只动 tool_result、只在已占槽位间动;分组 = 「连续 user 连串是一轮」,history 里的连串经 `mergeUserMessages` 合并,末尾连串整体是 currentMessage,同走 `processMessageRun`)。
2. 6 张图直接放 user 消息、正文写「附件 1…6 按序」两模型全对 → 上游 N=6 仍保序、模型数得清。
3. **6 个各含一张图的 tool_result**(Claude Code 并行 Read 形态,id 不透明、路径只在 tool_use 输入里)只靠 tool_result 里的序号占位符两模型 4/4 错位,Docker 真 CC 5/6 错、Codex 单次 exec 看 6 图 3/7 错;同一 wire 在 `content` 开头加一行图例后 4/4 全对 → `prependImageLegend`:≥2 张图且至少一张来自 tool_result 时,`content` 前置 `[Attached images, in order: image k = <图前最近一行文本 (tool call id)> | result of tool call <id> (<name> <input≤120>); …]`,tool_result 内占位符同步为 `[image k attached to this message]`。

图例只复述 wire 上已有的事实(序号、tool_use id、调用输入、工具自己打印在图前的路径),**不是指令**;单图、或图全来自用户时不加。别给所有 ≥2 图请求塞 `content index` / `source` 元数据或「不是拼块」类指令:那想修的「GPT 把两张相同图数成 1 张」是模型判断(token 计数证明两张都送到了),记在 README「已知限制」。OCR 误读(GPT 对 5×7 点阵 3↔2 / 0↔6 / 5↔6)是模型噪声,探针单独分类、不计失败。

## 流式传输 · 断连 · 空流

### 空流有界重试

上游偶发「200 OK + 零内容帧」,客户端无法与真实过载区分,retry-executor 看不到 2xx 的 event-stream body。**仅 pre-commit**(未写任何字节)对同一请求重发最多 `KIRO2CLAUDE_EMPTY_STREAM_RETRIES`(默认 2)次,已 commit 绝不重试。**确定性空流单次定案、不耗重试预算**(重发只会同样失败、白烧 credit),四类:`max_tokens` / `model_context_window_exceeded` / 截断 tool_use(宣告 tool_use 却无一帧 `isComplete`)/ 上游 Error·Exception 帧**且已开工**。末类限定词必须:零帧拒绝(未开工)属**瞬时**故障、走有界重试。判据用 `sawBillableWork()` 而**不是** `hasContent()`——GPT 加密 reasoning 计费但不 surface,`hasContent()` 会把烧了数千帧 reasoning 的流谎报为空;与 retryable 分类无关(那个集合实测不完整)。文案 `selectEmptyUpstreamMessage` 的 `deterministic` 参数必须显式传,别靠 `emptyAttempts` 倒推。新增判空分支先问「是不是内容绑定的」,是则加进排除列表。红线在 `stream-handler.ts` / `non-stream-handler.ts` / `stream.ts`(`sawCompletedToolUse`)/ `empty-capture.ts` 头注释;不明空流用 `KIRO2CLAUDE_CAPTURE_EMPTY_DIR` 抓包,别盲改 converter。

### 截断 tool_use 必须阻止残缺调用到达客户端

上游偶发宣告工具并发出 input 分片,却未发 `isComplete` 就断流。**只改 `stop_reason=max_tokens` 不够**——2026-09 用无缓存重建的 Claude Code 2.1.263 实测:客户端**先**解析已关闭的工具块、**后**才看终态,仍报 `InputValidationError: JSON parse failed`。故防线必须落在**协议时序**上:`stream.ts` 的 `pendingToolCalls` 按 id 缓存参数,**收到 `isComplete` 才分配 block index 并原样发 start/delta/stop**;文本继续实时流式、参数不修补,未完成调用不上 wire 也不占索引,交错调用按完成顺序发出且参数隔离。非流式 `non-stream-reduce.ts` 同样只保留完成调用。

**参数只有两种合法归宿**:普通工具经 `parseCompletedToolInput` 必须解出 JSON 对象,失败即协议错误——**绝不回退 `{}` 去执行**(那是静默换一个动作);只有 Responses 显式 `customToolNames` 的 raw-input allowlist 保留裸文本,其 wrapper/raw 解码仍由 `freeform-tool.ts` 独占。同族的 wire 校验在解码边界(`kiro/model/events/base.ts`):name/id/input 的类型、以及 `isComplete` 必须是 boolean——字符串 `"false"` 是 truthy,会把未收完的参数当成可执行调用放出去;`ToolUseSequence` 被两个归约共用,拒绝中途改名与完成后复用 id。计费字节仍累计;终态分两形态:零 input 空壳 → 确定性空流,由 `hasIncompleteToolUse()` 识别、单次定案不耗重试;已有 input 或其它内容 → 丢弃未完成调用并保留部分文本,终态 `max_tokens`(不覆盖更具体的既有终态)。Responses 流式/非流式把 `max_tokens` 与 context-window 耗尽映射为 `incomplete` + `max_output_tokens`,**不可改成 `completed`**。真实客户端复测:Claude Code 2.1.263 不再产残缺 tool_use、能续接恢复;Codex 0.153.4 对 incomplete 自动重连并恢复,不执行损坏的 exec。网关自己 abort/destroy 造成的截断由 `ctx.gatewayTruncatedUpstream` 打标并降 info。

### 断连计费

客户端断连后默认 drain 上游到 EOF 拿尾帧 Metering **全额计费**。`KIRO2CLAUDE_ABORT_UPSTREAM_ON_DISCONNECT`(默认 false)开启后断连**主动 abort 上游**(signal 经 provider→retry-executor `axiosConfig` 透传)省下断连点后的 credit;代价是拿不到 Metering、per-request 记账偏低。仅 Claude 端 stream;`logFields.drained_after_disconnect` 观测。该 flag 与 `metering_lost` 的关系是相反的:它消除前者、却让后者在每次断连时为真(见 `isMeteringLost` 头注释的口径偏差)。

### write 背压不是断连

`stream.write()` 返 `false` = 缓冲超 highWaterMark、应等 `'drain'`,socket 健康。误判会停读循环(对**活着的**客户端)、丢终结段 `message_stop`、上游仍 drain 到 EOF **全额计费**、日志还错记客户端;且因缓冲由**大量字节**填满,专咬最长最贵的响应。红线:存活只看 `destroyed`/`writableEnded`/write 抛错;背压走 `awaitDrain`(带 `close`+超时兜底);`disconnect_source` 区分 `client_close`/`write_failed`,**别退回单一 `aborted` 布尔**。真相源 `safeWrite`/`awaitDrain` 头注释。

### legacy thinking 文法

`<thinking>` 是 prompt 诱导出来的**文本内**协议,不是独立 event(产生式见 `claude/stream/legacy-thinking-decoder.ts` 头注释)。识别范围恰好卡在**行首**,两边都试错过:**收紧成「只认响应开头」→ 幻影执行**(模型写句前言再开思考,整段被判可见文本,思考里起草的 `<invoke>` 被救援物化成真 tool_use);**放宽成「任意位置」→ 整个响应变 thinking**(正文里内联提到标签就误开块,严格闭标签文法找不到 `\n\n` 就一路吞到 EOF)。同族三条:① 绝不按标点/引号包裹去猜是真标签还是模型在引用它——反例:按闭标签**前一个字符**查 30 字符黑名单,思考以英文句号收尾就被否决闭合(中文 `。` 不在表里,所以只测中文发现不了);② **未闭合的块在 EOF 仍归 thinking**,理由同幻影执行;③ **语法同源还不够,终态判定也必须同源**——实测踩过:流式判「thinking 阶段**开过**」、非流式跟着「thinking 内容非空」走,空块 `<thinking></thinking>\n\n` 于是在流式是 200 + `max_tokens`、在非流式先烧完重试预算再 503。新增任何 thinking 相关分支,先把两条路径对拍一遍。

### 原生 reasoning 的空帧

任意 `reasoningContentEvent`(含空/redacted)都锁 native 模式——那是 GPT 静态判定万一漏判模型别名时的运行时兜底。但空帧两条边界不能越:① 不打断已经开着的 thinking 块(`hasOpenThinking`),强行关块会把剩下的私有推理连同字面 `</thinking>` 推进可见文本;② 不 flush 泄漏工具调用的救援检测器,那只在真要开 thinking block 时才需要(保 wire order),在空帧上做会把跨帧候选拦腰截断——而 GPT 的推理帧在这里一律按空帧处理(原文交给 `OpaqueReasoningAccumulator`),受害的恰恰是 GPT 响应。真相源 `processReasoningContent` 头注释。

### 上游杀卡住的流

上游偶发生成中途发泛化 `Exception`(`code:"error"`、**无** `ContextUsage`+`Metering` 尾帧 = 真中途死),已 commit 只能转 in-band `error`,客户端见 mid-response 截断。判别子是**产出速率(token/s)不是总时长**——按时长分桶会得错误死线。网关侧无治本手段(上游行为)、post-commit 也无法重试或改状态码。缓解见 `stream-handler.ts` 的 `armDrainGrace`(目前只在**已断连**时武装,连接中 idle 无上界、只受 axios 720s 约束)。

### 读坏的 body 是失败的响应

三类损坏必须走与显式上游错误帧**同一条**终结路径,只记日志再补一个成功终态等于把故障洗成 `end_turn`:① 读流异常(`ECONNRESET` / `ERR_STREAM_PREMATURE_CLOSE`)经 `StreamContext.recordStreamReadError`(网关主动取消除外,那是自伤);② 帧 CRC 或已知事件的 JSON 解码失败;③ **HTTP 正常 EOF 也不保证 event-stream 完整**——`EventStreamDecoder.assertComplete()` 查残留半帧,末尾攒着不足一帧的字节就是截断。三类都仍继续 drain 以取 Metering,但**错误一旦确定就不再发出后续工具调用**。边界同样重要,别把「没见到」当**故障**:Unknown 事件、任意分片切法、没有 Metering 尾帧都不构成损坏(Metering 缺失只是漏账,见 `isMeteringLost`);「没见到 `metadataEvent`」是**未完成**而非故障(见下一条);已知事件只校验**正在消费的**字段类型(string / finite number),缺字段与原有 null 缺省保持兼容。`bodyReadFailed` 是单向标志——它记的是**已证明**的损坏,后到的显式错误帧覆盖 code/message 也清不掉它;`canRetryZeroWorkRejection` 据此只放行「无损坏 + 零计费工作」的显式拒绝去重试,**损坏的 body 不可当成空流**来蹭重试预算。

### 帧边界 EOF

上游偶发生成中途干净断流,EOF 恰落在帧边界——`assertComplete()` 无半帧可查、HTTP 正常收尾;若据此发 `end_turn`,真实 Claude Code 2.1.263 把半句话当任务完成,12 步做 4 步就 `exit 0`、`is_error:false`,后续两轮也不会补(2026-09-07 两种终态各实跑一次对照)。可判信号只有一个:352 条真实响应帧审计 351 条以 `metadataEvent → contextUsageEvent → meteringEvent` 收尾(Claude 与 GPT-5.6 皆然),唯一缺它的恰是 reasoning 中途 EOF;2026-09-08 两款 CLI 的真实长链会话又录到 376 条,372 条带尾帧,缺的 4 条全是真实故障或客户端断连(`ECONNRESET` / 上游 `Exception` 帧 / 探针超时掐断),**没有一条是干净完成**,即零误判。**kiro-cli 2.21.1 自己不做这个判定**(`PROBE_STREAM_SHAPE=text-eof` 实测:只发正文就 EOF 它照样 `exit 0` 打印半句、不重试,只少一行 Credits),网关有意比官方客户端严格——Claude Code / Codex 对 `max_tokens`/`incomplete` 会续接,对 `end_turn` 只会当任务完成。故解码层把它解成已知事件(`Metadata`),两条归约同源:**有内容 + 无错误 + 无尾帧 → `max_tokens`**(Responses → `incomplete`/`max_output_tokens`)。三条红线:① 只取「出现过」,**不用它的 `stopReason`**(带工具的响应里 124/325 报 END_TURN,终态仍由网关推断);② 零内容不进这里(仍归判空 + 有界重试),显式错误帧与读流损坏走 in-band error,网关自伤降 info;③ 已完成的 tool_use 无尾帧也报 `max_tokens` 而非 `tool_use`——后面可能还有没到的兄弟调用。`stream completed` / `openai stream completed` 日志的 `stop_reason` 必须取终结段**之后**的值(否则 wire 发 `max_tokens`、日志记 `tool_use`)。测试 fixture 里的「正常完成」**必须**带 `buildMetadataFrame()`(`framesWithMetering` 已含),不带 = 在测截断;手工服务器同样用 `test/helpers/event-stream.ts` 的 `completedFrames()` / `buildMetadataFrame()` 收尾(漏掉尾帧的症状:Codex 对每条回复 `max_output_tokens` 重连 5 次后 `turn.failed`,整套矩阵假阴性)。真实 CLI 复跑 `conversation-fault-server.ts` + `claude-conversation-probe.mjs`(`K2C_PROBE_SCENARIOS=text-eof-once`)。

## 多模型 · GPT · OpenAI · Codex

### GPT 完全相同上游

请求体逐字段相同,唯一差异 `modelId`(外加按模型 schema 生成的顶层 `additionalModelRequestFields`,GPT 为 `{reasoning:{effort}}`)——支持 GPT = `mapModel` 加分支即两端可用,无需新上游适配。响应侧唯一真差异:GPT reasoning 走**同名** `reasoningContentEvent`,V3 target 下 payload 是 `{text:"...", signature}`(文本只是占位,推理在签名的密文里;`{redactedContent}` 形态仍接收)。GPT 的推理帧因此不进 `processReasoningContent` 的 thinking 通道,只由 `OpaqueReasoningAccumulator` 原样保留供回传(见「推理往返」);「只有 GPT 推理」的响应仍按空流处理(守卫 `empty-response-contract` 的 encrypted reasoning only)。**「见过原生帧」与「原生帧有内容可 surface」是两件事,`processReasoningContent` 必须分开记**:前者(含空 / GPT 帧)决定锁 native 模式、关掉 legacy decoder,后者才决定开 thinking content block。合并是二选一的错——只留后者则 GPT 帧不锁模式,可见输出里的字面 `<thinking>` 会被 legacy 解码剥走;只留前者则开一个永远空的 thinking 块。`metadataEvent` 的 `stopReason` 故意不用,终态由网关推断(工具调用时 `tool_use` 比上游 `END_TURN` 准;见「帧边界 EOF」)。

### GPT context window 随上游漂移

`usage.input_tokens` 不是上游直接给的,是网关拿 `contextUsageEvent.contextUsagePercentage` 乘 `getContextWindowSize()` 反推的。**上游改窗口不报错、只缩放**:Kiro 2026-09-14 把 GPT-5.6 升到 1M(`kiro-cli chat --list-models` 写 "1M context window"),网关仍按 272K 算就整体低报 3.68 倍。

后果直通计费:Kiro 对 >272K 的请求**整条**按双倍档计(sol 4.4x→8.8x)。实测 gpt-5.6-luna 单请求 250,338 token 记 18.25 credit/M、301,919 token 记 36.50,恰好 2.0 倍(测量时每请求随机 conversationId,无缓存折扣干扰,见「会话身份映射到 kiro-cli」)。低报时客户端以为还有余量,真实上下文养到 ~95 万,整段会话每条都落双倍档,单请求可达正常档的数十倍。双倍档本身没有别的缓解。

**为什么 1M 是对的**:Codex 不读网关上报的窗口,只拿 `input_tokens` 比自己内置的常量——Codex 0.154 对 gpt-5.6-sol 内置 272000 再乘 0.95 保留系数 = 258,400(session rollout 的 `model_context_window` 可查)。上报准确时它在真实 258.4K 就压缩,**恰停在双倍线下方,余量 13,600**;低报 3.68 倍时同一个 258.4K 对应真实 950,912,于是整段双倍。★ 这个余量依赖客户端那个常量不变:**客户端若跟进抬到 1M,余量立刻消失**,届时须在客户端侧 pin `model_context_window`。

Kiro 逐账号灰度,还停在 272K 的账号把 `KIRO2CLAUDE_GPT_CONTEXT_WINDOW` 设回 272000(反过来高报会让客户端在 7 万就压缩)。判断口径:拿一段已知长度文本发 gpt-5.6-luna(最便宜),`input_tokens` 与 `count_tokens` 差 ~3.7 倍即窗口不匹配。守卫 `test/claude/reasoning-native.test.ts`;`exceeded`(`model_context_window_exceeded`)只看百分比是否到 100,与此常量无关。

### OpenAI prompt_tokens

`buildClaudeUsagePayload` 会应用 derived 插件的 `input_tokens` 覆写(缓存拆分语义),而 OpenAI `prompt_tokens` 是**输入总量(含缓存)**。故 `openai/` usage 必须直接读 reducer 原始 `contextInputTokens ?? inputTokens` 与 `outputTokens`、绕过 `buildClaudeUsagePayload`;计费 hook 仍跑,`addExtension` 的 `kiro_*` 扩展照常并入(经 `resolvePluginUsageExtensions`,`/api/*` 镜像剥掉)。插件覆写只取 `cache_read_input_tokens` 一项(`resolveCacheReadTokens`,夹到 `[0, prompt_tokens]`——插件契约只保证有限数),映射成 OpenAI 的 `cached_tokens`——它在 OpenAI 语义里本来就是 `prompt_tokens` 的子集,与 derived 的恒等式 `input + cache_read == 总量` 同口径;`input_tokens` 覆写仍不套。早先连这一项也不取,GPT 走 OpenAI 协议时 derived 反演出的命中全部丢在网关里,Codex 看到的缓存恒为 0。

### Codex 只说 Responses

`wire_api=chat` 在 Codex 0.122+ 移除,必须走 `/openai/v1/responses`(请求 `input` items + 扁平 tools,响应严格语义事件序列)。编码器红线全在 `openai/responses/response-stream.ts` 头注释(`content_part.added` 先于 `output_text.delta`、done 回填全文、纯工具调用不产空 message、thinking → reasoning summary 惰性开),改编码器前先跑真实 Codex(`tools/codex/`)。推理的 `encrypted_content` 往返见「推理往返」。

### 会话身份映射到 kiro-cli

GPT 的缓存折扣按 `conversationId` 给(Claude 按内容寻址、与 id 无关,见下「缓存作用域」),而且只在 credit 上体现:usage 里 `cache_read` / `cache_creation` 恒为 0,但同一 id 下后续轮次明显更便宜。2026-09-23 用 Docker Codex 0.156.1 + gpt-5.6-sol 跑同一个 5 步编码任务(A/B/A/B):每请求随机 id 两轮共 8.09 / 8.91 credit,每轮都按冷价(约 0.078 credit/1K input);从 `prompt_cache_key` 派生稳定 id 后是 2.59 / 2.56,首轮 0.84,之后每轮 0.12–0.27。kiro-cli 2.23.1 整个会话都用同一个 id,曲线与稳定组一致。

网关按 kiro-cli V3(KAS)的会话形态映射,不自创规则(真相源 `resolveConversationIdentity`,`claude/converter.ts`)。2.23.1 抓包:

| 字段 | KAS 的形态 |
|---|---|
| `conversationId` | 顶层会话 `sess_<uuid>`,一个会话一个,`--resume` 不变;子会话(subagent、标题生成)是裸 UUID |
| `rootConversationId` | 主会话等于自己;subagent 指向父会话 |
| `agentContinuationId` | 一个用户轮次一个,轮内工具往返不变(同进程第 2 轮与 `--resume` 都换新);子 agent 跑完、结果回到父会话时也不变 |
| 顶层 `agentMode` | 主会话 `vibe`;`invoke_sub_agent` 的子会话为子 agent 的模式(`general-task-execution`) |
| subagent | 独立会话:自己的 id 与 acid + 上面的 root / agentMode;结果以父会话 `invoke_sub_agent` 的 tool result 返回 |

- 客户端会话键:Chat / Responses 读 `prompt_cache_key`,Messages 读 `metadata.user_id` 里的 session;都没有时两个 id 每请求随机。轮次由消息结构推出(`userTurnKey`:不含 tool_result 的 user 连串算一轮,Claude Code 夹在 tool_result 里的 reminder 不算;键 = 序号 + 这一轮的输入文本,上下文压缩让序号回落时不会复用旧轮次的 id)。录得的 5923 条 Claude Code 请求重放:acid 切换 358 次全在用户新输入上,工具循环中途 0 次。
- **Codex subagent 映射成 KAS 的 subagent 会话**:父子共用 `prompt_cache_key`,线程身份只在 `thread-id` 头里(根线程等于 key)。与 key 不同的 `thread-id` 按 subagent 会话映射(`responsesSession`),上游照收子会话的 root / agentMode。子线程首个请求因此冷启动(实测 0.79,与 KAS 原生 0.87 同量级),之后命中自己的缓存,父线程缓存不受影响。别为了省这次冷启动让父子共用 id:那会偏离 kiro-cli 的会话形态。
- **缓存作用域**(2026-09-23 直打,前缀约 1.4 万 token):GPT(luna)同一 id 下 A1 之后插入 1 段 / 5 段别的对话、或一个小旁路请求,A2 仍按 0.1× 命中(0.026 对冷 0.249);插进来的对话还共享约 1.5K token 公共前缀(冷价 0.224);换 id 完全不命中;acid 相撞无影响。Claude(opus-5)同 id、跨 id 都命中(0.087 对冷 0.160),缓存与 id 无关。所以:Chat 的 `prompt_cache_key` 按 OpenAI 语义当缓存分组直接用,多用户共用一个 key 不互相挤缓存,不加指纹;Claude Code 的旁路请求(标题生成等)与主线程共用 session id,对缓存无影响。
- **Claude Code subagent 同样映射成 KAS 的 subagent 会话**:2.1.280 抓包,subagent 与主线程共用 `metadata.user_id` 的 session 和 `x-claude-code-session-id`,只多带 `x-claude-code-agent-id`(同一 subagent 的多个请求值不变,主线程不带;system 的 billing 行另有 `cc_is_subagent=true`)。网关按它映射(会话键 = session + agent-id,root 指主线程),主线程与 OpenAI 的键走同一条派生路径,客户端原始 session id 不上送。真实 Claude Code + Task 工具端到端:subagent 独立 id、root 指主线程、`general-task-execution`,工具循环内 acid 不变。
- **Codex 的 `<subagent_notification>` 不开新轮次**:子 agent 完成后,Codex 0.156.1 在父线程紧跟 `wait` 的工具输出插一条只含 `<subagent_notification>…</subagent_notification>` 的 user 消息,按通用规则会被当成新的用户输入、父会话 acid 中途换新。kiro-cli 里同一时刻是 tool result、acid 不变,所以 Responses 适配层把它并进前一条工具结果消息(只认「整条都是通知 + 前一条是工具输出」,Kiro 层两者本就合并上送,模型所见不变);这条 Codex 约定不进 Messages / Chat 共用的轮次规则。守卫 `test/openai/responses/subagent-notification.test.ts`(含 Messages / Chat 不受影响的反向用例)。
- kiro-cli 把会话标题生成当成独立子会话(agentMode `session-title`、root 指父会话);Claude Code 的同类旁路请求没有可靠标记,网关不做这层映射(对缓存无影响,见上)。子会话历史以 KAS 自造的「system + `I will follow these instructions.`」开头,网关有意不照抄(见「注入文本」)。
- **会话隔离不靠 id**:上游不按 conversationId 存历史。同一个 id 下开一段全新会话问 A 里埋的暗号,luna / sol / sonnet-5 共 7 次都答 NONE,带历史的正对照 7/7 答对。**并发在途也不串**(2026-09-23,`session-concurrency-live.mjs`):四组(Messages × opus-5 / luna、Chat、Responses + Codex 子线程)各自共用一个会话键,组内埋暗号、问回、不带历史的新对话与 subagent 全部同时在途,47/47:各自只答出自己的暗号,新对话与 subagent 全答 NONE。
- 复跑:`test/manual/session-isolation-live.mjs`、`session-concurrency-live.mjs`、`cache-scope-probe.ts`;守卫 `test/claude/converter-conversation-identity.test.ts` + `test/openai/responses/reasoning-roundtrip.test.ts`。

### 推理往返

KAS 每轮都把上一轮推理放回 history 的 `reasoningContent`,GPT 也是 `{reasoningText:{text:"...",signature}}`。网关把 Kiro 形态装进信封(`claude/reasoning-envelope.ts`),经客户端能原样带回的不透明通道往返,下一轮拆开还原:

- **Responses**:客户端声明 `include:["reasoning.encrypted_content"]` 时放在 reasoning item 的 `encrypted_content`(Codex 默认声明);Claude 的签名推理同样走这里。
- **Messages**:GPT 的推理以 `redacted_thinking` 块下发(data = 信封),converter 解开还原;外来的 `redacted_thinking` 原样透传,其它模型签发的丢弃。Claude Code 2.1.280 实测原样带回。Responses 带回的信封只校验计数,原样转成 `redacted_thinking` 交给同一个 converter——还原成 `reasoningContent` 只有这一处。
- **Chat**:没有对应通道,不回传。
- **GPT 的推理帧在 tool_use 之后、响应末尾才到**,所以它的块 / item 只能排在已发内容之后(流式收尾追加,非流式同序);回程靠 `mergeAssistantMessages` 与同一条 assistant 合并。
- 信封绑定上游 modelId,换模型后旧推理不回传;认不出的密文(真 OpenAI 的、被改坏的、`k2c.` 前缀但版本不认得的)一律丢弃。信封是 `k2c.r2.` + JSON,不再套 base64(签名本身就是 base64)。网关不存推理状态,不可能跨会话串。
- 实测(V3 target 直打):GPT 的签名上游不校验(改坏签名、改文本、换成 `{redactedContent}` 形态都 200),回传与否 credit 也看不出差别;Claude 签名仍严格校验(失效走剥离重试)。所以 GPT 回传的意义是对齐 KAS,不是省钱。

### Codex code mode

跨版本实测一致(版本号见 `tools/codex/README.md`):Codex 按模型名走**两套请求形态**。**认识**的名字(`gpt-5.6-sol`)→ code mode:顶层 `tools` 与 `instructions` **双双不存在**,工具改由 `input[0]` 的 `{type:"additional_tools"}` item 携带,含 `type:"custom"` 的 freeform 工具;**不认识**的名字(`gpt-5-codex`/`o3`/`sol`)→ 打 `Model metadata not found` 后 fallback 到标准顶层 `tools`。判别只看**字段在不在**,别按模型名分支;两套形态都要继续支持,真实抓包 fixture 在 `test/fixtures/responses/codex-code-mode-request.json`(扁平)+ `codex-code-mode-namespaced-request.json`(namespace 嵌套)。code mode 下**所有真实工具(`apply_patch`、`exec_command`)都不是独立 tool**,只写在 `exec` 的 description 里,模型必须调 `exec` 传 JS(`await tools.apply_patch(...)`)。

freeform 工具上游无通道 → 包成单 `input` 字符串字段的 JSON 工具(`FREEFORM_TOOL_SCHEMA`),**必须同时追加适配说明**(原描述明写 "not JSON",不说明则模型吐裸文本);工具名经 `customToolNames` 传到响应侧还原 `custom_tool_call`,漏传即错编成 `function_call`。流式**不能边收边发**:手里是 partial JSON,须缓冲到 block 结束解出 `input` 再一次性发。替身编解码的单一真相源是 `openai/freeform-tool.ts`。chat 端点**刻意**未实现 custom 工具:Chat Completions 规范同样有 `type:"custom"`(嵌套在 `custom` 下),但已知无客户端(Codex 0.122+ 只说 Responses、无法端到端验证);codec 放在 `openai/` 而非 `openai/responses/`,将来要接只需加一层 wire 形状适配。

**新版 Codex(0.147+)把工具再折进一层 namespace 容器(默认的 `functions` 与 subagent 的 `collaboration`),网关就地展开一层**:漏展开的症状 = 零工具上送、模型永远不调工具;为何只按名字认 `functions` 为默认命名空间(回程发裸名)、为何非递归、**`collaboration`(subagent 六个工具)展开后为何必须同时写回 `namespace` 字段**(2026-09-07:漏写 = `unsupported call`、模型无限重试,实测单轮 110+ 次计费请求),全在 `expandNamespaces` 头注释。

**multi-agent v2 的线程间信封 `agent_message` 三类都必须转**(`convertAgentMessage`):`NEW_TASK` 丢 → 子线程空 Payload;`MESSAGE`(`send_message` 的子 → 父中间消息)丢正文 → 父线程只见空 `Payload:`、误读成简短确认(2026-09-08:spawn/NEW_TASK/FINAL_ANSWER 全绿,只有它丢,固定 nonce 端到端返回 `EMPTY` 才看得出);`FINAL_ANSWER` 丢 → 父线程看不到子 agent 的答案(**最隐蔽**:`wait_agent` 的工具结果只有 `Wait completed.`、不含答案,丢了它链路全绿却在空手总结)。判据分两层见该函数头注释(白名单常量 `PLAINTEXT_BODY_MESSAGE_TYPES`,扩名单先抓脱敏 fixture);生命周期矩阵(并发不串线 / followup / interrupt / timeout / fork_turns / 故障重试不重复 spawn / `message` 正文入口,8/8)见 `tools/codex/README.md` + `test/manual/codex-subagent-{probe,lifecycle}-server.ts`。

### Messages hosted WebSearch

`websearch.ts` 的同一 Message 同时生成 JSON 与 SSE,遵守请求 `stream`;`web_search_tool_result.tool_use_id` 必须引用同次 `server_tool_use.id`。多轮请求读取最后一条 user query,不能反复搜首轮。MCP 错误、`isError` 和损坏结果按错误状态返回,429 保留 Retry-After,只有显式 `results:[]` 才算成功的零结果。普通 client function 即使同名 `web_search` 也不能被 MCP 旁路劫持。搜索时模型没参与,结果进入模型上下文的**唯一通道**是这条 Message 里的摘要文本:下一轮历史中的 `server_tool_use` / `web_search_tool_result` 在 Kiro 没有对应物、不上送,所以摘要必须照录 snippet 全文,不能截断(守卫 `test/claude/websearch-transport.test.ts` 的「WebSearch summary」)。

### Codex 侧无法用 web search

code mode 的 `additional_tools` 里**没有** `web_search`(`tools.web_search=true` 等三种配置均无效),fallback 形态倒是发 `{"type":"web_search"}`,但那是 hosted(服务端执行)工具、无 `parameters`,上游给不了。网关自带的 `claude/websearch.ts`(走 Kiro MCP)只处理 Messages 中「单个、名为 `web_search`、带日期版 hosted type」的工具——那是 Claude Code 的独立子请求路径,Codex 把它混在工具集里走不通。实测 Codex **接受**网关产的 `web_search_call` item(渲染成 `web search: <query>`),故要支持是可行的,但需新功能(注入工具 + 网关自己执行 MCP 搜索 + 产 item),不是转发能解决的。

### GPT credit 锚定与缓存反演

Kiro 不在 usage 里给缓存字段,折扣只落在 credits 上。GPT 的计费公式可以精确标定(2026-09-23,effort=none + 单词输出直打 V3 target):`credits = 倍率 × [k_in·(未命中 + 0.1·命中) + k_out·输出]`,`k_in = 1.6584e-5`、`k_out/k_in = 6.66`(公开价 $1.5 / $10 同比),倍率 = rateMultiplier(sol 4.4 / terra 2.2 / luna 1.1,三者逐点吻合)。缓存价恰为冷价的 0.1×;命中 = 同 conversationId 里此前请求的前缀(另有约 7 token 固定尾巴不进缓存;同一 id 下多段前缀互不挤占,见「会话身份映射到 kiro-cli」);**换 conversationId 发同样内容 credit 与冷请求逐位相同**——每请求随机 id 时测不到任何缓存。

plugin-derived 据此反演(`gptCacheDerivedBreakdown`):按隐藏推理 = 0 解命中数,推理成本被计入未命中输入,所以推理只会让 `cache_read` 低估(Codex 长会话回放:effort=low 命中 95.9–99.6%,high 85.5–97.8%;冷请求 ≈ 0)。成本仍锚定 `credits×0.04`(× multiplier),status 保持 `gpt_credit_anchored`。**绝不给 GPT 填 `CLAUDE_PRICE_USD_PER_TOK`**:Claude 的系数与缓存比例(0.5276)和 GPT 不同,分流必须在价格表查询**前**。

- **`kiro.inputTokens` 含本次输出**:contextUsage 百分比是生成之后的上下文占用,数到 100 / 1600 的对照里「输入」随输出长度增长,Claude 与 GPT 相同。GPT 反演已按此处理(先减可见输出);网关对所有模型上报的 `input_tokens` 目前都包含输出。
- **端到端验收**(2026-09-24,`test/manual/gpt-cache-derive-live.mjs` 走本地网关,命中真值由构造给出):把 o200k 数出的真实输出代入公式,7.6K–62K 前缀、部分命中、3.8K 输出、luna / sol、Chat / Responses 的误差恒为约 −24 token(续写边界的固定尾巴,与规模无关),credits 预测残差 < 1%——公式与常数成立,o200k 与上游计数一致。网关的误差全在可见输出 v:`estimateOutputTokens` 是字符启发式,对 GPT 偏差大(数字串估 1723、实为 3800;150 词短文估 287、实为 181),经 `(k_out/k_in − 1)/0.9 ≈ 6.3` 倍放大——估少则少报(续写 + 长输出少报 13K),**估多则虚报**(冷请求报出 642 命中,真值 0)。隐藏推理同理低估约 6.3×推理 token(实测约 32 个推理 token → −225)。
- **同会话切换 effort 让 GPT 缓存失效**:none → medium 的续写 credits 与冷请求逐位相同,两轮都 medium 则正常命中。反演据 credits 报 0 是对的;Codex 同一线程 effort 不变,不受影响。

## 错误流转 · 容量事件诊断

### 跨模型对照

典型形态:一批下游错误**全部**来自上游 5xx、网关自身零错误。判别顺序(每步独立否掉一批假设):① **同容器跨模型**——同容器同时段某模型大面积失败、另一模型零失败,只 `modelId` 变 → 上游**按模型**容量短缺,一击定案(上游 429/5xx 那几行(带 `capacity_reason`)不带模型字段,按 `reqId` 关联同请求 info 级入口行的 `model`(客户端原名);映射后的 `mapped_model` 只在 `debug`);② **分钟级时间轴**——失败集中在十几分钟窗口、窗口后流量更高却不失败 → 是事件非长期状态;③ **请求形状对照**(`max_tokens`/`tool_count`/`system_length` 分布相同)→ 非 converter 构造错;④ region/profileArn/`tier` 全同 → 非路由或配额档。**别按主机/账号先分桶**(同机同分钟有账号全挂也有毫发无伤,会误推「账号被封」)。**有界重试对此无效**(上游恢复远慢于请求内重试),空流有界重试的思路不能照搬 5xx;有效缓解是**切模型**。日志用 `capacity_reason` 结构化字段区分,别靠 substring 匹配 `error`。

### 容量不足的 5xx 为何是 503 而非 502

「压成 502」是给**未知**失败态的默认值,而 `MODEL_TEMPORARILY_UNAVAILABLE` / `INSUFFICIENT_MODEL_CAPACITY` 是已知态——同一件事上游还会用 429 和 mid-stream `ThrottlingException` 表达,那两条都是可重试信号(429 / 503),这条若压成 502,容量事件就看着像网关自己坏了。施加点在 `retry-executor.ts` 的 5xx 分支而非 `classifyErrorBody`:后者跑在 429 分支**之前**(会把更具体的 429 劫持成 503)且拿不到 header(丢 Retry-After)。只作用于 5xx——429 保持 `rate_limited`、408 保持透传,两条都有反向守卫。下游三元组复用 `upstreamErrorWire(true)`,与 mid-stream 容量信号同一份定义。判别子 / 判别顺序在 `matchModelCapacityReason()` 与 `MODEL_CAPACITY_REASONS` 的头注释(`kiro/provider-error.ts`);为何恒 503、不透传 504 见 `claude/error-mapper.ts` 的 `overloaded` 分支注释;为何绝不自己编 Retry-After 见 `parseRetryAfter` 头注释(`kiro/retry-executor.ts`)。反过来,额度判定(`matchQuotaExhausted`,见「额度耗尽」)对 402 根本不看 body——两者代价不对称:漏判额度 = 400「请检查请求体」(错且不可重试),而容量侧的产物是日志维度,一个似是而非的 token 比没有更糟。别统一这两个函数(反向守卫在 `test/kiro/provider-error.test.ts`)。

### 额度耗尽:402 一律算,其它 4xx 看声明的 reason

上游用两种异常报「额度用完」。依据是 kiro-cli 自带 KAS SDK 里 KiroRuntimeService 的错误 schema 和 KAS 自己的映射:

- `ServiceQuotaExceededException` = HTTP **402**,reason 为 `MONTHLY_REQUEST_COUNT` / `OVERAGE_REQUEST_LIMIT_EXCEEDED` / `CONVERSATION_LIMIT_EXCEEDED`。这个服务的 402 只有这一种异常。
- `ThrottlingException` = HTTP **429**,reason 里既有额度类(`MONTHLY_REQUEST_COUNT` / `DAILY_REQUEST_COUNT`),也有真限流(`INSUFFICIENT_MODEL_CAPACITY` / `CREDIT_CONSUMPTION_RATE_EXCEEDED` / `USER_REQUEST_RATE_EXCEEDED` / `SERVICE_REQUEST_RATE_EXCEEDED`)。
- KAS 把两种异常里 reason 为 `HOURLY/DAILY/WEEKLY/MONTHLY_REQUEST_COUNT` 或 `USAGE_LIMIT_REACHED` 的都转成不可重试的 `UsageLimitReachedError`,`OVERAGE_REQUEST_LIMIT_EXCEEDED` 转成 `OverageLimitReachedError`(「You've reached your overage limit.」);只有容量与速率类走可重试的限流错误。

公开报文与 schema 一致:402 + `MONTHLY_REQUEST_COUNT`、402 + `OVERAGE_REQUEST_LIMIT_EXCEEDED` 都有实际出现的记录。

网关规则(`matchQuotaExhausted`,`kiro/provider-error.ts`,在 `classifyErrorBody` 里、先于 429 分支):402 一律算额度耗尽;其它 4xx 只认**声明的** reason 属于 `USAGE_LIMIT_REASONS`,不扫 prose——429 上误判的代价是把可重试的限流变成硬停。下游统一 402 `billing_error`(Anthropic 原生类型,SDK 不重试),日志 `quota_reason` 记上游 reason。修之前(2026-09-27):402 + overage 上限 → 400「请检查请求体」;429 + 月 / 日额度 → 可重试的 429,客户端对着下个周期才重置的额度反复退避;402 + 月额度 → 402 但类型是 `api_error`。守卫 `test/kiro/provider-error.test.ts`、`test/kiro/retry-executor-429.test.ts`、`test/claude/quota-exhausted-e2e.test.ts`(真 provider + executor + 路由,四个端点)。

流中途的异常帧不看 reason:`ThrottlingException` 帧仍按可重试 503 处理,客户端重发后新请求在建流前就拿到 402 / 429,走上面的规则,只多一次空请求。帧形态的额度耗尽没人报过,不为它加第三种终态。

## 日志四条守卫的理由

部署形态是多容器同机 + 高频健康检查,日志既是排障依据也是磁盘成本。

- **每请求只打一行**:Fastify 内置请求日志与 `index.ts` 的 `onResponse` hook 并存会每请求三行、其中两行都叫 `request completed`——不只是体积,按 incoming/completed 配对做的分析会稳定算错。
- **业务字段一律 snake_case**:混用会逼运维为同一指标查两种拼写。
- **一个指标只有一个 owner**:`capacity_reason` 是上游容量事件的唯一计数维度,而同一件事有 429 / 5xx 两种线格式;mapper 手里也有 `err.kind.reason`、顺手再记一次就让 5xx 形态权重翻倍,只记 5xx 分支又漏掉更常见的 429。
- **网关自己造成的结果不记 `error`**:主动 `destroy()` socket、主动 abort 上游后读流抛错,都是那一行代码的必然结果而非上游故障。记成 error 会污染告警,且让人误判「上游在报错」(实测假 error 与自毁动作 1:1)。

## 上游与客户端的实测事实

### kiro-cli 自己怎么处理 5xx / 429 / 400

2.21.1 实测(`kiro-cli-probe.ts` 注入状态码):**500/502/503 → 共 9 次**(SDK 内层 attempt 1→2→3,带抖动退避 ~150–1800ms;应用层外层再来 3 轮,间隔 ~2.1s→4.6s);**429 → 3 次**,不走内层重试(attempt 恒为 1),间隔**严格等于 `Retry-After`**(实测 7007/7009ms);**400 → 不重试**。网关对瞬时故障**一次都不重试**、原样透传(架构决策见 `retry-executor.ts` 头注释 + 「跨模型对照」)——即 kiro-cli 有 9 次机会而网关只有 1 次,瞬时 5xx 上二者体感差距全在于此。仅有的两个语义重试各一次:401 换 token、400 `THINKING_SIGNATURE_INVALID` 剥掉 `reasoningContent`(见「原生 reasoning / effort / system 的 wire 真相」)。

### kiro-cli V3(KAS)的 wire

网关只模拟 kiro-cli `chat --v3` 一套 wire:对话、工具、subagent 由 KAS(kiro-cli 内嵌的 Node 进程 `@kiro/agent`,首次 `--v3` 时解压到 kiro-cli 数据目录的 `kas/<版本>-<hash>/`)发;登录、OIDC 刷新、GetUsageLimits 仍由 Rust 外壳发。profile 因此分 `kas` / `shell` 两个身份(`kiro/client-profile.ts`)。2.23.1 与 V2 的差异(V2 列 = 守卫禁止回流的形态):

| 项 | V2(Rust 引擎,不得回流) | V3(KAS,网关现行) |
|---|---|---|
| target / 路径 | `AmazonCodeWhispererStreamingService.*`,REST 路径 | `KiroRuntimeService.GenerateAssistantResponse` / `.InvokeMCP`,同 host 路径 `/` |
| UA | `aws-sdk-rust/… api/codewhispererstreaming/…` | `aws-sdk-js/1.0.0 ua/2.1 os/<platform>#<release> lang/js md/nodejs#… api/kiroruntime#1.0.0 m/N KiroCLI/<ver> KAS/<ver> os/<os> …` |
| 头 | accept、accept-encoding | 无这两个;GAR 多 `x-amzn-kiro-client-attribution: unrecognized`;都带 `connection: keep-alive` |
| body | `origin: KIRO_CLI`、每条 user 带 `envState`、assistant 带 `messageId` | `origin: AI_EDITOR`、顶层 `agentMode`、`sess_` id + `rootConversationId`;无 envState / messageId,空集合不发 |

- 与 KAS 唯一有意的偏离:system 折进首条 user,不造 `assistant:"I will follow these instructions."`(理由见「注入文本」)。KAS 自己的 `You are Kiro…` 人设不发,网关只发客户端的 system。同一 prompt 下网关与 KAS 的 GAR body 结构逐键一致(键序除外)。
- axios 会自动补 `accept` / `accept-encoding`,header 值置 `false` 才压得住(`SUPPRESS_AXIOS_DEFAULTS`,真实服务器实测)。
- 抓包:2.23.1 起非交互路径认 `KIRO_KAS_ENDPOINT` / `KIRO_KAS_CONTROL_PLANE_ENDPOINT`,`scripts/capture-kiro-cli.sh` 用它们把 KAS 指向本地 mock,Rust 外壳仍靠临时 settings。别用 `KIRO_KAS_SERVER_PATH` 包装器:录出来的 UA 是 `KAS/unknown`,与真实启动(`KAS/0.66.8`)不同,不能当画像。
- V3 形态的 body(无 envState / messageId、历史 user 不带 context、tool-use-only 的 assistant `content:""`)上游直打全部 200;Codex / Claude Code 真实验收与 V2 基线无回归(Codex 同任务 2.83 对 2.88 credit)。守卫 `test/static/kiro-wire-v3.test.ts`、`test/kiro/provider-v3-wire.test.ts`。

### 重试头的三个调用点

`applyRetryHeaders`(`kiro/retry-executor.ts`)是 `amz-sdk-invocation-id` / `amz-sdk-request` / `x-kiro-attempt` wire 格式的唯一 owner(`attempt=N; max=M` 拼法、第 2 次起才有的 `ttl=`、`x-kiro-attempt` 的无空格分隔),含 2.21.1 抓包形态。三个调用点的差异只走参数:executor(默认 `max=3`+Kiro 头)、OIDC refresh(`max=4`,另一个服务)、`getUsageLimits`(`max=1`,无抓包证据故不发 Kiro 头)。后两者**故意**不接 `RetryExecutor`:它抛 `ProviderError`,而 `/kiro/usage` 与 plugin capability 都按 `KiroHttpError` 分流(`routes/kiro.ts` `translateUsageError`),且 executor 的 body 分类器是按 messages 端点的错误体设计的——要统一得连错误语义一起迁。抓包侧同名清单 = `scripts/capture-kiro-cli.sh` 的 `RETRY_HEADERS`(剔出 fixture,免得抓包时点决定字段值)。**别搬回** `provider.ts` 的 `buildHeaders`:那里每次调用生成新 uuid,上游看到的每次重试都成了「attempt=1 的全新请求」。一次 `execute()` = 一次逻辑调用(共用 invocation-id、attempt 递增),空流重试每次重走 `execute()` = kiro-cli 的**外层**重试(换新 id、attempt 归 1)。

### web_search / web_fetch 分别在哪执行

`web_search` → **走上游 `InvokeMCP`**(V3:`x-amz-target: KiroRuntimeService.InvokeMCP`、路径 `/`,body 是 JSON-RPC `tools/call` + `{name:'web_search',arguments:{query}}` + `profileArn`,不带 `x-amzn-kiro-profile-arn` 头与 `x-kiro-attempt`),所以网关代为执行是对的(`claude/websearch.ts`);`web_fetch` → **零上游请求**,客户端本地直接抓——是客户端职责,网关不该实现。kiro-cli 把两者都当普通工具上送给模型;网关只旁路「单个、名为 `web_search` 且 type 为 `web_search_YYYYMMDD` 的 hosted 工具」。Claude Code 2.1.263 的实际独立子请求为 `web_search_20250305`,普通同名 function 必须仍交客户端执行。

### 工具调用往返与图片的 wire 形态

KAS(2.23.1)实测:assistant 侧 `{content, toolUses:[{toolUseId,name,input}], reasoningContent?}`(`input` 是**对象**;只有 toolUses 时 `content:""`;不带 V2 的 `messageId`)。user 侧 `toolResults:[{toolUseId, content:[{text}|{json}], status:"success"|"error"}]`,不带 `isError`;只含工具结果的 user 消息 `content:""`,没有工具结果的历史 user 消息整个不带 `userInputMessageContext`。`status` 表示**工具本身是否执行成功**,不是业务结果——`exit 42` 仍是 `success`。`status` 与 `isError` 都不作为失败信号到达模型(对照实验见 `ToolResult` 头注释),判定靠正文;`{json}` 通道我们不产,理由同在该注释。图片经 tool_result 回传时(`fs_read` 的 `Image` mode)**提升到 message-level `images: [{format:'png', source:{bytes:<base64>}}]`**,`toolResults[].content` 原位只留占位文本 `"See images data supplied"`;项目的提升逻辑一致,仅占位文案不同(带序号的 `[image k attached to this message]`,理由见 `imagePlaceholder` 与「多图归属只靠顺序」),实测两者上游都收。`toolResults[].content` 塞 `{image}` 上游 200 但静默丢弃(2026-09-09 直连实测),wire 没有结构化图片通道。

### Responses 的字节量约为 Claude 的 10×

同一段内容实测:Claude 15KB/118 事件 vs Responses 155KB/906 事件。**不是丢包也不是编码 bug**,拆开是两项:上游 GPT 的 delta 分片更碎(事件数 ~7.7×)+ Responses 每事件字段更多(`item_id`/`output_index`/`content_index`/`sequence_number`,每事件 171B vs 131B,~1.3×)。运维含义:同样内容 Responses 更吃带宽与写缓冲,背压也更早出现(慢读实测 26.6s vs 13.1s)。

### 客户端报 InputValidationError 怎么查

先看网关日志有没有 `upstream truncated tool_use (no isComplete frame)`——它只说明上游截断过,**不意味着客户端会收到残缺调用**:未完成的调用已被缓冲在网关内、从不上 wire。故若仍报 `JSON parse failed`,那是**新**问题(参数在网关侧就该解析失败并报协议错误),别当成同一条。其余两支在客户端侧:`ZOD_VALIDATION`(超 `questions≤4`/`options≤4`/`header≤12` 或违反「问题文本与同题内 option label 须唯一」的跨字段 refine,后两条 JSON Schema 里表达不出、模型看不到)、`PERMISSION_UPDATED_INPUT`。

### 原生 reasoning / effort / system 的 wire 真相

证据来自录真实 kiro-cli 2.22.1 的 V2 / V3 两个引擎(V3 形态见「kiro-cli V3(KAS)的 wire」)与用网关 provider 直打上游。复跑:`test/manual/reasoning-wire-probe.ts`(直打)、`kiro-cli-capture-proxy.mjs`(录 kiro-cli)、`reasoning-roundtrip-live.mjs`(走网关验收)。

**wire 事实**:

- history 里的 thinking 有原生字段:`assistantResponseMessage.reasoningContent = {reasoningText:{text,signature}}`(V3 下 GPT 也是这个形态,文本为占位 `...`;V2 target 的 GPT 是 `{redactedContent}`,该形态仍接收),不拼 `<thinking>` 文本。signature 缺失或改坏 → 400 `THINKING_SIGNATURE_INVALID`。KAS 的对策:无签名不发、换模型不发、收到该错剥掉全部 `reasoningContent` 重发一次。
- effort 只在顶层 `additionalModelRequestFields` 生效,形状由 `ListAvailableModels` 的 `additionalModelRequestFieldsSchema` 逐模型给出:Claude = `{thinking:{type:adaptive|disabled, display?:summarized|omitted}, output_config:{effort}, max_tokens}`(4.6 系无 xhigh);GPT = `{reasoning:{effort}}`(含 none)。写在 `userInputMessage.reasoning` 的 effort 上游不认(计费不随档位变)。`thinking.type: disabled` 真关;`display: omitted` 只回 signature 帧。Claude Code 发的就是 adaptive + omitted。
- 回包 reasoning 是摘要:文本长度不随 effort 变,credits 才是 effort 生效的判据;末尾单独一帧只有 `{signature}`。
- 顶层 `systemPrompt` 两个 target 都 400(只有 `com.amazon.kiro.runtimeservice` 命名空间有它,KAS 受 feature flag 控制也没发)——system 仍无 wire 通道,折进首条 user(见「注入文本」)。KAS 自己用开场假轮次承载 system,网关有意不照抄(见「kiro-cli V3(KAS)的 wire」)。
- 逐模型能力:opus-5 / 4.7 / 4.8 回摘要 reasoning + signature,opus-5 默认就思考(opus-5.5 的 thinking 关不掉,见下);sonnet-5 回 signature 帧、计费随 effort 变;sonnet-4.6 加字段后回明文 reasoning + signature,默认不思考;opus-5.5 的 schema 里 thinking 只有 adaptive(显式 disabled 回 400 `ValidationException`)、effort 默认 medium,reasoning 帧与签名同 opus-5;sonnet-5.5 的 schema 是 adaptive / between_tools(disabled 同样 400)、effort 默认 high——KAS 只对 enum 同时含 disabled 与 adaptive 的模型做开关,这两个常开模型它不发 thinking 字段、只发 effort,也从不发 `between_tools`(2.28.0 bundle);opus-4.6 发字段只涨计费、无帧无签名;4.5 及以下 / haiku 无 schema。非原生模型只认 `<thinking_mode>enabled</thinking_mode><max_thinking_length>N</max_thinking_length>` 前缀,收到 adaptive 形态或没有前缀都把推理写进正文——本项目不注入任何前缀,仅作记录。
- 其它:`ListAvailableModels` 带 `tokenLimits` / `promptCaching` / `refusalFallbackModels`,直打走 KAS 控制面 `management.{region}.kiro.dev`(runtime / codewhisperer 两个 host 不认该 target,入口见 `claude-rate-probe.ts` 的 `models` 阶段);`kiro-cli --effort` 与 `/effort` 在非交互和 legacy UI 下都不上 wire;KAS 发 `toolUses: []` 上游照收。
- **KAS 在客户端没指定时也发默认 effort**:取 schema 的 `default`(opus-4.7 为 xhigh、opus-5.5 为 medium,其余原生模型为 high;Claude 同时发 `thinking:{type:adaptive}`)。网关照此补默认(`defaultEffort`),只影响不带 thinking / reasoning_effort 的客户端——录得的 5923 条 Claude Code 请求全部显式带 thinking,Codex 总带 effort。
- Claude Code 2.1.278 headless 验收全过;请求含 `thinking.display`、`context_management` 与新 beta 头,均透传;启动时多一个 `HEAD /claude/api/hello`,网关 404 它照跑。

**本项目的做法**(真相源 `claude/converter.ts`、`claude/types.ts`、`kiro/retry-executor.ts`):

- thinking 只有 adaptive 一种语义:`normalizeThinking` 把 `enabled` 归一成 `adaptive`、丢掉 `budget_tokens`;`resolveEffort` 只看 `output_config.effort`(缺省为模型默认,见上)。
- 原生模型(`MODELS_WITH_NATIVE_REASONING`)由 `buildAdditionalModelRequestFields` 生成顶层字段,`toKiroRequest` 装配、三个 handler 共用;`display` 透传;sonnet-4.6 的 xhigh 降 high;`max_tokens` 不发(传了会让小 max_tokens 的客户端在思考阶段被截断);客户端没提 thinking 时同 KAS 补默认;thinking 常开的 opus-5.5 / sonnet-5.5(`MODELS_THINKING_ALWAYS_ON`)收到 `disabled`(客户端的 `between_tools` 在 `normalizeThinking` 归一成它)按 adaptive + effort low 发(Anthropic 文档对该模型「关思考」给的替代写法;客户端显式 effort 优先),响应照常带 thinking 块——Anthropic 上这个模型本来就一定回 thinking 块。本轮实际生效的 thinking 只由 `effectiveThinking` 判定,上游字段与响应侧 thinking 通道(`responseThinkingEnabled`)都从它推出;两边各算各的时,未提 thinking 的流式请求会先开一个空 text 块、thinking 被挤到其后。
- OpenAI 的 `reasoning_effort` / `reasoning.effort`(`reasoningConfigFromEffort`):`none` → disabled、`minimal` → low、缺省或未知取值按模型默认。OpenAI 两个端点不解析 `-thinking` 后缀,思考开关只看 effort;Messages 端点的后缀只开 adaptive,effort 取 `output_config.effort`、缺省按模型默认(`request-validator.ts`)。
- 非原生模型不做任何 thinking 控制:不发字段、不注入前缀;`-thinking` 后缀不改上游请求,只像客户端显式发 thinking 一样打开响应侧 legacy `<thinking>` 解码(解码器只对非原生模型保留)。
- history:只把带签名的 `thinking` 块放进 `reasoningContent`(`redacted_thinking`:网关信封还原成原生推理,外来的 → `redactedContent`),无签名的丢弃,content 只放可见文本,绝不拼 `<thinking>`;一条 Kiro 消息一个槽位,多块取最后一块。
- 签名失效:`RetryExecutor` 收到 `THINKING_SIGNATURE_INVALID` 剥掉全部 `reasoningContent` 重发一次(info 级),再失败才 400。

**验收**(`reasoning-roundtrip-live.mjs`):签名原样回传无重试;改坏签名恰好一次剥离重试后成功;omitted 回传成功;effort max 计费高于 low;GPT / sonnet-4.6 正常;流式含 `thinking_delta` + `signature_delta` 且过协议不变量。录得请求重放可见文本零丢失、无签名 thinking 全部丢弃且零泄漏;Claude Code harness 全过。守卫:`test/static/no-thinking-tag-stitching.test.ts`、`test/claude/converter-reasoning-content.test.ts`、`test/claude/native-effort-integrity.test.ts`、`test/kiro/retry-executor-thinking-signature.test.ts`。

### 支持哪些模型 / 加模型要同改的地方

`claude/models-catalog.ts` + `mapModel()`。**加 GPT 变体同改**:mapModel / MODELS_WITH_NATIVE_REASONING / claude catalog(openai catalog 复用它)/ plugin-derived `gptVariant`(跨包复制的变体 token sol·terra·luna·codex)+ `GPT_RATE_MULTIPLIER`(上游 rateMultiplier);context window 按 `isGptModelId` 前缀判定,不用改;**加 Claude 模型同改**(原生集合的 Claude 部分现为 opus-5.5 / opus-5 / opus-4.8 / opus-4.7 / sonnet-5.5 / sonnet-5 / sonnet-4.6):mapModel / MODELS_WITH_NATIVE_REASONING / `MODELS_WITHOUT_XHIGH`(上游 schema 无 xhigh 的才列)/ `CLAUDE_MODELS_WITH_1M_CONTEXT`(1M 窗口的才列)/ `DEFAULT_EFFORT_BY_MODEL`(schema 默认不是 high 的才列)/ claude catalog / plugin-derived price+threshold(漏了不报错、只会让该模型 `unknown_model`,守卫 `test/static/derived-price-coverage.test.ts`)——mapModel 里是往 `CLAUDE_FAMILIES` 对应家族加一个上游 id(版本号由 id 解析,有无小数点以 list-models 为准);表里没有的版本(比最新还新、或夹在已知版本之间,如 opus-5.1)一律 400、不降级,只有没有版本号或比最老已知版本还老的写法走家族兜底,所以新模型不加表就用不了;minor 后面紧跟字母的不算 minor(`opus-5-1m` 是 opus-5);openai catalog 自动继承。schema 里 thinking 没有 disabled 的模型(opus-5.5 / sonnet-5.5)另入 `MODELS_THINKING_ALWAYS_ON`,`disabled` 按 adaptive 发。threshold 取 Anthropic prompt-caching 文档的最小可缓存长度,随代际**不单调**(opus-5.5 / opus-5 / sonnet-5.5 512、opus-4.8 / sonnet-5 1024、opus-4.7 2048、opus-4.6 4096)。Fable 系列上游尚不成熟,不支持(mapModel 不认,400)。

**derived 价格表只管 `claudeEquivalentCostUsd`,反演用的是 Kiro 的计价**。Kiro 按上游 rateMultiplier 计价:同样的 token,sonnet-4.6 / sonnet-5 的冷价与输出价正好是 opus-5 × 1.3/2.2(四位有效数字一致)。opus 系标价正在倍率线上(倍率 / 输入单价 = 0.44),sonnet-4.x 的 $3 高 1.5%,haiku 的 $1 按倍率线算高 10%(与 `KIRO_CACHE_READ_RATIO` 记录的 haiku 残差同向),这些沿用标价反演;明显偏离的模型进 `KIRO_BILLING`(基价 = opus 标定线 × 倍率 / 2.2,另可带未命中溢价),否则反演系统性偏差。新模型先跑 `test/manual/claude-rate-probe.ts`(💰,两个模型默认全跑约 8 credit)与已标定模型同尺寸对照:

- **opus-5.5**(2026-09-27,对照 opus-5):上游倍率 2.0 / 2.2,输出与命中价正好按倍率缩放(0.909 / 0.911);**未命中输入却是 opus-5 的 1.770 倍**(三个冷尺寸共线,残差为零),折成基价 1.942 倍。同前缀重发命中价 = 基价 × 0.530,与全局 `KIRO_CACHE_READ_RATIO` 一致。照 $4/$20 标价反演等于把冷价当基价:重发 99.8% 命中算成 84%,命中前缀 + 新内容算成 0。对用户的含义:冷启动 / 缓存失效的请求比 opus-5 贵约 77%,稳态命中比 opus-5 便宜约 9%。溢价只落在 prompt 上:`kiro.inputTokens` 含本轮输出,而输出单价比 0.909 恰为倍率比(输出也带溢价的话应是 0.923),所以全未命中基线 = 溢价 ×(T − 可见输出)+ 可见输出;把溢价乘到整个 T 上,冷请求会凭空多出约 0.67 × 输出的命中。
- **sonnet-5**(2026-09-27,对照 sonnet-4.6):冷价、输出价与 sonnet-4.6 逐项相同,无未命中溢价;Anthropic 把 $2/$10 转为标准价后标价离开倍率线,按 $2 反演会把同前缀重发的 98.5% 命中算成 44%,故进 `KIRO_BILLING`(1.3x)。探针注意:sonnet-5 对 ≥30K token 的随机串文档会回 `metadataEvent.stopReason = CONTENT_FILTERED`(空流、不计费,但前缀照样写进缓存),标定时调小 `K2C_WORDS_*`。
- **sonnet-5.5**(2026-10-07,对照 sonnet-5,`K2C_WORDS_S=2000 K2C_WORDS_L=8000`):倍率同为 1.3,同前缀重发的 credits 与 sonnet-5 四位有效数字相同、输出价比 1.002;**未命中输入是基价的 1.942 倍**(六个冷点共线,最大残差 < 0.01%),与 opus-5.5 的溢价四位有效数字相同,故进 `KIRO_BILLING`(1.3x + 1.9423)。照 $2 标价反演会把重发的 99.6% 命中算成 45%。对用户的含义:冷启动 / 缓存失效的请求约是 sonnet-5 的 1.94 倍,稳态命中与输出与 sonnet-5 同价。「命中前缀 + 新内容」这一形状两个 sonnet 上游都没完整命中(已标定的 sonnet-5 同样反演为 0,credits 正落在冷价线上),是上游缓存行为、不是反演误差。
- 同一轮还复核了 opus-5:冷斜率与 `k_in` 线差 0.26%,命中比 0.529,老标定仍成立。sonnet-4.6 仍按 $3 反演,比倍率线高 1.5%,冷请求会多出约 3.7% 命中(既有残差)。
- 已知残差,未修:小 prompt + 长输出时 Claude 路径严重低估命中。同一轮 `output` 阶段(数到 100 / 500),KAS 自带的约 6.7K 前缀是真实命中,反演只得 0–5K。根因与 GPT 相同(「GPT credit 锚定与缓存反演」):core 按 4 字符 / token 估可见输出,数字串低估约一半,误差再经 `k_out` 放大。Claude Code 稳态流量的输出占比小,影响有限。
