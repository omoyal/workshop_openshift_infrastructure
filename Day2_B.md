# Lab: Securing the Realm – Authentication, RBAC, and SCC

Welcome to your first day as the Lead Cluster Administrator! Your organization just handed you a brand-new **OpenShift 4.20** cluster. 

A new development team, **Team Alpha**, needs access to the cluster to deploy their apps. As an admin, your job is to make sure they have exactly what they need to work—nothing more, nothing less. 

---

## 🎯 Learning Objectives
By the end of this lab, you will know how to:
1. Create local users in OpenShift using an Identity Provider (OAuth).
2. Group users together and manage permissions using **RBAC** (Roles and Bindings).
3. Experience the power of OpenShift's **SCC (Security Context Constraints)**—the ultimate shield that Vanilla Kubernetes doesn't give you out-of-the-box.

---

## 🧭 Scenario Roadmap
* **Part 1:** Open the Gates (Configure Authentication)
* **Part 2:** Define the Team (Groups & RBAC)
* **Part 3:** Test Your Powers (`oc auth can-i`)
* **Part 4:** The Security Shield (Encountering SCC)

---

## 🚪 Part 1: Open the Gates (Authentication)

In OpenShift, the cluster doesn't manage users directly inside its core database; it relies on an **Identity Provider (IdP)**. For this lab, we will use a simple local file-based provider called **HTPasswd**.

### Step 1: Create a local password file
Let's create two users: `alice` (our developer) and `bob` (our security auditor). Run the following commands on your terminal:

```bash
# Install htpasswd utilities if you don't have it (optional)
# Create the file and add Alice (password: alice123)
htpasswd -c -B -b ./htpasswd alice alice123

# Add Bob to the same file (password: bob123)
htpasswd -B -b ./htpasswd bob bob123
```

### Step 2: Upload the passwords to OpenShift
OpenShift needs this file inside a Secret in the `openshift-config` namespace so the OAuth server can read it:

```bash
oc create secret generic alpha-htpasswd-secret --from-file=htpasswd=./htpasswd -n openshift-config
```

### Step 3: Tell OpenShift to use this Identity Provider
Now, let's update the cluster's OAuth settings. Don't worry, we will use a safe `oc patch` command instead of manually editing giant YAMLs:

```bash
oc patch oauth cluster --type='json' -p='[{"op": "add", "path": "/spec/identityProviders", "value": [{"name": "alpha-users", "challenge": true, "login": true, "mappingMethod": "claim", "type": "HTPasswd", "htpasswd": {"fileData": {"name": "alpha-htpasswd-secret"}}}]}]'
```

> ⏳ **Patience is an Admin virtue:** OpenShift is now rolling out a new OAuth pod in the background. Give it about 1–2 minutes to apply.

---

## 👥 Part 2: Define the Team (Groups & RBAC)

Kubernetes doesn't have a native `Group` or `User` object that you can view with `kubectl get users`. But **OpenShift does!** This makes managing teams much cleaner.

### Step 1: Create a Project for Team Alpha
```bash
oc new-project team-alpha-prod
```

### Step 2: Create a Group and add Alice
Instead of assigning permissions to Alice directly, let's create a group called `developer-group` and add her to it. This way, when 10 more developers join tomorrow, you just add them to the group!

```bash
# Create the group
oc adm groups new developer-group

# Add Alice to the group
oc adm groups add-users developer-group alice
```

### Step 3: Grant Permissions using RBAC
We want everyone in `developer-group` to be able to deploy applications inside the `team-alpha-prod` project, but they shouldn't be able to touch cluster-wide settings. We will bind them to the built-in `edit` role.

```bash
oc adm policy add-role-to-group edit developer-group -n team-alpha-prod
```

---

## 🧪 Part 3: Test Your Powers (`oc auth can-i`)

As an administrator, you don't always want to log out and log back in just to test if your permissions worked. OpenShift allows you to "impersonate" users to test your RBAC configuration.

### Step 1: Test Alice (The Developer)
Let's see if Alice can create a deployment inside her project:
```bash
oc auth can-i create deployments -n team-alpha-prod --as=alice
```
👉 **Expected Output:** `yes`

Can Alice delete the entire project or look at cluster nodes?
```bash
oc auth can-i delete project team-alpha-prod --as=alice
oc auth can-i get nodes --as=alice
```
👉 **Expected Output:** `no`

### Step 2: Test Bob (The Auditor)
Bob hasn't been granted any roles yet. Let's check if he can see pods in `team-alpha-prod`:
```bash
oc auth can-i get pods -n team-alpha-prod --as=bob
```
👉 **Expected Output:** `no`

---

## 🛡️ Part 4: The Security Shield (Encountering SCC)

> 💡 **Kubernetes Veterans Note:** In Vanilla Kubernetes, any user with `edit` rights can run almost any container, even dangerous ones that run as `root` and can harm the host node. 
> 
> **OpenShift says: No way.** OpenShift uses **Security Context Constraints (SCC)** to intercept Pod creations and ensure they behave safely, regardless of what the container image wants.

Let's see what happens when Alice tries to deploy a standard web server like `nginx`, which by default wants to run as the `root` user (privileged).

### Step 1: Alice tries to deploy Nginx
Run this command to create a deployment as Alice:

```bash
oc create deployment secure-web --image=nginx -n team-alpha-prod
```

### Step 2: Investigate the crime scene
Let's see if the pod actually starts up:
```bash
oc get pods -n team-alpha-prod
```

Wait... Where is the pod? It's not there! Let's look at the deployment's status:
```bash
oc describe deployment secure-web -n team-alpha-prod
```
Look at the conditions or the ReplicaSet events (`oc get rs` then `oc describe rs <rs-name>`). You will see that OpenShift blocked the creation or modified the container because of the **`restricted-v2`** SCC policy!

OpenShift automatically forces containers to run with an unpredictable, random high-number User ID (UID) instead of `root` to protect the host. Since `nginx` absolutely requires root to open port 80, it crashes or fails to scale.

### Step 3: The Admin Fix (Granting exceptions)
As an administrator, if you **trust** Team Alpha and they absolutely must run this specific container, you can grant their service account access to a relaxed security constraint called `anyuid`.

```bash
# In OpenShift, deployments use a 'default' ServiceAccount inside the namespace
oc adm policy add-scc-to-user anyuid -z default -n team-alpha-prod
```

### Step 4: Verify the fix
Now, check your pods again. The ReplicaSet will successfully roll out the pod because OpenShift now allows it to run with its requested UID:

```bash
oc get pods -n team-alpha-prod
```
👉 **Expected Output:** Your `secure-web-...` pod should now be `Running`!

---

## 🧹 Clean Up
When you are done, clean up the resources to keep the cluster tidy:
```bash
oc delete project team-alpha-prod
oc adm groups delete developer-group
oc delete oauth cluster # (Note: In production, you would use patch to remove the IdP instead of deleting the object)
```

### 🏁 Summary Checklist for your Organization:
1. **Users/Groups** are native OpenShift objects, making access management easy.
2. **RBAC** controls *what actions* a user can do to an API object (Get, Create, Delete).
3. **SCC** controls *what the actual container* is allowed to do inside the Linux kernel on the host node (Run as root, access host network, etc.).
