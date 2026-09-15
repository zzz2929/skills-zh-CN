---
name: codexhost-delegation
version: 4
description: 将工作委托给另一个编码代理。当用户明确要求Claude Code、Pi、Codex/OpenAI、OMP、Grok、其他代理或被称为@<agent>的代理独立审查、调查、实施、测试或验证某事时使用。当用户仅在讨论、比较或配置代理、选择模型或提供者，或要求当前代理扮演另一个代理时，请勿使用。
---

# Execute the task

Before acting, run:

`codexhost delegate --help`

Treat its output as the sole authoritative source for:

- available commands;
- command parameters;
- available target Harness IDs;
- Thread identifier formats;
- waiting and reading behavior;
- response fields;
- errors and recovery guidance.

Do not construct commands, parameters, or Harness IDs from memory.

When the user asks for a specific Model or Thinking level, inspect the target
Harness first and use the exact opaque IDs returned by the authoritative CLI.
When they do not specify either setting, omit it so the target keeps its default.

Create an independent child session and submit the requested task.

After starting the task, choose the appropriate next action based on the
user’s request and the task:

- send a follow-up message to the same Thread;
- cancel its current Turn;
- read its current state immediately;
- wait for a bounded period;
- check it again later;
- leave it running in the background.

When the result is needed, explicitly read the child Thread. Report only the
visible result returned by that Thread.

Provide the user with the necessary tracking information, including:

- target agent;
- `delegationId`;
- `threadId`;
- `turnId`;
- `deepLink`;
- current or final status.
