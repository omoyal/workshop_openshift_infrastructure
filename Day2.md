In this module, you will deploy a sample application using the oc tool and learn about some of the core concepts, fundamental objects, and basics of application management on OpenShift Container Platform.

Core OpenShift Concepts
As a future administrator of OpenShift, it is important to understand several core building blocks as it relates to applications. Understanding these building blocks will help you better see the big picture of application management on the platform.

Projects
A Project is a "bucket" of sorts. It’s a meta construct where all of a user’s resources live. From an administrative perspective, each Project can be thought of like a tenant. Projects may have multiple users who can access them, and users may be able to access multiple Projects. Technically speaking, a user doesn’t own the resources, the Project does. Deleting the user doesn’t affect any of the created resources.

For this exercise, first create a Project to hold some resources:


oc new-project app-management










Deploy a Sample Application
The new-app command provides a very simple way to tell OpenShift to run things. You simply provide it with one of a wide array of inputs, and it figures out what to do. Users will commonly use this command to get OpenShift to launch existing images, to create builds of source code and ultimately deploy them, to instantiate templates, and so on.

You will now launch a specific image that exists on Quay


oc new-app quay.io/openshiftroadshow/mapit

<img width="910" height="1180" alt="image" src="https://github.com/user-attachments/assets/64b48045-9f23-4366-b366-7eb3f1997d24" />



Pods are one or more containers deployed together on a host. A pod is the smallest compute unit you can define, deploy and manage in OpenShift. Each pod is allocated its own internal IP address on the SDN and owns the entire port range. The containers within pods can share local storage space and networking resources.

Pods are treated as static objects by OpenShift, i.e., one cannot change the pod definition while running.

You can get a list of pods:

oc get pods







SERVICE
<img width="1423" height="975" alt="image" src="https://github.com/user-attachments/assets/397028b9-06a7-4c63-b635-b3e5d84a7bf0" />


Services provide a convenient abstraction layer inside OpenShift to find a group of like Pods. They also act as an internal proxy/load balancer between those Pods and anything else that needs to access them from inside the OpenShift environment. For example, if you needed more mapit instances to handle the load, you could spin up more Pods. OpenShift automatically maps them as endpoints to the Service, and the incoming requests would not notice anything different except that the Service was now doing a better job handling the requests.

When you asked OpenShift to run the image, the new-app command automatically created a Service for you. Remember that services are an internal construct. They are not available to the "outside world", or anything that is outside the OpenShift environment. That’s OK, as you will learn later.

The way that a Service maps to a set of Pods is via a system of Labels and Selectors. Services are assigned a fixed IP address and many ports and protocols can be mapped.

There is a lot more information about Services, including the YAML format to make one by hand, in the official documentation.

You can see the current list of services in a project with:


oc get services






Deployments and ReplicaSets in OpenShift
While Services provide routing and load balancing for Pods, which may go in and out of existence, ReplicaSets (RS) are used to specify and then ensure the desired number of Pods (replicas) are in existence. For example, if you always want an application to be scaled to 3 Pods (instances), a ReplicaSet is needed. Without an RS, any Pods that are killed or somehow die/exit are not automatically restarted. ReplicaSets are how OpenShift "self heals".

Now that we know the background of what a ReplicaSet and Deployment are, we can explore how they work and are related. Take a look at the Deployment (deploy) that was created for you when you told OpenShift to stand up the mapit image:
oc get deployment


To get more details, we can look into the ReplicaSet (RS).
Take a look at the ReplicaSet (RS) that was created for you when you told OpenShift to stand up the mapit image:

oc get replicaset





Routes
<img width="1483" height="1062" alt="image" src="https://github.com/user-attachments/assets/cb2d1276-c2da-4f2a-a750-1b468f61d817" />

While Services provide internal abstraction and load balancing within an OpenShift environment, sometimes clients (users, systems, devices, etc.) outside of OpenShift need to access an application. The way that external clients are able to access applications running in OpenShift is through the OpenShift routing layer. And the data object behind that is a Route.

The default OpenShift router (HAProxy) uses the HTTP header of the incoming request to determine where to proxy the connection. You can optionally define security, such as TLS, for the Route. If you want your Services (and by extension, your Pods) to be accessible to the outside world, then you need to create a Route.

Do you remember setting up the router? You probably don’t. That’s because the installation deployed an Operator for the router, and the operator created a router for you! The router lives in the openshift-ingress Project, and you can see information about it with the following command:

oc describe deployment router-default -n openshift-ingress




Creating a Route
Creating a Route is a pretty straight-forward process. You simply expose the Service via the command line. If you remember from earlier, your Service name is mapit. With the Service name, creating a Route is a simple one-command task:

oc expose service mapit
You will see:

route/mapit exposed

Verify the Route was created with the following command:

oc get route




