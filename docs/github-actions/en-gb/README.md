# GitHub Actions

**GitHub Action** is an automation tool integrated into **GitHub** that allows you to define and run **workflows** directly in repositories.

With **GitHub Actions**, you can:
- Automate repetitive tasks.
- Run custom scripts in response to events such as `push`, `pull_request` or `schedule`.
- Integrate external tools and services.
- Implement CI/CD.
- Build and deploy Docker images.
- Compile and deploy libraries.
- Create and share reusable actions with the community.
- Among other things.

Workflows are defined in YAML files under the `.github/workflows/` directory and may include multiple jobs (`jobs`) and steps (`steps`).

Actions are defined in YAML files under the `.github/actions/` directory. An action can include a `runs` block to execute multiple steps (`steps`).

## Workflows

A **workflow** is a YAML file inside `.github/workflows/` that defines:
- **When** it should run (`on:` → triggers/events).
- **What** should run (`jobs:` → one or more jobs).
- **Where** it runs (`runs-on`, containers, runners).
- **How** the execution is composed (matrices, conditionals, dependencies).

### Workflow features

Some common and useful features in **GitHub Actions** are:

- **Event-driven**: they run in response to events (push, PR, cron, manual, API, among others).
- **Parallel execution by default**: jobs run in parallel unless `needs:` is used.
- **Job dependencies**: with `needs:` you can orchestrate pipelines (preconditions → execution → reporting).
- **Conditionals at job or step level**: with `if:` you can avoid unnecessary runs or protect branches.
- **Reusability**: using `uses:` you can invoke **public Actions** or **local Actions** from the repository (`./.github/actions/...`). You can also invoke **private Actions** from another repository provided the repository performing the invocation or the token (`GITHUB_TOKEN` or another token) has permissions to access the private repository.
- **Job inputs/outputs**: with `outputs:` in a job and `needs.<job>.outputs.<variable>`.
- **Execution matrix**: `strategy.matrix` to expand runs (by environment, versions, shards, etc.).
- **Parallelism control**: `max-parallel`, `fail-fast` and `concurrency` to avoid overloading runners.
- **Variables and secrets**:
  - `env:` for runtime variables.
  - `vars.*` for non-sensitive variables configured in the repo/environment.
  - `secrets.*` for credentials.
- **Artifacts**: `actions/upload-artifact` to upload logs, reports and results, and `actions/download-artifact` to download those artifacts in later jobs or other workflows.
- **Containers**: `jobs.<job>.container` to run the whole job inside a Docker image.
- **Timeboxing**: `timeout-minutes` to avoid hung runs.
- **Path filters**: `on.push.paths` and `on.pull_request.paths` to trigger the workflow only when relevant paths change.

### Triggers (`on:`)

The `on:` block defines the **events** that trigger the workflow. You can use one or several.

#### `push`

Runs when there is a push to a branch or tag.

Common filters:
- `branches`: limit to specific branches.
- `tags`: limit to specific tags.
- `paths`: limit to changes in specific paths.

> Notes:
>`paths` filters by files changed in that push (reduces noisy runs).
>`paths-ignore` is the inverse.

#### `pull_request`

Runs on events related to **Pull Requests**.

Common filters:
- `types`: controls which PR events trigger the workflow. Typical examples:
  - `opened`: when the PR is opened.
  - `synchronize`: when commits are pushed to the PR branch.
  - `reopened`: when the PR is reopened.
- `paths`: triggers the workflow only when relevant paths change.

> Note: If you use `paths`, the workflow **will not** run if the PR does not touch those paths.

![Workflow -> Trigger -> Pull Request](../gifs/workflows-automatic-trigger-in-PR.gif)

#### `workflow_dispatch`

Allows running the workflow **manually** from the UI (or via API) and passing **inputs**.

Features:
- `inputs` can include:
  - `description`, `required`, `default`
  - `type` (`string`, `boolean`, `choice`, `environment`, among others)
  - `options` (if `type: choice`)

![Workflow -> Trigger -> Manual](../gifs/workflows-manual-trigger.gif)

#### `workflow_call`

