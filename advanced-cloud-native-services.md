## Advanced Cloud-Native Services

If you complete the `Cloud Native Workshop - Module 1`, you learned how to take an existing application to the cloud with JBoss EAP and OpenShift,
and you got a glimpse into the power of OpenShift for existing applications.

In this lab, you will go deeper into how to use the OpenShift Container Platform as a developer to build
and deploy applications. We'll focus on the core features of OpenShift as it relates to developers, and
you'll learn typical workflows for a developer (develop, build, test, deploy, and repeat).

#### Let's get started

---

If you are not familiar with the OpenShift Container Platform, it's worth taking a few minutes to understand
the basics of the platform as well as the environment that you will be using for this workshop.

The goal of OpenShift is to provide a great experience for both Developers and System Administrators to
develop, deploy, and run containerized applications.  Developers should love using OpenShift because it
enables them to take advantage of both containerized applications and orchestration without having the
know the details.  Developers are free to focus on their code instead of spending time writing Dockerfiles
and running docker builds.

Both Developers and Operators communicate with the OpenShift Platform via one of the following methods:

* **Command Line Interface** - The command line tool that we will be using as part of this training is called the *oc* tool. You used this briefly
in the last lab. This tool is written in the Go programming language and is a single executable that is provided for
Windows, OS X, and the Linux Operating Systems.
* **Web Console** -  OpenShift also provides a feature rich Web Console that provides a friendly graphical interface for
interacting with the platform. You can always access the Web Console using the link provided just above
the Terminal window on the right:
* **REST API** - Both the command line tool and the web console actually communicate to OpenShift via the same method,
the REST API.  Having a robust API allows users to create their own scripts and automation depending on
their specific requirements.  For detailed information about the REST API, check out the [official documentation](https://docs.openshift.org/latest/rest_api/index.html).
You will not use the REST API directly in this workshop.

During this workshop, you will be using both the command line tool and the web console.  However, it
should be noted that there are plugins for several integrated development environments as well.
For example, to use OpenShift from the Eclipse IDE, you would want to use the official [JBoss Tools](https://tools.jboss.org/features/openshift.html) plugin.

Now that you know how to interact with OpenShift, let's focus on some core concepts that you as a developer
will need to understand as you are building your applications!


#### Setup for Lab Environment

---

`SKIP this setup guide if you already completed the Cloud Native Workshop - Module 1`

####1. Open the file using CodeReady Workspaces

---

Follow these instructions to setup the development environment on CodeReady Workspaces. 

You might be familiar with the Eclipse IDE which is one of the most popular IDEs for Java and other
programming languages. [CodeReady Workspaces](https://access.redhat.com/documentation/en-us/red_hat_codeready_workspaces) is the next-generation Eclipse IDE which is web-based and gives you a full-featured IDE running in the cloud. You have an CodeReady Workspaces instance deployed on your OpenShift cluster
which you will use during these labs.

Go to the [CodeReady Workspaces URL]({{ ECLIPSE_CHE_URL }}) in order to configure your development workspace.

First, you need to register as a user. Register and choose the same username and password as 
your OpenShift credentials.

![codeready-workspace-register]({% image_path codeready-workspace-register.png %})

Log into CodeReady Workspaces with your user. You can now create your workspace based on a stack. A 
stack is a template of workspace configuration. For example, it includes the programming language and tools needed
in your workspace. Stacks make it possible to recreate identical workspaces with all the tools and needed configuration
on-demand. 

For this lab, click on the **Java Cloud-Native** stack and then on the **Create** button. 

![codeready-workspace-create-workspace]({% image_path codeready-workspace-create-workspace.png %})

Click on **Open** to open the workspace and then on the **Start** button to start the workspace for use, if it hasn't started automatically.

![codeready-workspace-start-workspace]({% image_path codeready-workspace-start-workspace.png %})

You can click on the left arrow icon to switch to the wide view:

![codeready-workspace-wide]({% image_path codeready-workspace-wide.png %})

It takes a little while for the workspace to be ready. When it's ready, you will see a fully functional 
CodeReady Workspaces IDE running in your browser.

![codeready-workspace-workspace]({% image_path codeready-workspace.png %})

Now you can import the project skeletons into your workspace.

In the project explorer pane, click on the **Import Projects...** and enter the following:

> You can find **GIT URL** when you log in {{GIT_URL}} with your credential(i.e. user1 / openshift).

  * Version Control System: `GIT`
  * URL: `{{GIT_URL}}/userXX/cloud-native-workshop-v2m2-labs.git`
  * Check **Import recursively (for multi-module projects)**
  * Name: `cloud-native-workshop-v2m2-labs`

![codeready-workspace-import]({% image_path codeready-workspace-import.png %}){:width="700px"}

The projects are imported now into your workspace and is visible in the project explorer.

CodeReady Workspaces is a full featured IDE and provides language specific capabilities for various project types. In order to 
enable these capabilities, let's convert the imported project skeletons to a Maven projects. In the project explorer, right-click on **monolith** and then click on **Convert to Project**.

![codeready-workspace-convert]({% image_path codeready-workspace-convert.png %}){:width="500px"}

Choose **Maven** from the project configurations and then click on **Save**.

![codeready-workspace-maven]({% image_path codeready-workspace-maven.png %}){:width="700px"}

Repeat the above for inventory and catalog projects.

> **NOTE**: the **Terminal** window in CodeReady Workspaces. For the rest of these labs, anytime you need to run 
a command in a terminal, you can use the CodeReady Workspaces **Terminal** window.

![codeready-workspace-terminal]({% image_path codeready-workspace-terminal.png %})

####2. Create the OpenShift project

---

First, open a new brower with the `OpenShift Web Console`

![openshift_login]({% image_path openshift_login.png %})

Login using:

* Username: `userXX`
* Password: `r3dh4t1`

You will see the OpenShift landing page:

![openshift_landing]({% image_path openshift_landing.png %})

> The project displayed in the landing page depends on which labs you will run today. If you will develop **Service Mesh and Identity** then you will see pre-created projects as the above screeenshot.

Click **Create Project**, fill in the fields, and click **Create**:

* Name: `userXX-coolstore-dev`
* Display Name: `USERXX Coolstore Monolith - Dev`
* Description: _leave this field empty_

> **NOTE**: YOU **MUST** USE `userXX-coolstore-dev` AS THE PROJECT NAME, as this name is referenced later
on and you will experience failures if you do not name it `userXX-coolstore-dev`!

![create_dialog]({% image_path create_dialog.png %})

This will take you to the project status. There's nothing there yet, but that's about to change.

![create_new]({% image_path create_new.png %}){:width="700px"}

This will take you to the project overview. There's nothing there yet, but that's about to change.

####3. Deploy the monolith

---

We'll use the CLI to deploy the components for our monolith. To deploy the monolith template using the CLI, execute the following commands via CodeReady Workspaces **Terminal** window:

Copy login command and Login OpenShift cluster:

![codeready-workspace-copy-login-cmd]({% image_path codeready-workspace-oc-login-copy.png %}){:width="700px"}

Then you will redirect to OpenShift Login page again. 

![openshift_login]({% image_path openshift_login.png %})

When you login with your credential, you will see `Display Token` link in the redirected page.

![openshift_login]({% image_path display_token_link.png %})

Click on the link and copy the **oc login** command:

![openshift_login]({% image_path your_token.png %})

Paste it on CodeReady Workspaces **Terminal** window.

Switch to the developer project you created earlier via CodeReady Workspaces **Terminal** window:

`oc project userXX-coolstore-dev`

And finally deploy template:

`oc new-app coolstore-monolith-binary-build`

This will deploy both a PostgreSQL database and JBoss EAP, but it will not start a build for our application.

Then open up the userXX-coolstore-dev project status page at `OpenShift Web Console`

and verify the monolith template items are created:

![no_deployments]({% image_path no_deployments.png %})

You can see the components being deployed on the
Project Status, but notice the **No running pod for Coolstore**. When you click on **coolstore DC**(Deployment Configs), you will see overview and resources.

![no_deployments]({% image_path dc_overview.png %})

You have not yet deployed the container image built in previous steps, but you'll do that next.

####4. Deploy application using Binary build

---

In this development project we have selected to use a process called binary builds, which
means that instead of pointing to a public Git Repository and have the S2I (Source-to-Image) build process
download, build, and then create a container image for us we are going to build locally
and just upload the artifact (e.g. the `.war` file). The binary deployment will speed up
the build process significantly.

First, build the project once more using the `openshift` Maven profile, which will create a
suitable binary for use with OpenShift (this is not a container image yet, but just the `.war`
file). We will do this with the `oc` command line.

Build the project via CodeReady Workspaces **Terminal** window:

`mvn clean package -Popenshift`

> **NOTE**: Make sure to run this mvn command at working directory(i.e monolith).

Wait for the build to finish and the `BUILD SUCCESS` message!

And finally, start the build process that will take the `.war` file and combine it with JBoss
EAP and produce a Linux container image which will be automatically deployed into the project,
thanks to the *DeploymentConfig* object created from the template:

`oc start-build coolstore --from-file=deployments/ROOT.war`

When you navigate `Builds` menu, you will find out `coolstore-xx` is `running` in Status field:

![building]({% image_path building.png %})

Wait for the build and deploy to complete:

`oc rollout status -w dc/coolstore`

This command will be used often to wait for deployments to complete. Be sure it returns success when you use it!
You should eventually see `replication controller "coolstore-1" successfully rolled out`.

> If the above command reports `Error from server (ServerTimeout)` then simply re-run the command until it reports success!

When it's done you should see the application deployed successfully with blue circles for the
database and the monolith:

![build_done]({% image_path build_done.png %})

Test the application by clicking on the Route link at `Networking > Routes` on the lefe menu:

![route_link]({% image_path route_link.png %})

#####Congratulations!

Now you're ready to create the pipeline for the next CI/CD labs on OpenShift.

![coolstore_web]({% image_path coolstore_web.png %})


#### Developer Concepts

---

There are several concepts in OpenShift useful for developers, and in this workshop
you should be familiar with them.

##### Projects

[Projects](https://docs.openshift.com/container-platform/latest/architecture/core_concepts/projects_and_users.html#projects)
are a top level concept to help you organize your deployments. An
OpenShift project allows a community of users (or a user) to organize and manage
their content in isolation from other communities. Each project has its own
resources, policies (who can or cannot perform actions), and constraints (quotas
and limits on resources, etc). Projects act as a wrapper around all the
application services and endpoints you (or your teams) are using for your work.

##### Containers

The basic units of OpenShift applications are called containers (sometimes called Linux Containers).
[Linux container technologies](https://access.redhat.com/articles/1353593) are lightweight mechanisms for isolating running processes
so that they are limited to interacting with only their designated resources.

Though you do not directly interact with the Docker CLI or service when using OpenShift Container
Platform, understanding their capabilities and terminology is important for understanding their
role in OpenShift Container Platform and how your applications function inside of containers.

##### Pods

OpenShift Container Platform leverages the Kubernetes concept of a pod, which is one or more
containers deployed together on one host, and the smallest compute unit that can be defined,
deployed, and managed.

Pods are the rough equivalent of a machine instance (physical or virtual) to a container.
Each pod is allocated its own internal IP address, therefore owning its entire port space,
and containers within pods can share their local storage and networking.

##### Images

Containers in OpenShift are based on Docker-formatted container images.
An image is a binary that includes all of the requirements for running a single container,
as well as metadata describing its needs and capabilities.

You can think of it as a packaging technology. Containers only have access to resources
defined in the image unless you give the container additional access when creating it.
By deploying the same image in multiple containers across multiple hosts and load balancing
between them, OpenShift Container Platform can provide redundancy and horizontal scaling
for a service packaged into an image.

##### Image Streams

An image stream and its associated tags provide an abstraction for referencing
Images from within OpenShift. The image stream and its tags allow you
to see what images are available and ensure that you are using the specific image you need
even if the image in the repository changes.

##### Builds

A build is the process of transforming input parameters into a resulting object. Most often,
the process is used to transform input parameters or source code into a runnable image. A
_BuildConfig_ object is the definition of the entire build process. It can build from different
sources, including a Dockerfile, a source code repository like Git, or a Jenkins Pipeline
definition.

##### Pipelines

Pipelines allow developers to define a _Jenkins_ pipeline for execution by the Jenkins
pipeline plugin. The build can be started, monitored, and managed by OpenShift Container
Platform in the same way as any other build type.

Pipeline workflows are defined in a Jenkinsfile, either embedded directly in the
build configuration, or supplied in a Git repository and referenced by the build configuration.

##### Deployments

An OpenShift Deployment describes how images are deployed to pods, and how the pods are deployed
to the underlying container runtime platform. OpenShift deployments also provide the ability to transition from
an existing deployment of an image to a new one and also define hooks to be run before or after
creating the replication controller.

##### Services

A Kubernetes service serves as an internal load balancer. It identifies a set of replicated
pods in order to proxy the connections it receives to them. Backing pods can be added to or
removed from a service arbitrarily while the service remains consistently available, enabling
anything that depends on the service to refer to it at a consistent address.

##### Routes

_Services_ provide internal abstraction and load balancing within an
OpenShift environment, sometimes clients (users, systems, devices, etc.)
**outside** of OpenShift need to access an application. The way that external
clients are able to access applications running in OpenShift is through the
OpenShift routing layer. And the data object behind that is a _Route_.

The default OpenShift router (HAProxy) uses the HTTP header of the incoming
request to determine where to proxy the connection. You can optionally define
security, such as TLS, for the _Route_. If you want your _Services_, and, by
extension, your _Pods_,  to be accessible to the outside world, you need to
create a _Route_.

##### Templates

Templates contain a collection of object definitions (BuildConfigs, DeploymentConfigs,
Services, Routes, etc) that compose an entire working project. They are useful for packaging
up an entire collection of runtime objects into a somewhat portable representation of a
running application, including the configuration of the elements.

You will use several pre-defined templates to initialize different environments for the
application. You've already used one in the previous lab to deploy the application
into a _dev_ environment, and you'll use more in this lab to provision the _production_
environment as well.

Consult the [OpenShift documentation](https://docs.openshift.com) for more details on these and other concepts.


####5. Verifying the Dev Environment

---

In the previous step or module 1 workshop, you created a new OpenShift project called `userXX-coolstore-dev` which represents
your developer personal project in which you deployed the CoolStore monolith.

##### Verify Application

Let's take a moment and review the OpenShift resources that are created for the Monolith:

* Build Config: **coolstore** build config is the configuration for building the Monolith
image from the source code or WAR file
* Image Stream: **coolstore** image stream is the virtual view of all coolstore container
images built and pushed to the OpenShift integrated registry.
* Deployment Config: **coolstore** deployment config deploys and redeploys the Coolstore container
image whenever a new coolstore container image becomes available. Similarly, the **coolstore-postgresql**
does the same for the database.
* Service: **coolstore** and **coolstore-postgresql** service is an internal load balancer which identifies a set of
pods (containers) in order to proxy the connections it receives to them. Backing pods can be
added to or removed from a service arbitrarily while the service remains consistently available,
enabling anything that depends on the service to refer to it at a consistent address (service name
or IP).
* Route: **www** route registers the service on the built-in external load-balancer
and assigns a public DNS name to it so that it can be reached from outside OpenShift cluster.

You can review the above resources in the OpenShift Web Console or using the `oc get` or `oc describe` commands
(`oc describe` gives more detailed info):

> You can use short synonyms for long words, like `bc` instead of `buildconfig`,
and `is` for `imagestream`, `dc` for `deploymentconfig`, `svc` for service,
etc.

> **NOTE**: Don't worry about reading and understanding the output of `oc describe`. Just make sure
the command doesn't report errors!

Run these commands to inspect the elements via CodeReady Workspaces **Terminal** window:

`oc get bc coolstore`

`oc get is coolstore`

`oc get dc coolstore`

`oc get svc coolstore`

`oc describe route www`

Verify that you can access the monolith by clicking on the
exposed OpenShift route to open up the sample application in a separate browser tab.

You should also be able to see both the CoolStore monolith and its database
running in separate pods via CodeReady Workspaces **Terminal** window:

`oc get pods -l application=coolstore`

The output should look like this:

~~~shell
NAME                           READY     STATUS    RESTARTS   AGE
coolstore-2-bpkkc              1/1       Running   0          4m
coolstore-postgresql-1-jpcb8   1/1       Running   0          9m
~~~

####6. Verify Database

---

You can log into the running Postgres container using the following via CodeReady Workspaces **Terminal** window:

`oc  rsh dc/coolstore-postgresql`

Once logged in, use the following command to execute an SQL statement to show some content from the database:

`psql -U $POSTGRESQL_USER $POSTGRESQL_DATABASE -c 'select name from PRODUCT_CATALOG;'`

You should see the following:

~~~
          name
------------------------
 Red Fedora
 Forge Laptop Sticker
 Solid Performance Polo
 Ogio Caliber Polo
 16 oz. Vortex Tumbler
 Atari 2600 Joystick
 Pebble Smart Watch
 Oculus Rift
 Lytro Camera
(9 rows)
~~~

Don't forget to exit the pod's shell with `exit`

With our running project on OpenShift, in the next step we'll explore how you as a developer can work with the running app
to make changes and debug the application!
