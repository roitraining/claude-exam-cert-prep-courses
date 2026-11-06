# Validating Lab Prerequisites

## Time Required

20 minutes

## Overview

In this lab, you will check that your computer is ready for the course labs and demos. You confirm Python and an editor, verify that Claude Code is installed and signed in, create your own Claude API key, and build the shared `ccarf-labs` workspace that the demos use. A short script then checks everything for you, makes one tiny API call, and runs Claude Code once without a prompt window.

### You learn how to:
- Verify Python 3.10 or newer and a code editor.
- Install, check, and sign in to Claude Code.
- Create a Claude Console API key and store it safely.
- Create the shared workspace and validate it with a checker script.

## Scenario

Every lab and demo in this course runs on your own machine. Fixing a missing tool in the middle of a 45-minute lab wastes your time, so you check everything now. The course uses two separate Claude accounts: your **Claude subscription** runs Claude Code, and a **Claude Console API key** lets your own Python code call the Claude API and the Claude Agent SDK.

## Lab Instructions

### Task 1: Check Python and your editor

In this task, you open a terminal and confirm that Python 3.10 or newer and a code editor are installed.

1. Open a terminal:

| Operating system | How to open it |
| :--- | :--- |
| macOS | Press Cmd+Space, type **Terminal**, and press Return |
| Windows | Click Start, type **PowerShell**, and open **Windows PowerShell** |

2. Check your Python version.

**macOS:**

```bash
python3 --version
```

**Windows (PowerShell):**

```powershell
python --version
```

3. Confirm that the version is **3.10 or newer** (for example `Python 3.12.4`).

