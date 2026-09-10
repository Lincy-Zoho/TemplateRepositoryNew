# GitHub Informer for Zoho Cliq — Setup Guide

---

## 1. Purpose of the workflow

The GitHub Informer workflow connects your GitHub repository to a Zoho Cliq channel. Once it is configured, important repository activity is shared in Cliq automatically, so your team does not need to keep checking GitHub manually.

It can do three things:

1. Post GitHub events to a Cliq channel. Pushes, pull requests, issues, releases, deployments, and many other events can be sent to the channel either individually or through a shared default message.
2. Keep pull request updates in one Cliq thread (optional). Instead of creating separate messages for every PR update, the workflow can continue the same conversation thread for that pull request. This requires saving the thread message ID between workflow runs, and a GitHub Project V2 field is used for that.
3. Run an AI review check on pull requests (optional). The action sends the PR diff to OpenAI, Claude, or Gemini and reports the result as a GitHub status check. If this check is marked as required, a failed review can block the merge.

The threading and AI review features are optional and independent. You can use notifications alone, notifications with threading, notifications with AI review, or all three together.

---

## 2. Configuration flow

To configure this workflow correctly, follow these three steps in order:

#### Step 1: Set the environment secrets and repository variables
#### Step 2: Create the workflow and run it
#### Step 3: Set branch rules and required status checks, then validate with a PR


## 3. How to configure environment secrets

Go to your repository or organization and configure the required values.

If no environment exists, create a new environment named `cliq-production` first. Then open:

#### GitHub → Settings → Environments → `cliq-production` → Secrets

Add the required secrets under this environment.

### 3.1 ENDPOINT - The channel endpoint URL

All notifications are sent to a single URL: the channel endpoint. Its format is:

`<region-base>/api/v2/channelsbyname/<CHANNEL_UNIQUE_NAME>/message?zapikey=<WEBHOOK_TOKEN>`

Three parts need to be filled in:

#### 3.1.a. Region base

This must match the data centre where your Cliq organisation is hosted. The wrong domain fails quietly and no message is posted.

| Region | Base URL |
| --- | --- |
| US | `https://cliq.zoho.com` |
| IN | `https://cliq.zoho.in` |
| EU | `https://cliq.zoho.eu` |
| AU | `https://cliq.zoho.com.au` |
| JP | `https://cliq.zoho.jp` |

If you are unsure, check the address bar in your browser while using Cliq.

#### 3.1.b. Channel unique name

This is not the display name. In Cliq, open the channel and go to **Channel Actions** or **Info**. The unique name is shown there.

Look for the value labeled **Unique Name** in the channel details panel. This is the exact value used in the URL after `/channelsbyname/`.

`Example Unique Name: githubreponotification`

**So the endpoint becomes:**

`https://cliq.zoho.com/api/v2/channelsbyname/githubreponotification/message?zapikey=1001.xxxxxxxx`

The part after `/channelsbyname/` must match the channel's unique name exactly.

#### 3.1.c. Webhook token

This is generated from your Zoho Cliq profile and must be created before you assemble the channel endpoint URL.

**Follow these steps:**

1. Click on your **profile picture** located in the top-right corner of the screen.
2. Select **Bots & Tools** from the dropdown menu.
3. Look at the left sidebar menu under the *Integrations* section and click on **Webhook Tokens**.
4. Complete the two-factor authentication (2FA) identity verification if prompted by the platform.
5. Click the **Generate New Token** button.
6. Give your token a recognizable name, then click create to view your unique hex string token.

After the token is created, copy the generated webhook token and combine it with the region base and channel unique name to build the final endpoint.

**Assembled example:**

`https://cliq.zoho.in/api/v2/channelsbyname/githubreponotification/message?zapikey=1001.xxxxxxxx`

This complete string, including the `?zapikey=` part, is what you store as the `ENDPOINT` secret.



### Check Once

| Variable Type | Name | Allowed Value |
| --- | --- | --- |
| Environment Secrets | `ENDPOINT` | `https://cliq.zoho.in/api/v2/channelsbyname/githubreponotification/message?zapikey=1001.xxxxxxxx` |


> Treat this value as a credential. Anyone who has it can post to the channel.


### 3.2 Creating the classic PAT for `PROJECT_TOKEN`

