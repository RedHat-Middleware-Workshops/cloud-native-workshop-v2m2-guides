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

**1. Create OpenShift Project**

In this step, we will deploy our new monitoring tools for our CoolStore application,
so create a separate project to house it and keep it separate from our monolith and our other microservices we already
created previously.

Create a new project for the _Monitoring_ tools:

Click **Create Project**, fill in the fields, and click **Create**:

* Name: `monitoring`
* Display Name: `CoolStore App Monitoring Tools`
* Description: _leave this field empty_

![create_dialog]({% image_path create_monitoring_dialog.png %}){:width="500"}

Click on the name of the newly-created project:

![create_new]({% image_path create_new_monitoring.png %}){:width="500"}

This will take you to the project overview. There's nothing there yet, but that's about to change.

**2. Deploy Jaeger to OpenShift**

This template uses an in-memory storage with a limited functionality for local testing and development. 
Do not use this template in production environments, although there are a number of parameters in the template to 
constrain the maximum number of traces and the amount of CPU/Memory consumed to prevent node instability.

Install everything in the current namespace via CodeReady Workspace **Terminal**:

`oc project monitoring`

`oc process -f https://raw.githubusercontent.com/jaegertracing/jaeger-openshift/master/all-in-one/jaeger-all-in-one-template.yml | oc create -f -`

You can also check if the deployment is complete via CodeReady Workspace **Terminal**:

`oc rollout status -w deployment.apps/jaeger`

> deployment "jaeger" successfully rolled out

When you navigate the **overview** page in OpenShift console, you will see as below:

![jaeger_deployments]({% image_path jaeger-deployment.png %})

**3. Exposing Jaeger-Collector**

**Collector** is by default accessible only to services running inside the cluster. The easiest approach to expose the collector outside of 
the cluster is via the `jaeger-collector-http` HTTP port using an **OpenShift Route** via CodeReady Workspace **Terminal**:

`oc create route edge --service=jaeger-collector --port jaeger-collector-http --insecure-policy=Allow`

This allows clients to send data directly to Collector via HTTP senders. If you want to use the Agent then use ExternalIP or NodePort to expose the Collector service.

> **NOTE:** Using Collector will open the collector to be used by any external party, who will then be able to create arbitrary spans. It's advisable to put an OAuth Security Proxy in front of the collector and expose this proxy instead.

**4. Observe Jaeger UI**

Once you deployed Jaeger to OpenShift, you will see the route that generated automatically.

![jaeger_route]({% image_path jaeger-route.png %})

Click on the route URL(i.e. https://jaeger-query-monitoring.apps.seoul-7b68.openshiftworkshop.com) then there is no **Service** and **Operation** at this moment.

![jaeger_ui]({% image_path jaeger-ui.png %})

Don't worry! We will utilize tracing data later.

**5. Utilizing Opentracing with Quarkus**

We have a catalog service on Spring Boot that calls inventory service on Quarkus as the cloud-native application. These applications would be 
better to trace using Jaeger rather than monolith coolstore for considering distributed networking system.

In this step, we will add Qurakus extensions to the Inventory application for using `smallrye-opentracing` and we'll use the Quarkus Maven Plugin.
Copy the following commands to add the tracing extension via CodeReady Workspaces **Terminal**:

Go to `inventory' directory:

`cd cloud-native-workshop-v2m2-labs/inventory/`

`mvn quarkus:add-extension -Dextensions="io.quarkus:quarkus-smallrye-opentracing"`

If builds successfully (you will see `BUILD SUCCESS`), you will see `smallrye-opentracing` dependency in `pom.xml` automatically.

![jaeger_add_extension]({% image_path jaeger-extension.png %})

**6. Create the configuration**

Before getting started with this step, confirm your **route URL of Jaeger** and we will use the following step to create the tracing configuration.
You can find out the URL in OpenShift Console or use the `oc` command as here:

![jaeger_ui]({% image_path jaeger-collector-route.png %})

`oc get route`

~~~shell
NAME               HOST/PORT                                                           PATH   SERVICES           PORT                    TERMINATION   WILDCARD
jaeger-collector   jaeger-collector-monitoring.apps.seoul-7b68.openshiftworkshop.com          jaeger-collector   jaeger-collector-http   edge/Allow    None
jaeger-query       jaeger-query-monitoring.apps.seoul-7b68.openshiftworkshop.com              jaeger-query       query-http              edge/Allow    None
~~~

The easiest way to configure the Jaeger tracer is to set up in the application(i.e. Inventory).

Open `src/main/resources/application.properties` file and add the following configuration via CodeReady Workspaces **Terminal**:

~~~java
# Jaeger configuration
quarkus.jaeger.service-name=inventory
quarkus.jaeger.sampler-type=const
quarkus.jaeger.sampler-param=1
quarkus.jaeger.endpoint=https://jaeger-collector-monitoring.apps.seoul-7b68.openshiftworkshop.com/api/traces
~~~

> You should replace **quarkus.jaeger.endpoint** with your own route URL(**HTTPS**) as you created.

You can also specify the configuration using `jvm.args` that Jaeger supplys the properties as [environment variables](https://www.jaegertracing.io/docs/1.12/client-features/).

If the **quarkus.jaeger.service-name** property (or **JAEGER_SERVICE_NAME** environment variable) is not provided then a "no-op" tracer will be configured, 
resulting in no tracing data being reported to the backend.

Currently the tracer can only be configured to report spans directly to the collector via HTTP, using the `quarkus.jaeger.endpoint` property (or `JAEGER_ENDPOINT` environment variable). Support for using the Jaeger agent, via UDP, will be available in a future version.

>**NOTE:** there is no tracing specific code included in the application. By default, requests sent to this endpoint will be traced without any code changes being required. It is also possible to enhance the tracing information. For more information on this, please see the [MicroProfile OpenTracing specification](https://github.com/eclipse/microprofile-opentracing/blob/master/spec/src/main/asciidoc/microprofile-opentracing.asciidoc).

**7. Re-Deploy to OpenShift**

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

`oc start-build inventory-quarkus --from-dir=target/binary --follow -n inventory`

You should see a **Push successful** at the end of the build output and it. To verify that deployment is started and completed automatically, 
run the following command via CodeReady Workspaces **Terminal** :

`oc rollout status -w dc/inventory-quarkus`

And wait for the result as below:

`replication controller "inventory-quarkus-XX" successfully rolled out`

**8. Observing Jaeger Tracing**

In order to trace networking and data transaction, we will call the Inventory service via `curl` commands via CodeReady Workspaces **Terminal**:

`curl http://inventory-quarkus-inventory.apps.seoul-7b68.openshiftworkshop.com/services/inventory/165613 ; echo`

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

**Congratulations!**