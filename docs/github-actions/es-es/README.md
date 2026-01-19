# GitHub Actions

**GitHub Action** es una herramienta de automatización integrada en **GitHub** que permite definir y ejecutar **flujos de trabajo** personalizados directamente en los repositorios.

Con **GitHub Actions**, puedes:
- Automatizar tareas repetitivas.
- Ejecutar scripts personalizados en respuesta a eventos como `push`, `pull_request` o `schedule`.
- Integrar herramientas y servicios externos.
- Implementar CI/CD.
- Construir y desplegar imágenes de Docker.
- Compilar y desplegar librerías.
- Crear y compartir acciones reutilizables para la comunidad.
- Entre otros.

Los **flujos de trabajo** se definen en archivos YAML dentro del directorio `.github/workflows/` y pueden incluir múltiples trabajos (`jobs`) y pasos (`steps`).

Las **acciones** se definen en archivos YAML dentro del directorio `.github/actions/`. Un action contiene un bloque `runs` para ejecutar múltiples pasos (`steps`).

## Workflows

Un **workflow** es un archivo YAML dentro de `.github/workflows/` que define:
- **Cuándo** debe ejecutarse (`on:` → triggers/eventos).
- **Qué** debe ejecutarse (`jobs:` → uno o varios jobs).
- **Dónde** se ejecuta (`runs-on`, contenedores, runners).
- **Cómo** se compone la ejecución (matrices, condicionales, dependencias).

### Características de workflows

Algunas características habituales y útiles en **GitHub Actions** son:

- **Orientado a eventos**: se ejecutan por eventos (push, PR, cron, manual, API, entre otros).
- **Ejecución paralela por defecto**: los jobs se ejecutan en paralelo si no hay `needs:`.
- **Dependencias entre jobs**: con `needs:` puedes orquestar pipelines (precondiciones → ejecución → reporte).
- **Condicionales por job o step**: con `if:` puedes evitar ejecuciones innecesarias o proteger ramas.
- **Reutilización**: usando `uses:` puedes invocar **Actions públicas** o **Actions locales** del repositorio (`./.github/actions/...`). También puedes invocar **Actions privadas** de otro repositorio siempre y cuando el repositorio que realiza la invocación o el token (`GITHUB_TOKEN` u otro token) tenga permisos para acceder al repositorio privado.
- **Entrada/salida entre jobs**: con `outputs:` en un job y `needs.<job>.outputs.<variable>`.
- **Matriz de ejecución**: `strategy.matrix` para expandir ejecuciones (por entornos, versiones, shards, entre otros.).
- **Control de paralelismo**: `max-parallel`, `fail-fast` y `concurrency` para evitar saturar runners.
- **Variables y secretos**:
  - `env:` para variables de ejecución.
  - `vars.*` para variables no sensibles configuradas en el repo/entorno.
  - `secrets.*` para credenciales.
- **Artefactos**: `actions/upload-artifact` para subir logs, reportes y resultados, y `actions/download-artifact` para descargar esos artefactos en jobs posteriores o en otros workflows.
- **Contenedores**: `jobs.<job>.container` para ejecutar todo el job dentro de una imagen de Docker.
- **Timeboxing**: `timeout-minutes` para evitar ejecuciones colgadas.
- **Filtros por rutas**: `on.push.paths` y `on.pull_request.paths` para disparar el workflow solo cuando cambian rutas relevantes.

### Triggers (eventos) en `on:`

El bloque `on:` define los **eventos** que disparan el workflow. Puedes usar uno o varios.

#### `push`

Se ejecuta cuando hay un push a una rama/tag.

Filtros comunes:
- `branches`: limita a ramas concretas.
- `tags`: limita a tags.
- `paths`: limita a cambios en rutas concretas.

> **Notas**  
>`paths` → filtra por archivos cambiados en ese push (reduce ejecuciones “ruidosas”).  
>`paths-ignore` → es el inverso.

#### `pull_request`

Se ejecuta ante eventos relacionados con **Pull Requests**.

