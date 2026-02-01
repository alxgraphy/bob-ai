# Bob: ZimaBoard AI Commander

A comprehensive guide to building "Bob" - an AI-powered server administration assistant running on a ZimaBoard using n8n, Telegram, and multiple AI models.

## Overview

Bob is a multi-AI assistant that can:
- Execute SSH commands on your ZimaBoard
- Read and send emails via Gmail
- Respond to commands via Telegram from anywhere
- Use multiple AI models (Groq, Mistral) for different tasks
- Auto-backup workflows to GitHub

## Architecture

```
Telegram → n8n (ZimaBoard) → Cloudflare Tunnel → Public Access
                ↓
         AI Agent (Groq - Fast Manager)
                ↓
         Tools:
         - SSH (Server Commands)
         - Gmail (Email Management)
         - Mistral AI (Specialist Tasks)
```

## Prerequisites

### Hardware
- ZimaBoard (832 or similar)
- Stable internet connection
- At least 8GB RAM recommended

### Accounts Required
1. **Telegram Account** - For bot interface
2. **Domain Name** - For Cloudflare tunnel (can use DuckDNS for free)
3. **Cloudflare Account** - For secure tunnel
4. **Gmail Account** - For email functionality
5. **Groq Account** - Free AI API
6. **Mistral Account** (optional) - Additional AI model
7. **GitHub Account** - For workflow backups

## Installation

### 1. Install n8n on ZimaBoard

```bash
# Install Docker (if not already installed)
sudo apt update && sudo apt install docker.io -y
sudo systemctl enable --now docker

# Add user to docker group
sudo usermod -aG docker $USER
newgrp docker

# Create n8n data directory
mkdir -p ~/n8n_data
sudo chmod -R 777 ~/n8n_data

# Run n8n
docker run -d \
  --name n8n \
  -p 5678:5678 \
  -v ~/n8n_data:/home/node/.n8n \
  -e N8N_SECURE_COOKIE=false \
  --restart always \
  n8nio/n8n
```

### 2. Setup Cloudflare Tunnel

```bash
# Run Cloudflare tunnel
docker run -d --name cloudflared \
  --restart always \
  cloudflare/cloudflared:latest \
  tunnel --no-autoupdate run --token YOUR_CLOUDFLARE_TOKEN
```

**Cloudflare Dashboard Steps:**
1. Go to Zero Trust → Networks → Tunnels
2. Create a new tunnel named "ZimaBoard_n8n"
3. Copy the token
4. Add public hostname:
   - Subdomain: `n8n` (or your choice)
   - Domain: `yourdomain.com`
   - Service: `http://192.168.0.185:5678`

### 3. Update n8n for HTTPS

```bash
docker rm -f n8n

docker run -d \
  --name n8n \
  -p 5678:5678 \
  -v ~/n8n_data:/home/node/.n8n \
  -e N8N_HOST="n8n.yourdomain.com" \
  -e N8N_PROTOCOL="https" \
  -e WEBHOOK_URL="https://n8n.yourdomain.com/" \
  --restart always \
  n8nio/n8n
```

## Configuration

### 1. Telegram Bot Setup

