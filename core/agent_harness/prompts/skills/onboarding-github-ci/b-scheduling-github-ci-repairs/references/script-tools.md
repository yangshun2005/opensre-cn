---
script_tools:
  - name: write_demo_evidence
    script: write_demo_evidence.py
    description: >-
      保存观察到的演示结果，并仅移除其拥有的临时检出。失败的演示可省略不可用的 ID。即使此工具失败，也要单独移除调度器循环。
    input_schema:
      type: object
      properties:
        repo: {type: string}
        pr_number: {type: integer, minimum: 1}
        loop_id: {type: string}
        outcome: {type: string, enum: [success, failed, blocked]}
        failed_run_id: {type: integer, minimum: 1}
        fix_commit: {type: string}
        passing_run_id: {type: integer, minimum: 1}
        blocker: {type: string}
      required: [repo, pr_number, loop_id, outcome]
      additionalProperties: false
---

# 演示辅助工具

此技能激活期间，运行时会注册这些内置脚本。