# 代码质量审查者 Prompt 模板

分派代码质量审查者子代理时使用此模板。

**目的：** 验证实现构建良好（clean、tested、maintainable）

**仅在规范符合性审查通过后分派。**

```
Task tool (general-purpose):
  Use template at requesting-code-review/code-reviewer.md

  DESCRIPTION: [task summary, from implementer's report]
  PLAN_OR_REQUIREMENTS: Task N from [plan-file]
  BASE_SHA: [commit before task]
  HEAD_SHA: [current commit]
```

**除标准代码质量关注点外，审查者还应当检查：**
- 每个文件是否有一个清晰的职责和明确定义的接口？
- 单元是否被分解为可以独立理解和测试的形式？
- 实现是否遵循计划中定义的文件结构？
- 本次实现是否创建了已经很大的新文件，或显著增长了现有文件？（不要标记先前就存在的文件大小——关注本次变更的贡献。）

**代码审查者返回：** Strengths、Issues（Critical/Important/Minor）、Assessment
