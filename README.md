# Deploying Trend Micro Smart Check on OpenShift

- [Deploying Trend Micro Smart Check on OpenShift](#deploying-trend-micro-smart-check-on-openshift)
  - [Getting Started](#getting-started)
    - [Create a Project](#create-a-project)
    - [Permissions and SecurityContextConstraints](#permissions-and-securitycontextconstraints)
    - [Create a Database](#create-a-database)
    - [Deploy Smart Check](#deploy-smart-check)
  - [Result](#result)
  - [Support](#support)
  - [Contribute](#contribute)

Get started running Smart Check on OpenShift.

Tested on CodeReadyContainers 4.10.

## Getting Started

### Create a Project

Create a project for Smart Check. For simplicity, we're going to deploy the database into the same project later on.

```sh
oc new-project smartcheck
```

### Permissions and SecurityContextConstraints

Create new Security Context Constraints for Smart Check and PostgreSQL. The defined one is a copy from the default restricted with a changed `priority` and `runAsUser` value. Assign the scc to the default service user within the project.  

```sh
cat <<EOF | oc apply -f -
kind: SecurityContextConstraints
apiVersion: security.openshift.io/v1
metadata:
  name: smartcheck
allowHostDirVolumePlugin: false
allowHostIPC: false
allowHostNetwork: false
allowHostPID: false
allowHostPorts: false
allowPrivilegeEscalation: true
allowPrivilegedContainer: false
allowedCapabilities: null
apiVersion: security.openshift.io/v1
defaultAddCapabilities: null
fsGroup:
  type: RunAsAny
groups:
- system:authenticated
priority: 10
readOnlyRootFilesystem: false
requiredDropCapabilities:
- KILL
- MKNOD
- SETUID
- SETGID
runAsUser:
  type: RunAsAny
seLinuxContext:
  type: RunAsAny
supplementalGroups:
  type: RunAsAny
users: []
volumes:
- configMap
- downwardAPI
- emptyDir
- persistentVolumeClaim
- projected
- secret
EOF

oc adm policy add-scc-to-user smartcheck -z default
```

A template for a PostgreSQL Database is available under the `postgresql` name in the Registry used by CRC. We will use it to deploy our database shortly, but if using it as it is, you will run into an issue with `mkdir: can't create directory '/var/lib/postgresql/data/pgdata': Permission denied`. To solve this, we give the postgresql some more privileges ;-)

```sh
oc adm policy add-scc-to-user privileged -z postgres
```

> In production environments one would typically use something like AWS RDS or similar.

### Create a Database

Next, setup a PostgreSQL. Therefore, you can create a new PostgreSQL application as follows:

```sh
oc process -n openshift postgresql-persistent -o yaml \
  -p POSTGRESQL_USER=dssc \
  -p POSTGRESQL_PASSWORD=dssc \
  -p POSTGRESQL_DATABASE=dssc \
  > postgresql.yaml
```

modify the postgresql.yaml

```yaml
- apiVersion: apps.openshift.io/v1
  kind: DeploymentConfig
  ...
  spec:
    replicas: 1
    selector:
      name: postgresql
    strategy:
      type: Recreate
    template:
      metadata:
        labels:
          name: postgresql
      spec:
        containers:
        ...
          securityContext:
            capabilities: {}
            privileged: false
          # add service account
          serviceAccount: postgres
          serviceAccountName: postgres
          terminationMessagePath: /dev/termination-log
        ...
```

Create the database with

```sh
oc create -f postgresql.yaml
```

The created user within the postgresql does not have the required permissions `create database` and `create role` which we need to grant to the user dssc. Connect to the database with

```sh
oc rsh $(oc get pods -o json | jq -r '.items[].metadata | select(.name | startswith("postgres")) | .name' | grep -v deploy)
```

Within the new shell...

```sh
psql
\du
```

```sh
                                   List of roles
 Role name |                         Attributes                         | Member of 
-----------+------------------------------------------------------------+-----------
 dssc      |                                                            | {}
 postgres  | Superuser, Create role, Create DB, Replication, Bypass RLS | {}
```

```sh
alter role dssc with createdb createrole;
```

```sh
\du
                                   List of roles
 Role name |                         Attributes                         | Member of 
-----------+------------------------------------------------------------+-----------
 dssc      | Create role, Create DB                                     | {}
 postgres  | Superuser, Create role, Create DB, Replication, Bypass RLS | {}
```

Alternatively, you could drop the DB and the role and recreated it:

```sql
drop database dssc;
drop role dssc;
create role "dssc" with password 'dssc' login createdb createrole;
create database "dssc" with owner "dssc";
```

You can test the connectivity by running

```sh
kubectl run -it --image=ubuntu ushell --restart=Never --rm -- /bin/bash

root@kshell:/ apt update ; apt install -y postgresql-client
root@kshell:/ psql -h postgresql dssc dssc
```

Password is `dssc`.

```sh
Password for user dssc:
psql (12.9 (Ubuntu 12.9-0ubuntu0.20.04.1), server 10.17)
Type "help" for help.

dssc=> \l
                                List of databases
  Name    |  Owner   | Encoding |  Collate   |   Ctype    |   Access privileges
-----------+----------+----------+------------+------------+-----------------------
dssc      | dssc     | UTF8     | en_US.utf8 | en_US.utf8 |
postgres  | postgres | UTF8     | en_US.utf8 | en_US.utf8 |
template0 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
          |          |          |            |            | postgres=CTc/postgres
template1 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
          |          |          |            |            | postgres=CTc/postgres
(4 rows)
```

### Deploy Smart Check

Create Overrides. Change the service type to LoadBalancer if you have a Load Balancer

```sh
cat <<EOF > overrides-smartcheck.yml
auth:
  secretSeed: 'just_anything-really_anything'
  userName: 'admin'
  password: 'trendmicro'
persistence:
  enabled: true
service:
  type: NodePort
db:
  host: postgresql
  port: 5432
  user: dssc
  password: dssc
  tls:
    mode: disable
EOF
```

Install Smart Check

```sh
helm install -n smartcheck --values overrides-smartcheck.yml smartcheck https://github.com/deep-security/smartcheck-helm/archive/master.tar.gz
```

> Note: As of Openshift 4.10 there is an issue with NetworkPolicies and Egress. To fix that after the `helm install` run the following:

```sh
#!/bin/bash
cat <<EOF > networkpolicy-patch.yaml
spec:
  egress:
    - {}
EOF

POLS=$(oc get networkpolicy -o json|jq -r '.items[].metadata.name')
for pol in $POLS; do kubectl patch networkpolicy $pol --patch-file networkpolicy-patch.yaml ;done;
``` 

## Result

If everything worked correctly you should have a running Smart Check deployment on your cluster.

In case you're using CodeReady Containers do the following to enable access to Smart Check:

```sh
oc create route passthrough --service proxy --hostname smartcheck.testing
```

Finally, add `smartcheck.testing` to the line with some other `.testing` names. Example:

```console
192.168.64.2 smartcheck.testing api.crc.testing console-openshift-console.apps-crc.testing default-route-openshift-image-registry.apps-crc.testing oauth-openshift.apps-crc.testing
```

Access Smart Check with your browser with <https://smartcheck.testing>.

## Support

This is an Open Source community project. Project contributors may be able to help, depending on their time and availability. Please be specific about what you're trying to do, your system, and steps to reproduce the problem.

For bug reports or feature requests, please [open an issue](../../issues). You are welcome to [contribute](#contribute).

Official support from Trend Micro is not available. Individual contributors may be Trend Micro employees, but are not official support.

## Contribute

I do accept contributions from the community. To submit changes:

1. Fork this repository.
1. Create a new feature branch.
1. Make your changes.
1. Submit a pull request with an explanation of your changes or additions.

I will review and work with you to release the code.
