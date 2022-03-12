# pg-logical-replication-demo <!-- omit in toc -->

- [prerequisites](#prerequisites)
- [setup](#setup)
  - [namespace](#namespace)
  - [postgresql](#postgresql)
    - [source](#source)
    - [destination](#destination)
    - [source](#source-1)
    - [destination](#destination-1)
- [cleanup](#cleanup)
- [references](#references)
- [TODO](#todo)

Logical replication is a flexible technique for replicating data between two postgresql databases.

In this repo, we are using the [Kubernetes](https://kubernetes.io/) to deploy the Postgresql instances.

## prerequisites
- [Rancher Desktop](https://github.com/rancher-sandbox/rancher-desktop): `1.1.1`
- Kubernetes: `v1.22.6`
- kubectl `v1.23.3`
- Helm: `v3.7.2`

## setup

### namespace

```sh
kubectl create namespace demo --dry-run=client -o yaml | kubectl apply -f -
```

### postgresql

follow the [bitnami postgresql chart](https://github.com/bitnami/charts/tree/master/bitnami/postgresql) to install postgresql

```sh
helm repo add bitnami https://charts.bitnami.com/bitnami
```

#### source

```sh
helm upgrade --install source-postgresql bitnami/postgresql -n demo -f postgresql/values.yaml
```

login to the postgresql

```sh
kubectl run source-postgresql-client --rm --tty -i --restart='Never' --namespace demo --image docker.io/bitnami/postgresql:14.2.0-debian-10-r25 --env="PGPASSWORD=demo_password" -- psql --host source-postgresql
```

check the wal_level to be 'logical'

```sh
SHOW wal_level;
```

```sh
 wal_level 
-----------
 logical
(1 row)
```

insert some data

```sh
INSERT INTO test_table VALUES (1), (2);
```

create a publication for test_table

```sh
CREATE PUBLICATION pub1 FOR TABLE test_table;
```

#### destination

```sh
helm upgrade --install destination-postgresql bitnami/postgresql -n demo -f postgresql/values.yaml
```

login to the postgresql

```sh
kubectl run destination-postgresql-client --rm --tty -i --restart='Never' --namespace demo --image docker.io/bitnami/postgresql:14.2.0-debian-10-r25 --env="PGPASSWORD=demo_password" -- psql --host destination-postgresql
```

check the wal_level to be 'logical'

```sh
SHOW wal_level;
```

```sh
 wal_level 
-----------
 logical
(1 row)
```

validate there are no rows in the table

```sh
SELECT * FROM test_table;
```

```sh
 id 
----
(0 rows)
```

create a subscription for pub1

```sh
CREATE SUBSCRIPTION sub1 CONNECTION 'host=source-postgresql port=5432 password=demo_password' PUBLICATION pub1;
```

validate the replication is working

```sh
SELECT * FROM test_table;
```

```sh
 id 
----
  1
  2
(2 rows)
```

#### source

check the status of a replication from pg_stat_subscription and pg_replication_slots

```sh
SELECT 
    confirmed_flush_lsn, 
    pg_current_wal_lsn(), 
    (pg_current_wal_lsn() - confirmed_flush_lsn) AS lsn_distance 
FROM pg_catalog.pg_replication_slots
WHERE slot_name = 'sub1';
```

```sh
 confirmed_flush_lsn | pg_current_wal_lsn | lsn_distance 
---------------------+--------------------+--------------
 0/170F910           | 0/170F910          |            0
```

#### destination

clean up unused replication slots

```sh
DROP SUBSCRIPTION sub1;
```

## cleanup

```sh
helm uninstall source-postgresql -n demo
helm uninstall destination-postgresql -n demo
kubectl delete pvc --all -n demo
kubectl delete namespace demo
```

## references
- [Making Sense of PostgreSQL Logical Replication.pdf](https://github.com/aiven/presentations/blob/master/Cloud_chats_by_Aiven/Making%20Sense%20of%20PostgreSQL%20Logical%20Replication.pdf)

## TODO
- Sniff the packets to verify the replication is working using [ksniff](https://github.com/eldadru/ksniff)

