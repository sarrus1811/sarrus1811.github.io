# Org Mode for Cyber security

[← Back to Home](../)

A very common pain point in security workflows is collecting data and writing reports from day-to-day operations and engagements. Outside of the usual copy-paste routine of manually grabbing screenshots and terminal scrollback, security reporting is often a wasteland. In this article, I propose a cleaner approach using literate programming with Emacs and Org mode.

## Quick Primer on Org Mode

Before getting into the offensive workflow, let's cover the absolute basics of Org mode.

### Core Syntax
Org mode uses plain-text symbols to structure documents, similar to Markdown ([1](https://lwn.net/Articles/957316/)):
- **Headings:** Start lines with one or more asterisks followed by a space (for example, `* Top Level` or `** Subheading`).
- **Text Formatting:** Wrap words in `*asterisks*` for bold, `/slashes/` for italics, and `_underscores_` to underline.
- **Lists:** Use `-` or `+` for bulleted lists, and `1.` or `1)` for numbered lists.
- **Checkboxes:** Add `[ ]` after a list item (e.g., `- [ ] Run subdomain enumeration`) to create an interactive checklist.
- **Links:** Links use double brackets like `[[[https://orgmode.org](https://orgmode.org)][Org Mode Website]]`.

> **Note:** In Emacs keybinding notation, `C-` is Ctrl, `M-` is Alt/Meta, and `S-` is Shift.

### Outlining and Navigation
Org mode treats headings as collapsible trees, making it easy to navigate large engagement files:
- **Fold and Unfold:** Press `TAB` on a heading to cycle its visibility. Alternatively, press `S-TAB` to collapse or expand the entire document.
- **Create Headings:** Press `M-RET` (Alt + Enter) to insert a new heading at the same level.
- **Restructure Sections:** Use `M-Left` or `M-Right` to promote or demote a heading level, and `M-Up` or `M-Down` to move an entire section up or down.

### Task Management
Any heading can become an actionable task (see the [Official Org Mode Compact Guide](https://orgmode.org/guide/Introduction.html) for more details):
- **TODO States:** Use `TODO` after the asterisks (e.g., `* TODO Finish report`). Press `C-c C-t` or `S-Right` to cycle between `TODO`, `DONE`, and no state.
- **Toggle Checkboxes:** Place the cursor on a `[ ]` checkbox and press `C-c C-c` to check or uncheck it.
- **Dates and Deadlines:** Press `C-c C-s` to schedule a task for a specific date, or `C-c C-d` to set a deadline.
- **Agenda View:** Press `C-c a` to open the Agenda, which aggregates scheduled tasks and deadlines across your `.org` files into a unified view.

### Tables and Exporting
- **Quick Tables:** Type `| Name | Role |` and press `TAB` to automatically format and align a plain-text table.
- **Exporting:** Press `C-c C-e` to open the export dispatcher, allowing you to convert your `.org` file into HTML, PDF, Markdown, or LaTeX.

### Source Blocks (Org Babel)
This is the most interesting part of Org mode for security practitioners. Source blocks are part of a built-in feature called **Org Babel**, which executes commands and scripts directly inside Emacs and captures the output inline. Consider the following example source block:

{% raw %}
```org
#+begin_src sh :results output verbatim :exports both
exec 2>&1
podman ps --format "table {{.Names}}\t{{.Image}}\t{{.Ports}}"
#+end_src
```
{% endraw %}

- `#+begin_src ... #+end_src` defines the delimiters of the code block.
- `sh` is the language identifier.
- `:results output verbatim` is the header argument controlling how `stdout` is captured and formatted.
- `:exports both` instructs Org mode to include both the code and the output when exporting to HTML, Markdown, or PDF.
- Everything between `#+begin_src` and `#+end_src` is the actual command payload being executed.

Pressing `C-c C-c` with your cursor inside the source block executes it. Notice the `exec 2>&1` line: many CLI tools print banners and status messages to `stderr`, which causes Emacs to pop open a separate error buffer. Redirecting `stderr` to `stdout` with `exec 2>&1` keeps the output inline inside your document (this can also be set globally via `:prologue "exec 2>&1"`, though applying it selectively gives you finer control over noisy tools).

---

## A Worked Example: Podman + ProjectDiscovery Lab

### Requirements
This walkthrough requires **Podman**, the **ProjectDiscovery Suite** (`httpx`, `katana`, `nuclei`), and **Emacs** with Org Babel configured. You can use Doom Emacs, Spacemacs, or a custom configuration like [mine](https://github.com/sarrus1811/emacs_config). Org file for in this article is available at [here](https://github.com/sarrus1811/sarrus1811.github.io/blob/main/docs/assets/org-lab.org).

To export the final engagement playbook into Markdown and Quarto reports, install `pandoc` and `quarto`:

```bash
sudo dnf install -y pandoc
mise use --global github:quarto-dev/quarto-cli@latest
```

Save the blocks below into `org-lab.org` to follow along, placing your cursor inside each source block and pressing `C-c C-c` to execute.

### 0. Update Nuclei Templates

```org
* 0. Update Nuclei Templates
#+begin_src sh :results output verbatim :exports both
exec 2>&1
nuclei -update-templates -no-color
#+end_src
```

A `#+RESULTS:` block will appear directly below the code block:

```org
#+RESULTS:
#+begin_example

                     __     _
   ____  __  _______/ /__  (_)
  / __ \/ / / / ___/ / _ \/ /
 / / / / /_/ / /__/ /  __/ /
/_/ /_/\__,_/\___/_/\___/_/   v3.11.1

		projectdiscovery.io

[INF] No new updates found for nuclei templates
#+end_example
```

### 1. Deploy Vulnerable Containers for the Lab

{% raw %}
```org
* 1. Spin Up Vulnerable Podman Targets
** 1.1 Launch Containers
#+begin_src sh :results output verbatim :exports both
exec 2>&1
podman run -d --rm --name vuln-apache -p 127.0.0.1:8081:80   docker.io/vulhub/httpd:2.4.50
podman run -d --rm --name vuln-juice  -p 127.0.0.1:8082:3000 docker.io/bkimminich/juice-shop:latest
podman run -d --rm --name vuln-vampi  -p 127.0.0.1:8083:5000 docker.io/erev0s/vampi:latest
podman run -d --rm --name vuln-dvwa   -p 127.0.0.1:8084:80   docker.io/vulnerables/web-dvwa:latest

sleep 5
#+end_src

#+RESULTS:
: 2f413d0482b9352b766708ac2b933bebd2ddf0d37bfecbdfc446aedfe338e29a
: ec904b7ba6960184767ca11005fbc2d2c83a2f659c9817b5a3f532271c3a2143
: c88834958506f16a2949aaa41c10c05e12e9d27e236bd3cdfd3202b31110ffa7
: 8d26a76066fbbff8baacc4b032a7d8dc56d0aacd588f040cf989b17f86bb143c

** 1.2 Confirm Deployment
#+begin_src sh :results output verbatim :exports both
podman ps --format "table {{.Names}}\t{{.Image}}\t{{.Ports}}"
#+end_src

#+RESULTS:
: NAMES        IMAGE                                   PORTS
: vuln-apache  docker.io/vulhub/httpd:2.4.50           127.0.0.1:8081->80/tcp
: vuln-juice   docker.io/bkimminich/juice-shop:latest  127.0.0.1:8082->3000/tcp
: vuln-vampi   docker.io/erev0s/vampi:latest           127.0.0.1:8083->5000/tcp
: vuln-dvwa    docker.io/vulnerables/web-dvwa:latest   127.0.0.1:8084->80/tcp
```
{% endraw %}

### 2. Define Target Scope & Fingerprint with HTTPX
Here we define an Org table named `target_ports` and pass it directly into our shell block as a variable (`:var scope=target_ports`), piping the first column (`awk '{print $1}'`) into `httpx`:

```org
* 2. Target Scope & HTTPX Fingerprinting
#+NAME: target_ports
| URL                   | App_Name        |
|-----------------------+-----------------|
| http://127.0.0.1:8081 | Apache_2.4.50   |
| http://127.0.0.1:8082 | OWASP_JuiceShop |
| http://127.0.0.1:8083 | VAmPI_REST_API  |
| http://127.0.0.1:8084 | DVWA            |

#+NAME: live_urls
#+begin_src sh :var scope=target_ports :results output verbatim :exports both
exec 2>&1
echo "$scope" | awk '{print $1}' | httpx -silent -status-code -title -tech-detect -no-color
#+end_src
```

### 3. Endpoint Discovery with Katana

```org
* 3. Endpoint Discovery with Katana
#+begin_src sh :results output verbatim :exports both
exec 2>&1
katana -u http://127.0.0.1:8082 -u http://127.0.0.1:8083 -d 2 -silent -no-color | head -n 25
#+end_src
```

### 4. Silent Vulnerability Scanning with Nuclei
Because raw `nuclei` scans generate massive output streams, we use `:results silent :exports code` and write the structured findings to `/tmp/nuclei-results.jsonl` on disk for downstream triage:

```org
* 4. Vulnerability Scanning with Nuclei
#+begin_src sh :var scope=target_ports :results silent :exports code
rm -f /tmp/nuclei-results.jsonl
echo "$scope" | awk '{print $1}' | nuclei \
  -silent \
  -no-color \
  -tags cve,exposures,misconfig,default-login,panel,tech \
  -jsonl -o /tmp/nuclei-results.jsonl >/dev/null 2>&1
#+end_src
```

### 5. Parsing Findings with Python into an Inline Org Table
Literate programming in Org mode is not limited to shell scripts—you can mix languages freely in the same file once enabled in `init.el`. This Python block parses `/tmp/nuclei-results.jsonl`, sorts findings by severity, and uses `:results value table` to render a native Org table:

```org
* 5. Automated Vulnerability Triage Table (Python -> Inline Org Table)
#+begin_src python :results value table :colnames '("Severity" "Template_ID" "Target_URL" "Matched_Name") :exports both
import json
from pathlib import Path

results_file = Path("/tmp/nuclei-results.jsonl")
rows = []

if results_file.exists():
    for line in results_file.read_text().splitlines():
        if not line.strip():
            continue
        item = json.loads(line)
        severity = item.get("info", {}).get("severity", "unknown").upper()
        template_id = item.get("template-id", "")
        matched = item.get("matched-at", "")
        name = item.get("info", {}).get("name", "")
        rows.append([severity, template_id, matched, name])

order = {"CRITICAL": 0, "HIGH": 1, "MEDIUM": 2, "LOW": 3, "INFO": 4}
rows.sort(key=lambda r: order.get(r[0], 99))
return rows if rows else [["NONE", "No findings", "-", "-"]]
#+end_src
```

Executing the block produces a clean, sorted table right inside the buffer:

```org
#+RESULTS:
| Severity | Template_ID          | Target_URL                          | Matched_Name                    |
|----------+----------------------+-------------------------------------+---------------------------------|
| HIGH     | dvwa-setup           | http://127.0.0.1:8084/setup.php     | DVWA Exposed Setup Page         |
| LOW      | apache-server-status | http://127.0.0.1:8084/server-status | Apache Server Status Disclosure |
| INFO     | tech-detect          | http://127.0.0.1:8084               | Wappalyzer Technology Detection |
```

### Bonus: Remote Execution Over SSH Jump Hosts with TRAMP
Everything shown above also works across remote hosts using Emacs **TRAMP**. By adding a `:dir` header argument to any source block, Emacs streams the commands over SSH (including multi-hop bastion pivots and remote `sudo` escalation) and returns the output directly into your local `.org` file without leaving scripts on the target:

```org
* Remote Triage Over SSH Bastion
#+begin_src sh :dir /ssh:op@jumpbox.corp|ssh:analyst@10.20.1.14:/tmp :results output verbatim
uname -a
ss -tulnp
#+end_src
```

### 6. Exporting the Playbook to Markdown & Quarto Reports
Once the engagement is complete, we can use a custom export block to convert the `.org` file into both GitHub-Flavored Markdown (`.md`) and an interactive Quarto notebook (`.qmd`):

```org
* 6. Export Playbook to GitBook Markdown & Quarto (.qmd)
#+begin_src sh :results output verbatim :exports none
INPUT_ORG="org-lab.org"

# Generate GitBook Markdown (.md)
pandoc "$INPUT_ORG" -f org -t gfm -s -o org-lab.md
echo "[+] Successfully generated org-lab.md."

# Generate Quarto Notebook (.qmd) with executable {bash} and {python} chunk headers
cat << 'EOF' > org-lab.qmd
---
title: "Org Mode for Security Professionals"
format:
  html:
    theme: darkly
    toc: true
  pdf: default
execute:
  eval: false
  echo: true
---

EOF

pandoc "$INPUT_ORG" -f org -t gfm \
  | sed -E 's/^```(sh|bash)$/```{bash}/g; s/^```python$/```{python}/g' \
  >> org-lab.qmd

echo "[+] Successfully generated org-lab.qmd."
#+end_src

#+RESULTS:
: [+] Successfully generated org-lab.md.
: [+] Successfully generated org-lab.qmd.
```

Executing this block produces `org-lab.md` and `org-lab.qmd` (with `eval: false` set in the YAML frontmatter so Quarto preserves our captured results without re-running the scans). Render the Quarto notebook to HTML with:

```bash
quarto render org-lab.qmd --to html
```
