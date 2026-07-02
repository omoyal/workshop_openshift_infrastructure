# Lab 2: Governance & Grit – Users, RBAC, SCC, and Storage 🔐💾

Welcome back, Admin! Our `mapit` application from Lab 1 is running great inside the `app-management` project. However, right now, it’s wide open, and if the pod restarts, any data inside it is wiped out. 

Today, we will secure the project, learn how OpenShift manages cluster identities, and attach persistent storage to our app.

---

## 🧠 Concept Refreshers: Today's Building Blocks

> **1. Cluster Identity (OAuth & Users):** OpenShift uses an OAuth layer to authenticate users. Once a user logs in, OpenShift creates a native `User` and `Group` object.
>
> **2. RBAC (Roles & Bindings):** Defines who can do what inside our project. Instead of managing individual users, we always map permissions to a **Group**.
>
> **3. SCC (Security Context Constraints):** The ultimate Linux kernel guardrail. It controls what permissions the *container* has on the actual host node.
>
> **4. Storage (PVC & StorageClass):** Containers are ephemeral (they forget everything when they die). A **PVC (Persistent Volume Claim)** is a request for a permanent hard drive, which a **StorageClass** automatically provisions for us.

---

## 👥 Part 1: Managing the Team (Users & RBAC via Console)

Instead of writing massive YAML files for permissions, let's use the CLI to quickly create an organizational group, and then use the **Web Console** to grant them access.

### 1. Create a Group and a User via CLI:
```bash
# Create a group for our developers
oc adm groups new alpha-developers

# Add a simulated user named 'clara' into the group
oc adm groups add-users alpha-developers clara
```

### 🖥️ Console Check #1: Granting Access Visually
1. Open your OpenShift Web Console as an **Administrator**.
2. Go to **User Management** 👈 on the left menu, and click on **Groups**.
3. Verify that `alpha-developers` exists and has `clara` inside it.
4. Now, switch to the **Developer Perspective** and select the `app-management` project.
5. Click on **Project** -> **Project Access** -> **Add Role Binding**.
6. Set the following fields:
   * **Select Role:** `edit` (Allows them to deploy apps but not delete the project).
   * **Subject Kind:** `Group`
   * **Name:** `alpha-developers`
7. Click **Save**. 

🎉 *Congratulations! Anyone added to this group can now manage apps in this project.*

### 2. Test Clara's Powers (Impersonation):
Let's verify Clara can work here, but cannot touch global cluster settings:
```bash
# Can Clara create pods here?
oc auth can-i create pods -n app-management --as=clara
# (Expected: yes)

# Can Clara see the cluster hardware nodes?
oc auth can-i get nodes --as=clara
# (Expected: no)
```

---

## 💾 Part 2: Giving our App a Memory (Persistent Storage)

Right now, our `mapit` app saves everything in memory. If the node dies, the data dies. Let's attach a permanent disk drive to it. 

Instead of writing a long PVC YAML and modifying the deployment manually, we will use an amazing OpenShift admin shortcut command: `oc set volume`.

### 1. Attack a 1-Gigabyte Disk to the App:
```bash
oc set volume deployment/mapit --add --name=mapit-storage --type=pvc --claim-size=1Gi --mount-path=/app/data
```
> 💡 **What just happened behind the scenes?** This single command created a **PVC**, talked to the cluster's default **StorageClass**, automatically provisioned a real **Persistent Volume (PV)** from the infrastructure, and mounted it inside the container at `/app/data`!

### 🖥️ Console Check #2: Watch the Storage Magic
1. Go back to the **Topology** view in the Web Console.
2. Click on the `mapit` circle. In the side-panel, click on the **Resources** tab.
3. Scroll down to the **Pods** section. Notice that a new pod is spinning up while the old one is terminating. OpenShift is doing a safe rolling update to apply the disk!
4. Go to **Storage** (in the left menu) -> **Persistent Volume Claims**.
5. You will see a PVC named `mapit-storage` with a status of **Bound**. OpenShift dynamically allocated the storage without you needing to provision hardware manually.

---

## 🛡️ Part 3: Inspecting the Shield (SCC)

Even though Clara can edit the project, and the app has a disk, OpenShift is still watching the container's security level. Let's inspect which **Security Context Constraint (SCC)** was assigned to our pod.

### 1. View the Cluster's default SCC list:
```bash
oc get scc
```
You will see a list of security profiles, ranging from `privileged` (dangerous, like root) to `restricted-v2` (super secure).

### 🖥️ Console Check #3: Find the Pod's Shield
1. In the Web Console **Topology** view, click on the `mapit` circle.
2. Under the **Resources** tab, click on the active running **Pod** name.
3. Switch to the **YAML** tab of the Pod.
4. Press `Ctrl + F` (or `Cmd + F`) and search for the word: **`scc`**
5. You will find an annotation called: `openshift.io/scc: restricted-v2`

> 🧠 **The Takeaway:** Because your cluster is an enterprise-grade platform, OpenShift automatically intercepted the pod creation and forced it into the `restricted-v2` sandbox. Even if a hacker breaches your container, they cannot access the underlying Linux host system because the SCC strips away their root privileges.

---

## 🏁 Summary Checklist
1. ✅ **Groups & RBAC:** Created a team and visually granted them rights in the console.
2. ✅ **Dynamic Storage:** Used a shortcut to create a PVC and watched the StorageClass provision a PV in real-time.
3. ✅ **SCC Awareness:** Inspected a running pod and discovered how OpenShift protects the infrastructure out-of-the-box using security constraints.

---

## 📌 Continuity Notice
Great job! Leave this environment exactly as it is. In the next module, we will explore how to monitor this application and look at cluster logs!
