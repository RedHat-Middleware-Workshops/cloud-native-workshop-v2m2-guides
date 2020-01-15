## Lab3 - Application Monitoring

In the previous labs, you learned how to debug cloud-native apps to fix errors quickly using CodeReady Workspaces
with Quarkus framework, and you got a glimpse into the power of Quarkus for developer joy.

You will now begin observing applications in term of a distributed transaction, performance and latency because as cloud-native applications are developed quickly, a distributed architecture in production gets ultimately complex in two areas: networking and observability. Later on we'll explore how you can better manage and monitor the application using service mesh.

In this lab, you will monitor coolstore applications using [Jaeger](https://www.jaegertracing.io/){:target="_blank"} and [Prometheus](https://prometheus.io/){:target="_blank"}.

![logo]({% image_path quarkus-jaeger-prometheus.png %}){:width="900px"}

**Jaeger** is an open source distributed tracing tool for monitoring and troubleshooting microservices-based distributed systems, including:

 * Distributed context propagation
 * Distributed transaction monitoring
 * Root cause analysis
 * Service dependency analysis
 * Performance and latency optimization

**Prometheus** is an open source systems monitoring and alerting tool that fits in recording any numeric time series, including:

 * Multi-dimensional time series data by metric name and key/value pairs
 * No reliance on distributed storage
 * Time series collection over HTTP
 * Pushing time series is supported via an intermediary gateway
 * Service discovery or static configuration

####1. Create OpenShift Project

---

In this step, we will deploy our new monitoring tools for our CoolStore application, so create a separate project to house it and keep it separate from our monolith and our other microservices we already created previously.

In the [OpenShift web console]({{ CONSOLE_URL}}){:target="_blank"}, create a new project for the _Monitoring_ tools:

Click **Create Project**, fill in the fields, and click **Create**:

* Name: **userXX-monitoring**
* Display Name: **USERXX CoolStore App Monitoring Tools**
* Description: _leave this field empty_

![create_dialog]({% image_path create_monitoring_dialog.png %}){:width="700"}

This will take you to the project overview. There's nothing there yet, but that's about to change.

####2. Deploy Jaeger to OpenShift

---

This template uses an in-memory storage with a limited functionality for local testing and development. Do not use this template in production environments, although there are a number of parameters in the template to constrain the maximum number of traces and the amount of CPU/Memory consumed to prevent node instability.

Install everything in the current namespace via CodeReady Workspaces Terminal:

`oc project userXX-monitoring`

`oc process -f /projects/cloud-native-workshop-v2m2-labs/monitoring/jaeger-all-in-one-template.yml | oc create -f -`

You can also check if the deployment is complete via CodeReady Workspaces Terminal:

`oc rollout status -w deployment.apps/jaeger`

> deployment jaeger successfully rolled out

When you navigate the **Project Status** page in OpenShift console, you will see as below:

![jaeger_deployments]({% image_path jaeger-deployment.png %})

####3. Examine Jaeger-Collector

---

**Collector** is by default accessible only to services running inside the cluster. We will use _jaeger-collector-http_ service with port _14268_ to gather tracing data of Inventoty service later.

![jaeger_deployments]({% image_path jaeger-collector.png %})


####4. Observe Jaeger UI

---

Once you deployed Jaeger to OpenShift, navigate to _Networking > Routes_ and you will see that the route was that generated automatically.

![jaeger_route]({% image_path jaeger-route.png %})

Click on the route URL for `jaeger-query`. This is the UI for Jaeger, but currently we have no apps being monitored so it's rather useless. Don't worry! We will utilize tracing data in the next step.

![jaeger_ui]({% image_path jaeger-ui.png %})

####5. Utilizing Opentracing with Inventory(Quarkus)

---

We have a catalog service written with Spring Boot that calls the inventory service written with Quarkus as part of our cloud-native application. These applications are easy to trace using Jaeger.

In this step, we will add Qurakus extensions to the Inventory application for using **smallrye-opentracing**.
Copy the following commands to add the tracing extension via CodeReady Workspaces Terminal:

Go to _inventory_ directory and add the extension with these commands:

`cd /projects/cloud-native-workshop-v2m2-labs/inventory`

`mvn quarkus:add-extension -Dextensions="opentracing"`

If builds successfully (you will see _BUILD SUCCESS_), you will see _smallrye-opentracing_ dependency in **pom.xml**.

![jaeger_add_extension]({% image_path jaeger-extension.png %})

>NOTE: There are many [more extensions](https://quarkus.io/extensions/){:target="_blank"} for Quarkus for popular frameworks like [CodeReady Workspaces Vert.x](https://vertx.io/){:target="_blank"}, [Apache Camel](http://camel.apache.org/){:target="_blank"}, [Infinispan](http://infinispan.org/){:target="_blank"}, Spring DI compatibility (e.g. @Autowired), and more.

####6. Create the configuration

---

Before getting started with this step, confirm your **jaeger-collector** service in _userXX-monitoring_ project via **oc** command in CodeReady Workspaces Terminal:

`oc get svc -n userXX-monitoring | grep jaeger`

~~~shell
jaeger-agent       ClusterIP      None             <none>                                                                         5775/UDP,6831/UDP,6832/UDP,5778/TCP   4d3h
jaeger-collector   ClusterIP      172.30.225.227   <none>                                                                         14267/TCP,14268/TCP,9411/TCP          4d3h
jaeger-query       LoadBalancer   172.30.88.160    af514f3d7b77711e98f2c06fc57e3ee7-2124798632.ap-southeast-1.elb.amazonaws.com   80:31616/TCP                          4d3h
~~~

The easiest way to configure the **Jaeger tracer** is to set up in the application(i.e. inventory).

Open `src/main/resources/application.properties` file and add the following configuration via CodeReady Workspaces Terminal:

> You need to replace **userXX** with your username in the configuration.

~~~
# Jaeger configuration
quarkus.jaeger.service-name=inventory
quarkus.jaeger.sampler-type=const
quarkus.jaeger.sampler-param=1
quarkus.jaeger.endpoint=http://jaeger-collector.userXX-monitoring:14268/api/traces
~~~

You can also specify the configuration using environment variables or JVM properties. See [Jaeger Features](https://www.jaegertracing.io/docs/1.12/client-features/){:target="_blank"}.

> If the `quarkus.jaeger.service-name` property (or `JAEGER_SERVICE_NAME` environment variable) is not provided then a "no-op" tracer will be configured,
> resulting in no tracing data being reported to the backend.

Currently the tracer can only be configured to report spans directly to the collector via HTTP, using the `quarkus.jaeger.endpoint` property (or `JAEGER_ENDPOINT` environment variable). Support for using the Jaeger agent, via UDP, will be available in a future version.

>NOTE: there is no tracing specific code included in the application. By default, requests sent to this endpoint will be traced without any code changes being required. It is also possible to enhance the tracing information. For more information on this, please see the [MicroProfile OpenTracing specification](https://github.com/eclipse/microprofile-opentracing/blob/master/spec/src/main/asciidoc/microprofile-opentracing.asciidoc){:target="_blank"}.

####7. Re-Deploy to OpenShift

---

Repackage the inventory application via clicking on **Package for OpenShift** in _Commands Palette_:

![codeready-workspace-maven]({% image_path quarkus-dev-run-packageforOcp.png %})

Start and watch the build, which will take about a minute to complete:

`oc start-build inventory-quarkus --from-file target/*-runner.jar --follow -n userXX-inventory`

You should see a **Push successful** at the end of the build output and it. To verify that deployment is started and completed automatically, run the following command via CodeReady Workspaces Terminal :

`oc rollout status -w dc/inventory-quarkus -n userXX-inventory`

And wait for the result as below:

`replication controller "inventory-quarkus-XX" successfully rolled out`

####8. Observing Jaeger Tracing

---

In order to trace networking and data transaction, we will call the Inventory service via **curl** commands via CodeReady Workspaces Terminal:
Be sure to use your route URL of Inventory.

`curl http://$(oc get route inventory-quarkus -o=go-template --template={% raw %}'{{ .spec.host }}'{% endraw %})/services/inventory/165613 ; echo`

Go to _Workloads > Pods_ in the left menu and click on **inventory-quarkus-xxxxxx**.

![codeready-workspace-maven]({% image_path quarkus-jaeger-pod.png %})

Click on **Logs** tab and you will see that tracer is initialized after you call the Inventory service at the first time.

~~~shell
2019-08-05 12:12:17,574 INFO [io.jae.Configuration] (executor-thread-1) Initialized tracer=JaegerTracer(version=Java-0.34.0, serviceName=inventory, reporter=RemoteReporter(sender=HttpSender(), closeEnqueueTimeout=1000), sampler=ConstSampler(decision=true, tags={sampler.type=const, sampler.param=true}), tags={hostname=inventory-quarkus-4-kc4t6, jaeger.version=Java-0.34.0, ip=10.131.8.48}, zipkinSharedRpcSpan=false, expandExceptionLogs=false, useTraceId128Bit=false)
~~~

![jaeger_ui]({% image_path jaeger-init.png %})

Now, reload the Jaeger UI then you will find that 2 services are created as here:

 * inventory
 * jaeger-query

Select the `inventory` service and then click on **Find Traces** and observe the first trace in the graph:

![jaeger_ui]({% image_path jaeger-reload.png %})

If you click on **Span** and you will see a logical unit of work in Jaeger that has an operation name, the start time of the operation, and the duration. Spans may be nested and ordered to model causal relationships:

![jaeger_ui]({% image_path jaeger-span.png %})

Let's make more traces! Open a new web browser to access _CoolStore Inventory Page_ using its route (on _Networking > Routes_ in the OpenShift Console):

> If you do not see the `inventory` route listed, be sure you've chosen the `userXX-inventory` project in the Project selector dropdown!

![jaeger_ui]({% image_path jaeger-coolstore.png %})

Go back to _Jaeger UI_ then click on _Find Traces_. You will see dozens of traces because the Inventory page continues to calling the endpoint of Inventory service in every 2 seconds like we called via **curl** command:

![jaeger_ui]({% image_path jaeger-traces.png %})

####9. Deploy Prometheus and Grafana to OpenShift

---

OpenShift Container Platform ships with a pre-configured and self-updating monitoring stack that is based on the [Prometheus](https://prometheus.io){:target="_blank"} open source project and its wider eco-system. It provides monitoring of cluster components and ships with a set of alerts to immediately notify the cluster administrator about any occurring problems and a set of [Grafana](https://grafana.com/){:target="_blank"} dashboards.

![Prometheus]({% image_path monitoring-diagram.png %}){:width="800px"}

However, we will deploy custom **Prometheus** to scrape services metrics of Inventory and Catalog applications. Then we will visualize the metrics data via custom **Grafana** dashboards deployment.

Go to _Project Status_ page in _userXX-monitoring_ project and click on **Deploy Image** under _Add_ on the right top menu:

![Prometheus]({% image_path add-to-project.png %})

Select **Image Name** and input _prom/prometheus_ to search the Prometheus container image via clicking on _Search_ icon.

![Prometheus]({% image_path search-prometheus-image.png %})

Once you find the image correctly as the above page, click on **Deploy**. It takes 1 ~ 2 mins to deploy a pod.

![Prometheus]({% image_path prometheus-deploy-done.png %})

Create the route to access Prometheus web UI. Navigate _Networking > Routes_ on the left menu and click on **Create Route**.

![Prometheus]({% image_path prometheus-create-route.png %})

Input the following variables and keep the rest of all default variables. Click on **Create**.

 * Name: **prometheus**
 * Service: **prometheus**
 * Target Port: **9090 -> 9090 (TCP)**

![Prometheus]({% image_path prometheus-route-detail.png %})

Now, you have the route URL(i.e. _http://prometheus-user0-monitoring.apps.cluster-seoul-a30e.seoul-a30e.openshiftworkshop.com_) as below and click on the URL to make sure if you can access the _Prometheus web UI_.

![Prometheus]({% image_path prometheus-route-link.png %})

You will see the landing page of Prometheus as shown:

![Prometheus]({% image_path prometheus-webui.png %})

Let's deploy **Grafana Dashboards** to OpenShift. Go to _Project Status_ page in _userXX-monitoring_ project and click on **Deploy Image** under _Add_ menu:

![Grafana]({% image_path add-to-project-grafana.png %})

Select **Image Name** and input _grafana/grafana_ to search the Prometheus container image via clicking on _Search_ icon.

![Grafana]({% image_path search-grafana-image.png %})

Once you find the image correctly as the above screenshot, click on **Deploy**. It takes 1 ~ 2 mins to deploy a pod.

![Grafana]({% image_path grafana-deploy-done.png %})

Create the route to access Grafana web UI. Navigate _Networking > Routes_ on the left menu and click on **Create Route**.

![Grafana]({% image_path grafana-create-route.png %})

Input the following variables and keep the rest of all default variables. Click on `Create`.

 * Name: **grafana**
 * Service: **grafana**
 * Target Port: **3000 -> 3000 (TCP)**

![Grafana]({% image_path grafana-route-detail.png %})

Now, you have the route URL(i.e. _http://grafana-user0-monitoring.apps.cluster-seoul-a30e.seoul-a30e.openshiftworkshop.com_) and click on the URL to make sure if you can access the Grafana web UI.

![Grafana]({% image_path grafana-route-link.png %})

You will see the landing page of Prometheus as shown:

![Grafana]({% image_path grafana-login.png %})

Log in Grafana web UI using the following variables:

* Username: admin
* Password: admin

**Skip** the Change Password.

![Grafana]({% image_path grafana-skip-changepwd.png %})

You will see the landing page of Grafana as shown:

![Grafana]({% image_path grafana-webui.png %})

####10. Add a data source to Grafana

---

Before we create a monitoring dashboard, we need to add a data source. Go to the cog on the side menu that will show you the configuration menu. If the side menu is not visible click the Grafana icon in the upper left corner.

![Grafana]({% image_path grafana-sidemenu-datasource.png %}){:width="600px"}

Click on data sources of the configuration menu and you’ll be taken to the data sources page where you can add and edit data sources.

![Grafana]({% image_path grafana-add-datasource.png %}){:width="700px"}

Click Add data source and select **Prometheus** as data source type.

![Grafana]({% image_path grafana-datasource-types.png %})

Next, input the following variables to configure the dashboard. Make sure to replace the default HTTP URL with your _Prometheus Route URL_ which should be `http://prometheus.userXX-monitoring:9090` (replace `userXX with your username).

Click on **Save & Test** then you will see the _Data source is working_ message.

* Name: CloudNativeApps
* HTTP URL: http://prometheus.userXX-monitoring:9090

![Grafana]({% image_path granfan-setting.png %})

Now Granana is set up to pull collected metrics from Prometheus as they are collected from the application(s) you are monitoring. We'll use this later.

###11. Utilize metrics specification for Inverntory(Quarkus)

---

In this step, we will learn how _Inventory(Quarkus)_ application can utilize the MicroProfile Metrics specification through the **SmallRye Metrics extension**. _MicroProfile Metrics_ allows applications to gather various metrics and statistics that provide insights into what is happening inside the application.

The metrics can be read remotely using JSON format or the **OpenMetrics** format, so that they can be processed by additional tools such as _Prometheus_, and stored for analysis and visualisation.

We will add Qurakus extensions to the Inventory application for using _smallrye-metrics_ and we'll use the Quarkus Maven Plugin. Copy the following commands to add the smallrye-metricsextension via CodeReady Workspaces Terminal:

Go to _inventory`_ directory:

`mvn quarkus:add-extension -Dextensions="metrics"`

If builds successfully (you will see _BUILD SUCCESS_), you will see _smallrye-opentracing_ dependency in _pom.xml_ automatically.

![metrics_add_extension]({% image_path metrics-extension.png %})

Let's add a few annotations to make sure that our desired metrics are calculated over time and can be exported for processing by _Prometheu_ and _Grafana_.

The metrics that we will gather are these:

 * `performedChecksAll`: A counter of how many times `getAll()` has been performed.
 * `checksTimerAll`: A measure of how long it takes to perform the `getAll()` method
 * `performedChecksAvail`: A counter of how many times `getAvailability()` is called
 * `checksTimerAvail`: A measure of how long it takes to perform the getAvailability() method

Open **InventoryResource** file in `src/main/java/com/redhat/coolstore` and replace the two methods `getAll()` and `getAvailability()` with the below code which adds several annotations for custom metrics (`@Counted`, `@Timed_`):

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

Add the necessary imports at the top:

~~~java
import org.eclipse.microprofile.metrics.MetricUnits;
import org.eclipse.microprofile.metrics.annotation.Counted;
import org.eclipse.microprofile.metrics.annotation.Timed;
~~~

###12. Redeploy to OpenShift

---

Repackage the inventory application via clicking on **Package for OpenShift** in _Commands Palette_:

![codeready-workspace-maven]({% image_path quarkus-dev-run-packageforOcp.png %})

Or you can run a maven plugin command directly in Terminal:

`mvn clean package -DskipTests` (make sure you're in the `inventory` directory)

Start and watch the build, which will take about a minute to complete:

`oc start-build inventory-quarkus --from-file target/*-runner.jar --follow -n userxx-inventory`

Finally, make sure it's actually done rolling out:

`oc rollout status -w dc/inventory-quarkus -n userxx-inventory`

Go to the _userXX-monitoring_ project in [OpenShift web console]({{ CONSOLE_URL}}){:target="_blank"} and then on the left sidebar, _Workloads > Config Maps_.

Now we will reconfigure Prometheus so that it knows about our application.

![prometheus]({% image_path prometheus-quarkus-configmap.png %})

Make sure you're in the `userXX-monitoring` project in OpenShift, and click on **Create Config Maps** button to create a config map. You'll copy and paste the below code into the field.

> In the below `ConfigMap` code, you need to replace `userXX-monitoring` with your username prefix (e.g. `user9-monitoring`), **and** replace
> `YOUR_PROMETHEUS_ROUTE` and `YOUR_INVENTORY_ROUTE` with values from your environment, so that Prometheus knows where to scrape metrics from.
> The values you need can be discovered by running the following commands in the Terminal:
>
> * Prometheus Route: `oc get route prometheus -n userxx-monitoring -o=go-template --template={% raw %}'{{ .spec.host }}'{% endraw %}`
> * Inventory Route: `oc get route inventory-quarkus -n userxx-inventory -o=go-template --template={% raw %}'{{ .spec.host }}'{% endraw %}`

Paste in this code and then replace the values as shown in the image below:

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

Config maps hold key-value pairs and in the above command a `prometheus-config` config map is created with `prometheus.yml` as the key and the above content as the value. Whenever a config map is injected into a container, it would appear as a file with the same name as the key, at specified path on the filesystem.

Confirm you created the config map using the terminal command:

`oc describe cm prometheus-config -n userXX-monitoring` (replace `userXX` with your username)

Next, we need to _mount_ this ConfigMap in the filesystem of the Prometheus container so that it can read it. Run this command to alter the Prometheus deployment to mount it (replace `userXX` with your username)

`oc set volume -n userXX-monitoring dc/prometheus --add -t configmap --configmap-name=prometheus-config -m /etc/prometheus/prometheus.yml --sub-path=prometheus.yml`

This will trigger a new deployment. Wait for it with:

`oc rollout status -w dc/prometheus -n userXX-monitoring` (replace `userXX` with your username)

###13. Generate some values for the metrics

---

Let's write a loop to call our inventory service multiple times. First, get the URL to it (replace `userXX` with your username):
`INV_URL=$(oc get route inventory-quarkus -n userxx-inventory -o=go-template --template={% raw %}'{{ .spec.host }}'{% endraw %})`

Next, run this in the same Terminal:

`for i in {1..50}; do curl http://${INV_URL}/services/inventory/329199 ; echo ; date ; sleep 1; done`

This will continually access the inventory project and cause it to generate metrics.

Let's review the generated metrics. We have 3 ways to view the metrics:

* `curl` commands
* Prometheus Web UI
* Grafana Dashboards

Let's look at it with `curl` in a separate terminal:

`INV_URL=$(oc get route inventory-quarkus -n userxx-inventory -o=go-template --template={% raw %}'{{ .spec.host }}'{% endraw %})`

and then

`curl $INV_URL/metrics/application`

You should something like:

~~~
# TYPE application_com_redhat_coolstore_InventoryResource_checksTimerAll_stddev_seconds gauge
application_com_redhat_coolstore_InventoryResource_checksTimerAll_stddev_seconds 1.497008378628945E-4
# HELP application_com_redhat_coolstore_InventoryResource_checksTimerAll_seconds A measure of how long it takes to perform the getAll().
# TYPE application_com_redhat_coolstore_InventoryResource_checksTimerAll_seconds summary
application_com_redhat_coolstore_InventoryResource_checksTimerAll_seconds_count 3107.0
application_com_redhat_coolstore_InventoryResource_checksTimerAll_seconds{quantile="0.5"} 0.001503754
application_com_redhat_coolstore_InventoryResource_checksTimerAll_seconds{quantile="0.75"} 0.001594015
application_com_redhat_coolstore_InventoryResource_checksTimerAll_seconds{quantile="0.95"} 0.001782487
application_com_redhat_coolstore_InventoryResource_checksTimerAll_seconds{quantile="0.98"} 0.001871631
application_com_redhat_coolstore_InventoryResource_checksTimerAll_seconds{quantile="0.99"} 0.001887301
application_com_redhat_coolstore_InventoryResource_checksTimerAll_seconds{quantile="0.999"} 0.001952198
~~~

This shows the raw metrics the application is collecting.

Now let's use Prometheus. Open the **Prometheus Web UI** via a web brower and input `scrape_duration_seconds` in the query box. This is a metric from Prometheus itself indicating how long it takes to scrape metrics. Click on **Execute** then you will see _quarkus job_ in the metrics:

![metrics_prometheus]({% image_path prometheus-metrics-console.png %})

Switch to **Graph** tab:

![metrics_prometheus]({% image_path prometheus-metrics-graph.png %})

You can play with the values for time and see different data across different time ranges for this metric.

Now let's use Grafana.

**3)** Open the **Grafana Web UI** (visit _Networking > Routes_ in the `userXX-monitoring project in the OpenShift console) via a web brower and create a new _Dashboard_ to review the metrics.

![metrics_grafana]({% image_path grafana-create-dashboard.png %})

Click on **New dashboard** then select _Add Query_ in a new panel:

![metrics_grafana]({% image_path grafana-add-query.png %})

Add **scrape_duration_seconds** in query box:

![metrics_grafana]({% image_path grafana-add-query-detail.png %})

Click on **Query Inspector** then you will see _inventory-quarkus metrics_ and change **5s** to refresh dashboard:

![metrics_grafana]({% image_path grafana-add-query-complete.png %})

###14. Utilize metrics specification for Catalog(Spring Boot)

---

In this step, we will learn how to export metrics to _Prometheus_ from _Spring Boot_ application by using the [Prometheus JVM Client](https://github.com/prometheus/client_java){:target="_blank"}.

Go to _Catalog_ project directory and open _pom.xml_ to add the following _Prometheus dependencies_:

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

Next, let's create a new class to configure our metrics for Spring.

In the `catalog` project in CodeReady, open the empty `src/main/java/com/redhat/coolstore/MonitoringConfig.java` class and add the following code to it:

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

Build and deploy the Catalog project using the following command, which will rebuild and redeploy to OpenShift:

`cd /projects/cloud-native-workshop-v2m2-labs/catalog`

and then

`mvn clean package spring-boot:repackage -DskipTests`

The build and deploy may take a minute or two. Wait for it to complete. You should see a **BUILD SUCCESS** at the
end of the build output.

And then re-build the container image, which will take about a minute to complete:

`oc start-build -n userNN-catalog catalog-springboot --from-file target/catalog-1.0.0-SNAPSHOT.jar --follow` (replace `userNN` with your username!)

Once the build is done, it will automatically start a new deployment. Wait for it to complete:

`oc rollout status -w dc/catalog-springboot`

Wait for that command to report replication controller "catalog-springboot-XX" successfully rolled out before continuing.

> NOTE: Even if the rollout command reports success the application may not be ready yet and the reason for that is that we currently don't have any liveness check configured, but we will add that in the next steps.

Let's acess the metrics from the catalog service. Access them via `curl` with these commands:

`CAT_URL=$(oc get route catalog-springboot -n userXX-catalog -o=go-template --template={% raw %}'{{ .spec.host }}'{% endraw %})`

and then:

`curl $CAT_URL/prometheus`

You will see a similar output as here:

~~~
# HELP jvm_gc_collection_seconds Time spent in a given JVM garbage collector in seconds.
# TYPE jvm_gc_collection_seconds summary
jvm_gc_collection_seconds_count{gc="PS Scavenge",} 6.0
jvm_gc_collection_seconds_sum{gc="PS Scavenge",} 0.125
jvm_gc_collection_seconds_count{gc="PS MarkSweep",} 7.0
jvm_gc_collection_seconds_sum{gc="PS MarkSweep",} 1.017
# HELP jvm_classes_loaded The number of classes that are currently loaded in the JVM
# TYPE jvm_classes_loaded gauge
jvm_classes_loaded 9238.0
# HELP jvm_classes_loaded_total The total number of classes that have been loaded since the JVM has started execution
# TYPE jvm_classes_loaded_total counter
jvm_classes_loaded_total 9238.0
# HELP jvm_classes_unloaded_total The total number of classes that have been unloaded since the JVM has started execution
# TYPE jvm_classes_unloaded_total counter
jvm_classes_unloaded_total 0.0
# HELP process_cpu_seconds_total Total user and system CPU time spent in seconds.
# TYPE process_cpu_seconds_total counter
process_cpu_seconds_total 16.45
...
~~~

This is the raw output from the application that Prometheus will periodically read ("scrape").

####16. Add Catalog(Spring Boot) Job

---

You'll need the catalog route once again, which you can discover using this in the Terminal:

`oc get route catalog-springboot -n userXX-catalog -o=go-template --template={% raw %}'{{ .spec.host }}'{% endraw %}; echo`

Navigate to your `userXX-monitoring` project in OpenShift console, and go to `Workloads > Config Maps > prometheus_config`. Click on the _YAML_ tab.

Edit to add the following contents below the existing `job_name` elements (and with the same indentation):

> Replace `YOUR_CATALOG_ROUTE` with the route emitted from the above `oc get route` command!

~~~yaml
  - job_name: 'spring-boot'
    metrics_path: '/prometheus'

    static_configs:
    - targets:  ['YOUR_CATALOG_ROUTE']
~~~

Click on **Save**.

![prometheus]({% image_path prometheus-quarkus-configmap-detail-sb.png %})

OpenShift does not automatically redeploy whenever ConfigMaps are changed, so let's force a redeployment. Select the `userXX-monitoring` project in the OpenShift console, navigate to  _Workloads > Deployment Configs > prometheus_  and select **Start Rollout** from the _Actions_ menu:

![prometheus]({% image_path prometheus-redeploy.png %})

####17. Observing metrics in Prometheus and Grafana

---

**1)** Open the Prometheus Web UI via a web brower and input(or select) `scrape_duration_seconds` in the query box. Click on **Execute** then you will see _spring-boot job_ in the metrics:

![metrics_prometheus]({% image_path prometheus-metrics-console-final.png %})

Switch to **Graph** tab:

![metrics_prometheus]({% image_path prometheus-metrics-graph-final.png %})

**2)** Open the Grafana Web UI via a web brower and access your existing Dashboard. Try to add a new Query with some of the application metrics.

![metrics_grafana]({% image_path grafana-add-query-complete.png %})

#### Summary

---

In this lab, you learned how to monitor cloud-nativa applications using Jaeger, Prometheus, and Grafana. You also learned how Quarkus makes your observation tasks easier as a developer and operator. You can use these techniques in future projects to observe your distributed cloud-native applications.