> [!IMPORTANT]
> If the command is not found or the version is older than 3.10, install Python from [python.org/downloads](https://www.python.org/downloads/). On Windows, select **Add python.exe to PATH** on the first installer screen, then close and reopen PowerShell. If typing `python` on Windows opens the Microsoft Store, install from python.org instead.

4. Confirm that you have a code editor. The labs use [VS Code](https://code.visualstudio.com/), but any editor that can open a folder works.

5. **Windows only:** install [Git for Windows](https://git-scm.com/downloads/win) if you do not have it. Claude Code uses its Git Bash shell to run commands. Accept the installer defaults.

### Task 2: Verify Claude Code and sign in

In this task, you confirm that Claude Code is installed and signed in with your Claude subscription.

1. Check whether Claude Code is installed:

```bash
claude --version
```

2. If the command prints a version number such as `2.1.316 (Claude Code)`, skip to step 4. Otherwise, install Claude Code.

**macOS:**

```bash
curl -fsSL https://claude.ai/install.sh | bash
```

**Windows (PowerShell):**

```powershell
irm https://claude.ai/install.ps1 | iex
```

3. Close your terminal, open a new one, and run `claude --version` again.

4. Run the built-in health check:

```bash
claude doctor
```

5. Read the output and confirm that it reports no errors.

6. Start Claude Code:

```bash
claude
```

7. If Claude Code asks you to sign in, choose your Claude subscription account and complete the sign-in in your browser.

8. At the Claude Code prompt, type the following and press Enter:

```text
In one sentence, what can you help me do in this folder?
```

9. When Claude answers, type `/exit` to leave Claude Code.

> [!NOTE]
> Claude Code requires a paid Claude plan (Pro, Max, Team, or Enterprise) or a Console account. The free Claude plan does not include Claude Code.

### Task 3: Create a Claude Console API key

In this task, you create your own API key so that your Python code can call the Claude API and the Claude Agent SDK. API usage is billed to the Console, separately from your Claude subscription.

1. Go to [platform.claude.com](https://platform.claude.com/) and sign in, or create an account.

2. Open **Settings**, then **Billing**, and add a small amount of prepaid credit. The Console shows the current minimum purchase. All of the demos together cost well under $1 because they use the low-cost Claude Haiku model.

3. Open the [API keys](https://platform.claude.com/settings/keys) page and click **Create key**.

4. Name the key `ccarf-labs`, keep the **Default** workspace, and click **Create**.

5. Click the copy button and keep the key in your clipboard for Task 4. It starts with `sk-ant-`.

> [!WARNING]
> The Console shows the full key only once. Treat it like a password: never paste it into a chat, a slide, a screenshot, or a Git repository. If you lose it or expose it, delete it in the Console and create a new one.

### Task 4: Create the workspace and run the checker

In this task, you create the `ccarf-labs` folder that the demos use, install the Python packages, store your API key in a `.env` file, and run a script that checks everything.

1. Create the workspace in your home folder and open it.

**macOS:**

```bash
cd ~
mkdir ccarf-labs
cd ccarf-labs
```

**Windows (PowerShell):**

```powershell
cd $HOME
mkdir ccarf-labs
cd ccarf-labs
```

2. Create a Python virtual environment and activate it.

**macOS:**

```bash
python3 -m venv .venv
source .venv/bin/activate
```

**Windows (PowerShell):**

```powershell
python -m venv .venv
.venv\Scripts\Activate.ps1
```

> [!NOTE]
> If Windows says that running scripts is disabled, run `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned`, answer **Y**, and run the activate command again. When the environment is active, your prompt starts with `(.venv)`, and `python` and `pip` work the same way on macOS and Windows.

3. Install the packages used in the demos and labs:

```bash
pip install anthropic python-dotenv mcp claude-agent-sdk
```

4. Open the `ccarf-labs` folder in your editor and create a file named `.env` that contains one line. Paste your key in place of the placeholder:

```text
ANTHROPIC_API_KEY=sk-ant-paste-your-key-here
```

5. In the same folder, create a file named `.gitignore` so the key and the environment are never committed:

```text
.env
.venv/
```

6. In the same folder, create a file named `check_setup.py`:

```python
# check_setup.py - verifies your machine is ready for the CCAR-F labs and demos
import importlib
import os
import shutil
import subprocess
import sys

results = []

def check(name, ok, detail=""):
    results.append(ok)
    print(f"[{'PASS' if ok else 'FAIL'}] {name}" + (f" - {detail}" if detail else ""))

# 1. Python version
check("Python 3.10 or newer", sys.version_info >= (3, 10), sys.version.split()[0])

# 2. Packages
for pkg in ("anthropic", "dotenv", "mcp", "claude_agent_sdk"):
    try:
        importlib.import_module(pkg)
        check(f"Package '{pkg}' installed", True)
    except ImportError:
        check(f"Package '{pkg}' installed", False, "run: pip install anthropic python-dotenv mcp claude-agent-sdk")

# 3. Claude Code on PATH
claude_path = shutil.which("claude")
check("Claude Code CLI on PATH", claude_path is not None, claude_path or "install Claude Code, then open a new terminal")

# 4. Claude Code answers without a prompt window, using your subscription
if claude_path:
    env = {k: v for k, v in os.environ.items() if k != "ANTHROPIC_API_KEY"}
    try:
        run = subprocess.run(
            [claude_path, "-p", "Reply with the single word: ready"],
            capture_output=True, text=True, timeout=180, env=env,
        )
        reply = run.stdout.strip()
        check("Claude Code runs with -p", run.returncode == 0 and reply != "",
              f"replied '{reply[:40]}'" if run.returncode == 0 else (run.stderr.strip()[:200] or "start claude once and sign in"))
    except subprocess.TimeoutExpired:
        check("Claude Code runs with -p", False, "timed out: start claude once and sign in")
else:
    check("Claude Code runs with -p", False, "skipped: Claude Code not found")

# 5. API key in .env
try:
    from dotenv import load_dotenv
    load_dotenv()
except ImportError:
    pass
key = os.environ.get("ANTHROPIC_API_KEY", "")
check("ANTHROPIC_API_KEY found in .env", key.startswith("sk-ant-"), "key loaded" if key else "create .env in this folder")

# 6. One tiny API call
if key.startswith("sk-ant-"):
    try:
        import anthropic
        client = anthropic.Anthropic()
        resp = client.messages.create(
            model="claude-haiku-5-5",
            max_tokens=1028,
            output_config={"effort": "low"},
            messages=[{"role": "user", "content": "Reply with the single word: ready"}],
        )
        text = "".join(b.text for b in resp.content if b.type == "text").strip()
        check("Claude API call succeeded", True, f"model replied '{text}' using {resp.usage.input_tokens + resp.usage.output_tokens} tokens")
    except Exception as err:  # show the real error so you can fix it
        check("Claude API call succeeded", False, f"{type(err).__name__}: {err}")
else:
    check("Claude API call succeeded", False, "skipped: no API key")

print()
print("All checks passed. You are ready for the labs!" if all(results) else "Fix the FAIL items above, then run this script again.")
```

7. Run the checker:

```bash
python check_setup.py
```

8. Confirm that every line shows **PASS** and the script ends with `All checks passed. You are ready for the labs!`

> [!NOTE]
> The Claude Code check can take up to a minute. If it fails, run `claude` once in this terminal, finish signing in, type `/exit`, and run the checker again. If the API check fails with `AuthenticationError`, re-copy the key into `.env` with no spaces or quotes. If it fails with a billing or credit message, add credit in the Console and wait a minute.

> [!IMPORTANT]
> Every demo starts the same way: open a terminal, change to `ccarf-labs`, and activate the virtual environment. Each lab creates its own `ccarf-lab-NN` folder; you can create those folders inside `ccarf-labs`. Always start Claude Code from an activated terminal so it can find the course Python. Your key stays in `.env`, so Claude Code keeps using your subscription.

### Bonus Task 5: Confirm that the Agent SDK can reach Claude

In this bonus task, you run one Claude Agent SDK query, the same library the Chapter 2 hook demo uses.

1. In `ccarf-labs`, create a file named `check_agent_sdk.py`:

```python
# check_agent_sdk.py - one Claude Agent SDK query using the key in .env
import asyncio
from dotenv import load_dotenv
from claude_agent_sdk import query, ClaudeAgentOptions, AssistantMessage, TextBlock

load_dotenv()

async def main():
    options = ClaudeAgentOptions(model="claude-haiku-5-5", max_turns=1)
    async for message in query(prompt="Reply with the single word: ready", options=options):
        if isinstance(message, AssistantMessage):
            for block in message.content:
                if isinstance(block, TextBlock):
                    print("Agent SDK replied:", block.text)

asyncio.run(main())
```

2. Run it:

```bash
python check_agent_sdk.py
```

3. Confirm that the terminal prints `Agent SDK replied:` followed by a short reply. A warning that claude.ai connectors are disabled may print first; that is expected, because the API key in `.env` takes precedence over your Claude sign-in.

## Congratulations!

In this lab, you have:
- Verified Python 3.10 or newer and a code editor.
- Installed, checked, and signed in to Claude Code.
- Created a Claude Console API key and stored it safely.
- Created the shared workspace and validated it with a checker script.
