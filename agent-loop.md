# Agent Loop

一次完整的观察→思考→行动→更新核心实现在 `packages/core/src/session/runner/llm.ts:173-348`。
```
    const runTurnAttempt = Effect.fn("SessionRunner.runTurn")(function* (
      sessionID: SessionSchema.ID,
      promotion: SessionInput.Delivery | undefined,   // "steer" = immediate, "queue" = deferred
      step: number,                                   // current step counter for step-limit enforcement
      recoverOverflow?: typeof compaction.compactAfterOverflow,  // compaction callback for overflow recovery
    ) {
      // --- Observe: load session state ---
      const session = yield* getSession(sessionID)                          // load session by ID from store
      if (session.location.directory !== location.directory || session.location.workspaceID !== location.workspaceID)
        return yield* Effect.interrupt                                      // location guard: reject cross-location execution
      const agent = yield* agents.select(session.agent)                     // resolve agent config (system prompt, permissions, steps)
      const initialized = yield* SessionContextEpoch.initialize(db, loadSystemContext(agent), session.id)  // init or restore context epoch
      const toolFibers = yield* FiberSet.make<void, ToolOutputStore.Error>()  // fiber set to track background tool executions
      let needsContinuation = false                                          // becomes true when LLM emits tool calls
      let currentStep = step                                                 // track step for agent step-limits

      // --- Observe: promote pending user inputs ---
      if (promotion) {                                                       // drain inputs into visible messages
        const cutoff = yield* EventV2.latestSequence(db, session.id)         // sequence cutoff for promotion boundary
        let promoted = 0
        if (promotion === "steer") promoted = yield* SessionInput.promoteSteers(db, events, session.id, cutoff)  // promote all pending steers
        if (promotion === "queue") {
          promoted += Number(yield* SessionInput.promoteNextQueued(db, events, session.id))  // promote one queued input
          promoted += yield* SessionInput.promoteSteers(db, events, session.id, cutoff)      // also promote steers that arrived alongside
        }
        if (promoted > 0) currentStep = 1                                    // new input = fresh agent turn, reset step counter
      }

      // --- Think: assemble system context, model, history, tools ---
      const system =
        initialized ?? (yield* SessionContextEpoch.prepare(db, events, loadSystemContext(agent), session.id))  // get system context (cached or fresh)
      const model = yield* models.resolve(session)                           // resolve LLM model + credentials for this session
      const entries = yield* SessionHistory.entriesForRunner(db, session.id, system.baselineSeq)  // load session history from baseline
      const context = entries.map((entry) => entry.message)                  // extract message objects from history entries
      const isLastStep = agent.info?.steps !== undefined && currentStep >= agent.info.steps  // check if agent step-limit reached
      const toolMaterialization = isLastStep ? undefined : yield* tools.materialize(agent.info?.permissions)  // build tool definitions filtered by permissions
      const promptCacheKey = /^ses_[0-9a-f]{64}$/.test(session.id) ? session.id.slice(4) : session.id
      const request = LLM.request({                                          // build the LLM request
        model,
        providerOptions: { openai: { promptCacheKey } },                     // prompt caching for long sessions
        system: [agent.info?.system, system.baseline]                        // system prompt: agent config + context baseline
          .filter((part): part is string => part !== undefined && part.length > 0)
          .map(SystemPart.make),
        messages: [...toLLMMessages(context, model), ...(isLastStep ? [Message.assistant(MAX_STEPS_PROMPT)] : [])],  // translate V2 history → LLM messages
        tools: toolMaterialization?.definitions ?? [],                       // advertise tool definitions to the model
        toolChoice: isLastStep ? "none" : undefined,                          // on last step: force text-only response
      })
      if (yield* compaction.compactIfNeeded({ sessionID: session.id, entries, model, request }))  // check if history exceeds threshold
        return yield* Effect.die(continueAfterCompaction(currentStep))        // compact then retry via TurnTransition

      // --- Prepare: snapshot before LLM turn ---
      const startSnapshot = yield* snapshots.capture()                       // capture filesystem state before LLM turn
      const publisher = createLLMEventPublisher(events, {                    // create publisher to durably persist LLM stream events
        sessionID: session.id,
        agent: agent.id,
        model: {
          id: ModelV2.ID.make(model.id),
          providerID: ProviderV2.ID.make(model.provider),
          ...(session.model?.variant === undefined ? {} : { variant: session.model.variant }),
        },
        snapshot: startSnapshot,
      })
      const withPublication = Semaphore.makeUnsafe(1).withPermit             // serialize publication to avoid race conditions
      const publish = (event: LLMEvent, outputPaths: ReadonlyArray<string> = []) =>
        withPublication(publisher.publish(event, outputPaths))
      let overflowFailure: ProviderErrorEvent | undefined                    // capture overflow error before assistant starts

      // --- Think + Act: stream LLM response, settle tool calls ---
      const providerStream = llm.stream(request).pipe(                       // send request to LLM, process streaming events
        Stream.runForEach((event) =>
          Effect.gen(function* () {
            if (overflowFailure || publisher.hasProviderError()) return      // skip events after failure
            if (LLMEvent.is.providerError(event)) {
              if (isContextOverflowFailure(event) && !publisher.hasAssistantStarted()) {  // overflow before any text → can recover via compaction
                overflowFailure = event
                return
              }
            }
            yield* publish(event)                                            // persist event (text, reasoning, etc.) durably

            // --- Act: execute non-provider tool calls in background fibers ---
            if (event.type !== "tool-call" || event.providerExecuted) return  // only handle local tool calls
            if (!toolMaterialization) {                                       // step limit reached, tools disabled
              yield* withPublication(publisher.failUnsettledTools("Tools are disabled after the maximum agent steps"))
              return
            }
            needsContinuation = true                                          // tool calls → inner loop must run another turn
            const assistantMessageID = yield* publisher.assistantMessageID(event.id)  // get durable assistant message ID for this tool call
            yield* Effect.uninterruptibleMask((restore) =>
              restore(
                toolMaterialization.settle({                                 // execute the tool via registry
                  sessionID: session.id,
                  agent: agent.id,
                  assistantMessageID,
                  call: event,
                }),
              ).pipe(
                Effect.flatMap((settlement) =>
                  publish(                                                   // persist tool result durably
                    LLMEvent.toolResult({
                      id: event.id,
                      name: event.name,
                      result: settlement.result,                             // tool output value
                      output: settlement.output,                             // serialized output for display
                    }),
                    settlement.outputPaths ?? [],
                  ),
                ),
              ),
            ).pipe(FiberSet.run(toolFibers))                                  // run in background, collected by toolFibers
          }),
        ),
        Effect.ensuring(withPublication(publisher.flush())),                 // flush buffered events when stream ends
      )

      // --- Update: post-stream settlement ---
      return yield* Effect.uninterruptibleMask((restore) =>
        Effect.gen(function* () {
          const stream = yield* restore(providerStream).pipe(Effect.exit)     // capture stream result as Exit
          const failure =
            stream._tag === "Failure" ? Option.getOrUndefined(Cause.findErrorOption(stream.cause)) : undefined  // extract LLM error if any

          // Overflow recovery: compact conversation and retry
          if (
            recoverOverflow &&
            !publisher.hasAssistantStarted() &&
            isContextOverflowFailure(overflowFailure ?? failure) &&
            (yield* restore(recoverOverflow({ sessionID: session.id, entries, model, request })))  // compact via LLM summarization
          )
            return yield* Effect.die(continueAfterOverflowCompaction(currentStep))  // retry without overflow recovery path

          // Publish deferred overflow failure or LLM errors
          if (overflowFailure) yield* publish(overflowFailure)                // publish overflow event if assistant started after it
          const llmFailure = failure instanceof LLMError ? failure : undefined
          if (llmFailure && !publisher.hasProviderError()) {
            yield* withPublication(publisher.failUnsettledTools("Provider did not return a tool result", true))  // mark pending tools as failed
            yield* withPublication(publisher.failAssistant(llmFailure.reason.message))  // persist LLM error as assistant message
          }

          // Handle stream interruption
          if (stream._tag === "Failure" && Cause.hasInterrupts(stream.cause)) yield* FiberSet.clear(toolFibers)

          // Await tool fibers and handle user-declined errors
          const settled = yield* restore(awaitToolFibers(toolFibers)).pipe(Effect.exit)  // wait for all tool executions to finish
          if (settled._tag === "Failure" && isUserDeclined(settled.cause)) {   // user declined a permission or rejected a question
            yield* FiberSet.clear(toolFibers)
            yield* withPublication(publisher.failUnsettledTools("Tool execution interrupted"))
            return yield* Effect.interrupt                                     // halt the entire drain
          }

          // Handle interruption from either stream or tool fibers
          if (
            (stream._tag === "Failure" && Cause.hasInterrupts(stream.cause)) ||
            (settled._tag === "Failure" && Cause.hasInterrupts(settled.cause))
          ) {
            yield* FiberSet.clear(toolFibers)
            yield* withPublication(publisher.failUnsettledTools("Tool execution interrupted"))
            if (publisher.hasActiveAssistant())
              yield* withPublication(publisher.failAssistant("Provider turn interrupted"))
          }

          // Handle tool execution failure (non-interrupt)
          if (settled._tag === "Failure" && !Cause.hasInterrupts(settled.cause)) {
            const failure = Cause.squash(settled.cause)
            const message = failure instanceof Error ? failure.message : String(failure)
            yield* withPublication(publisher.failUnsettledTools(`Tool execution failed: ${message}`))
          }

          // Publish Step.Ended with snapshot diff
          const stepSettlement = publisher.stepSettlement()                   // aggregate tokens, finish reason from publisher
          if (stepSettlement && !publisher.hasProviderError()) {
            const endSnapshot = yield* snapshots.capture()                    // capture filesystem state after the turn
            const files =
              startSnapshot && endSnapshot
                ? yield* snapshots
                    .files({ from: startSnapshot, to: endSnapshot })          // compute file changes (created/modified)
                    .pipe(Effect.catch(() => Effect.succeed(undefined)))
                : undefined
            yield* withPublication(
              events.publish(SessionEvent.Step.Ended, {                       // persist step settlement durably
                sessionID: session.id,
                timestamp: yield* DateTime.now,
                assistantMessageID: yield* publisher.startAssistant(),
                finish: stepSettlement.finish,                                // stop_reason / finish_reason
                cost: 0,
                tokens: stepSettlement.tokens,                                // token usage stats
                snapshot: endSnapshot,
                files,                                                        // list of changed file paths
              }),
            )
          }

          // Fallback error recording for edge cases
          if (publisher.hasProviderError())
            yield* withPublication(publisher.failUnsettledTools("Tool execution interrupted"))
          if (stream._tag === "Success" && !publisher.hasProviderError())
            yield* withPublication(publisher.failUnsettledTools("Provider did not return a tool result", true))
          if (stream._tag === "Failure") return yield* Effect.failCause(stream.cause)
          if (settled._tag === "Failure" && Cause.hasInterrupts(settled.cause))
            return yield* Effect.failCause(settled.cause)

          // Return continuation decision to the inner loop
          return { needsContinuation: !publisher.hasProviderError() && needsContinuation, step: currentStep }
        }),
      )
    }, Effect.scoped)
```
## 架构总览

