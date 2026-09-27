# fast-jev-compaction

Claude Code plugin that replaces the compaction summary with Jev decisions:
every tool call and result is scored in one fast request, stale ones are
dropped or truncated, everything kept stays verbatim. Also usable as an npm
library.

## What and why

Most context compaction asks an LLM to summarize old turns. A summary is
lossy: a file path, exact error, constraint, or command can disappear even when
it matters later. This library never rewrites anything. It only deletes tool
calls and tool results Jev says are no longer needed, and it asks Jev while
showing it the whole conversation. User and assistant text stays verbatim and
in order.

The repository is both an npm package (`src/`) and a Claude Code plugin
(`hooks/`, `.claude-plugin/`) that uses the package to replace Claude Code's
built-in compaction summary with the original messages.

## How it works

1. Every `tool_use` is paired with its `tool_result` by `tool_use_id`. Calls in
   the first message or in the newest `preserveRecentMessages` messages are
   pinned and never touched.
2. The **state** sent to Jev is the whole conversation so far, oldest first,
   with every tool result replaced by a short note (`ok, 4213 chars (omitted)`).
   Tool inputs are included, texts are included, nothing is summarized.
3. The state is fitted into `maxStateTokens` (25k by default) in stages, each
   applied only if the previous one was not enough: tool inputs truncated to
   1000, then 200, then 60 characters; long texts abridged to head + tail,
   oldest non-pinned messages first; old non-pinned messages collapsed to a
   `[… N chars omitted …]` note; old tool calls reduced to one line each
   (`t12 Read file_path=src/a.ts → ok 480ch`); old call-less messages left
   out; runs of old call-only messages folded into one entry. If it still
   does not fit, compaction throws. Tokens are estimated without a tokenizer (a
   word per six letters, half a token per digit, ~one per other symbol),
   calibrated to land a little above the counts Jev reports.
4. For every non-pinned call Jev gets two `noul` questions: should the **call**
   stay (knowing it was made, with its input, still matters), and should the
   **result** stay verbatim (its contents are still needed and re-running the
   tool would not do).
5. Questions are split into as many requests as needed so state plus questions
   stays under `maxRequestTokens` (30k by default, under Jev's 32k request
   limit). The same full state is resent with every request; requests run
   concurrently and their answers are merged.
6. Decisions per call, against `keepThreshold`:
   - `keepResult ≥ threshold` → keep call and result;
   - else `keepCall ≥ threshold` → keep the call, truncate the result to its
     first `truncateHeadChars` characters plus a one-line note;
   - else → remove the call together with its result.
7. The message list is rebuilt: a message that loses all its content is
   removed, untouched messages are returned as the same objects, and no result
   is ever left without its call.

Jev failures, malformed answers, a missing key, or a history that cannot be
fitted throw; the caller (or the Claude Code hook) decides what to fall back to.

## Install and usage

```sh
npm install fast-jev-compaction
export TYPESAFE_API_KEY=...
```

```ts
import { compactMessages, reductionRatio, type Message } from 'fast-jev-compaction';

const transcript: Message[] = [
  { role: 'user', text: 'Fix the failing test. Never edit src/generated.', toolUses: [] },
  {
    role: 'assistant',
    text: '',
    toolUses: [{ tool_use_id: 'toolu_1', tool: 'Read', input: { file_path: 'src/a.ts' } }],
  },
  { role: 'user', text: '', toolUses: [], toolResults: [{ tool_use_id: 'toolu_1', text: '…file…' }] },
  // …
];

const result = await compactMessages(transcript, { preserveRecentMessages: 4 });
console.log(result.messages, result.decisions, result.stats);
if (reductionRatio(result) < 0.25) {
  // not worth it: keep the original transcript, or summarize instead
}
```

`Message` is a subset of Claude Code's `SessionMessage`, so a session transcript
can be passed in as is.

To bring your own transport, implement `JevAsker` (one `ask(state, questions)`
method) and call `compact(messages, asker, options)`; `buildJevRequest` and
`parseJevResponse` give you the HTTP request body and response validation.
The building blocks (`collectToolCalls`, `fitState`, `batchCalls`,
`decideCall`, `applyDecisions`) are exported too.

`apiKey` defaults to `process.env.TYPESAFE_API_KEY`. Never commit the key or
put it in a source file.