Filtros comunes:
- `types`: controla *qué* acciones del PR disparan el workflow. Ejemplos típicos:
  - `opened`: cuando se abre el PR.
  - `synchronize`: cuando se empujan commits a la rama del PR.
  - `reopened`: cuando se reabre.
- `paths`: dispara el workflow solo cuando cambian rutas relevantes.

> **Nota** → Si pones `paths`, el workflow **no** se dispara si en la **Pull-request** no toca esas rutas o archivos.

![Workflow -> Trigger -> Pull Request](../gifs/workflows-automatic-trigger-in-PR.gif)

#### `workflow_dispatch`

Permite ejecutar el workflow **manualmente** desde la UI (o vía API) y pasar **inputs**.

Características:
- `inputs` pueden tener:
  - `description`, `required`, `default`
  - `type` (`string`, `boolean`, `choice`, `environment`, entre otros.)
  - `options` (si `type: choice`)

![Workflow -> Trigger -> Manual](../gifs/workflows-manual-trigger.gif)

#### `workflow_call`

Permite que un workflow sea **reutilizable** y sea invocado desde **otro workflow** (del mismo repo u otro repo, según permisos).

Características:
- Define una interfaz con `inputs:` y `secrets:`.
- Permite exponer `outputs:` del workflow llamado para usarlos en el workflow que lo invoca.
- Muy útil para estandarizar pipelines (por ejemplo: precondiciones → ejecutar tests → reportar) y reutilizarlos.

Ejemplo básico (estructura típica):

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

Permite disparar workflows mediante una llamada a la API de GitHub, enviando un `event_type` y opcionalmente un `client_payload`.

Características:
- `types: [<type>]` filtra por `event_type`.
- En el workflow, accedes a datos con `github.event.client_payload.<CAMPO>`.

Uso típico:
- Integraciones externas (por ejemplo, un orquestador o “train manager”) que pide ejecutar tests o despliegues.

Ejemplo de `repository_dispatch` con `client_payload`:

