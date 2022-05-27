Koo bear net ease!

What is it
Why I like it
Why this is the future

---

Let's talk about managing app servers - specifically web app servers, because nobody runs Visual Basic mid-tier servers anymore.

---

Most web apps tend to look a lot like this:

Load balancer -> Application server -> Database
  app --> Redis (maybe)
  app --> Memcache (maybe)
  app --> MongoDB (maybe)
  app --> etc

How we'd do this in the past. You'd set up a VM (or maybe even a real server), install redis, memcache, your app server's long list of required libraries, your _app's_ additional libraries...

Each one of these components needs to be set up and managed. Or at least not forgotten about.

---

What if you need to upgrade the app? Or need to set up a second app server?

---

A bit ago there was a movement around "let's automate (Linux) server configuration" and a few products that, if you squint, all look the same, came out: chef, puppet, ansible, salt. Their key idea is to have some kind of neutral recipe language that describes the _desired state_ of the server - which libraries should be installed, how ssh should be configured, and so on - and then apply the recipe to a server or a whole roomful of servers.

You can upgrade your app by changing the recipe and then re-applying it, that usually works, although it often works better to apply it to a new server entirely.

You might have decided to run a VM for each of those components - you'd run a VM for redis, one for memcache, etc. You'll have to set up DNS or something for each part to find the other.

---

This is great and all, but now you're managing recipes _and_ VMs _and_ networks. At least the recipe is a code-as-documentation "run book" for setting up a server. Go ahead and cross the street without being scared of the busses.

These recipes tend to turn into exquisite yak shaving projects, trying to get the most idealized, spherical yak possible.

---

Is this actually better?

---

Let's talk about Docker.

* It's _not_ virtualization - all programs run through the same kernel
  * Docker on Windows or Mac _does_ run a small Linux VM to actually run the containers, but on a Linux system, that VM does not exist.
* Containers must "pack in" everything they need - config files, libraries, etc
* Each container is given an IP address
* This isn't new or Linux exclusive; Docker is broadly similar to AppV, BSD jails, Solaris zones, and AIX LPARs.

Docker wraps some underling Linux technologies with a nice UI. Because Linux-land loves to reimplement identical technologies, RedHat ships an identical system called podman. We don't need to talk about it.

---

Docker provides a mini-network and naming system, so you could have your app's docker container and (say) redis and memcache in their own containers, and they would all be on a little private network. Docker hooks up host-names between them, so the app server just needs to connect to redis on host `redis` and it works!

---

One key feature of Docker is Dockerfile, which is a clear, repeatable set of steps to build these containers. It's a recipe. Even better, it's often included alongside your app's source code, which is better than keeping it on some Ansible playbook.

But once the container is built, Docker lets you ship the whole thing and store it in a Docker registry. There's no reason to have to rebuild them, unless you need to make a new container with an updated app.

Docker, Inc, runs a public registry that includes a lot of containers for core components, including redis and memcache, and a lot of third parties publish docker containers for their tools - like mongodb.

One additional key feature: containers don't need to start from empty, they can start with some _other_ container's contents, so you don't need to re-create Linux From Scratch on every project. You can just start with Debian and then add your special customizations.

There are also pre-made containers for various versions of common languages. We - along with thousands of others - start with the community-maintained `ruby` container - and that's based on the `debian` container under it.

---

Some tall Dutch guy tells you about this Docker thing and so you set up your app using a bunch of Docker containers. On the plus side, your container is now immune to ancient libraries in the underlying OS, and you can update by just building your container somewhere, then pulling it onto the VM.

On the minus side, you're still pulling and setting up these containers.

---

Suddenly, you find you no longer care about virtual machines at all.

---

Let's talk about Kubernetes.

---

At it's core, Kubernetes is a container deployment manager. You, super sysop person, give k8s a set of goals to achieve, and a fleet of VMs to use, and then leave it alone.

---

Pods are the k8s name for a docker container. Pods technically can have more than one container, but it's complicated and not only useful for some very specific reasons. So just substitute "pod" for "container" and you're set.

---

k8s includes meta-doodads called "controllers." One such doodad is the `deployment`, a controller that manages pods. You tell the deployment:

* I want three pods replica pods running a specific docker container.
* The pods need yea memory and yan CPU.
* Please mount this NFS server in each pod.
* If you can, try to put a pod on the same machine as the redis pod.
* It'd also be nice if the three pods weren't, like, on the same server.
* It can take the pod a minute to get ready, so give it a moment before you open the gates.
* You can ask the pod for an HTTP request like /_health to see if it's awake or not.
* If it seems like it's not, kindly restart it.
* If I ask you to change the container to a different version, would you do only 1 pod replacement at a time?

---

The two most basic fundamental components in k8s are "pods" - which is kube-speak for containers - and "services."

Pods are tagged with labels, like "app=redis".

Services publish DNS names on stable, virtual IPs - eg, there's a "service" called "redis" at IP 10.100.34.38. But the service doesn't actually do anything itself, they just route traffic by looking up matching labels.

That is, when a pod sends traffic to 10.100.34.38, k8s says "well, do I have a pod that is tagged app=redis?" It does - so it routes the traffic to the current pod at 172.27.21.67.

---

Look at all the work you don't need to do!

---

Any other tricks? Why yes: k8s can run load balancers for you, and manage incoming web traffic, directing clients to specific services (which, in turn, route to pods). Sometimes these are implemented with managed haproxy or nginx servers in pods. Or it can manage AWS or GCE load balancers directly, pluming them directly to a service.

Another thing you might need is block storage. k8s has a sibling to the deployment, the statefulset, which manages pods that need block storage. Statefulsets also have a kabuki-like upgrade process.

If you need to run a pod on _every_ node in your cluster, the daemonset is what you want. Daemonsets are often used for low-level management of nodes, such as gathering node health.

---

I haven't really talked about nodes so much, because there's nothing _to_ talk about. K8s nodes are usually "managed" with some kind of automation tool in your VM or cloud platform. We rely on AWS autoscalers to keep the nodes alive. If a node fails because the underlying host fails, we don't need to recover the VM, we just create a new one and join it to the cluster.

Any work that was on that node when it went offline would already have been relaunched on existing nodes, as long as there was capacity. If there wasn't, when the new node was available, pending pods would be deployed to it.

---

All of these k8s components are configured via a simple text file. You can even have one file with multiple components in it, so you can describe your app's entire environment - every deployment, statefulset, physical volume, service, and any incoming web routes - all in a single file, easy to `apply` to k8s.

---

The reason why I love this, and why I think it's the future, is because it frees me from doing any of this tedious work.

I can deploy a new application by just creating a k8s config file and submit it to the cluster, and hey presto everything is running.

I don't even need to care much about the cluster: Amazon manages that entirely. It's even possible to run pods on "serverless" VMs - VMs that Amazon completely manages.

In the future we wouldn't even need to think about the cluster, or nodes, or anything - just "here's some stuff to run, please run it, thank you."

---


