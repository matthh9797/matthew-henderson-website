---
title: DBT Basic
weight: 210
menu:
  notes:
    name: DBT Basic
    identifier: notes-dbt-basic
    parent: notes-dbt
    weight: 10
---

<!-- Useful Resources -->
{{< note title="Useful Resources" >}}
- [Syntax overview](https://docs.getdbt.com/reference/node-selection/syntax) - dbt's note selection syntax
- [Optimizing VS Code for dbt on Mac](https://towardsdatascience.com/optimizing-vs-code-for-dbt-on-mac-a56dd27ba8d5) - awesome vs code setup for dbt, article written specifically for Mac but same could be done for Windows 
- [Pro tips for workflows](https://docs.getdbt.com/guides/legacy/best-practices#pro-tips-for-workflows) - advanced dbt workflows
- [Youtube: Create Modular Date Models](https://www.youtube.com/watch?v=5W6VrnHVkCA) - presentation explaining how you can refactor dbt data models to make them more structured and performant
{{< /note >}}

<!-- Workflows -->
{{< note title="Workflows" >}}
DBT saves artifacts when jobs are run which allows for creating powerful workflows. The states of commands are saved as artifacts. For example, the below code will reference the status of the data's freshness and only build models that are *"fresh"*.
```bash 
# Command step order
dbt source freshness
dbt build --select source_status:fresher+
```
{{< /note >}}
