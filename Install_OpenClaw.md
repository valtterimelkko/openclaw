# Install OpenClaw

Favorite: No
Archived: No
Created: 2 February 2026 05:37
Updated: 2 February 2026 22:34
Project: Serveri (https://www.notion.so/Serveri-2bf45010ad5d80068a3ecdbbad8ef1c9?pvs=21)

I want to install openclawd here in this project folder.

Quick start

curl -fsSL https://openclaw.ai/install.sh | bash

OR

npm i -g openclaw AND openclaw onboard

OR

curl -fsSL https://openclaw.ai/install.sh | bash -s -- --install-method git

But BEFORE installation, look through this page: 

https://docs.openclaw.ai/start/getting-started

Github repo (also if the instructions of direct installation weren’t to work, you can pull from Github): https://github.com/openclaw/openclaw

The models I want to configure with it:

Primary model: GLM-4.7 (use via GLM API, not via Openrouter)

First fallback: Mistral Devstral 2 (via openrouter)

Second fallback: Google Gemini 2.5 Flash (via openrouter)

Third fallback: Meta Llama 3.3 70B Instruct (via openrouter)

Endpoints & model IDs:

GLM: https://api.z.ai/api/coding/paas/v4

Model ID GLM-4.7

Openrouter: https://openrouter.ai/api/v1/chat/completions

Model IDs: 

mistralai/devstral-2512

google/gemini-2.5-flash

meta-llama/llama-3.3-70b-instruct:free

All my API keys are set in the vault (the keys come from a secure vault) - but we have to set it up first (below)

I want to set up openclawd to work with telegram.

Here’s full instruction of the security hardening I want in place (the user for openclaw is already set up, and you’re using it as speak): OpenClaw-Security_Hardening.md.

DO NOTE: This file is made with Perplexity. You need to critically evaluate every command and every piece of code / setting and check. I want you to read the file in order to understand the security hardening I want in place - and following the security principles detailed in that file - but Perplexity could hallucinate every command, every syntax - and you NEED to check using your web search tool before using them - if necessary (you might already know the correct syntaxes without checking).

After setting up the vault, you need to pause - and tell me how I insert my API keys there - and Telegram bot key as well as my own user ID for only allowing me to chat with it. After I’m done with that, we can continue.

At the point 7 of the security hardening document, you need to guide me whether I need to set it up from the root user? And pause at that point for me to do it - and give me the commands for it.

At the end when installation and setup is fully complete and security hardening in place, I want you to make sure there’s a git system in place for this folder, an appropriate and security-focussed gitignore in place, make sure that you’ve done our first commit in the this folder - and that we’re syncing to the newly forked repo I’ve established at [https://github.com/valtterimelkko/openclaw](https://github.com/valtterimelkko/openclaw).