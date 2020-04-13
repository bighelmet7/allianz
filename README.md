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
* Go v1.13

# 1. Deploy Minishift cluster

```bash
minishift start --openshift-version=v3.10.0 --vm-driver=virtualbox
```
At this point Minishift will perfom all the needed actions to setup the Openshift cluster for us.

# Structure

The two main directories available are app/ and manifests/

1. manifests/ contains all the configuration and settings projects.
1.1 allianz/ is our main project. It  contains all the yaml files to set the Allianz environment.
1.2 jenkins/ contains our jenkins pipelines

# Setting User

For learning propose we will create a user called 'devops' that will perfom all our commands so we can difference between an admin user, developer and devops.

```bash
oc create user devops
# DONT USE THIS METHOD ON PRODUCTION (see LDAP, OAuth, ... https://docs.openshift.com/container-platform/3.11/install_config/configuring_authentication.html)
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

The namespace could be created with the oc CLI or with a yaml/json file; in this case we will be using the file option.

```bash
cd manifests/allianz

oc create -f allianz-ns.yaml
oc get ns allianz # here should be our namespace.
```

Or just do it with the CLI

```bash
oc create ns allianz
```

# 3. Deploy a Jenkins instance

OKD provide us some plug-n-play templates, in our case just select the Jenkins (ephemeral) template, hit the Create button minishift will create our new Jenkins.

NOTE: for a persistent Jenkins version just download it from here: https://github.com/openshift/origin/blob/master/examples/jenkins/jenkins-persistent-template.json

Go to the OKD console, select the allianz project and select Applications>Route and there should be our jenkins route (this route is the way Openshift/K8s exposes
our deployment with their internal load balancing), click on the URL and access with your devops user, also give the requests permissions to Jenkins, so it can 
communicate with our cluster.

# 4. Create a pipeline

We can deploy create a pipeline with Openshift and Jenkins in several ways:
* Template
* Custom yaml's structure

In this case we will perfom the pipeline using our custom YAMLs. The main reason is to have a separate view of how the component should be created and also if at 
some point our client needs a template we can create it merging our yamls in a template yaml (https://docs.openshift.com/container-platform/3.11/dev_guide/templates.html).

The first of all we have to think about how our application: where the source code is? contains a dockerfile? or a jenkinsfile? is it a local app? ...
For this exam:
* the source code lives at http://github.com/bighelmet7/ping
* it contains a dockerfile
* it doesnt contain a jenkinsfile
* is a local webapp (because our cluster is local)

We need to create a ImageStream that will help us to link our ImageStreamTag when we build the Dockerfile of our 'ping' application.

```bash
cd manifests/app
oc create -f ping-is.yaml # is stands for ImageStream
```
Then we need a BuildConfig file that will perfom the action of downloading the Github repository and create the docker image with the Dockerfile and also will output
an ImageStreamTag that can be use in a DeploymentConfig file.

```bash
oc create -f ping-bc.yaml # bc stands for BuildConfig
```

Move on to the Service, Route and DeploymentConfig creation, there are a few things to mark with the DeploymentConfig, first if see the spec.template.spec.containers[0]
element in the image property we can see that is empty, that is because this will be fullfil with the ImageStreamTag (created with the previous BuildConfig) and the last
important thing is the triggers part, if you can notice there is an ImageChange which means: whenever we push a new image of our ping-app this will perfom another
deployment with the new image (https://docs.openshift.com/container-platform/4.1/builds/triggering-builds-build-hooks.html#builds-using-image-change-triggers_triggering-builds-build-hooks)
that matches the provided ImageStreamTag, and the last important thing about this trigger is the **automatic** parameter setted to false, so Jenkins wont have any
conflicts deploying at the same time openshift is deploying the same application.

```bash
oc create -f ping-route.yaml -f ping-svc.yaml -f ping-dc.yaml
```

And finally the pipeline creation which a BuildConfig file but using a Jenkins as a type strategy; our dummy application doesnt contains any Jenkinsfile so then
we must use the jenkinsfile option. Also notice that we cant use declarative pipelines as default, you must install a plug-in for it, because the pipeline has to
execute some openshift methods in it, to connect to our Jenkins; another thing to have a close look is the label we pass to the node, there are three basic agents
in Jenkins with Openshift those are: base, Maven and NodeJS (https://docs.openshift.com/container-platform/3.11/using_images/other_images/jenkins_slaves.html) if
you need another agent you must create it using the Jenkins configure system option and making sure that oc and openshift are installed.

```bash
oc create -f ping-pipeline.yaml
```

For executing this pipeline we move to the OKD dashboard (making sure we are using the devops user), move to our project **allianz** and click on Builds>Pipelines
and there should existr our ping-pipeline hit on the Start pipeline and wait for Jenkins to execute the pipeline.
