# How to Post Your n8n Workflows to GitHub (Community Edition)

Since you're on the **Community Edition** of n8n, you don't have access to the built-in "Source Control" feature. But don't worry—there are several ways to backup and share your workflows on GitHub!

## Quick Overview

You have **3 options**:

1. **Manual Export** (Easiest, 2 minutes)
2. **Auto-Backup Workflow** (Professional, automatic)
3. **GitHub CLI Method** (Advanced, for power users)

---

## Option 1: Manual Export (Recommended for Beginners)

This is the simplest way to get your workflows on GitHub.

### Step 1: Create a GitHub Repository

1. Go to [github.com/new](https://github.com/new)
2. Name it something like `my-n8n-workflows` or `bob-ai-assistant`
3. Make it **Private** (to protect your workflow logic)
4. Click **Create repository**

### Step 2: Export Your Workflows from n8n

In your n8n dashboard:

1. Open each workflow you want to backup
2. Click the **⋮** (three dots menu) in the top right
3. Select **Download**
4. Save the `.json` file to your computer
5. Repeat for all workflows you want to backup

**Pro Tip:** Name your files descriptively:
- `bob-telegram-bot.json`
- `gmail-automation.json`
- `ssh-server-monitor.json`

### Step 3: Upload to GitHub

#### Option A: Via GitHub Website (No Git Required)

1. Go to your new repository on GitHub
2. Click **Add file** → **Upload files**
3. Drag and drop all your `.json` files
4. Add a commit message like "Initial workflow backup"
5. Click **Commit changes**

#### Option B: Via Git (If you have Git installed)

```bash
# On your computer (not ZimaBoard)
cd ~/Downloads  # or wherever you saved the files

# Initialize git
git init
git add *.json
git commit -m "Initial n8n workflow backup"

# Connect to GitHub
git remote add origin https://github.com/YOUR_USERNAME/YOUR_REPO_NAME.git
git branch -M main
git push -u origin main
```

---

## Option 2: Auto-Backup Workflow (Best for Regular Backups)

This creates a workflow IN n8n that automatically backs up all your OTHER workflows to GitHub.

### Prerequisites

1. **GitHub Personal Access Token**
   - Go to [github.com/settings/tokens](https://github.com/settings/tokens)
   - Click **Generate new token** → **Fine-grained tokens**
   - Name: `n8n-backup-token`
   - Repository access: Select your backup repo
   - Permissions: **Contents** → **Read and write**
   - Click **Generate token**
   - **SAVE THIS TOKEN** - you won't see it again!

2. **GitHub Credentials in n8n**
   - In n8n, click the **🔑 Credentials** icon (left sidebar)
   - Click **Add Credential**
   - Search for "GitHub"
   - Select **GitHub API**
   - Paste your token
   - Name it: `GitHub Backup Token`
   - Click **Save**

### The Auto-Backup Workflow

Here's the complete workflow to paste into n8n:

#### How to Import This Workflow:

1. In n8n, click **+ Add workflow** (top right)
2. Click the **⋮** menu → **Import from URL** or **Import from File**
3. Copy the JSON below and paste it, or save as a file and import

```json
{
  "name": "Auto-Backup to GitHub",
  "nodes": [
    {
      "parameters": {
        "rule": {
          "interval": [
            {
              "field": "cronExpression",
              "expression": "0 3 * * *"
            }
          ]
        }
      },
      "id": "schedule-trigger",
      "name": "Schedule Trigger",
      "type": "n8n-nodes-base.scheduleTrigger",
      "typeVersion": 1.2,
      "position": [250, 300]
    },
    {
      "parameters": {
        "resource": "workflow",
        "operation": "getAll",
        "returnAll": true
      },
      "id": "get-all-workflows",
      "name": "Get All Workflows",
      "type": "n8n-nodes-base.n8n",
      "typeVersion": 1,
      "position": [470, 300]
    },
    {
      "parameters": {},
      "id": "split-out",
      "name": "Split Out",
      "type": "n8n-nodes-base.splitOut",
      "typeVersion": 1,
      "position": [690, 300]
    },
    {
      "parameters": {
        "resource": "workflow",
        "operation": "get",
        "workflowId": "={{ $json.id }}"
      },
      "id": "get-workflow-details",
      "name": "Get Workflow Details",
      "type": "n8n-nodes-base.n8n",
      "typeVersion": 1,
      "position": [910, 300]
    },
    {
      "parameters": {
        "operation": "create",
        "owner": "YOUR_GITHUB_USERNAME",
        "repository": "YOUR_REPO_NAME",
        "filePath": "=workflows/{{ $json.name.replace(/[^a-z0-9]/gi, '-').toLowerCase() }}.json",
        "fileContent": "={{ JSON.stringify($json, null, 2) }}",
        "commitMessage": "=Auto-backup: {{ $json.name }} - {{ $now.toFormat('yyyy-MM-dd HH:mm') }}",
        "additionalParameters": {
          "author": {
            "name": "n8n Auto-Backup",
            "email": "backup@n8n.local"
          }
        }
      },
      "id": "create-or-update-file",
      "name": "Create/Update File on GitHub",
      "type": "n8n-nodes-base.github",
      "typeVersion": 1,
      "position": [1130, 300],
      "credentials": {
        "githubApi": {
          "id": "YOUR_CREDENTIAL_ID",
          "name": "GitHub Backup Token"
        }
      }
    }
  ],
  "connections": {
    "Schedule Trigger": {
      "main": [[{"node": "Get All Workflows", "type": "main", "index": 0}]]
    },
    "Get All Workflows": {
      "main": [[{"node": "Split Out", "type": "main", "index": 0}]]
    },
    "Split Out": {
      "main": [[{"node": "Get Workflow Details", "type": "main", "index": 0}]]
    },
    "Get Workflow Details": {
      "main": [[{"node": "Create/Update File on GitHub", "type": "main", "index": 0}]]
    }
  },
  "settings": {
    "executionOrder": "v1"
  }
}
```

### After Importing:

1. **Update the GitHub node:**
   - Open the "Create/Update File on GitHub" node
   - Change `YOUR_GITHUB_USERNAME` to your actual username
   - Change `YOUR_REPO_NAME` to your repository name
   - Select your "GitHub Backup Token" credential

2. **Activate the workflow:**
   - Click the **Inactive** toggle at the top right
   - It will now run every day at 3:00 AM

3. **Test it immediately:**
   - Click **Execute Workflow** to test
   - Check your GitHub repo - you should see a new `workflows/` folder with all your workflows!

---

## Option 3: Manual Backup via n8n API (Advanced)

If you prefer command-line control from your ZimaBoard:

### Step 1: Get Your n8n API Key

1. In n8n, go to **Settings** → **API**
2. Click **Create API Key**
3. Save it securely

### Step 2: Create a Backup Script

On your ZimaBoard, create this script:

```bash
#!/bin/bash
# save as: ~/backup-n8n.sh

# Configuration
N8N_URL="https://n8n.yourdomain.com"  # Your n8n URL
N8N_API_KEY="your-api-key-here"        # Your n8n API key
GITHUB_REPO="your-username/your-repo"  # Your GitHub repo
GITHUB_TOKEN="your-github-token"       # Your GitHub token
BACKUP_DIR="/tmp/n8n-backup"

# Create backup directory
mkdir -p $BACKUP_DIR
cd $BACKUP_DIR

# Get all workflows from n8n
curl -X GET "$N8N_URL/api/v1/workflows" \
  -H "X-N8N-API-KEY: $N8N_API_KEY" \
  -H "Accept: application/json" \
  -o workflows.json

# Extract individual workflows
cat workflows.json | jq -c '.data[]' | while read workflow; do
  id=$(echo $workflow | jq -r '.id')
  name=$(echo $workflow | jq -r '.name' | sed 's/[^a-zA-Z0-9]/-/g' | tr '[:upper:]' '[:lower:]')
  
  # Get full workflow details
  curl -X GET "$N8N_URL/api/v1/workflows/$id" \
    -H "X-N8N-API-KEY: $N8N_API_KEY" \
    -H "Accept: application/json" \
    -o "$name.json"
done

# Initialize git if needed
if [ ! -d .git ]; then
  git init
  git remote add origin "https://$GITHUB_TOKEN@github.com/$GITHUB_REPO.git"
fi

# Commit and push to GitHub
git add *.json
git commit -m "Automated backup: $(date '+%Y-%m-%d %H:%M:%S')"
git push -u origin main --force

# Cleanup
cd ~
rm -rf $BACKUP_DIR

echo "✅ Backup complete! Check https://github.com/$GITHUB_REPO"
```

### Step 3: Make it Executable and Run

```bash
chmod +x ~/backup-n8n.sh
./backup-n8n.sh
```

### Step 4: Automate with Cron (Optional)

```bash
# Edit crontab
crontab -e

# Add this line to run daily at 3 AM
0 3 * * * /home/alx/backup-n8n.sh >> /home/alx/n8n-backup.log 2>&1
```

---

## Important: What Gets Backed Up?

### ✅ Backed Up:
- Workflow structure (nodes, connections)
- Node settings and parameters
- Workflow settings
- Node positions (visual layout)

### ❌ NOT Backed Up:
- **Credentials** (passwords, API keys, tokens)
- **Execution history**
- **User settings**

### Critical: Save Your Encryption Key!

**This is CRUCIAL for restoring on a different machine:**

```bash
# On your ZimaBoard
docker exec -it n8n printenv N8N_ENCRYPTION_KEY
```

Save this key in a password manager. Without it, you cannot restore credentials from backups.

---

## How to Restore Workflows

### From GitHub to a New n8n Instance:

1. **Download the workflow files** from your GitHub repo
2. **In n8n**, click **+ Add workflow**
3. Click **⋮** → **Import from File**
4. Select the `.json` file
5. **Recreate credentials** (they won't import for security)
6. Click **Save**

---

## Recommended Repository Structure

Create a clean structure in your GitHub repo:

```
my-n8n-workflows/
├── README.md                    # Description of your project
├── workflows/
│   ├── bob-telegram-bot.json
│   ├── gmail-automation.json
│   ├── ssh-monitoring.json
│   └── github-backup.json       # The backup workflow itself!
├── docs/
│   ├── setup-guide.md
│   └── troubleshooting.md
└── .gitignore                   # Ignore sensitive files
```

### Create a `.gitignore` file:

```
# .gitignore
*.log
.env
credentials.json
secrets/
```

---

## Best Practices

1. **Use Private Repos** - Keep your workflow logic private
2. **Never commit credentials** - They should stay in n8n only
3. **Document your workflows** - Add a README explaining what each does
4. **Backup regularly** - Set up the auto-backup workflow
5. **Version your releases** - Use Git tags for stable versions:
   ```bash
   git tag -a v1.0.0 -m "Bob v1.0.0 - Initial stable release"
   git push origin v1.0.0
   ```

---

## Sharing Your Project Publicly

Want to share Bob with the world? Here's how:

### 1. Create a Public Repository

- Create a NEW public repo (don't make your private one public!)
- Only include workflows you want to share
- **Remove any personal information:**
  - Email addresses
  - Server IPs
  - Usernames
  - Specific file paths

### 2. Add a Detailed README

```markdown
# Bob: AI Server Commander

An n8n workflow that creates an AI assistant for server management via Telegram.

## Features
- SSH server monitoring
- Gmail integration
- Multi-AI support (Groq + Mistral)
- Auto-backup to GitHub

## Installation
1. Import workflows from the `workflows/` folder
2. Set up credentials (see SETUP.md)
3. Configure your server settings
4. Activate and enjoy!

## Credits
Made with ❤️ in Toronto, Canada 🇨🇦 by Alexander Wondwossen (@alxgraphy)
```

### 3. Add a License

```bash
# In your repo, create a LICENSE file
# MIT License is popular for open source projects
```

---

## Quick Reference: Export Methods Comparison

| Method | Difficulty | Automatic | Best For |
|--------|-----------|-----------|----------|
| Manual Download | Easy ⭐ | No | Beginners, one-time backup |
| Auto-Backup Workflow | Medium ⭐⭐ | Yes | Regular backups, peace of mind |
| API Script | Hard ⭐⭐⭐ | Yes | Power users, custom automation |

---

## Troubleshooting

### "GitHub API rate limit exceeded"

**Solution:** You're making too many commits. The auto-backup workflow runs once per day, which is fine. Don't test it repeatedly.

### "Permission denied" when pushing to GitHub

**Solution:** Your Personal Access Token doesn't have the right permissions. Regenerate it with "Contents: Read and Write" access.

### Workflows import but don't work

**Solution:** You need to recreate the credentials. GitHub backups don't include passwords/API keys for security.

### "Workflow already exists" error

**Solution:** In the GitHub node, change the operation to "Update" instead of "Create", or use the expression in the example which automatically handles both.

---

## Next Steps

1. **Backup right now** - Use the manual method to get your first backup
2. **Set up auto-backup** - Import the workflow so you never lose work again
3. **Document your workflows** - Future you will thank present you
4. **Share with the community** - Help others learn from your work!

---

Made with ❤️ in Toronto, Canada 🇨🇦 by Alexander Wondwossen (@alxgraphy)

**Need help?** Open an issue on this repo!
