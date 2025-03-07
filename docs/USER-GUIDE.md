# User Guide
  - [1. To start using Narrows](#1-to-start-using-narrows)
    - [1.1 With portal (recommended)](#11-with-portal--recommended-)
      - [1.1.1 Configurations](#111-configurations)
        - [Secret](#secret)
        - [Setting](#setting)
          - [Required fields](#required-fields)
          - [Known registry](#known-registry)
          - [Cache](#cache)
        - [Policy](#policy)
          - [Inspection setting](#inspection-setting)
          - [Inspection standard setting](#inspection-standard-setting)
          - [Inspection result setting](#inspection-result-setting)
      - [1.1.2 Assessments](#112-assessments)
        - [Image scanning reports](#image-scanning-reports)
        - [Kubebench scanning reports](#kubebench-scanning-reports)
        - [Risk scanning reports](#risk-scanning-reports)
      - [1.1.3 Insight](#113-insight)
        - [Cluster](#cluster)
        - [Namespace](#namespace)
        - [Workload](#workload)
    - [1.2 With CLI](#12-with-cli)
      - [1.2.1 Settings](#121-settings)
      - [1.2.2 Policy](#122-policy)
        - [1.2.2.1 Actions](#1221-actions)
      - [1.2.3 Assessment report](#123-assessment-report)
  - [2. Troubleshooting](#2-troubleshooting)
    - [2.1 With portal](#21-build-portal-from-scratch)
    - [2.2 Cannot find report on open search](#22-cannot-find-the-report-on-opensearch)
----

## 1. To start using Narrows
### 1.1 With portal (recommended)
#### 1.1.1 Configurations

Currently, we need one Secret, one setting and one Policy to be configured, then Narrows can work properly.

###### Secret
Create a secret for Harbor so that Narrows can talk to Harbor with the credentials.

For release 0.3, Harbor is mandatory for the image scanner and risk scanner, Narrows depends on the vulnerability report
of the images.

The secret you created will be referred by the setting. If you have already had the K8s secret, you can skip creating one and move forward to next step.
<img src="./pictures/create-secret.png">

| Field Name    | Type   | Description            |
|---------------|--------|------------------------|
| accessKey     | string | The username of Harbor |
| accessSecret  | string | The password of Harbor |
| Name          | string | The secret's name      |
| Namespace     | string | The secret's namespace |

##### Setting
Click on "Credential Name" and choose the secret you've created in the previous step. 
You also need to specify the endpoint of Harbor and schedule for Harbor for updating the CVE reports.

<img src="./pictures/create-setting.png">

###### Required Fields

| Field Name           | Field Type  | Description                                                                       |
|----------------------|-------------|-----------------------------------------------------------------------------------|
| Setting Name         | string      | The name of the data source                                                       |
| Credential Name      | string      | The name of the credential of data source                                         |
| Credential Namespace | string      | The namespace of the credential of data source                                    |
| Provider             | string      | The provider of the data source, currently only support Harbor                    |
| Name                 | string      | The name of the datasource setting, auto-generated and can't be modified          |
| Endpoint             | URL(string) | The endpoint of the data source                                                   |
| Schedule             | string      | The cron expression to be configured in Harbor to specify the CVE update interval |
| skipTLSVerify        | boolean     | Whether need to skip the TLS verify                                               |


###### Known Registry
This field is optional. If it is configured, replicate rules are set up in Harbor and images from 
the known registries can be automatically replicated to Harbor.

Currently, Harbor support 11 different image registries.
Navigate to this [page](https://goharbor.io/docs/2.5.0/administration/configuring-replication/create-replication-endpoints/) to check the details.

<img src="./pictures/setting-known-registry.png">

| Field Name           | Field Type  | Description                                      |
|----------------------|-------------|--------------------------------------------------|
| provider             | string      | The provider of the registry                     |
| name                 | string      | The name of the registry                         |
| endpoint             | URL(string) | The endpoint of the registry                     |
| Credential Name      | string      | The name of the credential of KnownRegistry      |
| Credential Namespace | string      | The namespace of the credential of KnownRegistry |
| skipTLSVerify        | boolean     | Whether need to skip the TLS verify              |

###### Cache
This filed is optional. If it is configured. Narrows can use the external Redis you specified here to cache the security data from Harbor.

<img src="./pictures/setting-cache.png">

| Field Name    | Field Type  | Description                                                                                              |
|---------------|-------------|----------------------------------------------------------------------------------------------------------|
| address       | URL(string) | The URL of the redis address                                                                             |
| livingTime    | int64       | LivingTime (seconds) specifies the living time of the cache data                                         |
| skipTLSVerify | boolean     | SkipTLSVerify indicates whether skip the TLS verification. Only active when 'redis' scheme is configured |

#### Policy
Once the Setting is done, you can create an inspection policy according to your security requirements.

By defining the inspection policy, users can:
1. Define the schedule for scanning.
2. Choose which scanners are included.
3. Define the baseline of the image scanner, which means that CVE above which baseline should be reported in image scanning reports.
4. Choose the namespace or the workloads to be scanned by the label selector.
5. Define the reports should be exported to which consumers, currently support OpenSearch and ElasticSearch.

We don't purge the obsolete data in release 0.3, users need to manager the lifecycle of the data in the consumers.


<img src="./pictures/policy-inspection-setting.png">

###### Inspection Setting
| Field Name      | Field Type        | Description                                                                                  |
|-----------------|-------------------|----------------------------------------------------------------------------------------------|
| Policy Name     | string            | The name of this policy                                                                      |
| Work Namespace  | string            | The namespace for the scanners                                                               |
| Schedule        | string            | Configure schedule of the Conjobs-typed scanners                                             |
| Scanners        | multiple configs  | The scanners to be included in the policy                                                    |
| ImagePullPolicy | corev1.PullPolicy | Image pull policy. Choose from Always, IfNotPresent and Never                                |
| Settings Name   | string            | Select the existing setting's name                                                           |
| OpenSearch      | multiple configs  | Enable this to leverage OpenSearch/ElasticSearch to store the time-series assessment reports |

###### Inspection standard setting
<img src="./pictures/policy-label-selector.png">

| Field Name                | Field Type | Description                                                                          |
|---------------------------|------------|--------------------------------------------------------------------------------------|
| Baseline                  | string     | The CVE issues above which baseline should be included in the image scanning reports |
| Namespace Labels Selector | map        | The label of the to-be-scanned namespaces                                            |
| Workload Labels Selector  | map        | The label of the to-be-scanned workloads                                             |

The action kind is about what to do if Narrows find vulnerable workloads, currently the only option is to quarantine the workload.
Narrows will add tags for the vulnerable workloads, and create a network policy to block the ingress and egress of the workload.

###### Inspection result setting
When the vulnerabilities are resolved, the labels will be revoked so that the network policy will not take effect on the workload.

| Field Name      | Field Type | Description                                                                       |
|-----------------|------------|-----------------------------------------------------------------------------------|
| Kind            | string     | Kind of action. Now the only choice is quarantine_vulnerable_workload             |

After the inspection policy created, Narrows Manager will start provision the scanners.

You can navigate to `Assessments` to check the security posture and risk trends of the cluster.

#### 1.1.2 Assessments

We can navigate to `Assessments` to check the security posture and risk trends of the cluster.

We can view the reports of the scanners.

##### Image scanning reports
A time-series chart will be generated for the image scanning reports, such as:

<img src="./pictures/report-harbor-scanner.png">

By clicking the name of each report, you can view the details of the report.
<img src="./pictures/report-harbor-detail.png">

If there is no CVE items whose severity is higher than the baseline you configured, then
the report will show the number of vulnerable containers is zero. But you can still review
all the CVE items equal or below the baseline in the `Details` section of the report.

##### Kubebench scanning reports
The Kubebench scanner is a set of daemon pods, each node will have a Kubebench scanner.
It helps to find the security issue in the K8s configuration files of each node.

<img src="./pictures/report-kubebench-scanner.png">

The checks lie in 5 different categories:
1. Worker node security config
2. Kubernetes policies
3. Control plane security config
4. Control plane config
5. Etcd node config

Each category also contains several sub-categories, example:
<img src="./pictures/report-kubebench-detail.png">

Each sub-category contains the actual security check items:
<img src="./pictures/report-kubebench-detail-more.png">

The checks are based on the [CIS Kubernetes Benchmark support](https://www.cisecurity.org/benchmark/kubernetes)

The Kubebench scanner is based on this [opensource implementation](https://github.com/aquasecurity/kube-bench),
we integrate this scanner into Narrows. The config files are mounted from
the host to the Kubebench scanner's cronjob pod, then those configuration files are checked.

When a Kubebench pod is up, the check will be conducted once. After that, the pod will start to watch the config
files we mounted to the pod. If there is any modification event against the files, the check will be trigger for
one more time.

The Kubebench scanner can detect the K8s verison and choose the corresponding baseline file.
Check all the supported baseline files [here](https://github.com/vmware-tanzu/cloud-native-security-inspector/tree/main/src/cmd/kubebench/cfg)

##### Risk scanning reports
This report lists all the pods along with pod name, namespace under which it is running,
as well as the number of vulnerabilities contained in the pod.

<img src="./pictures/report-risk-detail.png">

In release 0.3 the risk scanner will report 3 kinds of risks:
###### serious vulnerabilities
The report includes the vulnerabilities at least with medium severity. Risk scanner will give
a 3-scaled score for the risks. With Medium as 1, High as 2 and Critical as 3.

The risk scanner will decode the CVE vector of each serious vulnerabilities and find below
2 kinds of further risks.

###### network exposure risks
When the vulnerability's [CVSS v3.1 vector](https://www.first.org/cvss/specification-document) indicates that
this vulnerability could bring more risks when the workload is exposed to the network,
the risk scanner will add a new vulnerability item to indicate that.
Example:
```
resource Pod:mariadb-6df76645bf-hvdbj:kube-fate:bade5b13-1538-43e8-8649-7a27e385d74f is
exposed to network while it has vulneratbility CVE-2016-20013(severity: 2) with network exposure
```
The score scale of this kind of risk is 5. With None as 0, Low as 2, Medium as 3, High as 4 and Critical as 5.

The scale for this kind of issue is larger than the `serious vulnerabilities` because the risks become higher
after network exposure.

###### privilege risks
When the vulnerability's CVSS v3.1 vector indicates that this vulnerability could bring more risks 
when the workload has a lower required privilege to be accessed. Then the risk scanner will add a
new vulnerability item to report this potential risk.

The score scale for this kind of risk is the same with the `network exposure risks`.

<img src="./pictures/report-risk-scanner.png">
We can also check the time-series trend of the vulnerabilities in the container.

#### 1.1.3 Insight
In release 0.3, insight is all about the image scanning results, Kubebench scanning results and
risk scanning results are not included in the Insight section.

The security administrator can examine the CVE issues in 3 perspectives:

* Cluster
* Namespace
* Workload

##### Cluster
You can check the total number of workloads that passed or violates the baseline within the cluster CNSI was installed.
<img src="./pictures/insight-cluster-summary.png">
You can check the distribution of the vulnerable workloads across the namespaces within the cluster in this perspective.
<img src="./pictures/insight-cluster-violations.png">

##### Namespace
You can check the total number of workloads that passed or violates the baseline within a certain namespace.
<img src="./pictures/insight-namespace-summary.png">
You can check the distribution of the vulnerable workloads across different types of workload within the namespace.
<img src="./pictures/insight-namespace-violations.png">

##### Workload
You can also directly check all the workloads scanned by image scanner, and check their results:
<img src="./pictures/insight-workloads.png">

### 1.2 With CLI
#### 1.2.1 Settings
Firstly you need to create the settings of Cloud Native Security Inspector.

Create yaml files such as below examples.
```yaml
apiVersion: v1
data:
  accessKey: YWRtaW4=
  accessSecret: SGFyYm9yMTIzNDU=
kind: Secret
metadata:
  name: harbor
  namespace: default
type: Opaque

---
apiVersion: goharbor.goharbor.io/v1alpha1
kind: Setting
metadata:
  name: sample-setting
spec:
  dataSource:
    credentialRef:
      name: harbor
      namespace: default
    provider: Harbor
    name: source-harbor
    endpoint: https://demo.goharbor.io
    scanSchedule: "0 0 0 * * *"
    skipTLSVerify: false

  knownRegistries:
  - provider: docker-registry
    name: the-dockerhub
    endpoint: docker.io
    credentialRef:
      name: dockerhub-secret
      namespace: default
    skipTLSVerify: true
```

You should define an Opaque secret that has accessKey and accessSecret fields in the
data property, the value of accessKey is the base64 encoded harbor username and the
value of accessSecret is the base64 encoded harbor password.

knowRegistries field is optional when your cluster workloads images are managed
in the data source harbor, otherwise, you need to configure your private
registries here used for harbor replication. You can update the yaml file like above to 
define the dockerhub as the data source:

Using below command to check the status of the setting, make sure the "STATUS" is "Healthy"

```shell
$ kubectl get settings -A

NAME             ENDPOINT                STATUS    AGE
sample-setting   https://10.78.177.224   Healthy   97s
```

#### 1.2.2 Policy

Then users need to create inspection policies to define their security requirements, example yaml:

```yaml
apiVersion: goharbor.goharbor.io/v1alpha1
kind: InspectionPolicy
metadata:
  name: inspectionpolicy-sample
spec:
  settingsName: sample-setting
  enabled: true
  workNamespace: narrows-workspace
  schedule: "*/5 * * * *"
  strategy:
    historyLimit: 4
    suspend: false
    concurrencyRule: "Forbid"
  inspector:
    image: projects.registry.vmware.com/cnsi/inspector:0.3
    kubebenchImage: projects.registry.vmware.com/cnsi/kubebench:0.3
    riskImage: projects.registry.vmware.com/cnsi/risk:0.3
    workloadScanner: projects.registry.vmware.com/cnsi/workloadscanner:0.3
    imagePullPolicy: IfNotPresent
    exportConfig:
      openSearch:
        hostport: "https://opensearch-cluster-master.opensearch:9200"
        username: "admin"
        password: "admin"
        checkCert: false
        mutualTLS: false
  inspection:
    namespaceSelector:
      matchLabels:
        kubernetes.io/metadata.name: prometheus
    baselines:
      - kind: "vulnerability"
        baseline: "High"
        version: "v1.1"
        scheme: "application/vnd.security.vulnerability.report; version=1.1"
    actions:
      - kind: "quarantine_vulnerable_workload"
```

After the policy is created, scanners will be provisioned to scan the workloads, based
on the given label selector defined in the policy.

##### 1.2.2.1 Actions
If the `actions` field is not specified, no action will be performed on violation workloads. 
The following actions are supported by policies. 

###### quarantine_vulnerable_workload 

For now, Narrows only supports `quarantine_vulnerable_workload` type action. 
  
When violation is detected on workloads, Narrows will create a network
policy with "deny-all" rules for the namespace if not exist. The violation pods will be
labeled `goharbor.io/controller: "CNSI"` and `goharbor.io/inspection: "risk'` to match
the pod selector on network policy and the pods will be quarantined.

When the violation pods turns good, the labels will be removed from the pods. And the
quarantine will be revoked.

###### TBD
More actions will be supported in the future.

## 2. Troubleshooting
### 2.1 Build portal from scratch
If you encounter the problem of installing npm dependencies during the portal building process,
please modify the npm reference source in Dockerfile.portal

  <img src="./pictures/npm-error.png" />

```shell
$ npm --registry https://registry.npm.taobao.org install
```

### 2.2 Cannot find the report on OpenSearch
Check your OpenSearch is deployed in which namespace, and make sure the OpenSearch endpoint
is correctly configured in the policy.
