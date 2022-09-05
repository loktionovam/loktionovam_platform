# EX-2 Introduction to k8s. Setup a local environment. Launch a first container. Working with kubectl

* [EX-2 Introduction to k8s. Setup a local environment. Launch a first container. Working with kubectl](#ex-2-introduction-to-k8s-setup-a-local-environment-launch-a-first-container-working-with-kubectl)
  * [EX-2.1 What was done](#ex-21-what-was-done)
  * [EX-2.2 How to start the project](#ex-22-how-to-start-the-project)
  * [EX-2.3 How to check the project](#ex-23-how-to-check-the-project)
  * [EX-2.4 How to use the project](#ex-24-how-to-use-the-project)

* Main task 1: find out why all pods in kube-system namespace are restored after deletion

* Main task 2: building a web server docker image that works without root privileged, launching a pod with this image in minikube, using an init-container

## EX-2.1 What was done

* Scripts and ansible playbooks to install minikube, kind, k9s, port-forwarder
* Launched minikube and dashboard
* Checked out deletion of a pod in kube-system namespace and behavior of **static pods, deployment controlle, replicaset controller**
* Builded and pushed into dockerhub an nginx image **loktionovam/web:1.17.1-alpine** working on 8000 port without root privileges and served **/app** directory
* Developed and tested a manifest that launches two containers - **web-init** (init-container), **web** (an application)
* (advanced task *) Builded and pushed **loktionovam/hipster-frontend:v0.0.1**, fixed launch errors

* Start minikube

  ```bash
  minikube start
  ```

* Install a dashboard addon, create a service account and get a token

  ```bash
  kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-beta1/aio/deploy/recommended.yaml
  kubectl apply -f bootstrap/k8s/admin-user.yaml
  kubectl apply -f bootstrap/k8s/cluster-role-binding.yaml
  kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')
  ```

* Remove all containers

  ```bash
  minikube ssh
  docker ps
  docker rm -f $(docker ps -a -q)
  ```

* Remove all pods from kube-system namespace

  ```bash
  kubectl delete pod --all -n kube-system
  ```

* As the result the apiserver and so on will not be deleted because they are **static pods** that are started and managed from **/etc/kubernetes/manifests** by **kubelet**
  You can check it out if you place some pod manifest to this directory (in this case it is `adapter.yaml`) and restart **kubelet**. After that you can't delete this pod too!

  ```bash
  systemctl show kubelet | grep ExecStart
  ExecStart={ path=/usr/bin/kubelet ; argv[]=/usr/bin/kubelet --authorization-mode=Webhook --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --cgroup-driver=cgroupfs --client-ca-file=/var/lib/minikube/certs/ca.crt   --cluster-dns=10.96.0.10 --cluster-domain=cluster.local --container-runtime=docker --fail-swap-on=false --hostname-override=minikube --kubeconfig=/etc/kubernetes/kubelet.conf --pod-manifest-path=/etc/kubernetes/manifests ; ignore_errors=no ;   start_time=[Tue 2019-07-09 15:29:06 UTC] ; stop_time=[n/a] ; pid=3584 ; code=(null) ; status=0/0 }

  kubelet --help | grep pod-manifest-path
        --pod-manifest-path string                                                                                  Path to the directory containing static pod files to run, or the path to a single static pod file. Files starting with dots will   be ignored. (DEPRECATED: This parameter should be set via the config file specified by the Kubelet's --config flag. See https://kubernetes.io/docs/tasks/administer-cluster/kubelet-config-file/ for more information.)
  ```

  ```bash
  # ls -1 /etc/kubernetes/manifests/
  adapter.yaml
  addon-manager.yaml.tmpl
  etcd.yaml
  kube-apiserver.yaml
  kube-controller-manager.yaml
  kube-scheduler.yaml
  ```

* But `coredns` uses different approach and it will not be deleted because the pod is managed by `Deployment` which creates corresponding ReplicaSet

  ```bash
  kubectl get deployment  -n kube-system
  NAME      READY   UP-TO-DATE   AVAILABLE   AGE
  coredns   2/2     2            2           7d5h

  kubectl get rs  -n kube-system
  NAME                 DESIRED   CURRENT   READY   AGE
  coredns-5c98db65d4   2         2         2       7d5h
  ```

* kube-proxy is managed by DaemonSet controller so it will not be deleted too

  ```bash
  kubectl get ds -n kube-system
  NAME         DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR                 AGE
  kube-proxy   1         1         1       1            1           beta.kubernetes.io/os=linux   7d5h
  ```

* Build and push a web docker image

  ```bash
  docker build -t loktionovam/web:1.17.1-alpine kubernetes-intro/web
  docker push loktionovam/web:1.17.1-alpine
  ```

## EX-2.2 How to start the project

* Create a web server pod

  ```bash
  kubectl apply -f kubernetes-intro/web-pod.yaml
  ```

* Create hipster-frontend pod

  ```bash
  kubectl apply -f kubernetes-intro/frontend-pod-healthy.yaml
  ```

## EX-2.3 How to check the project

* pod web, frontend should be in Running state

  ```bash
  kubectl get pods
  kubectl get pod web -o yaml
  kubectl describe pod web
  kubectl get pod frontend -o yaml
  kubectl describe pod frontend
  ```

* After an execution of command

  ```bash
  kubectl port-forward --address 0.0.0.0 pod/web 8000:8000
  ```

  in a web browser should rendered a page with Express42 logo <http://127.0.0.1:8000/index.html>

## EX-2.4 How to use the project

* Follow the web address <http://127.0.0.1:8000/index.html>
