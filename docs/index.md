# Incident Bot Documentation

Incident Bot is an open-source incident management framework.

The core feature is a ChatOps bot to allow your teams to easily and effectively identify and manage technical incidents impacting your cloud infrastructure, your products, or your customers' ability to use your applications and services.

## Core Technologies

 - [Pydantic](https://docs.pydantic.dev/latest/) is used to handle data validation and type safety across the platform.
 - [Pydantic Settings](https://docs.pydantic.dev/latest/concepts/pydantic_settings/) is used to handle application configuration settings.
 - [SQLModel](https://sqlmodel.tiangolo.com/) is used to handle the relationship between application objects and backend databases.
 - [FastAPI](https://fastapi.tiangolo.com/) is used to handle the API.

## Features at a Glance

- Create incident rooms in Slack or Matrix to gather responders and handle incidents.
- Digest room or channel to keep the rest of the organization up to date with incidents at all times.
- Define your own roles, severities, and statuses, or use ones configured right out of the box.
- Keep stakeholders updated using dynamic updates.
- Craft a postmortem document using an integration with Confluence that allows you to use your own templates.
- Create issues in Jira directly from incident channels.
- Page teams in PagerDuty or OpsGenie.
- Manage Statuspage incidents directly from incident channels.
- Create Zoom meetings for each incident to keep communications organized.
- A web interface with advanced features and administrative functionality.

## Integrations

For more information on integrations, check out the [integrations](integrations.md) documentation.

## Quick Start

- Choose a platform for the deployment: `slack` or `matrix`.
- For Slack, [create a Slack app](https://api.slack.com/apps?new_app=1), select `from an app manifest`, and copy `manifest.yaml` from this repository to configure it.
- For Matrix, provision a bot user, obtain an access token, and identify the digest room the bot should use for incident updates and widget registration.
- You'll need a Postgres instance to connect to.
- Create a shared digest location for incident updates: a Slack channel like `#incidents` or a Matrix room such as `!incidents:example.com`.
- Configure the app using `config.yaml` and `.env`, then deploy it to Kubernetes, Docker, or whichever platform you choose. Check out the [installation](installation.md) guide for more details.
