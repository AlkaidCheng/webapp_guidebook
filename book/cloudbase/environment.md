---
title: Environments and API Keys
summary: "Creating an environment, choosing and renewing its package, enabling Cloud Run, and issuing the server API key the application uses."
references:
  createenv: "Tencent CloudBase, *创建云开发环境* (creating an environment), [docs.cloudbase.net/quick-start/create-env](https://docs.cloudbase.net/quick-start/create-env)."
  envoverview: "Tencent CloudBase, *云开发环境介绍* (environment overview), [docs.cloudbase.net/quick-start/env-overview](https://docs.cloudbase.net/quick-start/env-overview)."
  pricing: "Tencent Cloud, *云开发 CloudBase 计费说明* (billing), [cloud.tencent.com/document/product/876/127357](https://cloud.tencent.com/document/product/876/127357)."
  apikey: "Tencent CloudBase, *获取 ApiKey* (obtaining an API key), [docs.cloudbase.net/http-api/basic/apikey](https://docs.cloudbase.net/http-api/basic/apikey)."
  apikeycli: "Tencent CloudBase, *tcb env apikey create*, [docs.cloudbase.net/cli-v1/envs/apikey-create](https://docs.cloudbase.net/cli-v1/envs/apikey-create)."
  rpc: "Tencent CloudBase, *调用 RPC* (calling database functions over HTTP), [docs.cloudbase.net/database/postgresql/http/rpc](https://docs.cloudbase.net/database/postgresql/http/rpc)."
  gitdeploy: "Tencent CloudBase, *通过 Git 仓库部署* (deploying from a Git repository), [docs.cloudbase.net/run/deploy/deploy/deploying-git](https://docs.cloudbase.net/run/deploy/deploy/deploying-git)."
---

## Creating an environment

An environment is created once per stage of the release process (Chapter 8)
and lives as long as its package is paid. The choices made at creation, the
database mode and the package, are the ones that matter later: the mode can
never change, and the package decides what the deployment costs
(Section 9.3).

!!! recipe "Creating an environment"
    1. Sign in to the console at `https://tcb.cloud.tencent.com/dev`. A new
       account first accepts the service terms and authorises CloudBase's
       service role; it then receives a personal-package trial environment
       [@createenv]. An existing account opens the environment list and
       chooses 新建环境 (new environment), which leads to an order page.
    2. Choose the **PostgreSQL** database. Only an environment created this
       way runs in PG mode, and an existing environment cannot be converted
       [@envoverview].
    3. Choose the package from the expected load, not from the price of the
       first month: an API with one warm 1-core, 2 GiB instance already needs
       the standard package (Example 9.1).
    4. Give the environment a short name that says what it is for, such as
       `myapp-staging`. CloudBase appends a random suffix to form the
       environment ID.
    5. Pay, wait a few minutes for the resources to initialise, and record the
       environment ID. Every later chapter needs it.

    **Check:** the environment appears in the environment list with its
    package and expiry date, and its database opens in the SQL editor
    (`#/db/postgres/sql-editor`).

An environment cannot be transferred to another Tencent Cloud account
[@createenv]. Create it under the account that will own the deployment, and
keep application data portable (Chapter 12) for the day a move is needed.

## Renewal, expiry and switching off

A package has a term and an expiry date, shown with its usage under
环境设置 → 套餐用量 (environment settings → package usage). Automatic renewal
is the default for paid packages and is the right setting for any environment
that serves users.

When a package expires without renewal, the environment is isolated: its
services stop answering and its data cannot be reached. Renewing during the
isolation period restores it unchanged; at the end of the period the
environment is destroyed together with its database, its bucket, its services
and its keys [@pricing]. The isolation period is therefore the last chance to
copy anything out, and also a free rollback window after a move: an
environment that is no longer needed can be left to lapse rather than deleted.

!!! recipe "Turning off automatic renewal"
    1. Open 环境设置 → 套餐用量 for the environment.
    2. On the package card, use the 关闭 (turn off) link beside automatic
       renewal. In November 2026 it acted immediately, without a confirmation
       dialog.

    **Check:** the card reports automatic renewal as off and still shows the
    expiry date. Nothing else changes until that date.

Cloud Run has no pause button. A service whose minimum instance count is 0
and that currently runs no instance is shown as 已暂停 (paused), and that is
the closest thing to switching a service off while keeping it (Chapter 13).
An environment that should cost nothing until it lapses has every service at
a minimum of 0 and no traffic.

## Enabling Cloud Run and authorising Git

