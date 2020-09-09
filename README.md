# k8s-troubleshooting
## A guide for, you guessed it, troubleshooting K8s!
References: Kubernetes documentation as a whole, but also:
* https://kubernetes.io/docs/tasks/debug-application-cluster/debug-application/
* https://kubernetes.io/docs/tasks/debug-application-cluster/debug-cluster/
* https://kubernetes.io/docs/tasks/debug-application-cluster/debug-running-pod/
* https://kubernetes.io/docs/tasks/debug-application-cluster/get-shell-running-container/

### Step One: Is it the Application?
*"What is the problem? Is it your Pods, your Replication Controller or your Service?"*
* **Pods**
  * `kubectl describe pods ${POD_NAME}` to get a better idea of whats happening inside your pod.
    * Are the **Containers** inside the **Pod** running? Have they been restarted recently?
  * **Your Pod is Pending** - It cannot be scheduled onto the a Node!
    * Generally this means you don't have enough available resources on the Node. There are a few things you should validate if this is the case:
      * 1) **Check CPU and Memory Usage** in your Kubernetes dashboard. 
      * 2) Cross-reference 1 with your **Deployment manifest's Resource Requests**. There is probably a discrepancy between 1 and 2. If your deployment doesn't actually _need_ that much resources in practice, you could consider dropping the resource request to something that your current node pool can handle.
      * 3) You may need to **Configure Autoscaling** in AWS, or increase the the upper limit of instances allowed during scale-up events.
      * 4) You may also want to consider migrating to larger instances if the Pods you run on each node are beginning to outgrow the EC2's you are currently running (or if throughput has become more of a priority).
  * **Your Pod is Crash Looping or Running** but otherwise not functioning as expected. In this case, your Pod is already scheduled onto a Node.
    * **NOTE: You may need to ssh onto your Node if troubleshooting becomes more dire.** Make sure you have this setup already OR have someone you can get the required files from.
    * `kubectl logs ${POD_NAME} ${CONTAINER_NAME}` to look at the logs of the affected container. OR...
    * `kubectl logs --previous ${POD_NAME} ${CONTAINER_NAME}` to look at the logs of the **previously crashed** container. 
    * If the container image includes some debugging tools, you can exec into the container and dig into the issues that way: `kubectl exec ${POD_NAME} -c ${CONTAINER_NAME} -- ${CMD} ${ARG1} ${ARG2} ... ${ARGN}`.
      * Ex: If running Cassandra, you can look at the logs with `kubectl exec cassandra -- cat /var/log/cassandra/system.log`
      * Alternatively, you can open a shell into your Cassandra container with `kubectl exec -it cassandra -- sh`.
    * If you don't have any debugging tools in the container, you can try debugging with an ephemeral container that does have the tools you need.
      * `kubectl alpha debug` adds an ephemeral container to a running Pod.
      * More specifically...
        * `kubectl alpha debug -it ${POD_NAME} --image=busybox --target=${POD_NAME}`
  * **Your Pod is Running but not doing what it is supposed to.** This one is generally an issue with your Pod's description/manifest file.
    * Check your YAML indentation and make sure there are no typos throughout your Pod description.
    * First, delete your Pod. `kubectl delete pods ${POD_NAME}`
    * Then, apply your manifest again, but using the validate flag: `kubectl apply --validate -f ${POD}.yaml`
      * This should reveal some additional error logs that may help shed some light on the issue.
    * **Alternatively output the manifest on the K8s API Server and compare to what you wanted to create.**
      * `kubectl get pods/mypod -o yaml > mypod-on-apiserver.yaml`. Generally you should see a few extra lines of yaml in the manifest running on the API server. **HOWEVER** if lines are missing on the API server manifest vs your file, there's probably an issue with the Pod spec.