1. Go to **GitHub → Settings → Developer settings → Personal access tokens → Tokens (classic)**. [Link](https://github.com/settings/tokens)
2. Click **Generate new token (classic)**.
3. Set an **expiry date**. Choose a short duration and plan to rotate it before expiry. Do not create a non-expiring token.
4. Select the scopes: **`repo`** and **`project`**.
5. Copy the token and save it as the `PROJECT_TOKEN` environment secret.


### Check Once

| Variable Type | Name | Allowed Value |
| --- | --- | --- |
| Environment Secrets | `PROJECT_TOKEN` | `XXX_S7Up0fXXXXXXXX` |


> Treat this value as a credential. Anyone who has it can access your project.


### 3.3 AI_REVIEW_TOKEN for PR AI Review Gate

The workflow uses this token to authenticate the AI API request. Without it, the action cannot fetch the model output, cannot create the review result, and cannot post the status or PR comment.

#### 3.3.a Generate the API token

| Service | Token generation URL |
| --- | --- |
| OpenAI | https://platform.openai.com/api-keys |
| Claude | https://console.anthropic.com/settings/keys |
| Gemini | https://aistudio.google.com/app/apikey |


### Check Once

| Variable Type | Name | Allowed Value |
| --- | --- | --- |
| Environment Secrets | `AI_REVIEW_TOKEN` | `sk- / sk-ant / AIza` |


> Treat this value as a credential. Anyone who has it can access your AI service.



## Environment secrets check

| Secret name | Required | Allowed value |
| --- | --- | --- |
| `ENDPOINT` | Yes | `https://cliq.zoho.in/api/v2/channelsbyname/githubreponotification/message?zapikey=1001.xxxxxxxx` |
| `PROJECT_TOKEN` | Only if `CLIQ_THREAD_STORAGE_MODE=project` | `XXX_S7Up0fXXXXXXXX` |
| `AI_REVIEW_TOKEN` | Only if `AI_REVIEW_ENABLED=true` | Provider API key for the selected AI service. |



## 3. How to configure environment variables

 **Where these go:** repository **Settings → Secrets and variables → Actions → Variables**.

These are repository variables, not secrets, and they are not environment-scoped.

### 3.1 Decide: post as a user or as a bot?

Choose the mode based on who should appear as the sender of the notification:

| Mode | Who the message appears to come from | Extra setup |
| --- | --- | --- |
| User (webhook) mode | The person who created the webhook | No bot setup needed |
| Bot mode | A dedicated Cliq bot | Create the bot and add it to the channel |

Use this setting in GitHub repository variables:

### Check Once

| Variable Type | Variable Name | Allowed Values|
| --- | --- | --- |
| Repository Variables | `CLIQ_NOTIFICATION_MODE` | `user / bot` |


Use user mode when you want the fastest setup and do not mind the message appearing from the webhook creator. Use bot mode when you want a stable, shared sender for ongoing notifications, especially if the person creating the webhook leaves the team or the channel.

Both modes still use the same `ENDPOINT` secret. The only extra value needed in bot mode is `CLIQ_BOT_UNIQUE_NAME`.

### 3.2 If posting as a user

In user mode, there is no extra configuration beyond the channel endpoint itself.

The cliq channel message posted as user authentication with custom bot name and the custom bot thumbnail defaultly it was set in the yml file.

### 2.4 If posting as a bot — getting the bot unique name

1. Open Zoho Cliq.
2. Click on your profile picture in the top-right corner.
3. Select **Bots & Tools**.
4. In the left sidebar, open **Integrations** and click **Bots**.
5. Select the bot you created or want to use.
6. While creating the bot, enable the required channel permissions so the bot can post messages in the target channel.
7. Open the bot details panel and look for the **API Endpoint** value.
8. The value after `/bots/` is the bot unique name.

Example:

`https://cliq.zoho.com/api/v2/bots/githubnotificationbot/message`

The bot unique name is:

`githubnotificationbot`

This is not necessarily the display name. It is the unique identifier that must be added in `CLIQ_BOT_UNIQUE_NAME`.

9. Add the bot to the target channel. This is the most common bot-mode failure.
10. Set the variables:

### Check Once

| Variable Type | Variable Name | Allowed Values|
| --- | --- | --- |
| Repository Variables | `CLIQ_BOT_UNIQUE_NAME` | `githubnotificationbot` |
