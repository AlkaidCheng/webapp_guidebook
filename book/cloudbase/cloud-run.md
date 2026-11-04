---
title: Cloud Run Services
summary: "Deploying an API and a web service from a private Git repository: settings, variables, scaling, versions, logs and the Webshell."
references:
  gitdeploy: "Tencent CloudBase, *通过 Git 仓库部署* (deploying from a Git repository), [docs.cloudbase.net/run/deploy/deploy/deploying-git](https://docs.cloudbase.net/run/deploy/deploy/deploying-git)."
  imagedeploy: "Tencent CloudBase, *部署容器镜像* (deploying a container image), [docs.cloudbase.net/run/deploy/deploy/deploying-image](https://docs.cloudbase.net/run/deploy/deploy/deploying-image)."
  dockerfile: "Tencent CloudBase, *从 Dockerfile 构建* (building from a Dockerfile), [docs.cloudbase.net/run/develop/builds/dockerfile](https://docs.cloudbase.net/run/develop/builds/dockerfile)."
  envs: "Tencent CloudBase, *环境变量* (environment variables), [docs.cloudbase.net/run/deploy/configuring/environment/envs](https://docs.cloudbase.net/run/deploy/configuring/environment/envs)."
  autoscaling: "Tencent CloudBase, *运行模式与扩缩容* (run modes and autoscaling), [docs.cloudbase.net/run/deploy/configuring/autoscaling/about-instance-autoscaling](https://docs.cloudbase.net/run/deploy/configuring/autoscaling/about-instance-autoscaling)."
  internal: "Tencent CloudBase, *内网互联* (private networking), [docs.cloudbase.net/run/deploy/networking/internal-link](https://docs.cloudbase.net/run/deploy/networking/internal-link)."
  domains: "Tencent CloudBase, *配置自定义域名* (custom domains), [docs.cloudbase.net/run/deploy/networking/custom-domains](https://docs.cloudbase.net/run/deploy/networking/custom-domains)."
  revisions: "Tencent CloudBase, *查看或删除版本* (viewing and deleting versions), [docs.cloudbase.net/run/deploy/managing/revisions](https://docs.cloudbase.net/run/deploy/managing/revisions)."
---

## Services, versions and addresses

Cloud Run (云托管) runs containers built from the application's Dockerfiles
(Chapter 5). The unit of configuration is the **service**: a name, a source,
a port, environment variables and scaling settings. Every deployment of a
service creates a numbered **version** (版本, `001`, `002`, …) with its own
image and its own copy of the settings; traffic is routed to versions by
percentage, and an older version can take the traffic back. A service's name
is fixed at creation and cannot be changed afterwards.

An application in the shape of Chapter 1 runs as two services. The **web
service** is the only public one: it serves the pages and forwards `/api`
to the **API service**, which has only an internal address.

| Address | Where it is shown | Reaches the service | Notes |
|---|---|---|---|
| Default public domain | Service overview | From the internet over HTTPS | `<service>-<n>-<n>-<appid>.sh.run.tcloudbase.com`; for development and testing; no authentication of its own [@domains] |
| Custom domain | HTTP 网关 → 自定义域名 | From the internet over HTTPS | Needs a certificate, and ICP filing in mainland China (Chapter 2) |
| Internal default address (内网默认地址) | 服务配置 tab | From other services of the environment | `http://` with the service port; HTTPS and port 80 do not answer |

Table: The addresses of a Cloud Run service.

!!! warning "The internal address does not wake a service"
    A service whose minimum instance count is 0 is started again by a request
    to its public domain, but not by a request to its internal address. A web
    service that calls its API internally therefore needs the API at a
    minimum of one instance; at zero, every `/api` call fails until something
    else wakes the API. This behaviour was observed in 2026 and is not
    described in the documentation.

The documentation's page on private networking describes a separate
mechanism, attaching services to a VPC in the Shanghai region, which
the setup of this chapter does not need [@internal].

**Scaling.** The console offers several run modes (运行模式) [@autoscaling]:
始终自动扩缩容 (always autoscale, between a minimum and a maximum),
持续运行 (a fixed number of instances), 白天持续运行，夜间自动扩缩容
(fixed by day, autoscaling at night) and 自定义 (custom). Under autoscaling,
an idle instance is reclaimed and the service can reach zero; the console's
form states 30 idle minutes, the documentation 10. Scaling from zero is a cold
start that delays the first request by the time the container needs to
start.

| Service | Run mode | Minimum | Maximum | Size |
|---|---|---|---|---|
| API | 始终自动扩缩容 | 1 | 1 or more | 1 core, 2 GiB |
| Web | 始终自动扩缩容 | 0 (cold start accepted) or 1 | 1 or more | 1 core, 2 GiB |

