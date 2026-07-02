# Lab 2: Governance, Storage & The Admin Challenge 🔐💾

Welcome back, Admin! Our `mapit` application from Lab 1 is running great inside the `app-management` project. However, right now, it’s wide open, and if the pod restarts, any data inside it is wiped out. 

Today, we will balance our skills between the **CLI tool** and the **OpenShift 4.20 Web Console** to secure the project, provision storage, inspect security restrictions, and face your first mini-challenge.

---

## 🧠 Concept Refreshers: Today's Building Blocks

> **1. Cluster Identity (OAuth & Users):** OpenShift uses an OAuth layer to authenticate users. Once a user logs in, OpenShift creates a native `User` and `Group` object.
>
> **2. RBAC (Roles & Bindings):** Defines who can do what inside our project. Instead of managing individual users, we always map permissions to a **Group**.
>
> **3. SCC (Security Context Constraints):** The ultimate Linux kernel guardrail. It controls what permissions the *container* has on the actual host node.
>
> **4. Storage (PVC & StorageClass):** Containers are ephemeral. A **PVC (Persistent Volume Claim)** is a request for a permanent hard drive, which a **StorageClass** automatically provisions for us.

---

## 👥 Part 1: Managing the Team (Users & RBAC)

Instead of writing massive YAML files for permissions, let's use the CLI to quickly create an organizational group, and then use the **Web Console** to grant them access.

### 1. Create a Group and a User via CLI:
```bash
# Create a group for our developers
oc adm groups new alpha-developers

# Add a simulated user named 'clara' into the group
oc adm groups add-users alpha-developers clara
```

### 🖥️ Console Check #1: Granting Access Visually
1. Open your OpenShift Web Console.
2. In the top project dropdown, make sure your project **`app-management`** is selected.
3. In the left navigation menu, go to **User Management** 👈 and click on **Groups**. Verify that `alpha-developers` exists and has `clara` inside it.
4. Now, stay in the same menu, but navigate to **Project** -> **Project Access** (or **Access**).
5. Click on **Add Role Binding** and set the following fields:
   * **Select Role:** `edit` (Allows them to deploy apps but not delete the project).
   * **Subject Kind:** `Group`
   * **Name:** `alpha-developers`
6. Click **Save**. 

### 2. Verify the Binding via CLI (The Balance)
Let's make sure the CLI sees exactly what you just configured in the browser:
```bash
# Check the role bindings inside your project
oc get rolebindings

# Zoom in to see who is attached to the "edit" role
oc describe rolebinding edit
```

### 3. Test Clara's Powers (Impersonation):
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

Right now, our `mapit` app saves everything in memory. Let's attach a permanent disk drive to it. We will use an amazing OpenShift admin shortcut command: `oc set volume`.

### 1. Attach a 1-Gigabyte Disk to the App via CLI:
```bash
oc set volume deployment/mapit --add --name=mapit-storage --type=pvc --claim-size=1Gi --mount-path=/app/data
```
> 💡 **What just happened?** This single command created a **PVC**, talked to the cluster's default **StorageClass**, automatically provisioned a real **Persistent Volume (PV)** from the cloud/infrastructure, and mounted it inside the container at `/app/data`!

### 2. Verify Storage via CLI
Before looking at the console, let's inspect the automated storage creation through the terminal:
```bash
# Check if the Persistent Volume Claim was created and bound
oc get pvc

# See the details of the requested disk
oc describe pvc mapit-storage
```

### 🖥️ Console Check #2: Watch the Storage Magic
1. In the left menu of the Web Console, click on **Topology**.
2. Click on the `mapit` circle. In the side-panel that opens on the right, click on the **Resources** tab.
3. Scroll down to the **Pods** section. Notice that a new pod is spinning up while the old one is terminating. OpenShift is doing a safe rolling update to apply the disk!
4. Go to **Storage** in the left menu -> **Persistent Volume Claims** to see your green, healthy **Bound** disk.

---

## 🛡️ Part 3: Inspecting the Shield (SCC via CLI)

Even though Clara can edit the project, OpenShift is still watching the container's security level. Let's use the CLI to find out which **Security Context Constraint (SCC)** was assigned to our pod.

### 1. View the Cluster's default SCC list:
```bash
oc get scc
```

### 2. Find the Pod's Shield using the Terminal
Let's extract the exact security constraint OpenShift forced upon our pod:
```bash
# Get your active pod name
oc get pods

# Fetch the SCC annotation from the pod's live YAML configuration
oc get pod <YOUR_POD_NAME_HERE> -o yaml | grep scc
```
👉 **Expected Output Line:** `openshift.io/scc: restricted-v2`

> 🧠 **The Takeaway:** Because your cluster is an enterprise-grade platform, OpenShift automatically intercepted the pod creation and forced it into the `restricted-v2` sandbox. Even if a hacker breaches your container, they cannot access the underlying Linux host system because the SCC strips away their root privileges.

---

## ⚔️ Part 4: The Admin Mini-Challenge!

**Scenario:** The security department is sending an auditor named **`frank`** to inspect the `app-management` project. 
* Frank needs to be able to *see* and *describe* everything inside the project (Pods, Services, Routes).
* Frank **must NOT** be able to create, delete, or change anything.

### Your Mission (CLI Only!):
1. Create a new group named `alpha-auditors`.
2. Add the user `frank` to the `alpha-auditors` group.
3. Use the CLI command `oc adm policy add-role-to-group` to bind the group to the built-in **`view`** cluster role inside the `app-management` project. *(Hint: Use your previous commands from Part 1 as a reference, but change the role from `edit` to `view`)*.

### Verify Your Success:
Run the following impersonation checks. If they match the expected outputs, you passed the challenge!

```bash
# Can Frank see the pods?
oc auth can-i get pods -n app-management --as=frank
# (Should say: yes)

# Can Frank delete or scale the mapit deployment?
oc auth can-i delete deployment mapit -n app-management --as=frank
# (Should say: no)
```

---

## 🏁 Summary Checklist
1. ✅ **CLI & Console Balance:** Created groups via CLI, bound roles via Console, and verified using `oc describe`.
2. ✅ **Dynamic Storage:** Used `oc set volume` and analyzed the PVC bindings via terminal and storage graphs.
3. ✅ **SCC Inspection:** Used `grep` on a live Pod manifest to see OpenShift's security guardrails.
4. ✅ **Challenge Complete:** Applied your knowledge to configure a read-only auditor role entirely from the CLI.

---

## 📌 Continuity Notice
Great job! Leave this environment exactly as it is. In the next module, we will explore how to monitor this application, check metrics, and look at cluster logs!