Allows a workflow to be **reusable** and called from **another workflow** (from the same repo or another repo, subject to permissions).

Features:
- Defines an interface with `inputs:` and `secrets:`.
- Allows exposing `outputs:` from the called workflow for use by the caller.
- Very useful to standardise pipelines (for example: preconditions → run tests → report) and reuse them.

Example (typical structure):

```yaml
on:
  workflow_call:
    inputs:
      environment:
        description: "Execution environment"
        required: true
        type: string
    secrets:
      RTM_API_KEY:
        required: true
```

#### `repository_dispatch`

Allows triggering workflows via the GitHub API by sending an `event_type` and optionally a `client_payload`.

Features:
- `types: [<type>]` filters by `event_type`.
- In the workflow you access data with `github.event.client_payload.<FIELD>`.

Typical use:
- External integrations (for example, an orchestrator or "train manager") requesting runs, tests or deployments.

Example of `repository_dispatch` with `client_payload`:

```bash
# Send a repository_dispatch with client_payload using curl
curl -X POST \
  -H "Accept: application/vnd.github+json" \
  -H "Authorization: Bearer $GITHUB_TOKEN" \
  https://api.github.com/repos/<owner>/<repo>/dispatches \
  -d '{
    "event_type": "run-tests",
    "client_payload": {
      "environment": "ap-main",
      "features": "features/components/critical.feature"
    }
  }'
```

Example workflow consuming `client_payload`:

```yaml
on:
  repository_dispatch:
    types: [run-tests]

jobs:
  run-dispatched:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v5

      - name: Show client payload
        run: |
          echo "Environment: ${{ github.event.client_payload.environment }}"
          echo "Features: ${{ github.event.client_payload.features }}"
```

> Notes:
- If you use a `GITHUB_TOKEN` provided by GitHub Actions from another workflow, note that it **cannot** be used to call the `dispatches` API on another repository. To call `repository_dispatch` you typically need a Personal Access Token (PAT) with the appropriate scopes.
- Recommended PAT scopes for calling `repository_dispatch`:
  - `repo` Access to private repositories if applicable
  - `workflow` If you need to trigger workflows that require this scope
- Secure alternative: store the PAT as `secrets.PAT_TOKEN` in the repo that performs the call and use it in the `Authorization: Bearer ${{ secrets.PAT_TOKEN }}` header.


#### `schedule`

Allows scheduled runs using `cron`.

**Features**:
- The cron is evaluated in **UTC**.
- Syntax:
```bash

  ┌───────────── minute (0 - 59)
  │ ┌───────────── hour (0 - 23)
  │ │ ┌───────────── day of the month (1 - 31)
  │ │ │ ┌───────────── month (1 - 12 or JAN-DEC)
  │ │ │ │ ┌───────────── day of the week (0 - 6 or SUN-SAT)
  │ │ │ │ │
  * * * * *
```
- Ideal for maintenance: cleanup, artifact rotation, checks or periodic runs.

![Workflow -> Trigger -> Schedule](../gifs/workflows-automatic-trigger-with-schedule.gif)

#### `workflow_run`

`workflow_run` is the trigger that runs a workflow in response to the execution of another workflow within the same repository. Use it to chain pipelines (for example: run a report or deploy artifacts when tests finish) or to react to the start/end of a workflow execution.

Definition and behavior:
- **Scope**: only listens to workflow executions in the same repository. It does not fire for workflows in other repositories.
- **Event types (`types`)**: supports `requested` and `completed`.
  - `requested`: fires when a run of the originating workflow is requested/launched (before it actually starts). Useful for pre-run notifications or preparation steps.
  - `completed`: fires when the originating workflow run finishes.
- **Filter by workflow**: use `workflows:` to specify one or more workflow filenames (for example `run-tests.yml`) to limit invocations.
- **Filter by `conclusions`**: when using `types: [completed]` you can filter by final result using `conclusions:` (values: `success`, `failure`, `cancelled`, `skipped`, `neutral`, `timed_out`).
- **Context**: the receiving workflow has access to `github.event.workflow_run` with useful fields such as:
  - `id`, `html_url`, `head_sha`, `head_branch`, `run_number`, `status`, `conclusion`, `run_attempt`, `event`.
