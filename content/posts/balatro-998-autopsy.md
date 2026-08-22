---
title: "【AI 实验】一局 99.8 的解剖：LLM 在 Balatro 里是怎么学会赢的"
date: 2026-08-22T12:30:00+08:00
draft: false
tags:
  - LLM
  - Agent
  - Balatro
  - 记忆系统
categories:
  - AI 实验
summary: "逐层解剖第一局打到 Ante 12 的对局（99.8 分）：开局注入了什么攻略、13 次召回如何让攻略在局中换了两代、每步 6000+ token 的上下文长什么样、决胜时刻模型打出单张 8 留满手钢 K 时的原始推理。最值得写的发现：这局的胜利路径，是两局之前一次失败反思里写下的处方。"
---

## TL;DR

Memory Arena 跑出了第一局打到 Ante 12 的对局,score 99.8,含公共榜在内是历史最好成绩。523 步,92 分钟,376 万 token,1 次非法操作。

这篇把这局切开看:开局时模型"知道"什么、局中 13 次召回怎么让攻略中途换了两代、每一步的 prompt 有多长、决胜时刻模型在推理里说了什么。最值得写的是一件事:两局之前,它输在 Baron 这个构建上,反思里写下了怎么办;这局,它照着办了。每一步都有留档,可以回放。

## 1. 局的档案

| | |
|---|---|
| arm | `gemini37-flash-high-cur3`(file-full 记忆组) |
| 局 | train-r3 / BENCH05,RED 牌组,白注 |
| 结局 | won,Ante 12,score 99.8,非法 1 次 |
| 规模 | 523 步,92 分钟 |
| 成本 | tokens in 3,205,380 + out 556,834 |
| 动作构成 | 商店购买 93、pack 选卡 69、用消耗牌 56、出牌 56、reroll 49、弃牌 49、重排小丑 21、卖小丑 19 |

最后两个数字值得多看一眼:21 次重排、19 次卖。这不是"每回合打出能打的最大牌"的那种对局,是一局一直在调整资产负债表的运营局。

它也不是这组的第一局。开局时账本里已经躺着 6 局的反思(其中 3 局 error)和一份整合过的攻略。

## 2. 开局:模型"知道"什么

游戏开局时,记忆组件把一个 4,247 字节的 block 注进 system prompt,三段结构:攻略、本局笔记、近期反思。攻略段原文节选:

```
== STRATEGY MEMORY (lessons from your previous games) ==
### COMMON MISTAKES TO AVOID
- Holding utility jokers like DNA or Juggler into late Antes instead of
  swapping for pure scoring or xMult multipliers.
- Relying on Baron or in-hand xMult without sufficient base chips, hand
  levels, or retrigger effects.
- Mixing conflicting hand strategies (e.g., playing Two Pair scaling while
  holding High Card xMult synergies).
### EARLY GAME (Ante 1-2)
- Target Full House or Flush in Ante 1 to clear blinds in 1-2 hands
  unassisted.
...
== THIS RUN NOTES ==
Seed BENCH05. No established build yet.
```

> 译:== 策略记忆 == 常见错误:把 DNA、Juggler 这类功能牌拖到后期不换成 xMult;在基础筹码、牌型等级、重触发不足时就押 Baron;混合互相冲突的牌型路线。前期:Ante 1 打葫芦或同花,一到两手清场。== 本局笔记 == 种子 BENCH05,尚未成型。

但开局 block 里信息量最大的不是攻略,是一条反思。两局之前的 BENCH03(89.4 分,死在 Ante 11),原文节选:

```
[1] This run attempted a King-heavy Baron strategy (1.5x Multiplier per
King) with Blueprint to double the Baron effect... The run reached Ante 11,
but fell short of the blind requirements.
| failed: The single-card hands weren't enough to trigger DNA consistently
in a high-stakes environment.; Lacked the consistent chip/multiplier
scaling required for the high Ante 11 blind (tens of millions).
| lessons: Baron needs significant base chip/multiplier scaling to be
effective in Ante 11+. ... A successful Baron build needs Mime (to
retrigger Kings), Red Seals on Kings, Steel Kings, and enough King density.
| next: Prioritize Mime, Steel Kings, and Red Seals for a Baron build. ...
```

> 译:[1] 这局尝试了 K 密度 Baron 打法(每张手持 K ×1.5)配 Blueprint 复制……推进到 Ante 11,盲注要求没够到。失败原因:单张出牌无法稳定触发 DNA;缺 Ante 11 位数级盲注需要的持续筹码和乘数增长。教训:Baron 在 Ante 11+ 需要大量基础增长……一个成功的 Baron 构建需要 Mime(重触发 K)、K 上的红印、钢 K,和足够的 K 密度。下一步:为 Baron 构建优先找 Mime、钢 K、红印。

记住这条处方,第 5 节它会回来。

