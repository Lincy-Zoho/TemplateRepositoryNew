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

### Step 1: Set the environment secrets and repository variables
### Step 2: Create the workflow and run it
### Step 3: Set branch rules and required status checks, then validate with a PR


## 3. How to configure Environment secrets

Go to your repository/organization and configure the required values.

Open: GitHub → Settings → Environments → `cliq-production` → Secrets

#### 3.1 ENDPOINT - The channel endpoint URL

All notifications are sent to a single URL: the channel endpoint. Its format is:

```text
<region-base>/api/v2/channelsbyname/<CHANNEL_UNIQUE_NAME>/message?zapikey=<WEBHOOK_TOKEN>
```

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

This is not the display name. In Cliq, open the channel and go to Channel Actions or Info. The unique name is shown there.

Look for the value labeled **Unique Name** in the channel details panel. This is the exact value used in the URL after `/channelsbyname/`.

```text
Example Unique Name: githubreponotification
```

So the endpoint becomes:

```text
https://cliq.zoho.com/api/v2/channelsbyname/githubreponotification/message?zapikey=1001.xxxxxxxx
```

The part after `/channelsbyname/` must match the channel's unique name exactly.

#### 3.1.c. Webhook token

This is generated from your Zoho Cliq profile and must be created before you assemble the channel endpoint URL.

Follow these steps:

1. Click on your **profile picture** located in the top-right corner of the screen.
2. Select **Bots & Tools** from the dropdown menu.
3. Look at the left sidebar menu under the *Integrations* section and click on **Webhook Tokens**.
4. Complete the two-factor authentication (2FA) identity verification if prompted by the platform.
5. Click the **Generate New Token** button.
6. Give your token a recognizable name, then click create to view your unique hex string token.

After the token is created, copy the generated webhook token and combine it with the region base and channel unique name to build the final endpoint.

Assembled example:

```text
https://cliq.zoho.in/api/v2/channelsbyname/githubreponotification/message?zapikey=1001.xxxxxxxx
```

This complete string, including the `?zapikey=` part, is what you store as the `ENDPOINT` secret.

> Treat this value as a credential. Anyone who has it can post to the channel.

| Variable Type | Name | Allowed Value |
| --- | --- | --- |
| Environment Secrets | `ENDPOINT` | `https://cliq.zoho.in/api/v2/channelsbyname/githubreponotification/message?zapikey=1001.xxxxxxxx` |
