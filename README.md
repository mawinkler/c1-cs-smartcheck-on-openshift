# Deploying Trend Micro Smart Check on OpenShift

- [Deploying Trend Micro Smart Check on OpenShift](#deploying-trend-micro-smart-check-on-openshift)
  - [Getting Started](#getting-started)
  - [Result](#result)
  - [Support](#support)
  - [Contribute](#contribute)

Get started running Smart Check on OpenShift.

## Getting Started

1. Create Overrides. Change the service type to LoadBalancer if you have a Load Balancer

    ```sh
    cat <<EOF > overrides.yml
    activationCode: 'HULA'
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

2. Create a project for Smart Check. For simplicity, we're going to deploy the database into the same project later on.

    ```sh
    oc new-project smartcheck
    ```

3. Create new Security Context Constraints for Smart Check. The defined one is a copy from the default restricted with a changed `priority` and `runAsUser` value. Assign the scc to the default service user within the project.

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
      type: MustRunAs
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
      type: MustRunAs
    seLinuxContext:
      type: MustRunAs
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

4. Next, setup a PostgreSQL

    ```sh
    oc process -n openshift postgresql-persistent -o yaml \
      -p POSTGRESQL_USER=dssc \
      -p POSTGRESQL_PASSWORD=dssc \
      -p POSTGRESQL_DATABASE=dssc \
      | oc create -f -
    ```

5. The created user within the postgresql does not have the required permissions `create database` and `create role` which we need to grant to the user dssc. Connect to the database with

    ```sh
    oc rsh $(oc get pods -o json | jq -r '.items[].metadata | select(.name | startswith("postgres")) | .name' | grep -v deploy)
    ```

    Within the new shell...

    ```sh
    psql
    ```

    ```sh
    alter role dssc with createdb createrole;
    ```

6. Install Smart Check

    ```sh
    helm install -n smartcheck --values overrides.yml smartcheck https://github.com/deep-security/smartcheck-helm/archive/master.tar.gz
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

Access Smart Check with your broswer with <https://smartcheck.testing>.

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
