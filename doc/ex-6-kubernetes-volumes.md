# EX-6 Volumes, Storages, StatefulSet

* [EX-6 Volumes, Storages, StatefulSet](#ex-6-volumes-storages-statefulset)
  * [EX-6.1 What was done](#ex-61-what-was-done)
  * [EX-6.2 How to start the project](#ex-62-how-to-start-the-project)
  * [EX-6.3 How to check the project](#ex-63-how-to-check-the-project)
  * [EX-6.4 How to use the project](#ex-64-how-to-use-the-project)

* [x] Main task 1: create a statefulset
* [x] Advanced task (*): use `kubernetes secrets` to storing MinIO secret data

## EX-6.1 What was done

* Developed `minio-statefulset.yaml`, created MinIo StatefulSet
* Developed `minio-headless-service.yaml`, created Headless Service to access to a StatefulSet inside of the cluster
* Added `misc/scripts/generate_minio_secret.sh` to generate the MinIO secrets
* (*) Added secrets usage to MinIO StatefulSet
* (*) Added `minio-client.yaml` and `misc/scripts/check_minio.sh` to check that the secrets work and the MinIO server also works

## EX-6.2 How to start the project

* Create a MinIO secret by `misc/scripts/generate_minio_secret.sh`. The script only prints the manifest to a stdout and does not create the secret:

  ```bash
  misc/scripts/generate_minio_secret.sh <minio_access_key_here> <minio_secret_key_here> | kubectl apply -f -
  ```

* Create MinIO StatefulSet and a service:

  ```bash
  kubectl apply -f kubernetes-volumes/minio-statefulset.yaml
  kubectl apply -f kubernetes-volumes/minio-headless-service.yaml
  ```

## EX-6.3 How to check the project

* Execute `misc/scripts/check_minio.sh` from the root of the project which creates a Pod with `mc`, creates `bucket` in which `alpine-release` file will be copied and prints the `bucket` listing. On success you will see:

  ```bash
  misc/scripts/check_minio.sh

  pod/minio-client created
  pod/minio-client condition met
  Bucket created successfully `minio/kubernetes-volumes`.
  /etc/alpine-release:  7 B / 7 B  5.29 KiB/s 0s[2020-01-30 19:18:34 UTC]      7B alpine-release
  ```

## EX-6.4 How to use the project
