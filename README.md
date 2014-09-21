cloudify-nodecellar-cloudstack
==============================
A sample Cloudify 3 blueprint for CloudStack, consisted of a nodejs server and mongodb database.  The sample nodejs application used here is the [nodecellar](http://coenraets.org/blog/2012/10/nodecellar-sample-application-with-backbone-js-twitter-bootstrap-node-js-express-and-mongodb/), but it could have been any other application for that matter. 

## Before You Begin 

Before you can deploy this application using Cloudify, you'll need to have the following setup in your environment: 
* Linux or Mac OSX. For Linux, the Cloudify CLI has been tested with Ubuntu 13.04 or higher, and Arch Linux (but should work just as well on other Linux distros). For Mac it's been tested with OSX mavericks. Windows support will be added in the near future. 
* A CloudStack cloud environment and credentials. This blueprint and the provider and plugin it uses are based on [Exoscale](http://www.exoscale.ch/) endpoint URLs, so the easiest would be to [setup an account with Exoscale](https://portal.exoscale.ch/register). The blueprint also assumes flat network CloudStack setup. 
* Python 2.7 or higher installed
* `python-dev` installed. In Ubuntu you can use `apt-get install python-dev`, for Mac you can use homebrew or macports. 
* Pip (Python package manager) 1.5 or higher installed 

## Step 1: Install the Cloudify CLI

The first thing you'll need to do is install the Cloudify CLI, which will let you upload blueprints, create deployments from them and execute workflows on these deployments. This is quite simple once you have Python and pip installed. It is recommended to install the CLI in a new python [virtual environment](http://docs.python-guide.org/en/latest/dev/virtualenvs/). That will make it easier to upgrade and remove, and will also save you the need to you use `sudo` with the installation. To install the CLI follow the following steps: 
* If you're using a python `virtualenv`, activate it. 

```
cd /path/to/virtualenv
source bin/activate
```

* Type the following command: 

```
pip install cloudify
```

After the installation completes, you will have `cfy` command installed. Type `cfy -h` to verify that the installation completed successfully. You should see the help message. 

## Step 2: Install the Cloudify CloudStack Provider 

Next, you need to install the Cloudify CloudStack provider. The provider allows the CLI to initialize a CloudStack  configuration and bootstrap a Cloudify manager on an CloudStack cloud (we used [Exoscale](https://www.exoscale.ch/) for this example). The bootstrap process creates two keypairs (named `cloudify-agents-kp` and `cloudify-management-kp`), two security groups (named `cloudify-agents-sg` and `cloudify-management-sg`), starts a management VM on the CloudStack cloud, and installs the Cloudify management components on it. These include (among other things) an Nginx proxy, a nodejs server for the Cloudify Web UI, an Flask API server, a Ruby based workflow engine, ElasticSearch and Logstash for log aggregation and runtime state, RabbitMQ for messaging, and a Python Celery worker for processing tasks that are created when triggering workflows. But from a user's perspective, all it takes to bootstrap a manager is a few simple steps. To install the CloudStack provider, type the following command in your CLI:

```
pip install cloudify-openstack
```

Note for Mac users: One of the libraries that's installed with the CloudStack provider (`pycrypto`) may fail to compile. This seems to be a [known issue](http://stackoverflow.com/questions/19617686/trying-to-install-pycrypto-on-mac-osx-mavericks/22868650#22868650). To solve it, type the following command in your terminal windows and try the installation again: 

```
export CFLAGS=-Qunused-arguments
export CPPFLAGS=-Qunused-arguments
```

## Step 3: Initialize a CloudStack Configuraton 

Next, you need to create a CloudStack congifuration and save your credentials into it. To create the configuration, type the following command: 

```
cfy init exoscale
```

This will create a Cloudify configuration file named `cloudify-config.yaml` in the current directory (it will also create a file named `.cloudify` to save the current context for the `cfy` tool, but you shouldn't care about that for now). 

Next, open the file `cloudify-config.yaml` in your text editor of choice. You will need to change the following lines in this file and type in your api key and secret key. 

```yaml
authentication:
    api_key: 'API_KEY'
    api_secret_key: 'API_SECRET_KEY'

```

## Step 4: Boostrap the Cloudify Manager 
Now you're ready to bootstrap your cloudify manager. To do so type the following command in the terminal windows: 

```
cfy bootstrap
```

This should take a few minutes to complete. After validating the configuration, `cfy` will list all of the resources created, create the management VM and related networks and security groups (the latter two will not be created if they already exist), download the relevant Cloudify manager packages from the internet and install all of the components. At the end of this process you should see the following message: 

```
bootstrapping complete
management server is up at <YOUR MANAGER IP ADDRESS> (is now set as the default management server)
```

To validate this installation, point your web browser to the manager IP address (port 80). You should see the Cloudify web UI. At this point there's nothing much to see since you haven't yet uploaded any blueprint. 

## Step 5: Upload the Bluprint and Create a Deployment 

Next, we'll upload the sample blueprint and create a deployment based on it. You will first need to clone this repository into your local file system. To do so type the following command: 

```
git clone https://github.com/cloudify-cosmo/cloudify-nodecellar-cloudstack.git
```

This will create a directory called `cloudify-nodecellar-cloudstack` in your current directory. cd to this directory. You can see the blueprint file (named `blueprint.yaml`) alongside other resources related to this blueprint. 
Before uploading the blueprint, you need to update it and replace the place holders for the API key and secret key in it as well (this step will not be needed soon and will be automactically injected from the provider credentials).
After updating the credentials (there are two places in the blueprint that have them - 'exoscale_security_group' and exoscale_vm') upload the blueprint using the following command: 

```
cfy blueprints upload -b nodecellar1 blueprint.yaml
```

The `-b` parameter is the unique name we've given to this blueprint on the Cloudify manager. A blueprint is a template of an application stack. Blueprints cannot be materialize on their own. For that you will need to create a deployment, which is essintially an instance of this blueprint (kind of like what an instance is to a class in an OO model). But first let's go back to the web UI and see what this blueprint looks like. Point your browser to the manager URL again, and refresh the screen. You will see the nodecellar blueprint listed there. 

![Blueprints table](https://raw.githubusercontent.com/cloudify-cosmo/cloudify-nodecellar-cloudstack/master/blueprints_table.png)

Click the row with the blueprint. You will now see the topology of this blueprint. A topology is consisted of elements called nodes. In our case, we have the following nodes: a security group, two VMs, a nodejs server, a mongodb server, and a nodejs application called nodecellar (which is a nice sample nodejs application backed by mongodb). 

![Nodecellar Blueprint](https://raw.githubusercontent.com/cloudify-cosmo/cloudify-nodecellar-cloudstack/master/blueprint.png)

Next, we need to cretae a deployment so we can create this topology in our CloudStack cloud. To do so, type the following command: 

```
cfy deployments create -b nodecellar1 -d nodecellar1
```

With this command we've created a deployment named `nodecellar1` from a blueprint with the same name. This deployment is not yet materialized, since we haven't issued any command to install it. If you click the "Deployments" icon in the left sidebar in the web UI, you will see that all nodes are labeled with 0/1, which means they weren't yet created. 

## Step 6: Install the Deployment 

In Cloudify, every thing that is executed for a certain deployment is done in the context of a workflow. A workflow is essentially a set of steps, executed by Cloudify agents (which are essentially Celery workers). So whenever a workflow is triggered, it sends a set of tasks to the Cloudify agents, which then execute them and report back the results. For example, the `install` workflows which we're going to trigger, will send tasks to create the various CloudStack resources, and then install and start the application components on them. By default, the Cloudify manager will create one agent per deployment, on the management VM. When application VMs are created by the default `install` workflow (in our case there's two of them), this workflow also installs an agent on each of these VMs, and subsequent tasks to configure these VMs and install application componets are executed by these agents. 
To trigger the `install` workflow, type the following command in your terminal: 

```
cfy deployments execute -d nodecellar1 install
```

These will take a couple of minutes, during which the CloudStack resources and VMs will be create and configured. To track the progress of the installation, you can look at the events emitted to the terminal windows. Each event is labeled with its time, the deployment name and the node in our topology that it relates to, e.g.

```
2014-05-07T12:10:10 CFY <nodecellar1> [exoscale_security_group_1100c] Creating node
```

You can also view the events in the deployment screen in the web UI. 

![Events](https://raw.githubusercontent.com/cloudify-cosmo/cloudify-nodecellar-cloudstack/master/https://raw.githubusercontent.com/cloudify-cosmo/cloudify-nodecellar-cloudstack/master/events.png)

## Step 7: Test Drive the Application 

To test the application, you will need to access it using its public IP address. Locate the VM that runs the nodejs server in your Exoscale dashboard, and use port 8080 to access it from your web browser. You should see the nodecellar application. Click the "Browse wines" button to verify that the application was installed suceesfully and can access the mongodb database to read the list of wines. 

![Nodecellar](https://raw.githubusercontent.com/cloudify-cosmo/cloudify-nodecellar-cloudstack/master/nodecellar.png)

## Step 8: Uninstall the Deployment 

Uninstalling the deployment is just a matter of running another workflow, which will teardown all the resources that were provisionined by the `install` workflow. To run the uninstallation workflow, type the following command: 

```
cfy deployments execute -d nodecellar1 uninstall
```

Similarly to the `install` workflow, you can track the progress of the uninstallation in the CLI or the web UI using the events that are displayed in both. Once the workflow complates, you can verify that the VMs were indeed destroyed and the other application related resources have been also removed. 

## Step 9: Teardown the Manager 

Next, you can also teardown the manager if you have no use for it anymore. This can be done by issuing the following command:

```
cfy teardown -f --ignore-deployments
```

This will terminate the manager VM and delete the resources associated with it. 

## What's Next 

Visit us on the Cloudify community website at [getcloudify.org](http://getcloudify.org), where you'll soon find Cloudify documentation, our mailing lists and other Cloudify goodies. 




