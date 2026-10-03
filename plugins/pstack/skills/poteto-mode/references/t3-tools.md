# T3 Code delegation mapping for pstack

T3 Code is not a runtime of its own. A T3 thread runs on a provider's harness, such as Claude Code or Codex, and that harness's mapping still applies: the skills as written on Claude Code, [`codex-tools.md`](codex-tools.md) on Codex. T3 adds one thing, the `t3-code` MCP server, whose `delegate_task` tool starts a child agent on any provider and model the app has configured. Read this file when the session exposes `delegate_task` and a pstack skill dispatches a subagent on a model the harness's own subagent tool cannot run.

The harness prefixes the tool names. Claude Code shows `mcp__t3-code__delegate_task`, and Codex shows `mcp__t3_code__delegate_task`. This file uses the bare names.

## When to use which tool

| Dispatch | Tool |
|----------|------|
| A subagent on a model the harness's own tool accepts (the `Agent` tool on Claude Code, `spawn_agent` on Codex) | The harness's own tool, as the skill and the harness mapping say. Each dispatch field the skill names keeps the meaning it has on that harness. |
| A subagent on another provider's model, or on a model the harness's own tool does not list | `delegate_task` |
| A subagent that writes files | The harness's own tool, with a worktree for each writer as the harness mapping says. See [Subagent policy](#subagent-policy). |

The harness's own model list is not the full list. `orchestrator_capabilities` returns every provider instance with its models, and it is the only source for a `delegate_task` target. Use a provider only when its `canRunChildTask` is true, and when it is not the session's own provider, only when `canRunCrossProviderChildTask` is true too. A provider that cannot run a child lists the reason in `constraints`, such as a missing login.

## Tool actions