- **Permissions and security**: the receiver executes with its own permissions (`permissions:` in the receiving workflow); it does not automatically inherit elevated permissions from the originating workflow.
- **Be careful with loops**: avoid A → B → A cycles. Use `workflows:`, `conclusions:`, and `if:` conditions to prevent unintended loops.

Example 1 — `requested` (notify/prepare when a run is requested):

```yaml
on:
  workflow_run:
    workflows: ["run-tests.yml"]
    types: [requested]

jobs:
  notify-start:
    runs-on: ubuntu-latest
    steps:
      - name: Show requested run info
        run: |
          echo "Requested workflow: ${{ github.event.workflow_run.name }}"
          echo "Requested by: ${{ github.event.sender.login }}"
          echo "Commit: ${{ github.event.workflow_run.head_sha }}"
```

Example 2 — `completed` with `conclusions:` filter (only on success):

```yaml
on:
  workflow_run:
    workflows: ["run-tests.yml"]
    types: [completed]
    conclusions: [success]

jobs:
  report:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v5

      - name: Show triggering run info
        run: |
          echo "Triggered by workflow run: ${{ github.event.workflow_run.id }}"
          echo "Status: ${{ github.event.workflow_run.conclusion }}"
          echo "Commit: ${{ github.event.workflow_run.head_sha }}"
```

Practical tips:
- Use `workflows:` and `conclusions:` to minimise unnecessary runs.
- Add `if:` conditions on jobs to avoid secondary workflows firing on unexpected cases.
- Use `github.event.workflow_run.html_url` to link back to the originating run in reports.

#### Other common triggers

- `pull_request_target`: Similar to `pull_request` but runs with the base repository’s permissions.
- `release`: When a release is created/published.
- `issues`, `issue_comment`: Automations around issues/comments.
- `create`, `delete`: Branch/tag creation or deletion.
- `deployment`, `deployment_status`: Flows related to deployments.
- `check_run`, `check_suite`: Integrations with checks.

### Real workflow example

This example includes: inputs, path filters, matrix, container, outputs, `needs`, `concurrency`, among others

```yaml
name: "[EXAMPLE] 🧩 Workflow Example"
run-name: "[${{ github.event_name }}] ${{ github.ref_name }} by @${{ github.actor }}"

on:
  schedule:
    - cron: "0 6 * * 5"

  workflow_dispatch:
    inputs:
      git-revision:
        description: "Git revision/branch"
        required: true
        default: "master"
        type: string
      environment:
        description: "Execution environment"
        required: true
        default: "ap-main"
        type: choice
        options:
          - ap-next
          - ap-main
          - es-dev
      channel:
        description: "Execution channel"
        required: true
        default: "all"
        type: choice
        options:
          - al
          - gvp
          - all

  repository_dispatch:
    types: [run-tests]

  push:
    branches:
      - master
      - "release/**"
    paths:
      - "acceptance/**"
      - ".github/workflows/workflow.yaml"

  pull_request:
    types:
      - opened
      - synchronize
      - reopened
    paths:
      - "acceptance/**"
      - ".github/actions/**"
      - ".github/workflows/workflow.yaml"

env:
  SCRIPTS_PATH: ${{ github.workspace }}/acceptance/scripts
  DEFAULT_TIMEOUT_MINUTES: "60"

concurrency:
  group: "${{ github.workflow }} @ ${{ github.ref_name }}"
  cancel-in-progress: true

jobs:
  preconditions:
    runs-on: ubuntu-latest
    if: ${{ github.event_name != 'schedule' }}
    outputs:
      selected-features: ${{ steps.compute.outputs.selected-features }}
      workflow-start-time: ${{ steps.start-time.outputs.workflow-start-time }}
    steps:
      - name: Checkout
        uses: actions/checkout@v5

      - name: Capture workflow start time
        id: start-time
        run: echo "workflow-start-time=$(date +%s)" >> $GITHUB_OUTPUT

      - name: Compute what to run
        id: compute
        env:
          INPUT_FEATURES: ${{ inputs.features }}
        run: |
          echo "selected-features=${INPUT_FEATURES:-features/components/unittests/unittests.feature}" >> $GITHUB_OUTPUT

  run-tests:
    runs-on: ubuntu-latest
    needs: preconditions
    if: ${{ needs.preconditions.outputs.selected-features != '' }}
    container:
      image: ghcr.io/example/python-base:3.11
      credentials:
        username: ${{ secrets.REGISTRY_USERNAME }}
        password: ${{ secrets.REGISTRY_PASSWORD }}
      options: --user root
    env:
      FEATURES: ${{ needs.preconditions.outputs.selected-features }}
    strategy:
      fail-fast: false
      max-parallel: 4
      matrix:
        execute-environment:
          - ap-main
          - es-dev
    timeout-minutes: 90
    steps:
      - name: Checkout
        uses: actions/checkout@v5

      - name: Run tests in "${{ matrix.execute-environment }}"
        id: exec
        env:
          EXEC_ENV: ${{ matrix.execute-environment }}
        run: |
          echo "Running: $FEATURES on $EXEC_ENV"
        continue-on-error: true

  report:
    runs-on: ubuntu-latest
    needs: [preconditions, run-tests]
    if: ${{ always() }}
    steps:
      - name: Upload logs
        if: ${{ failure() }}
        uses: actions/upload-artifact@v4
        with:
          name: logs
          path: logs/**
```

