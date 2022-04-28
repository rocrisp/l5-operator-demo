#### Slide 1
Observability of Operator and the Operand

Metrics and Alert

By default, controller-runtime builds a global prometheus registry and publishes a colletion of performance metrics for each controller.

Custom Operator metrics
- operand upgrade counter
- operand upgrade failure state, 1 or 0

Operand metrics
- http://bestie-route-bestie.apps.rose.opdev.io/metrics

---
#### Slide 1.2

Store metrics ?
Prometheus is the answer.
Included in the Openshift platform
Prometheus Golang client 

#### Slide 2
Create a servicemonitor for the Operator ( we get this for free with OperatorSDK framwork ) in the same namespace.

oc get servicemonitor
NAME                                             AGE
l5-operator-controller-manager-metrics-monitor   ---

---
#### Slide 3

Create a ClusterRole, and a ClusterRoleBinding to bind the ServiceAccount, prometheus-k8s in the openshift-monitoring to the ClusterRole.

Clusterrole.yaml

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus-k8s-role 
rules:
  - apiGroups:
      - ""
    resources:
      - endpoints
      - pods
      - services
      - nodes
      - secrets
    verbs:
      - get
      - list
      - watch


ClusClusterRolebinding.yaml

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: prometheus-k8s-rolebinding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus-k8s-role
subjects:
  - kind: ServiceAccount
    name: prometheus-k8s
    namespace: openshift-monitoring
---

#### Slide 4

Custom metrics

1. Initialize
Best Practice Tip: We prefix the metric with the Operator-name_

var (
	applicationUpgradeCounter = prometheus.NewCounter(
		prometheus.CounterOpts{
			Name: "bestie_upgrade_counter",
			Help: "Number of successful bestie application upgrades processed",
		},
	)
)
2. Register
func init() {
	// Register custom metrics with the global prometheus registry
	metrics.Registry.MustRegister(applicationUpgradeCounter, applicationUpgradeFailure)
}

3. Create a metric counter
applicationUpgradeCounter.Inc()

#### Slide 5

Operand metrics

1. Bestie Application create /metrics path

2. Add a name to the service port, in this case port 80.
service.yaml

apiVersion: v1
kind: Service
metadata:
  labels:
    app: bestie
    name: bestie-service
  name: bestie-service
spec:
  ports:
  - protocol: TCP
    port: 80
    name: metrics
    targetPort: 8000
  selector:
    app: bestie
  type: LoadBalancer

3. Create a servicemonitor in the same namesapce where you create your CR.

bestie-operand-servicemonitor.yaml

apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: bestie-app-servicemonitor
  labels:
    name: bestie-app-servicemonitor
spec:
  endpoints:
    - path: /metrics
      port: metrics
      scheme: http
  selector:
    matchLabels:
      app: bestie

---

#### Slide 6
Alert

prometheusrule.yaml

apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: bestie-alert
spec:
  groups:
  - name: example
    rules:
    - alert: BestieImageFailureAlert
      expr: bestie_upgrade_failure{job="l5-operator-controller-manager-metrics-service"} == 1
      labels:
        severity: critical