1. Message [@BotFather](https://t.me/botfather) on Telegram
2. Send `/newbot`
3. Follow prompts to create your bot
4. Save the API token
5. Find your Telegram ID using [@userinfobot](https://t.me/userinfobot)

**n8n Configuration:**
- Add Telegram Trigger node
- Set credential with your bot token
- Add filter to restrict to your User ID only

### 2. Gmail Setup (Two Options)

#### Option A: App Password (Recommended for simplicity)

1. Enable 2-Factor Authentication on Gmail
2. Go to Google Account Settings → Security
3. Search for "App Passwords"
4. Create password named "n8n Bob"
5. Use credentials in n8n:
   - **SMTP** (for sending):
     - Host: `smtp.gmail.com`
     - Port: `465`
     - SSL/TLS: ON
   - **IMAP** (for reading):
     - Host: `imap.gmail.com`
     - Port: `993`
     - SSL/TLS: ON

#### Option B: OAuth2 (More secure but complex)

1. Create project in [Google Cloud Console](https://console.cloud.google.com)
2. Enable Gmail API
3. Create OAuth 2.0 credentials
4. Add yourself as test user
5. Use OAuth redirect URL from n8n

**Important:** Make sure the redirect URI matches exactly:
```
https://n8n.yourdomain.com/rest/oauth2-credential/callback
```

### 3. AI Models Setup

#### Groq (Free & Fast)
1. Visit [console.groq.com](https://console.groq.com)
2. Create API key
3. In n8n, add Groq Chat Model
4. Recommended model: `llama-3.3-70b-versatile`

#### Mistral (Optional Specialist)
1. Visit [Mistral Console](https://console.mistral.ai)
2. Create API key
3. Add Mistral Cloud Chat Model
4. Use for complex tasks

### 4. SSH Access

**ZimaBoard Credentials:**
```yaml
Host: 192.168.0.185 (or your ZimaBoard IP)
Port: 22
User: alx (or your username)
Auth: Password or SSH key
```

## Workflow Structure

### Main Components

1. **Telegram Trigger** - Listens for messages
2. **Security Filter** - Ensures only you can access
3. **AI Agent (Groq)** - Main brain, fast responses
4. **Simple Memory** - Remembers conversation context
5. **Tools:**
   - Execute SSH
   - Send Email (Gmail)
   - Read Email (Gmail)
   - Mistral Specialist (AI Agent Tool)

### AI Agent Configuration

#### Main Agent (Groq) System Prompt

```
You are "Bob," a witty and grounded AI server commander living on a ZimaBoard.

CAPABILITIES:
- Server access via SSH (check uptime, memory, docker, temperatures)
- Email reading and writing via Gmail
- Access to Mistral specialist for complex tasks

SAFETY PROTOCOLS:
1. Be concise and professional
2. CRITICAL: For destructive commands (rm, sudo, reboot, docker stop), 
   ask "Are you sure, Commander?" and wait for "Yes"
3. NEVER execute "rm -rf /" under any circumstances
4. Use tools appropriately based on user request
5. Delegate complex writing/analysis to Mistral specialist

TELEGRAM FORMATTING:
- Use Markdown for all responses
- Wrap commands in `backticks`
- Wrap output in ```triple backticks```
- Use ✅ for success, ⚠️ for warnings

CONTEXT AWARENESS:
- User is "alx" on ZimaBoard at 192.168.0.185
- Always ask which server if not specified
- Verify file paths before operations
```

#### Mistral Specialist System Prompt

```
You are the "Mistral Specialist," a high-level assistant for the ZimaBoard Commander.

OPERATIONAL RULES:
- Tone: Professional, technical, precise - no fluff
- Email Task: Clear subject lines, professional formatting
- Linux Task: Provide exact, safe bash commands for ZimaBoard
- Feedback: Your response goes to the Commander (Groq) for relay to user
- Output: Make answers "ready to send" - polished and complete

CONSTRAINTS:
- Do not mention you are an AI
- Provide finished products only
- No conversational filler
```

#### AI Agent Tool Description (for Mistral)

```
Use this tool for complex tasks including:
1) Professional email drafting and proofreading
2) Detailed Linux server log analysis or bash script generation
3) Complex reasoning requiring formal tone
Input should be the user's full request.
```

### Tool Configurations

#### SSH Tool
```yaml
Node: Execute SSH
Command: {{ $fromAI("command", "The full linux command to run") }}
Credentials: SSH Password (alx@192.168.0.185)
Description: "Use this to execute Linux commands. Required input: 'command'"
```

#### Gmail Send Tool
```yaml
To: {{ $fromAI("to", "Recipient email address") }}
Subject: {{ $fromAI("subject", "Email subject line") }}
Message: {{ $fromAI("message", "Email body content") }}
Email Type: Plain Text
```

#### Gmail Read Tool
```yaml
Operation: Get Many
Limit: 5-10
Filters: Unread Only (optional)
Description: "Lists recent emails. Use when user asks about new mail."
```

## Security Best Practices

### 1. User Restriction
Always add a Switch/Filter node after Telegram Trigger:

```
Condition: {{ $json.message.from.id }} equals YOUR_TELEGRAM_ID
True path: → Continue to AI Agent
False path: → Send "Access Denied" message
```

### 2. Command Safety
Bob's safety protocols prevent:
- `rm -rf /` - Filesystem destruction
- Unconfirmed destructive operations
- Running commands without context verification

### 3. Encryption Key Backup
**CRITICAL:** Save your n8n encryption key:

```bash
docker exec -it n8n printenv N8N_ENCRYPTION_KEY
```

Store this in a password manager. Without it, you cannot restore credentials from GitHub backups.

## GitHub Backup Workflow

### Auto-Backup Setup

Create a workflow with these nodes:

1. **Schedule Trigger** - Daily at 3:00 AM
2. **n8n Node** - Get all workflows
3. **Split Out Node** - Process each workflow
4. **n8n Node** - Get workflow details
5. **GitHub Node** - Create/update file
   ```yaml
   File Path: backups/{{ $json.name }}.json
   File Content: {{ JSON.stringify($json) }}
   Commit Message: Automated backup: {{ $now }}
   ```

### Manual Backup
Export individual workflows:
1. Open workflow in n8n
2. Click ⋮ menu → Download
3. Commit to GitHub repository

## Common Commands

### System Status
```
"Bob, give me a status report"
"Bob, check server uptime"
"Bob, how much RAM are we using?"
"Bob, what's the CPU temperature?"
```

### Docker Management
```
"Bob, list running containers"
"Bob, show me docker container status"
```

### Email Operations
```
"Bob, do I have any new emails?"
"Bob, write an email to [address] about [topic]"
"Bob, summarize my last 3 unread emails"
```

### Complex Tasks (Delegated to Mistral)
```
"Bob, have the specialist draft a professional email about..."
"Bob, analyze this error log: [paste log]"
```

## Troubleshooting

### Bob Not Responding on Telegram

1. Check workflow is **Active** (toggle in top right)
2. Verify Webhook URL is set correctly:
   ```bash
   docker logs n8n | grep "Webhook URL"
   ```
3. Ensure Cloudflare tunnel is healthy
4. Test with simple message like "hi"

### SSH Connection Fails

```bash
# Test SSH manually
ssh alx@192.168.0.185

# Check credentials in n8n
# Verify IP address is correct
# Ensure port 22 is accessible
```

### Gmail "Access Denied"

**For App Password:**
- Ensure 2FA is enabled
- Regenerate app password
- Verify SMTP/IMAP settings match exactly

**For OAuth2:**
- Add yourself to test users in Google Cloud Console
- Check redirect URI matches exactly
- Wait 2-5 minutes after changes

### Memory Issues

If Bob forgets conversations:
1. Verify Session ID is set: `{{ $json.message.chat.id }}`
2. Consider upgrading to Postgres Chat Memory for persistence
3. Check memory node is connected properly

### "Cannot GET /" Error

```bash
# Fix permissions
sudo chown -R 1000:1000 ~/n8n_data
sudo chmod -R 775 ~/n8n_data

# Restart container
docker restart n8n
```

## Advanced Features

### Multi-AI Setup

Bob uses a "Manager-Specialist" architecture:
- **Groq** - Fast responses for chat and simple commands
- **Mistral** - Deep analysis and professional writing

You can add more specialists:
- **Gemini** - Research and web search
- **Claude** - Complex reasoning and safety checks

### Memory Upgrades

Upgrade from Simple Memory to Postgres:
1. Persistent across restarts
2. Better for long-term "relationship" with Bob
3. Uses existing n8n database

### Workflow Versioning

Use GitHub to:
- Track changes to Bob's behavior
- Roll back problematic updates
- Share workflows with others
- Restore after hardware failure

## Performance Tips

### ZimaBoard Optimization

```bash
# Increase swap if needed
sudo nano /etc/dphys-swapfile
# Set CONF_SWAPSIZE=2048

# Restart swap
sudo systemctl restart dphys-swapfile
```

### n8n Performance

```bash
# Set executions to main process (saves RAM)
docker run -d \
  --name n8n \
  -e EXECUTIONS_PROCESS=main \
  -e EXECUTIONS_DATA_PRUNE=true \
  -e EXECUTIONS_DATA_MAX_AGE=72 \
  ...
```

### AI Model Selection
- **Quick status checks** → Groq (Llama 3.1 8B)
- **Complex analysis** → Groq (Llama 3.3 70B)
- **Professional writing** → Mistral
- **Web research** → Gemini (if added)

## Cost Analysis

| Service | Cost | Notes |
|---------|------|-------|
| Groq | Free | Generous rate limits |
| Mistral | ~$0.25/M tokens | Pay as you go |
| Gemini | Free tier available | 1,500 req/day |
| Cloudflare Tunnel | Free | Zero Trust plan |
| GitHub | Free | Private repos included |
| Domain | $10-15/year | One-time annual cost |
| **Total Monthly** | **~$0-2** | Excluding domain |

## Maintenance

### Daily
- Monitor Telegram for errors
- Check Bob's response quality

### Weekly
- Review GitHub backups
- Check ZimaBoard temperature/storage

### Monthly
- Update Docker containers:
  ```bash
  docker pull n8nio/n8n
  docker pull cloudflare/cloudflared
  docker restart n8n cloudflared
  ```
- Review and optimize workflows
- Check API usage/limits

## Resources

### Documentation
- [n8n Docs](https://docs.n8n.io)
- [Cloudflare Tunnel Docs](https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/)
- [Groq API Docs](https://console.groq.com/docs)
- [Telegram Bot API](https://core.telegram.org/bots/api)

### Community
- [n8n Forum](https://community.n8n.io)
- [ZimaBoard Forum](https://forum.zimaboard.com)

## License

This project is for personal use. Respect all API terms of service for Telegram, Gmail, Groq, and Mistral.

## Contributing

Feel free to submit improvements via GitHub pull requests to your forked repository.

## Acknowledgments

Built with:
- n8n (Workflow Automation)
- Groq (Fast AI Inference)
- Mistral AI (Specialized Tasks)
- Cloudflare (Secure Tunneling)
- Telegram (User Interface)

---

**Last Updated:** January 31, 2026
**Version:** 1.0.0
**Tested On:** ZimaBoard 832, Debian 12

---

Made with ❤️ in Toronto, Canada 🇨🇦 by Alexander Wondwossen (@alxgraphy)

---

*Remember: Bob is as smart as you configure him to be. Start simple, add features gradually, and always test in a safe environment before running destructive commands.*
