---
name: verification-before-completion
description: Use when about to claim work is complete, fixed, or passing, before committing or creating PRs - requires running verification commands and confirming output before making any success claims; evidence before assertions always
---

# Verification Before Completion（完成前验证）

## 概述

声称工作已完成而没有验证，是**不诚实**而非高效。

**核心原则：** 永远先有证据再下结论。

**违反这条规则的字面意思，就是违反它的精神。**

## 铁律

```
没有新鲜的验证证据，就不许宣称完成
```

如果你没有在**本条消息中**运行过验证命令，你就不许声称它通过。

## 门控函数

```
在宣称任何状态或表达满意之前：

1. 识别（IDENTIFY）：什么命令能证明这一断言？
2. 运行（RUN）：执行完整命令（全新的、完整的）
3. 读取（READ）：完整输出，检查退出码，统计失败数
4. 验证（VERIFY）：输出是否证实了断言？
   - 否：以实际状态+证据陈述
   - 是：以"断言+证据"陈述
5. 然后（ONLY THEN）：才能下断言

跳过任何一步 = 在说谎，不是在验证
```

## 常见失败

| 断言 | 需要 | 不充分 |
|------|------|--------|
| 测试通过 | 测试命令输出：0 失败 | 之前的运行、"应该过" |
| Linter 干净 | Linter 输出：0 错误 | 部分检查、外推 |
| 构建成功 | 构建命令：exit 0 | Linter 通过、日志看起来不错 |
| Bug 已修复 | 测试原始症状：通过 | 代码改了、假设修好了 |
| 回归测试有效 | 验证 red-green 循环 | 测试通过一次 |
| 代理已完成 | VCS diff 显示改动 | 代理报告"成功" |
| 需求已满足 | 逐条 checklist | 测试通过 |

## 红旗 —— 停下

- 使用 "should"、"probably"、"seems to"
- 在验证前表达满意（"Great!"、"Perfect!"、"Done!" 等）
- 准备 commit/push/PR 但没验证
- 相信代理的成功报告
- 依赖部分验证
- 想"就这一次"
- 累了想让工作结束
- **任何暗含成功但没运行验证的措辞**

## 防止自欺

| 借口 | 真相 |
|------|------|
| "现在应该工作了" | **去运行**验证 |
| "我有信心" | 信心 ≠ 证据 |
| "就这一次" | 没有例外 |
| "Linter 过了" | Linter ≠ 编译器 |
| "代理说成功" | 独立验证 |
| "我累了" | 疲惫 ≠ 借口 |
| "部分检查够了" | 部分证明不了任何事 |
| "换个说法规则就不适用了" | 精神大于字面 |

## 关键模式

**测试：**
```
✅ [运行测试命令] [看到：34/34 通过] "All tests pass"
❌ "Should pass now" / "Looks correct"
```

**回归测试（TDD Red-Green）：**
```
✅ 写 → 运行（通过）→ 撤销修复 → 运行（必须失败）→ 恢复 → 运行（通过）
❌ "I've written a regression test"（没有 red-green 验证）
```

**构建：**
```
✅ [运行 build] [看到：exit 0] "Build passes"
❌ "Linter passed"（linter 不检查编译）
```

**需求：**
```
✅ 重新读计划 → 创建 checklist → 逐条验证 → 报告差距或完成
❌ "Tests pass, phase complete"
```

**代理委派：**
```
✅ 代理报告成功 → 检查 VCS diff → 验证改动 → 报告实际状态
❌ 相信代理报告
```

## 为什么这很重要

来自 24 个失败记忆：
- 你的协作伙伴说"I don't believe you" —— 信任被打破
- 未定义函数被发出去 —— 本来会崩溃
- 需求缺失被发出去 —— 功能不完整
- 浪费时间在虚假完成上 → 改向 → 重做
- 违反："诚实是核心价值观。如果你撒谎，你会被替换。"

## 何时应用

**在以下场景之前始终应用：**
- 任何形式的成功/完成断言
- 任何满意表达
- 任何关于工作状态的积极陈述
- 提交、创建 PR、任务完成
- 移到下一个任务
- 委派给代理

**规则适用于：**
- 完全一样的措辞
- 改写和同义词
- 暗含成功
- **任何暗示完成/正确的沟通**

## 底线

**验证没有捷径。**

运行命令。读输出。然后**再**声明结果。

这是不可谈判的。