### Real cases

These real workflows exemplify the triggers and patterns described above:

#### Pull request + types + concurrency + lint
- File: `.github/workflows/linter.yaml`
- Trigger: `pull_request` with `types: [opened, synchronize, reopened]`.
- Key pattern: `concurrency.group` to cancel old runs for the same PR.
- Uses actions like `github/super-linter/slim` and uploads logs with `actions/upload-artifact` only on failure.

#### Push + paths (only when relevant) to build/push images
- File: `.github/workflows/pr-devops-update-python-img.yml`
- Trigger: `push` to `master` and `paths:` to avoid unnecessary builds.
- Key pattern: job with `timeout-minutes` and steps that produce dynamic tags (outputs) and build/push images.

#### PR with path filters + matrix + container for environment-specific tests
- File: `.github/workflows/pr-run-tests-aura.yml`
- Trigger: `pull_request` + `paths` (only runs when acceptance/actions change).
- Key pattern: preconditions job that computes `outputs` (modified features, environments/channels), followed by a `run-tests` job with `strategy.matrix`.
- Key pattern: execution inside a `container:` with `credentials:`.

#### Manual and API invocation (workflow_dispatch + repository_dispatch)
- File: `.github/workflows/run-tests-aura.yml`
- Triggers:
  - `workflow_dispatch` with inputs (choices for environment/channel/tag).
  - `repository_dispatch` with `types: [run-tests]` for API invocation from external systems.
- Key pattern: normalise inputs according to the trigger (inputs vs client_payload) and pipeline: preconditions → run-tests → report.

#### Schedule (cron) for maintenance tasks
- File: `.github/workflows/schedule-git-clean-workflows.yml`
- Trigger: `schedule.cron: "0 6 * * 5"`.
- Key pattern: small maintenance job that uses a local action to clean old runs via the GitHub CLI.

## Actions

An **Action** in **GitHub Actions** is a reusable unit of code that performs a specific task within a workflow. Actions encapsulate logic and common processes, making them easy to reuse across workflows and repositories.

Main characteristics of an Action:
- **Reusable**: You can use the same Action in multiple workflows.
- **Customisable**: Accepts inputs (`inputs`) and produces outputs (`outputs`).
- **Action types**: They can be composite actions (`composite`), Node.js-based (`node12` or `node16`), or run inside Docker containers (`docker`).

Actions are defined with an `action.yml` file that specifies configuration, inputs, outputs and execution steps.

### Action directory

- 📂 action-name
    - 🛠️ action.yml
    - 📚 readme.md
    - 👾 entrypoint.sh
    - ...

#### Action file

The `action.yml` file is the main metadata file that defines a GitHub Action. It contains information such as name, description, inputs, outputs and the steps required to run it. This file is essential for GitHub to recognise and execute the action correctly.

