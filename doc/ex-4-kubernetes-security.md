# EX-4 Setup accounts and permissions, setup containers restrictions and policies

* [EX-4 Setup accounts and permissions, setup containers restrictions and policies](#ex-4-%d0%9d%d0%b0%d1%81%d1%82%d1%80%d0%be%d0%b9%d0%ba%d0%b0-%d0%b0%d0%ba%d0%ba%d0%b0%d1%83%d0%bd%d1%82%d0%be%d0%b2-%d0%b8-%d0%bf%d1%80%d0%b0%d0%b2-%d0%b4%d0%bb%d1%8f-%d0%bd%d0%b8%d1%85-%d0%bd%d0%b0%d1%81%d1%82%d1%80%d0%be%d0%b9%d0%ba%d0%b0-%d0%be%d0%b3%d1%80%d0%b0%d0%bd%d0%b8%d1%87%d0%b5%d0%bd%d0%b8%d0%b9-%d0%b8-%d0%bf%d0%be%d0%bb%d0%b8%d1%82%d0%b8%d0%ba-%d0%b1%d0%b5%d0%b7%d0%be%d0%bf%d0%b0%d1%81%d0%bd%d0%be%d1%81%d1%82%d0%b8-%d0%b4%d0%bb%d1%8f-%d0%ba%d0%be%d0%bd%d1%82%d0%b5%d0%b9%d0%bd%d0%b5%d1%80%d0%be%d0%b2)
  * [EX-4.1 What was done](#ex-41-%d0%a7%d1%82%d0%be-%d0%b1%d1%8b%d0%bb%d0%be-%d1%81%d0%b4%d0%b5%d0%bb%d0%b0%d0%bd%d0%be)
  * [EX-4.2 How to start the project](#ex-42-%d0%9a%d0%b0%d0%ba-%d0%b7%d0%b0%d0%bf%d1%83%d1%81%d1%82%d0%b8%d1%82%d1%8c-%d0%bf%d1%80%d0%be%d0%b5%d0%ba%d1%82)
  * [EX-4.3 How to check the project](#ex-43-%d0%9a%d0%b0%d0%ba-%d0%bf%d1%80%d0%be%d0%b2%d0%b5%d1%80%d0%b8%d1%82%d1%8c-%d0%bf%d1%80%d0%be%d0%b5%d0%ba%d1%82)
  * [EX-4.4 How to use the project](#ex-44-%d0%9a%d0%b0%d0%ba-%d0%bd%d0%b0%d1%87%d0%b0%d1%82%d1%8c-%d0%bf%d0%be%d0%bb%d1%8c%d0%b7%d0%be%d0%b2%d0%b0%d1%82%d1%8c%d1%81%d1%8f-%d0%bf%d1%80%d0%be%d0%b5%d0%ba%d1%82%d0%be%d0%bc)

* [x] Main task 1: create service account named bob and bind it to admin cluster role

  Create service account named dave without access to cluster

* [x] Main task 2: create prometheus namespace

  Create a Service Account named carol in this namespace Namespace

  In prometheus namespace give permissions to get, list, watch Pods of all the cluster for all Service Accounts

* Main task 3: create dev namespace

  Create a Service Account named jane in Namespace dev

  Bind jane to admin role in Namespace dev

  Create a Service Account named ken in Namespace dev

  Bind ken to view role in Namespace dev

## EX-4.1 What was done

There were developed:

* **bob** and **dave** service accounts manifests
* **admin** - **bob** binding manifest
* **prometheus namespace** manifest
* **carol service account** manifest to create this service account in **prometheus namespace**
* **pod-reader** cluster role manifest that allows to get, list, watch of all pods in a whole cluster
* **pod-reader** manifest that binds the cluster role to all service accounts in **prometheus namespace**
* **dev namespace** manifest
* **jane service account** manifest that creates sa in **dev namespace**
* a manifest to bind **jane service account** to admin role in **dev namespace**
* a manifest to create **ken service account** in **dev namespace**
* a manifest to bind **ken service account** to view role in **dev namespace**
* script `misc/scripts/check_service_account_priviledges.sh` which prints privileges of given service account of given namespace

## EX-4.2 How to start the project

* Create **bob и dave** service accounts, bind **admin** role to **bob** sa for the whole cluster

  ```bash
  cat kubernetes-security/task01/*.yaml | kubectl apply -f -
  ```

* Create **prometheus namespace** and **carol service account** which can be able to get, list, watch for all pods in the cluster

  ```bash
  cat kubernetes-security/task02/*.yaml | kubectl apply -f -
  ```

* Create **dev namespace**. Bind **admin** role to **jane service account** and bind **view** role to **ken service account** in this namespace

  ```bash
  cat kubernetes-security/task03/*.yaml | kubectl apply -f -
  ```

## EX-4.3 How to check the project

* To check **carol service account** out a command `kubectl auth can-i...` should print **yes**

  ```bash
  for VERB in get list watch; do kubectl auth can-i $VERB pods --as system:serviceaccount:prometheus:carol; done
  yes
  yes
  yes
  ```

* To check **jane service account** out you can use `misc/scripts/check_service_account_priviledges.sh` in **dev** and **default** namespaces

  ```bash
  # dev namespace
  misc/scripts/check_service_account_priviledges.sh -n dev -s system:serviceaccount:dev:jane
  system:serviceaccount:dev:jane can create in dev: yes
  system:serviceaccount:dev:jane can delete in dev: yes
  system:serviceaccount:dev:jane can deletecollection in dev: yes
  system:serviceaccount:dev:jane can get in dev: yes
  system:serviceaccount:dev:jane can list in dev: yes
  system:serviceaccount:dev:jane can patch in dev: yes
  system:serviceaccount:dev:jane can update in dev: yes
  system:serviceaccount:dev:jane can watch in dev: yes
  ```

  ```bash
  # in default namespace jane can nothing
  misc/scripts/check_service_account_priviledges.sh -n default -s system:serviceaccount:dev:jane
  system:serviceaccount:dev:jane can create in default: no
  system:serviceaccount:dev:jane can delete in default: no
  system:serviceaccount:dev:jane can deletecollection in default: no
  system:serviceaccount:dev:jane can get in default: no
  system:serviceaccount:dev:jane can list in default: no
  system:serviceaccount:dev:jane can patch in default: no
  system:serviceaccount:dev:jane can update in default: no
  system:serviceaccount:dev:jane can watch in default: no
  ```

* To check **ken service account** out you can use `misc/scripts/check_service_account_priviledges.sh` in **dev** and **default** namespaces

  ```bash
  # in default namespace ken can nothing
  misc/scripts/check_service_account_priviledges.sh -n default -s system:serviceaccount:dev:ken
  system:serviceaccount:dev:ken can create in default: no
  system:serviceaccount:dev:ken can delete in default: no
  system:serviceaccount:dev:ken can deletecollection in default: no
  system:serviceaccount:dev:ken can get in default: no
  system:serviceaccount:dev:ken can list in default: no
  system:serviceaccount:dev:ken can patch in default: no
  system:serviceaccount:dev:ken can update in default: no
  system:serviceaccount:dev:ken can watch in default: no
  ```

  ```bash
  # in dev namespace ken can read-only priviledges
  misc/scripts/check_service_account_priviledges.sh -n dev -s system:serviceaccount:dev:ken
  system:serviceaccount:dev:ken can create in dev: no
  system:serviceaccount:dev:ken can delete in dev: no
  system:serviceaccount:dev:ken can deletecollection in dev: no
  system:serviceaccount:dev:ken can get in dev: yes
  system:serviceaccount:dev:ken can list in dev: yes
  system:serviceaccount:dev:ken can patch in dev: no
  system:serviceaccount:dev:ken can update in dev: no
  system:serviceaccount:dev:ken can watch in dev: yes
  ```

## EX-4.4 How to use the project

The exercises for creating **service accounts** are synthetic and do not involve "using the project"
