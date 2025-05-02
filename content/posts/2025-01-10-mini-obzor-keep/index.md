---
title: "Мини обзор Keep - AI мониторинг"
date: 2025-01-10T13:39:35+03:00
description: "Мониторинг и алертинг - это база. Без них не может нормально ехать ни один продукт и ни один бизнес. Если говорить про IT, то чтобы иметь полную картину у вас должны быть..."
tags: [tools]
---

Мониторинг и алертинг - это база. Без них не может нормально ехать ни один продукт и ни один бизнес. Если говорить про IT, то чтобы иметь полную картину у вас должны быть:

🟢 Infrastructure-related alerts - например, высокая утилизация CPU на ноде
🟢 Application-related alerts - например, прилка начала сыпать 500-ми ошибками
🟢 Business-related alerts - например, снижение конверсии

✍️ Давайте разберем простой кейс, вам прилетел алерт на CPU от ноды с postre в проде. Дежурный инженер открывает борду в grafana ( или zabbix ), смотрит какие то общие показатели. Потом подключается к базе, смотрит какие запросы сейчас обрабатываются, читает логи и тд. И принимает дальнейшие решения по тому в кого эскалировать проблему или решает вопрос сам. И вот этот порядок действий практически всегда один и тот же. 

🔛 Как было бы круто дообогатить алерт уже подготовленной информацией. Например, получить вместе с алертом список выполняющихся сейчас запросов в базе и их потребление по ресурсам. (Например, через SELECT pid AS process_id, query AS active_query FROM pg_stat_activity WHERE state = 'active'; )

🧠Вот ребята из keephq тоже так подумали и запилили просто офигенскую прилу, добавив нереальное количество интеграций (провайдеров), которые можно интегрировать в различные флоу обработки алертов

Просто взгляните на список провайдеров: AKS, AppDynamics, Auth0, Axiom, Azure, BigQuery, Centreon, Chat, Checkmk, Cilium, Clickhouse, Cloud, Cloudwatch, Coralogix, Datadog, Discord, Elastic, GCP, GKE, GitHub, GitLab, Google, Grafana, Graylog, Incident, Jira, Kafka, Kubernetes, Linear, LinearB, Mailchimp, Manager, Mattermost, Microsoft, MongoDB, Monitor, Monitoring, MySQL, Netdata, New, Now, On-Prem, OnCall, OpenAI, OpenObserve, Openshift, OpsGenie, PagerDuty, PagerTree, Pipelines, Planner, PostgreSQL, Pushover, QuickChart, Redmine, Relic, Resend, Rollbar, SIGNL4, SMTP, SSH, SendGrid, Service, SignalFx, Slack, Snowflake, Splunk, Squadcast, Statuscake, SumoLogic, Teams, Telegram, Trello, Twilio, UptimeKuma, Webhook, Zenduty

https://github.com/keephq/keep

Я пока сам не тестровал, но выглядит это просто 🔥 В ближайшее время буду поднимать и тестировать, а результатами поделюсь с вами 👍
