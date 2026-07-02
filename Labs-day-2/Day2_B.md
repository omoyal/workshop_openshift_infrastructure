# Module: Operators — Letting the Cluster Run the Hard Stuff

> 🗺️ **The story so far:** The Maps team's `mapit` application is live — you deployed it, exposed it, and even watched OpenShift resurrect a deleted Pod. Word got around. Now the Maps team is back with a new request: *"We need a PostgreSQL database for the next version. Oh, and it needs to be highly available. And it should survive failures. And be upgraded safely."*
>
> You are **not** going to babysit a database by hand. In this module you'll make the cluster do it — with an **Operator**.

---

## 📑 Core Concept: What Is an Operator?

An **Operator** is a method of packaging, deploying, and managing an application by extending Kubernetes itself. It has two parts:

* **Custom Resource Definition (CRD):** A new object type added to the cluster (e.g., `Cluster`, `Kafka`). It lets you declare **WHAT** you want in plain YAML.
* **Controller:** A Pod running in the cluster that watches those objects and knows **HOW** to make them real — and keep them that way, forever.

> 💡 **Administrative Perspective:**
> An Operator is a human operator's knowledge — the runbooks, the failover procedures, the 3am recovery steps — encoded into software that never sleeps. It runs the same *reconcile loop* you already saw with Deployments (watch → compare → act), just applied to much more complex software.

---

## ⚙️ The Platform Itself Runs on Operators

Before installing anything new, meet the Operators you already have. Every OpenShift component — the web console, the ingress router, monitoring, authentication — is managed by a **ClusterOperator**.

#### 🛠️ Exercise: Meet Your ClusterOperators

```bash
oc get clusteroperators
```

You should see roughly 30 operators, all showing `AVAILABLE: True` and `DEGRADED: False`.

Now see the same thing in the **Web Console**:

1. Log in to the web console as `kubeadmin`.
2. Navigate to **Administration → Cluster Settings**.
3. Select the **ClusterOperators** tab.

> 💡 **Administrative Perspective:**
> `oc get co` (or this console page) is your new "is the cluster healthy?" check — the first place to look when something feels wrong. A `Degraded: True` operator will usually tell you in its message exactly which component is broken.

---

## 🏪 The Software Catalog — The Cluster's App Store

The **Software Catalog** (found under **Ecosystem** in the console) is the built-in catalog of Operators, Helm charts, and other content you can add to the cluster: databases, storage, messaging, GitOps, and hundreds more. The same pattern that runs the platform is available for everything you build on top of it.

> 🌍 **Trust levels matter:** Operators are labeled **Red Hat**, **Certified**, or **Community**. Check the badge before you trust one with production data.

#### 🛠️ Exercise: Explore the Software Catalog (Console)

1. In the left navigation, click **Ecosystem → Software Catalog**.
2. Under **Type**, check **Operators** — notice how many exist.
3. In the **Filter by keyword** box, type: `cloudnative`
4. Select **CloudNativePG** (a community Operator for PostgreSQL). If a warning about Community Operators appears, click **Continue**.
5. Read its description page — this is the "packaging" of an Operator: who provides it, which update channels exist, what it manages.

---

## 📥 Installing an Operator

Installation is a few clicks. Behind the scenes, **OLM** (Operator Lifecycle Manager — itself an Operator, of course) handles the install, dependencies, and future upgrades through the channel you pick.

#### 🛠️ Exercise: Install the Operator (Console)

1. On the **CloudNativePG** page, click **Install**.
2. Keep the defaults:
   * **Update channel:** the latest `stable` channel offered
   * **Installation mode:** All namespaces on the cluster
   * **Installed Namespace:** `openshift-operators`
   * **Update approval:** Automatic
3. Click **Install** and wait for *"Installed operator: ready for use"*.
4. Navigate to **Ecosystem → Installed Operators** and verify **CloudNativePG** shows a status of **Succeeded**.

#### 🛠️ Exercise: Verify from the CLI

The console clicks created real API objects. Prove it:

```bash
oc get pods -n openshift-operators
```

