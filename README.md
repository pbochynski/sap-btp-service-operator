
[![Coverage Status](https://coveralls.io/repos/github/SAP/sap-btp-service-operator/badge.svg?branch=main)](https://coveralls.io/github/SAP/sap-btp-service-operator?branch=main)
[![Build Status](https://github.com/SAP/sap-btp-service-operator/workflows/Go/badge.svg)](https://github.com/SAP/sap-btp-service-operator/actions)
[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://github.com/SAP/sap-btp-service-operator/blob/master/LICENSE)
[![Go Report Card](https://goreportcard.com/badge/github.com/SAP/sap-btp-service-operator)](https://goreportcard.com/report/github.com/SAP/sap-btp-service-operator)
[![REUSE status](https://api.reuse.software/badge/github.com/SAP/sap-btp-service-operator)](https://api.reuse.software/info/github.com/SAP/sap-btp-service-operator)

# SAP Business Technology Platform (SAP BTP) Service Operator for Kubernetes

With the SAP BTP service operator, you can consume [SAP BTP services](https://platformx-d8bd51250.dispatcher.us2.hana.ondemand.com/protected/index.html#/viewServices?) from your Kubernetes cluster using Kubernetes-native tools. 
SAP BTP service operator allows you to provision and manage service instances and service bindings of SAP BTP services so that your Kubernetes-native applications can access and use needed services from the cluster.  
The SAP BTP service operator is based on the [Kubernetes Operator pattern](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/).

## Table of Contents
* [Architecture](#architecture)
* [Prerequisites](#prerequisites)
* [Setup](#setup)
  * [Managing access](#managing-access)
  * [Working with Multiple Subaccounts](#working-with-multiple-subaccounts)
* [Using the SAP BTP Service Operator](#using-the-sap-btp-service-operator)
    * [Service Instance](#service-instance)
    * [Service Binding](#service-binding)
      * [Formats of Service Binding Secrets](#formats-of-service-binding-secrets)
      * [Service Binding Rotation](#service-binding-rotation)
    * [Passing parameters](#passing-parameters)
* [Reference Documentation](#reference-documentation)
    * [Service Instance properties](#Service-Instance-properties)
    * [Service Binding properties](#service-binding-properties)
* [Uninstalling the Operator](#uninstalling-the-operator)
* [Troubleshooting and Support](#troubleshooting-and-support)

## Architecture
SAP BTP service operator communicates with [Service Manager](https://help.sap.com/viewer/09cc82baadc542a688176dce601398de/Cloud/en-US/3a27b85a47fc4dff99184dd5bf181e14.html) that uses the [Open service broker API](https://github.com/openservicebrokerapi/servicebroker) to communicate with service brokers, acting as an intermediary for the Kubernetes API Server to negotiate the initial provisioning and retrieve the credentials necessary for the application to use a managed service.<br><br>
It is implemented using a [CRDs-based](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/#custom-resources) architecture.

![img](./docs/images/architecture.png)

## Prerequisites
- SAP BTP [Global Account](https://help.sap.com/viewer/65de2977205c403bbc107264b8eccf4b/Cloud/en-US/d61c2819034b48e68145c45c36acba6e.html) and [Subaccount](https://help.sap.com/viewer/65de2977205c403bbc107264b8eccf4b/Cloud/en-US/55d0b6d8b96846b8ae93b85194df0944.html) 
- [Working with SAP Service Manager](https://help.sap.com/docs/service-manager/sap-service-manager/working-with-sap-service-manager).
- [Kubernetes cluster](https://kubernetes.io/) running version 1.17 or higher 
- [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) v1.17 or higher
- [helm](https://helm.sh/) v3.0 or higher

[Back to top](#sap-business-technology-platform-sap-btp-service-operator-for-kubernetes)

## Setup
1. Install [cert-manager](https://cert-manager.io/docs/installation/kubernetes)
   - for releases v0.1.18 or higher use cert-manager v1.6.0 or higher 
   - for releases v0.1.17 or lower use cert-manager lower than v1.6.0

2. Obtain the access credentials for the SAP BTP service operator:
   * Create an instance of the SAP Service Manager service (technical name: `service-manager`) with the plan:
       `service-operator-access`<br/>
     *Note:<br> If you can't see the needed plan, you need to entitle your subaccount to use SAP Service Manager service.For more information about how to entitle a service to a subaccount, see:[Configure Entitlements and Quotas for Subaccounts](https://help.sap.com/viewer/65de2977205c403bbc107264b8eccf4b/Cloud/en-US/5ba357b4fa1e4de4b9fcc4ae771609da.html)<br>
   For more information about creating service instances, see:
     * [Creating Service Instances Using the SAP BTP Cockpit](https://help.sap.com/viewer/09cc82baadc542a688176dce601398de/Cloud/en-US/bf71f6a7b7754dbd9dfc2569791ccc96.html)
     * [Creating Service Instances using BTP CLI](https://help.sap.com/docs/btp/btp-cli-command-reference/btp-create-services-binding) 
   
   * Create a binding to the created service instance.
      
      For more information about creating service bindings, see:  
      * [Creating Service Bindings Using the SAP BTP Cockpit](https://help.sap.com/viewer/09cc82baadc542a688176dce601398de/Cloud/en-US/55b31ea23c474f6ba2f64ee4848ab1b3.html)
      * [Creating Service Bindings Using BTP CLI](https://help.sap.com/docs/btp/btp-cli-command-reference/btp-create-services-binding). 
   
   * Retrieve the generated access credentials from the created binding:
   
         The example of the default binding object used if no credentials type is specified:
      
       ```json
        {
            "clientid": "xxxxxxx",
            "clientsecret": "xxxxxxx",
            "url": "https://mysubaccount.authentication.eu10.hana.ondemand.com",
            "xsappname": "b15166|service-manager!b1234",
            "sm_url": "https://service-manager.cfapps.eu10.hana.ondemand.com"
        }
       ```
       The example of the binding object with the specified X.509 certificate:
    
       ```json
       {
            "clientid": "xxxxxxx",
            "certificate": "-----BEGIN CERTIFICATE-----...-----END CERTIFICATE-----\n-----BEGIN CERTIFICATE-----..-----END CERTIFICATE-----\n-----BEGIN CERTIFICATE-----...-----END CERTIFICATE-----\n",
            "key": "-----BEGIN RSA PRIVATE KEY-----...-----END RSA PRIVATE KEY-----\n",
            "certurl": "https://mysubaccount.authentication.cert.eu10.hana.ondemand.com",
            "xsappname": "b15166|service-manager!b1234",
            "sm_url": "https://service-manager.cfapps.eu10.hana.ondemand.com"
        }
       ```
3. Add SAP BTP service operator chart repository  
   ```bash
    helm repo add sap-btp-operator https://sap.github.io/sap-btp-service-operator
   ```
4. Deploy the SAP BTP service operator in the cluster using the obtained access credentials:<br>
   
   *Note:<br>
    If you are deploying the SAP BTP service operator in the registered cluster based on the Service Catalog (svcat) and Service Manager agent so that you can migrate svcat-based content to service operator-based content, add ```--set cluster.id=<clusterID>  ``` to your deployment script.*<br>*For more information, see the step 2 of the Setup section of [Migration to SAP BTP service operator](https://github.com/SAP/sap-btp-service-operator-migration/blob/main/README.md).*
   
   An example of the deployment that uses the default access credentials type:
    ```bash
    helm upgrade --install <release-name> sap-btp-operator/sap-btp-operator \
        --create-namespace \
        --namespace=sap-btp-operator \
        --set manager.secret.clientid=<clientid> \
        --set manager.secret.clientsecret=<clientsecret> \
        --set manager.secret.sm_url=<sm_url> \
        --set manager.secret.tokenurl=<auth_url>
    ```
   An example of the deployment that uses the X.509 certificate:
    ```bash
    helm upgrade --install <release-name> sap-btp-operator/sap-btp-operator \
        --create-namespace \
        --namespace=sap-btp-operator \
        --set manager.secret.clientid=<clientid> \
        --set manager.secret.tls.crt="$(cat /path/to/cert)" \
        --set manager.secret.tls.key="$(cat /path/to/key)" \
        --set manager.secret.sm_url=<sm_url> \
        --set manager.secret.tokenurl=<auth_url>
    ```

The credentials which are provided during the installation are stored in a secret named 'sap-btp-service-operator', in the 'sap-btp-operator' namespace.
These credentials are used by the BTP service operator to communicate with the SAP BTP subaccount.

<details>
<summary> BTP Access Secret Structure </summary>

Default Access Credentials
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: sap-btp-service-operator
  namespace: sap-btp-operator
type: Opaque
stringData:
  clientid: "<clientid>"
  clientsecret: "<clientsecret>"
  sm_url: "<sm_url>"
  tokenurl: "<auth_url>"
  tokenurlsuffix: "/oauth/token"
```
mTLS Access Credentials

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: sap-btp-service-operator
  namespace: sap-btp-operator
type: Opaque
stringData:
  clientid: <clientid>
  tls.crt: <certificate>
  tls.key: <key>
  sm_url: <sm_url>
  tokenurl: <auth_url>
  tokenurlsuffix: "/oauth/token"
```

</details>


**Note:**<br> To rotate the credentials between the BTP service operator and Service Manager, you have to create a new binding for the service-operator-access service instance, and then to execute the setup script again, with the new set of credentials. Afterward, you can delete the old binding.

[Back to top](#sap-business-technology-platform-sap-btp-service-operator-for-kubernetes)

## Managing Access
By default, the SAP BTP operator has cluster-wide permissions.<br>You can also limit them to one or more namespaces; for this, you need to set the following two helm parameters:

```
--set manager.allow_cluster_access=false
--set manager.allowed_namespaces={namespace1, namespace2..}
```
**Note:**<br> If `allow_cluster_access` is set to true, then `allowed_namespaces` parameter is ignored.

[Back to top](#sap-business-technology-platform-sap-btp-service-operator-for-kubernetes)

## Working with Multiple Subaccounts

By default, a Kubernetes cluster is associated with a single subaccount (as described in step 4 of the [Setup](#setup) section). 
Consequently, any service instance created within any namespace will be provisioned in that subaccount.

However, the SAP BTP service operator also supports multi-subaccount configurations in a single cluster. This is achieved through:

- Namespace-based mapping: Connect different namespaces to separate subaccounts. This approach leverages dedicated credentials configured for each namespace.
  
- Explicit instance-level mapping: Define the specific subaccount for each service instance, regardless of the namespace context.

  Both can be achieved through dedicated secrets managed in the centrally managed namespace. Choosing the most suitable approach depends on your specific needs and application architecture.

**Note:**
The system's centrally managed namespace is set by the value in `.Values.manager.management_namespace`. You can provide this value during installation (refer to step 4 in the [Setup](#setup) section).
If you don't specify this value, the system will use the installation namespace as the default.

### Subaccount For a Namespace

To associate a namespace to a specific subaccount you maintain the access credentials to the subaccount in a `Secret` that is dedicated to a specific namespace.
Define a secret named: `<namespace-name>-sap-btp-service-operator` in the centrally-managed namespace.

Default Access Credentials
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: <namespace-name>-sap-btp-service-operator
  namespace: <centrally-managed-namespace>
type: Opaque
stringData:
  clientid: "<clientid>"
  clientsecret: "<clientsecret>"
  sm_url: "<sm_url>"
  tokenurl: "<auth_url>"
  tokenurlsuffix: "/oauth/token"
```
mTLS Access Credentials

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: <namespace-name>-sap-btp-service-operator
  namespace: <centrally-managed-namespace>
type: Opaque
stringData:
  clientid: <clientid>
  tls.crt: <certificate>
  tls.key: <key>
  sm_url: <sm_url>
  tokenurl: <auth_url>
  tokenurlsuffix: "/oauth/token"
```
### Subaccount for a ServiceInstance Resource

You can deploy service instances belonging to different subaccounts within the same namespace. To achieve this, follow these steps:

1. Store access credentials: Securely store the access credentials for each subaccount in separate `Secret` resources within the centrally-managed namespace.
2. Specify subaccount per service: In the `ServiceInstance` resource, use the `btpAccessCredentialsSecret` property to reference the specific `Secret` containing the relevant subaccount's credentials. This explicitly tells the operator which subaccount to use to provision the service instance.


#### Define a new secret

Default Access Credentials

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: <my-secret>
  namespace: <centrally managed namespace>
type: Opaque
stringData:
  clientid: "<clientid>"
  clientsecret: "<clientsecret>"
  sm_url: "<sm_url>"
  tokenurl: "<auth_url>"
  tokenurlsuffix: "/oauth/token"
```
mTLS Access Credentials

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: <my-secret>
  namespace: <centrally managed namespace>
type: Opaque
stringData:
  clientid: <clientid>
  tls.crt: <certificate>
  tls.key: <key>
  sm_url: <sm_url>
  tokenurl: <auth_url>
  tokenurlsuffix: "/oauth/token"
```
#### Configure the secret name in the `ServiceInstance` resource within the property `btpAccessCredentialsSecret`:
The secret must be located in the management namespace
```yaml
apiVersion: services.cloud.sap.com/v1
kind: ServiceInstance
metadata:
  name: sample-instance-1
spec:
  serviceOfferingName: service-manager
  servicePlanName: subaccount-audit
  btpAccessCredentialsSecret: mybtpsecret
```

##### Secrets Precedence
SAP BTP service operator searches for the credentials in the following order:
1. Explicit secret defined in the `ServiceInstance`
2. Default namespace secret
3. Default cluster secret

[Back to top](#sap-business-technology-platform-sap-btp-service-operator-for-kubernetes)

## Using the SAP BTP Service Operator

#### Service Instance

1.  To create an instance of a service offered by SAP BTP, first create a `ServiceInstance` custom-resource file:

```yaml
    apiVersion: services.cloud.sap.com/v1
    kind: ServiceInstance
    metadata:
        name: my-service-instance
    spec:
        serviceOfferingName: sample-service
        servicePlanName: sample-plan
        externalName: my-service-btp-name
        parameters:
          key1: val1
          key2: val2
   ```

   *   `<offering>` - The name of the SAP BTP service that you want to create. 
       To learn more about viewing and managing the available services for your subaccount in the SAP BTP cockpit, see [Service Marketplace](https://help.sap.com/viewer/09cc82baadc542a688176dce601398de/Cloud/en-US/affcc245c332433ba71917ff715b9971.html). 
        
        Tip: Use the *Environment* filter to get all offerings that are relevant for Kubernetes.
   *   `<plan>` - The plan of the selected service offering you want to create.

2.  Apply the custom resource file in your cluster to create the instance.

    ```bash
    kubectl apply -f path/to/my-service-instance.yaml
    ```

3.  Check that the service's status in your cluster is **Created**.

    ```bash
    kubectl get serviceinstances
    NAME                  OFFERING          PLAN        STATUS    AGE
    my-service-instance   <offering>        <plan>      Created   44s
    ```
[Back to top](#sap-business-technology-platform-sap-btp-service-operator-for-kubernetes)

#### Service Binding

To allow an application to obtain access credentials to communicate with a service, create a `ServiceBinding` custom resource. Set the `serviceInstanceName` field within the `ServiceBinding` to match the name of the `ServiceInstance` resource you previously created.

These access credentials are available to applications through a `Secret` resource generated in your cluster.

##### Structure of the ServiceBinding Custom Resource

```yaml
apiVersion: services.cloud.sap.com/v1
kind: ServiceBinding
metadata:
  name: sample-binding
spec:
  serviceInstanceName: sample-instance
  externalName: my-binding-external
  secretName: my-secret
  parameters:
    key1: val1
    key2: val2      
```

#### Procedure

1.  Apply the custom resource file in your cluster to create the `ServiceBinding`:

    ```bash
    kubectl apply -f path/to/my-binding.yaml
    ```

2.  Verify that your `ServiceBinding` status is **Created** before you proceed:

    ```bash
    kubectl get servicebindings
    NAME         INSTANCE              STATUS    AGE
    my-binding   my-service-instance   Created   16s
    
    ```

3.  Check that the `Secret` with the name as specified in the  `spec.secretName` field  of the `ServiceBinding` custom resource is created. Remember, the `Secret` contains access credentials needed for the apps to use the service:

    ```bash
    kubectl get secrets
    NAME         TYPE     DATA   AGE
    my-secret   Opaque   5      32s
    ```
    
    See [Using Secrets](https://kubernetes.io/docs/concepts/configuration/secret/#using-secrets) to learn about different options on how to use the credentials from your application running in the Kubernetes cluster, 

[Back to top](#sap-business-technology-platform-sap-btp-service-operator-for-kubernetes)

#### Formats of Service Binding Secrets 

You can use different attributes in your `ServiceBinding` resource to generate different formats of your `Secret` resources.

Even though `Secret` resources can come in various formats, they all share a common basic content. The parameters within the `Secret` fall into two categories:

- Credentials returned from the broker: These credentials allow your applications to access and consume the service.
- Attributes of the associated `ServiceInstance`: This information provides details about the service instance itself.
  
Now let's explore these various formats:

##### Key-Value Pairs (Default)

If you do not use any of the attributes, the generated `Secret` will be in a key-value pair format. 

`ServiceBinding`

```yaml
apiVersion: services.cloud.sap.com/v1
kind: ServiceBinding
metadata:
  name: sample-binding
spec:
  serviceInstanceName: sample-instance
```
`Secret`

```yaml
apiVersion: v1
metadata:
  name: sample-binding
kind: Secret
stringData:
  uri: https://my-service.authentication.eu10.hana.ondemand.com
  client_id: admin
  client_secret: ********
  instance_guid: your-sample-instance-guid // The service instance ID
  instance_name: sample-instance // Taken from the service instance external_name field if set. Otherwise from metadata.name
  plan: sample-plan // The service plan name                
  type: sample-service  // The service offering name
```

##### Credentials as JSON Object

To show credentials returned from the broker within the `Secret` resource  as a JSON object, use the `secretKey` attribute in the `ServiceBinding` spec.
The value of this `secretKey` is the name of the key that stores the credentials in JSON format:

`ServiceBinding`

```yaml
apiVersion: services.cloud.sap.com/v1
kind: ServiceBinding
metadata:
  name: sample-binding
spec:
  serviceInstanceName: sample-instance
  secretKey: myCredentials
```
`Secret`

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: sample-binding
stringData:
    myCredentials:
      uri: https://my-service.authentication.eu10.hana.ondemand.com,
      client_id: admin,
      client_secret: ********
    instance_guid: your-sample-instance-guid // The service instance ID
    instance_name: sample-binding // Taken from the service instance external_name field if set. Otherwise from metadata.name 
    plan: sample-plan // The service plan name
    type: sample-service // The service offering name
```

##### Credentials and Service Info as One JSON Object

To show both credentials returned from the broker and additional `ServiceInstance` attributes as a JSON object, use the `secretRootKey` attribute in the `ServiceBinding` spec.

The value of `secretRootKey` is the name of the key that stores both credentials and `ServiceInstance` info in JSON format.

`ServiceBinding`

```yaml
apiVersion: services.cloud.sap.com/v1
kind: ServiceBinding
metadata:
  name: sample-binding
spec:
  serviceInstanceName: sample-instance
  secretRootKey: myCredentialsAndInstance
```
`Secret`

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: sample-binding
stringData:
    myCredentialsAndInstance:
        uri: https://my-service.authentication.eu10.hana.ondemand.com,
        client_id: admin,
        client_secret: ********,
        instance_guid: your-sample-instance-guid, // The service instance id
        instance_name: sample-instance-name, // Taken from the service instance external_name field if set. Otherwise from metadata.name
        plan: sample-instance-plan, // The service plan name
        type: sample-instance-offering, // The service offering name
```
##### Custom Formats 

For additional flexibility, you can model the `Secret` resources according to your needs.<br>
To generate a custom-formatted `Secret`, use the `secretTemplate` attribute in the `ServiceBinding` spec.

This attribute expects a Go template as its value (for more information, see [Go Templates](https://pkg.go.dev/text/template)).<br>
Ensure the template is in YAML format, and its structure is of a Kubernetes `Secret`. 

In the provided `Secret`, you can customize the `metadata` and `stringData` sections with the following options:

- `metadata`: labels and annotations
- `stringData`: customize or utilize one of the available formatting options as detailed in the [Formats of Service Binding Secrets](#formats-of-service-binding-secrets) section.


**Important**:  If you customize `stringData`, it takes precedence over the pre-defined formats (if you parallelly provided one of them).

Provided templates are then executed on a map with the following available attributes:

| Reference         | Description                                |                                                                          
|:-----------------|:--------------------------------------------|
| `instance.instance_guid` |  The service instance ID.     |
| `instance.instance_name` |  The service instance name.   |                                                
| `instance.plan`   |  The name of the service plan used to create this service instance. |  
| `instance.type`   |  The name of the associated service offering. |  
| `credentials.attributes(var)`   |  The content of the credentials depends on a service. For more details, refer to the documentation of the service you're using. |  

Below are two examples demonstrating 'ServiceBinding' and generated 'Secret' resources. The first `ServiceBinding` example utilizes a custom template, while the second example combines a custom template with a predefined formatting option:


###### Example of a binding with customized metadata and stringData sections

In this example, you specify both `Metadata` and `stringData` in the `secretTemplate`:

`ServiceBinding`

```yaml
apiVersion: services.cloud.sap.com/v1
kind: ServiceBinding
metadata:
  name: sample-binding
spec:
  serviceInstanceName: sample-instance
  secretTemplate: |
    apiVersion: v1
    kind: Secret
    metadata:
      labels:
        service_plan: {{ .instance.plan }}
      annotations:
        instance: {{ .instance.instance_name }}
    stringData:
      USERNAME: {{ .credentials.client_id }}
      PASSWORD: {{ .credentials.client_secret }}
```

`Secret`

```yaml
apiVersion: v1
kind: Secret
metadata:
  labels:
    service_plan: sample-plan
  annotations:
    instance: sample-instance
stringData:
  USERNAME: admin
  PASSWORD: ********
```

###### Example of a binding with customized metadata section and applied pre-existing formating option for stringData (credentials as JSON object):

In this example, you omit `stringData` from the `secretTemplate` and use the `secretKey` to format your `stringData` instead. (remember: [Formats of Service Binding Secrets](https://github.com/SAP/sap-btp-service-operator/blob/fba58755e4e5b8a25510811349f08242b4b2a897/README.md#formats-of-service-binding-secrets)):


`ServiceBinding`

```yaml
apiVersion: services.cloud.sap.com/v1
kind: ServiceBinding
metadata:
  name: sample-binding
spec:
  serviceInstanceName: sample-instance
  secretKey: myCredentials
  secretTemplate: |
    apiVersion: v1
    kind: Secret
    metadata:
      labels:
        service_plan: {{ .instance.plan }}
      annotations:
        instance: {{ .instance.instance_name }}
```

`Secret`

```yaml
apiVersion: v1
kind: Secret
metadata:
  labels:
    service_plan: sample-plan
  annotations:
    instance: sample-instance
stringData:
  myCredentials:
    uri: https://my-service.authentication.eu10.hana.ondemand.com,
    client_id: admin,
    client_secret: ********
  instance_guid: your-sample-instance-guid // The service instance ID
  instance_name: sample-binding // Taken from the service instance external_name field if set. Otherwise from metadata.name
  plan: sample-plan // The service plan name
  type: sample-service // The service offering name
```

[Back to top](#sap-business-technology-platform-sap-btp-service-operator-for-kubernetes)

## Service Binding Rotation

Enhance security by automatically rotating the credentials associated with your service bindings. This process involves generating a new service binding while keeping the old credentials active for a specified period to ensure a smooth transition.

### Enabling Automatic Rotation

To enable automatic rotation of service bindings, use the `credentialsRotationPolicy` field within the `spec` section of the `ServiceBinding` resource. This field allows you to configure several parameters:

| Parameter         | Type     | Description                                                                                                                                   | Valid Values             |
|:-----------------|:---------|:----------------------------------------------------------------------------------------------------------------------------------------------|:-------------------------|
| `enabled` | bool | Controls whether the automatic rotation is enabled or disabled.                                  |                          |
| `rotationFrequency` | string | Specifies the desired time interval between binding rotations.                      | "m" (minute), "h" (hour) |                                   |
| `rotatedBindingTTL`   |  string | Determines how long to keep the old `ServiceBinding` resource after rotation (prior to deletion). The actual TTL may be slightly longer (details below). | "m" (minute), "h" (hour)                |   

**Note that the `credentialsRotationPolicy` does not manage the validity or expiration of the credentials themselves. This is determined by the specific service you are bound to.**

### Rotation Process

The `credentialsRotationPolicy` is evaluated periodically during a [control loop](https://kubernetes.io/docs/concepts/architecture/controller/), which runs on every service binding update or during a full reconciliation process. This means the actual rotation will occur in the closest upcoming reconciliation loop. 

### Immediate Rotation 

You can trigger an immediate rotation (regardless of the configured `rotationFrequency`) by adding the services.cloud.sap.com/forceRotate: "true" annotation to the `ServiceBinding` resource. This immediate rotation only works if automatic rotation is already enabled. 

#### Example

This example configures a `ServiceBinding` to rotate credentials every 25 days (600 hours) and keep the old `ServiceBinding` for 2 days (48 hours) before deleting it:

```yaml
apiVersion: services.cloud.sap.com/v1
kind: ServiceBinding
metadata:
  name: sample-binding
spec:
  serviceInstanceName: sample-instance
  credentialsRotationPolicy:
    enabled: true
    rotatedBindingTTL: 48h
    rotationFrequency: 600h
 ```

### After Rotation

Once the ServiceBinding is rotated:

The Secret is updated with the latest credentials. The old credentials are kept in a newly-created secret named 'original-secret-name(variable)-guid(variable)'.
This temporary secret is kept until the configured deletion time (TTL) expires.

### Checking Last Rotation

To view the timestamp of the last service binding rotation, refer to the `status.lastCredentialsRotationTime` field.

### Limitations

Automatic credential rotation cannot be enabled for a backup `ServiceBinding` (named: original-binding-name(variable)-guid(variable)) which is marked with the `services.cloud.sap.com/stale` label.
This backup service binding was created during the credentials rotation process to facilitate the process.

### Further Information

For more details about `ServiceBinding`, refer to the dedicated [Service Binding](#service-binding) section in this documentation.
  

[Back to top](#sap-business-technology-platform-sap-btp-service-operator-for-kubernetes)

### Passing Parameters

To set input parameters, you may use the `parameters` and `parametersFrom`
fields in the `spec` field of the `ServiceInstance` or `ServiceBinding` resource:
- `parameters`: can be used to specify a set of properties to be sent to the
  broker. The data specified will be passed "as-is" to the broker without any
  modifications - aside from converting it to JSON for transmission to the broker
  if the `spec` field is specified as `YAML`. Any valid `YAML` or
  `JSON` constructs are supported. Only one parameter field may be specified per
  `spec`.
- `parametersFrom`: enables you to specify one or more secrets, and the corresponding keys within those secrets, holding JSON-formatted parameters to be sent to the 
   broker.
  The `parametersFrom` field is a list that supports multiple sources referenced per `spec`, defining an asymmetric relationship where the `ServiceInstance` resource can define several related secrets.
- `watchParametersFromChanges`: (boolean) This field determines whether changes to the secret values referenced in `parametersFrom` should trigger an automatic update of the service instance. If `true`, any change to the referenced secret values will trigger the update of the service instance. Defaults to `false`.
 
While you may use either or both of `parameters` and `parametersFrom` fields, `watchParametersFromChanges` is only relevant when used alongside `parametersFrom`.

**Note** 
The `watchParametersFromChanges` field is only relevant for `ServiceInstance` resourcces because `ServiceBinding` resourcse can't be updated.


If multiple sources in the `parameters` and `parametersFrom` blocks are specified,
the final payload merges all of them at the top level.
If there are any duplicate properties defined at the top level, the specification
is considered to be invalid, the further processing of the `ServiceInstance`/`ServiceBinding`
resource stops and its `status` is marked with an error condition.

The format of the `spec` in YAML
```yaml
spec:
  ...
  parameters:
    name: value
  parametersFrom:
    - secretKeyRef:
        name: my-secret
        key: secret-parameter
```

The format of the `spec` in JSON
```json
{
  "spec": {
    "parameters": {
      "name": "value"
    },
    "parametersFrom": {
      "secretKeyRef": {
        "name": "my-secret",
        "key": "secret-parameter"
      }
    }
  } 
}
```
The `secret` with the `secret-parameter`- named key:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
type: Opaque
stringData:
  secret-parameter:
    '{
      "password": "password"
    }'
```
The final JSON payload to send to the broker:
```json
{
  "name": "value",
  "password": "password"
}
```

You can list multiple parameters in the `secret`. To do so, separate "key": "value" pairs with commas as in this example:
```yaml
secret-parameter:
  '{
    "password": "password",
    "key2": "value2",
    "key3": "value3"
  }'
```
[Back to top](#sap-business-technology-platform-sap-btp-service-operator-for-kubernetes).

## Reference Documentation

### Service Instance properties
#### Spec
| Parameter         | Type     | Description                                                                                                                                                                                                       |
|:-----------------|:---------|:------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| serviceOfferingName`*` | `string` | The name of the SAP BTP service offering.                                                                                                                                                                         |
| servicePlanName`*` | `string` | The plan to use for the service instance.                                                                                                                                                                         |
| servicePlanID   |  `string`  | The plan ID in case the service offering and plan name are ambiguous.                                                                                                                                                 |
| externalName       | `string` | The name for the service instance in SAP BTP, defaults to the instance `metadata.name` if not specified.                                                                                                          |
| parameters       | `[]object` | Some services support the provisioning of additional configuration parameters during the instance creation.<br/>For the list of supported parameters, check the documentation of the particular service offering. |
| parametersFrom | `[]object` | List of sources to populate parameters.                                                                                                                                                                           |
| watchParametersFromChanges | `bool` | This field determines whether changes to the secret values referenced in `parametersFrom` should trigger an automatic update of the service instance. When set to true, any change to the referenced secret values will trigger the update of the service instance. Defaults to `false`. It is only relevant when used in conjuction with the `parametersFrom` field. |
| customTags | `[]string` | A List of custom tags describing the ServiceInstance, will be copied to `ServiceBinding` secret in the key called `tags`.                                                                                           |
| userInfo | `object` | Contains information about the user that last modified this service instance.                                                                                                                                     |
| shared |  `*bool`   | The shared state. Possible values: true, false, or nil (value was not specified, counts as "false").                                                                                                              |
| btpAccessCredentialsSecret |  `string`   | Name of a secret that contains access credentials for the SAP BTP service operator. see [Working with Multiple Subaccounts](#Working-with-multiple-subaccounts)                                                                                 |


#### Status
| Parameter         | Type     | Description                                                                                                   |
|:-----------------|:---------|:-----------------------------------------------------------------------------------------------------------|
| instanceID   | `string` | The service instance ID in SAP Service Manager service.  |
| operationURL | `string` | The URL of the current operation performed on the service instance.  |
| operationType   |  `string`| The type of the current operation. Possible values are CREATE, UPDATE, or DELETE. |
| conditions       |  `[]condition`   | An array of conditions describing the status of the service instance.<br/>The possible condition types are:<br>- `Ready`: set to `true`  if the instance is ready and usable<br/>- `Failed`: set to `true` when an operation on the service instance fails.<br/> In the case of failure, the details about the error are available in the condition message.<br>- `Succeeded`: set to `true` when an operation on the service instance succeeded. In case of a `false` operation, it is considered as in progress unless a `Failed` condition exists.<br>- `Shared`: set to `true` when sharing of the service instance succeeded. set to `false` when unsharing of the service instance succeeded or when the service instance is not shared. |
| tags       |  `[]string`   | Tags describing the ServiceInstance as provided in the service catalog, will be copied to the `ServiceBinding` secret in the key called `tags`.|

#### Annotations
| Parameter         | Type                 | Description                                                                                                                                                                                                                         |
|:-----------------|:---------------------|:------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| services.cloud.sap.com/preventDeletion   | `map[string] string` | You can prevent deletion of any service instance by adding the following annotation: services.cloud.sap.com/preventDeletion : "true". To enable back the deletion of the instance, either remove the annotation or set it to false. |

### Service Binding properties
#### Spec
| Parameter                                   | Type       | Description                                                                                                                                                                                                                                                                                                                                                          |
|:--------------------------------------------|:---------|:---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| serviceInstanceName`*`                      | `string`   | The Kubernetes name of the service instance to bind.                                                                                                                                                                                                                                                                                                                 |
| serviceInstanceNamespace                    | `string`   | The namespace of the service instance to bind, if not specified the default is the binding's namespace.                                                                                                                                                                                                                                                              |
| externalName                                | `string`   | The name for the service binding in SAP BTP, defaults to the binding `metadata.name` if not specified.                                                                                                                                                                                                                                                               |
| secretName                                  | `string`   | The name of the secret where the credentials are stored, defaults to the binding `metadata.name` if not specified.                                                                                                                                                                                                                                                   |
| secretKey                                   | `string`  | The secret key is a part of the Secret object, which stores service-binding data (credentials) received from the broker. When the secret key is used, all the credentials are stored under a single key. This makes it a convenient way to store credentials data in one file when using volumeMounts. [Example](#formats-of-secret-objects)                         |
| secretRootKey                               | `string`  | The root key is a part of the Secret object, which stores service-binding data (credentials) received from the broker, as well as additional service instance information. When the root key is used, all data is stored under a single key. This makes it a convenient way to store data in one file when using volumeMounts. [Example](#formats-of-secret-objects) |
| parameters                                  |  `[]object`  | Some services support the provisioning of additional configuration parameters during the bind request.<br/>For the list of supported parameters, check the documentation of the particular service offering.                                                                                                                                                         |
| parametersFrom                              | `[]object` | List of sources to populate parameters.                                                                                                                                                                                                                                                                                                                              |
| userInfo                                    | `object`  | Contains information about the user that last modified this service binding.                                                                                                                                                                                                                                                                                         |
| credentialsRotationPolicy                   | `object`  | Holds automatic credentials rotation configuration.                                                                                                                                                                                                                                                                                                                  |
| credentialsRotationPolicy.enabled           | `boolean`  | Indicates whether automatic credentials rotation are enabled.                                                                                                                                                                                                                                                                                                        |
| credentialsRotationPolicy.rotationFrequency | `duration`  | Specifies the frequency at which the binding rotation is performed.                                                                                                                                                                                                                                                                                                  |
| credentialsRotationPolicy.rotatedBindingTTL | `duration`  | Specifies the time period for which to keep the rotated binding.                                                                                                                                                                                                                                                                                                     |
| SecretTemplate                              | `string`  | A Go template used to generate a custom Kubernetes v1/Secret, working on both the access credentials returned by the broker and instance attributes. Refer to [Go Templates](https://pkg.go.dev/text/template) for more details.                                                                                                                                     



#### Status
| Parameter         | Type     | Description                                                                                                   |
|:-----------------|:---------|:-----------------------------------------------------------------------------------------------------------|
| instanceID   |  `string`  | The ID of the bound instance in the SAP Service Manager service. |
| bindingID   |  `string`  | The service binding ID in SAP Service Manager service. |
| operationURL |`string`| The URL of the current operation performed on the service binding. |
| operationType| `string `| The type of the current operation. Possible values are CREATE, UPDATE, or DELETE. |
| conditions| `[]condition` | An array of conditions describing the status of the service instance.<br/>The possible conditions types are <br/>- `Ready`: set to `true` if the binding is ready and usable<br/>- `Failed`: set to `true` when an operation on the service binding fails.<br/> In the case of failure, the details about the error are available in the condition message.<br>- `Succeeded`: set to `true` when an operation on the service binding succeeded. In case of a `false` operation considered as in progress unless a `Failed` condition exists.
| lastCredentialsRotationTime| `time` | Indicates the last time the binding secret was rotated.

[Back to top](#sap-business-technology-platform-sap-btp-service-operator-for-kubernetes)

## Uninstalling the Operator

Before you uninstall the operator, we recommend you manually delete all associated service instances and bindings. This way, you'll ensure all data stored with service instances and bindings are properly taken care of. Instances and bindings that were not manually deleted will be automatically deleted once you start the uninstallation process.

To uninstall the operator, run the following command:
`helm uninstall <release name> -n <name space>`

Example: 

 >   ```
  >   helm uninstall sap-btp-operator -n sap-btp-operator

#### Responses

   - `release <release name> uninstalled` - The operator has been successfully uninstalled

   - `Timed out waiting for condition` 
   
      - What happened?
      
        The deletion of instances and bindings takes more than 5 minutes, this happens when there is a large number of instances and bindings.

      - What to do:
     
        Wait for the job to finish and re-trigger the uninstall process.
        To check the job status, run `kubectl get jobs --namespace=<name space>` or log on to the cluster and check the job log.
        Note that you may have to repeat this step several times untill the un-install process has been successfully completed.
     
     
   - `job failed: BackoffLimitExceeded`
      
     -  What happened?
      
        One of the service instances or bindings could not be deleted.
     
      - What to do:
      
        First, locate the service instance or binding in question and fix it, then re-trigger the uninstallation. 

        To find it, log on to the cluster and check the pre-delete job, or check the logs by running the following two commands:
        
          - `kubectl get pods --all-namespaces| grep pre-delete`  - which gives you the list of all namespaces and jobs
          - `kubectl logs <job_name> --namespace=<name_space_name>` - where you specify the desired job and namespace
          
        Note that the pre-delete job is only visible for approximately one minute after the job execution is completed. 
  
        If you don't have access to the pre-delete job, use kubectl to view details about the failed resource and check its status by running:
        
          - `kubectl describe <resource_type> <resource_name>` 
          
        Check for resources with the deletion timestamp to determine if it tried to be deleted. 


## Contributions
We currently do not accept community contributions. 

## License
This project is licensed under Apache 2.0 unless noted otherwise in the [license](./LICENSE) file.

[Back to top](#sap-business-technology-platform-sap-btp-service-operator-for-kubernetes)

## Troubleshooting and Support

### Cannot create a binding because service instance is in `Delete Failed` state

The deletion of my service instance failed. To fix the failure, I have to create a service binding, but I can't do that because the instance is in the `Delete  Failed` state.

**Solution**

Use the `force_k8s_binding` query param when creating the service binding and set it to `true` (`force_k8s_binding=true`).You can do & this   either with the Service Manager Control CLI (smctl) [bind](https://help.sap.com/docs/SERVICEMANAGEMENT/09cc82baadc542a688176dce601398de/f53ff2634e0a46d6bfc72ec075418dcd.html) command.

**Note:** <br>
Do not use the service-operator-access plan credentials to run this command.

smctl Example

>   ```bash
  >   smctl bind INSTANCE_NAME BINDING_NAME --param force_k8s_binding=true
  >   ```


<br>
  Once you've finished working on the service instance, delete it by running the following command:


>   ```bash
  >   smctl unbind INSTANCE_NAME BINDING_NAME --param force_k8s_binding=true
  >   ```
**Note:** `force_k8s_binding` is supported only for the Kubernetes instances that are in the `Delete Failed` state.<br>


### Cluster is unavailable, and I still have service instances and bindings

I cannot delete service instances and bindings because the cluster in which they were created is no longer available.

**Solution**

Use a dedicated Service Manager API to clean up cluster content.
Access the API with the subaccount-admin plan.<br>For more information, see [Technical Access](https://help.sap.com/docs/service-manager/sap-service-manager/sap-service-manager-broker-plans).

**Note:** <br>
Do not call this API with the service-operator-access plan credentials. 

###

#### Request

`DELETE /v1/platforms/{platformID}/clusters/{clusterID}`


#### Parameters
| Parameter                                   | Type       | Description                                                                               |
|:--------------------------------------------|:-----------|:------------------------------------------------------------------------------------------|
| platformID                                  | `string`   | The ID of the platform (should be the `service-operator-access` instance ID)             |
| clusterID                                   | `string`   | The ID of the cluster. You should specify the ID from step 4 of the [Setup](#setup) section. If you are unable to retrieve it, use the GET serrvice instance or binding API or equivalent btp CLI command and extract it from the response.                 |

#### Response
##### 
| Status Code            | Description                                                                                                                                                                                                                                                                                                  |
|:-----------------------|:-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 202 Accepted           | The request has been accepted for processing.  <br>  <br> **Headers:** <br> 'Location'- A path to the operation status. For more information about operations, see:  [Service Manager operation API](https://api.sap.com/api/APIServiceManager/path/getSingleOperation). |
| 404 Resource Not Found | Platform or cluster not found                                                                                                                                                                                                                                                                                |
| 429 Too Many Requests  | When the rate limit is exceeded, the client receives the HTTP 429 "Too Many Requests" response status code. <br>  <br> **Headers:** <br>  'Retry-After'-  indicates the time in seconds after which the client can retry the request.                                                                        |
                                
<b>Attention: **Use this option only for cleanup purposes for a no longer available cluster.** Applying it to an active and available cluster may result in unintended resource leftovers in your cluster.</b>

### I can see my service instance/binding in SAP BTP, but not its corresponding custom resource in my cluster. How can I restore the custom resource?


Let's break down how to recover your Kubernetes custom resource that exists in SAP BTP but not in your Kubernetes cluster:

#### Background

If a Kubernetes custom resource (CR) representing an SAP BTP service instance or service binding is lost, you can restore the connection to the existing BTP resource by manually recreating the CR using the information from the BTP side. To successfully re-establish the link, the new CR must have the same name, reside in the same namespace, and be associated with the same cluster ID as the original. The SAP BTP resource itself holds the configuration and remains unchanged, so as long as these identifying attributes match, creating a new CR will not result in provisioning a new BTP resource, but will instead reconnect to the existing one.

#### Steps

1. Retrieve CR Details:

a) Access the service instance or binding representing your CR in SAP BTP.

b) Obtain the following details from the service instance:

- The name of the custom resource.
- The Kubernetes namespace where the CR should reside.
  
2. Recreate the CR:

a) If you have a YAML definition or manifest for your CR, ensure it includes the exact name and namespace you retrieved from the SAP BTP service instance or binding. 

b) Use `kubectl apply -f <your_cr_manifest.yaml>` to create the CR in your Kubernetes cluster.

3. Verify:

a) Use `kubectl get <your_cr_kind> <your_cr_name> -n <your_namespace>` to verify that the CR is successfully created in Kubernetes.

b) Check the service instance or binding in SAP BTP to confirm it now recognizes the re-established connection with the CR in Kubernetes.

c) If the connection is not re-established, verify that the cluster ID in your Kubernetes cluster matches the one associated with the SAP BTP service instance or binding. You can find the cluster ID in the context details visible in the cockpit or BTP CLI. If the IDs don't match, reconfigure your cluster with the correct ID.


You're welcome to raise issues related to feature requests, or bugs, or give us general feedback on this project's GitHub Issues page.
The SAP BTP service operator project maintainers will respond to the best of their abilities.

 

[Back to top](#sap-business-technology-platform-sap-btp-service-operator-for-kubernetes)