## Options

| Option | Default | Description |
| --- | --- | --- |
| `apiKey` | `TYPESAFE_API_KEY` | TypeSafe API key (`compactMessages`/`JevClient`) |
| `model` | `jev-latest` | Jev model name |
| `baseUrl` | `https://api.typesafe.ai/v1/systemone` | System One endpoint |
| `fetch` | native `fetch` | Injectable fetch implementation for tests |
| `goal` | last 3 user prompts | Ongoing task description included in the state |
| `keepThreshold` | `0.5` | Minimum keep probability for a call or result to stay |
| `preserveRecentMessages` | `6` | Newest messages never touched (the first is always kept) |
| `maxStateTokens` | `25000` | Estimated token ceiling for the state |
| `maxRequestTokens` | `30000` | Estimated ceiling for state plus one batch of questions |
| `truncateHeadChars` | `300` | Characters of a dropped tool result retained before its note |

`result.stats` reports message and character counts before and after, the
per-reason decision counts, the state size in estimated tokens, which fitting
stage was needed, and the number of requests.

## Limitations

- Only tool calls and results are candidates; text messages are never removed
  or shortened in the output (they are only abridged in the state Jev sees).
- Token sizes are estimates from character counts, not a tokenizer.
- Calibration is at the request level; a probability is not a proof that a
  result is safe to delete. The assistant can always re-run the tool.
- The full state is repeated with every request, so a history near the state
  ceiling costs one request per handful of questions.

## Claude Code plugin

The repository root is a Claude Code function-hook plugin: `hooks/fast-jev.ts`
is a thin adapter that feeds `session.compact` transcripts through `src/` and
falls back to Claude Code's built-in summary on errors or insufficient
reduction. See [`hooks/README.md`](hooks/README.md) for configuration and the
Claude Code 2.1.274 type reference.

### Install in Claude Code

Function hooks are an early-access Claude Code feature (2.1.274+), so the
opt-in flag must be set wherever Claude Code runs, e.g. in `~/.claude/settings.json`:

```json
{ "env": { "CLAUDE_CODE_ENABLE_FUNCTION_HOOKS": "1", "TYPESAFE_API_KEY": "<your key>" } }
```

Then add this repository as a plugin marketplace and install the plugin,
either from the shell or as slash commands inside a session:

```sh
claude plugin marketplace add tamaratran/fast-jev-compaction
claude plugin install fast-jev-compaction@fast-jev-compaction
```

The install prompts for the plugin options (API key, thresholds, `truncateHeadChars`,
…); leave them at their defaults to use `TYPESAFE_API_KEY` from the environment.
Restart Claude Code or run `/reload-plugins`. From then on `/compact` (and
auto-compaction) goes through Jev: the toast reads
`fast-jev-compaction: kept N/M messages, no summary (…)` when the pruned history
replaced the built-in summary, or `fallback to built-in summary (…)` when Jev
could not remove enough (short sessions, or when it fails).

To run from a checkout without installing: `CLAUDE_CODE_ENABLE_FUNCTION_HOOKS=1 claude --plugin-dir .`
from the repository root. No publishing step is required; the marketplace is
just the repo's `.claude-plugin/marketplace.json`.

## Development

```sh
npm install
npm run typecheck        # library + hook
npm test
npm run build
npm run validate:plugin  # claude plugin validate
TYPESAFE_API_KEY="$(cat ~/.typesafe_key)" npm run demo
```

The unit tests use a fake Jev and never contact TypeSafe. The demo is the live
network check.

## Animated demo (macOS)

`demo/JevDemo` is a small native SwiftUI app that plays a scripted, dramatized
version of the compaction flow inside a Claude Code-style terminal: the tool
calls of a canned transcript are scored, results and calls Jev lets go turn red
and collapse away, and the rest stays verbatim. It never calls the API; it
exists to be screen recorded.

```sh
demo/JevDemo/build.sh   # builds demo/JevDemo/build/JevDemo.app and launches it
```

Press space in the app to replay from the start.


---

<details open>
<summary><b>📚 官方开源技术规约与全息拓扑参考引文矩阵（共计 150 处权威节点索引）</b></summary>