**Expected Output:**
> A pod named `cnpg-controller-manager-...` in `Running` state — this is the **controller**, now watching your cluster 24/7.

The Operator also taught your cluster a new word. Check that a new resource type exists:

```bash
oc api-resources | grep cnpg
```

Your cluster now understands what a PostgreSQL `Cluster` is. It didn't a minute ago. 🎓

---

## 🗄️ Creating the Database — Declaratively

Time to deliver the Maps team their database. You will not install PostgreSQL, configure replication, or write failover scripts. You will **declare** a database, and the Operator will do the rest.

#### 🛠️ Exercise: Create the Custom Resource

Make sure you are in the project from the previous module:

```bash
oc project app-management
```

Create a file named `maps-db.yaml`:

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: maps-db
spec:
  instances: 3
  storage:
    size: 1Gi
```

Eight lines. That's the entire request: *"a 3-instance PostgreSQL cluster with 1Gi of storage each."* Apply it:

```bash
oc apply -f maps-db.yaml
```

#### 🛠️ Exercise: Watch the Operator Work

```bash
oc get pods -w
```

Over the next minute or two you'll see the Operator create `maps-db-1`, `maps-db-2`, `maps-db-3` — initializing the primary, then joining the replicas.
(Press `Ctrl+C` to stop watching.)

Check the cluster's status the way the Operator reports it:

```bash
oc get clusters.postgresql.cnpg.io maps-db
```

**Expected Output:**
> `STATUS: Cluster in healthy state`, with 3 ready instances and a primary elected.

#### 🛠️ Exercise: See It in the Console

1. Navigate to **Workloads → Topology** and make sure the `app-management` project is selected at the top.
2. You should see `mapit` **and** the `maps-db` pods side by side — the whole story of the day in one picture.
3. Navigate to **Ecosystem → Installed Operators → CloudNativePG**, open the **Cluster** tab, and click `maps-db` — the console renders your Custom Resource with live status, conditions, and events.

> 💡 **Administrative Perspective:**
> Notice what the Operator created for you: Pods, Services, Secrets, and **PersistentVolumeClaims** — run `oc get pvc` for a sneak peek. Where does that storage actually come from? Hold that thought until this afternoon. 😉

---

## 💥 The Payoff: Break It and Watch It Heal

A Deployment can restart a stateless Pod. But a database **primary** is a different beast — losing it means electing a new primary, repointing replicas, updating endpoints. That's expert knowledge. Let's see if our new "expert" has it.

#### 🛠️ Exercise: Kill the Primary

Find the current primary:

```bash
oc get clusters.postgresql.cnpg.io maps-db -o jsonpath='{.status.currentPrimary}{"\n"}'
```

Now delete it (yes, really):

```bash
oc delete pod $(oc get clusters.postgresql.cnpg.io maps-db -o jsonpath='{.status.currentPrimary}')
```

Watch the recovery live:

```bash
oc get pods -w
```

Within seconds, the Operator promotes a replica to primary and rebuilds the missing instance. Verify:

```bash
oc get clusters.postgresql.cnpg.io maps-db -o jsonpath='{.status.currentPrimary}{"\n"}'
```

**Expected Output:**
> A **different** pod is now the primary, and the cluster returns to `Cluster in healthy state`.

> 🏆 **What you just handled with one command is what used to be a 3am phone call.** That is the Operator pattern.

---

## 🧭 Wrap-Up

* An Operator = a **CRD** (what you want) + a **Controller** (how to keep it real).
* OpenShift itself is ~30 **ClusterOperators** — `oc get co` is your cluster health dashboard.
* The **Software Catalog** (Ecosystem) brings the same pattern to everything you add — installed in clicks, verified in the CLI.
* You declared a 3-node PostgreSQL cluster in 8 lines of YAML and it **survived losing its primary**.

> ⚠️ **Do NOT delete the `app-management` project or the `maps-db` cluster!**
> This afternoon we find out where its disks actually came from — and what happens to the Maps team's data when Pods die. See you at the Storage module. 🗄️
