---
title: API and Web Services
summary: "Running the application as containers: images, ports and health checks, configuration, scaling, versions and diagnostics."
references:
  docker-file: "Docker, *Dockerfile reference*, Docker documentation, <https://docs.docker.com/reference/dockerfile/>."
  docker-best: "Docker, *Building best practices*, Docker documentation, <https://docs.docker.com/build/building/best-practices/>."
  docker-secrets: "Docker, *Build secrets*, Docker documentation, <https://docs.docker.com/build/building/secrets/>."
  run-contract: "Google Cloud, *Container runtime contract*, Cloud Run documentation, <https://docs.cloud.google.com/run/docs/container-contract>."
  twelve-port: "A. Wiggins, *The Twelve-Factor App: VII. Port binding*, <https://12factor.net/port-binding>."
  twelve-disposability: "A. Wiggins, *The Twelve-Factor App: IX. Disposability*, <https://12factor.net/disposability>."
  twelve-logs: "A. Wiggins, *The Twelve-Factor App: XI. Logs*, <https://12factor.net/logs>."
  run-health: "Google Cloud, *Configure container health checks for services*, Cloud Run documentation, <https://docs.cloud.google.com/run/docs/configuring/healthchecks>."
  compose-env: "Docker, *Set, use, and manage variables in a Compose file with interpolation* (`.env` file syntax), Docker documentation, <https://docs.docker.com/compose/how-tos/environment-variables/variable-interpolation/>."
  azure-revisions: "Microsoft, *Update and deploy changes in Azure Container Apps*, Microsoft Learn, <https://learn.microsoft.com/en-us/azure/container-apps/revisions>."
  run-min: "Google Cloud, *Set minimum instances for services*, Cloud Run documentation, <https://docs.cloud.google.com/run/docs/configuring/min-instances>."
  azure-scale: "Microsoft, *Scaling in Azure Container Apps*, Microsoft Learn, <https://learn.microsoft.com/en-us/azure/container-apps/scale-app>."
  wx-cloudrun: "WeChat Open Platform, *微信云托管开发常识* (WeChat Cloud Run essentials), <https://developers.weixin.qq.com/minigame/dev/wxcloudrun/src/guide/debug/know.md>."
  run-rollouts: "Google Cloud, *Rollbacks, gradual rollouts, and traffic migration*, Cloud Run documentation, <https://docs.cloud.google.com/run/docs/rollouts-rollbacks-traffic-migration>."
  docker-exec: "Docker, *docker container exec*, Docker CLI reference, <https://docs.docker.com/reference/cli/docker/container/exec/>."
  azure-console: "Microsoft, *Connect to a container console in Azure Container Apps*, Microsoft Learn, <https://learn.microsoft.com/en-us/azure/container-apps/container-console>."
---

## Images

Both services of [Chapter 1](anatomy.md) run as containers. A container image
packages the application with its runtime and system libraries, so the same
artefact runs on a laptop, in CI and on the platform. The image is built from a
*Dockerfile*, a list of instructions that start from a base image, copy files,
run commands and declare how the container starts [@docker-file].

A good image holds the code and its dependencies and nothing else. A
*multi-stage* build compiles in one stage and copies only the output into a
small final stage, so compilers and development dependencies stay out of the
image; a `.dockerignore` file keeps local files such as `.env` and `.git` out of
the build context; and a `USER` instruction runs the process without root
privileges [@docker-best]. Configuration and secrets are never part of the
image: values set with `ARG` or `ENV` persist in it [@docker-secrets], and an
image that carries one environment's configuration cannot be promoted to
another.

!!! example "A Dockerfile for a Node.js service"
    ```dockerfile
    FROM node:24-slim AS build
    WORKDIR /app
    COPY package.json package-lock.json ./
    RUN npm ci
    COPY . .
    RUN npm run build && npm prune --omit=dev

    FROM node:24-slim
    WORKDIR /app
    ENV NODE_ENV=production
    COPY --from=build /app/package.json ./
    COPY --from=build /app/node_modules ./node_modules
    COPY --from=build /app/dist ./dist
    USER node
    EXPOSE 8080
    CMD ["node", "dist/server.js"]
    ```

    The final stage contains the production dependencies and the compiled
    output only. `EXPOSE` documents the port and does not publish it; the
    platform decides which port receives traffic. Pinning the base image to a
    digest instead of a tag makes rebuilds reproducible, at the price of
    taking base-image security fixes only when the digest is updated
    [@docker-best].

There are two ways to get an image onto a platform (Table 5.1). The platform
can build it from a Git repository, on push or on request; or a CI pipeline
builds it and pushes it to a container registry, from which the platform
deploys a named tag or digest.

