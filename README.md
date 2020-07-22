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
  * Services provide load balancing across a set of Pods. There are severak common problems that can make Services not work properly.
  * **Verify there are endpoints for the Service** (for every Service object the api server makes an endpoints resource available).
    * Check this with `kubectl get endpoints ${SERVICE_NAME}`. If the endpoints match up with the number of containers that you expect to be a member of the Service. Ex: if you have 3 replicas, you should have 3 IP's output from that last command.
  * **Your Service is missing endpoints.** Try listing the pods using the labels that Service uses.
    * If you have a Service where the labels are: `- selector: name: nginx, type: frontend` for example, use `kubectl get pods --selector=name=nginx,type=frontend`
    * Depending on the output from above, there are a couple possibilities:
      * 1) **If the list of Pods matches expectations, but your endpoints are still empty**: You likely don't have the right ports exposed.
        * **If your service has a ContainerPort specified, but the Pods that are selected dob't have that port listed, then they won't be added to the endpoints list.** Verify the Pod's containerPort matches the Service's targetPort.
      * 2) **Network traffic is not forwarded**. So there are endpoints in the list, you can connect, but the connection is immediately dropped. This likely means your proxy can't contact your pods. Check the following:
        * Are your pods working correctly? (See above).
        * Can you connect to the pods directly?
        * Is your app serving the port that you configured? I.E., if your app serves 8080, `containerPort` needs to be 8080.

