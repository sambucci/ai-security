# How to isolate coding agents in a virtual machine

*Luca Sambucci · version 0.2 · September 2026*
*[CC BY-ND 4.0](https://creativecommons.org/licenses/by-nd/4.0/)*

A coding agent is a process that runs commands on your computer with your privileges, and that takes instructions from text nobody wrote for it: the web page it just read, the README of a dependency, a comment someone left in the code. That is what makes it useful. It is also why it should not run on the machine where your email, your documents and the keys to everything else live.

This guide comes out of my own experience. I have been running agents inside virtual machines since agents got a shell, and the setup below is what I arrived at through trial, error and the occasional deleted file. It is a recipe for people who work alone or in a small team and want a real boundary. Rolling something like this out to thirty developers is a different problem, and I cover it elsewhere.

A caveat: the agent configuration examples are for Claude Code, because that is what I use. The reasoning applies to any agent that has a shell.

## 1. What you are defending against

The risk that matters is **confidentiality**: something private leaving the machine without anyone noticing. Agents deleting files happens (it happened to me, and I found out by reading the agent's reasoning, because it had not mentioned it), but git and a backup take care of that.

The mechanism is prompt injection, and it requires no sophistication. The agent reads content nobody controls in order to do its job; if any of that content contains an instruction, the agent may execute it as if you had typed it. The adversary does not come in through the network. It comes in through text, and it uses the agent to get out. From there, data has two ways to leave, and each needs a different control:

1. **The agent reads something it should not have read** and puts it in its context. At that point the content has already gone to the model provider, or it ends up in an output. The only way to close this channel is to limit **what the agent can reach on disk**.
2. **The agent runs something that ships data out**: a `curl` to an arbitrary server, a `pip install` from a hostile index, a push to the wrong remote. This channel closes by limiting **where the agent can connect**.

A virtual machine, done properly, closes both. Done the way most people do it (a VM with NAT networking and a shared folder) it closes the first one, and only halfway. The rest of this guide is about the other half.

Say it up front: how much of this you need depends on what is inside the VM. If all that lives there is your own code on a private repository, and your sessions already sync to the provider's cloud, the minimal setup in §2 may be enough. If the VM holds code under NDA, credentials to real infrastructure, or other people's data, then you also need the controls in §3, and you need them enforced by construction rather than by good intentions.

Whatever sits *inside* the VM stays exposed either way: the project's code, the credential the VM uses to talk to the git server, the session history. You accept that exposure because it is contained and because you can lower its value (dedicated credentials that expire, one project at a time). Ignoring it is a different thing.

## 2. The reference setup

Hypervisor: anything with NAT networking. VirtualBox, Hyper-V, UTM, Multipass or Parallels all work the same way, and the whole thing can be scripted (§7).

Inside: Ubuntu, 8 GB of RAM, 2 vCPUs, a 60 GB dynamic disk. That is plenty. I ran this for months on a nine-year-old laptop. Ninety-five percent of the time the heavy computation happens on the model provider's servers; the VM needs to run a terminal, git and the project's tooling. If someone tells you it takes twice or four times that, they are describing the machine they have, not the one the job needs.

Network: NAT, and nothing else. No bridged adapter, no host-only network unless there is a specific reason (and if there is, write it down). The VM reaches the internet through the host and is not reachable from the local network.

Sharing with the host: one folder, call it `transfer`, empty ninety-nine percent of the time. When a file needs to cross in either direction, you drop it there, pick it up on the other side, and the folder goes back to empty. The clipboard is convenient and the risk is small as long as no secrets go through it. The working directory is never the shared folder: projects live on the VM's own filesystem, and only transfers and backups pass through the share. People who work directly in the shared folder soon learn that every sync hiccup stops their work, on top of handing the agent a door to the host.

Inside the VM runs the agent, with git and whatever the project needs. You work in the VM's desktop or, if the provider supports it, from the app on the host attached to the same session through the provider's cloud: the session stays in the VM, only the window changes.

Keep separate sessions per task, and dedicate one to managing the VM itself: it has its own memory files, it knows the configuration, and before answering it asks the questions it needs answered. It generates the commands, the human pastes them and reports the output back. In my experience that is where the maintenance scripts nobody would otherwise write come from.

Backup: the VM is built to be thrown away, so decide what exists only in there. If the code is on a remote repository and the sessions are in the provider's cloud, that is a few hundred megabytes and an occasional manual command will do. If there is more, a daily systemd timer that archives the projects folder and the agent's state into a subfolder of `transfer`, with a `YYYYMMDDHHMM` prefix, is ten lines of work. Two details that matter: make it persistent, so that if the VM was off at the scheduled time it runs at boot; and make the retention delete only the archives it created, or one day it will eat a file someone parked in the shared folder.

Maintenance: `apt update`, `apt upgrade`, `apt autoremove`, and the VM always shuts down cleanly. `unattended-upgrades` handles the updates, so they do not depend on anyone's memory.

## 3. The three controls that make the isolation real

Everything in §2 is what most people already do. The three things below are what few people do, and they are the ones that matter once there is something inside the VM worth stealing.

### 3.1 The network: NAT does not isolate you from the local network

This is the most common mistake, and you will find it written in black and white in otherwise careful guides: the VM is on NAT, therefore it is isolated from the corporate network. With NAT that is false. NAT sends the VM's traffic out *through the host*, so the VM can reach whatever the host can reach: the home LAN, the office network, and the corporate VPN whenever it is up on the host. The better guides know this and hand it to a manual test, to be repeated with the VPN off and on. Do the test. But a control that exists only as a test is not a control.

The fix lives inside the VM and costs about ten lines of `nftables`: outbound, deny every private address range (10/8, 172.16/12, 192.168/16, plus link-local and CGNAT), allow established connections (so SSH coming in from the host keeps working), and let DNS talk only to public resolvers.

```
#!/usr/sbin/nft -f
flush ruleset
define RESOLVERS  = { 1.1.1.1, 9.9.9.9 }
define PRIVATE_V4 = { 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16, 169.254.0.0/16, 100.64.0.0/10 }
table inet egress {
    chain output {
        type filter hook output priority filter; policy accept;
        oifname "lo" accept
        ct state established,related accept
        udp sport 68 udp dport 67 accept          # DHCP renewal towards the hypervisor's NAT
        ip daddr $RESOLVERS udp dport 53 accept
        ip daddr $RESOLVERS tcp dport 53 accept
        udp dport 53 counter drop
        tcp dport 53 counter drop
        ip daddr $PRIVATE_V4 counter drop
        ip6 daddr fc00::/7 counter drop
    }
}
```

Save it as `/etc/nftables.conf` and `systemctl enable --now nftables`. With this rule in place the VM **cannot reach the host, the LAN, or anything behind a VPN**, by construction, whatever happens to the host's network. Your git server has to be on the internet or be explicitly allowed; if it only lives behind the VPN, now is the time to decide that, rather than discover it.

An objection that always comes up, including from people who know networking: denying 10/8 kills internet access, because the NAT gateway is in that range. It does not. The filter matches the packet's *destination* address, not its next hop: a packet for `1.1.1.1` routed via `10.0.2.2` has destination `1.1.1.1` and passes. The gateway itself becomes unreachable, which is the point. It takes thirty seconds to measure (§6), and it is exactly the kind of mistake a rule enforced by construction prevents and a manual test does not.

So that DNS does not break everything, the VM pins public resolvers at system level instead of using whatever the NAT hands out over DHCP, which is almost always the host's private resolver: a three-line drop-in in `/etc/systemd/resolved.conf.d/` with `DNS=1.1.1.1 9.9.9.9` and `Domains=~.`, then restart `systemd-resolved`. On an existing VM this has to happen before the rules go live, or name resolution stops.

If you want to go further, switch from a blocklist to an allowlist: only the model provider's API, the git server, the package registries. It is safer and more brittle (addresses behind a CDN change, so the set has to be refreshed at boot by resolving names), and it is the natural next step once the blocklist is in place.

### 3.2 Credentials: the most valuable thing in the VM

Inside the VM sits a credential that opens the git server, an SSH key or an HTTPS token. It is the only secret worth anything to whoever hijacks the agent, and if the git server is reachable from the internet, a stolen credential works from anywhere in the world the moment it leaves. Four rules follow.

The credential is dedicated to the VM, generated inside it, and never the one from your laptop. It has a short expiry (git servers let you set one; use it). Where the server allows it, it is a deploy key or a token scoped to a single repository, so it opens one project rather than everything you have access to. And the agent **cannot read it**: `~/.ssh` and `~/.git-credentials` are denied in the agent's configuration (below), and the network rules stop it from shipping the credential out anyway. Together, a successful injection ends up with a credential it cannot read and, even if it could, cannot send anywhere. HTTPS tokens belong in an in-memory credential helper (`cache`), never in `store`, which writes them to disk in plaintext.

There are no other credentials in the VM. No password manager, no authenticated cloud CLI, no keys to other servers, no token that the project does not need. If a task requires one, create it with the smallest possible scope and revoke it when you are done. A key that opens a real server has no business living in the same home directory as a process that reads web pages for a living.

### 3.3 The agent: settings the agent cannot change

The agent has a settings file. If that file lives in the home directory of the user running the agent, a hijacked agent can rewrite it and lift its own restrictions. Claude Code has a managed settings file for exactly this: it overrides every other settings file and the user cannot modify it. On Linux it lives at `/etc/claude-code/managed-settings.json`, owned by root. It is used to deny reads of `~/.ssh`, `~/.git-credentials` and the other folders that hold secrets, and to deny `sudo`, `su`, anything that touches the firewall, and the `systemctl` verbs that change state, while leaving the read-only ones, which you need for diagnosis. Check the rule syntax against the current documentation, it changes often; at the time of writing:

```json
{
  "permissions": {
    "deny": [
      "Read(~/.ssh/**)", "Read(~/.git-credentials)", "Read(/etc/ssh/**)",
      "Read(~/.gnupg/**)", "Read(~/.aws/**)", "Read(~/.netrc)",
      "Edit(~/.ssh/**)", "Edit(~/.git-credentials)", "Edit(/etc/**)",
      "Bash(sudo:*)", "Bash(su:*)", "Bash(nft:*)", "Bash(iptables:*)",
      "Bash(systemctl start:*)", "Bash(systemctl stop:*)", "Bash(systemctl restart:*)",
      "Bash(systemctl enable:*)", "Bash(systemctl disable:*)", "Bash(systemctl mask:*)"
    ]
  }
}
```
 If project-level settings with allowlists already exist, read them first: the managed file wins wherever they conflict, and it is better to know in advance what will stop working.

The managed file alone is not enough if the user launching the agent can become root. So **the user that runs the agent has no `sudo`**. VM administration happens under a different user, and updates are handled by the system. It is the same user segregation you would do on the host, applied inside the VM, where it is cheap: the agent's home, its Python or Node environment, its git credential, and membership of the shared folder's group. All agent sessions run as that one user, and the human works inside those sessions; the admin user is needed about once a month.

The VM is also the only place where relaxing the agent's interactive permissions, the ones that ask for confirmation on every command, makes sense. Outside the VM they stay on. Inside, if the disk gets wrecked, so what: the data lives elsewhere.

## 4. Rules of use

1. Code, the project's technical documentation and synthetic test data go into the VM. Personal or client documents, email, and exports from real systems do not. If a task needs real data, scrub it outside the VM first.
2. One project at a time, or at least one trust domain at a time. Two clients that must not see each other means two VMs (or two users in the same VM, with permissions keeping them apart). Cloning a VM is faster than explaining an incident.
3. The shared folder is empty. When it is not, it is for the duration of a transfer. Nothing in it ever gets executed, in either direction.
4. The git credential is dedicated, expires, and opens a single repository whenever possible.
5. The human pushes, after reading the diff. The agent proposes; the remote receives only what someone has looked at.
6. The VM shuts down cleanly, stays updated, and is destroyed when the project ends. Whatever exists only inside it has a backup outside.
7. After any change to the host's network (a new VPN, a different network) repeat the isolation test (§6), even though the rules in the VM should hold. *Should* is the word that makes people run tests.
8. When the agent does something it was not asked to do that touches the network or the filesystem, stop and read its reasoning before continuing. That is where deleted files get found.

## 5. Variants

**With an IDE on the host.** If you want a real IDE rather than a terminal, keep the VM headless and connect from the host's IDE over SSH (VS Code does this with the Remote-SSH extension, with extensions installed in the remote context): the graphics stay on the host; code, git, terminal and agent stay in the VM. The VM gets lighter and the controls in §3 do not change. The cost is that the IDE has to support this way of working, and not all of them do.

**A container instead of the VM.** On hosts with little memory, or when the IDE does not support remote work, you can confine just the agent's process in a container, with the project folder as its only mount and the same network rules: the IDE stays on the host and works on the same working tree. Anthropic publishes a reference devcontainer for Claude Code with an allowlist firewall script; it is the right pattern. On Windows and macOS the container runs inside a lightweight VM managed by the runtime anyway, so you get VM-grade isolation from the host without administering a VM.

**No VM: users and permissions on the host.** On Linux and macOS you can run the agent under a dedicated user and keep documents under another. It works, and for agents that do not write code (the ones meant to work on documents, where project-level scoping makes no sense) it is often the only option. The price is permission maintenance, which drifts without anyone noticing, and on Windows the multi-user model makes everything clumsy.

**The agent's native sandbox.** Some agents have one, and it is getting better. I would not use it as the only boundary, on grounds of trust: a boundary that lives inside the process it is meant to contain is decided by that process, and I would rather the hypervisor decided. Inside the VM, on top of everything else, it does no harm.

## 6. How to verify

From the VM, with the host's VPN off and then on:

```
ping -c 2 10.0.2.2                    # the NAT gateway: must fail
curl -m 5 http://<router-ip>/         # the LAN: must fail
curl -m 5 https://<internal-service>  # behind the VPN: must fail
curl -m 5 -I https://api.anthropic.com   # must answer
ssh -T git@<git-server>               # must answer
sudo -n true                          # as the agent user: must fail
```

Then, inside an agent session: ask it to read `~/.ssh/config` and to make an HTTP request to a private address. The right answer is that it cannot, and it tells you why. If it can, there is no boundary.

`nft list ruleset` shows the counters on the drop rules: if they climb while nobody is doing anything unusual, something inside the VM is trying to get out.

On a VM that is already in use, bring the network rules up with a safety net: a background timer that flushes the ruleset after three minutes unless someone stops it. If the tests pass, stop the timer and enable the service; if something is off, the VM cleans itself up.

## 7. Provisioning

Rebuilding the VM by hand every time is why people stop using one. Everything described above (two users with separated privileges, packages, public resolvers, the network rules, automatic updates, the agent's managed settings, the backup timer) fits in a single `cloud-init` file that any Ubuntu cloud image reads on first boot. With Multipass, on Windows, macOS and Linux, that is one command:

```
multipass launch --name agentvm --cpus 2 --memory 8G --disk 60G --cloud-init agentvm.yaml
```

With Hyper-V, UTM or VirtualBox, attach the same file as a seed image or use Ubuntu Server's autoinstall. Write the file once, keep it in version control, and the VM stops being something you maintain and becomes something you regenerate. On VirtualBox, install the Guest Additions with DKMS already present, so the kernel modules rebuild themselves after every kernel update and the shared folder does not vanish. On an existing VM that works, leave it alone: the manual fix is one line, and a rare risk with a known remedy beats a preventive change to a healthy system.

## 8. What the VM does not solve

Whatever the agent reads goes to the model provider. The VM decides *what* it can read, not where it ends up: the plan you are on, data retention and whether your data trains anything are a contract, not a configuration, and they are worth reading before a client's code goes in.

The provider can suspend your account, and it happens without warning and without appeals being answered. Keep a second account already active and lightly used: if the first one goes, you switch accounts instead of stopping work. If your work depends on agents, this redundancy is worth as much as a second internet line.

The agent itself is a channel. An injection can ask it to *summarize* a file in its reply, and no firewall stops that: the content travels through the model, where no network rule reaches. That is why the rule about what goes into the VM (§4, item 1) is not advice.

And the code inside the VM stays readable by a hijacked agent. A short-lived credential and one project at a time reduce the damage; they do not remove it. If you have code under NDA, that goes into the decision of how many VMs to run.

---

*If you have a better setup, or a way this one breaks, tell me: open an issue.*
