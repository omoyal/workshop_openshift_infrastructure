# Module: Real Humans, Real Rules - Auth, RBAC & Budgets

<br>

> 🗺️ **The story so far:** Everything until now was done as `kubeadmin` . Today the Maps team gets what every real team needs: **their own logins, their own permissions, and a budget they can't blow through.**

> 💡 **Tip:** open **two terminals** — Terminal A stays `kubeadmin`, Terminal B will become Dana. It saves constant re-logins.

---

<br><br>

## 🔑 Part 1 — OAuth: Letting Humans In

OpenShift's built-in **OAuth server** doesn't store passwords — it delegates to an **Identity Provider**. Production uses LDAP/OIDC; today we use the simplest one: an **htpasswd file**.

<br>

#### 🛠️ Exercise: Create the Users File

In **Terminal A**:

```bash
htpasswd -c -B -b users.htpasswd dana redhat123
htpasswd -b users.htpasswd yossi redhat123
```

<br>

> ℹ️ Missing `htpasswd`? → `sudo dnf install httpd-tools`


<br>

#### 🛠️ Exercise: Configure the IdP (Web Console)

The console has a guided form for exactly this:

1. Log in to the console as `kubeadmin`.
2. Navigate to **Administration → Cluster Settings → Configuration** tab.
3. Find and click **OAuth**.
4. Under **Identity providers**, click **Add → HTPasswd**.
5. Name it `workshop-htpasswd`, upload (or paste) your `users.htpasswd` file, and click **Add**.

<br>

Behind the scenes this created a Secret and patched the `oauth/cluster` resource — the same thing a long `oc patch` would do. An Operator now rolls it out; watch it from Terminal A:

```bash
oc get co authentication -w
```

Wait until `PROGRESSING` returns to `False` (~2–3 min), then `Ctrl+C`.

<br>

#### 🛠️ Exercise: First Login

In **Terminal B**:

```bash
oc login -u dana -p redhat123 "$(oc whoami --show-server)"
```

**Expected:** `Login successful.` — followed by `You don't have any projects.`

<br>

> 💡 **Authentication ≠ Authorization.** Dana passed "who are you?" and hit "what may you do?" — which is currently: nothing. Her `User` object was auto-created on first login (`oc get users` in Terminal A).

---
<br><br>

## 👥 Part 2 — Groups & RBAC

**House rule: permissions go to Groups, never to individuals.** Dana leaves, Yossi joins — you edit one group.

<br>

#### 🛠️ Exercise: Team + Permissions (3 commands)

In **Terminal A**:

```bash
oc adm groups new maps-devs dana yossi
oc adm policy add-role-to-group edit maps-devs -n app-management
oc adm policy add-role-to-user admin dana -n app-management
```

The team can deploy (`edit`); Dana, as team lead, also owns the project (`admin`).


<br>

#### 🛠️ Exercise: Feel the Boundaries

In **Terminal B** (Dana):

```bash
oc auth can-i create deployments -n app-management     # yes
oc auth can-i create rolebindings -n app-management    # yes — she's project admin
oc auth can-i create projects                          # no
oc get nodes                                           # Forbidden
```
<br>

> 💡 **The three levels of "admin":**
> `admin` **in a namespace** — owns one project (Dana now). · `cluster-admin` — owns everything incl. nodes (`kubeadmin`; grant to almost no one). · And `edit` (Yossi) ships code but can't touch permissions at all.
>
> 👀 See it all in the console: **User Management → RoleBindings**, project `app-management`.

---
<br><br>

## ⚖️ Part 3 — Requests & Limits

Two numbers run everything: **request** = the scheduler's guaranteed reservation; **limit** = the hard ceiling (memory → OOMKilled, CPU → throttled).

<br>

#### 🛠️ Exercise: Give mapit Its Numbers (Web Console)

Dana can do this herself — in the **console, logged in as dana**:

1. Navigate to **Workloads → Deployments** in project `app-management`, click `mapit`.
2. From the **Actions** menu, select **Edit resource limits**.
3. Fill in:
   * **Request:** CPU `100` millicores, Memory `128` Mi
   * **Limit:** CPU `500` millicores, Memory `256` Mi
4. **Save** — a new rollout starts.

<br>

Verify from **Terminal B** — including what Kubernetes thinks of the pod's honesty:

```bash
oc get pods -l deployment=mapit -o jsonpath='{.items[0].status.qosClass}{"\n"}'
```

**Expected:** `Burstable` — requests set, limits higher. (Remember the quiz: who dies first?)

---

<br><br>

## 💰 Part 4 — ResourceQuota: The Team's Budget

Limits guard each container; a **quota** caps the whole project. That's a platform-team move — **Terminal A**:

```bash
oc create quota maps-budget \
  --hard=requests.cpu=1,requests.memory=1Gi,pods=6 \
  -n app-management
```

<br>

#### 🛠️ Exercise: Watch Dana Hit the Wall

In **Terminal B**, scale like it's Black Friday:

```bash
oc scale deployment/mapit --replicas=10
oc get pods -l deployment=mapit
```

Not 10 pods. Why? Read it like you'll read it in tickets:

```bash
oc get events --sort-by=.lastTimestamp | grep -i quota | tail -3
```

**Expected:** `exceeded quota: maps-budget ... used: pods=6, limited: pods=6`

<br>

📸 Now the good part — in the **console**: **Administration → ResourceQuotas → maps-budget**. The usage gauges are pinned at max. This page is where devs will *see* their budget instead of asking you.

<br>

Clean up the ambition:

```bash
oc scale deployment/mapit --replicas=2
```

> 💡 Notice **how** the quota enforced itself: nothing crashed. The API politely refused pod #7, and the Deployment waits. That's why every project should be born with a quota.

---

<br><br>

## 🧭 Wrap-Up

* **OAuth + IdP** — configured in a console form; the auth Operator rolled it out. Passwords never live in OpenShift.
* **Groups get permissions**, people get group membership. `edit` for the team, `admin` for the owner, `cluster-admin` for almost nobody.
* **`oc auth can-i`** answers every "why can't I?" — and **requests/limits + quota** make sharing the cluster drama-free, failing politely.

> ⚠️ **Keep the users, group, and quota — they star in the final exercise.**
> After lunch: all these rules live *on top of* the nodes. Time to look underneath. 🖥️
