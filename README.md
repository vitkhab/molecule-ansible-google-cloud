# molecule-ansible-aws-gcp-azure
[![Build Status](https://travis-ci.org/jonashackt/molecule-ansible-aws-gcp-azure.svg?branch=master)](https://travis-ci.org/jonashackt/molecule-ansible-aws-gcp-azure)
[![versionansible](https://img.shields.io/badge/ansible-2.7.8-brightgreen.svg)](https://docs.ansible.com/ansible/latest/index.html)
[![versionmolecule](https://img.shields.io/badge/molecule-2.19.0-brightgreen.svg)](https://molecule.readthedocs.io/en/latest/)
[![versiontestinfra](https://img.shields.io/badge/testinfra-1.16.0-brightgreen.svg)](https://testinfra.readthedocs.io/en/latest/)
[![versionawscli](https://img.shields.io/badge/awscli-1.16.110-brightgreen.svg)](https://aws.amazon.com/cli/)
[![versionazurecli](https://img.shields.io/badge/azurecli-2.0.59-brightgreen.svg)](https://aws.amazon.com/cli/)

Example projects showing how to do test-driven development of Ansible roles and running those tests on multiple Cloud providers at the same time

This project build on top of [molecule-ansible-docker-vagrant](https://github.com/jonashackt/molecule-ansible-docker-vagrant), where all the basics on how to do test-driven development of Ansible roles with Molecule is described. Have a look into the blog series so far:

* [Test-driven infrastructure development with Ansible & Molecule](https://blog.codecentric.de/en/2018/12/test-driven-infrastructure-ansible-molecule/)
* [Continuous Infrastructure with Ansible, Molecule & TravisCI](https://blog.codecentric.de/en/2018/12/test-driven-infrastructure-ansible-molecule/)
* [Continuous cloud infrastructure with Ansible, Molecule & TravisCI on AWS](https://blog.codecentric.de/en/2019/01/ansible-molecule-travisci-aws/)

## What about Multicloud?

Developing infrastructure code according to prinicples like test-driven development and continuous integration is really great! But what about pushing this to the next level? As [Molecule](https://molecule.readthedocs.io/en/latest/) is able to handle everything Ansible is albe to access, why not run our test automatically on all major cloud platforms at the same time?

With this, we would not only have a security net for our infrastructure code, but would also be safe regarding a switch of our current cloud or data center provider. Lot's of people talk about the unclear costs of this switch. **** If our infrastructure code would be able to run on every cloud platform possible, we would simply be able to switch to whatever platform we want - and all with just the virtually no expenses.Why not just reduce these to zero?!


## A selection of cloud providers: Azure, Google, AWS

So let's pick some more providers so that we can safely speak about going 'Multicloud':

* [Molecule's Azure driver](https://molecule.readthedocs.io/en/latest/configuration.html#azure)
* [Molecule's Google Compute Engine (GCE) driver](https://molecule.readthedocs.io/en/latest/configuration.html#gce)
* We already know [how to use Molecule with AWS EC2](https://blog.codecentric.de/en/2019/01/ansible-molecule-travisci-aws/). 


Let's start with AWS by just forking [molecule-ansible-docker-vagrant](https://github.com/jonashackt/molecule-ansible-docker-vagrant), since there should be mostly everything needed to use Molecule with AWS.

This should run in no time :)


## Add Google Cloud Platform to the game

First you'll need a valid [Google Cloud Platform](https://cloud.google.com) account - you should have at least 300$ using a test account for free.

Then we need to install Google Compute Engine (GCE) support for Molecule:

```
pip3 install molecule[gce]
```

Now let's initialize a new Molecule scenario calles `gcp-gce-ubuntu` inside our Ansible role:

```
cd molecule-ansible-aws-gcp-azure/docker

molecule init scenario --driver-name gce --role-name docker --scenario-name gcp-gce-ubuntu
```

That should create a new directory `gcp-gce-ubuntu` inside the `docker/molecule` folder.  We'll integrate the results into our multi scenario project in a second.

Now let's dig into the generated [molecule.yml](docker/molecule/gcp-gce-ubuntu/molecule.yml):

```yaml
s---
 scenario:
   name: gcp-gce-ubuntu
 
 driver:
   name: gce
 platforms:
   - name: gcp-gce-ubuntu
     zone: europe-west3
     machine_type: f1-micro
     image: ubuntu-1804-lts
 
 provisioner:
   name: ansible
   lint:
     name: ansible-lint
     enabled: false
   playbooks:
     converge: ../playbook.yml
 
 lint:
   name: yamllint
   enabled: false
 
 verifier:
   name: testinfra
   directory: ../tests/
   env:
     # get rid of the DeprecationWarning messages of third-party libs,
     # see https://docs.pytest.org/en/latest/warnings.html#deprecationwarning-and-pendingdeprecationwarning
     PYTHONWARNINGS: "ignore:.*U.*mode is deprecated:DeprecationWarning"
   lint:
     name: flake8
   options:
     # show which tests where executed in test output
     v: 1
```

As we already tuned the `molecule.yml` files for our other scenarios like `aws-ec2-ubuntu`, we know what to change here. `provisioner.playbook.converge` needs to be configured, so the one `playbook.yml` could be found.

Also the `verifier` section has to be enhanced to gain all the described advantages like supressed deprecation warnings and the better test result overview.

As you may noticed, the driver now uses `gce` and the platform is already pre-configured with a concrete `zone`, `machine_type` and a Google Compute Engine image. Here we just tune the instance name to `gcp-gce-ubuntu` and the `zone` according to our preferred region (see [region list here](https://cloud.google.com/compute/docs/regions-zones/?hl=en)).

Let's also configure a suitable image (see [the Image list here](https://cloud.google.com/compute/docs/images?hl=en)) - for us using our "Install Docker on Ubuntu use case", we should choose `ubuntu-1804-lts`. The preconfigure [Machine Type](https://cloud.google.com/compute/docs/machine-types?hl=en) `f1-micro` should suffice for us.


### Install gcloud & apache-libcloud

We need to have `gcloud cli` installed, which is packaged with the Google Cloud SDK. BUT don't install it this way, again use Python package manager pip instead:

```
pip3 install gcloud apache-libcloud
```

We also need to install [Apache Libcloud](https://libcloud.apache.org/), so it's already attached to the pip install command. Libcloud is used to interact with Google Compute Engine by Molecule.


### Create a Service Account inside GCE & configure Apache Libcloud

As [described in the docs](https://libcloud.readthedocs.io/en/latest/compute/drivers/gce.html#connecting-to-google-compute-engine) we need to [create a Service account](https://libcloud.readthedocs.io/en/latest/compute/drivers/gce.html#service-account) inside our Google Cloud Console:

> Select the existing or newly created project and go to IAM & Admin -> Service Accounts -> Create service account to create a new service account. 

Provide the service account with a speaking name like `libcloud`, then click __NEXT__. Grant the service account the `Owner` role and again click __NEXT__.

Select the role `Owner` and at the tab `Grant users access to this service account (optional)` you should click on __create key__ to create and download new private key you will use to authenticate (I went with the `.json` format).

At the end you're service account should be listed inside your projects settings:

![google-cloud-service-account](screenshots/google-cloud-service-account.png)



### Creating a Google Compute Engine instance with Molecule


Now we should have everything prepared. Let's try to run our first Molecule test on Google Compute Engine (including `--debug` so that we see what's going on):

```
molecule --debug create --scenario-name gcp-gce-ubuntu
```


## Add Azure to the party