顺带一个不那么光彩的观察:开局注入的另外两条反思来自 error 局,内容是"游戏在 0 号动作前就报错终止,没有教训可总结,但记录此类事件有助于识别 bug 模式"。error 局的反思也进账本,现在它们只占注入空间,不贡献信息。该过滤。

## 3. 局中:攻略的活体换血

这局打了 92 分钟,期间记忆组件一共做了 13 次召回(开局一次,每个 ante 结算后各一次),每次注入的完整文本都留了档:

| 时机 | 注入 block | 攻略版本 |
|---|---|---|
| 开局 | 4,247 B | v-train-r2:BENCH03 |
| ante 1-2 检查点 | 4,247-4,930 B | 同上 |
| ante 3 检查点 | 7,303 B | v-train-r3:BENCH04,第一次换血 |
| ante 4 检查点起 | 8,306 B | v-train-r3:BENCH08,第二次,此后稳定 |
| ante 12 检查点 | 8,353 B | 同上,本局笔记涨到 28 条 |

背景是这局跑在 3 号车道的同时,并行车道上别的局相继结束,各自发布了自己整合的新攻略。本局每个 ante 结束后做检查点召回,先后拉到了两代。注入的"前人经验"从 3 局的反思量长到 6 局,错误清单从 3 条涨到 8 条以上。到 ante 12 时,攻略"常见错误"的第一条长这样:

```
- Committing to Baron or in-hand xMult builds without Mime, Red Seal
  Steel Kings, and sufficient base chips/Planet levels.
```

> 译:在没有 Mime、红印钢 K 和足够的基础筹码、星球卡等级时,就押注 Baron 或手持 xMult 构建。

这就是 BENCH03 那次失败被整合器固化进攻略后的形态。同一时间,本局笔记每个盲注加一条,从"No established build"一路记到 ante 12 的完整构建状态。模型对自己的当前局,也有滚动记忆。

## 4. 每一步的上下文有多长

523 步的 tokensIn(prompt 长度,含 system 注入、局面序列化、工具定义):

| step 区间 | min | 平均 | max |
|---|---|---|---|
| 0-99 | 2,345 | 5,682 | 6,830 |
| 100-199 | 2,057 | 6,221 | 7,162 |
| 200-299 | 2,156 | 6,239 | 7,238 |
| 300-399 | 2,488 | 6,186 | 7,146 |
| 400-499 | 2,152 | 6,234 | 7,173 |
| 500-522 | 6,038 | 6,486 | 6,866 |

单步 prompt 稳定在 6,200 token 上下。注意模型每步都是无历史的:两步之间不累积对话,每步的 6,200 token 是全量重发。这也是一局 320 万 input token 的由来,平均 6,129 乘 523 步。

曲线是平的。攻略从 4 KB 换到 8 KB,平均步长只抬了 500 token 左右,因为局面文本(手牌和商店内容随构建复杂化会变长)和攻略长度互相抵消了一部分。8 KB 的注入预算,可控。偶发的低谷(2,057)来自错误重试一类的特殊步骤。

每步产出(reasoning 加决策)平均 1,064 token out。整局成本约等于一次中等规模的离线评测任务。换个说法:买一局 Ante 12,花了 376 万 token。

## 5. 决胜时刻:Ante 12,The Window

最后两步的留档,模型 reasoning 原文:

```
[step 521 · use_consumable]
**My Turn: Boss "The Window" Showdown**
Okay, here's the situation: Ante 12, Round 37, facing "The Window," which
means all Diamond cards are debuffed. Luckily, I don't have any Diamonds
in hand—just eight Steel Heart Kings and a Club 8. I have three hands and
three discards left.

[step 522 · play_hand {"cards":[8]}]
**Analyzing the Play: A Calculated High Card**
Okay, the state is Ante 12, facing "The Window" boss with a Diamond card
debuff, and we need 600,000,000 chips. ... Let's see, the hand is stacked:
eight Steel Heart Kings and an 8 of Club...
```

> 译:[第 521 步] Boss"窗口"决战:Ante 12 第 37 轮,"窗口"的效果是所有方块牌失效。运气不错,我手里没有方块,只有八张钢制红心 K 和一张梅花 8。还剩 3 次出牌 3 次弃牌。
> [第 522 步] 分析这次出牌:一次精心计算的单张。目标 6 亿筹码。手里是八张钢制红心 K 和一张梅花 8……

然后它执行的动作:打出那张梅花 8,八张钢 K 全部留在手里。

这是这套构建的全力一击。High Card 只吃基础分,但留在手里的每张钢 K 被 Baron 触发一次 ×1.5,再吃钢牌自身的 ×1.5,八个留手乘数叠在 Blueprint 复制过的 Baron 上。打出去的是单张垃圾牌,赢的是满手的 K。目标 6 亿,过了。`won: true`。