| | Built by the platform from Git | Pushed to a registry |
|---|---|---|
| Who builds | The platform's build service | CI or an operator |
| Access the platform needs | Read access to the repository | Pull access to the registry |
| Where build failures appear | The platform's build log | The CI log |
| What identifies a release | A commit | An image digest |
| Typical fit | Small teams; platforms without a registry integration | Several environments promoted from one build |

Table: The two routes from source to a running image.

Building from Git is the simpler route and needs one credential fewer, but it
grants the platform access to a repository, and every environment builds its
own image from the same commit. [Chapter 8](releases.md) shows how a dedicated
deployment repository keeps that access narrow.

## Ports, startup and health

A container platform sends requests to one port of the container and expects
the process to listen on all interfaces. Cloud Run, for example, sends requests
to port 8080 by default, tells the container the port in the `PORT`
environment variable, and requires the process to listen on `0.0.0.0`; a
process bound to `127.0.0.1` accepts connections only from inside its own
container and never receives a request [@run-contract]. Reading the port from
the environment and binding to it is the whole of the twelve-factor
*port binding* rule [@twelve-port].

!!! warning "A server bound to localhost is unreachable"
    A start command copied from local development may bind the server to
    `localhost`. In a container it then starts, logs a healthy message, and
    never receives a request. Set the host explicitly to `0.0.0.0` in the
    start command or configuration.

Startup is where configuration errors should end the process. A service that
reads its variables lazily starts successfully and fails later, on the first
request that needs the missing value, which turns a deploy error into an
intermittent user-facing one. A service that checks its configuration at
startup and exits with a clear reason fails the deploy instead, where the
platform keeps the previous version serving.

!!! recipe "A startup check"
    At startup, before listening on the port, the service:

    1. reads every variable in its inventory ([Chapter 1](anatomy.md)) and
       parses those that have structure: URLs, numbers, JSON;
    2. exits with a non-zero status on the first missing or malformed value,
       printing one line that names the variable and the problem, never the
       value;
    3. checks external facts it depends on and cannot work without, such as
       a bucket being private and unversioned ([Chapter 6](storage.md)), with
       the same rule for messages;
    4. only then starts listening, so that the platform's startup probe
       succeeds only for an instance that can serve.

The platform's probes take over from there (Section 1.4). A startup probe that
does not succeed within its window gets the instance shut down; a failing
liveness probe gets it restarted; and where no startup probe is configured,
Cloud Run adds one that only checks that the port accepts TCP connections
[@run-health].
At the other end of an instance's life, the platform sends `SIGTERM` before
stopping it; the process stops accepting new requests, finishes those in
progress and exits [@twelve-disposability]. Logs go to standard output, where
the platform collects them, not to files inside the container
[@twelve-logs].

## Environment variables

Each service's configuration is a set of environment variables kept by the
platform, with secrets marked as such. Two properties of platform consoles
cause most configuration incidents.

The first is that consoles take values literally. A `.env` file is parsed by
the tool that reads it, and tools such as Docker Compose remove the quotes
around a value, so that `FLAGS='{"uploads":true}'` yields the JSON object
`{"uploads":true}` [@compose-env]. A console's field stores exactly what was
typed. Many consoles offer a bulk editor that accepts `.env`-style text, and
whether that editor strips quotes depends on the platform; when it does not,
the variable arrives as `'{"uploads":true}'`, the quotes included. A service
that parses it as JSON fails at startup, which the startup check above at
least reports by name.

!!! recipe "Checking variables after a bulk paste"
    After pasting variables into a console, inspect them in a running
    instance (Section 5.6) without printing secret values. The first and last
    characters of a value show stray quotes; the length shows truncation:

    ```bash
    for name in FLAGS STORAGE_BUCKET WEB_ORIGIN; do
      value=$(printenv "$name")
      printf '%s: first=%s last=%s length=%s\n' \
        "$name" "${value:0:1}" "${value: -1}" "${#value}"
    done
    ```

    For a JSON value, `first` must be `{` or `[`, never `'` or `"`. For a
    secret, check only the length against the password manager entry.

The second is that a changed variable does not reach running instances. On
revision-based platforms a change to the container's configuration, its
variables included, creates a new revision, while a change to a stored secret
may require the running revision to be restarted before the container sees it
[@azure-revisions]. In either case the change takes effect with a new version
or a restart, and should be treated as a release: made deliberately, then
verified.

## Scaling

A platform runs between a minimum and a maximum number of instances of each
service, adding instances as requests arrive and removing idle ones. A minimum
of zero, the default on several platforms, means that an idle service costs
nothing: it is not billed while scaled to zero [@azure-scale]. The price is the
*cold start*: the first request after an idle period waits while an instance
starts. WeChat's container hosting, for example, scales a service with a
minimum of zero down after half an hour without requests, and the next request
waits for a new instance, for as long as the service takes to start
[@wx-cloudrun]. A minimum of one or more removes cold starts for the instances
kept warm, and those instances are billed even when idle, at a reduced rate on
some platforms [@run-min].