Table: Starting scaling settings for the two services. The API's minimum of
one follows from the warning above; its cost is in Section 9.3.

## The deployment repository

Cloud Run builds a private repository through an authorisation that covers
every repository the authorising Git account can reach [@gitdeploy]. The
platform therefore builds from a **deployment repository**: a private copy of
the application owned by a dedicated Git account that owns nothing else
(Chapter 8). Releases are fast-forward pushes of a tested `main` to that
repository, and Cloud Run builds the branch it is told to.

!!! recipe "Setting up the deployment repository"
    1. Create a dedicated GitHub account with two-factor authentication. As
       that account, create an **empty** private repository named like the
       application. A fork of a public repository cannot be made private.
    2. Create an SSH key pair for pushing and register its public half on the
       repository as a deploy key with write access.
    3. In the development clone, add the repository as a remote that pushes
       only `main` and uses only that key:

        ```bash
        git remote add deploy git@github.com:<deployment-account>/<app>.git
        git config remote.deploy.push refs/heads/main:refs/heads/main
        git config core.sshCommand "ssh -i <path-to-deploy-key> -o IdentitiesOnly=yes"
        git push deploy
        ```

        The last line binds the key for every remote of the clone; it is
        harmless when the development remote uses HTTPS. Otherwise use an
        SSH host alias with the same options.

    4. On the deployment repository, disable Actions. Make its default
       branch a one-commit orphan branch holding a README, so that files such
       as `.github/dependabot.yml` on `main` do not trigger automation there;
       Cloud Run builds `main` explicitly.
    5. Authorise Cloud Run with that account, once per environment
       (Chapter 10).

    **Check:** `git push deploy` fast-forwards the deployment repository's
    `main`, and the service form lists the repository under `GitHub (已授权)`.

## Creating the services

The service form is the same for creation and for 更新服务 (update service),
which prefills it from the live version. In November 2026 its sections were
the source, 服务端口设置 (ports), 构建设置 (build settings), 环境变量设置
(variables) and 更多配置 (more settings). The values below are those of a
Node.js API on port 4000 and a web service on port 3000; any language works if
its container listens on `0.0.0.0` and the declared port [@dockerfile].

!!! recipe "Creating the API service"
    1. Open 云托管 → 服务管理 (service management) → create a service, and
       choose deployment from a Git repository (Git 仓库). Select
       `GitHub (已授权)`, the deployment repository and the branch `main`.
       Leave 启用自动部署 (automatic deployment) off until the release
       procedure is trusted.
    2. **服务名称** (service name): `myapp-api`. It cannot be changed later.
    3. **端口映射** (port mapping): access port 80, **service port** (服务端口)
       4000, the port the process listens on.
    4. **构建设置**: target directory (目标目录) `.`, the repository root,
       as the build context; Dockerfile `apps/api/Dockerfile`, the path
       relative to that directory.
    5. **环境变量设置**: enter the variables (Section 13.4), and turn on the
       **API Key 设置** switch, selecting the server API key by name
       (Chapter 10). It is injected as `CLOUDBASE_APIKEY`.
    6. **更多配置**: 容器规格 (size) 1 core / 2 GiB; 公网访问 (outbound
       internet) **on**, because the API calls the gateway, COS and SES over
       the internet; 运行模式 始终自动扩缩容 with 实例副本数 (instance count)
       minimum 1 and maximum 1.
    7. 部署 (deploy). In the dialog, choose 发布版本并自动切换流量至新版本
       (release and switch traffic to the new version).
    8. After the first version runs, open the service's 服务配置 tab: turn the
       public domain off, and copy the **内网默认地址** (internal default
       address) with its port for the web service.

    **Check:** the version list shows the version as 正常 (normal) with 100%
    of traffic. Its 部署日志及详情 (deployment log and details) ends with the
    container's startup output.

!!! remark "公网访问 is outbound"
    In the form's 更多配置, 公网访问 controls whether the service's
    instances can reach the internet through the platform's default exit.
    It does not publish the service. An API that turns it off cannot reach
    the CloudBase gateway at its public address, nor COS or SES.

!!! recipe "Creating the web service"
    1. As for the API, with service name `myapp-web`, service port 3000 and
       Dockerfile `apps/web/Dockerfile`.
    2. Variables: the API's internal address with scheme and port and no path,
       for example `API_INTERNAL_URL=http://<internal-host>:4000`, plus
       whatever the web build needs. No API key: the web service never calls
       the gateway.
    3. Keep the public domain on. Minimum instances 0 or 1 (Table 13.2).

    **Check:** through the web service's public domain, the sign-in page
    answers 200, `/api/health` answers 200, and a request for the current
    session without a cookie answers 401. Together they show that the web
    service runs, reaches the API internally, and that the API reaches its
    database.