第 2 节那条处方,"成功的 Baron 构建需要钢 K 和足够的 K 密度",在这里被逐字执行:K 密度靠 DNA 复制堆到 8 张,全是钢制。也正因为 DNA 复制的是红心 K,天然躲开了 The Window 的方块克制。从失败反思到攻略注入到决胜出牌,这条经验走完了它的闭环。

## 6. 它自己怎么总结这局

局末反思器让模型复盘,它写的(节选):

```
summary: This was a King/Steel scaling engine. Early on, DNA was crucial:
it duplicated Steel Kings, which, combined with Blueprint, meant I was
getting two of these key cards every hand. ... Blueprint's flexibility was
key, first copying DNA, then switching to Baron ...
whatWorked: ["DNA + Blueprint synergy: I was able to generate 2 Steel Kings
every hand.", "Baron + Steel Joker + Steel Kings in hand: provided
excellent multiplicative triggers.", "Joker repositioning: Moving
Blueprint strategically to copy DNA and then Baron proved extremely
effective."]
lessons: ["Mid-round joker repositioning ... is a powerful, dual-phase
strategy.", "DNA is a triple threat: it generates deck value, scales mult
permanently, and provides held-in-hand trigger targets simultaneously."]
nextTime: ["Prioritize Red Seals on Kings early via Spectral packs...",
"Look for Mime to amplify Steel King triggers alongside Baron.", ...]
```

> 译:这是一台 K 与钢牌的增长引擎。前期 DNA 是关键:它复制钢 K,配合 Blueprint,我每回合白得两张核心牌……Blueprint 的灵活性是关键,先复制 DNA,后期切去复制 Baron……奏效的:DNA 加 Blueprint 每回合产 2 张钢 K;Baron、钢牌小丑加手持钢 K 提供了优秀的乘法触发;比赛中途重排小丑位置(Blueprint 从复制 DNA 移到复制 Baron)极其有效。教训:DNA 是三重威胁,它同时产出牌库价值、永久增长乘数、手持触发目标。下次:更早用幽灵包给 K 上红印;找 Mime 配 Baron 放大钢 K 触发。

注意最后一句。它的"下次"又往前走了一步:红印、Mime,正是 BENCH03 反思里提过、这局没集齐的两件套。经验链没有合上,它在继续往上摞。

## 7. 分析

归因链条很清楚。BENCH03 的失败反思把"Baron 在 Ante 11 缺乘数增长"写得非常具体,要钢 K、要 K 密度、DNA 复制要对准目标;整合器把教训固化进攻略,开局就注入;并行车道的新反思又让攻略在 ante 3 和 ante 4 两度增补;模型在局中确实按攻略执行了"功能牌后期换分"的纪律,19 次卖小丑、21 次重排,包括把 Blueprint 从 DNA 挪到 Baron 的关键一手;最后那记单张 8,是整套知识的直接应用。

归因也要克制。种子运气真实存在:红心 K 躲开方块克制,商店刷牌也配合。一局样本说明不了胜率,同一晚同组的 eval 两局都死在 Ante 4,训练种子上的进步和泛化是两回事。这局真正给的东西,是把"它赢的时候,经验系统在干什么"完整拍了下来:13 次召回的注入全文、523 步每步的 prompt 长度和推理原文、局末反思。每一环都可以回放。

后面几件事值得做。error 局的空反思该过滤,别让"记录此类事件有助于识别 bug 模式"这种占位文本挤占注入预算。注入的攻略随局数线性膨胀,4 KB 到 8 KB 只用了八局,20 局以上需要压缩或摘要策略。每步全量重发 6,200 token 是无历史架构的代价,其中攻略段在 92 分钟里基本不变,是可以吃 prompt caching 的部分。以及,一局 376 万 token 的账单,值得写在最前面给想复现的人看。

## 8. 结语

把这一局叫"学会"可能还太重。它更像是记忆系统第一次完整跑通了受伤、记账、查账、照办这个循环:上一局摔在 Ante 11 的地方,这一局踩着同一条路径走了过去,而且每一步都有据可查。泛化、稳定性、注入成本,下一轮的问题都在。但至少现在,"经验可以被用上"不再是假设,是一条可以回放的录像。

## 附:数据与可复现性

- 对局:`arena:gemini37-flash-high-cur3:train-r3:BENCH05:1:1787353073407`(runs 表 id=182)
- 523 步全量落 `moves` 表,每步含 pre-move 状态 JSON、tool 与 args、模型 reasoning 原文、tokensIn/Out
- 13 次召回的注入全文在 `memory/gemini37-flash-high-cur3/recalls.jsonl`
- 局末反思在 `memory/gemini37-flash-high-cur3/memories.jsonl`
- 本文引用的英文原文(注入 block、模型 reasoning、局末反思)一字未改