A further constraint applies to the internal API of the same-origin pattern. A
service scaled to zero is started again when a request for it arrives, but on
some platforms only requests that pass through the public routing layer count,
and a request to the service's internal address does not start an instance. An
API reached only through its internal address then stays down after its first
idle period, and every `/api` request fails with a `502` or `504`. Azure
documents a related case: an application with ingress disabled and no minimum
scales to zero and has no way of starting again [@azure-scale].

| Setting | Effect | Suits |
|---|---|---|
| Minimum 0 | No cost when idle; cold start after idle periods | Staging; a public web service whose cold start is acceptable |
| Minimum 1 | One warm instance always running and billed | An API reached only at an internal address; production services |
| Maximum *n* | Caps cost and the number of database connections | Every service; derive *n* from the database's connection limit |

Table: Scaling settings and where each one fits.

!!! remark "The cost of one warm instance"
    One always-on instance of a small container is usually the largest fixed
    cost of a small deployment, since everything else is billed by use. It is
    still the right default for an internal API, because the alternative
    fails in a way that looks like an outage. The web service can stay at
    zero where a cold start of a few seconds on the first visit is
    acceptable, because public requests wake it.

## Versions and rollback

Each deployment creates a *version* (Cloud Run and Azure call it a
*revision*): an immutable combination of one image and one configuration
[@azure-revisions]. The platform routes traffic to versions, all of it to the
newest by default. It can also deploy a version without traffic, split traffic
between versions for a gradual rollout, and roll back by routing traffic to an
earlier version again, which takes effect immediately because the old version
needs no build [@run-rollouts].

Rollback is only as good as the version kept for it. The previous working
version stays deployed, or at least available for reactivation, until the new
one has passed verification ([Chapter 8](releases.md)). Rolling back code is
fast; rolling back a schema is not, because a migration cannot be unapplied by
switching traffic, which is why migrations are written so that the previous
version of the code still works with the new schema.

Failed versions are deleted once their logs have been read. They serve no
traffic, but they clutter the list in which an operator looks for the version
to roll back to, and on platforms that cap the number of retained versions
they push out older, working ones [@azure-revisions].

## Diagnostics

A failed deploy is diagnosed from its log, and the log of a container that did
not start usually ends with the container's own output: the last lines show the
exception, the missing variable or the port the process tried to bind. Reading
from the end, not from the top of the build log, saves most of the time. The
common causes are few: the process listens on the wrong host or port, a
variable is missing or malformed, or the start command points at a file the
build did not produce.

Questions that the logs cannot answer are best asked from inside a running
instance. Most platforms offer a shell in a running container, as `docker
exec` does locally [@docker-exec] and the container console does on Azure
[@azure-console]. A command run there uses the container's own network, so it
can reach internal addresses, and its own credentials, so a check of the
database or the bucket runs with exactly the permissions the service has, and
no secret has to leave the container to make it.

!!! recipe "Checks from inside an instance"
    From a shell in a web-service instance, test the forwarding target:

    ```bash
    curl -sS -o /dev/null -w '%{http_code}\n' "$API_INTERNAL_URL/health"
    ```

    From a shell in an API instance, list the configured variable names
    without their values, and test outbound reachability of a backing
    service:

    ```bash
    printenv | cut -d= -f1 | sort
    curl -sS -o /dev/null -w '%{http_code}\n' https://<storage-endpoint>/
    ```

    A storage endpoint answering at all, even with `403`, proves the network
    path; whether the credentials work is a question for the application's
    own check, run in the same shell.

[Chapter 13](../cloudbase/cloud-run.md) applies this chapter to CloudBase
Cloud Run: deployment from a private repository, variables, scaling, versions,
logs, and the instance shell that the console calls Webshell.

!!! summary "Key points"
    - Build small images with multi-stage builds, a `.dockerignore` and a
      non-root user; keep configuration and secrets out of them.
    - Listen on `0.0.0.0` and the port the platform names. Check
      configuration at startup and exit with a clear reason; shut down
      cleanly on `SIGTERM`; log to standard output.
    - Consoles store variables literally. After a bulk paste, check that no
      quotes were kept, without printing secret values.
    - Scale-to-zero saves money at the cost of cold starts. An API reached
      only at an internal address needs a minimum of one instance where
      internal requests do not wake a service.
    - Each deploy is an immutable version; keep the previous one for
      rollback and delete failed ones.
    - Read a failed deploy's log from the end, and run checks from a shell
      inside an instance, where they use the service's own network and
      credentials.
