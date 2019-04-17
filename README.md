# This repo is for testing Red Hat Quay on OpenShift

## Prepare databases for Quay

Recommand rpm database cluster for production, we can use container for testing. Deploy database by default template in OpenShift. Quay supports two kinds of database: mysql and postgresql.

    $ oc new-project quay-db
    $ oc new-app mysql-persistent -p DATABASE_SERVICE_NAME=quay-db -p MYSQL_USER=quayuser -p MYSQL_PASSWORD=quaypass -p MYSQL_DATABASE=quaydb -p MEMORY_LIMIT=2Gi -n quay-db

OR

    $ oc new-app postgresql-persistent -p DATABASE_SERVICE_NAME=quay-db -p POSTGRESQL_USER=quayuser -p POSTGRESQL_PASSWORD=quaypass -p POSTGRESQL_DATABASE=quaydb -p MEMORY_LIMIT=2Gi -n quay-db

Add "pg_trgm" extension as superuser on this database.

    $  oc rsh <postgresql-pod>
    sh-4.2$ /bin/bash -c 'echo "ALTER USER quayuser WITH SUPERUSER;" | /opt/rh/rh-postgresql<-version:96>/root/usr/bin/psql'
    sh-4.2$ psql -U quayuser -W quaydb
    quaydb=# CREATE EXTENSION IF NOT EXISTS pg_trgm;
    CREATE EXTENSION
    quaydb=# \dx
                                        List of installed extensions
      Name   | Version |   Schema   |                            Description                            
    ---------+---------+------------+-------------------------------------------------------------------
     pg_trgm | 1.3     | public     | text similarity measurement and index searching based on trigrams
     plpgsql | 1.0     | pg_catalog | PL/pgSQL procedural language
    (2 rows)

## Setup Quay

### 1. Create project on openshift cluster

    $ oc new-project quay-enterprise  \\ project name is hard code

### 2. Create a user group to manage above project

    $ oc adm groups new quay <user>
    $ oc policy add-role-to-group admin quay -n quay-enterprise

### 3. Create cluster role and rolebinding by cluster admin

    $ oc create -f quay-role.yaml --config=<admin.kubeconfig>
    $ oc create -f quay-role-binding.yaml --config=<admin.kubeconfig>

Add privilege: Make sure that the service account has root privileges, because Quay runs strictly under root (this will be changed in the future versions):

    $ oc adm policy add-scc-to-user anyuid system:serviceaccount:quay-enterprise:default

### 4. Prepare redis via default template

    $ oc new-app redis-persistent -p DATABASE_SERVICE_NAME=quay-redis -p REDIS_PASSWORD=quayredis -n quay-enterprise

### 5. Import quay image into imagestream

    $ docker login quay.io
    $ oc create secret generic quay --from-file=.dockerconfigjson=/root/.docker/config.json --type='kubernetes.io/dockerconfigjson' -n quay-enterprise
    $ oc secrets link default quay --for=pull -n quay-enterprise
    $ oc secrets link deployer quay -n quay-enterprise
    $ oc import-image quay:2 --from=quay.io/coreos/quay:2.9.4-release --confirm -n quay-enterprise
    $ oc tag quay:2 quay:latest -n quay-enterprise

### 6. Create quay service and expose it via route

Also can use enternal load balancer

    $ oc create -f quay-service-clusterip.yaml

Against 2.x, modify targetPort 8080 to 80, 8443 to 443

    $ oc create route passthrough --service=quay-enterprise --port=https -n quay-enterprise

### 7. Create a key and server certificate valid for specified IPs and host names, signed by a specified CA.

#### First, create a root CA:

    $ openssl genrsa -out rootCA.key 2048
    $ openssl req -x509 -new -nodes -key rootCA.key -sha256 -days 1024 -out rootCA.pem

#### Next, create an openssl.cnf file. Replacing DNS.1 and IP.1 with the hostname and IP of the Quay Enterprise server:

openssl.cnf

    [req]
    req_extensions = v3_req
    distinguished_name = req_distinguished_name
    [req_distinguished_name]
    [ v3_req ]
    basicConstraints = CA:FALSE
    keyUsage = nonRepudiation, digitalSignature, keyEncipherment
    subjectAltName = @alt_names
    [alt_names]
    DNS.1 = reg.example.com
    IP.1 = 12.345.678.9

#### The following set of shell commands invoke the openssl utility to create a key for Quay Enterprise, generate a request for an Authority to sign a new certificate, and finally generate a certificate for Quay Enterprise, signed by the CA created earlier.

Make sure the CA certificate file rootCA.pem and the openssl.cnf config file are both available.

    $ openssl genrsa -out ssl.key 2048
    $ openssl req -new -key ssl.key -out ssl.csr -subj "/CN=quay-enterprise" -config openssl.cnf
    $ openssl x509 -req -in ssl.csr -CA rootCA.pem -CAkey rootCA.key -CAcreateserial -out ssl.cert -days 356 -extensions v3_req -extfile openssl.cnf

#### Create a secret from new certificate
    
Could skip this step if ssl certificate by external load balancer or edge route

    $ oc create secret generic quay-enterprise-cert-secret --from-file=ssl.cert --from-file=ssl.key -n quay-enterprise

### 8. Use quay config tool to initialize database and quay configurations

Skip this step against 2.x

    $ oc create -f quay-config-tool-svc-clusterip.yaml
    $ oc create -f quay-config-tool-dc.yaml
    $ oc rollout latest quay-config-tool
    $ oc create route passthrough --service=quay-config-tool --port=web -n quay-enterprise

Download quay-config.tar.gz

### 9. Deploy quay deployment

    $ oc create secret generic quay-enterprise-config-secret --from-file=config.yaml -n quay-enterprise          \\ the secret name is hard code
    $ oc create -f quay-app.yaml

Against 2.x, modify containerPort 8080 to 80, 8443 to 443
    
    $ oc rollout latest quay-enterprise-app

## Set up clair

    $ oc create -f clair-service.yaml
    $ oc create secret generic clair-cert-secret --from-file=rootCA.pem -n quay-enterprise    \\ rootCA.pem is for quay self-signed certificate
    
###    create a key for clair interact with quay, refer to https://coreos.com/quay-enterprise/docs/latest/security-scanning.html

    $ oc create secret generic clair-security-scanner --from-file=security_scanner.pem -n quay-enterprise

###    create new database for clair in Postgresl database

    $ oc rsh quay-db-1-krrmw
    sh-4.2$ /bin/bash -c 'echo "CREATE DATABASE clairdb" | /opt/rh/rh-postgresql96/root/usr/bin/psql'

###    create clair deploymentConfig

    $ oc import-image clair --from=quay.io/coreos/clair-jwt:v2.0.8 --confirm -n quay-enterprise
    $ oc create secret generic clair-config-secret --from-file=clair/config.yaml -n quay-enterprise
    $ oc create -f clair-app-dc.yaml


### P.S.
#### initial quay with secret or configMap that includes existing valid config file.
1. except 'quay-enterprise-config-secret' secret, UI save configuration cannot re-write to secret or configMap
2. base64 config.yaml -w 0
