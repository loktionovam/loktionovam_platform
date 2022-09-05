# EX-5 Network interaction of Pods and Services

* [EX-5 Network interaction of Pods and Services](#ex-5-%d0%a1%d0%b5%d1%82%d0%b5%d0%b2%d0%be%d0%b5-%d0%b2%d0%b7%d0%b0%d0%b8%d0%bc%d0%be%d0%b4%d0%b5%d0%b9%d1%81%d1%82%d0%b2%d0%b8%d0%b5-pod-%d1%81%d0%b5%d1%80%d0%b2%d0%b8%d1%81%d1%8b)
  * [EX-5.1 What was done](#ex-51-%d0%a7%d1%82%d0%be-%d0%b1%d1%8b%d0%bb%d0%be-%d1%81%d0%b4%d0%b5%d0%bb%d0%b0%d0%bd%d0%be)
  * [EX-5.2 How to start the project](#ex-52-%d0%9a%d0%b0%d0%ba-%d0%b7%d0%b0%d0%bf%d1%83%d1%81%d1%82%d0%b8%d1%82%d1%8c-%d0%bf%d1%80%d0%be%d0%b5%d0%ba%d1%82)
  * [EX-5.3 How to check the project](#ex-53-%d0%9a%d0%b0%d0%ba-%d0%bf%d1%80%d0%be%d0%b2%d0%b5%d1%80%d0%b8%d1%82%d1%8c-%d0%bf%d1%80%d0%be%d0%b5%d0%ba%d1%82)
  * [EX-5.4 How to use the project](#ex-54-%d0%9a%d0%b0%d0%ba-%d0%bd%d0%b0%d1%87%d0%b0%d1%82%d1%8c-%d0%bf%d0%be%d0%bb%d1%8c%d0%b7%d0%be%d0%b2%d0%b0%d1%82%d1%8c%d1%81%d1%8f-%d0%bf%d1%80%d0%be%d0%b5%d0%ba%d1%82%d0%be%d0%bc)

* [x] Main task 1: working with the test application (adding checks to pod, creating of a deployment, adding services to a cluster (ClusterIP), enabling IPVS loadbalancing)
* [x] Main task 2: access to the application outside from the cluster (metallb installation in layer2 mode, adding LoadBalancer service, Ingress-controller installation and `ingress-nginx` proxy, creation `Ingress` rules)
* [x] Advanced task 1 (*): create LoadBalancer that opens access to CoreDNS outside of the cluster
* [x] Advanced task 2 (*): access to `dashboard` via nginx ingress
* [x] Advanced task 3 (*): canary deployment of `web` application by using a HTTP header, via `nginx ingress`

## EX-5.1 What was done

* Developed `ClusterIP` manifest for internal loadbalancing of the traffic for web application
* Enabled and checked out `ipvs` instead of `iptables`
* Created metallb `LoadBalancer` service
* (*) Developed `LoadBalancer` manifest to access to CoreDNS outside of the cluster
* Added shell, python, yaml linters. Fixed warnings and errors
* Developed `nginx ingress` manifests that work via `MetalLB` and proxying the requests to the web application
* This configuration is valid and the checks will always have `0` exit code (if a container has a shell, ps and grep utilities)

```yaml
livenessProbe:
  exec:
    command:
      - 'sh'
      - '-c'
      - 'ps aux | grep my_web_server_process'
```

* (*) Developed manifests to access to `kubernetes dashboard` outside of the cluster
* (*) Builded `web-python:latest` image which imitates a new version of the application (the main image uses `nginx`).
  Developed manifests to canary deploy `web-python:latest` via `nginx ingress`

## EX-5.2 How to start the project

* In the root of the project:

```bash
minikube start
minikube addons disable dashboard
kubectl delete clusterrolebinding kubernetes-dashboard
kubectl apply -f https://raw.githubusercontent.com/google/metallb/v0.8.3/manifests/metallb.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/mandatory.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-rc2/aio/deploy/recommended.yaml
docker build kubernetes-networks/canary/web-python/ -t loktionovam/web-python:latest
docker push loktionovam/web-python:latest
find kubernetes-networks -name "*.yaml" -exec kubectl apply -f  '{}' \;
```

## EX-5.3 How to check the project

* The linters must work without errors:

  ```bash
  misc/scripts/lint_project.py
  ...
  Lint project: OK
  ```

* Add the route on the host to a minikube VM

  ```bash
  sudo ip r add to 172.17.255.0/24 via 192.168.99.1
  ```

* Check DNS out:

  ```bash
  misc/scripts/check_external_dns_address.sh
  # if there is an access to the DNS outside of the cluster the script will print 'External CoreDNS is OK'
  web-svc-cip.default.svc.cluster.local has address 10.96.229.152
  External CoreDNS is OK
  ```

* Check `web` application out:

  ```bash
  misc/scripts/check_nginx_ingress_web.sh
  'Web' application (prod) is OK via nginx ingress
  ```

* Check out the canary deployment of the `web` application:

  ```bash
  misc/scripts/check_nginx_canary_ingress.sh
  'Web' application (v2, canary) is OK via nginx ingress
  ```

## EX-5.4 How to use the project
