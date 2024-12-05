# Monorepo Workflow with GitLab Pipeline Runner

This guide demonstrates how to create a workflow for a monorepo using GitLab CI/CD pipelines. The setup is based on a **TurboRepo Kitchen Sink example project** featuring four application packages and one service package. The service package includes a UI library shared by three of the applications.

The goal is to create a streamlined workflow that ensures efficient builds and deployments based on changes in specific packages.

---

## Objectives

1. **Selective Builds and Deployments:**
   - Changes in an application package trigger only its build, test and deployment.
2. **Dependency Awareness:**

   - Changes in the shared UI package trigger its deployment first, followed by dependent applications in parallel.

3. **Combined Changes:**

   - When both the UI and applications are modified, the pipeline ensures:
     - UI deployment first.
     - All dependent applications deploy in parallel afterward.

4. **Independent Deployments:**

   - Changes in multiple independent application packages result in parallel deployments without unnecessary dependencies.

5. **UI and Independent Applications:**
   - Independent applications deploy in parallel.
   - UI-dependent applications deploy only after the UI package deployment.

---

## Prerequisites

### 1. Git and GitLab Repository

- **Git**: Ensure Git is installed .
  [Git](https://docs.gitlab.com/ee/topics/git/how_to_install_git/)
- **GitLab**: Create a GitLab account.
  [GitLab](https://docs.gitlab.com/ee/tutorials/)

### 2. Node.js and Package Manager

- **Node.js**: Install [Node.js](https://nodejs.org/) (version 18+ recommended).
- **Package Manager**: Use **pnpm** for efficient package management:
  ```bash
  npm install -g pnpm
  ```

### 3. turborepo set up

**Installation**:

```sh
pnpm install turbo --global
```

**create a project with turbo repo kitchen_sink example**:

```sh
pnpm dlx create-turbo@latest --example kitchen_sink
```

**Explore the project structure**:

1. apps/: Contains individual application packages (e.g., admin, blog).
2. packages/: Contains shared libraries (e.g., UI).

**Test the setup by running TurboRepo commands**:

```sh
pnpm run build
```

```sh
pnpm run dev
```

Familiarize yourself with TurboRepo concepts. Refer to the TurboRepo documentation.
[https://turbo.build/repo/docs](https://turbo.build/repo/docs)

# Workflow Structure

## Pipeline Work follow

create `.gitlab-ci.yml` file in the root of the project and similarly under each app folder.

1.  apps/admin
2.  apps/api
3.  apps/blog
4.  apps/storefront
5.  packages/ui

##### note:

- Here api has no dependency on ui.
- All other apps has a dependency on ui.

### How pipelines are identified?

- GitLab initially searches for a .gitlab-ci.yml file in the root directory of the project, which is considered the parent pipeline.
- If the parent pipeline references other .gitlab-ci.yml files, these pipelines are considered child pipelines.
- If a child pipeline itself contains a .gitlab-ci.yml file, that pipeline is further classified as a grandchild pipeline.

**Create Root (Parent) Pipeline: `.gitlab-ci.yml`**

1. **Create a `.gitlab-ci.yml` file:**

   - Create a `.gitlab-ci.yml` file in the root directory.

2. **Define Stages:**

   - The `trigger` stage is the only stage defined in the root pipeline.

3. **Specify Triggers:**

   - Each trigger should have a unique name.
   - Ensure that the child pipelines use the same names for their stages.
   - For example, for the `trigger_admin` trigger:
     - The child pipeline (e.g., `apps/admin/.gitlab-ci.yml`) should have stages ending with `_admin` (e.g., `build_admin`).

4. **Include Child Pipelines:**

   - Use the `include` keyword to specify the path to the child pipeline's `.gitlab-ci.yml` file.

5. **Set Job Dependencies:**

   - Use the `strategy: depend` setting to ensure that the trigger job succeeds only after all stages in the child pipeline have completed successfully.

6. **Define Trigger Conditions:**

   **Method 1: Using `rules`**

   - Use the `rules` keyword to define specific conditions for triggering the child pipeline.
   - Each rule can have one or more of the following conditions:
     - `if`: A specific condition to be met.
     - `changes`: A list of file paths that, when changed, trigger the pipeline.
     - `exists`: A file or directory that must exist to trigger the pipeline.
     - `when`: A specific event or schedule to trigger the pipeline.

   For example, to trigger the API child pipeline when changes are made to files in the `api` folder:

   ```yml
   trigger_api:
     stage: triggers
     trigger:
       include: apps/api/.gitlab-ci.yml
       strategy: depend
     rules:
       - changes:
           - apps/api/**/*
   ```

   **Method 2: Using `only` and `except`**

Use the `only` keyword to specify the file paths that, when changed, should trigger the pipeline. Use the `except` keyword to specify file paths that, when changed, should _not_ trigger the pipeline.

For instance, to trigger the admin child pipeline only when changes occur in the apps/admin folder and not in the packages/ui folder:
If changes are made in both folders, the except condition will take precedence.

```yml
trigger_admin:
  stage: triggers
  trigger:
    include: apps/admin/.gitlab-ci.yml
    strategy: depend
  only:
    changes:
      - "apps/admin/**/*"
  except:
    changes:
      - "packages/ui/**/*"
```

### Create Child Pipelines

consider the child pipeline for admin application.

**Cretae child (Downstream) Pipeline: `.gitlab-ci.yml`**

1. **Create a `.gitlab-ci.yml` file:**

   - Create a `.gitlab-ci.yml` file in the `apps/admin` directory.

2. **Define Stages:**

- Defines the stages of the pipeline in sequential order.

  - build: The first stage where the application is built.
  - test: The second stage, which runs after the build stage, to test the application.
  - deploy: The final stage, where the application is deployed.

```yml
   stages:
  - build
  - test
  - deploy
```

3. **Set Default Image:**
   - Specifies the default Docker image for all jobs in the pipeline.
   - node:20-alpine: A lightweight Node.js image based on Alpine Linux, used for running Node.js-related scripts.

```yml
default:
  image: node:20-alpine
```

4. **Build Admin Job:**
   - `stage: build`: Assigns this job to the build stage.
   - `script`: Lists the commands executed during the job.
     1. `echo "starting script"`: Prints a message to indicate the script has started.
     2. `apk add --no-cache --virtual .build-deps alpine-sdk`: Installs build dependencies using the Alpine package manager.
     3. `npm install -g pnpm`: Installs pnpm globally as the package manager.
     4. `pnpm install`: Installs the project's dependencies.
     5. `pnpm run build --filter admin`: Runs the build script specifically for the admin application using the pnpm workspace filter.
     6. `echo "build complete"`: Prints a message indicating the build is complete.

```yml
build_admin: # This job runs in the build stage, which runs first.
  stage: build
  script:
    - echo "starting script"
    - apk add --no-cache --virtual .build-deps alpine-sdk
    - npm install -g pnpm
    - pnpm install
    - echo "starting blog build"
    - pnpm run build --filter admin
    - echo "build complete"
```

5. **Test Admin Job:**
   - `stage: test`: Assigns this job to the test stage.
   - `needs: [build_admin]`: Specifies that this job depends on the successful completion of the build_admin job.
   - `script`: Contains the testing commands.
     1. eg: `echo "This job tests something."`: A placeholder command for testing logic.

```yml
test_admin:
  stage: test
  needs: [build_admin]
  script:
    - echo "This job tests something."
```

6. **Deploy Admin Job:**
   - `stage: deploy`: Assigns this job to the deploy stage.
   - `needs: [test_admin]`: Specifies that this job depends on the successful completion of the test_admin job.
   - `script`: Contains the deployment commands. 1. eg: `echo "This job deploys something."`: A placeholder command for deployment logic.

```yml
deploy_admin:
  stage: deploy
  needs: [test_admin]
  script:
    - echo "This job deploys something."
```

### Grandchild Pipelines

**Grandchild Pipeline**

A grandchild pipeline is a pipeline triggered by a child pipeline.

**Creating a Grandchild Pipeline:**

1. **Create a `.gitlab-ci.yml` file:**

   - Create a `.gitlab-ci.yml` file in the `ui` folder.

2. **Define a Trigger Stage:**

   - Add a `triggers` stage as the last stage in the `ui` pipeline.

```yml
stages:
  - build
  - test
  - deploy
  - triggers
```

3.  **Test and Deploy:**

- Build, test and deploy the changes in the ui.

```yml
build_ui: # This job runs in the build stage, which runs first.
  stage: build
  script:
    - echo "starting script"
    - apk add --no-cache --virtual .build-deps alpine-sdk
    - npm install -g pnpm
    - pnpm install
    - echo "starting blog build"
    - pnpm run build
    - echo "build complete"
test_ui:
  stage: test
  needs: [build_ui]
  script:
    - echo "This job tests something."

deploy_ui:
  stage: deploy
  needs: [test_ui]
  script:
    - echo "This job deploys something."
  environment: production
```

4. **Trigger second level child Pipelines:**

   - add a new trigger job.
   - specify the name of the trigger.
   - `stage:triggers` - Assigns this job to the triggers stage.
   - `needs: [deploy_ui]`: Specifies that this job depends on the successful completion of the deploy_ui job.
   - `trigger`: trigger a new pipeline based on the configuration in the specified file (`apps/admin/.gitlab-ci.yml`).

```yml
trigger_admin:
  stage: triggers
  needs: [deploy_ui]
  trigger:
    include: apps/admin/.gitlab-ci.yml
```

5. **Add all dependent application triggers:**
   - add trigger_app
   - add trigger_storefront

### Push your project to GitLab

1. **create a new project in GitLab without readme file**

2. **Configure your Git identity**
   [Git/GitLab configure](https://docs.gitlab.com/ee/topics/git/how_to_install_git/index.html#configure-git)

3. **Git local setup**
   - Configure your Git identity locally to use it only for this project:

```sh
  git config --local user.name < GitLab user name >
  git config --local user.email < GitLab user email >
```

note: This wont effect your git/github configuration in local.

4.  Push your monorepo to GitLab after committing your changes.

```sh
git remote add origin < GitLab project url >
git push --set-upstream origin main
```

5. Gitlab pipeline configuration

   - **GitLab Runner**:GitLab shared runners are always available by default.
     [GitLab- hosted-runners](https://docs.gitlab.com/ee/ci/runners/)
   - If you want to create your own runner follow this documentation
     [Custom Gitlab Runner](https://docs.gitlab.com/ee/ci/quick_start/index.html)

6. Test your pipelines.
   - make changes to only one app and push to remote
   - make changes to multiple independent app and push to remote
   - make changes to only ui and push to remote
   - make changes to ui and any dependent app and push to remote
   - make changes to ui and independent app(apps/api) and push to remote
7. Verify the pipelines.
