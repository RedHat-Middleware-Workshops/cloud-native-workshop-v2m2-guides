## Lab1 - Automating Deployments Using Pipelines

In the previous scenarios, you deployed the Coolstore monolith using an OpenShift Template into the **userXX-coolstore-dev** Project. The template created the necessary objects (BuildConfig, DeploymentConfig, ImageStreams, Services, and Routes) and gave you as a Developer a "playground" in which to run the app, make changes and debug.

In this step, we are now going to setup a separate production environment and explore some best practices and techniques for developers and DevOps teams for getting code from the developer **(that's YOU!)** to production with less downtime and greater consistency.

#### Production vs. Development

---

The existing **userXX-coolstore-dev** project is used as a developer environment for building new versions of the app after code changes and deploying them to the development environment.

In a real project on OpenShift, _dev_, _test_ and _production_ environments would typically use different OpenShift projects and perhaps even different OpenShift clusters.

For simplicity in this scenario we will only use a _dev_ and _prod_ environment, and no test/QA environment.

####1. Create the production environment

---

First, open a new brower with the [OpenShift web console]({{ CONSOLE_URL}}){:target="_blank"}

![openshift_login]({% image_path openshift_login.png %})

Login using:

* Username: `userXX`
* Password: `r3dh4t1!`

> **NOTE**: Use of self-signed certificates
>
> When you access the OpenShift web console]({{ CONSOLE_URL}}) or other URLs via _HTTPS_ protocol, you will see browser warnings
> like `Your > Connection is not secure` since this workshop uses self-signed certificates (which you should not do in production!).
> For example, if you're using **Chrome**, you will see the following screen.
>
> Click on `Advanced` then, you can access the HTTPS page when you click on `Proceed to...`!!!
>
> ![warning]({% image_path browser_warning.png %})
>
> Other browsers have similar procedures to accept the security exception.

You will see the OpenShift landing page:

![openshift_landing]({% image_path openshift_landing.png %})

> The project displayed in the landing page depends on which labs you will run today. If you will develop `Service Mesh and Identity` then you will see pre-created projects as the above screeenshot.

Click `Create Project`, fill in the fields, and click `Create`:

* Name: **userXX-coolstore-prod**
* Display Name: **USERXX Coolstore Monolith - Production**
* Description: _leave this field empty_

> NOTE: YOU `MUST` USE `userXX-coolstore-prod` AS THE PROJECT NAME, as this name is referenced later on and you will experience failures if you do not name it `userXX-coolstore-prod`.

This will create a new OpenShift project called **userXX-coolstore-prod** from which our production application will run.

![create_dialog]({% image_path create_prod_dialog.png %}){:width="700px"}

####2. Add the production elements

---

In this case we'll use the production template to create the objects. Execute via CodeReady Workspaces Terminal window:

`oc project userXX-coolstore-prod`

And finally deploy template:

`oc new-app --template=coolstore-monolith-pipeline-build`

We have to deploy a **Jenkins Server** in the namespace because OpenShift 4 doesn't deploy a Jenkins server automatically when we use _Jenkins Pipeline_ build strategy.

`oc new-app --template=jenkins-ephemeral -l app=jenkins -p JENKINS_SERVICE_NAME=jenkins -p DISABLE_ADMINISTRATIVE_MONITORS=true`

`oc set resources dc/jenkins --limits=cpu=1,memory=2Gi --requests=cpu=1,memory=512Mi`

This will use an OpenShift Template called **coolstore-monolith-pipeline-build** to construct the production application.
As you probably guessed it will also include a Jenkins Pipeline to control the production application (more on this later!)

Navigate to the Web Console to see your new app and the components using this link:

* Coolstore Prod Project Status at [OpenShift web console]({{ CONSOLE_URL}}){:target="_blank"}:

![Prod]({% image_path coolstore-prod-overview.png %})

You can see the production database, and an application called _Jenkins_ which OpenShift uses to manage CI/CD pipeline deployments. There is no running production
app just yet. The only running app is back in the _dev_ environment, where you used a binary build to run the app previously.

In the next step, we'll _promote_ the app from the _dev_ environment to the _production_ environment using an OpenShift pipeline build. Let's get going!

#### Promoting Apps Across Environments with Pipelines

---

##### Continuous Delivery

So far you have built and deployed the app manually to OpenShift in the _dev_ environment. Although it's convenient for local development, it's an error-prone way of delivering software when extended to test and production environments.

