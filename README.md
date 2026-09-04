# GitHub Informer for Zoho Cliq
The GitHub Action is used to integrate GitHub and Zoho Cliq, by notifying about the GitHub Events performed, to the Zoho Cliq Channels.

GitHub Informer requires the following inputs to integrate the **GitHub Actions** with your **Cliq** channels
- Cliq Webhook Token
- Cliq Channel API Endpoint
- Individual messages for each of the GitHub events (in the name of **event**-message)
- A default message that you want to send if the message is not specified for that event.

Two posting modes are supported:

1. Webhook mode
  - **channel-endpoint**: `<Cliq Channel Endpoint>?zapikey=<Cliq Webhook Token>`

2. Bot-auth mode
  - **channel-endpoint**: `https://cliq.zoho.com/api/v2/channelsbyname/<CHANNEL_UNIQUE_NAME>/message?bot_unique_name=<BOT_UNIQUE_NAME>`
  - **channel-auth-token**: Cliq OAuth bearer token
  - the **bot must already be created** by the user and **added to the target channel**

### I. Initial setup order for a new repository

For a new repository, use this order:

1. Add the required repository variables.
2. Add the required environment secrets.
3. Commit and push the workflow file.
4. Run the workflow once so the GitHub status check is created.
5. Then add the branch protection or ruleset and require the AI Review Gate check.

This order matters because GitHub cannot require a status check before that check has been created by the workflow.

### II. Where to set bot or user notification mode

This must be configured as a **GitHub Repository Variable**, not hardcoded in the workflow YAML.

Use the workflow env to read the value from the variable:

```yaml
env:
  CLIQ_NOTIFICATION_MODE: ${{ vars.CLIQ_NOTIFICATION_MODE }}
  CLIQ_BOT_UNIQUE_NAME: ${{ vars.CLIQ_BOT_UNIQUE_NAME }}
```

- Set `CLIQ_NOTIFICATION_MODE` to `user` for normal webhook/user mode.
- Set `CLIQ_NOTIFICATION_MODE` to `bot` for bot notification mode.
- **`CLIQ_BOT_UNIQUE_NAME` is required when mode is `bot`.**

Set these in GitHub repository settings:

1. In GitHub repository settings, go to `Secrets and variables` -> `Actions` -> `Variables`.
2. Create **`CLIQ_NOTIFICATION_MODE`** with value **`user`** or **`bot`**.
3. If mode is `bot`, create **`CLIQ_BOT_UNIQUE_NAME`** with your actual bot unique name.

Do not add these values directly in `.github/workflows/CliqConnector.yml`.

### IV. Bot mode setup

If you want **bot notification mode**, configure:

1. **Create the bot in Zoho Cliq**.
2. **Get the bot unique name** from the bot configuration/details page.
  - Open **Integrations** -> **Bots**.
  - Select your bot.
  - In the bot details panel, check the **API Endpoint**.
  - The value after `/bots/` in the endpoint is the **bot unique name**.
  - Example: if the API endpoint is `https://cliq.zoho.com/api/v2/bots/githubnotification/message`, then the bot unique name is **`githubnotification`**.
3. **Add the bot to the target Cliq channel** where notifications should be posted.
4. In GitHub repository settings, go to `Secrets and variables` -> `Actions` -> `Variables`.
5. Create **`CLIQ_NOTIFICATION_MODE=bot`**.
6. Create **`CLIQ_BOT_UNIQUE_NAME=<your bot unique name>`**.

In bot mode:

- The workflow reuses the existing **`ENDPOINT`** secret.
- The workflow appends **`bot_unique_name`** automatically.
- If **`CLIQ_BOT_UNIQUE_NAME`** is missing, workflow fails fast.

### V. User mode setup

If you want normal user/webhook notification mode, configure:

1. `CLIQ_NOTIFICATION_MODE=user`

In user mode:

- **`CLIQ_BOT_UNIQUE_NAME` is not required.**
- Workflow will use the webhook endpoint as-is.
- **You do not need to reconfigure endpoint in this section**; keep using the existing **`ENDPOINT`** secret configured under **GitHub Secret for Channel Endpoint**.

