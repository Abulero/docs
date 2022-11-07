---
title: "Private Cloud License Manager"
linktitle: "PCLM – License Manager"
url: /developerportal/deploy/private-cloud-license-manager/
description: "Describes setting up a license server and applying licenses to your Mendix app namespaces in private cloud"
weight: 47
tags: ["Deploy", "Private Cloud", "Licensing", "PCLM", "License Server", "License Bundle"]
---

## 1 Introduction

When deploying your Mendix app for production use, it needs to be licensed. This removes the restrictions which are placed on unlicensed apps. For more information, see [Licensing Mendix for Private Cloud](/developerportal/deploy/private-cloud/#licensing) in the *Private Cloud* documentation.

Apps which have access to the internet have licenses which work on a subscription basis and contact the Mendix license server for validating the license. This method is not available to apps which are not connected to the internet (air-gapped), which need to use static, offline, licenses.

The Mendix **Private Cloud License Manager** (PCLM) provides a repository of your offline Mendix licenses to enable you to manage these centrally, rather than needing to apply these manually to each app. This reduces the possibility of errors, and enables the production of license usage reports.

The PCLM runs as a Kubernetes service on your cluster. This means that it can be used by all your Mendix apps which run in namespaces within that cluster.

## 2 Prerequisites

To install and use the PCLM, you need the following prerequisites:

* a standalone Mendix for Private Cloud cluster
* administrative rights to a Kubernetes namespace to install PCLM server (a dedicated namespace is recommended). This can be within your Mendix for Private Cloud cluster, or in another cluster which is accessible over HTTP
* a Postgres or SQLServer database server and within it:
    * a dedicated database to store your licenses, user authorization details, and usage information
    * a dedicated administrator user role with all grants over this database (including the `Create Table` server role, and `Select`, `Insert`, `Update`, `Delete`, and `Truncate` database roles for the tables that are created)
* kubectl or the OpenShift CLI
* the mx-pclm-cli tool
* the default password for the PCLM server `administrator` account – you can obtain this from [Mendix Support](https://support.mendix.com)

## 3 Installing the PCLM server

You install the PCLM server by applying a manifest using `kubectl` or `oc`. This manifest can be created for you by the mx-pclm-cli tool.

### 3.1 Creating the Manifest

The `installer-gen` option of mx-pclm-cli is used to create a yaml manifest file to apply to your Kubernetes namespace.
Use the following command:

```bash {linenos=false}
mx-pclm-cli installer-gen --db-type <db-type> \
    --db-hostname <hostname> \ 
    --db-name <db-name> \
    --db-user <db-user> \
    --db-password <db-pass> \
    --db-port <port> \ 
    --db-strict-tls <tls-boolean> \
    --ssl-cert-file <ssl-root-certificate> \
    --env <environment-type> \
    --image-repo <docker-repo> \
    --image-tag <docker-tag> \
    --out-file <out-file>
```

Where you need to supply the following parameters

* `<db-type>` – the sort of database, either `postgres` *(default)* or `sqlserver`
* `<hostname>` – the hostname of the database service
* `<db-name>` – the name of the database where you want to hold the PCLM data
* `<db-user>` – a database user with the rights described in the prerequisites section
* `<db-pass>` – the password for the database user
* `<port>` – the port used to access the database, default: *5432*
* `<tls-boolean>` – whether the database uses strict TLS, `true` or `false` *(default)*
* `<ssl-root-certificate>` – the location of the SSL Root certificate file, if `<tls-boolean>` is `true`
* `<environment-type>` – the type of environment, `prod` *(default)* or `dev`
* `<docker-repo>` – location of the image repo, default: `private-cloud.registry.mendix.com/privatecloud-license-manager`
* `<docker-tag>` – the docker image tag, default: `master`
* `<out-file>` – the name of the file where the yaml is written, for example `manifest.yaml`

### 3.2 Applying the Manifest

To apply the installation manifest to install the server in the Kubernetes namespace, use the following command:

`$ kubectl -n <namespace> apply -f <out-file>`

Where you need to supply the following parameters

* `<namespace>` – the namespace in which the PCLM server will run
* `<out-file>` – the name of the file which contains the yaml manifest, for example `manifest.yaml`

This creates the following resources:

* `mendix-pclm` – the deployment CR of the server
* `mendix-pclm`  – a secret containing the credentials to access the Database
* `mx-private-cloud-mx-privatecloud-license-manager` – the service name of the PCLM server

## 4 Reaching the HTTP REST API of the PCLM Server

You will now be able to communicate with the PCLM Server through an HTTP REST endpoint. The endpoint will be different depending on whether you are using a Kubernetes Service or Kubernetes Ingress.

### 4.1 Using the Kubernetes Service

If PCLM is installed in the same Kubernetes cluster as the Private Cloud environments, the PCLM server can be reached at `http://mx-privatecloud-license-manager.<namespace>.svc.cluster.local`, where `<namespace>` is the namespace where the PCLM Server was installed.

You can confirm that you can connect to the PCLM server using the following URLs:

* `http://mx-privatecloud-license-manager.<namespace>.svc.cluster.local/health` should return `HTTP 200 OK`
* `http://mx-privatecloud-license-manager.<namespace>.svc.cluster.local/metrics` should return `HTTP 200 OK` together with the collected server metrics

### 4.2 Using Kubernetes Ingress

If the PCLM server is installed in a separate cluster, you have to create an [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) for your Kubernetes cluster or a [Route](https://docs.openshift.com/enterprise/3.0/architecture/core_concepts/routes.html) for your OpenShift cluster in order to allow those HTTP connections.

You must also define a host domain in the DNS service (for example, AWS Route53).

The configuration to connect to the server running in the namespace `my-pclm-1` will look similar to this:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: pclm-ingress
  namespace: my-pclm-1 
spec:
  rules:
    - host: pclm.<domain> # e.g. pclm.mydomain.io
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: mx-privatecloud-license-manager
                port:
                  number: 80
```

Given the many possible Ingress controllers which depend on the Cloud provider, you will need to follow the instructions for your Ingress controller and Cloud.

You can confirm that you can connect to the PCLM server using the following URLs:

* `http<s>://pclm.<domain>/health` should return `HTTP 200 OK`
* `https<s>://pclm.<domain>/metrics` should return `HTTP 200 OK` together with the collected server metrics

## 5 Setting Up Users

Once the PCLM server is running, you can set up users.
The PCLM server supports two **user types:**

* Admin – this user type has read-write permissions of users and licenses resources. It should only be used from the cli.
* Operator – this user type has only read-only permissions for license resources. This user is designed to allow the Mendix for Private Cloud Operator to obtain licenses.

### 5.1 Administrator

When the PCLM server is set up, it contains one user `administrator` with a default password. This password should be modified immediately using the command:

```bash {linenos=false}
mx-pclm-cli user update \
  -s <pclm-http-url> -u administrator -p <default-password> \
  --username administrator --password='<new-password>' --type admin
```

Where:

* `<pclm-http-url>` – is the HTTP REST endpoint of the PCLM server
* `<default-password>` – is the default password which is set for the `administrator` user – you can obtain this from [Mendix Support](https://support.mendix.com)
* `<new-password>` – is the new password for the `administrator` user

#### 5.1.1 Authenticating Using a Config File

To avoid supplying the `<admin-user>`, `<admin-password>`, and `<pclm-http-url>` on every call, these can be set up once in a config file. The PCLM command line, mx-pclm-cli, will then pick up these values and use them for all commands.

The default location for the config file is `$HOME/.pclm/config`. Alternatively, you can use the flag `-c` (`--config-file`) on the mx-pclm-cli commands to specify an alternative location for the config file.

The format of the config file is:

```yaml
admin_user: <admin-user>
admin_pass: <admin-password>
server_url: <pclm-http-url>
```

Where:

* `<admin-user>` – is a user of type *admin* which can update users, default: `administrator`
* `<admin-password>` – is the password for the chosen *admin* user
* `<pclm-http-url>` – is the HTTP REST endpoint of the PCLM server

### 5.2 Additional Users

You will want to set up *operator* users and (optionally) additional *admin* users. To do this, use the following command:

``` bash {linenos=false}
mx-pclm-cli user create \
  -s <pclm-http-url> -u <admin-user> -p <admin-password> \
  --username=<new-user>  --password='<password>' --type=<user-type>
```

Where:

* `<pclm-http-url>` – is the HTTP REST endpoint of the PCLM server (overrides the config file)
* `<admin-user>` – is a user of type *admin* which can update users, default: `administrator` (overrides the config file)
* `<admin-password>` – is the password for the chosen *admin* user (overrides the config file)
* `<new-user>` – is the name of the new user you are creating
* `<password>` – is the password for the new user
* `<user-type>` – is the type of user you are creating, either `admin` or `operator`

## 6 Installing Licenses

Licenses are supplied by Mendix as a **License Bundle**. A license bundle can contain both Mendix Runtime (app) licenses and Mendix Operator licenses.
Runtime licenses are required for each Mendix runtime environment, and Operator licenses are required for each namespace where the Operator runs.
To purchase a license bundle, please contact [Mendix Support](https://support.mendix.com/). You will receive your license(s) as a .zip file.
The following command will import a license bundle into the PCLM server:

```bash {linenos=false}
mx-pclm-cli license import  
    -s <pclm-http-url> \
    -u <admin-user> \
    -p <admin-password> \
    -f <bundle-zip-file-path>
```

Where:

* `<pclm-http-url>` – is the HTTP REST endpoint of the PCLM server (overrides the config file)
* `<admin-user>` – is a user of type *admin* which can update users, default: `administrator` (overrides the config file)
* `<admin-password>` – is the password for the chosen *admin* user (overrides the config file)
* `<bundle-zip-file-path>` – is the location of your license bundle file

You will get a report of the results of your import operation:

```text {linenos=false}
-- Loading zip file...
┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓
┃ License Bundle Contents                            ┃
┣━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━┫
┃ LICENSE-ID                           ┃ TYPE        ┃
┣━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━╋━━━━━━━━━━━━━┫
┃ c97ecdae-0376-42ab-9d91-22a45a88a3e4 ┃ mx-operator ┃
┣━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━╋━━━━━━━━━━━━━┫
┃ 5025defa-a442-47c3-ae2e-2ac6628926e3 ┃ mx-runtime  ┃
┃ c823eeb1-7eb2-471c-a818-7be132c9cdb1 ┃ mx-runtime  ┃
┗━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┻━━━━━━━━━━━━━┛
-- Importing License: 5025defa-a442-47c3-ae2e-2ac6628926e3 [Duplicated]
-- Importing License: c823eeb1-7eb2-471c-a818-7be132c9cdb1 [OK]
-- Importing License: c97ecdae-0376-42ab-9d91-22a45a88a3e4 [OK]
-- Licenses imported successfully
```

If a license has previously been imported, you will be told that it is `[Duplicated]`.

## 7 Applying Licenses to Your Operator and Apps

To use the licenses, you have to add information to the operator configuration. For this, you need to have set up the operator in a namespace on your cluster. See [Installing and Configuring the Mendix Operator](/developerportal/deploy/private-cloud-cluster/#install-operator) in the *Private Cloud Cluster* documentation.
Assume that the operator is running in the namespace `<operator-ns>`.

### 7.1 Storing Operator User Credentials

The credentials you have created for an operator type user need to be stored in the repository. To do this, create the following .yaml file:

```yaml
apiVersion: v1
kind: Secret
type: Opaque
metadata:
  name: <secret-name>
stringData:
  username: <operator-user>
  password: <operator-password>
```

Where:

* `<secret-name>` – is the name of the secret where you want to store the credentials
* `<operator-user>` – is a user of type *operator*
* `<operator-password>` – is the password for the chosen *operator* user

You store this secret using the following command:

```bash {linenos=false}
kubectl apply -n <operator-ns> <yaml-file>
```

Where:

* `<operator-ns>` – is the namespace where the Mendix Operator is running
* `<yaml-file>` – is the location of the file where you stored the instructions for creating the credentials secret

### 7.2 Configuring the Mendix Operator

You need to patch the Mendix Operator with the location and credentials for accessing the PCLM server. To do this, create the following .yaml file:

```yaml
spec:
  licenseManager:
    credentialsSecretName: <secret-name>
    serverURL: http://<pclm-http-url>
```

Where:

* `<secret-name>` – is the name of the secret where you stored the credentials
* `<pclm-http-url>` – is the HTTP REST endpoint of the PCLM server

Patch the Mendix Operator using the following command:

```bash {linenos=false}
kubectl -n <operator-ns> patch OperatorConfiguration mendix-operator-configuration  --type=merge --patch="$(cat <yaml-file>)"
```

Where:

* `<operator-ns>` – is the namespace where the Mendix Operator is running
* `<yaml-file>` – is the location of the file where you stored the instructions for patching the Mendix Operator

### 7.3 Applying Licenses

Once you have patched the Mendix Operator, all app environments which are controlled through the operator will have licenses applied automatically.

If you delete a licensed app environment, the mx-runtime license will be marked as unused and will be applied to new app environments. This may take up to 30 minutes.

If you do not have any unused licenses, new app environments will not be licensed.

The Mendix Operator will also pick up a Mendix Operator license if one has been imported into the PCLM server.

### 7.4 Verifying That the Licenses Are Applied

There are multiple ways to verify whether the licenses (both Operator and MendixApp) are applied and where.

#### 7.4.1 Using the PCLM CLI

You can see which licenses are currently used by which environments and operators, as well as unused licenses, using the following command.

```bash {linenos=false}
mx-pclm-cli license list-usage -s <pclm-http-url> \
    -u <admin-user> \
    -p <admin-password>
```

Where:

* `<pclm-http-url>` – is the HTTP REST endpoint of the PCLM server (overrides the config file)
* `<admin-user>` – is a user of type *admin* which can update users, default: `administrator` (overrides the config file)
* `<admin-password>` – is the password for the chosen *admin* user (overrides the config file)

Which would reply with something similar to this:

| LICENSE-ID | NAMESPACE | APP-ID | TYPE | CREATED-AT |
| --- | --- | --- | --- | --- |
| `<license-id>` | `<namepace>` | `<app-ID>` | mx-operator | yyyy-mm-dd hh:mm:ss |
| `<license-id>` | `<namepace>` | `<app-ID>` | mx-runtime  | yyyy-mm-dd hh:mm:ss |

#### 7.4.2 From Mendix Application Custom Resources Installed in the Namespace

This way of checking is more advanced, and should be used only for debugging.

##### 7.4.2.1 Checking the Operator License

Use the following command to verify whether the Operator license was applied correctly:

`<oc|kubectl> -n <namespace> get MendixApp <app_id> -o yaml`

In the section `status.licenseStatus` you should see something similar to the following:

```yaml {linenos=false}
status:
  licenseStatus:
    licenseID: 1ca080f8-c54e-4e24-b09c-14353505a65d
    mode: Licensed
```

##### 7.4.2.2 Checking the Runtime License

Use the following command to verify whether the Runtime license was applied correctly:

`<oc|kubectl> -n <namespace> get Runtime <app_id> -o yaml`

In the section `spec.resources.runtimeLicense` you should see something similar to the following:

```yaml {linenos=false}
spec:
  resources:
    runtimeLicense:
      id: c823eeb1-7eb2-471c-a818-7be132c9cdb1
      key: <base64-encoded-license-key>
      type: offline
```

### 7.5 Auditing Licenses

You can retrieve an audit report showing a history of which apps had which licenses, and which license(s) were applied to which operator(s) using the following command:

```bash {linenos=false}
mx-pclm-cli report generate \
    --num-days <days> \
    --zip-file <file-name> \
    -s <pclm-http-url> \
    -u <admin-user> \
    -p <admin-password>
```

Where:

* `<days>` – is the number of past days to report (default 365)
* `<file-name>` – is the name of the file where a zipped version of the report is saved — if this is omitted, the report will be output to the console
* `<pclm-http-url>` – is the HTTP REST endpoint of the PCLM server (overrides the config file)
* `<admin-user>` – is a user of type *admin* which can update users, default: `administrator` (overrides the config file)
* `<admin-password>` – is the password for the chosen *admin* user (overrides the config file)

The report is presented as a CSV file containing a summary of the licenses at various times over the specified period.

## 8. Migration

If you are migrating a Mendix Operator and Mendix apps which are already licensed to use the PCLM, you will need to remove the existing licenses. If your namespace was never licensed, please ignore this section.

Once you have configured the Mendix Operator running in a specific `<namespace>` to use the PCLM (following steps 7.1 and 7.2) you need to restart the Mendix operator to remove the existing licenses. You can do this as follows:

```bash {linenos=false}
kubectl -n <namespace> scale deployment mendix-operator --replicas=0
kubectl -n <namespace> scale deployment mendix-operator --replicas=1
```

This restarts the Mendix Operator, and associated Mendix App Runtimes, and configures both with licenses fetched from the PCLM server.

You can confirm this by running the following command:

```bash {linenos=false}
mx-pclm-cli license list-usage -s <pclm-http-url> \
    -u <admin-user> \
    -p <admin-password>
```

This will indicate that licenses have been applied to the operator and apps in the selected namespace:

| License-ID                           | Namespace  | App-ID   | Type        |
| ------------------------------------ | ---------- | -------- | ----------- |
| `<license-id>` | `<namepace>` | `<app-ID>` | mx-operator |
| `<license-id>` | `<namepace>` | `<app-ID>` | mx-runtime  |

## 9 Troubleshooting

### 9.1 mx-pclm-cli Help

You can get help on using mx-pclm-cli and its individual commands at any time by using the help command.
For example, `mx-pclm-cli --help` or `mx-pclm-cli user --help`.