##### Full example of an `action.yml` using `using: composite`

```yaml
name: Name of the action
description: "This action makes a custom task as an example."

inputs:
  example:
    description: "An example input for the action."
    required: true
  another_input:
    description: "Another optional input."
    required: false
    default: "default value"

outputs:
  result:
    description: "The result generated by the action."
    value: "The calculated or generated value."
  value:
    description: "The result of the output from a step."
    value: ${{ steps.output-from-steps.outputs.value }}

runs:
  using: "composite" # Indicates that the action type uses steps.
  steps:
    - name: Print message
      shell: bash
      run: |
        echo "Executing the action with the parameter: ${{ inputs.example }}"

    - name: Run script
      shell: bash
      if: ${{ steps.print.outputs.outcome == 'success' }}
      env:
        CUSTOM_ENV: "custom_env_value"
      run: |
        echo "This is a script with an environment variable: $CUSTOM_ENV"

    - name: Uses another action
      uses: actions/checkout@v3
      with:
        repository: "Telefonica/aura-tests"
        ref: "main"

    - name: Step with allowed errors
      shell: bash
      continue-on-error: true
      run: |
        echo "This step will continue even if there are errors."

    - name: Step with custom directory
      id: directory
      working-directory: ./sub_directory
      shell: bash
      run: |
        echo "Executing in a specific directory. Current directory ${pwd}"
    
    - name: Example of output from step
      id: output-from-steps
      shell: bash
      run: |
        value=$(echo "${{ inputs.example }}" | sed 's/, / /g')
        echo "value=$value" >> $GITHUB_OUTPUT
```

#### Basic `action.yml` example using `using: docker`

```yaml
name: Docker Action Example
description: "This action runs a task inside a Docker container."

inputs:
  input_var:
    description: "An input variable for the action."
    required: true

env:
  ENV_VAR: "default_value"

runs:
  using: "docker"
  image: "docker://node:16"
  args:
    - "${{ inputs.input_var }}"

  steps:
    - name: Basic Step
      run: echo "This is a basic step running inside the Docker container."
```

#### Basic `action.yml` example using `using: node16`

```yaml
name: Node.js Action Example
description: "This action runs a Node.js script."

inputs:
  input_var:
    description: "An input variable for the action."
    required: true

runs:
  using: "node16"
  main: "dist/index.js"
  pre: "dist/pre.js"

  steps:
    - name: Basic Node.js Step
      run: node -e "console.log('Running a basic Node.js step')"
```

#### Entrypoint file

The `entrypoint.sh` file is a shell script that acts as the main entrypoint for a GitHub Action. This file is executed when the action is invoked and contains the instructions necessary to perform the tasks defined in the action. It is commonly used in actions that run inside Docker containers or that require custom logic.
This file must be executable, so ensure it has the proper permissions (`chmod +x entrypoint.sh`).

**Example:**
```bash
#!/bin/bash
set -e
INPUT_PARAM1="$1"
echo "Executing the action with parameter: $INPUT_PARAM1"
```

### Using GITHUB_ENV

`GITHUB_ENV` is a special file provided by GitHub Actions to persist environment variables between steps within the same job. When a step writes `NAME=value` into the file path referenced by the `GITHUB_ENV` environment variable, subsequent steps will receive `NAME` as a normal environment variable.

When to use it:
- When you need a value computed in one step to be available as an environment variable in later steps within the same job.
- Prefer it over `echo export VAR=...` because it correctly supports multiline values and special characters.

Practical example:

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Calculate version
        id: calc
        run: |
          VERSION="1.2.$(date +%s)"
          echo "VERSION=$VERSION" >> $GITHUB_ENV

      - name: Use version in next step
        run: |
          echo "Calculated version is $VERSION"
