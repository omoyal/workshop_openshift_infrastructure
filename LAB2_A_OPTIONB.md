# Lab 1: Application Management Basics on OpenShift 🚀

Welcome to your first hands-on experience with **OpenShift 4.20**! 

In this module, you will deploy your very first application using the `oc` CLI tool, explore its components, and see how OpenShift's Web Console makes managing applications incredibly visual and intuitive.

---

## 📑 Part 1: The "Bucket" (Creating a Project)

In OpenShift, everything lives inside a **Project**. Think of a Project as a secure Kubernetes namespace with extra corporate superpowers (like built-in user access and quotas).

### 1. Create your project via CLI:
```bash
oc new-project app-management
```

### 🖥️ Console Check #1: Verify in the Browser
1. Open your OpenShift Web Console.
2. In the top-left corner, switch from the **Administrator** perspective to the **Developer** perspective.
3. Look at the **Project** dropdown at the top. Ensure `app-management` is selected.
4. Click on **Topology** in the left menu. Right now, it's completely empty. Let's fix that!

---

## 🚀 Part 2: Deploying Your First App

We will use the smart `oc new-app` command. You point it at a container image, and OpenShift automatically figures out how to run it, build a deployment, and set up networking.

### 1. Launch a sample map application from Quay:
```bash
oc new-app quay.io/openshiftroadshow/mapit
```

### 🖥️ Console Check #2: Watch it Come to Life!
1. Look at your **Topology** screen in the browser right now.
2. You will see a visual circular icon representing your application spinning up! 
3. Wait until the ring around the `mapit` icon turns **solid blue**. This means your application is fully ready and running.

<img width="495" height="646" alt="image" src="https://github.com/user-attachments/assets/8ee5d045-0bc9-44fd-aaca-ade24e2dd060" />

---

## 📦 Part 3: Pods & The Self-Healing Magic

A **Pod** is the smallest compute unit in OpenShift. It hosts your running container.

### 1. List your running pods via CLI:
```bash
oc get pods
```

### 🧪 The Self-Healing Experiment
Let's see why OpenShift is a managed platform. What happens if a pod crashes or gets deleted?

1. In the Web Console **Topology** view, click on the center of the `mapit` circle.
2. A side-panel will open on the right. Click on the **Resources** tab.
3. Look under the **Pods** section, find your running pod, and click on it.
4. On the top right, click **Actions** 👈 and choose **Delete Pod**.
5. **Quickly look back at the Topology screen!** > 💡 **What just happened?** OpenShift's underlying **ReplicaSet** noticed a pod died and instantly fired up a new one to keep your desired state active. That is automated self-healing!

---

## 🔌 Part 4: Internal Networking (Services)

![Services Architecture](https://github.com/user-attachments/assets/397028b9-06a7-4c63-b635-b3e5d84a7bf0)

When OpenShift deployed your app, it automatically created a **Service**. A Service acts as an internal load balancer/proxy. If you scale your app to 5 pods, the Service knows how to distribute traffic between them internally.

### 1. See your service via CLI:
```bash
oc get services
```
> ⚠️ **Note:** Notice the IP address. This is a cluster-internal IP. The outside world (your laptop's browser) cannot reach this IP yet.

---

## 🌐 Part 5: Opening the Gates to the World (Routes)

![Routes Architecture](https://github.com/user-attachments/assets/cb2d1276-c2da-4f2a-a750-1b468f61d817)

To make an application accessible from outside the cluster, OpenShift uses an object called a **Route**. A Route configures the cluster's built-in HAProxy router to send external web traffic directly to your internal Service.

### 1. Expose the service via CLI to create a Route:
```bash
oc expose service mapit
```

### 2. Verify the Route:
```bash
oc get route
```

### 🖥️ Console Check #3: Click and Launch!
1. Go back to your **Topology** view in the Web Console.
2. Look at the `mapit` visual circle. You will notice a small **external link icon** (a little arrow pointing top-right) has appeared on the edge of the circle.
3. **Click that icon!** It will automatically open a new browser tab running your live application! It should look like this:

<img width="691" height="408" alt="image" src="https://github.com/user-attachments/assets/5700d3f0-a4d3-4903-a406-0636d2ec81bc" />

---

## 📌 Important Notice: No Cleanup!
**Do NOT delete this project or application.** We will be using this exact deployment and project as our foundation for the upcoming advanced configuration labs!
