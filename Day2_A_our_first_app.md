# Module: Application Management Basics on OpenShift

In this module, you will deploy a sample application using the `oc` CLI tool. You will also learn about the core concepts, fundamental objects, and basics of application management on the **OpenShift Container Platform**.

---

## 📑 Core OpenShift Concepts

As a future administrator of OpenShift, it is important to understand several core building blocks as they relate to applications. Understanding these building blocks will help you better see the "big picture" of application management on the platform.

### 1. Projects
A **Project** is a "bucket" of sorts. It’s a meta-construct where all of a user’s resources live. 

> 💡 **Administrative Perspective:** Each Project can be thought of like a tenant. Projects may have multiple users who can access them, and users may be able to access multiple Projects. 
> 
> Technically speaking, a user doesn’t own the resources—the Project does. Deleting a user does not affect any of the created resources.

#### 🛠️ Exercise: Create a Project
For this exercise, first create a Project to hold your resources:

```bash
oc new-project app-management
```

---

## 🚀 Deploy a Sample Application

The `oc new-app` command provides a very simple way to tell OpenShift to run things. You simply provide it with one of a wide array of inputs, and it figures out what to do. 

Users commonly use this command to:
* Launch existing container images.
* Create builds of source code and deploy them.
* Instantiate templates.

#### 🛠️ Exercise: Launch a Sample Image
You will now launch a specific image that exists on Quay:

```bash
oc new-app quay.io/openshiftroadshow/mapit
```

![Application Architecture](https://github.com/user-attachments/assets/64b48045-9f23-4366-b366-7eb3f1997d24)

---

## 📦 Pods

**Pods** are one or more containers deployed together on a host. A Pod is the smallest compute unit you can define, deploy, and manage in OpenShift. 

* **Networking:** Each Pod is allocated its own internal IP address on the SDN (Software Defined Network) and owns the entire port range.
* **Resource Sharing:** The containers within a Pod can share local storage space and networking resources.
* **Immutability:** Pods are treated as **static objects** by OpenShift. You cannot change a Pod definition while it is running.

#### 🛠️ Exercise: List Current Pods
You can get a list of running Pods using the following command:

```bash
oc get pods
```

---

## 🔌 Services

![Services Architecture](https://github.com/user-attachments/assets/397028b9-06a7-4c63-b635-b3e5d84a7bf0)

**Services** provide a convenient abstraction layer inside OpenShift to find a group of identical Pods. They act as an **internal proxy/load balancer** between those Pods and anything else that needs to access them from inside the OpenShift environment. 

> 🌍 **Internal Construct:** Remember that Services are strictly internal. They are **not** available to the "outside world" or anything outside the OpenShift cluster environment.

* **Scaling:** If you need more `mapit` instances to handle the load, you can spin up more Pods. OpenShift automatically maps them as endpoints to the Service, and incoming requests won't notice any change (except better performance!).
* **Labels & Selectors:** A Service maps to a set of Pods via a system of **Labels** and **Selectors**. 
* **IP Allocation:** Services are assigned a fixed IP address, allowing many ports and protocols to be mapped.

#### 🛠️ Exercise: List Current Services
When you ran the `new-app` command, OpenShift automatically created a Service for you. Verify it by running:

```bash
oc get services
```

---

## 🔄 Deployments and ReplicaSets

While Services provide routing and load balancing for Pods (which may come and go), **ReplicaSets (RS)** are used to specify and ensure that the desired number of Pod replicas are running at any given time.

* **Self-Healing:** If you want an application to always scale to 3 Pods, a ReplicaSet is required. Without an RS, any Pods that crash or die are **not** automatically restarted. This is how OpenShift "self-heals".
* **Deployments:** Manage the rollout of ReplicaSets and provide declarative updates to Pods.

#### 🛠️ Exercise: Explore Deployments & ReplicaSets
Take a look at the Deployment (`deploy`) that was created for you when you stood up the `mapit` image:

```bash
oc get deployment
```

To get more details, examine the underlying ReplicaSet (RS):

```bash
oc get replicaset
```

---

## 🌐 Routes

![Routes Architecture](https://github.com/user-attachments/assets/cb2d1276-c2da-4f2a-a750-1b468f61d817)

While Services handle internal traffic, external clients (users, external systems, devices) often need access to the application. The **OpenShift Routing Layer** is what enables external access, and the data object behind it is a **Route**.

* **How it works:** The default OpenShift router (**HAProxy**) uses the HTTP header of the incoming request to determine where to proxy the connection.
* **Security:** You can optionally define security configurations, such as **TLS**, directly on the Route.
* **The Router Operator:** The installation automatically deployed an Operator for the router inside the `openshift-ingress` Project.

#### 🛠️ Exercise: Inspect the Default Router
You can view information about the default cluster router using:

```bash
oc describe deployment router-default -n openshift-ingress
```

---

## 🏗️ Creating a Route

Creating a Route is a straightforward process. You simply "expose" the internal Service via the command line.

#### 🛠️ Exercise: Expose the Service
Since your Service name is `mapit`, you can expose it to the outside world with a single command:

```bash
oc expose service mapit
```

**Expected Output:**
> `route/mapit exposed`

#### 🛠️ Exercise: Verify the Route
Verify that the Route was successfully created and retrieve its external URL:

```bash
oc get route
```

```bash
oc get route mapit -o jsonpath='{.spec.host}' | sed 's/^/http:\/\//'
```

click on the Url Link


