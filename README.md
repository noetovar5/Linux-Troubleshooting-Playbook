# Linux-Troubleshooting-Playbook
<img align="right" src="https://visitor-badge.laobi.icu/badge?page_id=noetovar5.Linux-Troubleshooting-Playbook"/>
My Real-World Linux Troubleshooting Playbook

LInk to view on browser
https://www.linkedin.com/pulse/website-down-my-real-world-linux-troubleshooting-noe-noe-tovar-mba-ualhc


Tip: Todays lecture is one you will want to save.
By Noe Tovar — Linux/DevOps Sysadmin

No one wants to hear the following in a production environment: “The site is down.” We host an Apache web app on Ubuntu. The load balancer says the backend is unhealthy. Users see errors. No one changed anything (…famous last words).

My job: get from symptom ➜ signal ➜ cause ➜ fix as efficiently as possible.

Below is exactly how I do it—live, on a real server—in a way anyone can learn and reuse.

The Three Tools I Reach For First
journalctl — the systemd log investigator that shows what happened and why a service failed. (systemd routes services’ stdout/stderr to the journal by default.) (Arch Manual Pages)
top / htop — real-time performance monitors that show what’s happening now (CPU, memory, runaway processes). htop adds interactive sorting, scrolling, and easy process actions. (man7.org)
grep — the laser-focused search to cut through massive logs and configs so I can find the one line that matters (with handy color highlighting). (gnu.org)

Industry insight: Red Hat’s docs and other vendor guides consistently place “check the logs first” at the top of any troubleshooting flow. It’s the fastest path to a root cause. (Red Hat Documentation)
Hands-On Walkthrough (Do This With Me)
Scenario: Users report 503/“Service Unavailable.” I can SSH into the server. I suspect either a service crash, a dependency failure (DB/DNS), or resource starvation.
Step 0 — Quick health glance
uptime
df -h
free -m 
I’m just checking if the box is overloaded or out of disk/RAM. If anything looks scary, I’ll confirm in top/htop next.

Step 1 — Live view with top (or htop)
top
# or, if installed:
htop 
What I’m looking for (in plain English):

A process pegging CPU at ~100% (tight loops, runaway jobs)
RAM exhaustion or huge swap usage (memory leaks)
Do I even see apache2/httpd running?

In htop, I can sort by CPU or MEM and kill a bad actor directly from the UI. (It’s a more accessible alternative to top for new admins.) (man7.org)

If I find a single process maxing out CPU, I might F9 (in htop) or:

sudo kill -9 <PID> 
…but killing is a temporary band-aid. I still need the why. That’s where logs shine.

Step 2 — Ask the system what happened with journalctl
# See recent system messages
sudo journalctl -b --no-pager | tail -200

# Focus on Apache (unit name varies: apache2.service on Debian/Ubuntu, httpd.service on RHEL)
sudo journalctl -u apache2.service --since "30 min ago"
sudo journalctl -u apache2.service -f   # follow live 
How to read this like a pro: I scan for red/error lines: “permission denied,” “address already in use,” “failed dependency,” “file not found,” “segfault,” “could not resolve host,” etc.

Why journalctl? With systemd, services write to the journal by default, so this is the most complete, time-ordered story of the failure. (Arch Manual Pages)

Pro move: If logs are huge, filter them:
sudo journalctl -u apache2.service | grep -i "error" 
(We’re combining the log stream with grep to surface only the relevant lines.) (gnu.org)

Step 3 — Chase the clue with grep
Suppose journalctl shows:

AH00558: apache2: Could not reliably determine the server's fully qualified domain name
(13)Permission denied: AH00072: make_sock: could not bind to address [::]:80 
I pivot into configs and logs with pinpoint searches:

# Search all Apache logs for "permission denied" or "bind"
grep -RniE "permission denied|bind to address" /var/log/apache2/

# Search vhost configs for the wrong port or typo
grep -Rni "VirtualHost" /etc/apache2/sites-enabled/ 
Why grep? When logs are massive (they always are), grep + a smart pattern narrows the haystack to a handful of needles, with --color=auto to highlight matches. (gnu.org)

Step 4 — Fix, reload, verify
Common quick fixes I see in the field:

Port in use: another service grabbed :80 or :443. Find it:
Bad config (typo/permission/path): validate and reload:
DNS dependency failure (app can’t resolve DB host): fix /etc/resolv.conf or the Netplan file and apply:

Step 5 — Confirm recovery
Back to htop/top:

CPU stabilizes, memory normalizes.
journalctl -u apache2.service -f shows healthy startup messages.
App is reachable; load balancer marks backend healthy.

Why This Flow Works (Expert View)
Logs first, always. Vendor and distro docs emphasize logs as the first stop because they record the exact error and time-sequence that led to failure. (Red Hat Documentation)
systemd + journalctl give a unified, queryable ledger for services—no hunting across scattered files—and services connect to the journal by default. (Arch Manual Pages)
htop improves speed of diagnosis with interactive, visual sorting and actions—faster than raw ps/kill when you’re under pressure. (man7.org)
grep is timeless: massive logs, instant answers—with helpful color and filters. (gnu.org)

Mini Lab You Can Do Today (10–15 minutes)
Create load: Run a CPU-hungry loop in a screen/tmux session:
Observe with htop: Sort by CPU. Notice the runaway process. Kill it from htop or with kill -9.
Break Apache on purpose: Edit a vhost to a bad port or invalid path.
Investigate with journalctl:
Fix & verify: Correct the config, apachectl configtest, restart, and re-check with journalctl -f and htop.

 Cheatsheet (copy/paste)
Journalctl

sudo journalctl -u <service>        # logs for a unit
sudo journalctl -f                   # follow live
sudo journalctl -b                   # since last boot 
top / htop

top                                  # live processes
sudo apt install htop && htop        # interactive viewer 
grep

grep -Rni "error" /var/log/          # recursive, numbered, case-insensitive
tail -f /var/log/syslog | grep -i fail 
(Color behavior for grep --color is documented in GNU grep.) (gnu.org)

Best Tips to Master Linux Troubleshooting
Start with symptoms, not guesses. Reproduce the error, then check logs first. (Red Hat Documentation)
Timeline matters. Use journalctl --since "1 hour ago" to focus only on the failure window. (Oracle Docs)
Watch it live. journalctl -f + htop show cause and effect in real time. (man7.org)
Filter aggressively. Pipe to grep -iE "error|fail|denied" to slash noise. (gnu.org)
Validate before restart. Config test (apachectl configtest, nginx -t, etc.) saves you from compounding errors.
Document the fix. Paste the exact error and resolution into your team runbook for next time.
Practice “chaos-lite.” Safely break and fix in a lab weekly; you’ll be calm when production breaks.

Master journalctl (what happened), top/htop (what’s happening now), and grep (where exactly)—and you’ll move from running commands to running the system. In my opinion it is so difficult to memorize all the LInux commands we have been learning. I think the point should be to become aware they exist and with experience they will slowly become second nature. I had the same difficulty when learning PowerShell and then one day it just clicked.