```bash
# Enviar un repository_dispatch con client_payload usando curl
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

Ejemplo de workflow que consume `client_payload`:

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

>Notas
>- Si usas un `GITHUB_TOKEN` proporcionado por GitHub Actions desde otro workflow, ten en cuenta que **no** puede usarse para llamar a la API `dispatches` contra otro repositorio. Para llamar a `repository_dispatch` normalmente necesitas un Personal Access Token (PAT) con los scopes adecuados.
>- Scopes recomendados para un PAT que llame a `repository_dispatch`:
>   - `repo` Acceso a repositorios privados si aplica
>   - `workflow` Si necesitas disparar workflows que requieren este scope
>- Alternativa segura: almacenar el PAT como `secrets.PAT_TOKEN` en el repo que ejecuta la llamada y usarlo en la cabecera `Authorization: Bearer ${{ secrets.PAT_TOKEN }}`.


#### `schedule`

Permite ejecuciones programadas usando `cron`.

**Características**:
- El cron se evalúa en **UTC**.
- Sintaxis:
  ```bash

    ┌───────────── minute (0 - 59)
    │ ┌───────────── hour (0 - 23)
    │ │ ┌───────────── day of the month (1 - 31)
    │ │ │ ┌───────────── month (1 - 12 or JAN-DEC)
    │ │ │ │ ┌───────────── day of the week (0 - 6 or SUN-SAT)
    │ │ │ │ │
    * * * * *
  ```
- Ideal para mantenimiento: limpieza, rotación de artefactos, checks o ejecucciones periódicas.

![Workflow -> Trigger -> Schedule](../gifs/workflows-automatic-trigger-with-schedule.gif)

#### Otros triggers comunes

- `pull_request_target`: Parecido a `pull_request` pero con permisos del repo base.
- `release`: Cuando se crea/pública una release.
- `issues`, `issue_comment`: Automatizaciones sobre issues/comentarios.
- `create`, `delete`: Creación/eliminación de ramas/tags.
- `deployment`, `deployment_status`: Flujos relacionados con despliegues.
- `check_run`, `check_suite`: Integraciones con checks.

### Ejemplo real de un `workflow`

Este ejemplo incluye: inputs, filtros por paths, matriz, contenedor, outputs, `needs`, `concurrency`, entre otros

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

## Actions

Una **Action** en **GitHub Actions** es una unidad reutilizable de código que realiza una tarea específica dentro de un flujo de trabajo. Las **Actions** permiten encapsular lógica y procesos comunes, facilitando su reutilización en diferentes flujos de trabajo y repositorios.

Características principales de una Action:
- **Reutilizable**: Puedes usar la misma Action en múltiples flujos de trabajo.
- **Personalizable**: Acepta entradas (`inputs`) y genera salidas (`outputs`).
- **Tipos de Actions**: Pueden ser acciones compuestas (`composite`), basadas en Node.js (`node12` o `node16`), o ejecutadas en contenedores Docker (`docker`).

Las **Actions** se definen mediante un archivo `action.yml` que especifica su configuración, entradas, salidas y pasos de ejecución.

### Archivos de un action

- 📂 action-name
    - 🛠️ action.yml
    - 📚 readme.md
    - 👾 entrypoint.sh
    - ...

#### Action file

El archivo `action.yml` es el archivo de metadatos principal que define una **GitHub Action**. Contiene información sobre la acción, como su nombre, descripción, entradas, salidas y los pasos necesarios para ejecutarla. Este archivo es esencial para que **GitHub** reconozca y ejecute la acción correctamente.


##### Ejemplo completo de un `action.yml` utilizando `using: composite`

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

#### Ejemplo básico de un `action.yml` utilizando `using: docker`

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

#### Ejemplo básico de un `action.yml` utilizando `using: node16`

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

El archivo `entrypoint.sh` es un script de shell que actúa como el punto de entrada principal para una GitHub Action. Este archivo se ejecuta cuando la acción es invocada y contiene las instrucciones necesarias para realizar las tareas definidas en la acción. Es comúnmente utilizado en acciones que se ejecutan en contenedores Docker o que requieren lógica personalizada.
Este archivo debe ser ejecutable, por lo que es importante asegurarse de que tenga los permisos adecuados (`chmod +x entrypoint.sh`).

**Ejemplo:**
```bash
#!/bin/bash
set -e
INPUT_PARAM1="$1"
echo "Ejecutando la acción con el parámetro: $INPUT_PARAM1"
```


## Workflows vs Actions

Aunque ambos se ejecutan dentro de **GitHub Actions**, **un Workflow** y **una Action** resuelven problemas distintos:

- Un **Workflow** orquesta un proceso completo: cuándo se ejecuta, en qué runner, con qué permisos, qué jobs corren, en qué orden y cómo se reporta.
- Una **Action** encapsula una tarea reutilizable: un bloque de lógica que puedes invocar como un *step* dentro de un job (por ejemplo: “configura credenciales”, “calcula features modificadas”, “sube artefactos”).

### Tabla comparativa

| Aspecto | Workflows | Actions |
|---|---|---|
| Definición / ubicación típica | `.github/workflows/*.yml` | `.github/actions/<name>/action.yml` (o repos externos/Marketplace) |
| ¿Qué es? | Pipeline/orquestación (eventos → jobs → steps) | Unidad reutilizable (un step reutilizable) |
| Trigger (`on:`) | Sí (push, pull_request, schedule, workflow_dispatch, workflow_call, entre otros.) | No (una action no “escucha” eventos; la ejecuta un workflow) |
| Estructura | `jobs` (con `needs`, `strategy`, `concurrency`, `permissions`) | `runs` (composite, docker o node) + `inputs/outputs` |
| Paralelismo / matrices | Sí (`strategy.matrix`, `max-parallel`, `fail-fast`) | No como tal (depende del workflow que la invoque) |
| Selección de runner / contenedor | Sí (`runs-on`, `container`, `services`) | No (hereda el entorno del job que la ejecuta) |
| Permisos y seguridad | Sí (por workflow/job: `permissions`, `environment`, approvals) | No define permisos de repo por sí sola (usa los del workflow/job) |
| Publicación y reutilización | Reutilizable vía `workflow_call` (otros workflows) | Muy reutilizable: se usa como `uses:` en muchos workflows/repos, puede publicarse en Marketplace |
| Objetivo principal | Definir el “qué/cuándo/dónde” de la automatización | Reutilizar el “cómo” (lógica repetible) |

### Casos reales de usos de Workflows

#### **Pull request + types + concurrency + lint**
- Archivo: `.github/workflows/linter.yaml`
- Trigger: `pull_request` con `types: [opened, synchronize, reopened]`.
- Patrón clave: `concurrency.group` para cancelar ejecuciones antiguas del mismo PR.
- Uso de acciones: `github/super-linter/slim` y subida de logs con `actions/upload-artifact` solo si falla.

#### **Push + paths + construir y pushear imagen de Docker**
- Archivo: `.github/workflows/pr-devops-update-python-img.yml`
- Trigger: `push` a `master` y además `paths:` para evitar builds innecesarios.
- Patrón clave: job con `timeout-minutes` y pasos que construyen tags dinámicos (outputs) y hacen build/push.

#### **PR con filtro por paths + matriz + contenedor de docker para tests por environment**
- Archivo: `.github/workflows/pr-run-tests-aura.yml`
- Trigger: `pull_request` + `paths` (solo se ejecuta si cambian rutas relevantes de acceptance y acciones).
- Patrón clave: job de precondiciones que calcula `outputs` (features modificadas, entornos/canales) y luego un job `run-tests` con `strategy.matrix`.
- Patrón clave: ejecución dentro de `container:` con `credentials:`.

#### **Ejecución manual y por API (workflow_dispatch + repository_dispatch)**
- Archivo: `.github/workflows/run-tests-aura.yml`
- Triggers:
  - `workflow_dispatch` con inputs (choices para environment/channel/tag).
  - `repository_dispatch` con `types: [run-tests]` para invocación vía API desde sistemas externos.
- Patrón clave: normalización de inputs según el trigger (inputs vs client_payload) y pipeline: preconditions → run-tests → report.

#### **Schedule (cron) para tareas de mantenimiento**
- Archivo: `.github/workflows/schedule-git-clean-workflows.yml`
- Trigger: `schedule.cron: "0 6 * * 5"`.
- Patrón clave: job sencillo de mantenimiento que usa una acción local para limpiar ejecuciones antiguas utilizando la cli de GitHub.

## Variables

### Vars

Las **variables** en **GitHub Actions** son valores definidos la **configuración** del repositorio. Estas variables pueden ser utilizadas para personalizar la ejecución de los pasos del flujo de trabajo. Para añadir una variable del repositorio tienes que tener los **permisos de usuario** requeridos y seguir los siguientes pasos:

#### Web

1. Desde el navegador ve a tu repositorio, ex: `https://github.com/<organization>/<repository_name>`
2. Click en **Settings**.
3. Navega hasta el apartado de **"Security"**.
4. Click en **"Secrets and Variables"**.
5. Click en **Actions**.
6. Click en **"Variables"**.
6. Click en **"New repository variable"**.
7. Rellenar los campos de nombre y valor.
8. Click en **"Add variable"**.

![Add a Variable](../gifs/add-a-variable-in-github.gif)

#### gh CLI

1. Ejecutar el comando desde una CLI `gh variable set <SECRET_NAME> --body "<secret_value>" --repo <organization>/<repository_name>`

#### Ejemplo de uso:
```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Imprimir variable
        run: echo "El entorno es ${{ vars.MY_VAR }}"
```

### Secrets

Los **secretos** en **GitHub Actions** son valores sensibles, como credenciales o tokens, que se almacenan de forma segura en el repositorio. Estos secretos se utilizan para proteger información confidencial durante la ejecución del Workflow.

Para añadir un secreto del repositorio tienes que tener los **permisos de usuario** requeridos y seguir los siguientes pasos.

#### Web

1. Desde el navegador ve a tu repositorio, ex: `https://github.com/<organization>/<repository_name>`
2. Click en **Settings**.
3. Navega hasta el apartado de **"Security"**.
4. Click en **"Secrets and Variables"**.
5. Click en **Actions**.
6. Click en **"New repository Secret"**.
7. Rellenar los campos de **nombre** y **valor**.
8. Click en **"Add secret"**.

![Add a secret](../gifs/add-a-secret-in-github.gif)

#### gh CLI

1. Ejecutar el comando desde una CLI `gh secret set SECRET_NAME --body "secret_value" --repo <organization>/<repository_name>`


#### Ejemplo de uso:
```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Desplegar aplicación
        run: |
          curl -H "Authorization: Bearer ${{ secrets.API_KEY }}" https://api.telefonica.com/deploy
```

### GitHub

| Variable                               | Descripción                                                     | Ejemplo                               |
|----------------------------------------|-----------------------------------------------------------------|---------------------------------------|
| github.repository                      | Nombre del repositorio en el formato owner/repo.                | Telefonica/aura-tests                 |
| github.repository_owner                | Propietario del repositorio.                                    | Telefonica                            |
| github.run_id                          | ID único del flujo de trabajo en ejecución.                     | 123456789                             |
| github.run_attempt                     | Número de intentos de ejecución del flujo de trabajo.           | 1                                     |
| github.job                             | Nombre del trabajo en ejecución.                                | build                                 |
| github.ref_type                        | Tipo de referencia del evento actual (branch o tag).            | branch                                |
| github.server_url                      | URL del servidor de GitHub.                                     | https://github.com                    |
| github.api_url                         | URL de la API de GitHub.                                        | https://api.github.com                |
| github.graphql_url                     | URL de la API GraphQL de GitHub.                                | https://api.github.com/graphql        |
| github.workspace                       | Directorio de trabajo del repositorio clonado.                  | /home/runner/work/repo-name/repo-name |
| github.action_path                     | Ruta al directorio donde se encuentra la acción en ejecución.   | /home/runner/work/_actions/org/action |
| github.token                           | Token de autenticación para interactuar con la API de GitHub.   | ghp_XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX  |
| github.head_ref                        | Nombre de la rama de la solicitud de extracción (pull request). | feature-branch                        |
| github.ref_name                        | Nombre de la referencia del evento actual.                      | main                                  |
| github.sha                             | SHA del commit que desencadenó el flujo de trabajo.             | ffac537e6cbbf934b08745a378932722dfff  |
| github.event.pull_request.title        | Título de la solicitud de extracción.                           | "Fix bug in authentication"           |
| github.event.pull_request.number       | Número de la solicitud de extracción.                           | 42                                    |
| github.event.pull_request.head.label   | Etiqueta de la rama de la solicitud de extracción.              | user:feature-branch                   |
| github.ref                             | Referencia del evento actual.                                   | refs/heads/main                       |
| github.actor                           | Usuario que desencadenó el evento.                              | spiderman                             |
| github.base_ref                        | Rama base de la solicitud de extracción.                        | main                                  |
| github.event_name                      | Nombre del evento que desencadenó el flujo de trabajo.          | push                                  |
| github.event.client_payload.<variable> | Valor de la variable del evento "repository_dispatch".          | production                            |
| github.run_number                      | Número de ejecución único para el flujo de trabajo.             | 123                                   |

### Runner

| Variable          | Descripción                                                             | Ejemplo              |
|-------------------|-------------------------------------------------------------------------|----------------------|
| runner.os         | Sistema operativo del runner que ejecuta el flujo de trabajo.           | Linux                |
| runner.arch       | Arquitectura del sistema operativo del runner.                          | X64                  |
| runner.name       | Nombre del runner que ejecuta el flujo de trabajo.                      | GitHub Actions 12    |
| runner.temp       | Directorio temporal disponible en el runner para almacenar archivos.    | /tmp                 |
| runner.tool_cache | Directorio donde se almacenan las herramientas instaladas en el runner. | /opt/hostedtoolcache |
| runner.workspace  | Directorio de trabajo del runner.                                       | /opt/workspace       |

## Documentación extra

- [GitHub Marketplace -> Actions ](https://github.com/marketplace?type=actions)
- [Quickstart for GitHub Actions](https://docs.github.com/en/actions/get-started/quickstart)
- [Telefonica -> QACDCO -> GitHub Actions Common](https://github.com/Telefonica/qacdco-github-actions)