## Available Regions

Use the same base domain as your Zoho Cliq organization region when configuring the channel endpoint.

Common region domains:

- US: `https://cliq.zoho.com`
- IN: `https://cliq.zoho.in`
- EU: `https://cliq.zoho.eu`
- AU: `https://cliq.zoho.com.au`
- JP: `https://cliq.zoho.jp`

Endpoint pattern:

- `<region-base>/api/v2/channelsbyname/<CHANNEL_UNIQUE_NAME>/message?zapikey=<WEBHOOK_TOKEN>`

Examples:

- `https://cliq.zoho.com/api/v2/channelsbyname/GitHubupdates/message?zapikey=1001.xxxxx`
- `https://cliq.zoho.in/api/v2/channelsbyname/GitHubupdates/message?zapikey=1001.xxxxx`
  
## VI. GitHub Secret for Channel Endpoint 🔗
You must add **GitHub Secret** which contains the channel endpoint in the format 

```
<Cliq Channel Endpoint>?zapikey=<Cliq Webhook Token>
```

You must create a **GitHub Secret** for providing the **Channel Endpoint** by
  - Go to the Repository where the CliqInformer will be added and go to the '**Settings**' tab.
  - Select '**Secrets and variables**' and click on '**Actions**' in the dropdown.
  - Click on '**New repository secret**' and enter the name of your secret and also enter the Cliq channel endpoint  as the Secret (in above mentioned format)

and use the secret as the '**channel-endpoint**' input in the job of your workflow.

```yaml
  steps:
    - uses: Integrations-dev/GitHub-Informer@v1
      with:
        channel-endpoint: ${{ secrets.SECRET_NAME }}
```

## VII. Store Cliq Thread ID in GitHub Project

For PR events, you can store and reuse the Cliq thread ID in a GitHub Project V2 custom text field (recommended).

Create GitHub classic token for `PROJECT_TOKEN`:

1. Open **GitHub** → **Settings** → **Developer settings** → **Personal access tokens** → **Tokens (classic)**
2. Click **Generate new token (classic)**
3. Set the **expiration time** (expiry date) for the token. Recommended: choose a short valid duration and rotate it before expiry.
4. Select scopes:
  - **`repo`**
  - **`project`**
5. Copy the token and save it as an **environment secret** named **`PROJECT_TOKEN`** in the `cliq-production` environment.

> Note: Classic PATs must be created from the **Tokens (classic)** page and should always have an expiry date configured. Do not leave an unrestricted long-lived token in place.

Required configuration for thread reply mode:

- Secret: **`PROJECT_TOKEN`**
  - Use a classic PAT with **`repo`** and **`project`** scopes.
- Variable: **`CLIQ_THREAD_STORAGE_MODE=project`**
- Variable: **`GITHUB_PROJECT_OWNER=<owner login>`**
- Variable: **`PROJECT_NUMBER=<project number>`**
- Variable: **`PROJECT_THREAD_FIELD_ID=<field identifier>`**
  - Can be either:
    - numeric field identifier from the Project field settings URL, or
    - GraphQL field node id (`PVTF_*`).

For a normal channel reply, **`PROJECT_NUMBER`** and **`PROJECT_THREAD_FIELD_ID`** are not required.

Only the items below are user-configured values that must be added in GitHub repo variables:

- `CLIQ_THREAD_STORAGE_MODE`
- `GITHUB_PROJECT_OWNER`
- `PROJECT_NUMBER` (only for thread reply mode)
- `PROJECT_THREAD_FIELD_ID` (only for thread reply mode)
- `CLIQ_NOTIFICATION_MODE`
- `CLIQ_BOT_UNIQUE_NAME` (only for bot mode)
- `AI_REVIEW_ENABLED`
- `AI_REVIEW_SERVICE`
- `AI_REVIEW_MODEL`

The secret values are:

- `PROJECT_TOKEN`
- `ENDPOINT`
- `AI_REVIEW_TOKEN` (only if AI review is enabled)

