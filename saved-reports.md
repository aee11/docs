# Saved Reports Documentation

### Summary

Kubecost supports configuring saved report parameters through [`values.yaml`](https://github.com/kubecost/cost-analyzer-helm-chart/blob/master/cost-analyzer/values.yaml), allowing users to configure multiple saved custom reports on start up in addition to configuration through the UI. This reference outlines the process of configuring saved reports through `values.yaml` and provides documentation on the required parameters.
  
### Saved Report Parameters  
  
The saved report settings, under `global.savedReports`, accept two parameters:

* `enabled` determines whether Kubecost will read saved reports configured via `values.yaml`; default `false`   
* `reports` is a list of report parameter maps
    
The following fields apply to each map item under the `reports` key:

* `title` -- title of your custom report; any non-empty string is accepted
* `window` -- the time window the allocation report covers, supports:
	* `today`
	* `yesterday`
	* `week` -- Week-to-date
	* `month` -- Month-to-date
	* `<N>d` -- last **N+1** days (e.g., `29d` spans the last 30 days)
	* `<start-date>,<end-date>` -- **custom window**, consisting of  two comma-separated ISO 8601 date strings:
		* `<start-date>` -- `<YYYY-MM-DD>T00:00:00Z`, start date in UTC; time of day is **rounded down** to the nearest day
		* `<end-date>` -- `<YYYY-MM-DD>T23:59:59Z`, end date in UTC; time of day is **rounded up** to the nearest day
		* Example custom window that spans *2020-11-11* to *2020-12-10*:
			* `2020-11-11T00:00:00Z,2020-12-09T23:59:59Z`
			* Note that `2020-12-09` rounds up cover the window up until *2020-12-10*, meaning that the parameter above is semantically equivalent to `2020-11-11T12:58:21Z,2020-12-09T00:30:24Z` in the Allocation API

* `aggregateBy` -- aggregation parameter -- equivalent to *Breakdown* in the Kubecost UI, supports `cluster`, `node`, `namespace`, `controllerKind`, `controller`, `service`, `department`, `environment`, `owner`, and `product`
* `idle` -- idle costs, supports `hide`, `share`, and `separate`
* `accumulate` -- determines whether or not to sum Allocation costs across the entire window -- equivalent to *Resolution* in the UI, supports `true` (Entire window resolution) and `false` (Daily resolution)
* `filters` -- a list of maps consisting of a property and value
	* `property` -- supports `cluster`, `node`, `namespace`, and `label`
	* `value` -- property value(s) to filter on, supports wildcard filtering with a `*` suffix
		* Special case `label` `value` examples: `app:cost-analyzer`, `app:cost*`
			* Wildcard filters only apply for the label value. e.g., `ap*:cost-analyzer` is not valid
	* *Note: multiple filter properties evaluate as ANDs, multiple filter values evaluate as ORs*
		* *e.g., (namespace=foo,bar), (node=fizz) evaluates as (namespace == foo || namespace == bar) && node=fizz*
	* **Important:** If no filters used, supply an empty list `[]`

### Example Helm values.yaml Saved Reports Section

*Note: `values.yaml` will overwrite any previously saved reports. Any reports saved afterward through the UI will also be overwritten on restart.*

```
   # Set saved report(s) accessible in reports.html
   # View configured saved reports in <front-end-url>/model/reports
  savedReports:
    enabled: true # If true, overwrites report parameters set through UI
    reports:
      - title: "Example Saved Report 0"
        window: "today"
        aggregateBy: "namespace"
        idle: "separate"
        accumulate: false # daily resolution
        filters:
          - property: "cluster"
            value: "cluster-one,cluster*" # supports wildcard filtering and multiple comma separated values
          - property: "namespace"
            value: "kubecost"
      - title: "Example Saved Report 1"
        window: "month"
        aggregateBy: "controllerKind"
        idle: "share"
        accumulate: false
        filters:
          - property: "label"
            value: "app:cost*,environment:kube*"
          - property: "namespace"
            value: "kubecost"
      - title: "Example Saved Report 2"
        window: "2020-11-11T00:00:00Z,2020-12-09T23:59:59Z"
        aggregateBy: "service"
        idle: "hide"
        accumulate: true # entire window resolution
        filters: [] # if no filters, specify empty array

```

### Troubleshooting

Review these steps to verify that saved reports are being passed to the Kubecost application correctly:

-   Ensure that the Helm values are successfully read into the configmap
-   Check that `global.savedReports.enabled` is set to `true`
-   Run `helm template ./cost-analyzer -n kubecost > test-saved-reports-config.yaml`
-   Open `test-saved-reports-config`
-   Find the section starting with `# Source: cost-analyzer/templates/cost-analyzer-saved-reports-configmap.yaml`
-   Ensure that the Helm values are successfully read into the configmap under the `data` field.
-   Example:
```
# Source: cost-analyzer/templates/cost-analyzer-saved-reports-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: saved-report-configs
  labels:
    
    app.kubernetes.io/name: cost-analyzer
    helm.sh/chart: cost-analyzer-1.70.0
    app.kubernetes.io/instance: RELEASE-NAME
    app.kubernetes.io/managed-by: Helm
    app: cost-analyzer
data:
  saved-reports.json: '[{"accumulate":false,"aggregateBy":"namespace","filters":[{"property":"cluster","value":"cluster-one,cluster*"},{"property":"namespace","value":"kubecost"}],"idle":"separate","title":"Example Saved Report 0","window":"today"},{"accumulate":false,"aggregateBy":"controllerKind","filters":[{"property":"label","value":"app:cost*,environment:kube*"},{"property":"namespace","value":"kubecost"}],"idle":"share","title":"Example Saved Report 1","window":"month"},{"accumulate":true,"aggregateBy":"service","filters":[],"idle":"hide","title":"Example Saved Report 2","window":"2020-11-11T00:00:00Z,2020-12-09T23:59:59Z"}]'# Source: cost-analyzer/templates/cost-analyzer-alerts-configmap.yaml
```

-   Ensure that the json string is successfully mapped to the appropriate configs
-   Navigate to `<front-end-url>/model/reports` and ensure that the configured report parameters have been set
