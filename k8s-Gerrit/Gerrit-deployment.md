# Creating and configuring an Gerrit Deployment authenticating against an LDAP database using Kubernetes

This project details steps used to set up gerrit deployment using kubernetes.


## Table of Contents
- [Creating an LDAP container](#LDAP)
- [Creating an LDAP-ADMIN container](#LDAP-ADMIN)
- [Configuring the LDAP-ADMIN container](#CONFIGURE-LDAP-ADMIN)
- [Creating a MYSQL container](#MYSQL)
- [Creating a GERRIT container](#GERRIT)

## LDAP
#### 1. create a PersistentVolumeClaim using persistent disks
    a. create a pvc manifest file
    cat > ldap-volumeclaim.yaml

    kind: PersistentVolumeClaim
    apiVersion: v1
    metadata:
      name: ldap-volumeclaim
    spec:
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 200Gi

    b. deploy the manifest
    kubectl claim -f ldap-volumeclaim.yaml

    c. check to see if the claim has been bound
    kubectl get pvc

#### 2. create a Kubernetes secret to store the password for the admin user
    a. kubectl create secret generic ldap --from-literal=password=<your-admin-password>

#### 3. create an ldap Deployment
    a. create a manifest for the deployment
    cat > ldap.yaml

    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: ldap
      labels:
        app: ldap
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: ldap
      template:
        metadata:
          labels:
            app: ldap
        spec:
          containers:
            - name: ldap
              image: docker.io/accenture/adop-ldap:latest
              ports:
                - containerPort: <your-port-number>
              volumeMounts:
                - name: ldap-persistent-storage
                  mountPath: /var/lib/ldap
              env:
                - name: SLAPD_DOMAIN
                  value: btech.net
                - name: SLAPD_PASSWORD
                  valueFrom:
                    secretKeyRef:
                      name: ldap
                      key: password
                - name: SLAPD_FULL_DOMAIN
                  value: dc=btech,dc=net
                - name: INITIAL_ADMIN_USER
                  value: user
                - name: INITIAL_ADMIN_PASSWORD
                  valueFrom:
                    secretKeyRef:
                      name: ldap
                      key: password
          volumes:
            - name: ldap-persistent-storage
              persistentVolumeClaim:
                claimName: ldap-volumeclaim
    b. deploy the manifest
    kubectl create -f ldap.yaml

    c. check its health, it might take a couple of minutes
    kubectl get pod -l app=ldap

#### 4. create a Service to expose the ldap container and make it accessible from the ldap-admin container
    a. cat > ldap-service.yaml

    apiVersion: v1
    kind: Service
    metadata:
      name: ldap
      labels:
        app: ldap
    spec:
      type: ClusterIP
      ports:
        - port: <your-port-number>
          targetPort: 389
      selector:
        app: ldap

    b. deploy the manifest and launch the service
    kubectl create -f ldap-service.yaml

    c. check the health of the created service
    kubectl get service ldap
    
#### 5. view more details about the deployment
    a. kubectl describe deploy -l app=ldap

#### 6. view more details about the pod
    a. kubectl describe po -l app=ldap

## LDAP-ADMIN
#### 1. create an ldap-admin Deployment
    a. create a manifest for the deployment
    cat > ldap-admin.yaml

    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: ldap-admin
      labels:
        app: ldap-admin
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: ldap-admin
      template:
        metadata:
          labels:
            app: ldap-admin
        spec:
          containers:
            - name: ldap-admin
              image: docker.io/dinkel/phpldapadmin:latest
              ports:
                - containerPort: <your-port-number>
              volumeMounts:
                - name: ldap-admin-persistent-storage
                  mountPath: /var/lib/ldap-admin
              env:
		- name: LDAP_SERVER_HOST
		  value: ldap:389
		  
    b. deploy the manifest
    kubectl create -f ldap-admin.yaml

    c. check its health, it might take a couple of minutes
    kubectl get pod -l app=ldap-admin

#### 4. create a Service to expose the ldap-admin container and make it accessible from the public
    a. cat > ldap-admin-service.yaml

    apiVersion: v1
    kind: Service
    metadata:
      name: ldap-admin
      labels:
        app: ldap-admin
    spec:
      type: LoadBalancer
      ports:
        - port: <your-port-number>
          targetPort: 80
          protocol: TCP
      selector:
        app: ldap-admin

    b. deploy the manifest and launch the service
    kubectl create -f ldap-admin-service.yaml

    c. check the health of the created service
    kubectl get service ldap-admin
    
#### 5. view more details about the deployment
    a. kubectl describe deploy -l app=ldap-admin

#### 6. view more details about the pod
    a. kubectl describe po -l app=ldap-admin

#### 7. visit the app in the browser using the EXTERNAL-IP value obtained from Service details

## CONFIGURE-LDAP-ADMIN
#### 1. create a posix group (e.g. gerrit-users) so as to be able to create user accounts
    a. click "ou=groups"
    b. click "create a child entry"
    c. choose "Generic: Posix Group"
    d. click "Create Object" button to proceed
    e. on next screen click "Commit" button to persist the changes

#### 2. create users for use on gerrit
    a. click "ou=people"
    b. click "create a child entry"
    c. choose "Generic: User Account"
    d. fill in "gid", "lastname", "login shell", "password"
    e. click "Create Object" button to proceed
    f. on next screen click "Commit" to persist the changes
    g. click "Add new attribute" to add "Email" and "displayName" attributes
#### N.B. the first login on gerrit becomes the admin user so choose a "cn" wisely

## MYSQL
#### 1. create a PersistentVolumeClaim using persistent disks
    a. create a pvc manifest file
    cat > mysql-volumeclaim.yaml

    kind: PersistentVolumeClaim
    apiVersion: v1
    metadata:
      name: mysql-volumeclaim
    spec:
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 200Gi

    b. deploy the manifest
    kubectl claim -f mysql-volumeclaim.yaml

    c. check to see if the claim has been bound
    kubectl get pvc

#### 2. create a mysql Deployment
    a. create a manifest for the deployment
    cat > mysql.yaml

    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: mysql
      labels:
        app: mysql
        role: database
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: mysql
      template:
        metadata:
          labels:
            app: mysql
            role: database
        spec:
          containers:
            - image: docker.io/mysql/mysql-server:5.7
                  name: mysql
                  ports:
                    - containerPort: <your-port-number>
                      name: mysql
          volumes:
            - name: mysql-persistent-storage
              persistentVolumeClaim:
                claimName: mysql-volumeclaim

    b. deploy the manifest
    kubectl create -f mysql.yaml

    c. check its health, it might take a couple of minutes
    kubectl get pod -l app=mysql

#### 3. create a Service to expose the mysql container and make it accessible from the other containers
    a. cat > mysql-service.yaml

    apiVersion: v1
    kind: Service
    metadata:
      name: mysql
      labels:
        app: mysql
    spec:
      type: ClusterIP
      ports:
        - port: <your-port-number>
          targetPort: 3306
      selector:
        app: mysql

    b. deploy the manifest and launch the service
    kubectl create -f mysql-service.yaml

    c. check the health of the created service
    kubectl get service mysql
    
#### 4. view more details about the deployment
    a. kubectl describe deploy -l app=mysql

#### 5. view more details about the pod
    a. kubectl describe po -l app=mysql

#### 6. get its default password
    a. kubectl logs -l app=mysql 2>&1 | grep GENERATED

#### 7. enter container and change default password before you can start using it
    a. kubectl exec -it MYSQL5.7 bash
    b. mysql -u root -p<GENERATED PASSWORD>
    c. ALTER USER 'root'@'localhost' IDENTIFIED BY 'secret';
    d. create USER 'root'@'%' IDENTIFIED BY 'secret';

#### 8. create a database for gerrit
    a. create database gerritdb;

#### 9. create a Kubernetes secret to store the password for the new root user
    a. kubectl create secret generic mysql --from-literal=password=<your-root-password>

## GERRIT
#### 1. create a PersistentVolumeClaim using persistent disks
    a. create a pvc manifest file
    cat > gerrit-volumeclaim.yaml

    kind: PersistentVolumeClaim
    apiVersion: v1
    metadata:
      name: gerrit-volumeclaim
    spec:
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 200Gi

    b. deploy the manifest
    kubectl claim -f gerrit-volumeclaim.yaml

    c. check to see if the claim has been bound
    kubectl get pvc


#### 2. create a gerrit Deployment
    a. create a manifest for the deployment
    cat > gerrit.yaml

    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: gerrit
      labels:
        app: gerrit
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: gerrit
      template:
        metadata:
          labels:
            app: gerrit
        spec:
          containers:
            - name: gerrit
              image: docker.io/openfrontier/gerrit
              ports:
                - containerPort: <your-port-number>
              env:
                - name: GITWEB_TYPE
                  value: gitiles
                - name: WEBURL
                  value: http://<web-server-address>:<your-port-number>
                - name: DATABASE_TYPE
                  value: mysql
                - name: DATABASE_HOSTNAME
                  value: mysql
                - name: DATABASE_PORT
                  value: <your-mysql-port-number>
                - name: DATABASE_DATABASE
                  value: gerritdb
                - name: DATABASE_USERNAME
                  value: root
                - name: DATABASE_PASSWORD
                  valueFrom:
                    secretKeyRef:
                      name: mysql
                      key: password
                - name: AUTH_TYPE
                  value: LDAP
                - name: LDAP_SERVER
                  value: ldap://ldap:<your-ldap-port-number>
                - name: LDAP_ACCOUNTBASE
                  value: ou=people,dc=btech,dc=net
                - name: SMTP_SERVER
                  value: <your-smtp-server-ip>
                - name: SMTP_SERVER_PORT
                  value: <your-smtp-server-port>
                - name: SMTP_ENCRYPTION
                  value: <smpt-encryption>
                - name: SMTP_USER
                  value: <smtp-username>
                - name: SMTP_PASS
                  value: <smtp-password>
                - name: SMTP_CONNECT_TIMEOUT
                  value: 10sec
                - name: SMTP_FROM
                  value: <smtp-from>
                - name: USER_NAME
                  value: <gerrit-username>
                - name: USER_EMAIL
                  value: <gerrit-username-email>
              volumeMounts:
                - name: gerrit-persistent-storage
                  mountPath: /var/gerrit/review_site
          volumes:
            - name: gerrit-persistent-storage
              persistentVolumeClaim:
                claimName: gerrit-volumeclaim
		  
    b. deploy the manifest
    kubectl create -f gerrit.yaml

    c. check its health, it might take a couple of minutes
    kubectl get pod -l app=gerrit

#### 3. create a Service to expose the gerrit container and make it accessible from the public
    a. cat > gerrit-service.yaml

    apiVersion: v1
    kind: Service
    metadata:
      name: gerrit
      labels:
        app: gerrit
    spec:
      type: LoadBalancer
      ports:
        - port: <your-port-number>
          targetPort: 8080
          protocol: TCP
      selector:
        app: gerrit

    b. deploy the manifest and launch the service
    kubectl create -f gerrit-service.yaml

    c. check the health of the created service
    kubectl get service gerrit
    
#### 4. view more details about the deployment
    a. kubectl describe deploy -l app=gerrit

#### 5. view more details about the pod
    a. kubectl describe po -l app=gerrit

#### 6. visit the app in the browser using the EXTERNAL-IP value obtained from Service details