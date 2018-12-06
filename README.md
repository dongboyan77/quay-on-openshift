# This repo is for how to deploy quay enterperise on openshift

## 1.Create project on openshift cluster

    $ oc new-project quay-enterprise  \\ hard code

### create a user group to manage above project

    $ oc adm groups new quay <user>
    $ oc policy add-role-to-group admin quay -n quay-enterprise

## 2.Prepare related redis and database via default template

    $ oc new-app redis-ephemeral -p DATABASE_SERVICE_NAME=quay-redis -p REDIS_PASSWORD=quayredis
    $ oc new-app postgresql-persistent -p DATABASE_SERVICE_NAME=quay-db -p POSTGRESQL_USER=quayuser -p POSTGRESQL_PASSWORD=quaypass -p POSTGRESQL_DATABASE=quaydb
    $  oc rsh postgresql-1-x2s4m
    sh-4.2$ /bin/bash -c 'echo "SELECT * FROM pg_available_extensions" | /opt/rh/rh-postgresql96/root/usr/bin/psql'
    sh-4.2$ /bin/bash -c 'echo "CREATE EXTENSION pg_trgm" | /opt/rh/rh-postgresql96/root/usr/bin/psql'
    sh-4.2$ /bin/bash -c 'echo "SELECT * FROM pg_extension" | /opt/rh/rh-postgresql96/root/usr/bin/psql'
    sh-4.2$ /bin/bash -c 'echo "ALTER USER quayuser WITH SUPERUSER;" | /opt/rh/rh-postgresql96/root/usr/bin/psql'

## 3.Import quay image into imagestream

    $ docker login quay.io
    $ oc create secret generic quay --from-file=.dockerconfigjson=/root/.docker/config.json --type='kubernetes.io/dockerconfigjson'
    $ oc secrets link default quay --for=pull
    $ oc secrets link deployer quay
    $ oc import-image quay --from=quay.io/quay/quay:2.9.4-release --confirm

## 4.Create cluster role and rolebinding

    $ oc create -f quay-role.yaml --config=<admin.kubeconfig>
    $ oc create -f quay-role-binding.yaml --config=<admin.kubeconfig>

### Add privilege: Make sure that the service account has root privileges, because Quay runs strictly under root (this will be changed in the future versions):

    $ oc adm policy add-scc-to-user anyuid system:serviceaccount:quay-enterprise:default

## 5.Deploy quay

    $ oc create -f quay-config-secret.yaml
    $ oc create secret generic quay-enterprise-config-secret --from-file=config.yaml \\ hard code
    $ oc create -f quay-service-nodeport.yaml
    $ oc create -f quay-app.yaml

## P.S.
### initial quay with secret or configMap that includes existing valid config file.
1. the secret name cannot be 'quay-enterprise-config-secret', or will prompt 500 internal error
2. if secret or configMap is null data, then will prompt 500 internal error
3. except 'quay-enterprise-config-secret' secret, UI save configuration cannot re-write to secret or configMap
4. base64 test-config.yaml -w 0
