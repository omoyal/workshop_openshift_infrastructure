# Lab 2: Secure & Store – Access, SCC, and Storage 🔐💾

Welcome back! Our `mapit` application from Lab 1 is still running inside the `app-management` project. 

In this lab, you will act as the **`kubeadmin`** to create a new developer, restrict their access to just one project, witness OpenShift's built-in security constraints (SCC) block a dangerous container, and add persistent storage to our app.

---

## 👥 Part 1: Create a Developer & Grant Permissions

OpenShift manages users natively. Let's create a developer user named `clara` and grant her access *only* to the `app-management` namespace.

### 1. Create the user and group via CLI:
```bash
# Create a group for developers
oc adm groups new alpha-developers

# Add user 'clara' to the group
oc adm groups add-users alpha-developers clara
```

### 🖥️ Console Check #1: Grant Access Visually
1. Open your OpenShift Web Console (Logged in as `kubeadmin`).
2. In the top project dropdown, ensure your project **`app-management`** is selected.
3. In the left navigation menu, scroll down to **User Management** 👈 and click on **RoleBindings**.
4. Click on the blue **Create Binding** button at the top right.
5. Set the following fields in the form:
   * **Binding Type:** Namespace role binding (RoleBinding)
   * **Name:** `clara-developer-binding`
   * **Namespace:** `app-management`
   * **Role Name:** `edit`
   * **Subject Kind:** `Group`
   * **Subject Name:** `alpha-developers`
6. Click **Create**.

### 2. Verify Clara's limited access via CLI:
```bash
# Can Clara manage pods inside app-management?
oc auth can-i create pods -n app-management --as=clara
# (Expected: yes)

# Can Clara see cluster nodes or other projects?
oc auth can-i get nodes --as=clara
# (Expected: no)
```

---

## 🛡️ Part 2: The Security Shield (SCC in Action)

OpenShift uses **Security Context Constraints (SCC)** to stop containers from running with dangerous privileges (like `root`). Let's try to deploy a standard `nginx` image, which by default demands root permissions, and see how OpenShift blocks it.

### 1. Attempt to deploy a root application via CLI:
```bash
oc create deployment unsafe-web --image=nginx -n app-management
```

### 🖥️ Console Check #2: Investigate the Refusal
1. In the Web Console left menu, click on **Topology**.
2. Look at the new `unsafe-web` component. It is **not** turning blue.
3. Click on the `unsafe-web` circle, and in the side-panel click on the **Resources** tab.
4. Click on the failing Pod/Deployment or look at the **Events** tab.
5. You will see a clear warning: OpenShift's **`restricted-v2`** policy refused to start the container because it requires root privileges! 

---

## 💾 Part 3: Adding Storage to our App

Right now, our `mapit` app from Lab 1 saves everything in temporary memory. Let's give it a permanent 1-Gigabyte disk using an awesome OpenShift admin shortcut command.

### 1. Attach a disk to the deployment via CLI:
```bash
oc set volume deployment/mapit --add --name=mapit-storage --type=pvc --claim-size=1Gi --mount-path=/app/data
```
> 💡 **What happened?** OpenShift instantly created a Persistent Volume Claim (PVC), communicated with the default cluster **StorageClass**, provisioned a real disk, and mounted it to `/app/data`.

### 2. Verify the storage via CLI:
```bash
oc get pvc -n app-management
```
*(You should see `mapit-storage` with a status of **Bound**)*

### 🖥️ Console Check #3: See the Live Rollout
1. Go back to the **Topology** view in the Web Console.
2. Click on the `mapit` circle.
3. Under the **Resources** tab, look at the **Pods** section. You will see OpenShift seamlessly terminating the old diskless pod and rolling out a brand new pod with the persistent disk attached!

---

## 🏁 Summary Checklist
1. ✅ **Targeted RBAC:** Created a user and limited their powers strictly to one project.
2. ✅ **SCC Security:** Saw firsthand how OpenShift protects host nodes from root containers.
3. ✅ **Dynamic Storage:** Provisioned and attached a persistent disk with a single command.

---

## 📌 Continuity Notice
Leave this project and the `mapit` application running. We will use them in the next lab to explore logging, monitoring, and cluster metrics!