* **Services**
  * Services provide load balancing across a set of Pods. There are several common problems that can make Services not work properly.
  * **Verify there are endpoints for the Service** (for every Service object the api server makes an endpoints resource available).
    * Check this with `kubectl get endpoints ${SERVICE_NAME}`. If the endpoints match up with the number of containers that you expect to be a member of the Service. Ex: if you have 3 replicas, you should have 3 IP's output from that last command.
  * **Your Service is missing endpoints.** Try listing the pods using the labels that Service uses.
    * If you have a Service where the labels are: `- selector: name: nginx, type: frontend` for example, use `kubectl get pods --selector=name=nginx,type=frontend`
    * Depending on the output from above, there are a couple possibilities:
      * 1) **If the list of Pods matches expectations, but your endpoints are still empty**: You likely don't have the right ports exposed.
        * **If your service has a ContainerPort specified, but the Pods that are selected don't have that port listed, then they won't be added to the endpoints list.** Verify the Pod's containerPort matches the Service's targetPort.
      * 2) **Network traffic is not forwarded**. So there are endpoints in the list, you can connect, but the connection is immediately dropped. This likely means your proxy can't contact your pods. Check the following:
        * Are your pods working correctly? (See above).
        * Can you connect to the pods directly?
        * Is your app serving the port that you configured? I.E., if your app serves 8080, `containerPort` needs to be 8080.
  
### Step Two: Is it the Cluster?
*So, you've already ruled out the problem is not with your application, let's troubleshoot the cluster.*  
* `kubectl get nodes` to verify your cluster actually knows it has nodes!
  * One thing I've personally run into on this is when running the AWS provider in Terraform via Windows things don't play well, and you end up with everything looking pretty except that EKS doesn't know about the nodes you created for it to use.
  * Verify they are all ready too!
  * Similarly, run `kubectl cluster-info dump` to get more detailed info on the overall health of your cluster.
* **Check the logs**
  * Master Nodes:
    * `/var/log/kube-apiserver.log` - API Server.
    * `/var/log/kube-scheduler.log` - Scheduler, makes scheduling decisions.
    * `/var/log/kube-controller-manager.log` - Controller Manager runs controllers to regulate behavior in the cluster (replication).
  * Worker Nodes:
    * `/var/log/kubelet.log` - kubelet is responsible for running containers on the node. 
    * `/var/log/kube-proxy.log` - Responsible for service load balancing.
* **Stuff Goes Wrong, Here are some Root Causes and how to mitigate them:**
Directly from above links (remember this doc is for my own studying and local use, obviously the above live Kubernetes documents are more living, maintained, and detailed.)  
  * General causes to watch for:
    * VM(s) shutdown
    * Network partition within cluster, or between cluster and users
    * Crashes in Kubernetes software
    * Data loss or unavailability of persistent storage (e.g. GCE PD or AWS EBS volume)
    * Operator error, for example misconfigured Kubernetes software or application software.
  * Mitigations:
    * Action: Use IaaS provider's automatic VM restarting feature for IaaS VMs
      * Mitigates: Apiserver VM shutdown or apiserver crashing
      * Mitigates: Supporting services VM shutdown or crashes
    * Action: Use IaaS providers reliable storage (e.g. GCE PD or AWS EBS volume) for VMs with apiserver+etcd
      * Mitigates: Apiserver backing storage lost
    * Action: Use high-availability configuration
      * Mitigates: Control plane node shutdown or control plane components (scheduler, API server, controller-manager) crashing
        * Will tolerate one or more simultaneous node or component failures
      * Mitigates: API server backing storage (i.e., etcd's data directory) lost
        * Assumes HA (highly-available) etcd configuration
    * Action: Snapshot apiserver PDs/EBS-volumes periodically
      * Mitigates: Apiserver backing storage lost
      * Mitigates: Some cases of operator error
      * Mitigates: Some cases of Kubernetes software fault
    * Action: use replication controller and services in front of pods
      * Mitigates: Node shutdown
      * Mitigates: Kubelet software fault
    * Action: applications (containers) designed to tolerate unexpected restarts
      * Mitigates: Node shutdown
      * Mitigates: Kubelet software fault


