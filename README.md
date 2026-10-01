# docker

A [Dagger](https://dagger.io) module, written in Dang, for Dockerfile and
Docker Compose projects: it lints Dockerfiles with
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

The module works on four kinds of things, each a collection you can select
from by key:

| Collection                                     | Key                              | Flag                             |
| ---------------------------------------------- | -------------------------------- | -------------------------------- |
| Docker projects (`projects`)                   | directory holding a `Dockerfile` | `--docker-project=PATH`          |
| Dockerfile stages (`projects/stages`)          | stage name                       | `--docker-dockerfile-stage=NAME` |
| Compose projects (`compose/projects`)          | directory holding a Compose file | `--docker-compose-project=PATH`  |
| Compose services (`compose/projects/services`) | service name                     | `--docker-compose-service=NAME`  |

Paths are relative to the workspace root (`.` for the root). A Compose file
is `compose.yaml`, `compose.yml`, `docker-compose.yaml` or
`docker-compose.yml`; when a directory holds several, the first in that order
is used, as Compose does.

List them:

```sh
dagger list docker-projects -a
dagger list docker-dockerfile-stages -a
dagger list docker-compose-projects -a
dagger list docker-compose-services -a
```

Listing never starts a container: projects come from file names, stages from
the Dockerfile's text, and services from the Compose file's text.

### Stages

The stages collection holds only the named stages a default build never
reaches: stages that no other stage references (as a `FROM` base, a
`COPY`/`ADD --from` source or a `RUN --mount` `from=` source, by name or by
index) and that are not the final stage. Those are typically `test` or
`lint` stages; the project's own build covers the rest. Unnamed stages can't
be targeted, so they are never keys.

`allStages` lists every stage with its `index`, `name`, `base` and
`referenced` flag, and `stage(name:)` returns any one of them.

Stages are read line by line from `FROM` instructions: a `FROM` inside a
heredoc or after a line continuation is not told apart from a real one.

### Services

Service names are read from the Compose file's top-level `services` mapping
without running Compose. Block mappings at any indentation, flow mappings
(`services: {web: {...}}`), quoted keys and comments are understood. Not
seen:

- services pulled in with `include:` or from other Compose files (override
  files, `-f`);
- services merged in with YAML anchors and merge keys (`<<: *base`) at the
  `services` level, or a `services:` mapping given as an alias;
- environment interpolation in service names.

A file the module can't read this way lists no services (it never fails the
listing). The checks and runners then use `docker compose config`, so they
see the file as Compose resolves it.

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

| Address                                  | Runs                                           |
| ---------------------------------------- | ---------------------------------------------- |
| `docker/projects/lint`                   | hadolint on the project's `Dockerfile`         |
| `docker/projects/build`                  | a build of the `Dockerfile`'s final stage      |
| `docker/projects/stages/build`           | a build of each stage in the stages collection |
| `docker/compose/projects/lint`           | `docker compose config --quiet`                |
| `docker/compose/projects/services/build` | a build of each service that declares `build`  |

A service that runs an image has nothing to build, so its build check
passes. Services behind a profile are built too.

```sh
dagger check                                        # every check in the workspace
dagger check --docker                               # every check from this module
dagger check --docker-project=app                   # one project: lint, build and its stages
dagger check --docker-projects --check lint         # hadolint on every project
dagger check --check build                          # every build check, in every module
dagger check docker/projects/stages/build --docker-dockerfile-stage=test
dagger check --docker-compose-service=api           # build one Compose service
dagger check -l --all --docker                      # list one line per check
dagger check -l --all --docker -f=cli               # ...as flags you can paste back
```

The selected items run in parallel, and every failure is reported with the
project, stage or service and the step that failed, for example:

```
docker build failed in 1 of 2 projects:
- broken: docker build failed:
failed to clone built container state: exit code: 3
```

Run checks with `dagger check`, in CI especially: `dagger call` on a check
function does not fail the command when the check fails.

The flags for this module (see `dagger check --help`):

| Flag                                | Selects                                |
| ----------------------------------- | -------------------------------------- |
| `--docker`, `--by-docker`           | checks from this module                |
| `--docker-project=PATH`             | one Docker project (repeatable)        |
| `--docker-projects`                 | every Docker project                   |
| `--docker-dockerfile-stage=NAME`    | one stage (repeatable)                 |
| `--docker-stages`                   | every stage                            |
| `--docker-compose-project=PATH`     | one Compose project (repeatable)       |
| `--docker-compose-projects`         | every Compose project                  |
| `--docker-compose-service=NAME`     | one Compose service (repeatable)       |
| `--docker-services`                 | every Compose service                  |
| `--check NAME`                      | checks with that name, in every module |

The flag names can change when another installed module has an item type
with the same name; `dagger check --help` lists the flags in effect.

## Running Compose services

`up` on a project's services runs the services that declare a port, behind
one [proxy](https://github.com/dagger/proxy) with a frontend port per service
(its published port, or its target port when none is published). Each
service runs its image, or the image its build context builds with its build
args, target and Dockerfile, with its entrypoint, command and environment.
Services behind a profile are not started.

```sh
dagger up -l                                        # what dagger up would start
dagger up --docker-compose-project=svc              # the services of one project
```

`dagger up` starts every service in the workspace, including this module's
`engine` functions (Docker-in-Docker), so select the Compose project.

`injectServices` returns the `composeBase` container with every service of
the Compose project in your working directory bound to it by service name,
and `composeEnvs` set as environment variables. Exactly one Compose project
must be visible. `composeBase` is an empty container by default, so set it
to an image you can run commands in:

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
run(projects.get(key: "app").stages(ws).batch.build(ws))
projects.get(key: "app").container(ws, target: "test")  # the built image

let services = docker.compose.projects(ws).get(key: "svc").services(ws)
services.subset(keys: ["web", "db"]).batch.up(ws)    # a Service
services.subset(keys: ["db"]).batch.injectServices(ws)  # a Container
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
settings.composeBase = "alpine:3.20"           # default: an empty container; what injectServices binds services to
settings.composeEnvs = ["LOG_LEVEL=debug"]     # default: []; KEY=VALUE set on composeBase
```

Or from the CLI:

```sh
dagger settings docker composeBase alpine:3.20    # set
dagger settings -u docker composeBase             # unset, back to the default
dagger settings docker                            # show
```

In `composeEnvs`, only the first `=` separates the name from the value, and
an entry without `=` sets an empty value.
