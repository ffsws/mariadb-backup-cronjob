# Simple Backup Maria DB/MySQL DB

## Overview

The maria DB backup uses the `cronjob` functionality of OpenShift to start a pod in regular intervals. The pod dumps the database to a persistent volume and exits.

It uses an existing RedHat Maria DB container and overrides its command. There is no need to build a container.

The project contains multiple templates:

* mariadb-backup-template.yaml
* mariadb-backup-template-with-secret.yaml (this example provides two keys in the secret: database-user, database-password)
* mariadb-backup-template-with-icinga.yaml
* mariadb-backup-template-with-icinga-and-secret.yaml (this example provides two keys in the secret: database-user, database-password)

As the names are stating the second template has also an implementation with a monitoring support from icinga.
More about the monitoring can be found int the section [Monitoring](#Monitoring)

## How to deploy the maria DB backup pod

### Prequisits

* Log in using  
`oc login`
* Switch to the right project using  
`oc project <yourproject>`

### Create a pv for the backup

I would recommend to use the GUI for this part.

### Take a look at the parameters of the template

```bash
oc process --parameters -f mariadb-backup-template.yaml
```

#### Using Secrets for DB

Instead of passing the DB Credentials in plaintext to the cronjob it's possible to use the appropriate secrets.

```bash
...
      env:
        - name: DATABASE_USER
          valueFrom:
            secretKeyRef:
              key: [NAME_OF_THE_KEY_USED_IN_THE_SECRET e.g. user]
              name: [NAME_OF_THE_SECRET_USED_IN_PROJECT]
        - name: DATABASE_PASSWORD
          valueFrom:
            secretKeyRef:
              key: [NAME_OF_THE_KEY_USED_IN_THE_SECRET e.g. password]
              name: [NAME_OF_THE_SECRET_USED_IN_PROJECT]
...
```

**The following parameters are mandatory:**

* *DATABASE_USER* (see Note)
* *DATABASE_PASSWORD* (see Note)
* *DATABASE_SECRET* (see Note)
* DATABASE_HOST
* DATABASE_PORT
* DATABASE_NAME
* DATABASE_BACKUP_VOLUME_CLAIM

**Note:** Either provide DATABASE_USER AND DATABASE_PASSWORD or DATABASE_SECRET

### Create the cronjob (with icinga support)

```bash
oc process -f mariadb-backup-template-with-icinga.yaml DATABASE_USER=<dbuser> DATABASE_PASSWORD=<dbpassword> DATABASE_HOST=<dbhost> DATABASE_PORT=<dbport> DATABASE_NAME=<dbname> DATABASE_BACKUP_VOLUME_CLAIM=<pvc-claim-name> ICINGA_USERNAME=<icinga-user> ICINGA_PASSWORD=<icinga-password> ICINGA_SERVICE_URL=<icinga-service-url> | oc create -f -
```

### Create the cronjob (without icinga support)

```bash
oc process -f mariadb-backup-template.yaml DATABASE_USER=<dbuser> DATABASE_PASSWORD=<dbpassword> DATABASE_HOST=<dbhost> DATABASE_PORT=<dbport> DATABASE_NAME=<dbname> DATABASE_BACKUP_VOLUME_CLAIM=<pvc-claim-name> | oc create -f -
```

### Create the cronjob (with credentials & without icinga suppport)

The secret used in this template provided database name, database user and database password.

```bash
oc process -f mariadb-backup-template-with-secret.yaml DATABASE_SECRET=<secretname> DATABASE_HOST=<dbhost> DATABASE_PORT=<dbport> DATABASE_NAME=<dbname> DATABASE_BACKUP_VOLUME_CLAIM=<pvc-claim-name> | oc create -f -
```

You can also store the template in the project using and `oc process` afterwards

```bash
oc create -f mariadb-backup-template.yaml
oc process mariadb-backup-template DATABASE_USER=<dbuser> DATABASE_PASSWORD=<dbpassword> ... | oc create -f -
```

To check if the cronjob is present:

````bash
oc get cronjob
````

### Housekeeping

To disable the backup, you can simply suspend the cronjob:

`oc edit cronjob mariadb-backup`

Find the attribute `suspend` and set the value to `true`

To restore the backup you start a backup pod (e.g. in debug mode) connect to the pod and use:

````bash
oc rsh mariadb-backup-[xyz]-debug
mysql -u<db-user> -p<db-user-password> -h<host> < <path-to-backupfile> (the backupfile has to be unpacked)
````

### Monitoring

In the template `mariadb-backup-template-with-icinga.yaml` an passive incinga service is monitoring the backup. Should the bash script (responsible for the backup) throw an error at any point during the executing the notification will not be sent to icinga. The passive service checks periodically if a notification was received. If not the service will update its status. The following parameters are used for the monitoring:

* ICINGA_USERNAME
* ICINGA_PASSWORD
* ICINGA_SERVICE_URL