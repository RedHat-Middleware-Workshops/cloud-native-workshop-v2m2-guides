The CCN Roadshow(Dev Track) Module 2 Guide 2021
===
This module enables developers to create CI/CD pipeline with cloud-native apps on Red Hat Application Rumties(i.e. Quarkus, Spring Boot) with OpenShift.
The developers also will learn how to debug, monitoring the cloud-native apps via Web IDE(CodeReady Workspace).

Agenda
===
* Advanced Cloud-Native Services
* Implementing Continuous Delivery
* Debugging Applications
* Application Monitoring

Lab Instructions on OpenShift
===

Note that if you have installed the lab infra via APB, the lab instructions are already deployed.

Here is an example Ansible playbook to deploy the lab instruction to your OpenShift cluster manually.

```
- name: Create Guides Module 2
  hosts: localhost
  tasks:
  - import_role:
      name: siamaksade.openshift_workshopper
    vars:
      project_name: "guide-m2"
      workshopper_name: "Cloud-Native Workshop V2 Module-2"
      project_suffix: "-XX"
      workshopper_content_url_prefix: https://raw.githubusercontent.com/RedHat-Middleware-Workshops/cloud-native-workshop-v2m2-guides/master
      workshopper_workshop_urls: https://raw.githubusercontent.com/RedHat-Middleware-Workshops/cloud-native-workshop-v2m2-guides/master/_cloud-native-workshop-module2.yml
      workshopper_env_vars:
        PROJECT_SUFFIX: "-XX"
        COOLSTORE_PROJECT: coolstore{{ project_suffix }}
        OPENSHIFT_CONSOLE_URL: "https://YOUR_OCP_MASTER_URL"
        ECLIPSE_CHE_URL: "http://che-labs-infra.YOUR_OCP_ROUTE_SUBFFIX"
        GIT_URL: "http://gogs-labs-infra.YOUR_OCP_ROUTE_SUBFFIX"
        NEXUS_URL: "http://nexus-labs-infra.YOUR_OCP_ROUTE_SUBFFIX"
        LABS_DOWNLOAD_URL: "http://gogs-labs-infra.YOUR_OCP_ROUTE_SUBFFIX"
         
      openshift_cli: "oc --server https://YOUR_OCP_MASTER_URL"
```
