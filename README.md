# Kubernetes Express

This is a mini Kubernetes project.
Create a Webapp with a database using Mongo-Express and MongoDB

## Overview

    │─────────│    │───────────│    │─────────│      │───────────│      │─────────│
    │         │    │  Mongo    │    │   Pod:  │      │  MongoDB  │      │   Pod:  │
    │ Browser │ => │  Express  │ => │  Mongo  │  =>  │  Internal │  =>  │ MongoDB │
    │         │    │  External │    │ Express │      │  Service  │      └─────────
    └─────────     │  Service  │    └─────────       └───────────            **
                   └───────────         **                              - mongo.yaml*
                                   - deployment: express.yaml           - secret.yaml
                                   - configMap.yaml                       - DB User
                                     - DB Url                             - DB Pwd

## Steps

1.  Assumption:

    - Minikube cluster is already running:
    - kubectl get all

2.  Create a root project folder for the project

    - K8sExpress

3.  Pod - MongoDB Creation:

    a. Create Secret: mongo-secret.yaml

            apiVersion: v1
            kind: Secret
            metadata:
                name: mongodb-secret
            type: Opaque
            data:
                mongo-root-username:       // value must be base64 encoded
                mongo-root-password:       // value must be base64 encoded

    b. Create Base64 encoded value for username and password:

        - open terminal
        - echo -n 'username' | base64
        - echo -n 'password' | base64
        - Copy returned values to the Secret file

    c. Create Secret File:

        - Open terminal
        - Change dir into the Project folder

            kubectl apply -f mongo-secret.yaml

        - Confirm Secret creation

            kubectl get secret

    d. Create config-file for MongoDB creation: mongo.yaml

            apiVersion: apps/v1
            kind: Deployment
            metadata:
                name: mongodb-deployment
                labels:
                    app: mongodb
            spec:
                replicas: 1
                selectors:
                    matchLabels:
                        app: mongodb
                template:
                    metadata:
                        labels:
                            app: mongodb
                    spec:
                        containers:
                        - name: mongodb
                          image: mongo            // check for image spec on hub.docker.com
                          ports:
                          - containerPort: 27017  // default ports
                          env:
                          - name: MONGO_INITDB_ROOT_USERNAME
                            valueFrom:                  // DB User from Secret will come here
                              secretKeyRef:
                                name: mongodb-secret
                                key: mongo-root-username
                            - name: MONGO_INITDB_ROOT_PASSWORD
                            valueFrom:                  // DB Pwd from Secret will come here
                              secretKeyRef:
                                name: mongodb-secret
                                key: mongo-root-password

    e. Apply the Deployment and Verify Deployment Creation:

            kubectl apply -f mongo.yaml

            kubectl get all
            kubectl get pod
            kubectl describe pod [podname]
            kubectl get pod --watch

4.  Create MongoDB Internal Service

    a. Update the mongo.yaml file with the following:

            ---
            apiVersion: v1
            kind: Service
            metadata:
              name: mongodb-service
            spec:
              selector:
                app: mongodb
              ports:
                - protocol: TCP
                  port: 27017
                  targetPort: 27017

    b. Apply the deployment and Verify deployment creation

            kubectl apply -f mongo.yaml

            kubectl get service
            kubectl describe service [serviceName]

            kubectl get all | grep mongodb

5.  Pod - Mongo Express Deployment Service

    a. Create the ConfigMap that would contain the MongoDB server address: mongo-configmap.yaml

            apiVersion: apps/v1
            kind: ConfigMap
            metadata:
              name: mongodb-configmap
            data:
              database_url: mongodb-service

    b. Create the deploment service file: mongo-express.yaml

            apiVersion: apps/v1
            kind: Deployment
            metadata:
              name: mongodb-express
              labels:
                app: mongo-express
            spec:
              replicas: 1
              selectors:
                matchLabels:
                  app: mongo-express
              template:
                metadata:
                  labels:
                    app: mongo-express
                spec:
                containers:
                    - name: mongo-express
                      image: mongo-express
                      ports:
                      - containerPort: 8081
                      env:
                      - name: ME_CONFIG_MONGODB_ADMINUSERNAME
                        valueFrom:
                          secretKeyRef:
                            name: mongodb-secret
                            key: mongo-root-username
                      - name: ME_CONFIG_MONGODB_ADMINPASSWORD
                        valueFrom:
                          secretKeyRef:
                            name: mongodb-secret
                            key: mongo-root-password
                      - name: ME_CONFIG_MONGODB_SERVER
                        valueFrom:
                            configMapKeyRef:
                            name: mongodb-configmap
                            key: database_url

    c. Create the ConfigMap and Mongo Express deployment, then verify them

            kubectl apply -f mongo-configmap.yaml
            kubectl apply -f mongo-express.yaml

            kubectl get pod
            kubectl logs [podname]

6.  Create MongoExpress External Service

    - Final step: Create external service to help access Mongo-Express from the browser

    a. Add the following lines to mongo-express.yaml

            ---
            apiVersion: v1
            kind: Service
            metadata:
              name: mongo-express-service
            spec:
              selector:
                app: mongo-express
            type: LoadBalancer
            ports:
                - protocol: TCP
                  port: 8081
                  targetPort: 8081
                  nodePort: 30000

    b. Apply the External Service:

            kubectl apply -f mongo-express.yaml
            kubectl get service

            minikube service [serviceName]  // Assigns the External Service a public IP address
                                            // ...and loads the application in a browser

    c. End