```
SessionExecution.resume(sessionID)
  └─ SessionRunCoordinator.run(sessionID)        [run-coordinator.ts]
      └─ SessionRunner.run({ sessionID, force }) [runner/llm.ts:383]
          └─ 双层循环
```

`SessionRunCoordinator` 负责每个 Session 的序列化执行（同一时刻一个 Session 只有一个 drain 运行）、wake 合并、interrupt 路由。不同 Session 可以并发执行。

## 双层循环结构

### 外层循环（Drain Inputs）

```
while (shouldRun)                          ← 只要有待处理的输入就继续
  ├─  promotion = steer / queue             ← 决定交付模式
  ├─  inner loop                            ← 处理一个完整的 think→act 周期
  └─  check pending queue inputs            ← 全部 steer 消化完后处理 queue
```

外层循环决定 **下一个用户输入是什么**。`steer` 输入在 provider turn 边界插入（立即生效），`queue` 输入在 Session 即将空闲时才提升一个。

### 内层循环（Think → Act）

```
while (needsContinuation)                   ← 只要模型产出 tool call 就继续
  └─ runTurn(sessionID, promotion, step)
      └─ runTurnAttempt(...)
```

内层循环处理 **一个 think→act 周期**。每次迭代 = 一次 LLM provider turn + tool 结果回传后的下一次 provider turn。

