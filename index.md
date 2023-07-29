---
title: Azure OpenAI Exercises
permalink: index.html
layout: home
---

# SQL Server migration exercises

The following exercises are designed to support the modules in the [Migrate SQL Server workloads to Azure SQL](https://learn.microsoft.com/training/paths/migrate-sql-workloads-azure/) learning path on Microsoft Learn.

{% assign labs = site.pages | where_exp:"page", "page.url contains '/Instructions/Labs'" %}
{% for activity in labs  %}
- [{{ activity.lab.title }}]({{ site.github.url }}{{ activity.url }})
{% endfor %}