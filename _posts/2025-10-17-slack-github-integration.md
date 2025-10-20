---
title: "Integrate Slack with Github"
date: 2025-10-17
---

This article explains how to create a Slack Bot and configure a simple Github Action workflow using `slackapi/slack-github-action`, which is one of the many ways to implement this integration.

## Create a Slack Bot

### Step 1: Go to Slack API Apps

*URL*: https://api.slack.com/apps

Click "Create New App" and then "From Scratch"
Give it a name (e.g. GitHub PR Bot) and choose your workspace below.


### Step 2: Add Permissions

- In the left sidebar, go to OAuth & Permissions.
- Under Scopes / Bot Token Scopes, click Add an OAuth Scope.
- Add this permission: `chat:write` (you can add others as well)


### Step 3: Install the app to your workspace

- Scroll up and click the "Install to Workspace" button.
- Slack will ask you to authorize the app — approve it.

### Step 4: Copy the bot token

- After installation, you’ll see a new section: OAuth Tokens for Your Workspace. There will be a token labeled: `Bot User OAuth Token`. Copy it, it will start with `xoxb-`.

### Step 5: Add it to GitHub Secrets

In your GitHub repository:

- Go to Settings / Secrets and variables / Actions
- Click New repository secret
- Name it: `SLACK_BOT_TOKEN` for example
- Paste the token you copied from Slack

## Github workflow

- Create a folder `.github/workflows` in your repo.
- Add a yml file with the workflow, e.g. `pr-alert.yml`. Note that this uses `@v2` of the action, which uses different payload format from `@v1`.

```yml
name: slack-notification

on:
  pull_request:
    types: [opened, ready_for_review]
    paths:
      [
        "src/**",
        ".github/workflows/pr-alert.yml",
      ]

jobs:
  slack-notifications:
    name: Sends a message to Slack when a pull request is opened
    runs-on: ubuntu-latest
    if: ${{ !github.event.pull_request.draft }}
    steps:
      - name: Post to Slack (v2)
        uses: slackapi/slack-github-action@v2
        with:
          method: chat.postMessage
          token: ${{ secrets.SLACK_BOT_TOKEN }}
          payload: |
            channel: "<channel_id>"
            text: "*New PR waiting for review:* <${{ github.event.pull_request.html_url }}|${{ github.event.pull_request.title }}>"
            blocks:
              - type: section
                text:
                  type: mrkdwn
                  text: "*New PR waiting for review:*\n<${{ github.event.pull_request.html_url }}|${{ github.event.pull_request.title }}>"
              - type: section
                fields:
                  - type: mrkdwn
                    text: "*Opened by:*\n${{ github.event.pull_request.user.login }}"

```

- Replace `<channel_id>` with the id of the Slack channel where you would like to post your messages. By the time of writing, you can find this id by clicking on the `# channel-name` in the header, a popup will open, and at the bottom there is `Channel ID: C19FYW2BHDF` for example.

> IMPORTANT: Don't forget to invite the bot in the channel, otherwise you'll get an error.

> IMPORTANT: It goes without saying, but don't paste you API key inside the yml file, use the `$\{{ secrets.SLACK_BOT_TOKEN }}` variable, as described above!

- Use whatever template you like, this is just an example that worked well for us.