Continuous Delivery (CD) refers to a set of practices with the intention of automating various aspects of delivery software. One of these practices is called delivery pipeline which is an automated process to define the steps a change in code or configuration has to go through in order to reach upper environments and eventually to production.

OpenShift simplifies building CI/CD Pipelines by integrating the popular [Jenkins pipelines](https://jenkins.io/doc/book/pipeline/overview/){:target="_blank"} into the platform and enables defining truly complex workflows directly from within OpenShift.

The first step for any deployment pipeline is to store all code and configurations in a source code repository. In this workshop, the source code and configurations are stored in a [GitHub repository](https://github.com/RedHat-Middleware-Workshops/cloud-native-workshop-v2m2-labs){:target="_blank"} we've been using. This repository has been copied locally to your environment and you've been using it ever since!

##### Pipelines

OpenShift has built-in support for CI/CD pipelines by allowing developers to define a [Jenkins pipeline](https://jenkins.io/solutions/pipeline/){:target="_blank"} for execution by a Jenkins
automation engine, which is automatically provisioned on-demand by OpenShift when needed.

The build can get started, monitored, and managed by OpenShift in the same way as any other build types e.g. S2I. Pipeline workflows are defined in a Jenkinsfile, either embedded directly in the build configuration, or supplied in \a Git repository and referenced by the build configuration. They are written using the
[Groovy scripting language](http://groovy-lang.org/).

As part of the production environment template you used in the last step, a Pipeline build object was created. Ordinarily the pipeline would contain steps to build the project in the _dev_ environment, store the resulting image in the local repository, run the image and execute tests against it, then wait for human approval to _promote_ the resulting image to other environments like test or production.

####3. Inspect the Pipeline Definition

---

Our pipeline is somewhat simplified for the purposes of this Workshop. Inspect the contents of the pipeline by navigating _Builds > Build Configs_ and click on `monolith-pipeline` in the [OpenShift web console]({{ CONSOLE_URL}}){:target="_blank"}. Then, you will see the details of _Jenkinsfile_ on the right side:

![monolith-pipeline]({% image_path coolstore-prod-monolith-bc.png %})

You can also inspect this via the following command via CodeReady Workspaces Terminal window:

`oc describe bc/monolith-pipeline`

You can see the Jenkinsfile definition of the pipeline in the output:

~~~shell
Jenkinsfile contents:
  pipeline {
    agent {
      label 'maven'
    }
    stages {
      stage ('Build') {
        steps {
          sleep 5
        }
      }
      stage ('Run Tests in DEV') {
        steps {
          sleep 10
        }
      }
      stage ('Deploy to PROD') {
        steps {
          script {
            openshift.withCluster() {
              openshift.tag("userXX-coolstore-dev/coolstore:latest", "userXX-coolstore-prod/coolstore:prod")
            }
          }
        }
      }
      stage ('Run Tests in PROD') {
        steps {
          sleep 30
        }
      }
    }
  }
~~~

> NOTE: You have to replace your username with **userXX** in Jenkinsfile via clicking on **YAML** tab. For example, if your username is user0, it will be user0-coolstore-dev and user0-coolstore-prod. Don't forget to click on **Save**.

![monolith-pipeline]({% image_path coolstore-prod-monolith-update-jenkins.png %})

The pipeline syntax allows creating complex deployment scenarios with the possibility of defining checkpoints for manual interaction and approval processes using [the large set of steps and plugins that Jenkins provides](https://jenkins.io/doc/pipeline/steps/) in order to adapt the pipeline to the processes used in your team. You can see a few examples of advanced pipelines in the [OpenShift GitHub Repository](https://github.com/openshift/origin/tree/master/examples/jenkins/pipeline){:target="_blank"}.

To simplify the pipeline in this workshop, we simulate the build and tests and skip any need for human input. Once the pipeline completes, it deploys the app from the _dev_ environment to our _production_ environment using the above `tag()` method within the `openshift` object, which simply re-tags the image you already created using a tag which will trigger deployment in the production environment.

####4. Promote the dev image to production using the pipeline

---

Before promoting the dev image, you need to modify a **RoleBinding** to access the dev image by Jenkins. This allows the Jenkins service account in the **userXX-coolstore-prod** project to access the image within the **userXX-coolstore-dev** project.

Go to overview page of the `userXX-coolstore-dev` project, then navigate to _Administration > Role Bindings_. Click on **ci_admin**:

![Prod]({% image_path coolstore-dev-ci-admin.png %})

Move to **YAML** tab and replace your username with _userXX_ then click on **Save**:

![Prod]({% image_path coolstore-dev-ci-admin-save.png %})

Let's invoke the build pipeline by using [OpenShift web console]({{ CONSOLE_URL}}){:target="_blank"}. Open the production project in the web console.

Next, navigate to _Builds > Build Configs > monolith-pipeline_, click the small menu at the far right, and click _Start Build_:

![Prod]({% image_path pipe-start.png %})

This will start the pipeline. _It will take a minute or two to start the pipeline!_ Future runs will not take as much time as the Jenkins infrastructure will already be warmed up. You can watch the progress of the pipeline:

![Prod]({% image_path pipe-prog.png %})

Once the pipeline completes, return to the Prod Project Status at [OpenShift web console]({{ CONSOLE_URL}}){:target="_blank"} and notice that the application is now deployed and running!

![Prod]({% image_path pipe-done.png %})

It may take a few moments for the container to deploy fully.

#####Congratulations!

You have successfully setup a development and production environment for your project and can use this workflow for future projects as well.

In the next step, we'll add a human interaction element to the pipeline, so that you as a project lead can be in charge of approving changes.

##### More Reading

* [OpenShift Pipeline Documentation](https://docs.openshift.com/container-platform/4.1/builds/build-strategies.html#builds-strategy-pipeline-build_build-strategies){:target="_blank"}

#### Adding Pipeline Approval Steps

---

In previous steps, you used an OpenShift Pipeline to automate the process of building and deploying changes from the dev environment to production.

In this step, we'll add a final checkpoint to the pipeline which will require you as the project lead to approve the final push to production.

####5. Edit the pipeline

---

Ordinarily your pipeline definition would be checked into a source code management system like Git, and to change the pipeline you'd edit the _Jenkinsfile_ in the source base. For this workshop we'll just edit it directly to add the necessary changes. You can edit it with the **oc** command but we'll use the Web Console.

Go back to _Builds > Build Configs > monolith-pipeline_ then click on _Edit Build Config_.

![Prod]({% image_path pipe-edit.png %})

Click on **YAML** tab and add _a new stage_ to the pipeline, just before the _Deploy to PROD_ stage:

> NOTE: You will need to copy and paste the below code into the right place as shown in the below image.

~~~groovy
            stage ('Approve Go Live') {
              steps {
                timeout(time:30, unit:'MINUTES') {
                  input message:'Go Live in Production (switch to new version)?'
                }
              }
            }
~~~

Your final pipeline should look like:

![Prod]({% image_path pipe-edit2.png %})

Click **Save**.

####6. Make a simple change to the app

---

With the approval step in place, let's simulate a new change from a developer who wants to change the color of the header in the coolstore to a blue background color.

First, open _monolith/src/main/webapp/app/css/coolstore.css_ via CodeReady Workspace, which contains the CSS stylesheet for the CoolStore app.

Add the following CSS to turn the header bar background to Red Hat red (**Copy** to add it at the bottom):

~~~java

.navbar-header {
    background: blue
}

~~~

Now we need to update the catalog endpoint in the monolith application. Copy the route URL of catalog service using following **oc** command in CodeReady Workspaces Terminal. Replace your username with **userXX**:

`echo "http://$(oc get route -n userXX-catalog | grep catalog | awk '{print $2}')"`

In the **monolith** project (within the root **cloud-native-workshop-v2m2-labs** project), open **catalog.js** in **src/main/webapp/app/services** and add a line as shown in the image to define the value of **baseUrl**.

`baseUrl="http://REPLACEURL/services/products";`

> Replace **REPLACEURL** with the URL emitted from the previous **echo** command

Next, re-build the app once more via CodeReady Workspaces Terminal:

`cd /projects/cloud-native-workshop-v2m2-labs/monolith`

`mvn clean package -Popenshift`

And re-deploy it to the dev environment using a binary build just as we did before via CodeReady Workspaces Terminal:

`oc start-build -n userXX-coolstore-dev coolstore --from-file=deployments/ROOT.war --follow`

Now wait for it to complete the deployment via CodeReady Workspaces Terminal:

`oc -n userXX-coolstore-dev rollout status -w dc/coolstore`

And verify that the blue header is visible in the dev application by navigating to the `userXX-coolstore-dev` project in the OpenShift Console, and then going to _Networking > Routes_ and clicking on the route URL. It should look like the following:

> If it doesn't, you may need to do a hard browser refresh. Try holding the shift key while clicking the browser refresh button.

![Dev]({% image_path nav-blue.png %})

Then navigating to the `userXX-coolstore-prod` project in the OpenShift Console, and then going to _Networking > Routes_ and clicking on the route URL for the production app. It should still be black:

![Prod]({% image_path pipe-orig.png %})

We're happy with this change in dev, so let's promote the new change to prod, using the new approval step!

####7. Run the pipeline again

---

Invoke the pipeline once more by navigating to _Builds > Build Configs > monolith-pipeline > Rebuild_. The same pipeline progress will be shown, however before deploying to prod, you will see a prompt in the pipeline:

![Prod]({% image_path pipe-start2.png %}).

Click on the link for **Input Required**. This will open a new tab and direct you to Jenkins itself, where you can login with the same credentials as OpenShift:

* Username: `userXX`
* Password: `r3dh4t1!`

Accept the browser certificate warning and the Jenkins/OpenShift permissions, and then you'll find yourself at the approval prompt:

Click on **Console Output** on left menu then click on `Proceed`.

![Prod]({% image_path pipe-jenkins-prompt.png %})

####8. Approve the change to go live

---

Click **Proceed**, which will approve the change to be pushed to production. You could also have clicked **Abort** which would stop the pipeline immediately in case the change was unwanted or unapproved.

Once you click _Proceed_, you will see the log file from Jenkins showing the final progress and deployment.

Wait for the production deployment to complete via CodeReady Workspaces Terminal:

`oc rollout -n userXX-coolstore-prod status -w dc/coolstore-prod`

Once it completes, verify that the production application has the new change (blue header):

![Prod]({% image_path nav-blue.png %})

> If it doesn't, you may need to do a hard browser refresh. Try holding the shift key while clicking the browser refresh button.

####9. Run the Pipeline on Every Code Change

---

Manually triggering the deployment pipeline to run is useful but it would be better to run the pipeline automatically on every change in code or configuration, at least to lower environments (e.g. dev and test) and ideally all the way to production with some manual approvals in-place.

In order to automate triggering the pipeline, you can define a webhook on your Git repository to notify OpenShift on every commit that is made to the Git repository and trigger a pipeline execution.

You can get see the webhook links in the [OpenShift web console]({{ CONSOLE_URL}}){:target="_blank"} by going to _Builds > Build Configs > monolith-pipeline_. Look for the _Generic secret_ value in the _YAML_ tab. Copy this down.

Then go back to the _Overview_ tab. At the bottom you'll find the _Generic_ webhook url which you will need (along with the secret) in the next steps.

Go to your Git repository at `{{ GIT_URL }}/userXX/cloud-native-workshop-v2m2-labs.git` (replace `userXX` with your username and open this URL in a new tab), Click **Sign In** and sign in with your credentials:

* Username: `userXX` (replace with your username)
* Password: `r3dh4t1!`

 then click on **Settings**.

![Repository Settings]({% image_path cd-gogs-settings-link.png %}){:width="900px"}

On the left menu, click on **Webhooks** and then on **Add Webhook** button and then **Gogs**.

Create a webhook with the following details:

* Payload URL: paste the Generic webhook url you copied from the **monolith-pipeline** (make sure to replace the `secret` value in the URL!)
* Content type: **application/json**

Click on **Add Webhook**.

![Repository Webhook]({% image_path cd-gogs-webhook-add.png %}){:width="660px"}

All done. You can **click on the newly defined webhook** to see the list of *Recent Delivery*.  Click on the **Test Delivery** button allows you to manually trigger the webhook for testing purposes. Click on it and verify that the _monolith-pipeline_ starts running immediately (navigate to _Builds > Builds_ then you should see one running. Click on it to ensure the pipeline is executing, and optionally confirm the _Approve Go Live_ as before).

`Congratulations!` You have added a human approval step for all future developer changes. You now have two projects that can be visualized as:

![Prod]({% image_path goal.png %}){:width="800px"}

#### Summary

---

In this lab, you learned how to use the OpenShift Container Platform as a developer to build, and deploy applications. You also learned how OpenShift makes your life easier as a developer, architect, and DevOps engineer.

You can use these techniques in future projects to modernize your existing applications and add a lot of functionality without major re-writes.

The monolithic application we've been using so far works great, but is starting to show its age. Even small changes to one part of the app require many teams to be involved in the push to production.
