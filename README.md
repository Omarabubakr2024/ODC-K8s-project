## Project Overview

This project involves setting up a three-tier architecture (Proxy, Backend, Database) on Kubernetes. Each tier is deployed as a Kubernetes Deployment with two replicas for redundancy. All resources are organized under the `webapp` namespace. The project also integrates Persistent Volume (PV) and Persistent Volume Claim (PVC) for data persistence and a secret for database credentials.

### Components

1. **Namespace (`ns.yaml`)**:
   - A separate namespace, `webapp`, is used to isolate all resources in the project.

    ```yaml
    apiVersion: v1
    kind: Namespace
    metadata:
      name: webapp
    ```

2. **Backend Deployment (`backend-deployment.yaml`)**:
   - This deployment runs two replicas of the backend application.
   - Secret for database credentials (`db-secret`) is mounted as a volume.
   - An init container pre-configures secrets before the backend container starts.
   
    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: backend-deployment
      namespace: webapp
      labels:
        app: backend-deployment
    spec:
      replicas: 2
      selector:
        matchLabels:
          app: backend
      template:
        metadata:
          labels:
            app: backend
        spec:
          volumes:
          - name: db-password
            secret:
              secretName: db-secret
          - name: pre-install
            emptyDir: {}
          initContainers:
          - name: pre-install
            image: busybox
            command: ['sh', '-c', 'cat /db-secret/db-password >> /run/secrets/db-password']
            volumeMounts:
            - name: db-password
              mountPath: "/db-secret"
            - name: pre-install
              mountPath: "/run/secrets"
          containers:
          - name: backend
            image: omarbanna/backend10:latest
            imagePullPolicy: Never
            ports:
            - containerPort: 8000
            volumeMounts:
            - name: pre-install
              mountPath: "/run/secrets"
            - name: db-password
              mountPath: "/db-secret"
    ```

3. **Backend Service (`backend-service.yaml`)**:
   - A `ClusterIP` service for the backend to allow internal communication between other pods.

    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: backendservice
      namespace: webapp
    spec:
      selector:
        app: backend
      ports:
      - protocol: TCP
        port: 8000
        targetPort: 8000
    ```

4. **Database Deployment (`db-deployment.yaml`)**:
   - Runs a MySQL database with one replica.
   - The database password is sourced from the secret `db-secret`.
   - Persistent storage is managed using a PVC (`my-pvc`).

    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: database-deployment
      namespace: webapp
      labels:
        app: database-deployment
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: database
      template:
        metadata:
          labels:
            app: database
        spec:
          volumes:
          - name: db-password
            secret:
              secretName: db-secret
          - name: db-data
            persistentVolumeClaim:
              claimName: my-pvc
          containers:
          - name: database
            image: mysql:8.0
            env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: db-password
            - name: MYSQL_DATABASE
              value: "example"
            ports:
            - containerPort: 3306
            volumeMounts:
            - name: db-data
              mountPath: "/var/lib/mysql"
    ```

5. **Database Service (`db-service.yaml`)**:
   - A `ClusterIP` service for the database, allowing internal access to the MySQL instance from other services.

    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: db
      namespace: webapp
    spec:
      selector:
        app: database
      ports:
      - protocol: TCP
        port: 3306
        targetPort: 3306
    ```

6. **Proxy Deployment (`proxy-deployment.yaml`)**:
   - Runs the proxy with two replicas.
   - Configured to forward traffic to the backend.

    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: proxy-deployment
      namespace: webapp
      labels:
        app: proxy-deployment
    spec:
      replicas: 2
      selector:
        matchLabels:
          app: proxy
      template:
        metadata:
          labels:
            app: proxy
        spec:
          containers:
          - name: proxy
            image: omarbanna/proxy10:latest
            imagePullPolicy: IfNotPresent
            ports:
            - containerPort: 443
    ```

7. **Proxy Service (`proxy-service.yaml`)**:
   - A `NodePort` service for external access to the proxy, exposing it on port `30002`.

    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: proxy-service
      namespace: webapp
    spec:
      selector:
        app: proxy
      type: NodePort
      ports:
      - name: https
        protocol: TCP
        port: 443
        targetPort: 443
        nodePort: 30002
    ```

8. **Persistent Volume (`PV.yaml`)**:
   - A 5GB persistent volume for the database storage.

    ```yaml
    apiVersion: v1
    kind: PersistentVolume
    metadata:
      name: db-pv
      namespace: webapp
    spec:
      capacity:
        storage: 5Gi
      storageClassName: standard
      accessModes:
      - ReadWriteMany
      persistentVolumeReclaimPolicy: Retain
      hostPath:
        path: "/var/mysql"
    ```

9. **Persistent Volume Claim (`PVC.yaml`)**:
   - A 1GB persistent volume claim used by the MySQL database.

    ```yaml
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: my-pvc
      namespace: webapp
    spec:
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 1Gi
    ```

10. **Credentials Volume (`credentials-Volume.yaml`)**:
    - The `db-credentials.txt` secret is mounted in the backend and database deployments, providing secure access to sensitive data.

    ```yaml
    apiVersion: v1
    kind: Secret
    metadata:
      name: db-secret
      namespace: webapp
    data:
      db-password: U3Ryb25nUEBzc3cwcmQyMDI0
    ```
11.pods testing
![image](https://github.com/user-attachments/assets/2dc2867f-91fd-4d69-b005-b4f1756f94de)


### Key Considerations

- **Namespace Isolation**: All resources are contained within the `webapp` namespace, ensuring isolation.
- **Service Communication**: The proxy communicates with the backend using the `ClusterIP` service, while the backend communicates with the database service.
- **Data Persistence**: The MySQL database uses a persistent volume claim (`PVC`) to ensure data persistence across pod restarts.
- **Secret Management**: Sensitive database credentials are stored securely in Kubernetes secrets and mounted into the appropriate pods.
  
