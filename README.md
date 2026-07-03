## 🗺️ The Story
 
These labs are not isolated exercises — they're **one continuous story**:
 
> The **Maps team** hands you their first app, `mapit`. You deploy it and expose it to the world. Then they need a database — so you hand it to an Operator. Then the data needs to survive — storage. Then they ship an image that runs as root — the platform catches it, you fix it *right*. Then their developers want to log in themselves — identity, RBAC, budgets. And finally... a second team arrives, and you onboard them end-to-end. 🏁
 
Every lab picks up where the previous one ended.
**⚠️ Which means: don't delete anything between labs** — your projects and apps carry the story forward.
 
<br>
---
 
## 📚 What's in Here
 
| Day | Theme | Labs |
|-----|-------|------|
| **Day 1** | Foundations — containers, K8s, architecture, cluster installation | [`Labs-day-1/`](Labs-day-1/) *(runs in the Workshop Lab Interface)* |
| **Day 2** | Working with the cluster | [`A — Our First App`](Labs-day-2/Day2_A_our_first_app.md) · [`B — Operators`](Labs-day-2/Day2_B_Operators.md) · [`C — Storage`](Labs-day-2/Day2_C_Storage.md) · [`D — Security (SCC)`](Labs-day-2/Day2_D_Security.md) |
| **Day 3** | Running it as a service | [`A — Auth, RBAC & Budgets`](Labs-day-3/Lab_3_A.md) · [`🏁 Final Mission`](Labs-day-3/Lab_3_Final.md) |
| **Anytime** | Commands you'll actually use | [`⚡ Admin Commands Kit`](Admin_commands_kit.md) |
 
<br>
---
 
## 🧭 The Journey, Day by Day
 
### Day 1 — A New Era for Applications
Microservices, VMs vs containers, what Kubernetes is and what OpenShift adds on top. Cluster architecture — a dive into the engine. Installation types, and a lab: **install a cluster**. Meet the ClusterOperators that run the platform.
 
### Day 2 — Working with the Cluster
`oc`, the Web Console, and the four objects behind every app (Pod → Deployment → Service → Route). Then the big idea of the day: **Operators** — you install one and watch it build and heal a PostgreSQL cluster. **Storage** — where data actually lives, and proving it survives pod death. **SCC** — why containers don't run as root here, and how to fix the most common breakage you'll ever meet.
 
### Day 3 — Running It as a Service
Real humans log in: **IdP, Users & Groups, RBAC** (`view` / `edit` / `admin`). **Budgets**: requests & limits, LimitRanges, ResourceQuotas, NetworkPolicies. Under the hood: **nodes** — MachineSets, drain/cordon, taints for dedicated hardware. Day-2 ops: upgrades, etcd backup, `must-gather`, `oc debug`. And the finale: **Team Weather goes live** — everything from 3 days, one mission, and you leave with a survival kit.
 
<br>
---
 
## ✅ Prerequisites
 
Each participant gets:
 
* 🖥️ A personal **OpenShift 4.20** cluster (yes, your own — break it with confidence)
* 🔑 `kubeadmin` credentials
* 💻 A terminal with `oc` installed (`htpasswd` needed on Day 3 → `sudo dnf install httpd-tools`)
* 🌐 Access to the **Web Console** — we use it a lot, on purpose
No prior OpenShift experience assumed. Coming from Kubernetes? Your knowledge transfers — we'll show you exactly where OpenShift adds. Coming from sysadmin world? Every lab includes analogies from home. 🏠
 
<br>
---
 
## 🛠️ How to Work Through the Labs
 
1. **In order.** The story (and the objects on your cluster) build on each other.
2. **Type or paste — both fine.** Understanding the output matters more than typing the input.
3. Look for the recurring signposts:
   * 🛠️ **Exercise** — your hands on the keyboard
   * 💡 **Administrative Perspective** — the "why" behind the "what"
   * ✅ **Checkpoint** — verify before moving on
   * 📸 **Console** — switch to the browser; the UI is a first-class tool here
4. Stuck? `oc get co` first (you'll learn why), events second, then ask. That habit **is** the workshop.
<br>
---
 
## 🎁 What You Leave With
 
By the end of Day 3 you'll have committed your own **Day-One Kit** — three files you'll actually open at work:
 
* 📘 `onboarding-playbook.md` — checklist for landing a new team on the platform
* ⚡ `cheatsheet.md` — the ~20 commands of daily platform life
* 🚑 `first-15-minutes.md` — the incident runbook. *Print it. Tape it to the wall.*
<br>
---
 
> 🌅 **And after the workshop?** Everything you'll build here is YAML and commands — which means it can live in Git and apply itself. That's **GitOps**, and it's exactly where this journey continues.
 
*Built with ❤️ by Red Hat. Questions, fixes, or war stories from running it — issues and PRs welcome.*
