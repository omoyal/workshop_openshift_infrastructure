# Module: Storage — Where the Data Actually Lives

> 🗺️ **The story so far:** This morning you gave the Maps team a 3-node PostgreSQL cluster, and it even survived losing its primary. But during lunch, a suspicious developer asked a fair question: *"Wait — pods are ephemeral. You told us so yourselves. So when a database pod dies... where is our data?"*
>
> Time to answer properly. In this module you'll discover the storage your Operator quietly created this morning, understand how OpenShift provisions disks on demand, and then give `mapit` itself a disk that outlives its pods.

---

## 📑 Core Concepts: The Storage Triangle

Three objects, one clean separation of concerns:

* **PersistentVolume (PV):** The actual piece of storage — a disk, an NFS export, a cloud volume. Cluster-scoped. Usually you never create these by hand.
* **PersistentVolumeClaim (PVC):** A **request** for storage made by an application: *"I need 1Gi, read-write."* Lives inside a Project. This is the object you'll work with daily.
* **StorageClass:** The **menu** of storage types the cluster offers (fast SSD, cheap bulk, file share...). A PVC points at a class; the class knows how to provision a matching PV.

> 💡 **Administrative Perspective:**
> This is **dynamic provisioning**: a PVC arrives → the StorageClass's provisioner creates a PV on the fly → the two are **Bound**. Nobody files a storage ticket, nobody carves a LUN by hand. The "storage admin" is now a piece of software — sound familiar? It's usually an Operator. 😏

**Access modes** (you'll see these on every PVC):

| Mode | Meaning | Typical use |
|------|---------|-------------|
| `RWO` (ReadWriteOnce) | One node can mount it read-write | Databases, most apps |
| `RWX` (ReadWriteMany) | Many nodes can mount it read-write | Shared file storage |
| `ROX` (ReadOnlyMany) | Many nodes, read-only | Shared static content |

---

## 🔍 Solving the Morning Mystery

This morning, your 8-line YAML asked for `storage: size: 1Gi` — and the Operator took care of the rest. Let's see what "the rest" actually was.

#### 🛠️ Exercise: Find the Database's Claims

```bash
oc project app-management
oc get pvc
```

**Expected Output:**
> Three PVCs — `maps-db-1`, `maps-db-2`, `maps-db-3` — all `Bound`, 1Gi each. One claim per database instance.

Inspect one of them:

```bash
oc describe pvc maps-db-1
```

Note three things in the output: the **StorageClass** used, the **Access Mode** (`RWO` — it's a database), and **Used By** (the pod mounting it).

#### 🛠️ Exercise: Follow the Claim to the Real Disk

Every `Bound` PVC has a PV behind it. PVs are cluster-scoped — see them all:

```bash
oc get pv
```

Find the rows whose `CLAIM` column says `app-management/maps-db-...` — those are the actual disks that were created *for you*, on demand, this morning.

#### 🛠️ Exercise: Read the Menu

```bash
oc get storageclass
```

Note which class is marked **`(default)`** — that's what any PVC gets when it doesn't ask for a specific class. That's how the Operator's PVCs got disks without ever naming a StorageClass.

Now the same picture in the **Web Console**:

1. Navigate to **Storage → PersistentVolumeClaims** (make sure project `app-management` is selected at the top).
2. Click `maps-db-1` — the console shows capacity, status, access mode, and the bound PV, all in one page.
3. Navigate to **Storage → StorageClasses** and find your default class.

> 💡 **Administrative Perspective:**
> Mystery solved: **PVC created by the Operator → default StorageClass → provisioner created a PV → Bound.** As the platform team, the StorageClasses page is *your* responsibility — it defines what storage your developers can ask for.

---

## 💾 Giving mapit a Disk of Its Own

The Maps team's next feature lets users upload map files. `mapit`'s filesystem is **ephemeral** — pod dies, files die. Let's fix that with one command.

#### 🛠️ Exercise: Add a Volume to a Running App

```bash
oc set volume deployment/mapit --add \
  --name=mapit-data \
  --type=persistentVolumeClaim \
  --claim-name=mapit-data \
  --claim-size=1Gi \
  --mount-path=/data
```

This single command did two things: **created a PVC** named `mapit-data` and **mounted it** at `/data` in the deployment (which triggers a new rollout).

Verify:

```bash
oc get pvc mapit-data
```

> ℹ️ If the status shows `Pending` for a moment — that's normal. Many StorageClasses wait for the first consumer pod to be scheduled before provisioning (`WaitForFirstConsumer`). Once the new `mapit` pod starts, it becomes `Bound`.

```bash
oc get pods
```

Wait until the new `mapit` pod is `Running`.

---

## 💥 The Test: Kill the Pod, Keep the Data

#### 🛠️ Exercise: Write, Destroy, Verify

Write a file onto the persistent volume:

```bash
oc exec deployment/mapit -- sh -c 'echo "The maps survived!" > /data/proof.txt'
```

Confirm it's there:

```bash
oc exec deployment/mapit -- cat /data/proof.txt
```

Now destroy the pod:

```bash
oc delete pod -l deployment=mapit
```

Wait for the replacement pod to reach `Running`:

```bash
oc get pods
```

The new pod is a completely different pod — new name, new IP, empty container filesystem. Except:

```bash
oc exec deployment/mapit -- cat /data/proof.txt
```

**Expected Output:**
> `The maps survived!`

> 🏆 **The pod is cattle. The claim is the pet.** Pods come and go; the PVC — and the data on it — stays. This is the exact mechanism protecting the Maps team's database.

#### 🛠️ Exercise (Bonus, if time allows): See It in the Topology

1. Navigate to **Workloads → Topology** in project `app-management`.
2. Click the `mapit` node and open the **Resources** tab in the side panel — the PVC is now listed as part of the application.

---

## 🧭 Wrap-Up

* **PVC** = the request, **PV** = the disk, **StorageClass** = the menu. Dynamic provisioning connects them with zero tickets.
* The Operator's `storage: 1Gi` this morning silently became 3 PVCs on the **default StorageClass** — now you know how to trace every step of it.
* `oc set volume` gave a running app persistent storage in one command — and the data survived the pod's death.
* Access modes matter: databases want `RWO`; shared file use-cases need `RWX` — check what your StorageClasses support before promising it to a team.

> ⚠️ **Keep everything running — we're not done with the Maps team.**
> Right after the break, they're shipping a **new version of their image**... and it insists on running as **root**. OpenShift is going to have opinions about that. See you at the SCC module. 🔒
