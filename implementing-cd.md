##  Automating Deployments Using Pipelines

#### Deploying the Production Environment

In the previous scenarios, you deployed the Coolstore monolith using an
OpenShift Template into the `coolstore-dev` Project. The template
created the necessary objects (BuildConfig, DeploymentConfig, ImageStreams, Services, and Routes)
and gave you as a Developer a "playground" in which to run the app, make
changes and debug.

In this step we are now going to setup a separate production environment and explore some
best practices and techniques for developers and DevOps teams for getting code from
the developer (that's YOU!) to production with less downtime and greater consistency.

## Prod vs. Dev

The existing `coolstore-dev` project is used as a developer environment for building new
versions of the app after code changes and deploying them to the development environment.

In a real project on OpenShift, _dev_, _test_ and _production_ environments would typically use different
OpenShift projects and perhaps even different OpenShift clusters.

For simplicity in this scenario we will only use a _dev_ and _prod_ environment, and no test/QA
environment.

## Create the production environment

We will create and initialize the new production environment using another template
in a separate OpenShift project.

**1. Initialize production project environment**

Execute the following `oc` command to create a new project:

`oc new-project coolstore-prod --display-name='Coolstore Monolith - Production'`

This will create a new OpenShift project called `coolstore-prod` from which our production application will run.

**2. Add the production elements**

In this case we'll use the production template to create the objects. Execute:

`oc new-app --template=coolstore-monolith-pipeline-build`

This will use an OpenShift Template called `coolstore-monolith-pipeline-build` to construct the production application.
As you probably guessed it will also include a Jenkins Pipeline to control the production application (more on this later!)

Navigate to the Web Console to see your new app and the components using this link:

* Coolstore Prod Project Overview at 

`https://$OPENSHIFT_MASTER/console/project/coolstore-prod/overview`

![Prod](../../../assets/developer-intro/coolstore-prod-overview.png)

You can see the production database, and an application called _Jenkins_ which OpenShift uses
to manage CI/CD pipeline deployments. There is no running production
app just yet. The only running app is back in the _dev_ environment, where you used a binary
build to run the app previously.

In the next step, we'll _promote_ the app from the _dev_ environment to the _production_
environment using an OpenShift pipeline build. Let's get going!

## Promoting Apps Across Environments with Pipelines

#### Continuous Delivery
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
in a GitHub repository we've been using at [https://github.com/RedHat-Middleware-Workshops/modernize-apps-labs].
This repository has been copied locally to your environment and you've been using it ever since!

You can see the changes you've personally made using `git --no-pager status` to show the code changes you've made using the Git command (part of the
[Git source code management system](https://git-scm.com/)).

## Pipelines

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

**1. Inspect the Pipeline Definition**

Our pipeline is somewhat simplified for the purposes of this Workshop. Inspect the contents of the
pipeline using the following command:

`oc describe bc/monolith-pipeline`

You can see the Jenkinsfile definition of the pipeline in the output:

```
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
```

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

**2. Promote the dev image to production using the pipeline**

You can use the _oc_ command line to invoke the build pipeline, or the Web Console. Let's use the
Web Console. Open the production project in the web console:

* Web Console - Coolstore Monolith Prod at 

`https://$OPENSHIFT_MASTER/console/project/coolstore-prod`

Next, navigate to _Builds -> Pipelines_ and click __Start Pipeline__ next to the `coolstore-monolith` pipeline:

![Prod](../../../assets/developer-intro/pipe-start.png)

This will start the pipeline. **It will take a minute or two to start the pipeline** (future runs will not
take as much time as the Jenkins infrastructure will already be warmed up). You can watch the progress of the pipeline:

![Prod](../../../assets/developer-intro/pipe-prog.png)

Once the pipeline completes, return to the Prod Project Overview at 

`https://$OPENSHIFT_MASTER/console/project/coolstore-prod`
and notice that the application is now deployed and running!

![Prod](../../../assets/developer-intro/pipe-done.png)

View the production app **with the blue header from before** is running by clicking: CoolStore Production App at 

`http://www-coolstore-prod.$ROUTE_SUFFIX` (it may take
a few moments for the container to deploy fully.)

## Congratulations!

You have successfully setup a development and production environment for your project and can
use this workflow for future projects as well.

In the final step we'll add a human interaction element to the pipeline, so that you as a project
lead can be in charge of approving changes.

## More Reading

* [OpenShift Pipeline Documentation](https://docs.openshift.com/container-platform/3.7/dev_guide/dev_tutorials/openshift_pipeline.html)


## Adding Pipeline Approval Steps

In previous steps you used an OpenShift Pipeline to automate the process of building and
deploying changes from the dev environment to production.

In this step, we'll add a final checkpoint to the pipeline which will require you as the project
lead to approve the final push to production.

**1. Edit the pipeline**

Ordinarily your pipeline definition would be checked into a source code management system like Git,
and to change the pipeline you'd edit the _Jenkinsfile_ in the source base. For this workshop we'll
just edit it directly to add the necessary changes. You can edit it with the `oc` command but we'll
use the Web Console.

Open the `monolith-pipeline` configuration page in the Web Console (you can navigate to it from
_Builds -> Pipelines_ but here's a quick link):

* Pipeline Config page at 

`https://$OPENSHIFT_MASTER/console/project/coolstore-prod/browse/pipelines/monolith-pipeline?tab=configuration`

On this page you can see the pipeline definition. Click _Actions -> Edit_ to edit the pipeline:

![Prod](../../../assets/developer-intro/pipe-edit.png)

In the pipeline definition editor, add a new stage to the pipeline, just before the `Deploy to PROD` step:

> **NOTE**: You will need to copy and paste the below code into the right place as shown in the below image.

```groovy
  stage 'Approve Go Live'
  timeout(time:30, unit:'MINUTES') {
    input message:'Go Live in Production (switch to new version)?'
  }
```

Your final pipeline should look like:

![Prod](../../../assets/developer-intro/pipe-edit2.png)

Click **Save**.

**2. Make a simple change to the app**

With the approval step in place, let's simulate a new change from a developer who wants to change
the color of the header in the coolstore back to the original (black) color.

As a developer you can easily un-do edits you made earlier to the CSS file using the source control
management system (Git). To revert your changes, execute:

`git checkout src/main/webapp/app/css/coolstore.css`

Next, re-build the app once more:

`mvn clean package -Popenshift`

And re-deploy it to the dev environment using a binary build just as we did before:

`oc start-build -n coolstore-dev coolstore --from-file=deployments/ROOT.war`

Now wait for it to complete the deployment:

`oc -n coolstore-dev rollout status -w dc/coolstore`

And verify that the original black header is visible in the dev application:

* Coolstore - Dev at 

`http://www-coolstore-dev.$ROUTE_SUFFIX`

![Prod](../../../assets/developer-intro/pipe-orig.png)

While the production application is still blue:

* Coolstore - Prod at 

`http://www-coolstore-prod.$ROUTE_SUFFIX`

![Prod](../../../assets/developer-intro/nav-blue.png)

We're happy with this change in dev, so let's promote the new change to prod, using the new approval step!

**3. Run the pipeline again**

Invoke the pipeline once more by clicking **Start Pipeline** on the Pipeline Config page at 

`https://$OPENSHIFT_MASTER/console/project/coolstore-prod/browse/pipelines/monolith-pipeline`

The same pipeline progress will be shown, however before deploying to prod, you will see a prompt in the pipeline:

![Prod](../../../assets/developer-intro/pipe-prompt.png)

Click on the link for `Input Required`. This will open a new tab and direct you to Jenkins itself, where you can login with
the same credentials as OpenShift:

* Username: `developer`
* Password: `developer`

Accept the browser certificate warning and the Jenkins/OpenShift permissions, and then you'll find yourself at the approval prompt:

![Prod](../../../assets/developer-intro/pipe-jenkins-prompt.png)

**3. Approve the change to go live**

Click **Proceed**, which will approve the change to be pushed to production. You could also have
clicked **Abort** which would stop the pipeline immediately in case the change was unwanted or unapproved.

Once you click **Proceed**, you will see the log file from Jenkins showing the final progress and deployment.

Wait for the production deployment to complete:

`oc rollout -n coolstore-prod status dc/coolstore-prod`

Once it completes, verify that the production application has the new change (original black header):

* Coolstore - Prod at 

`http://www-coolstore-prod.$ROUTE_SUFFIX`

![Prod](../../../assets/developer-intro/pipe-orig.png)

## Congratulations!

You have added a human approval step for all future developer changes. You now have two projects that can be visualized as:

![Prod](../../../assets/developer-intro/goal.png)



## Summary

In this scenario you learned how to use the OpenShift Container Platform as a developer to build,
and deploy applications. You also learned how OpenShift makes your life easier as a developer,
architect, and DevOps engineer.

You can use these techniques in future projects to modernize your existing applications and add
a lot of functionality without major re-writes.

The monolithic application we've been using so far works great, but is starting to show its age.
Even small changes to one part of the app require many teams to be involved in the push to production.

In the next few scenarios we'll start to modernize our application and begin to move away from
monolithic architectures and toward microservice-style architectures using Red Hat technology. Let's go!
