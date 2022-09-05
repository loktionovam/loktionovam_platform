# EX-8 Templating Kubernetes manifests

* [EX-8 Templating Kubernetes manifests](#ex-8-%d0%a8%d0%b0%d0%b1%d0%bb%d0%be%d0%bd%d0%b8%d0%b7%d0%b0%d1%86%d0%b8%d1%8f-%d0%bc%d0%b0%d0%bd%d0%b8%d1%84%d0%b5%d1%81%d1%82%d0%be%d0%b2-kubernetes)
  * [EX-8.1 What was done](#ex-81-%d0%a7%d1%82%d0%be-%d0%b1%d1%8b%d0%bb%d0%be-%d1%81%d0%b4%d0%b5%d0%bb%d0%b0%d0%bd%d0%be)
  * [EX-8.2 How to start the project](#ex-82-%d0%9a%d0%b0%d0%ba-%d0%b7%d0%b0%d0%bf%d1%83%d1%81%d1%82%d0%b8%d1%82%d1%8c-%d0%bf%d1%80%d0%be%d0%b5%d0%ba%d1%82)
  * [EX-8.3 How to check the project](#ex-83-%d0%9a%d0%b0%d0%ba-%d0%bf%d1%80%d0%be%d0%b2%d0%b5%d1%80%d0%b8%d1%82%d1%8c-%d0%bf%d1%80%d0%be%d0%b5%d0%ba%d1%82)
  * [EX-8.4 How to use the project](#ex-84-%d0%9a%d0%b0%d0%ba-%d0%bd%d0%b0%d1%87%d0%b0%d1%82%d1%8c-%d0%bf%d0%be%d0%bb%d1%8c%d0%b7%d0%be%d0%b2%d0%b0%d1%82%d1%8c%d1%81%d1%8f-%d0%bf%d1%80%d0%be%d0%b5%d0%ba%d1%82%d0%be%d0%bc)

## EX-8.1 What was done

* Added installation of gcloud, terraform, tflint, helm3, helmfile, kubecfg, qbec, jsonnet to the bootstrap scripts
* Added a GKE terraform configuration
* Main task 1: Added `ClusterIssuer` manifest for cert-manager
* Main task 2: Added `values.yaml` for `chartmuseum`
* (*) Advanced task 1: Working with `chartmuseum`
* Main task 3: installing `harbor` with enabled TLS and a valid certificate
* (*) Advanced task 2: developing a `helmfile` to install `nginx-ingress`, `cert-manager`, `harbor`
* Добавлены `hipster-shop` и `frontend` helm chart
* (*) Advanced task 3: usage of redis community chart as a dependency for `hipster-shop` chart
* Main task 4: secrets for `frontend` chart, working with `helm secrets`, described working with secrets.
* Main task 5: services `paymentservice`, `shippingservice` are templated via `kubecfg`
* (*) Advanced task 4: service `adservice` is templated via `qbec`
* Main task 6: `cartservice` is templated via `kustomize`

How to work with `helm secrets`:

* You can check the secrets out:

  ```bash
  get secrets sh.helm.release.v1.frontend.v2  -n hipster-shop  -o=jsonpath='{.data.release}' | base64 --decode
  ```

* The way of the usage in CI/CD is similar to the `ansible-vault` - the encrypted data is stored in the repository and CI/CD environment variable stores the key `(masked, protected)` to decrypt this data while a CI/CD process is running:
* Usage of the secrets which are located in the repository is not good practice because you can accidentally commit them and thus you must rewrite a project commit history which can greatly impact other participants of a development process. On the other hand storing them outside of the repository might be inconvenient so as a compromise decision you can:
  * Create a directory named`secrets` inside the repository where there will be `.gitignore`:

    ```bash
    # assume that the data inside this directory must not be committed and pushed to a remote repository
    cat secrets/.gitignore
    *
    !.gitignore
    ```

    This explicitly points to which directory contains the secret data (without "spreading" them on the whole repository) and decreases the risk of errors in the root `.gitignore`. For instance, if we rename `secrets` directory to `ssl` the excluding rules will be automatically saved without additional rewrites of `.gitignore`.

  * Use git hooks for filtering secret data <https://github.com/futuresimple/helm-secrets#important-tips>

    You can combine these two methods

## EX-8.2 How to start the project

* Start GKE cluster

  ```bash
  cd infra/kubernetes/terraform
  terraform init
  terraform apply
  cd gke
  terraform init
  terraform apply
  ....
  Outputs:

  kubernetes_endpoint = ip_address_here
  ```

* Setup kubectl to use GKE

  ```bash
  gcloud beta container clusters get-credentials primary --zone <zone_here>
  ```

* Install nginx-ingress

  ```bash
  helm repo add stable https://kubernetes-charts.storage.googleapis.com
  helm repo list
  kubectl apply -f kubernetes-templating/nginx-ingress/namespace.yaml
  helm upgrade --install nginx-ingress stable/nginx-ingress --wait --namespace=nginx-ingress --version=1.11.1
  ```

* Install `cert-manager`

  ```bash
  helm repo add jetstack https://charts.jetstack.io
  kubectl apply -f https://raw.githubusercontent.com/jetstack/cert-manager/release-0.9/deploy/manifests/00-crds.yaml
  kubectl apply -f kubernetes-templating/cert-manager/namespace.yaml
  kubectl label namespace cert-manager certmanager.k8s.io/disable-validation="true"
  helm upgrade --install cert-manager jetstack/cert-manager --wait --namespace=cert-manager --version=0.9.0
  kubectl apply -f kubernetes-templating/cert-manager/cluster-issuer.yaml
  ```

* Install `chartmuseum`

  ```bash
  kubectl apply -f kubernetes-templating/chartmuseum/namespace.yaml
  helm upgrade --install chartmuseum stable/chartmuseum --wait --namespace=chartmuseum --version=2.3.2 -f kubernetes-templating/chartmuseum/values.yaml
  ```

* (*) Working with the `chartmuseum`. Use `misc/scripts/check_chartmuseum.sh` to create a test chart, upload it to the `chartmuseum`, print the information about the uploaded artifact, add the `chartmuseum` repository in `dry-run` mode, "install" the chart into k8s, and finally delete the chart from the `chartmuseum`

  ```bash
  # we assume that the access to API is granted (look to chartmuseum's values.yaml )
  misc/scripts/check_chartmuseum.sh
  Create test helm package check_chartmuseum
  Creating check_chartmuseum
  Successfully packaged chart and saved it to: /tmp/tmp.yHpOUv1FC0/check_chartmuseum/check_chartmuseum-0.1.0.tgz
  Upload test helm package check_chartmuseum-0.1.0.tgz to the https://chartmuseum.35.240.65.117.nip.io/api/charts
  Get information about uploaded chart
  [
    {
      "name": "check_chartmuseum",
      "version": "0.1.0",
      "description": "A Helm chart for Kubernetes",
      "apiVersion": "v2",
      "appVersion": "1.16.0",
      "urls": [
        "charts/check_chartmuseum-0.1.0.tgz"
      ],
      "created": "2020-02-02T14:07:19.521814922Z",
      "digest": "9644296cefc22aa8e4be717a85d648e5524d8b17bc43c75a4af10673ee9c9902"
    }
  ]
  Try to add chartmuseum repo to helm and install test chart
  "chartmuseum" has been added to your repositories
  NAME: 0.1.0
  LAST DEPLOYED: Sun Feb  2 17:20:29 2020
  NAMESPACE: default
  STATUS: pending-install
  REVISION: 1
  HOOKS:
  ---
  # Source: check_chartmuseum/templates/tests/test-connection.yaml
  apiVersion: v1
  ...
  ...
  ...
  Remove check_chartmuseum from https://chartmuseum.35.240.65.117.nip.io
  {
    "deleted": true
  }
  Remove chartmuseum repo from helm repositories list
  "chartmuseum" has been removed from your repositories
  ```

* Install the `harbor`

  ```bash
  helm repo add harbor https://helm.goharbor.io
  kubectl apply -f kubernetes-templating/harbor/namespace.yaml
  helm upgrade --install harbor harbor/harbor --atomic --namespace=harbor --version=1.1.2  -f kubernetes-templating/harbor/values.yaml
  ```

* Install the `harbor` via `helmfile`

  ```bash
  cd kubernetes-templating/helmfile/
  helmfile --log-level=debug --interactive apply
  ```

* Install the `hipster-shop`

  ```bash
  kubectl create ns hipster-shop
  helm upgrade --install hipster-shop --atomic kubernetes-templating/hipster-shop --namespace hipster-shop
  # helm dep update kubernetes-templating/hipster-shop
  ```

* Crete a `hipster-shop` `helm` package

  ```bash
  helm package kubernetes-templating/hipster-shop
  ```

* Install the `paymentservice`, `shippingservice` via `kubecfg`:

  ```bash
  cd kubernetes-templating/kubecfg/
  kubecfg update services.jsonnet --namespace hipster-shop
  ```

* Install the `adservice` via `qbec` (a `how-to` about `qbec` is here <https://habr.com/ru/post/481662/#qbec>)

  ```bash
  cd kubernetes-templating/jsonnet/qbec/adservice/
  qbec show default
  qbec apply default
  ```

* Install the `cartservice` via `kustomize`

  ```bash
  kubectl kustomize kubernetes-templating/kustomize/overrides/default/  | kubectl apply -f -
  ```

## EX-8.3 How to check the project

* Check out the address of installed `harbor`:

  ```bash
  kubectl get ingress -n harbor  -o jsonpath='{.items[*].spec.rules[*].host}'
  ```

* Check out the `hipster-shop`:

  ```bash
  kubectl get ingress -n hipster-shop  -o jsonpath='{.items[*].spec.rules[*].host}'
  ```

## EX-8.4 How to use the project
