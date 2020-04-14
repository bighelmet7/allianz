# Allianz
Technical interview for Allianz

# TODO

The steps that your project should follow are the following:

- [x] Deploy Minishift cluster
- [x] Create devops user
- [x] Create a namespace
- [x] Deploy a Jenkins instance
- [x] Create a pipeline that compiles a web application code and deploys it in the namespace. The build step should be performed in an ephemeral slave node, if possible. 

# Tools

* Minishift version v1.34.2 (running with virtualbox as vm-driver)
* Openshift version v3.10.0

# 1. Deploy Minishift cluster

```bash
minishift start --openshift-version=v3.10.0 --vm-driver=virtualbox
```
At this point Minishift will perfom all the needed actions to set up the Openshift cluster.

# Structure

The two main directories

* manifests/ contains all the project configuration and its settings.
* allianz/ contains all the yaml files to set up the Allianz environment.
* app/ contains all the yamls to set up and configure our web application.

# Setting up an user

We will create a user called 'devops' that will perfom all our commands.

```bash
oc create user devops
# DONT USE THIS METHOD IN PRODUCTION (see LDAP, OAuth, ... https://docs.openshift.com/container-platform/3.11/install_config/configuring_authentication.html)
oc create identity anypassword:devops 
oc create useridentitymapping anypassword:devops

oc adm policy add-cluster-role-to-user cluster-admin devops # Finally bind this user to the cluster-admin role.
```

Logout and login with our devops user.

```bash
oc logout
oc login -u devops -p writeAnythingHere
```

# 2. Create a Namespace

The namespace can be created either using the **oc CLI** or a yaml/json file. In our case we will be using a yaml file.

```bash
cd manifests/allianz

oc create -f allianz-ns.yaml
oc get ns allianz # here should be our namespace.
```

Or it can be done using the CLI

```bash
oc create ns allianz
```

# 3. Deploy a Jenkins instance

OKD provides us with a catalog of plug-n-play templates. Instead of creating our own Jenkins environment we will able to select the Jenkins (ephemeral) template by: selecting the created project, clicking the 'Catalog' option, selecting the suitable 'Jenkins (ephemeral)' and  then clicking the 'Create' option. Finally, the minishift will perform all the needed steps to set up our new Jenkins.

NOTE: for a persistent Jenkins version, it can be downloaded using the following link: https://github.com/openshift/origin/blob/master/examples/jenkins/jenkins-persistent-template.json

To access the Jenkins environment, go to the OKD console, select the **allianz** project then Applications>Route and a display of our Jenkins route (this route is how Openshift/K8s exposes any deployment with their internal load balancing) will be available. Click on the URL and login with the created devops user. Allow Jenkins's permission requests, so it can communicate with our cluster.

# 4. Create a pipeline

We can create a pipeline with Openshift and Jenkins in two ways:
* Template
* Custom yaml's structure

In our case we will create the pipeline using our custom yaml's file. The prupose is to have a more detailed view of how all the components should be set up. If at a certain point our client needs a unified template, we can do it by merging our yamls into a yaml template (https://docs.openshift.com/container-platform/3.11/dev_guide/templates.html).

First of all we have to analyze the application we want to deploy: where do the source codes exist? does it contain any dockerfile? or a jenkinsfile? is it a local app? ...
For this test:
* the source can be found at http://github.com/bighelmet7/ping
* it contains a dockerfile
* it doesn't contain a jenkinsfile
* it is a local webapp

We need to create an ImageStream that will help us to link our ImageStreamTag when we build the Dockerfile of our 'ping' application.

```bash
cd manifests/app
oc create -f ping-is.yaml # is stands for ImageStream
```
We need a BuildConfig file that will perform the action of downloading the Github repository and create a docker image using the Dockerfile. Afterwards it will output an ImageStreamTag 'ping-app:latest' that can be used in a DeploymentConfig file.

```bash
oc create -f ping-bc.yaml # bc stands for BuildConfig
```

We need to create the Service, Route and DeploymentConfig files. Three aspects of the DeploymentConfig need to be noted:
* The image property of spec.template.spec.containers[0] must be empty, because it will be automatically fulfilled by the ImageStreamTag (previously created by the BuildConfig).
* Only when a new build image 'ping-app:latest' is detected; will ImageChange only then trigger the redeployment of our application. (https://docs.openshift.com/container-platform/4.1/builds/triggering-builds-build-hooks.html#builds-using-image-change-triggers_triggering-builds-build-hooks)
* ImageChange has an **automatic** parameter that must be set to _false_ to avoid any conflict between Jenkins and Openshift deploying at the same time.

```bash
oc create -f ping-route.yaml -f ping-svc.yaml -f ping-dc.yaml
```

Finally, through the Jenkins strategy a pipeline is created that is essentially a BuildConfig file. Also, it's worth noting that we can't use declarative pipelines as default; for that a plug-in must be installed. Our pipeline uses a 'nodejs' agent to execute our defined stages. Jenkins provide us with three agents, which are: _Base, Maven and NodeJS_ (https://docs.openshift.com/container-platform/3.11/using_images/other_images/jenkins_slaves.html). If another agent is needed, create it using the Jenkins configuration system option and make sure that **oc** is installed.

```bash
oc create -f ping-pipeline.yaml
```

We can execute this pipeline by opening the OKD dashboard (login using devops), selecting our project **allianz** and clicking on Builds>Pipelines. There should be a panel with our 'ping-pipeline' and a 'Start pipeline' buttons. Click on 'Start pipeline' and wait for Jenkins to execute it.
