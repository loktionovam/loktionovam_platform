# EX-3 Kubernetes controllers. ReplicaSet, Deployment, DaemonSet

* [EX-3 Kubernetes controllers. ReplicaSet, Deployment, DaemonSet](#ex-3-kubernetes-controllers-replicaset-deployment-daemonset)
  * [EX-3.1 What was done](#ex-31-what-was-done)
  * [EX-3.2 How to start the project](#ex-32-how-to-start-the-project)
  * [EX-3.3 How to check the project](#ex-33-how-to-check-the-project)
  * [EX-3.4 How to use the project](#ex-34-how-to-use-the-project)

* [x] Main task 1: write a **ReplicaSet** for **hipster-frontend** and **hipster-paymentservice**. Check out pod management and updating via ReplicaSet and explain why does updating of ReplicaSet not lead to updating of pods.

* [x] Main task 2: write a **Deployment** for **hipster-frontend** and **hipster-paymentservice**. Check out **Rolling Update** via these deployments. Check out a rollback.****
* [x] Advanced task 1 (*): **Blue-green deployment** and **Reverse Rolling Update**
* [x] Main task 3: working with **probes**. Rollout and rollback of a broken application
* [x] Advanced task (*): develop a **DaemonSet** to install **prometheus node exporter**
* [x] Advanced task (**): deployment of **node exporter** to master nodes. Taints и tolerations

## EX-3.1 What was done

* Deployed a k8s kind cluster (3 worker and 3 master nodes)
* Created and updated **hipster-frontend** **replicaset**. Checked out an automatic pod restarting after deletion
  The updating of **ReplicaSet** does not lead to updating already launched **pods** is explained in **ReplicaSet** documentation
  <https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/#example>

  ```plain
  Once the original is deleted, you can create a new ReplicaSet to replace it. As long as the old and new .spec.selector are the same,
  then the new one will adopt the old Pods. However, it will not make any effort to make existing Pods match a new, different pod template.
  To update Pods to a new spec in a controlled way, use a Deployment, as ReplicaSets do not support a rolling update directly.
  ```

  i.e. **ReplicaSet** can not update already existing **Pods** after an image tag was changed. Instead use **Deployment** for this.

* Created, updated and tested **hipster-frontend** **deployment**
* Builded and pushed **hipster-paymentservice** image.
* Developed and tested **Blue green deploy** and **Reverse Rolling update** manifests.
* Tested the updating of "buggy" application via **hipster-frontend** **readiness probe**
* Developed node-exporter `DaemonSet` manifest that supports deployment to control plane nodes:
  **Control plane nodes** have **taints**:

  ```yaml
      taints:
      - effect: NoSchedule
        key: node-role.kubernetes.io/master
  ```

  and this means that the **pod** do not be scheduled to the master nodes.

  It described in documentation **Taints and Tolerations** <https://kubernetes.io/docs/concepts/configuration/taint-and-toleration/>:

  ```plain
  The taint has key key, value value, and taint effect NoSchedule. This means that no pod will be able to schedule onto node1 unless it has a matching toleration.
  ```

  so if you would like to disable **NoSchedule** and start the **pod** on the master nodes you have to add **tolerations**:

  ```yaml
        tolerations:
          - key: node-role.kubernetes.io/master
            effect: NoSchedule
  ```

## EX-3.2 How to start the project

Apply the manifests from `kubernetes-controllers` directory:

```bash
kubectl apply -f paymentservice-deployment-bg.yaml
kubectl apply -f frontend-deployment.yaml
kubectl apply -f node-exporter-daemonset.yaml
```

## EX-3.3 How to check the project

* Check that the deployments are started and in the ready state:

  ```bash
  kubectl get deployments.apps
  NAME             READY   UP-TO-DATE   AVAILABLE   AGE
  frontend         3/3     3            3           123m
  paymentservice   3/3     3            3           3h20m
  ```

* Check that the daemonsets are started and in the ready state:

  ```bash
  kubectl get daemonsets.apps
  NAME            DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
  node-exporter   6         6         6       6            6           <none>          101m
  ```

## EX-3.4 How to use the project

* Get node exporter metrics:

  ```bash
  kubectl port-forward <node-exporter-pod-here> 9100:9100
  curl localhost:9100/metrics
  ```
