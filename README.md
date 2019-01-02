# This repo is for how to deploy Red Hat Quay enterperise v3 on OpenShift

## Prepare related component for Quay

### 1. Create project on openshift cluster

    $ oc new-project quay-enterprise  \\ hard code

### 2. Create a user group to manage above project

    $ oc adm groups new quay <user>
    $ oc policy add-role-to-group admin quay -n quay-enterprise

### 3. Prepare related redis and database via default template

    $ oc new-app redis-ephemeral -p DATABASE_SERVICE_NAME=quay-redis -p REDIS_PASSWORD=quayredis -n quay-enterprise
    $ oc new-app postgresql-persistent -p DATABASE_SERVICE_NAME=quay-db -p POSTGRESQL_USER=quayuser -p POSTGRESQL_PASSWORD=quaypass -p POSTGRESQL_DATABASE=quaydb -n quay-enterprise
    $  oc rsh postgresql-1-x2s4m
    sh-4.2$ /bin/bash -c 'echo "SELECT * FROM pg_available_extensions" | /opt/rh/rh-postgresql96/root/usr/bin/psql'
    sh-4.2$ /bin/bash -c 'echo "CREATE EXTENSION pg_trgm" | /opt/rh/rh-postgresql96/root/usr/bin/psql'
    sh-4.2$ /bin/bash -c 'echo "SELECT * FROM pg_extension" | /opt/rh/rh-postgresql96/root/usr/bin/psql'
    sh-4.2$ /bin/bash -c 'echo "ALTER USER quayuser WITH SUPERUSER;" | /opt/rh/rh-postgresql96/root/usr/bin/psql'

### 4. Import quay image into imagestream

    $ docker login quay.io
    $ oc create secret generic quay --from-file=.dockerconfigjson=/root/.docker/config.json --type='kubernetes.io/dockerconfigjson' -n quay-enterprise
    $ oc secrets link default quay --for=pull -n quay-enterprise
    $ oc secrets link deployer quay -n quay-enterprise
    $ oc import-image quay --from=quay.io/quay/quay:2.9.4-release --confirm -n quay-enterprise

### 5. Create cluster role and rolebinding

    $ oc create -f quay-role.yaml --config=<admin.kubeconfig>
    $ oc create -f quay-role-binding.yaml --config=<admin.kubeconfig>

####   Add privilege: Make sure that the service account has root privileges, because Quay runs strictly under root (this will be changed in the future versions):

    $ oc adm policy add-scc-to-user anyuid system:serviceaccount:quay-enterprise:default

### 6. Use quay config tool to initialize database (Optional)

    $ oc create -f quay-config-tool-svc-nodeport.yaml
    $ oc create -f quay-config-tool-dc.yaml

### 7. Create quay service and expose it via route

    $ oc create -f quay-service-clusterip.yaml
    $ oc create route passthrough --service=quay-enterprise --port=https -n quay-enterprise

### 8. Create a key and server certificate valid for specified IPs and host names, signed by a specified CA.

    $ oc get svc
    $ oc get route
    $ oc adm ca create-server-cert \
    --signer-cert=/etc/origin/master/ca.crt \
    --signer-key=/etc/origin/master/ca.key \
    --signer-serial=/etc/origin/master/ca.serial.txt \
    --hostnames='<route host name>,<service ip>' \
    --cert=ssl.cert \
    --key=ssl.key

### 9. Deploy quay

    $ oc create secret generic quay-enterprise-config-secret --from-file=config.yaml -n quay-enterprise \\ hard code
    $ oc create secret generic quay-enterprise-cert-secret --from-file=ssl.cert --from-file=ssl.key -n quay-enterprise

    P.S. against v3, need to create two cert and key named 'ssl.old.cert' and 'ssl.old.key'

    $ oc create -f quay-app.yaml





## Set up clair

    $ oc create -f clair-service.yaml
    $ oc create secret generic clair-cert-secret --from-file=/etc/origin/master/ca.crt -n quay-enterprise \\ ca.crt is for quay self-signed certificate, located in openshift master node
    
###    create a key for clair interact with quay, refer to https://coreos.com/quay-enterprise/docs/latest/security-scanning.html

    $ oc create secret generic clair-security-scanner --from-file=security_scanner.pem -n quay-enterprise

###    create new database for clair in Postgresl database

    $ oc rsh quay-db-1-krrmw
    sh-4.2$ /bin/bash -c 'echo "CREATE DATABASE clairdb" | /opt/rh/rh-postgresql96/root/usr/bin/psql'

###    create clair deploymentConfig

    $ oc import-image clair --from=quay.io/coreos/clair-jwt:v2.0.7 --confirm -n quay-enterprise
    $ oc create secret generic clair-config-secret --from-file=clair/config.yaml -n quay-enterprise
    $ oc create -f clair-app-dc.yaml


### P.S.
#### initial quay with secret or configMap that includes existing valid config file.
1. the secret name cannot be 'quay-enterprise-config-secret', or will prompt 500 internal error
2. if secret or configMap is null data, then will prompt 500 internal error
3. except 'quay-enterprise-config-secret' secret, UI save configuration cannot re-write to secret or configMap
4. base64 test-config.yaml -w 0
