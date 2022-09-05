# EX-9 Custom Resource Definitions. Operators

* [EX-9 Custom Resource Definitions. Operators](#ex-9-custom-resource-definitions-operators)
  * [EX-9.1 What was done](#ex-91-what-was-done)
  * [EX-9.2 How to start the project](#ex-92-how-to-start-the-project)
  * [EX-9.3 How to check the project](#ex-93-how-to-check-the-project)
  * [EX-9.4 How to use the project](#ex-94-how-to-use-the-project)

## EX-9.1 What was done

* Main task 1: develop and apply CR, CRD for mysql with scheme validation
* Main task2: add mandatory fields to the CRD
* Main task3: develop mysql operator to manage a persistent volume, a persistent volume claim, a deployment and a service
* Main task4: deploy mysql operator
* Advanced task 1 (*): update `subresource status` via the operator
* Advanced task 2 (*): if password in mysql-instance resource is changed a mysql password of the deployed instance should be changed too

The question: why was the objects created despite the fact that CR had been created BEFORE the operator was started?

The answer: the object was created by the operator because every time when the kopf base operator is started it checks the state of served resources and if the state is changed since the last start then a new reconciliation cycle is started that is knows as `level triggering`

```plain
If the operator is down and not running, any changes to the objects are ignored and not handled. They will be handled when the operator starts: every time a Kopf-based operator starts, it lists all objects of the served resource kind, and checks for their state; if the state has changed since the object was last handled (no matter how long time ago), a new handling cycle starts.

Only the last state is taken into account. All the intermediate changes are accumulated and handled together. This corresponds to the Kubernetes’s concept of eventual consistency and level triggering (as opposed to edge triggering).
```

This is described here: <https://kopf.readthedocs.io/en/latest/continuity/#downtime>

## EX-9.2 How to start the project

```bash
kubectl apply -f kubernetes-operators/deploy/crd.yml
kopf run kubernetes-operators/build/mysql_operator.py
kubectl apply -f kubernetes-operators/deploy/cr.yml
misc/scripts/fill_mysql_instance.sh
```

## EX-9.3 How to check the project

* Remove the started project and recreate it again - the data will be saved and restored via the backup/restore jobs

  ```bash
  # remove started mysql
  delete mysqls.otus.homework mysql-instance

  # recreate it again
  kubectl apply -f kubernetes-operators/deploy/cr.yml

  # check the data out
  export MYSQLPOD=$(kubectl get pods -l app=mysql-instance -o jsonpath="{.items[*].metadata.name}")
  kubectl exec -it $MYSQLPOD -- mysql -potuspassword -e "select * from test;" otus-database
  mysql: [Warning] Using a password on the command line interface can be insecure.
  +----+-------------+
  | id | name        |
  +----+-------------+
  |  1 | some data   |
  |  2 | some data-2 |
  +----+-------------+
  ```

* Check that the backup/restore jobs completed without errors:

  ```bash
  kubectl get jobs
  NAME                         COMPLETIONS   DURATION   AGE
  backup-mysql-instance-job    1/1           1s         111s
  restore-mysql-instance-job   1/1           20s        75s
  ```

* (*) To update `status subresources` we have to enable `/apis/otus.homework/v1/namespaces/default/mysqls/mysql-instance/status` endpoint in the crd:

  ```yaml
    subresources:
    status: {}
  ```

  and update the status by invoking `patch_namespaced_custom_object_status`:

  ```python
    ...
    ...
        status = {
            "status": {
                'kopf': {
                    'message': 'mysql-instance created WITH restore-job'}}}
    ...
    ...
      api = kubernetes.client.CustomObjectsApi()
    try:
        crd_status = api.patch_namespaced_custom_object_status(group, version, namespace, plural, name, body=status)
        logging.info(crd_status)
    except kubernetes.client.rest.ApiException as e:
        print("Exception when calling CustomObjectsApi->get_namespaced_custom_object: %s\n" % e)
  ```

  Check the `status subresource` `/apis/otus.homework/v1/namespaces/default/mysqls/mysql-instance/status` endpoint:

  ```bash
  kubectl proxy --port=8080
  curl http://localhost:8080/apis/otus.homework/v1/namespaces/default/mysqls/mysql-instance/status 2>/dev/null | jq .status
  {
    "kopf": {
      "message": "mysql-instance created WITHOUT restore-job"
    }
  }
  ```

  It also will be printed in `kubectl describe`:

  ```bash
  kubectl describe mysqls.otus.homework mysql-instance
  Name:         mysql-instance
  Namespace:    default
  Labels:       <none>
  Annotations:  kopf.zalando.org/last-handled-configuration:
                  {"spec": {"database": "otus-database", "image": "mysql:5.7", "password": "otuspassword", "storage_size": "1Gi"}}
                kubectl.kubernetes.io/last-applied-configuration:
                  {"apiVersion":"otus.homework/v1","kind":"MySQL","metadata":{"annotations":{},"name":"mysql-instance","namespace":"default"},"spec":{"datab...
  API Version:  otus.homework/v1
  Kind:         MySQL
  Metadata:
    Creation Timestamp:  2020-03-09T15:57:40Z
    Finalizers:
      kopf.zalando.org/KopfFinalizerMarker
    Generation:        1
    Resource Version:  174341
    Self Link:         /apis/otus.homework/v1/namespaces/default/mysqls/mysql-instance
    UID:               0a69a0e9-b820-4f20-8fe8-585fa638b624
  Spec:
    Database:      otus-database
    Image:         mysql:5.7
    Password:      otuspassword
    storage_size:  1Gi
  Status:
    Kopf:
      Message:  mysql-instance created WITHOUT restore-job
  Events:
    Type    Reason   Age   From  Message
    ----    ------   ----  ----  -------
    Normal  Logging  16m   kopf  All handlers succeeded for creation.
    Normal  Logging  16m   kopf  Handler 'mysql_on_create' succeeded.

  ```

* (*) Password autochange is implemented by usage of `@kopf.on.update` decorator. It similar to `@kopf.on.delete` and the old password is stored in the resource annotation (**do not do this in production**!) `kopf.zalando.org/last-handled-configuration`:

  ```bash
  # change the password
  kubectl apply -f kubernetes-operators/deploy/cr-passwd.yml
  ```

  ```bash
  # the event is processed
  [2020-03-05 23:13:57,381] root                 [INFO    ] Old password: 'otuspassword'
  [2020-03-05 23:13:57,381] root                 [INFO    ] New password: 'newpassword'
  [2020-03-05 23:13:57,381] root                 [INFO    ] otus-database
  [2020-03-05 23:13:57,388] root                 [INFO    ] {'apiVersion': 'batch/v1', 'kind': 'Job', 'metadata': {'namespace': 'default', 'name': 'passwd-mysql-instance-job'}, 'spec': {'template': {'metadata': {'name': 'passwd-mysql-instance-job'}, 'spec': {'restartPolicy': 'OnFailure', 'containers': [{'name': 'passwd-mysql-instance', 'image': 'mysql:5.7', 'imagePullPolicy': 'IfNotPresent', 'command': ['/bin/sh', '-c', 'mysql -u root -h mysql-instance -potuspassword -e "UPDATE mysql.user SET authentication_string=PASSWORD(\'newpassword\') WHERE User=\'root\'; FLUSH PRIVILEGES;";']}]}}}}
  job with passwd-mysql-instance-job  found,wait untill end
  job with passwd-mysql-instance-job  found,wait untill end
  job with passwd-mysql-instance-job  success
  [2020-03-05 23:13:59,445] kopf.objects         [INFO    ] [default/mysql-instance] Handler 'update_object_password' succeeded.
  [2020-03-05 23:13:59,445] kopf.objects         [INFO    ] [default/mysql-instance] All handlers succeeded for update.
  ```

  ```bash
  # connect with the new password
  kubectl exec -ti mysql-instance-6c76bcf945-vngx8 -- mysql -u root -pnewpassword -e 'show databases;'
  mysql: [Warning] Using a password on the command line interface can be insecure.
  +--------------------+
  | Database           |
  +--------------------+
  | information_schema |
  | mysql              |
  | otus-database      |
  | performance_schema |
  | sys                |
  +--------------------+

  ```

## EX-9.4 How to use the project
