# Honeypot
## Table of Contents
1. [Introduction](#1-introduction)
2. [Usage](#2-usage)
    1. [Honeypot Management](#21-honeypot-management)
    2. [Logs Management](#22-logs-management)
    3. [Permissions Management](#23-permissions-management)
3. [Hosting the bot](#3-hosting-the-bot)
     1. [Creating a Discord Application](#31-creating-a-discord-application-on-the-discord-developers-portal)
    2. [Picking a hosting provider](#32-picking-a-hosting-provider)
    3. [Installing the bot](#33-installing-the-bot)
    4. [Persistence with Systemd](#34-making-the-bot-persistent-with-systemd)

## 1. Introduction
This bot was made to counter MrBeast scams and other spam messages. It does this by registering a specific channel as a honeypot. Sending any message in that channel will then trigger whatever punishment was set by the creator. The punishment types are mute, kick and ban. Along with executing the aforementioned punishment, the bot will also delete all messages sent by the person who triggered from the last 10 seconds. This is to make sure that any spam messages that were successfully sent before the punishment was executed, are gone and can no longer harm your community. The default length for the mute and ban punishment is seven days. A member will also receive a DM saying they triggered a honeypot.

This bot is mostly meant to target compromised user accounts. This is why it's recommended to set the punishment type to mute as not to ban a member who clicked on the wrong link. This means that the honeypot channel must be a public channel. A few recommendations to go with this:
- Name the channel something inconspicuous. This serves two purposes: members are less likely to accidentally chat in the channel, and it helps stay ahead of attackers using LLMs to scan for honeypots.
- Make it known that this channel is not to be chatted in. I'd highly recommend using an image with text for this, again to confuse AI.
- Make sure members are aware of the punishment of talking in a honeypot channel. This further discourages them from triggering it.
## 2. Usage
### 2.1. Honeypot management
Creating a honeypot channel:
```
/honeypot create [channel] [punishment_type] [punishment_duration]
```
Editing a honeypot channel:
```
/honeypot edit [channel] [punishment_type] [punishment_duration]
```
Removing a honeypot channel:
```
/honeypot remove [channel]
```
*Note: this will only remove the honeypot association in the database. The actual Discord channel object will stay*

List all honeypots:
```
/honeypot list
```
### 2.2. Logs management
Setting up logs:
```
/logs setup [channel] [ping_role]
```
View the current log setup:
```
/logs info
```
*Note: ping_role refers to the Discord role that will be pinged whenever a honeypot is triggered and logged.*
### 2.3. Permissions management
Setting which role can manage honeypots:
```
/perms setup [role]
```
View the current permissions setup:
```
/perms info
```

## 3. Hosting the bot
Refer to this quick guide on how to host this Discord bot. (Parts of) this guide may also be used to host other Discord bots, so consider it a free tutorial.
### 3.1. Creating a Discord application on the Discord Developers portal
First, head to the [Discord Developers Portal](https://discord.com/developers/home). After signing in, go to *Applications* and click *New Application*. Give your bot a name, e.g.: *Honeypot Bot*, and create the bot. Now go to the *Bot* tab and enable the *Message Content Intent*. This is to make sure the bot can read and interact with messages. After this, go to the *OAuth2* tab, and head to the *OAuth2 URL Generator*. Click on *bot*. This will open a new panel called *Bot Permissions*. From here, select the following permissions:
- Kick Members
- Ban Members
- Moderate Members
- Send Messages
- Manage Messages
- Embed Links
- Use Slash Commands

Lastly, copy the generated URL in your browser and add the bot to your Discord server.

*Note: technically you could just use the Administrator permission, but this is HIGHLY DISCOURAGED as it would mean that if this bot gets compromised, the attacker can do anything they want.*
### 3.2. Picking a hosting provider
The easiest way to host this bot would be using a dedicated hosting provider, such as *Bot-Hosting.net*, *Wispbyte*, etc. This can however be expensive. That's why this part will show you how to set it up using a VPS. I highly recommend *Oracle Cloud*, as they have a generous free tier that will allow you to run this bot without spending a dime. The rest of this tutorial assumes you have shell access to a Debian-based Linux environment. 

*Note: If you use a different distro, only the `apt` commands should change. Use `dnf` for RHEL-based systems, `pacman` for Arch-based systems, etc.*
### 3.3. Installing the bot
To get the actual source code of this bot onto your Linux server, you must first ensure you have Git installed.
```bash
# Updating the package mirrors and installing Git
sudo apt update && sudo apt install git -y
```
When Git is successfully installed, simply clone this repository:
```bash
# Clone (download) the repository
git clone https://github.com/matisse-priem/honeypot-discord-bot
```
Git will automatically make a new directory called *honeypot-discord-bot* in the directory you ran the command from. Now enter this new directory.
```bash
# Enter the new directory
cd honeypot-discord-bot
```
Next we need to set up the Python environment. To do this, a virtual environment is recommended. The reason for this is to make sure that this bot has its own contained package list and doesn't need to share packages with other projects. If there would be a conflict between one project requiring version X of a package and another requiring version Y, this would solve that issue.
```bash
# Ensure Python dependencies are present
sudo apt install python3-venv python3-pip -y

# Create the virtual environment
python3 -m venv .venv
```
Now we must activate the virtual environment and install the required packages.
```bash
# Activate the virtual environment
source .venv/bin/activate

# Optionally update pip
pip install --upgrade pip

# Install the packages
pip install -r requirements.txt
```
Next up is arguably the most important part. It's time to link your Discord bot with this code. To do this, go back to your Discord Developers portal, go to *Bot*, and click *Reset Token*. If you have MFA enabled, you will need to verify because this is a risky thing to do if the bot were already deployed. Now just copy the bot token.
```bash
# Link the bot with the code
# Make sure to replace [YOUR-DISCORD-TOKEN-HERE] with the token you find on the Discord Developer portal
echo "DISCORD_TOKEN=[YOUR-DISCORD-TOKEN-HERE]" > .env

# Restrict access to this file as it contains sensitive info
chmod 600 .env
```
*Side note: if you ever push this code to your own repo or fork it, remember to add `.env` to your `.gitignore` file in order to prevent token leakage.*

After this, the bot is technically ready to be launched. Make sure the setup runs by starting it:
```bash
python3 main.py
```
### 3.4. Making the bot persistent with Systemd
If everything works, stop the bot using `CTRL + C`. While you could run the bot like this, this isn't ideal for various reasons. The main reason is that if the server were to crash and restart, the bot stops, and you'd have to manually start it again. To solve this, we'll make a custom service. This guide assumes your VPS uses Systemd.
```bash
# Create the Systemd service
# Run this FROM the honeypot-discord-bot directory, so $(pwd) is correct
BOT_DIR="$(pwd)"
BOT_USER="$(whoami)"
echo "
[Unit]
Description=Honeypot Discord Bot, Docs at https://github.com/matisse-priem/honeypot-discord-bot

[Service]
User=${BOT_USER}
ExecStart=${BOT_DIR}/.venv/bin/python ${BOT_DIR}/main.py
WorkingDirectory=${BOT_DIR}/
Restart=on-failure

[Install]
WantedBy=multi-user.target
" | sudo tee /etc/systemd/system/honeypot.service

# Start and enable the service so it starts on boot
sudo systemctl enable --now honeypot.service
```
Now just run a status check to see if everything works, and you're done!
```bash
sudo systemctl status honeypot
```
*Final note: if you're using a distro that doesn't use Systemd, like Alpine, it isn't too hard to adapt the service. Just make sure it doesn't run as root (it doesn't need any root privileges) and runs from the correct directory with the venv's python binary.*

---

## License

This project is licensed under the [MIT License](LICENSE).

## Disclaimer

This bot is provided as-is, without warranty of any kind. The authors and
contributors are not responsible for any moderation actions taken by or
misconfiguration of this bot. You are responsible for complying with the
[Discord Terms of Service](https://discord.com/terms) and
[Developer Terms](https://discord.com/developers/docs/policies-and-agreements)
when using it.

## Credits

Created by [@matisse-priem](https://github.com/matisse-priem).
Contributions are welcome — feel free to open issues and pull requests!