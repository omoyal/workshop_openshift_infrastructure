# Module: SCC & RBAC — Who May Do What (and as Whom)

> 🗺️ **The story so far:** Your platform now runs the Maps team's app, a self-healing PostgreSQL cluster, and storage that survives anything. Then, at 15:00, the Maps team ships **v2 of their web frontend** — a stock `nginx` image straight from Docker Hub. They tested it on a laptop. It works on their machine. 🙃
>
> It will **not** work on your cluster — and by the end of this module you'll know exactly why, why that's a *good* thing, and how to fix it the right way.

---

## 📑 Core Concept: Security Context Constraints (SCC)

On plain Kubernetes, a container that wants to run as **root** — runs as root. On OpenShift, every pod must pass through a gatekeeper first: the **Security Context Constraint**.

* An SCC defines what a pod is **allowed to request**: which UID it may run as, which volumes it may use, which capabilities it may hold.
* The default SCC, **`restricted-v2`**, says among other things: *no root, run as a random high UID, drop dangerous capabilities.*
* Pods don't get SCCs directly — they get them through their **ServiceAccount** (more on that in a minute).

> 💡 **Administrative Perspective:**
> This is the #1 difference people hit when coming from vanilla Kubernetes or a laptop: *"my image works everywhere but crashes on OpenShift."* It's not a bug — it's the platform refusing to run software as root on your nodes. In a multi-tenant cluster, that default protects every team from every other team.

---

## 💥 Watch It Break

#### 🛠️ Exercise: Deploy the Maps Team's v2 Image

```bash
oc project app-management
oc new-app --name=maps-web docker.io/library/nginx
```

Now watch it fail:

```bash
oc get pods -l deployment=maps-web
```

**Expected Output:**
> The pod lands in `CrashLoopBackOff` (give it a few seconds and re-run the command).

Read the actual error — always the first move:

```bash
oc logs deployment/maps-web
```

**Expected Output:**
> Permission errors, e.g. `mkdir() "/var/cache/nginx/..." failed (13: Permission denied)`

The official nginx image assumes it's running as root. OpenShift ran it as a random unprivileged UID — and nginx couldn't write to its own directories.

Check which SCC the pod actually received:

```bash
oc get pods -l deployment=maps-web \
  -o jsonpath='{.items[0].metadata.annotations.openshift\.io/scc}{"\n"}'
```

**Expected Output:**
> `restricted-v2`

> 💡 **Administrative Perspective:**
> The **best** fix is a rootless image (e.g., `nginxinc/nginx-unprivileged`) — and that's the answer you should push dev teams toward. But sometimes it's a vendor image you can't change. For that, OpenShift gives you a **controlled exception** — scoped to one identity, in one project. Never cluster-wide.

---

## 🪪 Core Concept: ServiceAccounts

A **ServiceAccount (SA)** is an identity for *software*, the way a user account is an identity for a human. Every pod runs **as** a ServiceAccount (by default, one named `default`).

This is the key to a scoped exception: instead of loosening security for the whole project, we create a **dedicated identity** for this one workload and attach the permission to it.

#### 🛠️ Exercise: The Controlled Fix

Create a dedicated ServiceAccount:

```bash
oc create serviceaccount maps-web-sa
```

Grant **that identity only** the `anyuid` SCC (allows running with the image's own UID):

```bash
oc adm policy add-scc-to-user anyuid -z maps-web-sa
```

> ℹ️ The `-z` flag means "a ServiceAccount in the current project" — no need to type the full name.

Tell the deployment to run as that ServiceAccount (this triggers a new rollout):

```bash
oc set serviceaccount deployment/maps-web maps-web-sa
```

Verify the recovery:

```bash
oc get pods -l deployment=maps-web
```

**Expected Output:**
> A new pod, `Running`, `1/1 Ready`.

Confirm it received the right SCC this time:

```bash
oc get pods -l deployment=maps-web \
  -o jsonpath='{.items[0].metadata.annotations.openshift\.io/scc}{"\n"}'
```

**Expected Output:**
> `anyuid`

> 🏆 **Notice what you did NOT do:** you didn't disable security, didn't touch other workloads, didn't grant anything cluster-wide. One identity, one permission, one project. That's the pattern.

---

## 📑 Core Concept: RBAC — Roles & RoleBindings

That `add-scc-to-user` command you just ran? It quietly created a **RoleBinding**. Time to meet the mechanism behind *all* permissions in OpenShift:

* **Role:** a named list of allowed verbs on resources (*"can get/list/create pods"*), scoped to one project. A **ClusterRole** is the same, cluster-wide.
* **RoleBinding:** attaches a Role to an identity — a user, a group, or a ServiceAccount — within a project.
* **Identity + Role + Binding = permission.** Nothing else grants access.

OpenShift ships three built-in roles you'll use constantly as a platform team:

| Role | Grants |
|------|--------|
| `view`  | Read everything in a project, change nothing |
| `edit`  | Create and modify workloads, but not permissions |
| `admin` | Everything in the project, including granting access to others |

#### 🛠️ Exercise: Grant Access Like a Platform Team

The Maps team wants a **CI bot** that can deploy to the project — software, so: a ServiceAccount.

```bash
oc create serviceaccount maps-ci
oc adm policy add-role-to-user edit -z maps-ci
```

See the binding you just created:

```bash
oc get rolebindings | grep maps-ci
```

#### 🛠️ Exercise: Test Permissions Without Being the Bot

`oc auth can-i` lets you ask permission questions **as any identity** — one of the most useful troubleshooting commands you'll own:

```bash
oc auth can-i create deployments \
  --as=system:serviceaccount:app-management:maps-ci -n app-management
```

**Expected Output:** `yes`

```bash
oc auth can-i delete rolebindings \
  --as=system:serviceaccount:app-management:maps-ci -n app-management
```

**Expected Output:** `no` — `edit` can ship apps, but it can **not** hand out permissions. That separation is the whole point.

#### 🛠️ Exercise: See It in the Console

1. Navigate to **User Management → RoleBindings** (project `app-management`).
2. Find the bindings for `maps-web-sa` and `maps-ci` — every permission you granted today is visible, auditable, and deletable right here.

---

## 🧭 Wrap-Up — Module & Day

**This module:**
* OpenShift refuses root by default — **`restricted-v2`** guards every pod, and the crash you saw is the platform working as designed.
* The clean exception: dedicated **ServiceAccount** → grant an SCC **to that identity only** → assign it to the workload.
* **RBAC** = Role + Binding + Identity. `view` / `edit` / `admin` cover 90% of daily grants, and `oc auth can-i --as=...` answers every "why can't X do Y?" ticket.

**The day, in one story:** you deployed the Maps team's app and exposed it to the world 🚀 → handed its database to an Operator that survives failures ⚙️ → gave its data a home that outlives any pod 💾 → and when they shipped a root image, the platform caught it and you fixed it *with precision*, not with a sledgehammer 🔒.

> 🌅 **Tomorrow:** one thing was missing today — you did everything as `kubeadmin`, and permissions went to software identities. Real clusters have real humans: the Maps team's developers want to log in **themselves**. Tomorrow morning we connect an Identity Provider and turn today's RBAC into a full onboarding flow. Same time, same cluster — **don't delete anything.** 👋