```bash
WEB=https://<web-service-domain>
for path in /sign-in /api/health /api/auth/session; do
  printf '%s ' "$path"; curl -s -o /dev/null -w '%{http_code}\n' "$WEB$path"
done
```

The first visit to a default domain may show a CloudBase 页面访问提示
(page access notice) interstitial, which 确定访问 dismisses. Custom domains
are bound in the environment's HTTP 网关 (HTTP gateway): add the domain
with its certificate under 自定义域名, then create a route under 路由管理 that
points the domain's path `/` at the web service [@domains].

## Environment variables

Variables belong to a version: each deployment carries its own copy, and a
service-level value overrides an `ENV` default baked into the image
[@envs]. The form accepts them in three modes: 可视化输入 (one row per
variable), JSON 输入 (a JSON object) and 配置文件输入 (`.env`-style text).
Secrets such as CAM keys are entered here and nowhere else; anyone with
console access to the environment can read them, so console access is itself
a secret to guard.

!!! warning "Pasting variables keeps their quotes"
    Text pasted into 配置文件输入 is taken literally. A line such as
    `TEMPLATES="{\"welcome\":101}"`, copied from a `.env` file, arrives as a
    string that begins with a quote mark, not as a JSON object, and an
    application that parses it fails at startup; a quoted `EMAIL_FROM` keeps
    its quote marks in every message. After any bulk paste, switch to
    可视化输入 and read each value that should be JSON, a URL or an address.

!!! warning "Switching modes resets the API key"
    Changing the input mode after the API Key switch has been set turned the
    switch off again. Set the variables first, then the switch, and confirm
    the switch before deploying.

Changing a variable means deploying a new version. When only settings change,
there is no need to rebuild: in 更新服务, the deployment method 使用容器镜像部署
(deploy a container image) prefills the live image and its settings, and
deploys in about a minute [@imagedeploy]. After any deployment, compare the
variable list with the previous version's; a failed Git-mode deployment was
seen to drop variables that had just been added.

## Operating versions

The service's 部署版本 (versions) tab lists every version with its status,
traffic share and instance count, and offers 实例详情 (instance details),
部署日志及详情, 回退 (roll back) and 删除 (delete) [@revisions].

**Scaling in place.** Instance counts can be changed without a new version:
服务配置 → 编辑 (edit) → 实例副本数 → 保存 (save). This is how a service is
"paused" (minimum 0, shown as 已暂停 once its instances are gone). Raising the
minimum again does not always resume it: on an internal-only service with no
instance the new minimum was saved, but no instance had started eleven minutes
later. Plan on a new version to bring such a service back.

**Logs.** A version's 部署日志及详情 shows the build (checkout, image build
and push) and the rollout, and for a version that failed to start it ends
with the container's own startup output. That is usually enough to diagnose a
failed deployment without a paid log service.

**The Webshell.** 实例详情 → Webshell opens a shell in a running instance,
with the instance's environment variables. It is the place for checks that
need the service's own credentials, such as the storage probe of Chapter 11
or a one-off administrative script bundled in the image, because the secrets
never leave the container.

!!! recipe "Rolling back and cleaning up"
    1. To roll back, choose 回退 on the version to return to, or deploy it
       again with 使用容器镜像部署 and its image.
    2. A replaced version drains its instances on its own, usually within an
       hour or two; its instance count cannot be edited per version.
    3. Delete a failed version, or one too old to be a useful rollback, with
       删除. The confirmation dialog does not name the version, so open it
       from the right row; deletion cannot be undone.

    **Check:** the list holds the live version and one known-good previous
    version.

!!! remark "Build not found"
    Git-mode deployments have failed at the first second with
    `check_build_image : fail, [ErrorCode]:300502, build not found`, although
    the build itself went on to complete and push its image: the deployment
    checked for the build before the build was registered. The image the
    failed attempt pushed is named in its log. Deploying that image through
    使用容器镜像部署, with the variables re-entered, skips the build check. A
    later Git-mode deployment usually succeeds again.

!!! summary "Key points"
    - A service has a fixed name, versions with their own image and settings,
      and three kinds of address: a public default domain for testing, a
      custom domain through the HTTP gateway, and an `http://` internal
      address with the service port.
    - The internal address does not wake a service at zero instances, so an
      API reached internally needs a minimum of one.
    - Build from a private deployment repository owned by a dedicated Git
      account; target directory `.`, Dockerfile path relative to it, the
      service port equal to the port the process listens on.
    - Enter variables in 可视化输入 or check them there after a paste; set the
      API Key switch last; settings-only changes deploy the live image without
      a rebuild.
    - Diagnose from the version's deployment log and the Webshell; keep one
      known-good version for rollback and delete the rest.
