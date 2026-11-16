# Emacs for security professionals

A very common pain point for security workflows is reporting and collecting data from day to day operations and engagements. If you don't include the normal copy paste method of including screenshots and command output, territory of reporting is really in InfoSec wastelands. In this article, I would like to propose a different solution in this domain using literate programming with Emacs and Org mode.
## Quick primer on Org mode
Before the interesting stuff, lets cover the absolute basics of Org mode. 
### Core Syntax
Org mode uses plain-text symbols to structure documents, similar to Markdown. [[1](https://lwn.net/Articles/957316/)]
- **Headings.** Start lines with one or more asterisks followed by a space (for example, `* Top Level` or `** Subheading`).
- **Text Formatting.** Wrap words in `*asterisks*` for bold, `/slashes/` for italics and `_underscores_` to underline.
- **Lists.** Use `-` or `+` for bulleted lists, and `1.` or `1)` for numbered lists.
- **Checkboxes.** Add `[ ]` after a list item (e.g. `- [ ] Buy groceries`) to create an interactive checklist.
- **Links.** Links use double brackets like `[[`[https://orgmode.org](https://orgmode.org/)`][Org Mode Website]]`.
**Note**: In Emacs notation, `C-` is Ctrl, `M-` is Alt/Meta, and `S-` is Shift.
### Outlining and Navigation
Org mode treats headings as collapsible trees, making it easy to navigate large files.

- **Fold and Unfold.** Press `TAB` on a heading to cycles its visibility. Alternatively press `S-TAB` to collapse or expand the entire document.
- **Create Headings.** Press `M-RET` (Alt + Enter) to insert a new heading at the same level.
- **Restructure Sections.** Use `M-Left` or `M-Right` to promote or demote a heading level, and `M-Up` or `M-Down` to move an entire section up or down. [
### Task Management
Any heading can become an actionable task. See more at [Official Org Mode Compact Guide](https://orgmode.org/guide/Introduction.html)

- **TODO States.** Use `TODO` after the asterisks (e.g., `* TODO Finish report`). Press `C-c C-t` or `S-Right` to cycle between `TODO`, `DONE`, and no state.
- **Toggle Checkboxes.** Place cursor on a `[ ]` checkbox and press `C-c C-c` to check or uncheck it.
- **Dates and Deadlines.** Press `C-c C-s` to schedule a task for a specific date, or `C-c C-d` to set a deadline.
- **Agenda View.** Press `C-c a` to open the Agenda. Agenda manages the scheduled tasks and deadlines across the `.org` files and collects into the calendar view. 
### Tables and Exporting
- **Quick Tables.** Type  for example, `| Name | Role |` and press `TAB` to format and align a plain-text table.
- **Exporting.** Press `C-c C-e` to open the export menu, allowing you to convert your `.org` file into HTML, PDF, Markdown, or LaTeX. 
### Source Blocks
This is the most interesting part of Org mode as far this article is concerned. Source blocks are part of a built-in feature called Org Babel, which lets you execute commands directly from Emacs. Lets consider the  example Source Block as follows:
```
#+begin_src sh :results output verbatim :exports both
exec 2>&1
podman ps --format "table {{.Names}}\t{{.Image}}\t{{.Ports}}"
#+end_src
```

- `#+begin_src ... #+end_src` defines the delimiters of the code block.
- `sh` is the language identifier.
- `:results output verbatim` is the header argument controlling how the output is captured.
- `:exports both` is the header argument controlling what happens during an export. (HTML, markdown, etc.)
- Everything in between  `#+begin_src` and `#+end_src` (body) is actual command(s) being run.

The Source block written so can be executed with `C-c C-c`. On a side note, I am using `exec 2>&1` to control the exact output of certain blocks. This can also be done with the header `:prologue "exec 2>&1"`, which however will execute the same for every source block.
## A worked example
### Requirements 
This section requires Podman, Project discovery Suite and of course Emacs with a adequate configuration. You can use Doom emacs, Space emacs or build one like [Mine](https://github.com/sarrus1811/emacs_config).  

This section also require Quarto and Pandoc for the purposes of creating reports. Install them as follows(I am using Fedora. Adjust the commands as required).
```
sudo dnf install -y pandoc 
mise use --global github:quarto-dev/quarto-cli@latest # Or download the latest version from quarto.org
```

Use the attached org mode file to follow along. To execute the source blocks, move the cursor over it and use `C-c C-c`. 
0. A quick update
```
* 0. Update Nuclei Templates
#+begin_src sh :results output verbatim :exports both
exec 2>&1
nuclei -update-templates -no-color
#+end_src

```

A new block should appear directly below. This should look like: 
```
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

1. Deploy the containers for this lab
```
* 1. Spin Up Vulnerable Podman Targets
** 1.1
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

** 1.2
Confirm *success* on deployment
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

2. This section contains the targets in table which directly formatted by Awk and fed to Httpx:
```
 * 2. Target Scope and Fingerprinting
Target list:
#+NAME: target_ports
| URL                   | App_Name        |
|-----------------------+-----------------|
| http://127.0.0.1:8081 | Apache_2.4.50   |
| http://127.0.0.1:8082 | OWASP_JuiceShop |
| http://127.0.0.1:8083 | VAmPI_REST_API  |
| http://127.0.0.1:8084 | DVWA            |

Fingerprinting with *httpx*.
*Command:*
#+NAME: live_urls
#+begin_src sh :var scope=target_ports :results output verbatim :exports both
exec 2>&1
echo "$scope" | awk '{print $1}' | httpx -silent -status-code -title -tech-detect -no-color
#+end_src

*Findings:*
#+RESULTS: live_urls
: http://127.0.0.1:8081 [200] [Apache HTTP Server:2.4.50,UNIX]
: http://127.0.0.1:8084 [302] [Apache HTTP Server:2.4.25,Debian,PHP]
: http://127.0.0.1:8083 [200] [Flask:2.2.3,Python:3.11.15]
: http://127.0.0.1:8082 [200] [OWASP Juice Shop]
```

3. Mapping the endpoints.
```
* 3. Endpoint Discovery
Content discovery with *Katana*.

*Command:*
#+begin_src sh :results output verbatim :exports both
exec 2>&1
katana -u http://127.0.0.1:8082 -u http://127.0.0.1:8083 -d 2 -silent -no-color | head -n 25
#+end_src

*Findings:*
#+RESULTS:
#+begin_example
http://127.0.0.1:8083
http://127.0.0.1:8082
...
http://127.0.0.1:8082/main.js
#+end_example
```

4. Vulnerability scan.
```
* 4. Vulnerability Scanning with Nuclei
*Command:*
#+begin_src sh :var scope=target_ports :results silent :exports code
rm -f /tmp/nuclei-results.jsonl
echo "$scope" | awk '{print $1}' | nuclei \
  -silent \
  -no-color \
  -tags cve,exposures,misconfig,default-login,panel,tech \
  -jsonl -o /tmp/nuclei-results.jsonl >/dev/null 2>&1
#+end_src
```

5. This Source block is actually in python. This is to show that literate programming in Org Mode is actually not limited to shell scripting. Which brings to the most important point. This feature actually has to be configured in the emacs configuration file(`init.el`). Configuring Emacs is beyond the scope of this article.
```
* 5. Automated Vulnerability Triage Table (Python -> Inline Org Table)
Formatted findings.
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

Expect a result something like this: 
```
*Result:*
#+RESULTS:
| Severity | Template_ID                     | Target_URL                          | Matched_Name                                |
|----------+---------------------------------+-------------------------------------+---------------------------------------------|
| HIGH     | dvwa-setup                      | http://127.0.0.1:8084/setup.php     | DVWA Exposed Setup Page                     |
| LOW      | apache-server-status            | http://127.0.0.1:8084/server-status | Apache Server Status Disclosure             |
...
| INFO     | tech-detect                     | http://127.0.0.1:8084               | Wappalyzer Technology Detection             |

```
### Bonus: Remote Execution Over SSH Jump Hosts with TRAMP
Everything shown here also works with Emacs TRAMP. By adding the `:dir` header to a source block, Emacs streams commands over SSH(including multi-hop bastion pivots and remote `sudo` escalation) and returns the output directly into local `.org` file without leaving any scripts on the target.
```
* Remote Triage Over SSH Bastion 
  #+begin_src sh :dir /ssh:op@jumpbox.corp|ssh:analyst@10.20.1.14:/tmp :results output verbatim 
  uname -a 
  ss -tulnp 
  #+end_src
```
### Reporting
Which brings the discussion to creating reports. 
```
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

Executing this source block will create a markdown file(`org-lab.md`) and a Quarto notebook(`org-lab.qmd`). The generated Quarto notebook can used as it is or exported to HTML as follows:
```
quarto render org-lab.qmd --to html
```

Once completed, Open the `org-lab.html` file to see a report like this:
![[quarto-report.png|872]]

7. Be sure to tear the lab down with:
```
#+begin_src sh :results output verbatim :exports both
exec 2>&1
podman stop vuln-apache vuln-juice vuln-vampi vuln-dvwa
#+end_src
```