> Important: In this workflow template, these secrets are expected to be created under the GitHub environment named **`cliq-production`**:
> - **Settings → Environments → `cliq-production` → Secrets**
>
> If a user chooses a different environment name, the environment name in the workflow YAML must be updated to match exactly. Otherwise, the secrets will not be resolved.
>
> Also, any value referenced from the workflow YAML as `${{ vars.* }}` must be created in the repository settings under **Settings → Secrets and variables → Actions → Variables**. If it is missing, the workflow will not run correctly.
>
> The required values that must exist in repository variables are:
> - `AI_REVIEW_ENABLED`
> - `AI_REVIEW_SERVICE`
> - `AI_REVIEW_MODEL`
>
> These three are mandatory for the AI review flow and must be set before the workflow is triggered.

The values `github.repository_owner`, `github.repository`, and `github.event.pull_request.number` are GitHub runtime values and do not need to be entered by the user.

### VIII. Setup steps for first-time configuration

1. Go to **Settings** → **Secrets and variables** → **Actions** → **Variables**.
2. Add the required repository variables:
   - `CLIQ_NOTIFICATION_MODE`
   - `CLIQ_BOT_UNIQUE_NAME` (only for bot mode)
   - `CLIQ_THREAD_STORAGE_MODE`
   - `GITHUB_PROJECT_OWNER`
   - `PROJECT_NUMBER` (only for thread reply mode)
   - `PROJECT_THREAD_FIELD_ID` (only for thread reply mode)
   - `AI_REVIEW_ENABLED`
   - `AI_REVIEW_SERVICE`
   - `AI_REVIEW_MODEL`
3. Go to your repository and open **Settings** → **Environments** → `cliq-production` → **Secrets**.
4. Add the required environment secrets:
   - `PROJECT_TOKEN`
   - `ENDPOINT`
   - `AI_REVIEW_TOKEN` (only if AI review is enabled)


### IX. Workflow File set up in target repository

Use this exact workflow file path in the target repository:

- `.github/workflows/CliqConnector.yml`
- Public template repo: https://github.com/Lincy-Zoho/TemplateRepositoryNew

How to add it in GitHub:

1. Open the target repository in GitHub.
2. Click the `Add file` button or create the `.github/workflows` folder if it does not already exist.
3. Create a file named `CliqConnector.yml` inside `.github/workflows/`.
4. Copy the contents from the template repository or from the public template file.
5. Commit and push the workflow file.

The workflow file must be committed to the default branch or to the branch you are using for the PR run. After the workflow runs once, GitHub will create the status check context for the branch protection rule.

The branch rule is not set in the workflow YAML. It is created in GitHub after the workflow has produced the status check.

### X. GitHub branch rule / status check behavior

Important: GitHub branch protection is enforced by GitHub itself. The workflow cannot override a repository rule.

- If you set the rule to **all branches**, GitHub will protect every branch, but the AI review check will not block merges unless the check is explicitly added as a required status check for that branch rule.
- If you want the AI Review Gate to block merges, add **AI Review Gate** as a required status check on the specific protected branch(es) such as `main`, `master`, or `release`.
- If you want AI review only as a PR validation signal and not as a merge blocker, do not mark the check as required.
- The recommendation for this action is to apply protection only to the target release branches and keep AI review as PR-only validation instead of turning it into a full branch-wide enforcement rule.

This is the expected behavior:

- Workflow still runs on PR events and posts the AI review result.
- GitHub only blocks the merge when the status check is required by the branch protection policy.
- If the rule is configured for all branches but the AI Review Gate is not required, the merge is not blocked by that check.

Behavior:

1. Action resolves the PR item in the configured Project V2.
2. Action reads existing thread ID from the configured custom field.
3. If not found, action posts a new Cliq message and captures `message_id`.
4. Action writes captured thread ID to the same project field.
5. Later PR events reuse that saved value and continue in the same Cliq thread.

If project mode is enabled and project field update fails, action logs the failure. Ensure **`PROJECT_TOKEN`**, **project owner/number**, and **field identifier** are correct.

## XI. Custom Event Messages ⚙️

