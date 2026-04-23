# Installation

## Prerequisites

- Choose a single chat platform for the deployment: `slack` or `matrix`.
- For Slack, [create a Slack app](https://api.slack.com/apps?new_app=1). Select `from an app manifest` and copy `manifest.yaml` out of this repository and paste it in to automatically configure the app and its required settings. Be sure to override any customizable settings like name, etc.
- For Slack, provide the app token, bot token, and user token as the `SLACK_APP_TOKEN`, `SLACK_BOT_TOKEN`, and `SLACK_USER_TOKEN` environment variables. These can be found within the app's configuration page in Slack. For more information on Slack tokens, see the documentation [here](https://api.slack.com/authentication/token-types).
- For Matrix, provision a bot user, obtain its access token, and identify the digest room ID the bot should use for incident notifications. If you want in-room incident creation from Element, plan a public base URL for the widget endpoint as well.
- You'll need a Postgres instance to connect to. If trying the bot out using Docker Compose or Helm, there are options to run a database alongside the app.
- Configure and deploy the application using one of the methods described below, or however you choose. (There's a Docker image available.)

### Database Migrations

The application does not handle database migrations automatically. This means that database migrations should be run using a bootstrap or init process.

If you use the official Helm chart, two init containers are created - one to wait for the database to become available, and another to run the migrations.

This feature is enabled by default:

```yaml
# values.yaml
init:
  enabled: true
  command: ['/bin/sh']
  args: ['-c', 'alembic upgrade head']
  image:
    tag:
```

We provide an image called `eb129/incidentbot:util` that is used for this step. You can provide your own image using the `image` and/or `tag` options shown above.

!!! warning

    If you choose to use your own image, be sure it conforms to the requirements of the application.

!!! note

    If you do not use the Helm chart and install using other methods, take note of how this is done using Docker Compose:

    ```yaml
    migrations:
      build:
        context: .
        dockerfile: Dockerfile.util
      depends_on:
        db:
          condition: service_healthy
      command: ['sh', '-c', 'alembic upgrade head']
      environment:
        IS_MIGRATION: true
        POSTGRES_HOST: db
        POSTGRES_DB: incident_bot
        POSTGRES_USER: incident_bot
        POSTGRES_PASSWORD: somepassword
        POSTGRES_PORT: 5432
      volumes:
        # Wherever the config file lives, root by default
        - ./config.yaml:/app/config.yaml
      networks:
        - inc_bot_network
    ```

    In the end, you simply need a process that runs `alembic upgrade head` before the application starts.

!!! note

    If using the Helm chart and setting `envFromSecret`, those variables will be passed to the init containers.

## Required Variables

These variables are **required** for all installation methods:

- `PLATFORM` - the chat platform for this deployment. Valid values are `slack` and `matrix`. If omitted, it defaults to `slack`.
- `POSTGRES_HOST` - the hostname of the database.
- `POSTGRES_DB` - database name to use.
- `POSTGRES_USER` - database user to use.
- `POSTGRES_PASSWORD` - password for the user.
- `POSTGRES_PORT` - the port to use when connecting to the database.

Set `SECRET_KEY` as well. It is used for JWT signing and widget token signing. If you leave it unset, Incident Bot will generate a random value at each startup, which will invalidate existing Matrix widget tokens after a restart.

### Slack-Only Variables

- `SLACK_APP_TOKEN` - the app-level token for enabling websocket communication. Found under your Slack app's `Basic Information` tab in the `App-Level Tokens` section.
- `SLACK_BOT_TOKEN` - the API token to be used by your bot once it is deployed to your workspace for `bot`-scoped pemissions. Found under your Slack app's `OAuth & Permissions` tab.
- `SLACK_USER_TOKEN` - the API token to be used by your bot for `user`-scoped permissions. Found under your Slack app's `OAuth & Permissions` tab.

These variables are required when `platform: slack`.

### Matrix-Only Variables

- `MATRIX_HOMESERVER` - the base URL of the homeserver the bot account uses, for example `https://matrix.example.com`.
- `MATRIX_USER_ID` - the full Matrix user ID for the bot, for example `@incidentbot:example.com`.
- `MATRIX_ACCESS_TOKEN` - the access token for the bot account.
- `MATRIX_DEVICE_ID` - optional device ID for the Matrix client. Defaults to `INCIDENTBOT`.
- `MATRIX_DIGEST_ROOM_ID` - the room ID where digest updates are posted and where the widget is registered.
- `MATRIX_WIDGET_BASE_URL` - optional public base URL for the Incident Bot deployment serving `/widget/incident`.

These variables are required when `platform: matrix`, except for `MATRIX_DEVICE_ID` and `MATRIX_WIDGET_BASE_URL`.

### Platform Configuration

Select the platform in either `.env` or `config.yaml`:

```bash
PLATFORM=slack
```

or:

```yaml
platform: slack
```

or:

```bash
PLATFORM=matrix
```

or:

```yaml
platform: matrix
```

Matrix connection settings can be provided via environment variables:

```bash
MATRIX_HOMESERVER=https://matrix.example.com
MATRIX_USER_ID=@incidentbot:example.com
MATRIX_ACCESS_TOKEN=syt_...
MATRIX_DEVICE_ID=INCIDENTBOT
MATRIX_DIGEST_ROOM_ID=!abcdef:example.com
MATRIX_WIDGET_BASE_URL=https://incidentbot.example.com
```

You can also keep them in `config.yaml`:

```yaml
matrix:
  homeserver: https://matrix.example.com
  user_id: "@incidentbot:example.com"
  access_token: "syt_..."
  device_id: INCIDENTBOT
  digest_room_id: "!abcdef:example.com"
  widget_base_url: https://incidentbot.example.com
```

For Matrix deployments:

- Environment variables override the `matrix` block when both are present.
- `MATRIX_DIGEST_ROOM_ID` or `matrix.digest_room_id` is required and is the room where digest updates are posted.
- `MATRIX_WIDGET_BASE_URL` or `matrix.widget_base_url` should point to the Incident Bot deployment that serves `/widget/incident`. Without it, the bot can still run and respond to `!incident` commands, but users will not get the embedded create-incident widget in Element.
- The widget routes are served outside `/api/v1`, so Matrix incident creation does not require `api.enabled: true`.

## Architecture Support

Images are built for both `amd64` and `arm64`.

To adjust which one is used with Helm:

```yaml
# values.yaml
image:
  suffix: arm64
```

## Kubernetes

### Helm

You can get started quickly by using the Helm chart:

```bash
helm repo add incidentbot https://docs.incidentbot.io/charts
helm repo update
```

Sensitive data should come from Kubernetes `Secret` objects. 

!!! warning

    Secrets management is outside of the scope of this application. Choose the solution that works best for you. Any solution that renders a Kubernetes `Secret` that contains the key/value data for your sensitive application information will work.

One method is to used something like [sealed-secrets](https://github.com/bitnami-labs/sealed-secrets).

If using `sealed-secrets`, you could put your sensitive environment variables in a `.env` file and create the `Secret` using the following command:

```bash
kubectl create secret generic incidentbot-secret --from-env-file=.env --dry-run='client' -ojson --namespace incidentbot >incidentbot-secret.json &&
  kubeseal --controller-name sealed-secrets <incidentbot-secret.json >incidentbot-secret-sealed.json &&
  kubectl create -f incidentbot-secret-sealed.json
```

Contained with `.env`, you'd want to include the sensitive values for this application. For example:

```bash
PLATFORM=slack
POSTGRES_HOST=db
POSTGRES_DB=incident_bot
POSTGRES_USER=incident_bot
POSTGRES_PASSWORD=somepassword
POSTGRES_PORT=5432
SECRET_KEY=replace-with-a-stable-secret
# Slack only when platform: slack
# SLACK_APP_TOKEN=xapp-1-...
# SLACK_BOT_TOKEN=...
# SLACK_USER_TOKEN=xoxp-...
# Matrix only when platform: matrix
# MATRIX_HOMESERVER=https://matrix.example.com
# MATRIX_USER_ID=@incidentbot:example.com
# MATRIX_ACCESS_TOKEN=syt_...
# MATRIX_DEVICE_ID=INCIDENTBOT
# MATRIX_DIGEST_ROOM_ID=!abcdef:example.com
# MATRIX_WIDGET_BASE_URL=https://incidentbot.example.com
# any integration secrets
# and so on...
```

This will create the required `Secret` in the `Namespace` `incidentbot`. You may need to create the `Namespace` if it doesn't exist.

Create a `values.yaml` file. We'll call this one `incidentbot-values.yaml`:

```yaml
envFromSecret:
  enabled: true
  secretName: incidentbot-secret
ingress:
  enabled: true
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: letsencrypt-prod
  hosts:
    - host: incidentbot.mydomain.com
      paths:
        - path: /
          pathType: ImplementationSpecific
  tls:
    - secretName: incidentbot-tls
      hosts:
        - incidentbot.mydomain.com
```

Run the following command to install the resources using the chart:

```bash
VERSION=$(helm search repo incidentbot --output=json | jq '.[0].version' | tr -d '"')
helm install incidentbot/incidentbot --version $VERSION --values incidentbot-values.yaml --namespace incidentbot
```

To clean everything up:

```bash
helm uninstall incidentbot --namespace incidentbot
```

#### Using the built-in database

There is an option for testing or demo environments to deploy a database alongside the application:

```yaml
# values.yaml
database:
  enabled: true
  password: somepassword
```

!!! warning

    This is not recommended for production use. You should setup a database independently and provide its credentials to the application instead.

#### Configuration

The application's `config.yaml` settings can be set using the `configMap` option:

```yaml
# values.yaml
configMap:
  create: true
  data:
    options:
      skip_logs_for_user_agent:
        - kube-probe
        - ELB-HealthChecker/2.0
      timezone: America/New_York
    maintenance_windows:
      components:
        - Website
        - API
        - Auth
        - Databases
```

Any data under the `data` key will be added to the `ConfigMap` and will be made available to the application.

You are not required to provide this option if you wish to use all of the default settings.

Consult the [configuration](configuration.md) page for details on all configurable options.

## Docker Compose

A sample compose file is provided with sample variables. This is useful for running the application locally. In this scenario, the database runs as a container. This is not recommended for production usage.

!!! warning

    Management of a database is outside of the scope of this application. Setup for a containerized database is provided for convenience when using Docker Compose.
