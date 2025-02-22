# Cloud Integrations

Integration with the Cloud Service Providers via their respective billing APIs allow Kubecost to display out-of-cluster costs, which are the costs incurred on a billing account from Services Outside of the cluster(s) where Kubecost is installed, in addition to the ability to reconcile Kubecosts in-cluster predictions with actual billing data to improve accuracy. For more details on these integrations continue reading below. For guides on how to set up these integrations follow the relevant link:
- [Multi-Cloud](https://github.com/kubecost/docs/blob/master/multi-cloud.md)
- [AWS](https://github.com/kubecost/docs/blob/master/aws-cloud-integrations.md)
- [GCP](https://cloud.google.com/billing/docs/how-to/export-data-bigquery)
- [Azure](https://docs.microsoft.com/en-us/azure/cost-management-billing/costs/tutorial-export-acm-data?tabs=azure-portal) 

## Cloud Processes
As indicated above, setting up a cloud integration with your Cloud Service Provider allows Kubecost to pull in additional billing data. The two processes that incorporate this information are Reconciliation and Cloud Assets.

### Reconciliation
Reconciliation matches in-cluster Assets with items found in the billing data pulled from the Cloud Service Provider. This allows Kubecost to display the most accurate depiction of your in-cluster spend. Additionally, the reconciliation process creates `Network` assets for in-cluster nodes based on the information in the billing data. The main drawback of this process is that the Cloud Service Providers have between a 6 to 24 hour delay in releasing billing data, and reconciliation requires a complete day of cost data to reconcile with the in-cluster assets. This requires a 48 hour window between resource usage and reconciliation. If reconciliation is performed within this window, asset cost is deflated to the partially complete cost shown in the billing data.

### Cloud Assets
The Cloud Assets process allows Kubecost to pull in out-of-cluster cloud spend from your Cloud Service Provider's billing data. This includes any services run by the Cloud Service Provider in addition to compute resources outside of clusters monitored by Kubecost. Additionally, by labeling these Cloud Assets their cost can be distributed to Allocations as external costs. This can help teams get a better understanding of the proportion of out-of-cluster cloud spend that their in-cluster usage is dependant on. Cloud Assets become available as soon as they appear in the billing data, with the 6 to 24 hour delay mentioned above, and are updated as they become more complete.

## Cloud Integration Configurations
The Kubecost helm chart provides values that can enable or disable each cloud process on the deployment once a cloud integration has been set up. Turning off either of these processes will disable all the benefits provided by them.

Value | Default | Description
--: | :--: | :--
`.Values.kubecostModel.etlAssetReconciliationEnabled` | true | Enables Reconciliation processes and endpoints. This Helm value corresponds to the `ETL_ASSET_RECONCILIATION_ENABLED` environment variable.
`.Values.kubecostModel.etlCloudAssets` | true | Enables Cloud Assets processes and endpoints. This Helm value corresponds to the `ETL_CLOUD_ASSETS_ENABLED` environment variable.
`.Values.kubecostModel.cloudAssetsExcludeProviderID` | false | **This is a BETA feature, support may be dropped in future releases.** Enabling this flag pre-aggregates cloud assets at the service and user label level for higher performance build speed and improved asset query times, at the cost of no longer being able to see the cost of individual cloud assets. This Helm value corresponds to the `CLOUD_ASSETS_EXCLUDE_PROVIDER_ID` environment variable. Currently only available on AWS. Users should ensure that `eks:cluster-name` is enabled as described in the [AWS OOC documentation](https://github.com/kubecost/docs/blob/master/aws-out-of-cluster.md#step-2-tag-your-resources) to prevent double counting of EKS resources.
`.Values.kubecostModel.etlUseUnblendedClost` | false | **This is a BETA feature, support may be dropped in future releases.** Enabling this flag makes Cloud Assets and Reconciliation use unblended cost for all line items in the CUR including those with savings plans and RI's applied to them. This will cause the amortized upfront costs of these resources to not appear in Kubecost and may cause some assets to have a $0 value if their cost was entirely upfont. This Helm value corresponds to the `ETL_USE_UNBLENDED_COST` environment variable. Currently only available on AWS.
## Cloud Stores
The ETL contains a Map of Cloud Stores, each of which represents an integration with a Cloud Service Provider. Each Cloud Store is responsible for the Cloud Asset and Reconciliation Pipelines which add Out-of-Cluster costs and Adjust Kubecost's estimated cost respectively via cost and usage data pulled from the Cloud Service Provider. Each Cloud Store has a unique identifier called the `ProviderKey` which varies depending on which Cloud Service Provider is being connected to and ensures that duplicate configurations are not introduced into the ETL. The value of the `ProviderKey` is the following for each Cloud Service Provider at a scope that the billing data is being for:

- AWS: Account Id
- GCP: Project Id
- Azure: Subscription Id

The `ProviderKey` can be used as an argument for the endpoints for Cloud Assets and Reconciliation, to indicate that the specified operation should only be done on a single Cloud Store rather than all of them, which is the default behavior. Additionally, the Cloud Store keeps track of the Status of the Cloud Connection Diagnostics for each of the Cloud Assets and Reconciliation. The Cloud Connection Status is meant to be used as a tool in determining the health of the Cloud Connection that is the basis of each Cloud Store. The Cloud Connection Status has various failure states that are meant to provide actionable information on how to get your Cloud Connection running properly. These are the Cloud Connection Statuses:

 - INITIAL_STATUS is the zero value of Cloud Connection Status and means that cloud connection is untested. Once
Cloud Connection Status has been changed and it should not return to this value. This status is assigned on creation
to the Cloud Store

- MISSING_CONFIGURATION means that Kubecost has not detected any method of Cloud Configuration. This value is only
possible on the first Cloud Store that is created as a wrapper for the open source cloud provider. This status
is assigned during failures in Configuration Retrieval.

- INCOMPLETE_CONFIGURATION means that Cloud Configuration is missing required values to connect to the cloud provider. This status is assigned during failures in Configuration Retrieval.

- FAILED_CONNECTION: means that all required Cloud Configuration values are filled in, but a connection with the
Cloud Provider cannot be established. This is indicative of a typo in one of the Cloud Configuration values or an
issue in how the connection was set up in the Cloud Provider's Console. The assignment of this status varies
between Providers but should happen if there if an error is thrown when an interaction with an object from
from the Cloud Service Provider's SDK occurs.

- MISSING_DATA:  means that the Cloud Integration is properly configured, but the cloud provider is not returning
billing/cost and usage data. This status is indicative of the billing/cost and usage data export of the Cloud Provider
being incorrectly set up or the export being set up in the last 48 hours and not having started populating data yet.
This status is set when a query has been successfully made but the results come back empty. If the cloud provider,
already has a SUCCESSFUL_CONNECTION status then this status should not be set, because this indicates that the specific query made may have been empty.

- SUCCESSFUL_CONNECTION: means that the Cloud Integration is properly configured and returning data. This status is set on any successful query where data is returned

After starting or restarting Cloud Assets or Reconciliation two subprocesses are started, one which fills in historic data over the coverage of the Daily Asset Store and one which runs periodically, on a predefined interval, to collect and process new cost and usage data as it is made available by the Cloud Service Provider. The ETL's status endpoint contains a `cloud` object that provides information about each Cloud Store including the Cloud Connection Status and diagnostic information about Cloud Assets and Reconciliation. The diagnostic items on the Cloud Assets and Reconciliation are:

- Coverage: The window of time that historical subprocess has covered
- LastRun: The last time that the process ran, updates each time the periodic subprocess runs
- NextRun: Next scheduled run of the periodic subprocess
- Progress: Ratio of Coverage to Total amount of time to be covered
- RefreshRate: The interval that the periodic subprocess runs
- Resolution: The size of the assets being retrieved
- StartTime: When the Cloud Process was started


## EndPoints

`http://<kubecost-address>/model/etl/asset/cloud/rebuild`

Description:

Completely restart Cloud Asset Pipeline. This operation ends the currently running Cloud Asset Pipeline and rebuilds historic Cloud Assets in the Daily Asset Store.

Example uses:

`http://localhost:9090/model/etl/asset/cloud/rebuild` // this will not run because it is missing the commit parameter

`http://localhost:9090/model/etl/asset/cloud/rebuild?commit=true`

`http://localhost:9090/model/etl/asset/cloud/rebuild?commit=true&provider=######-######-######`

API parameters include the following:

- `commit` is a boolean flag that acts as a safety precaution, these can be long-running processes so this endpoint should not be run arbitrarily. a `true` value restarts the process.

- `provider` is an optional parameter that takes the Provider Key described above. If included only the specified Cloud Store will run the operation, if it is not included all Cloud Stores in the ETL will run the operation.

---

`http://<kubecost-address>/model/etl/asset/cloud/repair`

Description:

Reruns queries for Cloud Assets in the given window for the given Cloud Store or all Cloud Stores if no provider is set.

Example uses:

`http://localhost:9090/model/etl/asset/cloud/repair`

`http://localhost:9090/model/etl/asset/cloud/repair?window=7d`

`http://localhost:9090/model/etl/asset/cloud/repair?window=yesterday&provider=######-######-######`

API parameters include the following:

- `window` dictates the applicable window for repair by the Cloud Store. Current support options:
"15m", "24h", "7d", "48h", etc.
"today", "yesterday", "week", "month", "lastweek", "lastmonth"
"1586822400,1586908800", etc. (start and end unix timestamps)
"2020-04-01T00:00:00Z,2020-04-03T00:00:00Z", etc. (start and end UTC RFC3339 pairs)

- `provider` an optional parameter that takes the Provider Key described above. If included only the specified Cloud Store will run the operation, if it is not included all Cloud Stores in the ETL will run the operation.

---

`http://<kubecost-address>/model/etl/asset/reconciliation/rerun`

Description:

Completely restart Reconciliation Pipeline. This operation ends the currently running Reconciliation Pipeline and reconciles historic Assets in the Daily Asset Store.

Example uses:

`http://localhost:9090/model/etl/asset/reconciliation/rerun` // this will not run because it is missing the commit parameter

`http://localhost:9090/model/etl/asset/reconciliation/rerun?commit=true`

`http://localhost:9090/model/etl/asset/cloud/reconciliation/rerun?commit=true&provider=######-######-######`

API parameters include the following:

- `commit` a boolean flag that acts as a safety precaution, these can be long running processes so this endpoint should not be run arbitrarily. a `true` value restarts the process.

- `provider` an optional parameter that takes the Provider Key described above. If included only the specified Cloud Store will run the operation, if it is not included all Cloud Stores in the ETL will run the operation.

---

`http://<kubecost-address>/model/etl/asset/reconciliation/repair`

Description:

Reruns queries for Reconciliation in the given window for the given Cloud Store or all Cloud Stores if no provider is set.

Example uses:

`http://localhost:9090/model/etl/asset/reconciliation/repair`

`http://localhost:9090/model/etl/asset/reconciliation/repair?window=7d`

`http://localhost:9090/model/etl/asset/reconciliation/repair?window=yesterday&provider=######-######-######`

API parameters include the following:

- `window` dictates the applicable window for repair by the Cloud Store. Current support options:
"15m", "24h", "7d", "48h", etc.
"today", "yesterday", "week", "month", "lastweek", "lastmonth"
"1586822400,1586908800", etc. (start and end unix timestamps)
"2020-04-01T00:00:00Z,2020-04-03T00:00:00Z", etc. (start and end UTC RFC3339 pairs)

- `provider` is an optional parameter that takes the Provider Key described above. If included only the specified Cloud Store will run the operation, if it is not included all Cloud Stores in the ETL will run the operation.

---

`http://<kubecost-address>/model/etl/status`

Description:

Returns a status object for the ETL. This includes sections for `allocation`, `assets`, and `cloud`.

Example uses:

`http://localhost:9090/model/etl/status`
