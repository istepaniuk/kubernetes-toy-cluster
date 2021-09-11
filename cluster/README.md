istepaniuk@kubo1:~$ 
  sudo kubeadm init --pod-network-cidr=10.244.0.0/16
  kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

...
istepaniuk@kubo2:~$ sudo kubeadm join 
istepaniuk@kubo3:~$ sudo kubeadm join 


istepaniuk@kubo1:~$
  kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.3.1/aio/deploy/recommended.yaml
  

apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard

  kubectl apply -f dashboard-adminuser.yaml

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
  
  kubectl apply -f cluster-role-binding.yaml


local:
kubectl port-forward -n kubernetes-dashboard service/kubernetes-dashboard 8080:443
