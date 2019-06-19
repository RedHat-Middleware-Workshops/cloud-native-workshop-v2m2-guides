## Lab3 - Application Monitoring

In the previous labs, you learned how to debug cloud-native apps to fix errors quickly using CodeReady Workspace 
with Quarkus framework, and you got a glimpse into the power of Quarkus for developer joy.

You will now begin observing applications in term of a distributed transaction, performance and latency because 
as cloud-native applications are developed quickly, a distributed architecture in production gets ultimately complex 
in two areas: networking and observability. Later on we'll explore how you can better manage and monitor the application using
service mesh.

In this lab, you will monitor coolstore applications using [Jaeger](https://www.jaegertracing.io/) and [Prometheus](https://prometheus.io/).

![logo]({% image_path quarkus-jaeger-prometheus.png %}){:width="900px"}

Jaeger is an open source distributed tracing tool for monitoring and troubleshooting microservices-based distributed systems, including:

 * Distributed context propagation
 * Distributed transaction monitoring
 * Root cause analysis
 * Service dependency analysis
 * Performance and latency optimization 

Also, Prometheus is an open source systems monitoring and alerting tool that fits in recording any numeric time series, including:

 * Multi-dimensional time series data by metric name and key/value pairs
 * No reliance on distributed storage
 * Time series collection over HTTP
 * Pushing time series is supported via an intermediary gateway
 * Service discovery or static configuration

####1. Create OpenShift Project

---

In this step, we will deploy our new monitoring tools for our CoolStore application,
so create a separate project to house it and keep it separate from our monolith and our other microservices we already
created previously.

Create a new project for the _Monitoring_ tools:

Click **Create Project**, fill in the fields, and click **Create**:

* Name: `userXX-monitoring`
* Display Name: `USERXX CoolStore App Monitoring Tools`
* Description: _leave this field empty_

![create_dialog]({% image_path create_monitoring_dialog.png %}){:width="500"}

Click on the name of the newly-created project:

![create_new]({% image_path create_new_monitoring.png %}){:width="500"}

This will take you to the project overview. There's nothing there yet, but that's about to change.

####2. Deploy Jaeger to OpenShift

---

This template uses an in-memory storage with a limited functionality for local testing and development. 
Do not use this template in production environments, although there are a number of parameters in the template to 
constrain the maximum number of traces and the amount of CPU/Memory consumed to prevent node instability.

Install everything in the current namespace via CodeReady Workspace **Terminal**:

`oc project userXX-monitoring`

`oc process -f https://raw.githubusercontent.com/RedHat-Middleware-Workshops/cloud-native-workshop-v2m2-labs/master/monitoring/jaeger-all-in-one-template.yml | oc create -f -`

You can also check if the deployment is complete via CodeReady Workspace **Terminal**:

`oc rollout status -w deployment.apps/jaeger`

> deployment "jaeger" successfully rolled out

When you navigate the **overview** page in OpenShift console, you will see as below:

![jaeger_deployments]({% image_path jaeger-deployment.png %})

####3. Exposing Jaeger-Collector

---

**Collector** is by default accessible only to services running inside the cluster. The easiest approach to expose the collector outside of 
the cluster is via the `jaeger-collector-http` HTTP port using an **OpenShift Route** via CodeReady Workspace **Terminal**:

`oc create route edge --service=jaeger-collector --port jaeger-collector-http --insecure-policy=Allow`

This allows clients to send data directly to Collector via HTTP senders. If you want to use the Agent then use ExternalIP or NodePort to expose the Collector service.

> **NOTE:** Using Collector will open the collector to be used by any external party, who will then be able to create arbitrary spans. It's advisable to put an OAuth Security Proxy in front of the collector and expose this proxy instead.

####4. Observe Jaeger UI

---

Once you deployed Jaeger to OpenShift, you will see the route that generated automatically.

![jaeger_route]({% image_path jaeger-route.png %})

Click on the route URL(i.e. https://jaeger-query-monitoring.apps.seoul-7b68.openshiftworkshop.com) then there is no **Service** and **Operation** at this moment.

![jaeger_ui]({% image_path jaeger-ui.png %})

> Don't worry! We will utilize tracing data later.

####5. Utilizing Opentracing with Inventory(Quarkus)

---

We have a catalog service on Spring Boot that calls inventory service on Quarkus as the cloud-native application. These applications would be 
better to trace using Jaeger rather than monolith coolstore for considering distributed networking system.

In this step, we will add Qurakus extensions to the Inventory application for using `smallrye-opentracing` and we'll use the Quarkus Maven Plugin.
Copy the following commands to add the tracing extension via CodeReady Workspaces **Terminal**:

Go to **inventory** directory:

`mvn quarkus:add-extension -Dextensions="io.quarkus:quarkus-smallrye-opentracing"`

If builds successfully (you will see `BUILD SUCCESS`), you will see `smallrye-opentracing` dependency in `pom.xml` automatically.

![jaeger_add_extension]({% image_path jaeger-extension.png %})

>**NOTE:** There are many [more extensions](https://quarkus.io/extensions/) for Quarkus for popular frameworks 
like [CodeReady Workspaces Vert.x](https://vertx.io/), [Apache Camel](http://camel.apache.org/), [Infinispan](http://infinispan.org/), 
Spring DI compatibility (e.g. @Autowired), and more.

####6. Create the configuration

---

Before getting started with this step, confirm your **route URL of Jaeger** and we will use the following step to create the tracing configuration.
You can find out the URL in OpenShift Console or use the `oc` command as here:

![jaeger_ui]({% image_path jaeger-collector-route.png %})

`oc get route`

~~~shell
NAME               HOST/PORT                                                                 PATH   SERVICES           PORT                    TERMINATION   WILDCARD
jaeger-collector   jaeger-collector-user1-monitoring.apps.seoul-7b68.openshiftworkshop.com          jaeger-collector   jaeger-collector-http   edge/Allow    None
jaeger-query       jaeger-query-user1-monitoring.apps.seoul-7b68.openshiftworkshop.com              jaeger-query       query-http              edge/Allow    None
~~~

The easiest way to configure the Jaeger tracer is to set up in the application(i.e. Inventory).

Open `src/main/resources/application.properties` file and add the following configuration via CodeReady Workspaces **Terminal**:

~~~java
# Jaeger configuration
quarkus.jaeger.service-name=inventory
quarkus.jaeger.sampler-type=const
quarkus.jaeger.sampler-param=1
quarkus.jaeger.endpoint=https://jaeger-collector-user1-monitoring.apps.seoul-7b68.openshiftworkshop.com/api/traces
~~~

> You should replace **quarkus.jaeger.endpoint** with your own route URL(**HTTPS**) as you created.

You can also specify the configuration using `jvm.args` that Jaeger supplys the properties as [environment variables](https://www.jaegertracing.io/docs/1.12/client-features/).

If the **quarkus.jaeger.service-name** property (or **JAEGER_SERVICE_NAME** environment variable) is not provided then a "no-op" tracer will be configured, 
resulting in no tracing data being reported to the backend.

Currently the tracer can only be configured to report spans directly to the collector via HTTP, using the `quarkus.jaeger.endpoint` property (or `JAEGER_ENDPOINT` environment variable). Support for using the Jaeger agent, via UDP, will be available in a future version.

>**NOTE:** there is no tracing specific code included in the application. By default, requests sent to this endpoint will be traced without any code changes being required. It is also possible to enhance the tracing information. For more information on this, please see the [MicroProfile OpenTracing specification](https://github.com/eclipse/microprofile-opentracing/blob/master/spec/src/main/asciidoc/microprofile-opentracing.asciidoc).

####7. Re-Deploy to OpenShift

---

> **NOTE**: Be sure to rollback Postgres database configuration as defined in `src/main/resources/application.properties`:
 
~~~java
quarkus.datasource.url=jdbc:postgresql:inventory
quarkus.datasource.driver=org.postgresql.Driver
~~~

Repackage the inventory application via clicking on `Package for OpenShift` in `Commands Palette`:

![codeready-workspace-maven]({% image_path quarkus-dev-run-packageforOcp.png %})

Next, update a temp directory to store only previously-built application with necessary lib directory via CodeReady Workspaces **Terminal**:

`rm -rf target/binary && mkdir -p target/binary && cp -r target/*runner.jar target/lib target/binary`

And then start and watch the build, which will take about a minute to complete:

`oc start-build inventory-quarkus --from-dir=target/binary --follow -n userXX-inventory`

You should see a **Push successful** at the end of the build output and it. To verify that deployment is started and completed automatically, 
run the following command via CodeReady Workspaces **Terminal** :

`oc rollout status -w dc/inventory-quarkus -n userXX-inventory`

And wait for the result as below:

`replication controller "inventory-quarkus-XX" successfully rolled out`

####8. Observing Jaeger Tracing

---

In order to trace networking and data transaction, we will call the Inventory service via `curl` commands via CodeReady Workspaces **Terminal**:

`curl http://inventory-quarkus-userXX-inventory.apps.seoul-7b68.openshiftworkshop.com/services/inventory/165613 ; echo`

Go to `Pods > inventory-quarkus-11-sfg4n > Logs` in OpenShift console, you will see that tracer is initialized after you call the Inventory service at the first time.

~~~shell
2019-06-13 08:59:28,731 INFO  [io.jae.Configuration] (executor-thread-1) Initialized tracer=JaegerTracer(version=Java-0.34.0, serviceName=inventory, reporter=RemoteReporter(sender=HttpSender(), closeEnqueueTimeout=1000), sampler=ConstSampler(decision=true, tags={sampler.type=const, sampler.param=true}), tags={hostname=inventory-quarkus-11-sfg4n, jaeger.version=Java-0.34.0, ip=10.1.16.21}, zipkinSharedRpcSpan=false, expandExceptionLogs=false, useTraceId128Bit=false)
~~~

![jaeger_ui]({% image_path jaeger-init.png %})

Now, reload the Jaeger UI then you will find that 2 services are created as here:

 * Inventory
 * Jaeger-query

Click on `Find Traces` and observe the first trace in the graph:

![jaeger_ui]({% image_path jaeger-reload.png %})

If you click on `Span` and you will see a logical unit of work in Jaeger that has an operation name, the start time of the operation, 
and the duration. Spans may be nested and ordered to model causal relationships:

![jaeger_ui]({% image_path jaeger-span.png %})

Let's make more traces!! Open a new web browser to access **CoolStore Inventory Pagre**(i.e. http://inventory-quarkus-inventory.apps.seoul-7b68.openshiftworkshop.com):

![jaeger_ui]({% image_path jaeger-coolstore.png %})

Go back to **Jaeger UI** then click on **Find Traces**. You will see dozens of traces because the Inventory page continues to calling the endpoint of Inventory service in every 2 seconds like we called via **curl** command:

![jaeger_ui]({% image_path jaeger-traces.png %})

####9. Deploy Prometheus and Grafana to OpenShift

---

OpenShift Container Platform ships with a pre-configured and self-updating monitoring stack that is based 
on the [Prometheus](https://prometheus.io) open source project and its wider eco-system. It provides monitoring 
of cluster components and ships with a set of alerts to immediately notify the cluster administrator about any occurring problems 
and a set of [Grafana](https://grafana.com/) dashboards.

![Prometheus]({% image_path monitoring-diagram.png %}){:width="800px"}

However, we will deploy custom **Prometheus** to scrape services metrics of Inventory and Catalog applications. 
Then we will visualize the metrics data via custom **Grafana** dashboards deployment.

Go to **Overview** page in `CoolStore App Monitoring Tools` project and click on `Deploy Image` under `Add to Project` menu:

![Prometheus]({% image_path add-to-project.png %})

Select **Image Name** and input `prom/prometheus` to search the Prometheus container image via clicking on **Search** icon.

![Prometheus]({% image_path search-prometheus-image.png %})

Once you find the image correctly as the above screenshot, click on **Deploy**.

![Prometheus]({% image_path prometheus-deploy-done.png %})

Create the route to access **Prometheus** web UI. Navigate **Application > Services** on the left menu and click on **Prometheus** service.
Next, you need to click on **Actions > Create Route**:

![Prometheus]({% image_path prometheus-create-route.png %})

 Click on **Create** with keeping all default variables:

![Prometheus]({% image_path prometheus-route-detail.png %})

Now, you have the route URL(i.e. `http://prometheus-user1-monitoring.apps.seoul-7b68.openshiftworkshop.com`) and click on the URL to make sure if you can access the Prometheus web UI:

![Prometheus]({% image_path prometheus-route-link.png %})

You will see the landing page of Prometheus as shown:

![Prometheus]({% image_path prometheus-webui.png %})

Let's deploy **Grafana Dashboards** to OpenShift. Go to **Overview** page in `CoolStore App Monitoring Tools` project 
and click on `Deploy Image` under `Add to Project` menu:

![Grafana]({% image_path add-to-project-grafana.png %})

Select **Image Name** and input `grafana/grafana` to search the Prometheus container image via clicking on **Search** icon.

![Grafana]({% image_path search-grafana-image.png %})

Once you find the image correctly as the above screenshot, click on **Deploy**.

![Grafana]({% image_path grafana-deploy-done.png %})

Create the route to access **Grafana** web UI. Navigate **Application > Services** on the left menu and click on **Prometheus** service.
Next, you need to click on **Actions > Create Route**:

![Grafana]({% image_path grafana-create-route.png %})

 Click on **Create** with keeping all default variables:

![Grafana]({% image_path grafana-route-detail.png %})

Now, you have the route URL(i.e. `http://grafana-user1-monitoring.apps.seoul-7b68.openshiftworkshop.com`) and 
click on the URL to make sure if you can access the Grafana web UI.

![Grafana]({% image_path grafana-route-link.png %})

You will see the landing page of Prometheus as shown:

![Grafana]({% image_path grafana-login.png %})

Log in Grafana web UI using the following variables:

* Username: admin
* Password: admin

Skip the **Change Password".

![Grafana]({% image_path grafana-skip-changepwd.png %})

You will see the landing page of Grafana as shown:

![Grafana]({% image_path grafana-webui.png %})

####10. Add a data source to Grafana

---

Before we create monitoring dashboard, we need to add a data source.
Go to the cog on the side menu that will show you the configuration menu. If the side menu is not visible click the Grafana icon in the upper left corner.

![Grafana]({% image_path grafana-sidemenu-datasource.png %}){:width="600px"}

Click on data sources of the configuration menu and you’ll be taken to the data sources page where you can add and edit data sources. 

![Grafana]({% image_path grafana-add-datasource.png %}){:width="700px"}

Click Add data source and select **Prometheus** as data source type.

![Grafana]({% image_path grafana-datasource-types.png %})

Next, input the following variables to configure the dashboard. Make sure to replace `HTTP URL` with your **Prometheus Route URL**. 
Click on **Save & Test** then you will see the **Data source is working** message.

* Name: CloudNativeApps
* HTTP URL: http://prometheus-user1-monitoring.apps.seoul-7b68.openshiftworkshop.com

![Grafana]({% image_path granfan-setting.png %})

###11. Utilize Metrics specification on Apps

---

In this lab, we will learn how **Inventory(Quarkus)** application can utilize the MicroProfile Metrics specification through the **SmallRye Metrics extension**.
**MicroProfile Metrics** allows applications to gather various metrics and statistics that provide insights into what is happening inside the application.

The metrics can be read remotely using JSON format or the **OpenMetrics** format, so that they can be processed by additional tools such as **Prometheus**, 
and stored for analysis and visualisation.

We will add Qurakus extensions to the Inventory application for using `smallrye-metrics` and we'll use the Quarkus Maven Plugin.
Copy the following commands to add the smallrye-metricsextension via CodeReady Workspaces **Terminal**:

Go to **inventory** directory:

`mvn quarkus:add-extension -Dextensions="io.quarkus:quarkus-smallrye-metrics"`

If builds successfully (you will see `BUILD SUCCESS`), you will see `smallrye-opentracing` dependency in `pom.xml` automatically.

![metrics_add_extension]({% image_path metrics-extension.png %})

Let's add a few annotations to make sure that our desired metrics are calculated over time and can be exported for processing by **Prometheus** and **Grafana**.

The metrics that we will gather are these:

 * performedChecksAll: A counter of how many getAll() have been performed.
 * checksTimerAll: A measure of how long it takes to perform the getAll().
 * performedChecksAvail: A counter of how many getAvailability() have been performed.
 * checksTimerAvail: A measure of how long it takes to perform the getAvailability().

Open **InventoryResource** file in `src/main/java/com/redhat/coolstore` and add **@Counted**, **@Timed** annotations to `getAll(), getAvailability(@PathParam String itemId)` methods:

~~~java
@GET
    @Counted(name = "performedChecksAll", monotonic = true, description = "How many getAll() have been performed.")
    @Timed(name = "checksTimerAll", description = "A measure of how long it takes to perform the getAll().", unit = MetricUnits.MILLISECONDS)
    public List<Inventory> getAll() {
        return Inventory.listAll();
    }

    @GET
    @Counted(name = "performedChecksAvail", monotonic = true, description = "How many getAvailability() have been performed.")
    @Timed(name = "checksTimerAvail", description = "A measure of how long it takes to perform the getAvailability().", unit = MetricUnits.MILLISECONDS)
    @Path("{itemId}")
    public List<Inventory> getAvailability(@PathParam String itemId) {
        return Inventory.<Inventory>streamAll()
        .filter(p -> p.itemId.equals(itemId))
        .collect(Collectors.toList());
    }
~~~

Add import Microprofile Metrics classes as below:

~~~java
import org.eclipse.microprofile.metrics.MetricUnits;
import org.eclipse.microprofile.metrics.annotation.Counted;
import org.eclipse.microprofile.metrics.annotation.Timed;
~~~

###12. Redeploy to OpenShift

---

Repackage the inventory application via clicking on `Package for OpenShift` in `Commands Palette`:

![codeready-workspace-maven]({% image_path quarkus-dev-run-packageforOcp.png %})


Or you can run a maven plugin command directly in **Terminal**:

`mvn clean package -DskipTests`

Next, create a temp directory to store only previously-built application with necessary lib directory via CodeReady Workspaces **Terminal**:

`rm -rf target/binary && mkdir -p target/binary && cp -r target/*runner.jar target/lib target/binary`

And then start and watch the build, which will take about a minute to complete:

`oc start-build inventory-quarkus --from-dir=target/binary --follow -n userxx-inventory`

Finally, make sure it's actually done rolling out:

`oc rollout status -w dc/inventory-quarkus -n userxx-inventory`

###13. Generate some values for the metrics

---

Open a web browser to access the route URL(i.e. http://inventory-quarkus-user1-inventory.apps.seoul-7b68.openshiftworkshop.com/services/inventory) for invoking **getAll()** method in Inventory service. 

Next, call **getAvailability(@PathParam String itemId)** method as well via the following **CURL** command:

`for i in {1..50}; do curl http://inventory-quarkus-user1-inventory.apps.seoul-7b68.openshiftworkshop.com/services/inventory/329199 ; date ; sleep 1; done`

Let's review the generated metrics. We have 3 ways to view the metircs such as **1)using CURL**, **2)using Prometheus Web UI**, and **3)using Grafana Dashboards**.

**1)** Execute `curl -H"Accept: application/json" inventory-quarkus-user1-inventory.apps.seoul-7b68.openshiftworkshop.com/metrics/application` via CodeReady Workspace Terminal. You should use your own route URL of the Inventory service abd You will receive a similar response as here:

~~~shell
{
  "com.redhat.coolstore.InventoryResource.performedChecksAll" : 291,
  "com.redhat.coolstore.InventoryResource.performedChecksAvail" : 1565,
  "com.redhat.coolstore.InventoryResource.checksTimerAvail" : {
    "p50": 0.826921,
    "p75": 0.995018,
    "p95": 1.151974,
    "p98": 1.245133,
    "p99": 1.319448,
    "p999": 57.101168,
    "min": 0.414609,
    "mean": 1.0356057126214142,
    "max": 64.367892,
    "stddev": 3.0846163605968893,
    "count": 1565,
    "meanRate": 1.188115433328135,
    "oneMinRate": 1.6294141302009826E-4,
    "fiveMinRate": 0.38480021619008725,
    "fifteenMinRate": 0.7264577289402652
  },
  "com.redhat.coolstore.InventoryResource.checksTimerAll" : {
    "p50": 1.308033,
    "p75": 1.418407,
    "p95": 1.606427,
    "p98": 1.68953,
    "p99": 1.70755,
    "p999": 1.70755,
    "min": 0.534711,
    "mean": 1.324642559612213,
    "max": 1357.387543,
    "stddev": 0.1674978928932415,
    "count": 291,
    "meanRate": 0.22092094182355232,
    "oneMinRate": 0.20160825649767997,
    "fiveMinRate": 0.20633505627674062,
    "fifteenMinRate": 0.21085120335191826
  }
}
~~~

**2)** Open the Prometheus Web UI via a web brower and input(or select) `scrape_duration_seconds` in query box. Click on **Execute** then you will see the metrics:

![metrics_prometheus]({% image_path prometheus-metrics-console.png %})

Switch to **Graph** tab:

![metrics_prometheus]({% image_path prometheus-metrics-graph.png %})

**3)** Open the Grafana Web UI via a web brower and create a new **Dashboard** to review the metrics.

![metrics_grafana]({% image_path grafana-create-dashboard.png %})

Click on **New dashboard** then select **Add Query** in a new panel:

![metrics_grafana]({% image_path grafana-add-query.png %}) 

Add `scrape_duration_seconds` in query box:

![metrics_grafana]({% image_path grafana-add-query-detail.png %}) 

Click on **Query Inspector** then you will see the metrics and change **5s** to refresh dashboard:

![metrics_grafana]({% image_path grafana-add-query-complete.png %}) 


#### Summary

---

In this lab, you learned how to monior the cloud-nativa application using Jaeger, Prometheus, and Grafana.
You also learned how Quarkus makes your observation tasks easier as a developer and operator. You can use these techniques 
in future projects to observe your distributed cloud-native applications and container platfrom.