```

Important notes:
- `GITHUB_ENV` only shares variables within the same job. To pass values between jobs use job `outputs` and `needs.<job>.outputs.<name>` in dependent jobs.
- Avoid writing secrets directly to `GITHUB_ENV`; use `secrets.*` or `env:` safely.
- If you run different shells or languages, make sure to use the correct syntax for that shell to redirect to `$GITHUB_ENV` (for example `>> $GITHUB_ENV`).


## Workflows vs Actions

Although both run within GitHub Actions, a **Workflow** and an **Action** solve different problems:

- A **Workflow** orchestrates an end-to-end process: when it runs, which runner it uses, what permissions, which jobs run, in what order and how it reports.
- An **Action** encapsulates a reusable task: a block of logic you can invoke as a *step* inside a job (for example: "configure credentials", "compute modified features", "upload artifacts").

| Aspect | Workflows | Actions |
|---|---|---|
| Definition / typical location | `.github/workflows/*.yml` | `.github/actions/<name>/action.yml` (or external repos/Marketplace) |
| What is it? | Pipeline/orchestration (events → jobs → steps) | Reusable unit (a reusable step) |
| Trigger (`on:`) | Yes (push, pull_request, schedule, workflow_dispatch, workflow_call, among others.) | No (an action does not "listen" to events; a workflow runs it) |
| Structure | `jobs` (with `needs`, `strategy`, `concurrency`, `permissions`) | `runs` (composite, docker or node) + `inputs/outputs` |
| Parallelism / matrices | Yes (`strategy.matrix`, `max-parallel`, `fail-fast`) | Not directly (depends on the invoking workflow) |
| Runner / container selection | Yes (`runs-on`, `container`, `services`) | No (inherits the job environment) |
| Permissions and security | Yes (per workflow/job: `permissions`, `environment`, approvals) | Does not define repo permissions by itself (uses the workflow/job permissions) |
| Publication and reuse | Reusable via `workflow_call` (other workflows) | Highly reusable: used via `uses:` in many workflows/repos, can be published in Marketplace |
| Primary goal | Define the "what/when/where" of automation | Reuse the "how" (repeatable logic) |

> Note: there are **reusable workflows** (`workflow_call`), but they remain workflows (called as jobs from another workflow). They do not replace actions when you need reuse at the step level.

## Variables

### Vars

Variables in GitHub Actions are values defined in the workflow or repository configuration. These values can be used to customise the execution of workflow steps.

To add a repository variable you need the required user permissions and follow these steps.

#### Web

1. From your browser go to your repository, e.g. `https://github.com/<organization>/<repository_name>`
2. Click on **Settings**.
3. Navigate to **Security**.
4. Click **Secrets and Variables**.
5. Click **Actions**.
6. Click **Variables**.
7. Click **New repository variable**.
8. Fill in name and value.
9. Click **Add variable**.

![Add a Variable](../gifs/add-a-variable-in-github.gif)

#### gh CLI

1. Run the command from a CLI: `gh variable set <SECRET_NAME> --body "<secret_value>" --repo <organization>/<repository_name>`

#### Example usage:
```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Print variable
        run: echo "The environment is ${{ vars.MY_VAR }}"
```

### Secrets

Secrets in GitHub Actions are sensitive values, such as credentials or tokens, stored securely in the repository. These secrets are used to protect confidential information during workflow execution.

To add a repository secret you need the required user permissions and follow these steps.

#### Web

1. From your browser go to your repository, e.g. `https://github.com/<organization>/<repository_name>`
2. Click on **Settings**.
3. Navigate to **Security**.
4. Click **Secrets and Variables**.
5. Click **Actions**.
6. Click **New repository Secret**.
7. Fill in name and value.
8. Click **Add secret**.

![Add a secret](../gifs/add-a-secret-in-github.gif)

#### gh CLI

1. Run the command from a CLI: `gh secret set SECRET_NAME --body "secret_value" --repo <organization>/<repository_name>`

#### Example usage:
```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy application
        run: |
          curl -H "Authorization: Bearer ${{ secrets.API_KEY }}" https://api.telefonica.com/deploy
```

### GitHub context variables

| Variable                               | Description                                                     | Example                               |
|----------------------------------------|-----------------------------------------------------------------|---------------------------------------|
| github.repository                      | Repository name in owner/repo format.                           | Telefonica/aura-tests                 |
| github.repository_owner                | Repository owner.                                               | Telefonica                            |
| github.run_id                          | Unique workflow run id.                                          | 123456789                             |
| github.run_attempt                     | Number of attempts for the workflow run.                        | 1                                     |
| github.job                             | Name of the job running.                                         | build                                 |
| github.ref_type                        | Type of reference of the current event (branch or tag).         | branch                                |
| github.server_url                      | URL of the GitHub server.                                        | https://github.com                    |
| github.api_url                         | URL of the GitHub API.                                           | https://api.github.com                |
| github.graphql_url                     | URL of the GitHub GraphQL API.                                   | https://api.github.com/graphql        |
| github.workspace                       | Working directory of the checked-out repository.                 | /home/runner/work/repo-name/repo-name |
| github.action_path                     | Path to the action directory being executed.                     | /home/runner/work/_actions/org/action |
| github.token                           | Token to interact with the GitHub API.                           | ghp_XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX  |
| github.head_ref                        | Name of the head ref for the pull request.                       | feature-branch                        |
| github.ref_name                        | Reference name of the current event.                             | main                                  |
| github.sha                             | Commit SHA that triggered the workflow.                          | ffac537e6cbbf934b08745a378932722dfff  |
| github.event.pull_request.title        | Pull request title.                                              | "Fix bug in authentication"           |
| github.event.pull_request.number       | Pull request number.                                             | 42                                    |
| github.event.pull_request.head.label   | Pull request head label.                                         | user:feature-branch                   |
| github.ref                             | The reference of the current event.                              | refs/heads/main                       |
| github.actor                           | Actor who triggered the event.                                   | spiderman                             |
| github.base_ref                        | Base branch for the pull request.                                | main                                  |
| github.event_name                      | Event name that triggered the workflow.                           | push                                  |
| github.event.client_payload.<variable> | Value from a repository_dispatch `client_payload`.                | production                            |
| github.run_number                      | Unique run number for the workflow.                              | 123                                   |
| github.event.workflow_run.id          | ID of the workflow run that triggered the event.                 | 123456789                             |
| github.event.workflow_run.name        | Name of the originating workflow.                                | run-tests.yml                         |
| github.event.workflow_run.html_url    | URL to the originating run in GitHub Actions.                    | https://github.com/.../actions/runs/123456789 |
| github.event.workflow_run.head_sha    | Commit SHA that triggered the originating run.                   | ffac537e6cbbf934b08745a378932722dfff  |
| github.event.workflow_run.head_branch | Branch associated with the originating run.                      | feature-branch                        |
| github.event.workflow_run.run_number  | Run number of the originating workflow.                          | 42                                    |
| github.event.workflow_run.status      | Status of the run (`queued`, `in_progress`, `completed`).        | completed                             |
| github.event.workflow_run.conclusion  | Final conclusion (`success`, `failure`, `cancelled`, ...).       | success                               |
| github.event.workflow_run.run_attempt | Attempt number for the run (1, 2, ...).                          | 1                                     |
| github.event.workflow_run.event       | Event that triggered the originating run (`push`, `workflow_dispatch`, ...). | push                                  |
| github.event.workflow_run.workflow_id | Internal numeric ID of the workflow.                             | 987654                                |

### Runner variables

| Variable          | Description                                                             | Example              |
|-------------------|-------------------------------------------------------------------------|----------------------|
| runner.os         | Operating system of the runner executing the workflow.                  | Linux                |
| runner.arch       | Runner architecture.                                                     | X64                  |
| runner.name       | Name of the runner.                                                      | GitHub Actions 12    |
| runner.temp       | Temporary directory available on the runner for storing files.          | /tmp                 |
| runner.tool_cache | Directory where tools are cached on the runner.                         | /opt/hostedtoolcache |
| runner.workspace  | Runner working directory.                                                | /opt/workspace       |

## Additional documentation

- [GitHub Marketplace -> Actions ](https://github.com/marketplace?type=actions)
- [Quickstart for GitHub Actions](https://docs.github.com/en/actions/get-started/quickstart)
- [Telefonica -> QACDCO -> GitHub Actions Common](https://github.com/Telefonica/qacdco-github-actions)