## 单轮交互（runTurnAttempt, :173-348）

```
1. 加载 Session、Agent、System Context          ← 观察（Observe）
2. 解析模型（model resolution）                  ← 准备
3. 加载历史，翻译为 LLM Messages                 ← 准备上下文
4. 物化 Tool（根据权限过滤）                      ← 准备能力
5. 检查是否需要 compaction                        ← 内存管理
6. llm.stream(request) → 流式事件                ← 思考（Think）
   ├─ text / reasoning → 持久化
   ├─ tool-call 且非 provider-executed →          ← 行动（Act）
   │   toolMaterialization.settle(call)          ← 在 fiber 中执行 tool
   │   结果发布为 LLMEvent.toolResult
   └─ provider error → 处理
7. 后处理                                        ← 更新状态（Update）
   ├─ 等待所有 tool fiber
   ├─ 处理中断/错误/overflow
   ├─ 发布 Step.Ended 事件（snapshot, tokens, finish）
   └─ 返回 { needsContinuation, step }
```

## 各阶段详解

### 观察（Observe）

由 `SessionInput` 模块（`packages/core/src/session/input.ts`）管理：

- `admit()` — 持久化记录用户输入
- `hasPending(sessionID, delivery)` — 检查是否有待处理的 `steer` 或 `queue` 输入
- `promoteSteers()` — 将 pending steer 提升为可见的用户消息
- `promoteNextQueued()` — 提升一个 queued 输入