Suppose you need a notification in Cliq for only selected events or actions,
  - you may change the '**_on_**' key of the YAML File where the Action is called.
  - Or additionally, you can set all the required messages and declare the input '**set-message-if-none**' as '**false**' to avoid messages from events you don't want. 
  
You provide default messages to all kind of actions that triggers a workflow.

The messages can be customized by giving the input '**_event_-message**' (where '_event_' is the event for which you would like to customize the message).

For ex: To set a custom message for a Pull Request event, you must define the input as,

```yaml
  pull-request-message: 'A Pull Request has been Opened'
```

## XII. Default Message 📓

If you wish to add a single custom message for all kinds of events, you may use the '**default-message**'. 

For ex: To set a default custom message , you must define the _default-message_ as

```yaml
  default-message: 'A (event) has been (action)'
```

## XIII. Shortcuts ⏩

We also provide several shortcuts to obtain the variables that you want to insert in the message, such as,
  - **(event)**: which will be replaced with the event that the workflow is triggered by
  - **(action)**: which will be replaced by the Action the Event is performing with
  - **(me)**: which will be replaced with the GitHub user performing the action.
  - **(repo)**: which will be replaced by the Repository where the GitHub action is performed
  - **(ref)**: which will be replaced by the Branch/Tag where the GitHub action is performed
  - **(workflow)**: which will be replaced by the workflow on which the GitHub Action is performed
  - **(rule)**: which will be replaced by the Branch Protection Rule (if the Event is Branch Protection Rule)
  - **(run)**: which will be replaced by the Check Run (if the Event is Check Run)
  - **(branch)**: which will be replaced by the Branch/Tag which is Created/Deleted (if the Event is Create / Delete)
  - **(deployment)**: which will be replaced by the deployment (if the Event is Deployment or Deployment Status)
  - **(discussion)**: which will be replaced by the discussion which is worked on
  - **(category)**: which will be replaced by the Category Name to which the discussion is changed to
  - **(issue)**: which will be replaced by the issue (if the Event is Issue or Issue Comment)
  - **(label)**: which will be replaced by the label that is being worked on or added to
  - **(milestone)**: which will be replaced by the milestone that is being worked on or added to
  - **(assignee)**: which will be replaced by the User Assigned to the Issue or Pull Request
  - **(pull)**: which will be replaced by the Pull Request that is worked on
  - **(package)**: which will be replaced the Registry Package that is being worked on
  - **(release)**: which will be replaced by the release that is being worked on
  - **(status)**: which will be replaced by the Status of the Event (if the Event is Deployment Status or Status)

Example:

A GitHub Action is triggered by (me) at (repo).

will change to 

