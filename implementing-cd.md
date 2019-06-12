## Lab1 - Automating Deployments Using Pipelines

In the previous scenarios, you deployed the Coolstore monolith using an
OpenShift Template into the `coolstore-dev` Project. The template
created the necessary objects (BuildConfig, DeploymentConfig, ImageStreams, Services, and Routes)
and gave you as a Developer a "playground" in which to run the app, make
changes and debug.

In this step, we are now going to setup a separate production environment and explore some
best practices and techniques for developers and DevOps teams for getting code from
the developer **(that's YOU!)** to production with less downtime and greater consistency.

---

#### Production vs. Development

The existing `coolstore-dev` project is used as a developer environment for building new
versions of the app after code changes and deploying them to the development environment.

In a real project on OpenShift, _dev_, _test_ and _production_ environments would typically use different
OpenShift projects and perhaps even different OpenShift clusters.

For simplicity in this scenario we will only use a _dev_ and _prod_ environment, and no test/QA
environment.

**1. Create the production environment**

We will create and initialize the new production environment using another template
in a separate OpenShift project.

In OpenShift web console, click **Create Project**, fill in the fields, and click **Create**:

* Name: `coolstore-prod`
* Display Name: `Coolstore Monolith - Production`
* Description: _leave this field empty_

> **NOTE**: YOU **MUST** USE `coolstore-prod` AS THE PROJECT NAME, as this name is referenced later
on and you will experience failures if you do not name it `coolstore-prod`!

This will create a new OpenShift project called `coolstore-prod` from which our production application will run.

![create_dialog]({% image_path create_prod_dialog.png %}){:width="500"}

**2. Add the production elements**

In this case we'll use the production template to create the objects. Execute via Eclipse Che **Terminal** window:

`oc project coolstore-prod`

And finally deploy template:

`oc new-app --template=coolstore-monolith-pipeline-build`

This will use an OpenShift Template called `coolstore-monolith-pipeline-build` to construct the production application.
As you probably guessed it will also include a Jenkins Pipeline to control the production application (more on this later!)

Navigate to the Web Console to see your new app and the components using this link:

* Coolstore Prod Project Overview at `OpenShift Web Console`:

![Prod]({% image_path coolstore-prod-overview.png %})

You can see the production database, and an application called _Jenkins_ which OpenShift uses
to manage CI/CD pipeline deployments. There is no running production
app just yet. The only running app is back in the _dev_ environment, where you used a binary
build to run the app previously.

In the next step, we'll _promote_ the app from the _dev_ environment to the _production_
environment using an OpenShift pipeline build. Let's get going!

#### Promoting Apps Across Environments with Pipelines

---

##### Continuous Delivery

So far you have built and deployed the app manually to OpenShift in the _dev_ environment. Although
it's convenient for local development, it's an error-prone way of delivering software when
extended to test and production environments.

Continuous Delivery (CD) refers to a set of practices with the intention of automating 
various aspects of delivery software. One of these practices is called delivery pipeline 
which is an automated process to define the steps a change in code or configuration has 
to go through in order to reach upper environments and eventually to production. 

OpenShift simplifies building CI/CD Pipelines by integrating
the popular [Jenkins pipelines](https://jenkins.io/doc/book/pipeline/overview/) into
the platform and enables defining truly complex workflows directly from within OpenShift.

The first step for any deployment pipeline is to store all code and configurations in 
a source code repository. In this workshop, the source code and configurations are stored
in a GitHub repository we've been using at [https://github.com/RedHat-Middleware-Workshops/cloud-native-workshop-v2m2-labs].
This repository has been copied locally to your environment and you've been using it ever since!

You can see the changes you've personally made using `git --no-pager status` to show the code changes you've made using the Git command (part of the
[Git source code management system](https://git-scm.com/)).

##### Pipelines

OpenShift has built-in support for CI/CD pipelines by allowing developers to define
a [Jenkins pipeline](https://jenkins.io/solutions/pipeline/) for execution by a Jenkins
automation engine, which is automatically provisioned on-demand by OpenShift when needed.

The build can get started, monitored, and managed by OpenShift in
the same way as any other build types e.g. S2I. Pipeline workflows are defined in
a Jenkinsfile, either embedded directly in the build configuration, or supplied in
a Git repository and referenced by the build configuration. They are written using the
[Groovy scripting language](http://groovy-lang.org/).

As part of the production environment template you used in the last step, a Pipeline build
object was created. Ordinarily the pipeline would contain steps to build the project in the
_dev_ environment, store the resulting image in the local repository, run the image and execute
tests against it, then wait for human approval to _promote_ the resulting image to other environments
like test or production.

**3. Inspect the Pipeline Definition**

Our pipeline is somewhat simplified for the purposes of this Workshop. Inspect the contents of the
pipeline by navigating **Builds > Pipelines > monolith-pipeline > Configuration** in OpenShift Web Console:

![monolith-pipeline]({% image_path coolstore-prod-monolith-pipeline.png %})

You can also inspect this via the following command via Eclipse Che **Terminal** window:

`oc describe bc/monolith-pipeline`

You can see the Jenkinsfile definition of the pipeline in the output:

~~~
Jenkinsfile contents:
  node ('maven') {
    stage 'Build'
    sleep 5

    stage 'Run Tests in DEV'
    sleep 10

    stage 'Deploy to PROD'
    openshiftTag(sourceStream: 'coolstore', sourceTag: 'latest', namespace: 'coolstore-dev', destinationStream: 'coolstore', destinationTag: 'prod', destinationNamespace: 'coolstore-prod')
    sleep 10

    stage 'Run Tests in PROD'
    sleep 30
  }
~~~

Pipeline syntax allows creating complex deployment scenarios with the possibility of defining
checkpoint for manual interaction and approval process using
[the large set of steps and plugins that Jenkins provides](https://jenkins.io/doc/pipeline/steps/) in
order to adapt the pipeline to the process used in your team. You can see a few examples of
advanced pipelines in the
[OpenShift GitHub Repository](https://github.com/openshift/origin/tree/master/examples/jenkins/pipeline).

To simplify the pipeline in this workshop, we simulate the build and tests and skip any need for human input.
Once the pipeline completes, it deploys the app from the _dev_ environment to our _production_
environment using the above `openshiftTag()` method, which simply re-tags the image you already
created using a tag which will trigger deployment in the production environment.

**4. Promote the dev image to production using the pipeline**

You can use the _oc_ command line to invoke the build pipeline, or the Web Console. Let's use the
Web Console. Open the production project in the web console:

* Web Console - Coolstore Monolith Prod at `OpenShift Web Console`.

Next, navigate to _Builds -> Pipelines_ and click __Start Pipeline__ next to the `coolstore-monolith` pipeline:

![Prod]({% image_path pipe-start.png %}){:width="800px"}

This will start the pipeline. **It will take a minute or two to start the pipeline** (future runs will not
take as much time as the Jenkins infrastructure will already be warmed up). You can watch the progress of the pipeline:

![Prod]({% image_path pipe-prog.png %}){:width="800px"}

Once the pipeline completes, return to the Prod Project Overview at `OpenShift Web Console`

and notice that the application is now deployed and running!

![Prod]({% image_path pipe-done.png %}){:width="800px"}

View the production app **with the blue header from before** is running by clicking: CoolStore Production App at `OpenShift Web Console`

(it may take a few moments for the container to deploy fully.)

**Congratulations!** You have successfully setup a development and production environment for your project and can
use this workflow for future projects as well.

In the next step, we'll add a human interaction element to the pipeline, so that you as a project
lead can be in charge of approving changes.

##### More Reading

* [OpenShift Pipeline Documentation](https://docs.openshift.com/container-platform/3.7/dev_guide/dev_tutorials/openshift_pipeline.html)


---

#### Adding Pipeline Approval Steps

In previous steps, you used an OpenShift Pipeline to automate the process of building and
deploying changes from the dev environment to production.

In this step, we'll add a final checkpoint to the pipeline which will require you as the project
lead to approve the final push to production.

**5. Edit the pipeline**

Ordinarily your pipeline definition would be checked into a source code management system like Git,
and to change the pipeline you'd edit the _Jenkinsfile_ in the source base. For this workshop we'll
just edit it directly to add the necessary changes. You can edit it with the `oc` command but we'll
use the Web Console.

Open the `monolith-pipeline` configuration page in the Web Console (you can navigate to it from
_Builds -> Pipelines_ but here's a quick link):

* Pipeline Config page at `OpenShift Web Console`.

On this page you can see the pipeline definition. Click _Actions -> Edit_ to edit the pipeline:

![Prod]({% image_path pipe-edit.png %}){:width="800px"}

In the pipeline definition editor, add a new stage to the pipeline, just before the `Deploy to PROD` step:

> **NOTE**: You will need to copy and paste the below code into the right place as shown in the below image.

~~~groovy
  stage 'Approve Go Live'
  timeout(time:30, unit:'MINUTES') {
    input message:'Go Live in Production (switch to new version)?'
  }
~~~

Your final pipeline should look like:

![Prod]({% image_path pipe-edit2.png %}){:width="800px"}

Click **Save**.

**6. Make a simple change to the app**

With the approval step in place, let's simulate a new change from a developer who wants to change
the color of the header in the coolstore back to the original (black) color.

First, open `src/main/webapp/app/css/coolstore.css` via Eclipse Che, which contains the CSS stylesheet for the
CoolStore app.

Add the following CSS to turn the header bar background to Red Hat red (**Copy** to add it at the bottom):

~~~java

.navbar-header {
    background: blue
}

~~~

Next, re-build the app once more via Eclipse Che **Terminal**:

`mvn clean package -Popenshift`

And re-deploy it to the dev environment using a binary build just as we did before via Eclipse Che **Terminal**:

`oc start-build -n coolstore-dev coolstore --from-file=deployments/ROOT.war`

Now wait for it to complete the deployment via Eclipse Che **Terminal**:

`oc -n coolstore-dev rollout status -w dc/coolstore`

And verify that the blue header is visible in the dev application:

* Coolstore - Dev at 

![Prod]({% image_path nav-blue.png %}){:width="800px"}

While the production application is still black:

* Coolstore - Prod at 

![Prod]({% image_path pipe-orig.png %}){:width="800px"}

We're happy with this change in dev, so let's promote the new change to prod, using the new approval step!

**7. Run the pipeline again**

Invoke the pipeline once more by clicking **Start Pipeline** on the Pipeline Config page at `OpenShift Web Console`.

The same pipeline progress will be shown, however before deploying to prod, you will see a prompt in the pipeline:

![Prod]({% image_path pipe-prompt.png %}){:width="800px"}

Click on the link for `Input Required`. This will open a new tab and direct you to Jenkins itself, where you can login with
the same credentials as OpenShift:

* Username: `userXX`
* Password: `openshift`

Accept the browser certificate warning and the Jenkins/OpenShift permissions, and then you'll find yourself at the approval prompt:

![Prod]({% image_path pipe-jenkins-prompt.png %}){:width="800px"}

**8. Approve the change to go live**

Click **Proceed**, which will approve the change to be pushed to production. You could also have
clicked **Abort** which would stop the pipeline immediately in case the change was unwanted or unapproved.

Once you click **Proceed**, you will see the log file from Jenkins showing the final progress and deployment.

Wait for the production deployment to complete via Eclipse Che **Terminal**:

`oc rollout -n coolstore-prod status dc/coolstore-prod`

Once it completes, verify that the production application has the new change (blue header):

* Coolstore - Prod at 

![Prod]({% image_path nav-blue.png %}){:width="800px"}

**9. Run the Pipeline on Every Code Change**

Manually triggering the deployment pipeline to run is useful but the real goes is to be able 
to build and deploy every change in code or configuration at least to lower environments 
(e.g. dev and test) and ideally all the way to production with some manual approvals in-place.

In order to automate triggering the pipeline, you can define a webhook on your Git repository 
to notify OpenShift on every commit that is made to the Git repository and trigger a pipeline 
execution.

You can get see the webhook links in the OpenShift Web Console by going to **Build >> Pipelines**, clicking 
on the pipeline and going to the **Configurations** tab.

Copy the Generic webhook url which you will need in the next steps.

Go to your [Git repository]({{GIT_URL}}/userXX/cloud-native-workshop-v2m1-labs.git), then click on **Settings**.

![Repository Settings]({% image_path cd-gogs-settings-link.png %}){:width="900px"}

On the left menu, click on **Webhooks** and then on **Add Webhook** button and then **Gogs**. 

Create a webhook with the following details:

* **Payload URL**: paste the Generic webhook url you copied from the `monolith-pipeline`
* **Content type**: `application/json`

Click on **Add Webhook**. 

![Repository Webhook]({% image_path cd-gogs-webhook-add.png %}){:width="660px"}

All done. You can click on the newly defined webhook to see the list of *Recent Delivery*. 
Clicking on the **Test Delivery** button allows you to manually trigger the webhook for 
testing purposes. Click on it and verify that the `monolith-pipeline` start running 
immediately.

**Congratulations!** You have added a human approval step for all future developer changes. You now have two projects that can be visualized as:

![Prod]({% image_path goal.png %}){:width="800px"}

#### Summary

In this lab, you learned how to use the OpenShift Container Platform as a developer to build,
and deploy applications. You also learned how OpenShift makes your life easier as a developer,
architect, and DevOps engineer.

You can use these techniques in future projects to modernize your existing applications and add
a lot of functionality without major re-writes.

The monolithic application we've been using so far works great, but is starting to show its age.
Even small changes to one part of the app require many teams to be involved in the push to production.
