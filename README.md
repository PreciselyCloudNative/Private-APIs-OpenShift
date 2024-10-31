# Deployment of GeoAddressing Service on OpenShift with AWS EFS CSI Driver Operator

This guide provides a step-by-step walkthrough for deploying the GeoAddressing service on an OpenShift cluster using the AWS Elastic File System (EFS) Container Storage Interface (CSI) Driver Operator. By following this guide, you will set up persistent storage using AWS EFS, configure necessary secrets for image pulling from AWS ECR, and deploy the GeoAddressing service along with its dependencies.

## Table of Contents

1.  [Prerequisites](#prerequisites)
2.  [Infrastructure Setup](#infrastructure-setup)
    *   [Install AWS EFS CSI Driver Operator](#install-aws-efs-csi-driver-operator)
    *   [Create Secret for Image Pull](#create-secret-for-image-pull)
3.  [Deployment](#deployment)
    *   [Create Storage Class](#create-storage-class)
    *   [Create Persistent Volume](#create-persistent-volume)
    *   [Create Persistent Volume Claim](#create-persistent-volume-claim)
    *   [Deploy Addressing-USA Service](#deploy-addressing-usa-service)
    *   [Expose Addressing-USA Service](#expose-addressing-usa-service)
    *   [Deploy Regional-Addressing Service](#deploy-regional-addressing-service)
    *   [Expose Regional-Addressing Service](#expose-regional-addressing-service)
    *   [Create Route for External Access](#create-route-for-external-access)
4.  [Testing the Deployment](#testing-the-deployment)
5.  [Cleanup](#cleanup)
6.  [Troubleshooting](#troubleshooting)
7.  [References](#references)

- - -

## Prerequisites

*   An OpenShift cluster deployed on AWS.
*   AWS CLI configured with appropriate permissions.
*   Access to AWS ECR where the Docker images are stored.
*   Basic understanding of Kubernetes and OpenShift concepts.

- - -

## Infrastructure Setup

### Install AWS EFS CSI Driver Operator

The AWS EFS CSI Driver allows you to use Amazon EFS with your Kubernetes cluster. Install the operator by following the official OpenShift documentation:

[Install AWS EFS CSI Driver Operator](https://docs.openshift.com/rosa/storage/container_storage_interface/persistent-storage-csi-aws-efs.html)

### Create Secret for Image Pull

To pull Docker images from a private AWS ECR repository, you need to create a Kubernetes secret with Docker registry credentials.

#### Obtain AWS ECR Credentials

Run the following command to get the Docker login password for AWS ECR:

```
aws ecr get-login-password --region <your-region>
```

Replace `<your-region>` with your AWS region (e.g., `us-east-2`).

#### Create Kubernetes Secret via Command Line

```
oc create secret docker-registry aws-ecr-secret \
  --docker-server=<your-ecr-repo> \
  --docker-username=AWS \
  --docker-password=<output-from-previous-command>
```

*   Replace `<your-ecr-repo>` with your ECR repository URL (e.g., `022735856553.dkr.ecr.us-east-2.amazonaws.com`).
*   Replace `<output-from-previous-command>` with the password obtained from the AWS CLI command.

#### Create Kubernetes Secret via YAML

Alternatively, you can create the secret using a YAML file:

```
apiVersion: v1
kind: Secret
metadata:
  name: aws-ecr-secret
  namespace: default
data:
  .dockerconfigjson: <base64-encoded-docker-config>
type: kubernetes.io/dockerconfigjson
```

*   Replace `<base64-encoded-docker-config>` with the base64-encoded Docker config JSON.

- - -

## Deployment

### Create Storage Class

Create a StorageClass named `geoaddressing-efs-sc` to define how volumes are provisioned.

```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: geoaddressing-efs-sc
provisioner: efs.csi.aws.com
parameters:
  directoryPerms: '775'
  fileSystemId: fs-0abc4e38a1bebb600  # Replace with your EFS File System ID
  gid: '0'
  provisioningMode: efs-ap
  uid: '0'
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

**Note**: Replace `fs-0abc4e38a1bebb600` with your actual EFS File System ID.

Apply the configuration:

```
oc apply -f storage-class.yaml
```

### Create Persistent Volume

Define a PersistentVolume (PV) named `geoaddressing-pv` that uses the storage class created earlier.

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: geoaddressing-pv
spec:
  capacity:
    storage: 200Gi
  accessModes:
    - ReadWriteMany
  storageClassName: geoaddressing-efs-sc
  csi:
    driver: efs.csi.aws.com
    volumeHandle: fs-0abc4e38a1bebb600  # Replace with your EFS File System ID
    volumeAttributes:
      path: /
  mountOptions:
    - tls
  persistentVolumeReclaimPolicy: Delete
  volumeMode: Filesystem
```

**Note**: Replace `fs-0abc4e38a1bebb600` with your EFS File System ID.

Apply the configuration:

```
oc apply -f persistent-volume.yaml
```

### Create Persistent Volume Claim

Create a PersistentVolumeClaim (PVC) named `geoaddressing-pvc` to request storage from the PV.

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: geoaddressing-pvc
  namespace: default
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 200Gi
  volumeName: geoaddressing-pv
  storageClassName: geoaddressing-efs-sc
  volumeMode: Filesystem
```

Apply the configuration:

```
oc apply -f persistent-volume-claim.yaml
```

### Deploy Addressing-USA Service

Deploy the `addressing-usa` service using a Deployment resource.

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: addressing-usa
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: addressing-usa
  template:
    metadata:
      labels:
        app: addressing-usa
    spec:
      volumes:
        - name: geoaddressing-host-volume
          persistentVolumeClaim:
            claimName: geoaddressing-pvc
      containers:
        - name: addressing-svc
          image: '022735856553.dkr.ecr.us-east-2.amazonaws.com/addressing-service:2.0.1'  # Update with your Image Repo
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 8080
          env:
            - name: DATA_PATH
              value: /mnt/data/geoaddressing-data
            - name: GEOCODE_VERIFY_ENABLED
              value: 'true'
            - name: LOOKUP_ENABLED
              value: 'false'
            - name: REVERSEGEOCODE_ENABLED
              value: 'false'
            - name: AUTOCOMPLETE_ENABLED
              value: 'false'
            - name: DATA_REGION
              value: USA
            - name: OTEL_TRACES_EXPORTER
              value: none
            - name: AUTH_ENABLED
              value: 'false'
          volumeMounts:
            - name: geoaddressing-host-volume
              mountPath: /mnt/data/geoaddressing-data
      imagePullSecrets:
        - name: aws-ecr-secret  # Use the secret created earlier
```

**Note**: Update the `image` field with your own Docker image repository.

Apply the configuration:

```
oc apply -f addressing-usa-deployment.yaml
```

### Expose Addressing-USA Service

Create a Service to expose the `addressing-usa` Deployment internally within the cluster.

```
apiVersion: v1
kind: Service
metadata:
  name: addressing-usa
  namespace: default
spec:
  selector:
    app: addressing-usa
  ports:
    - protocol: TCP
      port: 8080
      targetPort: http
  type: ClusterIP
```

Apply the configuration:

```
oc apply -f addressing-usa-service.yaml
```

### Deploy Regional-Addressing Service

Deploy the `regional-addressing` service, which depends on the `addressing-usa` service.

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: regional-addressing
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: regional-addressing
  template:
    metadata:
      labels:
        app: regional-addressing
    spec:
      containers:
        - name: geo-addressing
          image: '022735856553.dkr.ecr.us-east-2.amazonaws.com/regional-addressing-service:2.0.1'  # Update with your Image Repo
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 8080
          env:
            - name: ADDRESSING_BASE_URL
              value: 'http://addressing-usa.default.svc.cluster.local:8080'
            - name: ADDRESSING_EXPRESS_BASE_URL
              value: 'http://geo-addressing-express.default.svc.cluster.local:8080'
            - name: SUPPORTED_COUNTRIES_GEOCODE
              value: 'usa,express'
            - name: SUPPORTED_COUNTRIES_VERIFY
              value: 'usa,express'
            - name: SUPPORTED_COUNTRIES_LOOKUP
              value: usa
            - name: SUPPORTED_COUNTRIES_AUTOCOMPLETE
              value: 'usa,express'
            - name: SUPPORTED_COUNTRIES_REVERSE_GEOCODE
              value: usa
            - name: AUTH_ENABLED
              value: 'false'
            - name: IS_HELM_SOLUTION
              value: 'true'
            - name: IS_NO_COUNTRY_ENABLED_V2
              value: 'true'
            - name: COUNTRY_FINDER_SINGLE_LINE_ENABLED
              value: 'false'
            - name: OTEL_TRACES_EXPORTER
              value: none
      imagePullSecrets:
        - name: aws-ecr-secret  # Use the secret created earlier
```

**Note**: Update the `image` field with your own Docker image repository.

Apply the configuration:

```
oc apply -f regional-addressing-deployment.yaml
```

### Expose Regional-Addressing Service

Create a Service to expose the `regional-addressing` Deployment internally within the cluster.

```
apiVersion: v1
kind: Service
metadata:
  name: regional-addressing
  namespace: default
spec:
  selector:
    app: regional-addressing
  ports:
    - protocol: TCP
      port: 80
      targetPort: http
  type: ClusterIP
```

Apply the configuration:

```
oc apply -f regional-addressing-service.yaml
```

### Create Route for External Access

To access the `regional-addressing` service from outside the cluster, create an OpenShift Route.

```
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: regional-addressing
  namespace: default
spec:
  to:
    kind: Service
    name: regional-addressing
  port:
    targetPort: http
  wildcardPolicy: None
```

Apply the configuration:

```
oc apply -f regional-addressing-route.yaml
```

This will create a URL through which you can access the `regional-addressing` service externally.

- - -

## Testing the Deployment

Use Postman or any API testing tool to test the endpoints exposed by the `regional-addressing` service.

*   Obtain the route URL created in the previous step.
*   Send HTTP requests to the service to verify it is functioning as expected.
*   Example endpoints:
    *   `/geocode`
    *   `/verify`
    *   `/lookup`
    *   `/autocomplete`
    *   `/reversegeocode`

- - -

## Cleanup

To remove all the resources created during this deployment, execute the following commands:

```
oc delete deployment addressing-usa regional-addressing
oc delete service addressing-usa regional-addressing
oc delete route regional-addressing
oc delete pvc geoaddressing-pvc
oc delete pv geoaddressing-pv
oc delete sc geoaddressing-efs-sc
oc delete secret aws-ecr-secret
```

- - -

## Troubleshooting

*   **Pods Not Starting**: Check if the `imagePullSecrets` are correctly referenced and the secret contains valid credentials.
*   **Persistent Volume Issues**: Ensure that the EFS File System ID and the `volumeHandle` are correctly specified.
*   **Service Connectivity**: Verify that the services are correctly exposing the pods and that the selectors match the labels.
*   **Route Not Accessible**: Ensure that the route has been created and that there are no network policies blocking access.

Use the following commands to get more information:

*   View pod logs:
    
    ```
    oc logs <pod-name>
    ```
    
*   Describe resources:
    
    ```
    oc describe pod/service/deployment <resource-name>
    ```
    
*   Check events:
    
    ```
    oc get events
    ```
    

- - -

## References

*   [AWS EFS CSI Driver Operator Documentation](https://docs.openshift.com/rosa/storage/container_storage_interface/persistent-storage-csi-aws-efs.html)
*   [Kubernetes Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
*   [OpenShift Routes](https://docs.openshift.com/container-platform/4.7/networking/routes/route-configuration.html)
*   [AWS ECR Authentication](https://docs.aws.amazon.com/AmazonECR/latest/userguide/Registries.html)

- - -
