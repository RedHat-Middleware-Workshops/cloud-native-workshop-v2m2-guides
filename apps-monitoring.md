## Lab3 - Application Monitoring

In the previous labs, you learned how to debug cloud-native apps to fix errors quickly using CodeReady Workspaces 
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

Click `Create Project`, fill in the fields, and click `Create`:

* Name: `userXX-monitoring`
* Display Name: `USERXX CoolStore App Monitoring Tools`
* Description: _leave this field empty_

![create_dialog]({% image_path create_monitoring_dialog.png %}){:width="700"}

This will take you to the project overview. There's nothing there yet, but that's about to change.

####2. Deploy Jaeger to OpenShift

---

This template uses an in-memory storage with a limited functionality for local testing and development. 
Do not use this template in production environments, although there are a number of parameters in the template to 
constrain the maximum number of traces and the amount of CPU/Memory consumed to prevent node instability.

Install everything in the current namespace via CodeReady Workspaces `Terminal`:

`oc project userXX-monitoring`

`oc process -f /projects/cloud-native-workshop-v2m2-labs/monitoring/jaeger-all-in-one-template.yml | oc create -f -`

You can also check if the deployment is complete via CodeReady Workspaces `Terminal`:

`oc rollout status -w deployment.apps/jaeger`

> deployment "jaeger" successfully rolled out

When you navigate the `Project Status` page in OpenShift console, you will see as below:

![jaeger_deployments]({% image_path jaeger-deployment.png %})

####3. Examine Jaeger-Collector

---

`Collector` is by default accessible only to services running inside the cluster. We will use `jaeger-collector-http` service with port `14268` to gather tracing data of Inventoty service later.

![jaeger_deployments]({% image_path jaeger-collector.png %})


####4. Observe Jaeger UI

---

Once you deployed Jaeger to OpenShift, you will see the route that generated automatically.

![jaeger_route]({% image_path jaeger-route.png %})

Click on the route URL(i.e. `https://jaeger-query-user0-monitoring.apps.cluster-seoul-a30e.seoul-a30e.openshiftworkshop.com`) then there is no `Service` and `Operation` at this moment.

![jaeger_ui]({% image_path jaeger-ui.png %})

> Don't worry! We will utilize tracing data later.

####5. Utilizing Opentracing with Inventory(Quarkus)

---

We have a catalog service on Spring Boot that calls inventory service on Quarkus as the cloud-native application. These applications would be 
better to trace using Jaeger rather than monolith coolstore for considering distributed networking system.

In this step, we will add Qurakus extensions to the Inventory application for using `smallrye-opentracing` and we'll use the Quarkus Maven Plugin.
Copy the following commands to add the tracing extension via CodeReady Workspaces `Terminal`:

Go to `inventory` directory:

`mvn quarkus:add-extension -Dextensions="opentracing"`

If builds successfully (you will see `BUILD SUCCESS`), you will see `smallrye-opentracing` dependency in `pom.xml`.

![jaeger_add_extension]({% image_path jaeger-extension.png %})