输入送达由 `promotion` 参数控制（:187-196）：steer 在下一个 provider-turn 边界立即生效；queue 在 Session 即将空闲时生效。只要有输入被提升，step 计数器重置为 1（表示新的 Agent turn 开始）。

### 思考（Think）

`llm.stream(request)` 调用底层 LLM SDK（:232-275）。请求包含：

- **System context**: 由 `SystemContextRegistry` 加载 + `SkillGuidance` + `ReferenceGuidance` 合并而成
- **History**: `toLLMMessages()` 将 V2 Session 消息翻译为 LLM SDK Message 格式
- **Tools**: `ToolRegistry.Materialization.definitions`（经过权限过滤）
- **Step limit**: 如果达到 `agent.info.steps`，禁用 tool 并注入 `MAX_STEPS_PROMPT`

事件通过 `createLLMEventPublisher` 实时持久化（`publish-llm-event.ts`）。

### 行动（Act）

当流式事件中的 `tool-call` 到达（:243-271）：

1. `needsContinuation = true` — 标记需要继续内层循环
2. 在 `FiberSet` 中启动 tool 执行：`toolMaterialization.settle(call)`
3. tool 完成后的结果通过 `LLMEvent.toolResult` 发布，包含输出路径
4. 多个 tool 并行执行，fiber 间等待使用 `FiberSet.join` + `FiberSet.awaitEmpty`

### 更新状态（Update）

流结束后（:277-347）：

1. **Overflow 处理**: 如果 LLM 返回 context overflow，触发 compaction 后重试
2. **LLM 错误处理**: Provider 错误、中断等 → publish `failUnsettledTools`/`failAssistant`
3. **Tool 错误处理**: 用户拒绝（`PermissionV2.DeclinedError`/`QuestionV2.RejectedError`）→ 中断；其他 tool 错误 → `failUnsettledTools`
4. **Step 结算**: publish `SessionEvent.Step.Ended`（含开始/结束 snapshot、token 用量、finish reason、变更文件列表）
5. **返回决策**: `{ needsContinuation, step }` 给内层循环

### 决策逻辑总结

| 条件 | 行为 |
|------|------|
| LLM 产出 tool call | `needsContinuation = true`，内层循环继续 |
| LLM 产出纯文本回复 | `needsContinuation = false`，内层循环结束 |
| 内层结束 + 有 pending steer | 提升 steer，重置 step=1，再跑一轮内层 |
| 内层结束 + 无 steer + 有 queue | 提升一个 queue，切换 `promotion="queue"`，外层继续 |
| 无任何 pending 输入 | `shouldRun = false`，外层结束 |

## 关键文件

| 文件 | 行号 | 职责 |
|------|------|------|
| `packages/core/src/session/runner/llm.ts` | 383-406 | 主循环（双层 drain loop） |
| `packages/core/src/session/runner/llm.ts` | 369-381 | `runTurn` — 单 provider turn + compaction 重试 |
| `packages/core/src/session/runner/llm.ts` | 173-348 | `runTurnAttempt` — 一次完整的观察→思考→行动→更新 |
| `packages/core/src/session/run-coordinator.ts` | 24-104 | 按 Session ID 串行化 drain、wake 合并、interrupt |
| `packages/core/src/session/execution.ts` | 9-18 | `SessionExecution` 接口定义 |
| `packages/core/src/session/runner/publish-llm-event.ts` | 54-423 | LLM 事件流持久化 |
| `packages/core/src/session/runner/to-llm-message.ts` | 1-171 | Session 消息 → LLM SDK 消息 |
| `packages/core/src/session/runner/model.ts` | 182-216 | 模型解析 |
| `packages/core/src/session/runner/max-steps.ts` | 1-16 | Step 限制提示 |
| `packages/core/src/session/input.ts` | 170-288 | 用户输入管理（admit/promote/hasPending） |
| `packages/core/src/tool/registry.ts` | 50-122 | Tool 物化与执行 |
