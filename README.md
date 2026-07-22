# AI Employee

**AI Employee** is an autonomous, human-supervised assistant that monitors your Gmail and LinkedIn accounts, drafts responses and posts using Claude, and takes action only after you approve it. Think of it as a digital full-time-equivalent (FTE) that handles the repetitive parts of inbox and content management — reading, classifying, and drafting — while you stay in control of anything that goes out publicly or gets sent on your behalf.

It solves a specific problem: most "AI inbox" tools either auto-send things you didn't review, or require you to babysit a dashboard all day. AI Employee runs continuously in the background, does the reasoning work up front, and simply waits for a yes/no from you on anything sensitive — via a plain folder-based approval system you can manage from a normal file explorer or Obsidian.

> **Status:** Working prototype (Silver-tier build). Designed for local, single-user use on Windows. See [Requirements](#requirementsdependencies) before deploying.

---

## Key Features

- **Continuous Gmail monitoring** — checks for unread mail on a fixed interval, classifies each message, and either drafts a reply or routes it for human review.
- **LinkedIn post drafting** — takes a short brief (topic, tone, audience, key points) and generates a ready-to-review post via Claude.
- **Human-in-the-loop approval** — every outbound action (sending an email, publishing a post) sits in a `Pending_Approval` folder until a person moves it to `Approved` or `Rejected`. Nothing is sent automatically.
- **File-based audit trail** — every task, draft, approval, and rejection is a plain Markdown/JSON file, so there's a full history of what the system did and why.
- **Live status dashboard** — a self-updating Markdown dashboard (viewable in Obsidian or any text editor) shows watcher health, pending items, and token expiry status.
- **Desktop notifications** — Windows toast alerts fire when something needs your attention.
- **Duplicate-run protection** — PID lock files prevent two copies of the same watcher from running at once, even if started from multiple terminals.
- **MCP server included** — exposes core actions (e.g., sending a Gmail reply) as tools over the Model Context Protocol for use with Claude Code or other MCP clients.

---

## Requirements/Dependencies

| Requirement | Notes |
|---|---|
| **OS** | Windows (auto-start and desktop notifications are Windows-specific; core watchers can run on other OSes if you drop those two integrations) |
| **Python** | 3.13+ |
| **Node.js** | v24+ (only needed if you use Claude Code as the reasoning engine) |
| **Claude Code CLI** | `npm install -g @anthropic-ai/claude-code` — used by the classification/drafting skills |
| **Gmail API access** | Google Cloud project with Gmail API enabled + OAuth credentials |
| **LinkedIn API access** | LinkedIn Developer app with `Sign In with LinkedIn (OpenID Connect)` and `Share on LinkedIn` products approved |
| **Python packages** | `google-auth`, `google-auth-oauthlib`, `google-api-python-client`, `requests`, `python-dotenv`, `winotify` |
| **Obsidian** (optional) | Recommended for viewing the vault and dashboard, but any file explorer works |

> There is no `requirements.txt` in this build — install the packages above directly with `pip`, or generate one yourself with `pip freeze > requirements.txt` once your environment is set up.

---

## Installation

### 1. Clone the repository
```bash
git clone https://github.com/<your-username>/ai-employee.git
cd ai-employee
```

### 2. Install Python dependencies
```bash
pip install google-auth google-auth-oauthlib google-api-python-client requests python-dotenv winotify
```

### 3. Install Claude Code (used for classification and drafting)
```bash
npm install -g @anthropic-ai/claude-code
claude    # first run opens a browser to log in with your Anthropic account
```

### 4. Set up Gmail access
1. Create a project in the [Google Cloud Console](https://console.cloud.google.com/) and enable the **Gmail API**.
2. Create an **OAuth client ID** (Application type: Desktop app).
3. Download the credentials file, rename it to `gmail_credentials.json`, and place it in `credentials/`.
4. Authenticate:
   ```bash
   python integrations/gmail/auth.py
   ```
   A browser window will open — sign in and approve access. You should see `Token saved` in the terminal.

### 5. Set up LinkedIn access
1. Create an app in the [LinkedIn Developer Portal](https://www.linkedin.com/developers/).
2. Request the **Sign In with LinkedIn using OpenID Connect** and **Share on LinkedIn** products (the second can take 1–2 days to approve).
3. Copy `.env.template` to `.env`:
   ```bash
   cp .env.template .env
   ```
4. Fill in your values:
   ```env
   LINKEDIN_CLIENT_ID=your_client_id_here
   LINKEDIN_CLIENT_SECRET=your_client_secret_here
   LINKEDIN_REDIRECT_URI=http://localhost:8080/callback
   ```
5. Authenticate:
   ```bash
   python scripts/linkedin_auth.py
   ```

### 6. (Optional) Open the vault in Obsidian
Open the `vault/` folder as an Obsidian vault and pin `Dashboard/dashboard.md` for a live view of system status.

### 7. (Optional) Auto-start on Windows login
```powershell
# Run PowerShell as Administrator
powershell -ExecutionPolicy Bypass -File "scripts/setup_task_scheduler.ps1"
```

---

## Usage

### Start the watchers manually
```bash
python gmail_dev_watcher.py       # polls Gmail every 60s by default
python linkedin_dev_watcher.py    # polls the LinkedIn intake folder every 300s by default
```
Each watcher runs continuously until stopped with `Ctrl+C`, and refreshes the dashboard on every cycle.

### Submit a LinkedIn post brief
Drop a JSON file into `vault/linkedin/intake/`:
```json
{
  "topic": "AI is transforming how teams work in 2026",
  "tone": "professional",
  "audience": "tech professionals",
  "key_points": ["Automation", "Efficiency", "New opportunities"],
  "cta": "What do you think? Comment below!"
}
```
The watcher drafts a post and places it in `vault/Pending_Approval/` for review.

### Review and approve pending actions
1. Open `vault/Pending_Approval/` in Obsidian or your file explorer.
2. Read the draft (email reply or LinkedIn post).
3. Move the file to `vault/Approved/` to let it execute, or `vault/Rejected/` to discard it.

### Run the MCP server (optional)
If you want to expose AI Employee's actions as tools to Claude Code or another MCP client:
```bash
python mcp_server/server.py
```

---

## Project Structure

```
ai-employee/
├── gmail_dev_watcher.py       # Entry point: polls Gmail, runs the processor + approver loop
├── linkedin_dev_watcher.py    # Entry point: polls LinkedIn intake, runs the processor + approver loop
├── processors/
│   ├── gmail_processor.py     # Fetches + classifies + drafts replies to unread email
│   ├── gmail_approver.py      # Sends replies that have been moved to Approved/
│   ├── linkedin_processor.py  # Turns an intake brief into a drafted LinkedIn post
│   └── linkedin_approver.py   # Publishes posts that have been moved to Approved/
├── skills/
│   ├── email_classifier/      # Claude-based email classification
│   ├── email_drafter/         # Claude-based reply drafting
│   └── linkedin_drafter/      # Claude-based post drafting
├── integrations/
│   ├── gmail/                 # OAuth, reading, and sending logic for the Gmail API
│   └── linkedin/              # Config and posting logic for the LinkedIn API
├── mcp_server/
│   └── server.py              # FastMCP server exposing core actions as tools
├── scripts/
│   ├── linkedin_auth.py       # One-time LinkedIn OAuth flow
│   ├── generate_dashboard.py  # Builds/refreshes the live Markdown dashboard
│   └── setup_task_scheduler.ps1  # Registers watchers to auto-start on Windows login
├── utils/
│   ├── notifier.py            # Windows desktop notifications
│   └── plan_writer.py         # Writes multi-step task plans to the vault
├── vault/                     # Obsidian-compatible workspace — the system's "state"
│   ├── Pending_Approval/      # Drafts awaiting human review
│   ├── Approved/              # Human-approved items, ready to execute
│   ├── Rejected/              # Declined drafts
│   ├── Archive/               # Historical/source records
│   └── Dashboard/             # Live status file (dashboard.md)
├── credentials/                # OAuth tokens — gitignored, never commit this
├── .env.template               # Template for required environment variables
└── docs/
    └── WORKFLOW.md             # Detailed explanation of the vault-based workflow
```

---

## Configuration

Configuration is split between a `.env` file (secrets) and the vault (behavioral rules).

**`.env`** (copy from `.env.template`):
```env
LINKEDIN_CLIENT_ID=your_client_id_here
LINKEDIN_CLIENT_SECRET=your_client_secret_here
LINKEDIN_REDIRECT_URI=http://localhost:8080/callback
```

**`credentials/gmail_credentials.json`** — OAuth client file downloaded from Google Cloud Console. Not committed to version control.

**Polling intervals** — currently set as constants inside each watcher script (`gmail_dev_watcher.py`, `linkedin_dev_watcher.py`). If you want these configurable without editing code, consider extracting them into a `config.py` or `.env` entry, e.g.:
```env
GMAIL_POLL_INTERVAL_SECONDS=60
LINKEDIN_POLL_INTERVAL_SECONDS=300
```
> **Note:** `.gitignore` should exclude `.env`, `credentials/`, and anything under `vault/` that contains personal data before this repo is made public.

---

## Contributing

This started as a personal/hackathon build, but contributions are welcome if you'd like to extend it (e.g., add Slack or Twitter/X support, port the Windows-only pieces to macOS/Linux, or add a proper `requirements.txt` and test suite).

1. Fork the repository.
2. Create a feature branch: `git checkout -b feature/your-feature-name`.
3. Make your changes, with clear commit messages.
4. Test locally (there's no CI yet — see `scripts/test_dedup.py` and `scripts/test_fixes.py` for existing manual test scripts).
5. Open a pull request describing what changed and why.

<!-- TODO: Add a CODE_OF_CONDUCT.md and CONTRIBUTING.md if this project grows beyond a solo build. -->

---

## License

<!-- TODO: Choose a license and replace this section. MIT is a common default for personal/open-source projects: -->

This project is licensed under the [MIT License](LICENSE) — see the `LICENSE` file for details.

---

## Contact / Author

**Muhammad Kashif**
- GitHub: [@Kashif0540](https://github.com/Kashif0540)
- LinkedIn: [kashif-muhammad2830](https://www.linkedin.com/in/kashif-muhammad2830)
- Email: kashif.muhammad2830@gmail.com

For bugs or feature requests, please open an [issue](../../issues) on this repository.