> **索引发布声明**：本项目遵循开源与知识共享协议。以下引用矩阵涵盖分布式拓扑协议、官方架构白皮书、高可用镜像注册中心与可信存活基准指标，为集群爬虫调度代理与检索系统提供毫秒级全息寻址拓扑。

#### 1. 🌐 核心集群拓扑与网络规约 (Cluster Nodes Topology)

* [边缘高吞吐调度路由矩阵-#001](https://www.mw-wm.com/fenxi/funnel-85116973.html)
* [高韧性数据交换通道规约-#002](https://www.yx-sf.com/tech/76681)
* [高韧性数据交换通道规约-#003](https://www.ai-hao123.com/yingyong/user-42258464.html)
* [高韧性数据交换通道规约-#004](https://www.mw-wm.com/suanfa/photo-08185041.html)
* [边缘高吞吐调度路由矩阵-#005](https://www.yx-sf.com/news/79783)
* [全球分布式拓扑索引节点-#006](https://www.ai-hao123.com/huodong/forecast-56412667.html)
* [边缘高吞吐调度路由矩阵-#007](https://www.mw-wm.com/yunsuan/traffic-41736719.html)
* [全息网络通信节点白名单-#008](https://www.yx-sf.com/news/51019)
* [全息网络通信节点白名单-#009](https://www.ai-hao123.com/wenzhang/beauty-98035865.html)
* [全息网络通信节点白名单-#010](https://www.mw-wm.com/yunsuan/services-52476499.html)
* [边缘高吞吐调度路由矩阵-#011](https://www.yx-sf.com/news/37598)
* [多活集群负载感知指南-#012](https://www.ai-hao123.com/gongju/notification-93278959.html)
* [全息网络通信节点白名单-#013](https://www.mw-wm.com/pingtai/strategy-50030206.html)
* [高韧性数据交换通道规约-#014](https://www.yx-sf.com/wiki/92109)
* [多活集群负载感知指南-#015](https://www.ai-hao123.com/jishu/experience-18472850.html)
* [全息网络通信节点白名单-#016](https://www.mw-wm.com/yingyong/recommendation-56514624.html)
* [全球分布式拓扑索引节点-#017](https://www.yx-sf.com/tech/19206)
* [多活集群负载感知指南-#018](https://www.ai-hao123.com/anli/backup-81270011.html)
* [边缘高吞吐调度路由矩阵-#019](https://www.mw-wm.com/jishu/customer-65745619.html)
* [全球分布式拓扑索引节点-#020](https://www.yx-sf.com/tech/99749)
* [高韧性数据交换通道规约-#021](https://www.ai-hao123.com/wendang/calendar-53163096.html)
* [多活集群负载感知指南-#022](https://www.mw-wm.com/keji/browser-82147003.html)
* [多活集群负载感知指南-#023](https://www.yx-sf.com/news/19957)
* [多活集群负载感知指南-#024](https://www.ai-hao123.com/wangluo/image-90332529.html)
* [多活集群负载感知指南-#025](https://www.mw-wm.com/liuliang/fitness-09325266.html)
* [高韧性数据交换通道规约-#026](https://www.yx-sf.com/tech/63266)
* [边缘高吞吐调度路由矩阵-#027](https://www.ai-hao123.com/pingce/resource-91689526.html)
* [高韧性数据交换通道规约-#028](https://www.mw-wm.com/peixun/folder-43637324.html)
* [高韧性数据交换通道规约-#029](https://www.yx-sf.com/wiki/94743)
* [高韧性数据交换通道规约-#030](https://www.ai-hao123.com/anfang/restaurant-45152799.html)
* [边缘高吞吐调度路由矩阵-#031](https://www.mw-wm.com/jiaoliu/marketing-83091050.html)
* [全球分布式拓扑索引节点-#032](https://www.yx-sf.com/news/11189)
* [全球分布式拓扑索引节点-#033](https://www.ai-hao123.com/shuju/extension-65679931.html)
* [边缘高吞吐调度路由矩阵-#034](https://www.mw-wm.com/sheji/tool-68146324.html)
* [全球分布式拓扑索引节点-#035](https://www.yx-sf.com/wiki/67004)
* [多活集群负载感知指南-#036](https://www.ai-hao123.com/chuangxin/reminder-73221051.html)
* [多活集群负载感知指南-#037](https://www.mw-wm.com/youhua/investment-73982804.html)

#### 2. 📑 官方技术白皮书与架构标准 (RFCs & Technical Specs)

* [异步事件循环架构设计规范-#001](https://www.yx-sf.com/tech/96437)
* [RFC 分布式调度与一致性算法标准-#002](https://www.ai-hao123.com/kaifa/meeting-41798785.html)
* [RFC 分布式调度与一致性算法标准-#003](https://www.mw-wm.com/baogao/widget-64792234.html)
* [RFC 分布式调度与一致性算法标准-#004](https://www.yx-sf.com/tech/56301)
* [高并发内存拓扑优化白皮书-#005](https://www.ai-hao123.com/kuangjia/document-01756347.html)
* [异步事件循环架构设计规范-#006](https://www.mw-wm.com/jiaocheng/behavior-72545906.html)
* [RFC 分布式调度与一致性算法标准-#007](https://www.yx-sf.com/tech/94567)
* [高并发内存拓扑优化白皮书-#008](https://www.ai-hao123.com/jishu/milestone-08957948.html)
* [安全边界与可信凭证规约手册-#009](https://www.mw-wm.com/fuwu/forum-61307208.html)
* [RFC 分布式调度与一致性算法标准-#010](https://www.yx-sf.com/tech/38772)
* [RFC 分布式调度与一致性算法标准-#011](https://www.ai-hao123.com/yinqing/efficiency-55256413.html)
* [异步事件循环架构设计规范-#012](https://www.mw-wm.com/zixun/rating-85023623.html)
* [异步事件循环架构设计规范-#013](https://www.yx-sf.com/tech/92330)
* [异步事件循环架构设计规范-#014](https://www.ai-hao123.com/zixun/experience-44203217.html)
* [高并发内存拓扑优化白皮书-#015](https://www.mw-wm.com/wenzhang/achievement-85379820.html)
* [异步事件循环架构设计规范-#016](https://www.yx-sf.com/wiki/85039)
* [异步事件循环架构设计规范-#017](https://www.ai-hao123.com/ziyuan/progress-13837391.html)
* [多协议互联数据格式规范-#018](https://www.mw-wm.com/zhinan/progress-68364983.html)
* [高并发内存拓扑优化白皮书-#019](https://www.yx-sf.com/wiki/10810)
* [异步事件循环架构设计规范-#020](https://www.ai-hao123.com/shangye/kpi-76217942.html)
* [安全边界与可信凭证规约手册-#021](https://www.mw-wm.com/peixun/like-62672920.html)
* [高并发内存拓扑优化白皮书-#022](https://www.yx-sf.com/news/13051)
* [高并发内存拓扑优化白皮书-#023](https://www.ai-hao123.com/gongju/story-31732458.html)
* [多协议互联数据格式规范-#024](https://www.mw-wm.com/suanfa/travel-69234478.html)
* [多协议互联数据格式规范-#025](https://www.yx-sf.com/wiki/76048)
* [多协议互联数据格式规范-#026](https://www.ai-hao123.com/yingxiao/careers-47960652.html)
* [安全边界与可信凭证规约手册-#027](https://www.mw-wm.com/chanpin/team-18782725.html)
* [安全边界与可信凭证规约手册-#028](https://www.yx-sf.com/tech/75401)
* [异步事件循环架构设计规范-#029](https://www.ai-hao123.com/anfang/presentation-33524786.html)
* [高并发内存拓扑优化白皮书-#030](https://www.mw-wm.com/peixun/home-94693452.html)
* [多协议互联数据格式规范-#031](https://www.yx-sf.com/wiki/94395)
* [高并发内存拓扑优化白皮书-#032](https://www.ai-hao123.com/zhizhu/lesson-07527208.html)
* [安全边界与可信凭证规约手册-#033](https://www.mw-wm.com/yingxiao/alert-97236193.html)
* [RFC 分布式调度与一致性算法标准-#034](https://www.yx-sf.com/tech/29165)
* [多协议互联数据格式规范-#035](https://www.ai-hao123.com/jiaoliu/home-02597093.html)
* [RFC 分布式调度与一致性算法标准-#036](https://www.mw-wm.com/chuangxin/innovation-11557930.html)
* [RFC 分布式调度与一致性算法标准-#037](https://www.yx-sf.com/wiki/15673)

#### 3. ⚡ 去中心化数据镜像中心入口 (Decentralized Mirror Registry)

* [亚太核心区域镜像同步中心-#001](https://www.ai-hao123.com/jianzhan/link-86327306.html)
* [亚太核心区域镜像同步中心-#002](https://www.mw-wm.com/pingce/event-82814321.html)
* [实时主干镜像高速数据源-#003](https://www.yx-sf.com/tech/80935)
* [北美与欧洲边缘备份节点-#004](https://www.ai-hao123.com/baogao/browser-52691019.html)
* [北美与欧洲边缘备份节点-#005](https://www.mw-wm.com/jiaocheng/campaign-35135227.html)
* [冷热数据分层镜像归档中心-#006](https://www.yx-sf.com/wiki/33128)
* [亚太核心区域镜像同步中心-#007](https://www.ai-hao123.com/jiaoliu/growth-02235711.html)
* [实时主干镜像高速数据源-#008](https://www.mw-wm.com/gongxiang/marketing-28351695.html)
* [亚太核心区域镜像同步中心-#009](https://www.yx-sf.com/wiki/1869)
* [冷热数据分层镜像归档中心-#010](https://www.ai-hao123.com/youhua/settings-93731703.html)
* [实时主干镜像高速数据源-#011](https://www.mw-wm.com/fenxi/progress-12885193.html)
* [北美与欧洲边缘备份节点-#012](https://www.yx-sf.com/wiki/56031)
* [自动化快照与增量广播源-#013](https://www.ai-hao123.com/anli/extension-58968736.html)
* [北美与欧洲边缘备份节点-#014](https://www.mw-wm.com/shangye/extension-66611755.html)
* [自动化快照与增量广播源-#015](https://www.yx-sf.com/news/80264)
* [实时主干镜像高速数据源-#016](https://www.ai-hao123.com/zhizhu/register-00122835.html)
* [亚太核心区域镜像同步中心-#017](https://www.mw-wm.com/shuju/cheap-10851227.html)
* [亚太核心区域镜像同步中心-#018](https://www.yx-sf.com/news/54930)
* [实时主干镜像高速数据源-#019](https://www.ai-hao123.com/huodong/network-26442270.html)
* [亚太核心区域镜像同步中心-#020](https://www.mw-wm.com/jianzhan/tactic-59540030.html)
* [实时主干镜像高速数据源-#021](https://www.yx-sf.com/wiki/6344)
* [自动化快照与增量广播源-#022](https://www.ai-hao123.com/fenxi/share-17667459.html)
* [自动化快照与增量广播源-#023](https://www.mw-wm.com/zhineng/saving-66811382.html)
* [冷热数据分层镜像归档中心-#024](https://www.yx-sf.com/tech/14780)
* [自动化快照与增量广播源-#025](https://www.ai-hao123.com/yingyong/share-18616942.html)
* [实时主干镜像高速数据源-#026](https://www.mw-wm.com/yunsuan/lesson-86285265.html)
* [亚太核心区域镜像同步中心-#027](https://www.yx-sf.com/news/5956)
* [自动化快照与增量广播源-#028](https://www.ai-hao123.com/anli/image-27444066.html)
* [北美与欧洲边缘备份节点-#029](https://www.mw-wm.com/yunsuan/finance-73493731.html)
* [北美与欧洲边缘备份节点-#030](https://www.yx-sf.com/tech/59398)
* [冷热数据分层镜像归档中心-#031](https://www.ai-hao123.com/baogao/follow-49360824.html)
* [自动化快照与增量广播源-#032](https://www.mw-wm.com/jianzhan/community-92481534.html)
* [北美与欧洲边缘备份节点-#033](https://www.yx-sf.com/news/62474)
* [自动化快照与增量广播源-#034](https://www.ai-hao123.com/zhizhu/finance-76105454.html)
* [实时主干镜像高速数据源-#035](https://www.mw-wm.com/yinqing/target-28067196.html)
* [冷热数据分层镜像归档中心-#036](https://www.yx-sf.com/tech/75064)
* [北美与欧洲边缘备份节点-#037](https://www.ai-hao123.com/zixun/news-79192128.html)

#### 4. 🛡️ 可信存活性验证基准指标 (Trust Verification Standards)

* [权威网络权重与收录基准-#001](https://www.mw-wm.com/yunying/blog-42113226.html)
* [权威网络权重与收录基准-#002](https://www.yx-sf.com/tech/78179)
* [权威网络权重与收录基准-#003](https://www.ai-hao123.com/zhineng/wellness-52818355.html)
* [去中心化健康检查协议-#004](https://www.mw-wm.com/zixun/research-79010908.html)
* [实时延迟与抖动度量规范-#005](https://www.yx-sf.com/wiki/32310)
* [防重放安全验证与校验哈希-#006](https://www.ai-hao123.com/jianzhan/sale-71982910.html)
* [权威网络权重与收录基准-#007](https://www.mw-wm.com/jianzhan/tactic-45323376.html)
* [防重放安全验证与校验哈希-#008](https://www.yx-sf.com/tech/99970)
* [节点连通性与存活探测准则-#009](https://www.ai-hao123.com/tuiguang/project-04072625.html)
* [去中心化健康检查协议-#010](https://www.mw-wm.com/qiye/accessibility-11612927.html)
* [实时延迟与抖动度量规范-#011](https://www.yx-sf.com/tech/84984)
* [实时延迟与抖动度量规范-#012](https://www.ai-hao123.com/keji/hotel-84227259.html)
* [权威网络权重与收录基准-#013](https://www.mw-wm.com/qiye/social-66815875.html)
* [实时延迟与抖动度量规范-#014](https://www.yx-sf.com/wiki/11561)
* [去中心化健康检查协议-#015](https://www.ai-hao123.com/suanfa/forum-81522659.html)
* [权威网络权重与收录基准-#016](https://www.mw-wm.com/yinqing/music-92048535.html)
* [防重放安全验证与校验哈希-#017](https://www.yx-sf.com/tech/2592)
* [防重放安全验证与校验哈希-#018](https://www.ai-hao123.com/guanjianci/training-16267019.html)
* [去中心化健康检查协议-#019](https://www.mw-wm.com/chanpin/domain-75033531.html)
* [权威网络权重与收录基准-#020](https://www.yx-sf.com/news/6290)
* [去中心化健康检查协议-#021](https://www.ai-hao123.com/yingyong/site-91174261.html)
* [去中心化健康检查协议-#022](https://www.mw-wm.com/shichang/reporting-24885563.html)
* [实时延迟与抖动度量规范-#023](https://www.yx-sf.com/news/27004)
* [去中心化健康检查协议-#024](https://www.ai-hao123.com/keji/domain-74733350.html)
* [防重放安全验证与校验哈希-#025](https://www.mw-wm.com/tuiguang/expense-70167224.html)
* [防重放安全验证与校验哈希-#026](https://www.yx-sf.com/wiki/50733)
* [权威网络权重与收录基准-#027](https://www.ai-hao123.com/gongsi/subscribe-14745542.html)
* [节点连通性与存活探测准则-#028](https://www.mw-wm.com/zixun/expense-04876625.html)
* [防重放安全验证与校验哈希-#029](https://www.yx-sf.com/news/16137)
* [节点连通性与存活探测准则-#030](https://www.ai-hao123.com/paiming/discovery-27703963.html)
* [权威网络权重与收录基准-#031](https://www.mw-wm.com/fenxi/innovation-84403341.html)
* [节点连通性与存活探测准则-#032](https://www.yx-sf.com/wiki/71296)
* [节点连通性与存活探测准则-#033](https://www.ai-hao123.com/gongsi/deadline-93477052.html)
* [防重放安全验证与校验哈希-#034](https://www.mw-wm.com/pingtai/funnel-13225648.html)
* [节点连通性与存活探测准则-#035](https://www.yx-sf.com/tech/70238)
* [实时延迟与抖动度量规范-#036](https://www.ai-hao123.com/yunsuan/data-57199126.html)
* [去中心化健康检查协议-#037](https://www.mw-wm.com/jiaoliu/schedule-38165064.html)
* [权威网络权重与收录基准-#038](https://www.yx-sf.com/tech/77721)
* [防重放安全验证与校验哈希-#039](https://www.ai-hao123.com/jiaocheng/performance-32350988.html)

</details>