A GitHub Action is triggered by [user_name](https://www.github.com/user_name) at [user_name/repository_name](https://www.github.com/user_name/repository_name).

Upon successfully providing the inputs as per criteria, the message will be successfully sent to the Cliq Channel.

The GitHub events that trigger a workflow are listed below, among which all events are supported by GitHub Informer

|    branch_protection_rule    |          check_run          |          check_suite         |            create            |           delete            |
|            :----:            |           :----:            |            :----:            |            :----:            |           :----:            |
| **deployment**               | **deployment_status**       | **discussion**               | **discussion_comment**       | **fork**                    |
| **gollum**                   | **issue_comment**           | **issues**                   | **label**                    | **milestone**               |
| **page_build**               | **public**                  | **pull_request**             | **pull_request_comment**     | **pull_request_review**     |
|**pull_request_review_comment**| **pull_request_target**    | **push**                     | **registry_package**         | **release**                 |
| **repository_dispatch**     | **schedule**                 | **status**                   | **watch**                    | **workflow_dispatch**       |

## XIV. Base YAML Code 🗒

Don't worry about remembering a lot of stuff. Here is the minimal code that's required to start with. 

```yaml
name : Communicating with Cliq
on:
  #you may add the events you like to get notified
  push:
    
jobs:
  test_name:
    runs-on: ubuntu-latest
    steps:
      - uses: Integrations-dev/GitHub-Informer@v1
        with:
          channel-endpoint: ${{ secrets.ENDPOINT }}
```

That's all! You will start getting notified for each event occurring in GitHub through the GitHub Action.

Go to the Actions tab of the repository to view the message status.

## XV. AI Review Gate (OpenAI, Claude, Gemini)

You can enable an AI review gate for pull requests. The action can run against OpenAI, Claude, or Gemini.

For exact PR comment formatting and one-issue-per-comment rules, see [docs/AI-Review-Comment-Construction.md](docs/AI-Review-Comment-Construction.md).

- If the AI decision is pass, the check passes and no PR comment is added.
- If the AI decision is fail (or the AI call fails), the check fails, a PR comment is added, and a failure message is posted to the Cliq PR thread.

### XVI. Required GitHub Workflow Permissions

Your workflow must include:

```yaml
permissions:
  contents: read
  checks: write
  issues: write
  pull-requests: write
```

### XVII. Recommended Workflow Usage

```yaml
name: PR AI Review Gate

on:
  pull_request:
    types: [opened, reopened, synchronize, labeled]

permissions:
  contents: read
  checks: write
  issues: write
  pull-requests: write

jobs:
  ai-review:
    runs-on: ubuntu-latest
    steps:
      - uses: Integrations-dev/GitHub-Informer@v1
        with:
          channel-endpoint: ${{ secrets.CLIQ_ENDPOINT }}
          ai-review-enabled: true
          ai-review-trigger: auto
          ai-review-on-sync: true
          ai-review-label: ai-review
          ai-review-check-name: AI Review Gate
          ai-review-token: ${{ secrets.AI_REVIEW_TOKEN }}
          ai-review-api-url: https://api.openai.com/v1/chat/completions
          ai-review-model: gpt-4.1-mini
```

### XVIII. Provider Configuration

Only the token should be stored as a secret. Endpoint and model can be plain workflow values.

OpenAI

```yaml
ai-review-token: ${{ secrets.AI_REVIEW_TOKEN }}   # token starts with sk-
ai-review-api-url: https://api.openai.com/v1/chat/completions
ai-review-model: gpt-4.1-mini
```

Claude

```yaml
ai-review-token: ${{ secrets.AI_REVIEW_TOKEN }}   # token starts with sk-ant-
ai-review-api-url: https://api.anthropic.com/v1/messages
ai-review-model: claude-3-5-sonnet-latest
```

Gemini

```yaml
ai-review-token: ${{ secrets.AI_REVIEW_TOKEN }}   # token starts with AIza
ai-review-api-url: https://generativelanguage.googleapis.com/v1beta/models/gemini-1.5-pro:generateContent
ai-review-model: gemini-1.5-pro
```

### XIX. Branch Protection

This is required for enforcement. The AI Review Gate check must be added as a **required status check** in GitHub repository Rules / Branch Protection.

This is not a workflow variable and not something to add in the YAML. It is a GitHub repository setting that blocks merge until the check passes.

To enforce mentor approval policy, add the check name from **`ai-review-check-name`** (default: **`AI Review Gate`**) as a **required status check** in branch protection.

### XX. Configure Required Status Check (Step-by-step)

1. In workflow, set the check name you want:

```yaml
ai-review-check-name: AI Review Gate
```

2. Run the workflow once on a PR so GitHub can see this check context.

3. In GitHub repository settings, open `Rules` (or `Branches` -> branch protection rule).

4. Enable `Require status checks to pass`.

5. Add the same check name exactly (example: `AI Review Gate`) under required checks.

Important:

- **If you rename `ai-review-check-name`, update the required check name in rules to the same value.**
- **Name matching is exact.** Any mismatch causes **`Waiting for status to be reported`**.
- After changing rules/check name, **push a new commit or re-run checks once** so the new context is picked.

Here is a full template repository for the GitHub Informer which you can use as a baseline to work with and customize to your usage.

- Public template repo: https://github.com/Lincy-Zoho/TemplateRepositoryNew
- Older reference repo: https://www.github.com/Integrations-dev/RepositoryTemplate
