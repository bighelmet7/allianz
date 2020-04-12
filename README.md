# allianz
Technical interview for Allianz

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

1. app/ here lives the sample of the web service written in Go.
2. manifests/ contains all the configuration and settings projects.
  2.1 allianz/ is our main project. It  contains all the yaml files to set the Allianz environment.
  2.2 jenkins/ settings for using Jenkins with a local storage (exam propose, this should  be a nfs or another networking system storage)

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
# 3. Create Jenkins instance

3.1 

# TODO

The steps that your project should follow are the following:

(1. Deploy Minishift cluster) [# 1. Deploy Minishift cluster]
  1.1 Create devops user
2. Create a namespace
3. Deploy a Jenkins instance
  3.1 Creation via template
  3.2 Creation via manifests
4. Create a pipeline that compiles a web application code and deploys it in the namespace. The build step should be performed in an ephemeral slave node, if possible. 