>`NOTE:` There are many [more extensions](https://quarkus.io/extensions/) for Quarkus for popular frameworks 
like [CodeReady Workspaces Vert.x](https://vertx.io/), [Apache Camel](http://camel.apache.org/), [Infinispan](http://infinispan.org/), 
Spring DI compatibility (e.g. @Autowired), and more.

####6. Create the configuration

---

Before getting started with this step, confirm your `jaeger-collector` service in `userXX-monitoring` project via `oc` command in CodeReady Workspaces `Terminal`:

`oc get svc -n userXX-monitoring | grep jaeger`

~~~shell
jaeger-agent       ClusterIP      None             <none>                                                                         5775/UDP,6831/UDP,6832/UDP,5778/TCP   4d3h
jaeger-collector   ClusterIP      172.30.225.227   <none>                                                                         14267/TCP,14268/TCP,9411/TCP          4d3h
jaeger-query       LoadBalancer   172.30.88.160    af514f3d7b77711e98f2c06fc57e3ee7-2124798632.ap-southeast-1.elb.amazonaws.com   80:31616/TCP                          4d3h
~~~

The easiest way to configure the `Jaeger tracer` is to set up in the application(i.e. inventory).

Open `src/main/resources/application.properties` file and add the following configuration via CodeReady Workspaces `Terminal`:

> You need to replace `userXX` with your username in the configuration.

~~~java
# Jaeger configuration
quarkus.jaeger.service-name=inventory
quarkus.jaeger.sampler-type=const
quarkus.jaeger.sampler-param=1
quarkus.jaeger.endpoint=http://jaeger-collector.userXX-monitoring:14268/api/traces
~~~

You can also specify the configuration using `jvm.args` that Jaeger supplys the properties as [environment variables](https://www.jaegertracing.io/docs/1.12/client-features/).

If the `quarkus.jaeger.service-name` property (or `JAEGER_SERVICE_NAME` environment variable) is not provided then a "no-op" tracer will be configured, 
resulting in no tracing data being reported to the backend.

Currently the tracer can only be configured to report spans directly to the collector via HTTP, using the `quarkus.jaeger.endpoint` property (or `JAEGER_ENDPOINT` environment variable). 
Support for using the Jaeger agent, via UDP, will be available in a future version.

>`NOTE:` there is no tracing specific code included in the application. By default, requests sent to this endpoint will be traced without any code changes being required. It is also possible to enhance the tracing information. For more information on this, please see the [MicroProfile OpenTracing specification](https://github.com/eclipse/microprofile-opentracing/blob/master/spec/src/main/asciidoc/microprofile-opentracing.asciidoc).

####7. Re-Deploy to OpenShift

---

Repackage the inventory application via clicking on `Package for OpenShift` in `Commands Palette`:

![codeready-workspace-maven]({% image_path quarkus-dev-run-packageforOcp.png %})

Start and watch the build, which will take about a minute to complete:

`oc start-build inventory-quarkus --from-file target/*-runner.jar --follow -n userXX-inventory`

You should see a `Push successful` at the end of the build output and it. To verify that deployment is started and completed automatically, 
run the following command via CodeReady Workspaces `Terminal` :

`oc rollout status -w dc/inventory-quarkus -n userXX-inventory`

And wait for the result as below:

`replication controller "inventory-quarkus-XX" successfully rolled out`

####8. Observing Jaeger Tracing

---

In order to trace networking and data transaction, we will call the Inventory service via `curl` commands via CodeReady Workspaces `Terminal`:
Be sure to use your route URL of Inventory.

`curl http://YOUR_INVENTORY_ROUTE_URL/services/inventory/165613 ; echo`

Go to `Workloads > Pods ` in the left menu and click on `inventory-quarkus-xxxxxx`.

![codeready-workspace-maven]({% image_path quarkus-jaeger-pod.png %})

Click on `Logs` tab and you will see that tracer is initialized after you call the Inventory service at the first time.

~~~shell
2019-08-05 12:12:17,574 INFO [io.jae.Configuration] (executor-thread-1) Initialized tracer=JaegerTracer(version=Java-0.34.0, serviceName=inventory, reporter=RemoteReporter(sender=HttpSender(), closeEnqueueTimeout=1000), sampler=ConstSampler(decision=true, tags={sampler.type=const, sampler.param=true}), tags={hostname=inventory-quarkus-4-kc4t6, jaeger.version=Java-0.34.0, ip=10.131.8.48}, zipkinSharedRpcSpan=false, expandExceptionLogs=false, useTraceId128Bit=false)
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

Let's make more traces!! Open a new web browser to access `CoolStore Inventory Pagre`(i.e. http://inventory-quarkus-inventory.apps.seoul-7b68.openshiftworkshop.com):

![jaeger_ui]({% image_path jaeger-coolstore.png %})

Go back to `Jaeger UI` then click on `Find Traces`. You will see dozens of traces because the Inventory page continues to calling the endpoint of Inventory service in every 2 seconds like we called via `curl` command:

![jaeger_ui]({% image_path jaeger-traces.png %})

####9. Deploy Prometheus and Grafana to OpenShift

---

OpenShift Container Platform ships with a pre-configured and self-updating monitoring stack that is based 
on the [Prometheus](https://prometheus.io) open source project and its wider eco-system. It provides monitoring 
of cluster components and ships with a set of alerts to immediately notify the cluster administrator about any occurring problems 
and a set of [Grafana](https://grafana.com/) dashboards.

![Prometheus]({% image_path monitoring-diagram.png %}){:width="800px"}

However, we will deploy custom `Prometheus` to scrape services metrics of Inventory and Catalog applications. 
Then we will visualize the metrics data via custom `Grafana` dashboards deployment.

Go to `Project Status` page in `userXX-monitoring` project and click on `Deploy Image` under `Add` on the right top menu:

![Prometheus]({% image_path add-to-project.png %})

Select `Image Name` and input `prom/prometheus` to search the Prometheus container image via clicking on `Search` icon.

![Prometheus]({% image_path search-prometheus-image.png %})

Once you find the image correctly as the above page, click on `Deploy`. It takes 1 ~ 2 mins to deploy a pod.

![Prometheus]({% image_path prometheus-deploy-done.png %})

Create the route to access `Prometheus` web UI. Navigate `Networking > Routes` on the left menu and click on `Create Route` service.

![Prometheus]({% image_path prometheus-create-route.png %})

Input the following variables and keep the rest of all default variables. Click on `Create`.

 * Name: `prometheus`
 * Service: `prometheus`
 * Target Port: `9090 -> 9090 (TCP)`

![Prometheus]({% image_path prometheus-route-detail.png %})

Now, you have the route URL(i.e. `http://prometheus-user0-monitoring.apps.cluster-seoul-a30e.seoul-a30e.openshiftworkshop.com`) as below 
and click on the URL to make sure if you can access the `Prometheus web UI`.

![Prometheus]({% image_path prometheus-route-link.png %})

You will see the landing page of Prometheus as shown:

![Prometheus]({% image_path prometheus-webui.png %})

Let's deploy `Grafana Dashboards` to OpenShift. Go to `Project Status` page in `userXX-monitoring` project 
and click on `Deploy Image` under `Add` menu:

![Grafana]({% image_path add-to-project-grafana.png %})

Select `Image Name` and input `grafana/grafana` to search the Prometheus container image via clicking on `Search` icon.

![Grafana]({% image_path search-grafana-image.png %})

Once you find the image correctly as the above screenshot, click on `Deploy`. It takes 1 ~ 2 mins to deploy a pod.

![Grafana]({% image_path grafana-deploy-done.png %})

Create the route to access `Grafana` web UI. Navigate `Networking > Routes` on the left menu and click on `Create Route` service.

![Grafana]({% image_path grafana-create-route.png %})

Input the following variables and keep the rest of all default variables. Click on `Create`.

 * Name: `grafana`
 * Service: `grafana`
 * Target Port: `3000 -> 3000 (TCP)`

![Grafana]({% image_path grafana-route-detail.png %})

Now, you have the route URL(i.e. `http://grafana-user0-monitoring.apps.cluster-seoul-a30e.seoul-a30e.openshiftworkshop.com`) and 
click on the URL to make sure if you can access the Grafana web UI.

![Grafana]({% image_path grafana-route-link.png %})

You will see the landing page of Prometheus as shown:

![Grafana]({% image_path grafana-login.png %})

Log in Grafana web UI using the following variables:

* Username: admin
* Password: admin

`Skip` the Change Password.

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

Click Add data source and select `Prometheus` as data source type.

![Grafana]({% image_path grafana-datasource-types.png %})

Next, input the following variables to configure the dashboard. Make sure to replace `HTTP URL` with your `Prometheus Route URL`. 
Click on `Save & Test` then you will see the `Data source is working` message.

* Name: CloudNativeApps
* HTTP URL: http://YOUR_PROMETHEUS_ROUTE_URL

![Grafana]({% image_path granfan-setting.png %})

###11. Utilize metrics specification for Inverntory(Quarkus)

---

In this step, we will learn how `Inventory(Quarkus)` application can utilize the MicroProfile Metrics specification through the `SmallRye Metrics extension`.
`MicroProfile Metrics` allows applications to gather various metrics and statistics that provide insights into what is happening inside the application.

The metrics can be read remotely using JSON format or the `OpenMetrics` format, so that they can be processed by additional tools such as `Prometheus`, 
and stored for analysis and visualisation.

We will add Qurakus extensions to the Inventory application for using `smallrye-metrics` and we'll use the Quarkus Maven Plugin.
Copy the following commands to add the smallrye-metricsextension via CodeReady Workspaces `Terminal`:

Go to `inventory` directory:

`mvn quarkus:add-extension -Dextensions="metrics"`

If builds successfully (you will see `BUILD SUCCESS`), you will see `smallrye-opentracing` dependency in `pom.xml` automatically.

![metrics_add_extension]({% image_path metrics-extension.png %})

Let's add a few annotations to make sure that our desired metrics are calculated over time and can be exported for processing by `Prometheus` and `Grafana`.

The metrics that we will gather are these:

 * performedChecksAll: A counter of how many getAll() have been performed.
 * checksTimerAll: A measure of how long it takes to perform the getAll().
 * performedChecksAvail: A counter of how many getAvailability() have been performed.
 * checksTimerAvail: A measure of how long it takes to perform the getAvailability().

Open `InventoryResource` file in `src/main/java/com/redhat/coolstore` and add `@Counted`, `@Timed` annotations to `getAll(), getAvailability(@PathParam String itemId)` methods:

~~~java
    @GET
    @Counted(name = "performedChecksAll", description = "How many getAll() have been performed.")
    @Timed(name = "checksTimerAll", description = "A measure of how long it takes to perform the getAll().", unit = MetricUnits.MILLISECONDS)
    public List<Inventory> getAll() {
        return Inventory.listAll();
    }

    @GET
    @Counted(name = "performedChecksAvail", description = "How many getAvailability() have been performed.")
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

Or you can run a maven plugin command directly in `Terminal`:

`mvn clean package -DskipTests`

Start and watch the build, which will take about a minute to complete:

`oc start-build inventory-quarkus --from-file target/*-runner.jar --follow -n userxx-inventory`

Finally, make sure it's actually done rolling out:

`oc rollout status -w dc/inventory-quarkus -n userxx-inventory`

Go to the `userXX-monitoring` project in [OpenShift web console]({{ CONSOLE_URL}}) and then on the left sidebar, `Workloads > Config Maps`. 

![prometheus]({% image_path prometheus-quarkus-configmap.png %})

Click on `Create Config Maps` button to create a config map with the following info:

> You need to replace `userXX-monitoring`, `YOUR_PROMETHEUS_ROUTE_URL` and `YOUR_INVENTORY_ROUTE_URL` except `http://` align with your environment.

 * Prometheus Route URL example: _prometheus-user0-monitoring.apps.cluster-seoul-a30e.seoul-a30e.openshiftworkshop.com_
 * Inventory Route URL example: _inventory-quarkus-user0-inventory.apps.cluster-seoul-a30e.seoul-a30e.openshiftworkshop.com_

~~~yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: userXX-monitoring
data:
  prometheus.yml: >-
    # my global config

    global:
      scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
      evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
      # scrape_timeout is set to the global default (10s).

    # Alertmanager configuration

    alerting:
      alertmanagers:
      - static_configs:
        - targets:
          # - alertmanager:9093

    # Load rules once and periodically evaluate them according to the global
    'evaluation_interval'.

    rule_files:
      # - "first_rules.yml"
      # - "second_rules.yml"

    # A scrape configuration containing exactly one endpoint to scrape:

    # Here it's Prometheus itself.

    scrape_configs:
      # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
      - job_name: 'prometheus'

        # metrics_path defaults to '/metrics'
        # scheme defaults to 'http'.

        static_configs:
        - targets: ['YOUR_PROMETHEUS_ROUTE_URL']
      
      - job_name: 'quarkus'
        metrics_path: '/metrics/application'

        static_configs:
        - targets:  ['YOUR_INVENTORY_ROUTE_URL']
~~~
 
![prometheus]({% image_path prometheus-quarkus-configmap-detail.png %})

Config maps hold key-value pairs and in the above command an `prometheus-config` config map 
is created with `prometheus.yml` as the key and the above content as the value. Whenever a config map is injected into a container, 
it would appear as a file with the same name as the key, at specified path on the filesystem.

You can see the content of the config map in the [OpenShift web console]({{ CONSOLE_URL}}) or by 
using `oc describe cm prometheus-config -n userXX-monitoring` command via CodeReady Workspaces `Terminal`.

Modify the `Prometheus deployment config` so that it injects the `prometheus.yml` configuration you just created as 
a config map into the Prometheus container. Go to `Workloads > Deployment Configs` in `userXX-monitoring` project page 
and click on `prometheus` deployment:

![prometheus]({% image_path prometheus-dc.png %})

Move to `YAML` tab menu and add the `prometheus-config` in `spec.volumes` and `spec.containers.volumeMounts` sections.

 * `spec.volumes` section:

 ~~~yaml
 volumes:
  - name: prometheus-1
    emptyDir: {}
  - configMap:
      defaultMode: 420
      name: prometheus-config
    name: volume-tcjnf
 ~~~

 * `spec.containers.volumeMounts` section:

~~~yaml
volumeMounts:
  - name: prometheus-1
    mountPath: /prometheus
  - name: volume-tcjnf
    mountPath: /etc/prometheus
~~~

![prometheus]({% image_path prometheus-configuration-tab.png %})

You can also use `oc set volume` command for this:

`oc set volume dc/prometheus --add --configmap-name=prometheus-config --mount-path=/etc/prometheus -n userxx-monitoring`

###13. Generate some values for the metrics

---

Open a web browser to access the inventory route URL for invoking `getAll()` method in Inventory service. 

Next, call `getAvailability(@PathParam String itemId)` method as well via the following `CURL` command with your Inventory's route URL: 

`for i in {1..50}; do curl http://YOUR_INVENTORY_ROUTE_URL/services/inventory/329199 ; date ; sleep 1; done`

Let's review the generated metrics. We have 3 ways to view the metircs such as `1)using CURL`, `2)using Prometheus Web UI`, and `3)using Grafana Dashboards`.

`1)` Execute `curl -H"Accept: application/json" http://YOUR_INVENTORY_ROUTE_URL/metrics/application` via CodeReady Workspaces Terminal. You should use your own route URL of the Inventory service abd You will receive a similar response as here:

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

`2)` Open the `Prometheus Web UI` via a web brower and input(or select) `scrape_duration_seconds` in query box. 
Click on `Execute` then you will see `quarkus job` in the metrics:

![metrics_prometheus]({% image_path prometheus-metrics-console.png %})

Switch to `Graph` tab:

![metrics_prometheus]({% image_path prometheus-metrics-graph.png %})

`3)` Open the `Grafana Web UI` via a web brower and create a new `Dashboard` to review the metrics.

![metrics_grafana]({% image_path grafana-create-dashboard.png %})

Click on `New dashboard` then select `Add Query` in a new panel:

![metrics_grafana]({% image_path grafana-add-query.png %}) 

Add `scrape_duration_seconds` in query box:

![metrics_grafana]({% image_path grafana-add-query-detail.png %}) 

Click on `Query Inspector` then you will see `inventory-quarkus metrics` and change `5s` to refresh dashboard:

![metrics_grafana]({% image_path grafana-add-query-complete.png %}) 

###14. Utilize metrics specification for Catalog(Spring Boot)

---

In this step, we will learn how to export metrics to `Prometheus` from `Spring Boot` application by 
using the [Prometheus JVM Client](https://github.com/prometheus/client_java).

Go to `Catalog` project directory and open `pom.xml` to add the following `Prometheus dependencies`:

~~~xml
<!-- Prometheus dependency  -->
<dependency>
    <groupId>io.prometheus</groupId>
    <artifactId>simpleclient_spring_boot</artifactId>
    <version>0.6.0</version>
</dependency>

<dependency>
    <groupId>io.prometheus</groupId>
    <artifactId>simpleclient_hotspot</artifactId>
    <version>0.6.0</version>
</dependency>

<dependency>
    <groupId>io.prometheus</groupId>
    <artifactId>simpleclient_servlet</artifactId>
    <version>0.6.0</version>
</dependency>
~~~

![metrics_grafana]({% image_path catalog-prometheus-dependency.png %}) 

Next, create `MonitoringConfig.java` class in `src/main/java/com/redhat/coolstore/` and copy the following codes into `MonitoringConfig.java` file:

~~~java
package com.redhat.coolstore;

import io.prometheus.client.exporter.MetricsServlet;
import io.prometheus.client.hotspot.DefaultExports;
import io.prometheus.client.spring.boot.SpringBootMetricsCollector;
import org.springframework.boot.actuate.endpoint.PublicMetrics;
import org.springframework.boot.web.servlet.ServletRegistrationBean;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.util.Collection;

@Configuration
class MonitoringConfig {

    @Bean
    SpringBootMetricsCollector springBootMetricsCollector(Collection<PublicMetrics> publicMetrics) {

        SpringBootMetricsCollector springBootMetricsCollector = new SpringBootMetricsCollector(publicMetrics);
        springBootMetricsCollector.register();

        return springBootMetricsCollector;
    }

    @Bean
    ServletRegistrationBean servletRegistrationBean() {
        DefaultExports.initialize();
        return new ServletRegistrationBean(new MetricsServlet(), "/prometheus");
    }
}
~~~

####15. Re-Build and Re-Deploy to OpenShift

---

Build and deploy the Catalog project using the following command, which will use the maven plugin to deploy via CodeReady Workspaces `Terminal`:

`oc project userxx-catalog`

`mvn package fabric8:deploy -Popenshift -DskipTests`

The build and deploy may take a minute or two. Wait for it to complete. You should see a `BUILD SUCCESS` at the
end of the build output.

After the maven build finishes it will take less than a minute for the application to become available.
To verify that everything is started, run the following command and wait for it complete successfully:

![catalog_deploy_success]({% image_path catalog_deploy_success.png %})

You can also check if the deployment is complete via CodeReady Workspaces `Terminal`:

`oc rollout status -w dc/catalog -n userXX-catalog`

Let's `collect metrics`. Open a web browser with the following URL after replacing with your `Catalog route URL`:

`http://catalog-userXX-catalog.apps.seoul-7b68.openshiftworkshop.com/prometheus`

You will see a similar output as here:

~~~shell
# HELP jvm_gc_collection_seconds Time spent in a given JVM garbage collector in seconds.
# TYPE jvm_gc_collection_seconds summary
jvm_gc_collection_seconds_count{gc="PS Scavenge",} 52.0
jvm_gc_collection_seconds_sum{gc="PS Scavenge",} 1.147
jvm_gc_collection_seconds_count{gc="PS MarkSweep",} 2.0
jvm_gc_collection_seconds_sum{gc="PS MarkSweep",} 0.504
# HELP process_cpu_seconds_total Total user and system CPU time spent in seconds.
# TYPE process_cpu_seconds_total counter
process_cpu_seconds_total 26.38
# HELP process_start_time_seconds Start time of the process since unix epoch in seconds.
# TYPE process_start_time_seconds gauge
process_start_time_seconds 1.560957605855E9
# HELP process_open_fds Number of open file descriptors.
# TYPE process_open_fds gauge
process_open_fds 47.0
# HELP process_max_fds Maximum number of open file descriptors.
# TYPE process_max_fds gauge
process_max_fds 1048576.0
# HELP process_virtual_memory_bytes Virtual memory size in bytes.
# TYPE process_virtual_memory_bytes gauge
process_virtual_memory_bytes 4.748623872E9
# HELP process_resident_memory_bytes Resident memory size in bytes.
# TYPE process_resident_memory_bytes gauge
process_resident_memory_bytes 2.91467264E8
# HELP jvm_memory_pool_allocated_bytes_total Total bytes allocated in a given JVM memory pool. Only updated after GC, not continuously.
# TYPE jvm_memory_pool_allocated_bytes_total counter
jvm_memory_pool_allocated_bytes_total{pool="Code Cache",} 1.84624E7
jvm_memory_pool_allocated_bytes_total{pool="PS Eden Space",} 2.070413312E9
jvm_memory_pool_allocated_bytes_total{pool="PS Old Gen",} 4.1407848E7
jvm_memory_pool_allocated_bytes_total{pool="PS Survivor Space",} 5595280.0
jvm_memory_pool_allocated_bytes_total{pool="Compressed Class Space",} 7020064.0
jvm_memory_pool_allocated_bytes_total{pool="Metaspace",} 5.8175456E7
# HELP jvm_classes_loaded The number of classes that are currently loaded in the JVM
# TYPE jvm_classes_loaded gauge
jvm_classes_loaded 10770.0
# HELP jvm_classes_loaded_total The total number of classes that have been loaded since the JVM has started execution
# TYPE jvm_classes_loaded_total counter
jvm_classes_loaded_total 10770.0
...
~~~

####16. Add Catalog(Spring Boot) Job

---

Edit `prometheus-config` configmap in `userXX-monitoring` project with the following contents:

~~~java
  - job_name: 'spring-boot'
    metrics_path: '/prometheus'

    static_configs:
    - targets:  ['YOUR_CATALOG_ROUTE_URL']
~~~

Click on `Save`.

![prometheus]({% image_path prometheus-quarkus-configmap-detail-sb.png %})

Redeploy `Workloads > Deployment Configs` in the left menu and click on `Start Rollout`:

![prometheus]({% image_path prometheus-redeploy.png %})

####17. Observing metrics in Prometheus and Grafana

---

`1)` Open the Prometheus Web UI via a web brower and input(or select) `scrape_duration_seconds` in query box. 
Click on `Execute` then you will see `quarkus job` in the metrics:

![metrics_prometheus]({% image_path prometheus-metrics-console-final.png %})

Switch to `Graph` tab:

![metrics_prometheus]({% image_path prometheus-metrics-graph-final.png %})

`2)` Open the Grafana Web UI via a web brower and click on `Query Inspector` then you will see 
`inventory-quarkus` and `catalog-sprinb-boot` metrics:

![metrics_grafana]({% image_path grafana-add-query-complete.png %}) 

#### Summary

---

In this lab, you learned how to monior the cloud-nativa application using Jaeger, Prometheus, and Grafana.
You also learned how Quarkus makes your observation tasks easier as a developer and operator. You can use these techniques 
in future projects to observe your distributed cloud-native applications and container platfrom.