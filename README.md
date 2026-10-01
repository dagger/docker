# docker

A [Dagger](https://dagger.io) module, written in Dang, for Dockerfile and
Docker Compose projects. It lints Dockerfiles with
[hadolint](https://github.com/hadolint/hadolint), builds them, validates
Compose files, builds and runs Compose services, and wraps a Docker engine
and CLI.

## Requirements

Requires Dagger v1.0.0-beta.15 or later.

## Install

```sh
dagger install github.com/dagger/docker
```

## What it finds

The module works on four collections. You can select from each by key:

| Collection                                     | Key                              | Flag                            |
| ---------------------------------------------- | -------------------------------- | ------------------------------- |
| Docker projects (`projects`)                   | directory holding a `Dockerfile` | `--docker-project=PATH`         |
| Dockerfile stages (`projects/stages`)          | stage name                       | `--docker-stage=NAME`           |
| Compose projects (`compose/projects`)          | directory holding a Compose file | `--docker-compose-project=PATH` |
| Compose services (`compose/projects/services`) | service name                     | `--docker-compose-service=NAME` |

Paths are relative to the workspace root (`.` for the root). List them:

```sh
dagger list docker-projects -a
dagger list docker-stages -a
dagger list docker-compose-projects -a
dagger list docker-compose-services -a
```

Listing never starts a container. Projects come from file names, stages from
the Dockerfile's text, and services from the Compose files' text.

### Stages

The stages collection holds only the named stages that the project build
never reaches. A stage is in it when:

- no other stage references it, by name or by index: not as a `FROM` base,
  not as a `COPY`/`ADD --from` source and not as a `RUN --mount` `from=`
  source;
- it is not the final stage;
- it is not the `buildTarget` setting.

Those are typically `test` or `lint` stages; the project build covers the
rest. Unnamed stages can't be targeted, so they are never keys.

`allStages` lists every stage with its `index`, `name`, `base` and
`referenced` flag, and `stage(name:)` returns any one of them.

Stages are read line by line from `FROM` instructions, so a `FROM` inside a
heredoc or after a line continuation looks like a real one. A leading UTF-8
byte order mark is ignored.

### Compose files

A Compose project's file is `compose.yaml`, `compose.yml`,
`docker-compose.yml` or `docker-compose.yaml`; when a directory holds several,
the first in that order wins, as in Compose. Compose also merges an override
file over it: the first of `compose.override.yml`, `compose.override.yaml`,
`docker-compose.override.yml` and `docker-compose.override.yaml`. The checks
and runners call `docker compose` in the project directory without `-f`, so
Compose picks and merges the files itself (including `COMPOSE_FILE` from a
`.env` file).

Service names are read from the top-level `services` mapping of the Compose
file and its override file, without running Compose. The parser understands:

- block mappings at any indentation
- flow mappings (`services: {web: {...}}`)
- quoted keys, comments and a leading byte order mark

It does not see:

- services pulled in with `include:`, or from files named by `-f` or
  `COMPOSE_FILE`;
- services merged in with YAML anchors and merge keys (`<<: *base`) at the
  `services` level, or a `services:` mapping given as an alias;
- environment interpolation in service names.

A file the module can't read this way lists no services, and never fails the
listing.

## Working directory

What you see depends on where you run `dagger`:

- **At a project root:** that project and the projects below it.
- **Inside a project's subdirectory:** the enclosing project, plus any
  projects below where you stand.
- **Outside any project:** the projects below you.

So you don't need a flag to check the project you are working in:

```sh
cd app/src && dagger check --docker    # checks the app project and its stages
```

## Checks

| Address                                          | Check name      | Runs                                           |
| ------------------------------------------------ | --------------- | ---------------------------------------------- |
| `docker/projects/lint`                           | `lint`          | hadolint on the project's `Dockerfile`         |
| `docker/projects/build`                          | `build`         | a build of the project's `Dockerfile`          |
| `docker/projects/stages/build-stage`             | `build-stage`   | a build of each stage in the stages collection |
| `docker/compose/projects/lint`                   | `lint`          | `docker compose config --quiet`                |
| `docker/compose/projects/services/build-service` | `build-service` | a build of each service that declares `build`  |

- The project `build` builds the final stage. When the `buildTarget` setting
  names a stage the Dockerfile declares, it builds that stage instead.
- A service's build uses its build args, target and Dockerfile from the
  resolved Compose configuration. Its context may be anywhere in the
  workspace (`context: ../shared`); a context outside the workspace fails the
  check.
- A service that runs an image has nothing to build, so its check passes.
  Services behind a profile are built too.

```sh
dagger check                                  # every check in the workspace
dagger check --docker                         # every check from this module
dagger check --docker-project=app             # one project: lint, build and its stages
dagger check --docker-projects --check lint   # hadolint on every project
dagger check --check build-stage              # every stage build
dagger check --docker-project=app --docker-stage=test
dagger check --docker-compose-service=api     # build one Compose service
dagger check -l --all --docker                # list one line per check
dagger check -l --all --docker -f=cli         # ...as flags you can paste back
```

`--docker-stage=NAME` and `--docker-compose-service=NAME` match that name in
every project. In a repository where many Dockerfiles declare a `dev-envs`
stage, `--docker-stage=dev-envs` selects all of them. Add `--docker-project`
or `--docker-compose-project` to narrow it down.

The selected items run in parallel. Every failure is reported with the
project, stage or service and the step that failed. A build that fails in a
`RUN` instruction reports its exit code. The instruction's output is in the
trace, not in the message, because the engine's build error doesn't carry
it:

```
docker build failed in 1 of 2 projects:
- broken: docker build failed (exit code 3 in a RUN step; see the trace for its output)
```

Other build errors are reported without the engine's internal object IDs.

Run checks with `dagger check`, in CI especially: `dagger call` on a check
function does not fail the command when the check fails.

The flags for this module (see `dagger check --help`):

| Flag                            | Selects                                  |
| ------------------------------- | ---------------------------------------- |
| `--docker`, `--by-docker`       | checks from this module                  |
| `--docker-project=PATH`         | one Docker project (repeatable)          |
| `--docker-projects`             | every Docker project                     |
| `--docker-stage=NAME`           | stages with that name (repeatable)       |
| `--docker-stages`               | every stage                              |
| `--docker-compose-project=PATH` | one Compose project (repeatable)         |
| `--docker-compose-projects`     | every Compose project                    |
| `--docker-compose-service=NAME` | services with that name (repeatable)     |
| `--docker-services`             | every Compose service                    |
| `--check NAME`                  | checks with that name, in every module   |

The flag names can change when another installed module has an item type
with the same name; `dagger check --help` lists the flags in effect.

### hadolint

hadolint runs with `--no-color`. It reads the nearest `.hadolint.yaml` or
`.hadolint.yml` from the project's directory up to the workspace root. Extra
arguments come from the `lintArgs` setting, for example
`["--failure-threshold", "warning"]`.

## Running Compose services

`up` on a project's services runs the selected services the way
`docker compose up` would, as far as Dagger can:

- **Dependencies:** the selected services and every service they
  `depends_on` are started.
- **Networking:** each service's name is its hostname for the whole Dagger
  session, so once a service is running, any other service can reach it by
  name, as on Compose's default network. What differs is start order and
  waiting. Each service is bound to the services that start before it, so it
  starts only after they are up: Dagger waits until their exposed ports
  accept connections. Services start in `depends_on` order; among services
  ready to start, image-only services come first, then by name. A service
  isn't held back for one that starts after it. In prometheus-grafana, for
  example, `grafana` starts before `prometheus` (by name), so its queries
  fail until prometheus is up and then succeed. Declare `depends_on` when a
  service needs another to be up when it starts. Dagger service bindings
  can't form cycles, so a `depends_on` cycle is broken in the same order.
- **Ports:** every published TCP port of every started service is published
  through one [proxy](https://github.com/dagger/proxy) service, as raw TCP,
  including the same target port on several published ports. A port without
  a published port is published on its target port. Two services can't
  publish the same port, as with Compose. UDP ports are not published (see
  the warnings below). Ports from `ports` and `expose` are what Dagger waits
  for before starting dependents.
- **Container:** each service runs its image, or the image its build context
  builds. Compose's entrypoint, command, environment (including `env_file`),
  user, working directory and `privileged` are applied.
- **Mounts:**
  - bind mounts of files or directories inside the workspace are mounted as
    copies, so writes don't reach your files;
  - named volumes become cache volumes, one per project and volume. As with a
    new Docker volume, an empty one starts as a copy of the image's directory
    at the mount path when the image has one. The volume belongs to the user
    the service runs as, so non-root images (Prometheus runs as `nobody`) can
    write to it;
  - `tmpfs` mounts become temporary directories;
  - file-based secrets and configs are mounted at their targets
    (`/run/secrets/<name>` by default).
- **Profiles:** services behind a profile are not started.

What it can't reproduce is printed as a warning in the trace, per service,
for example `warning: app (service web): cap_add is ignored`. The same list
is available as `warnings` on a service:

- UDP ports and `expose` entries that aren't single TCP ports;
- bind mounts outside the workspace (such as `/var/run/docker.sock`) and
  missing bind sources;
- secrets and configs from the environment or external;
- other mount types;
- `cap_add`, `cap_drop`, `devices`, `dns`, `deploy`, `extra_hosts`, `init`,
  `ipc`, `network_mode`, `pid`, `platform`, `runtime`, `security_opt`,
  `shm_size`, `stop_signal`, `sysctls`, `tmpfs` and `ulimits`.

Not reproduced, and not warned about:

- healthchecks and `depends_on` conditions: Dagger waits for exposed ports
  instead;
- networks and network aliases;
- restart policies and replicas;
- the exact ownership of a seeded volume: the whole volume belongs to the
  service's user, where Docker keeps the image directory's ownership.

```sh
dagger up -l                                # what dagger up would start
dagger up --docker-compose-project=svc      # the services of one project
```

A bare `dagger up` starts every service in the workspace, including this
module's `engine` functions (Docker-in-Docker), so select the Compose
project.

`injectServices` returns the `composeBase` container with services bound to
it by name, and `composeEnvs` set as environment variables. On the services
collection, it binds the selected services and their dependencies. The
top-level `injectServices` binds every service of the one Compose project in
your working directory; exactly one project must be visible. The services
are started as for `up`. `composeBase` is an empty container by default, so
set it to an image you can run commands in:

```sh
dagger settings docker composeBase alpine:3.20
cd svc && dagger call docker inject-services with-exec --args sh,-c,'getent hosts web' stdout
```

To run or bind only some services, use `up` or `injectServices` on a subset
of the services collection from another module (below).

## Using it from another module

`projects(ws)`, `stages(ws)`, `compose.projects(ws)` and `services(ws)`
return collections. Use `keys`, `get(key:)` and `subset(keys:)` to select
items, and `batch` to run a function over the selection:

```dang
let projects = docker.projects(ws)
projects.keys                                        # ["app", "svc/api"]
run(projects.batch.lint(ws))                         # lint every project
run(projects.subset(keys: ["app"]).batch.build(ws))  # build one
run(projects.get(key: "app").stages(ws).batch.buildStage(ws))
projects.get(key: "app").container(ws, target: "test")  # the built image

let services = docker.compose.projects(ws).get(key: "svc").services(ws)
services.subset(keys: ["web", "db"]).batch.up(ws)    # a Service
services.subset(keys: ["db"]).batch.injectServices(ws)  # a Container
services.get(key: "web").warnings(ws)                # what up can't reproduce
```

A check called through a dependency returns a `Check` that has not run yet.
Wrap it to run it and raise its failure:

```dang
let run(check: Check!): Void {
  if (check.pass == false) {
    raise check.error.message ?? "check failed"
  }
  null
}
```

From the CLI, `dagger call` can't step into a collection yet; a Dagger script
can:

```sh
dagger -c 'docker | projects | get app | container | with-exec cat /etc/os-release | stdout'
dagger -c 'docker | projects | get app | all-stages | name'
```

## Docker engine and CLI

| Function                                             | Returns                                                              |
| ---------------------------------------------------- | -------------------------------------------------------------------- |
| `engine(version, persist, namespace)`                | an ephemeral Docker engine (Docker-in-Docker) as a `Service`         |
| `cli(version, engine)`                               | a Docker CLI wired to an engine (a new ephemeral one by default)     |
| `CLI.pull`, `push`, `load`, `run`, `image`, `images` | the matching `docker` commands                                       |
| `Image.export`, `push`, `ref`                        | an image in the engine's cache, exported into Dagger or pushed       |
| `project(path, findUp)`                              | the Docker project holding a path relative to your working directory |

## Settings

Set these in your workspace `dagger.toml`:

```toml
[modules.docker]
source = "github.com/dagger/docker"
settings.lintImage = "docker.io/hadolint/hadolint:v2.14.0-alpine"  # the default
settings.lintArgs = ["--failure-threshold", "warning"]  # default: []; extra hadolint arguments
settings.buildTarget = "production"            # default: ""; stage the project build builds where declared
settings.composeImage = "docker.io/docker/compose-bin:v5.5.0"  # the default; provides /docker-compose
settings.composeBase = "alpine:3.20"           # default: an empty container; what injectServices binds services to
settings.composeEnvs = ["LOG_LEVEL=debug"]     # default: []; KEY=VALUE set on composeBase
```

Or from the CLI:

```sh
dagger settings docker buildTarget production    # set
dagger settings -u docker buildTarget            # unset, back to the default
dagger settings docker                           # show
```

`buildTarget` is useful when Dockerfiles end in a development stage, such as
the `dev-envs` stage some repositories add last. A Dockerfile that doesn't
declare the stage still builds its final stage.

In `composeEnvs`, only the first `=` separates the name from the value, and
an entry without `=` sets an empty value.
