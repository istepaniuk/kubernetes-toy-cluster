# Setting up of a Kubernetes (toy) cluster

> WARNING: This is not really a step-by-step guide. YMMV. Do not try this at home. 
> Keep away from children. 

This is a sort of log I kept while ~~banging my head against the keyboard~~
setting up a home-grown kubernetes cluster in "bare-metal" nodes (actually VMs). 

## Preparation

### Create (3 or more) VMs:
  * Use bridged networking 
  * Use Debian 10/**buster**, bullseye did not work for me for 
[reasons likely related to this bug](https://github.com/kubernetes/kops/issues/7379)
  * The node VMs must have 2GB of RAM and 2 CPU cores. With less it would fail the
 "pre-flight" checks, I have not tried skipping these.
  * Assign them static IPs
  
> Using static IPs on bridged adapters is not a good idea. I did this on a "lab" environment. 
> Use IPs outside of your DHCP server's address range.

## Provisioning

You need to install docker and kubernetes. Set up the VMs so you can log in and `sudo` with no password.

> This is a good moment to make a snapshot of that VM!

Run the ansible playbook in `/cluster-provisioning/playbook.yml`.
The inventory in `/cluster-provisioning/hosts` assumes 3 nodes named kubo1, kubo2 and kubo3, and so does the rest of 
this document.  I added the static IPs from these 3 VMs in my laptop's /etc/hosts file.

## Init the cluster

It would be a good idea to make a snapshot of your VM before you continue, 
it is not always easy to just try again if something did not work. Other than 
reverting to a snapshot, this should bring you back to where you started:

   
    If things go south... 

    # kubeadm reset
    # apt remove --purge kubelet kubeadm
    # docker stop $(sudo docker ps -aq)
    # docker rm $(sudo docker ps -aq)
    # iptables -F && sudo iptables -t nat -F && sudo iptables -t mangle -F && sudo iptables -X
    # rm -rf /var/lib/etcd
    # rm -rf /etc/kubernetes
    # systemctl daemon-reload
    # apt install kubelet kubeadm


In the node that is going to have the control-plane:

    kubo1# kubeadm init --pod-network-cidr=10.244.0.0/16

## Join the rest of the nodes to the cluster

Following the instructions you got from `kubeadm init`...

    kubo2# kubeadm join [...]  
    kubo3# kubeadm join [...] 
    ...

## Set up kubectl, the Kubernetes client
    laptop$ mkdir -p $HOME/.kube

Copy the admin.conf file from the control-pane as .kube/config in your $HOME directory

    laptop$ scp root@node1:/etc/kubernetes/admin.conf $HOME/.kube/config
    laptop$ sudo chown $(id -u):$(id -g) $HOME/.kube/config

> OK... you won't likely scp from `root@ ...` but you get the idea.
 
At this point, you should be able to connect to the cluster and issue commands

    $ kubectl get nodes

... Will return a list of nodes but they will be "Pending", as you still have to set up 
networking. From here on things start getting increasingly complex.

## Set up Flannel
    $ kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

At this point, nodes should display as "Ready":

    $ kubectl get nodes
    NAME    STATUS   ROLES                  AGE     VERSION
    kubo1   Ready    control-plane,master   19m     v1.22.1
    kubo2   Ready    <none>                 8m12s   v1.22.1
    kubo3   Ready    <none>                 8m1s    v1.22.1


## Optional: remove taints
By default, the master/control-plane node has a "taint" that does not allow 
other pods to be scheduled there. If you remove this, the cluster will also
run regular pods in that node. Probably, this only makes sense in our resource-limited
toy setup.

    $ kubectl taint nodes --all node-role.kubernetes.io/master-
    node/kubo1 untainted
    taint "node-role.kubernetes.io/master" not found
    taint "node-role.kubernetes.io/master" not found

## Adding a (web UI) dashboard
    $ cd dashboard
    $ kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.3.1/aio/deploy/recommended.yaml
    $ kubectl apply -f dashboard-adminuser.yaml
    $ kubectl apply -f dashboard-cluster-role-binding.yaml


And to access the dashboard from your pc:

    $ kubectl port-forward -n kubernetes-dashboard service/kubernetes-dashboard 8443:8443

If everything goes well, you can access the dashboard by browsing to [https://localhost:8443]()

To log in, you will need a token that you can obtain with this brutal one-liner:

    $ kubectl -n kubernetes-dashboard get secret \
      $(kubectl -n kubernetes-dashboard get sa/admin-user -o jsonpath="{.secrets[0].name}") \
      -o go-template="{{ .data.token | base64decode }}" ; echo


## Adding an ingress controller
This is a Pandora's box! 

There are several ways to get a working ingress controller. 
What I settled on was to use `ingress-nginx` with `NodePort` (each node listens on a 
specific port so the "outside" traffic can reach any node that is running the controller pod). 

I also used `hostNetwork: true` (which is repeatedly discouraged through the docs) which allows
the nodes to bind to privileged ports (80 and 443).

    $ cd dashboard
    $ kubectl apply -f ingress-controller.yaml
    
Additionally, instead of a `Deployment`, I used a `DaemonSet` which forces the ingress controller
pod to run on every node (and if the taint for the master has been removed, on every node).

    $ kubectl apply -f ingress-controller-daemonset.yaml

Listing the pods with `-o wide` shows more information:

    $ kubectl get pods -n ingress-nginx  -o wide
    NAME                                      READY   STATUS      RESTARTS   AGE   IP                NODE 
    ingress-nginx-admission-create--1-zh22g   0/1     Completed   0          89m   10.244.2.3        kubo3
    ingress-nginx-admission-patch--1-bhszw    0/1     Completed   1          89m   10.244.1.4        kubo2
    ingress-nginx-controller-qglqq            1/1     Running     0          86m   192.168.178.231   kubo1
    ingress-nginx-controller-fk5b4            1/1     Running     0          89m   192.168.178.232   kubo2
    ingress-nginx-controller-6wcbc            1/1     Running     0          89m   192.168.178.233   kubo3


## Adding an Ingress for the dashboard
Now that we have an ingress controller, we could expose the dashboard to "the outside world" applying 

something like this (also in this repo in `dashboard/dashboard-ingress.yml`):

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: dashboard-ingress
  annotations:
    kubernetes.io/ingress.class: nginx
    # These annotations tell the controller that the service speaks SSL
    # Without them you'd get a very confusing error 400.
    nginx.ingress.kubernetes.io/backend-protocol: HTTPS
    nginx.ingress.kubernetes.io/configuration-snippet: |-
      proxy_ssl_server_name on;
      proxy_ssl_name $host;
  namespace: kubernetes-dashboard
spec:
  rules:
    # This is the 'Host:' header that the controller will match, think virtualhosts,
    # just a lot more complicated.
    - host: kdashboard
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: kubernetes-dashboard
                port:
                  number: 8443
```

So, doing... 
    
    $ kubectl apply -f dashboard-ingress.yaml

... You should now be able to reach the dashboard pointing your browser to [https://kdashboard/](), provided that 
that name can be locally resolved to ANY of the node IPs. (I mean, add it to your /etc/hosts).