| pstack / Claude action | T3 equivalent |
|------------------------|---------------|
| Dispatch a subagent on another provider (the `Agent` tool with a `model`) | `delegate_task` with `task` (the whole prompt), `target.providerInstanceId`, and `target.model`, both copied from `orchestrator_capabilities` |
| `run_in_background: true` | `mode: "async"`, which is the default. The call returns a `taskId` at once. Keep it. |
| A foreground subagent | `mode: "wait"`. `timeoutMs` is only how long you wait. When it runs out the call returns `waitTimedOut: true` and the child keeps running, so keep the `taskId`. |
| Dispatch N parallel subagents in one turn | N `delegate_task` calls in one response, each with its own `clientRequestId` |
| Wait for a subagent result | The completion arrives as a message that names the `taskId`. Then read the result with `task_status`, whose `summary` is the child's final reply. Do not poll and do not sleep. Call `task_status` earlier only when the turn cannot continue without the result. |
| Stop a subagent | `task_cancel` with the `taskId` |
| A label for the subagent (`description`) | `title` |
| `subagent_type` | No equivalent. See [Subagent policy](#subagent-policy). |
| `readonly: true` | No equivalent. See [Subagent policy](#subagent-policy). |
| `isolation: "worktree"` | No equivalent. See [Subagent policy](#subagent-policy). |
| A role value's `@<level>` | An entry in `target.options`. See [Model names](#model-names). |

A `task_status` result is done when `workState` is `result_available`. `waiting_for_children` means the child's turn ended while work it started is still running, so the task is not finished.

Retry a `delegate_task` call with the same `clientRequestId`, so a lost response does not start a second child.

## Subagent policy

poteto-mode's Subagents section applies. These points differ for a `delegate_task` child:

- The child starts with the `task` text and nothing else. It gets no parent history, so the file-pointers rule matters more: name every file by absolute path.
- The child may not see the pstack skills. A provider loads only what is installed for it, and a T3 child saw none of them in the session that verified this mapping. Do not tell a child to invoke a skill by name. Give it the absolute path of the file to read first: `poteto-mode/SKILL.md` for the `pstack:poteto-agent` style, `poteto-mode/references/agents/comment-sicko.md` for `pstack:comment-sicko`, and the reviewer or explorer prompt file the skill names for a panel seat.
- The child runs in the parent's working directory, on the parent's checkout and branch. `delegate_task` has no worktree option, and a child that runs `git worktree add` or `cd` does not move its T3 binding. So use `delegate_task` for read-only seats: panel reviewers, critics, judges, explorers, investigators. Keep writers (code delegates, **swarm** workers, the `orchestrate` and autopilot playbooks) on the harness's own tool, each in its own worktree. On Claude Code that is `isolation: "worktree"`. Codex's `spawn_agent` has no such field, so make the worktree first and name its path in the instructions.
- Nothing enforces read-only. `interactionMode: "plan"` did not stop a Codex child writing a file when this mapping was verified. Say in the `task` that the child must not change files, and check `git status` when a read-only child returns.
- The child has `delegate_task` too. Tell a panel seat not to delegate, so the fan-out stays the size the skill set.
- Provider, model, runtime mode, and interaction mode inherit from the parent unless the call sets them. Leave `runtimeMode` alone: a child that waits for an approval nobody sees does not finish.
- A separate top-level thread (`t3_thread_launch`, `create_threads`) is not a subagent. Use one only when the user asks for a separate thread.
- Keep the rest of the policy unchanged. Review every child's result yourself, and write your own summary.

## Model names

Skills name models by the family names in their Models sections. On T3 those names still go to the harness's own subagent tool, resolved as on that harness: as written on Claude Code, and replaced by your Codex models on Codex. A role value in the override sheet that has the form `<providerInstanceId>/<model>` goes to `delegate_task`, with the part before the first `/` as `target.providerInstanceId` and the rest as `target.model`. For example, `codex/gpt-6-sol` is the Codex instance and its `gpt-6-sol` model. A model id may contain `/` itself.

A diverse-model panel (`arena`, `architect`, `interrogate`, `reflect`) gets its adversarial signal from model diversity, and T3 is where one panel can cross providers. A sheet line such as `interrogate reviewers: <family name>, codex/gpt-6-sol, <another provider>/<model>` runs the first seat on the harness's own tool and the other two through `delegate_task`. The defaults stay on one provider. Cross-provider seats are the user's choice in the sheet, because each one bills another account.

A value's `@<level>` goes in `target.options`. Each model in `orchestrator_capabilities` lists its options, and the reasoning option has a different id per provider: `reasoningEffort` on a Codex model and `effort` on a Claude model when this mapping was verified. Pass `{"<that id>": "<level>"}`, and only a level the model lists. A value without `@` takes the sheet's `default effort` level in the same way. `session` passes no option, and the child then runs at the provider's own default.

Never write a `<providerInstanceId>/<model>` value that `orchestrator_capabilities` did not list in this session.

## Per-skill notes

| Skill | On T3 |
|-------|-------|
| `setup-pstack` | Detect models with `orchestrator_capabilities` as well as the harness's own list. Offer each model of a provider that can run a child as `<providerInstanceId>/<model>`, for the roles that only read: `interrogate reviewers`, `arena cross-judge pool`, and the `how`, `why`, and `reflect` roles. The other roles write files, so they take only the harness's own models. Validate a written value against that list. The sheet path and how it loads are the harness's. Tell the user that a seat on another provider bills that provider's account. |
| `interrogate` | A reviewer seat with a `<providerInstanceId>/<model>` value is a `delegate_task` call. Put the absolute path of the reviewer prompt file and the diff's range in `task`, and state the read-only rule there. |
| `arena` | Candidates write files, so they stay on the harness's own tool. The cross-judge only reads, so a `<providerInstanceId>/<model>` value in the cross-judge pool is a `delegate_task` call. |
| `architect` | The runner panel goes through the **arena** skill, so the same split applies. |
| `how` | Explorers and the explainer only read, so either tool works. |
| `why` | Investigators and the synthesizer only read, so either tool works. A child lists MCP servers from its own provider, not the parent's. |
| `reflect` | The reviewers and the synthesizer only read. Give each the transcript path, because a child on another provider cannot find the parent's transcript. |
| `swarm` | Workers write, so they stay on the harness's own tool, each in its own worktree. |
| `no-comments` | `pstack:comment-sicko` stays on the harness's own tool, because it edits the working tree. |
