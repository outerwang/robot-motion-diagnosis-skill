# robot-motion-diagnosis

A reusable agent Skill for diagnosing robot motion-planning runs from MCP tool results.

## Install

Copy `SKILL.md` into the Skill directory supported by your agent, keeping the folder name `robot-motion-diagnosis`:

```text
<agent-skills-directory>/robot-motion-diagnosis/SKILL.md
```

The Skill expects the agent to have access to the MCP tools `list_tasks`, `get_task_info`, `get_experiment_report`, `start_motion_plan`, `get_plan_status`, and `validate_motion`. It only interprets their returned evidence; it does not run robot hardware or replace an independent motion validator.