Cloud Run is enabled per environment, from the console's 云函数 / 托管
(functions and hosting) section, under 云托管. Enabling it accepts the service
terms and its pay-as-you-go pricing for that environment; it is an account
decision rather than an application setting, and it is taken once.

A service built from a private Git repository needs the platform to read that
repository, which it does through an authorisation granted to a Git hosting
account [@gitdeploy]. In November 2026 that authorisation was held per
environment: a second environment showed GitHub as 未授权 (not authorised)
although the account had authorised it for the first. The authorisation covers
every repository the authorising Git account can reach, so it is given from a
dedicated deployment account that owns nothing but the deployment repository
(Chapter 8, Chapter 13).

!!! recipe "Authorising GitHub for an environment"
    1. In a browser where GitHub is signed in as the **deployment account**,
       not as a personal account, open the environment's 云托管 page and start
       creating a service.
    2. Under the repository field, choose 点击授权 (authorise) and approve the
       GitHub application.

    **Check:** the repository field shows `GitHub (已授权)` and lists the
    deployment repository and its branches.

## The server API key

The application's API reaches the environment's database through the HTTP
gateway (Chapter 12), and the gateway admits a server process by its **API
key**. A CloudBase API key carries administrator-level rights over the
environment and is sent as a bearer token [@apikey]. It is the most powerful
credential the application holds, so it lives only in the API service's
settings and in the owner's password manager.

!!! definition "Server API key"
    A long-lived token issued per environment that identifies a trusted
    server process to the CloudBase gateway with administrator rights. It is
    presented as `Authorization: Bearer <key>`. A key belongs to one
    environment and is useless against another.

!!! recipe "Creating the server API key"
    1. Open the environment's 环境配置 (environment configuration) → API Key
       page. Under 服务端 API Key (server API key), choose 创建 API Key
       [@apikey].
    2. Name the key after its purpose and environment, for example
       `myapp-run`, so that a later rotation knows what it replaces.
    3. Choose the expiry. A key that never expires (永不过期) cannot stop the
       service on a deadline; a dated key limits how long a leaked copy works
       but must be rotated before its date, or the API refuses to start.
       Staging can use a non-expiring key; production should use a dated one
       with the rotation in the calendar.
    4. Copy the key into the password manager at once. The plaintext is
       returned only at creation [@apikeycli].

    The CLI does the same:

    ```bash
    tcb env apikey create myapp-run -e <env-id>                 # never expires
    tcb env apikey create myapp-run -e <env-id> --expire 7776000  # 90 days
    ```

    `--expire` takes seconds, at least 7,200; omitting it or passing 0 gives
    a key that never expires [@apikeycli].

    **Check:** the key appears in the list with the expected name and expiry.

The API service does not need the key's text. Cloud Run's service form has an
**API Key** switch that picks a key of the environment by name and injects it
into the container, where it appears as the environment variable
`CLOUDBASE_APIKEY` (Chapter 13). The text is needed only by tools that run
outside the platform, such as a local test harness, and those read it from a
local, ignored configuration file.

!!! recipe "Checking a key against the gateway"
    With the key in the shell's environment (never on the command line, where
    it would enter the shell history), call any function the schema defines:

    ```bash
    curl -s -o /dev/null -w '%{http_code}\n' -X POST \
      -H "Authorization: Bearer $CLOUDBASE_APIKEY" \
      -H 'Content-Type: application/json' -d '{}' \
      "https://<env-id>.api.tcloudbasegateway.com/v1/rdb/rest/rpc/<function>"
    ```

    The address follows the gateway's RPC convention [@rpc]. A status of 200
    means the key is valid for this environment and the function exists; a
    refused key never reaches the function.

A key is rotated the way any credential is (Chapter 3): create the new key,
switch the API service to it and deploy, confirm the service starts and
answers, and only then delete the old key from the list. Deleting a key that a
running service still uses stops that service's database access at once.

!!! summary "Key points"
    - Create environments in PG mode (it cannot be changed later) and choose
      the package from the expected Cloud Run load.
    - An expired package isolates the environment; at the end of the
      isolation period everything in it is destroyed. Keep automatic renewal
      on for anything that serves users.
    - Cloud Run is enabled per environment, and Git authorisation is granted
      per environment, from a dedicated deployment account.
    - The server API key has administrator rights over one environment. It is
      shown once, stored in a password manager, and injected into the API
      service by name through the API Key switch.
    - Rotate keys by creating, deploying, verifying and only then deleting.
