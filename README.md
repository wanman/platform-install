# CORD platform-install

This repository contains [Ansible](http://docs.ansible.com) playbooks for
installing and configuring software components that build a CORD POD:
OpenStack, ONOS, and XOS.

It is used as a sub-module of the [main CORD
repository](https://github.com/opencord/cord), but can independently bring up
various CORD profiles for development work.

If you want to set up an entire CORD pod on physical hardware, or set up the
Cord-in-a-Box deployment, you should start at the [CORD
repository](https://github.com/opencord/cord).

## Using platform-install for development

### Bootstrapping your development environment

There's a helper script,
[scripts/cord-bootstrap.sh](https://github.com/opencord/platform-install/blob/master/scripts/cord-bootstrap.sh).
that will install development environment prerequisites on a Ubuntu 14.04 node.
You can download it with:

```
curl -o ~/cord-bootstrap.sh https://raw.githubusercontent.com/opencord/platform-install/master/scripts/cord-bootstrap.sh
```

Running the script will install the [repo](https://code.google.com/p/git-repo/)
tool, [Ansible](https://docs.ansible.com/ansible/index.html), and
[Docker](https://www.docker.com/), as well as make a checkout of the CORD
[manifest](https://gerrit.opencord.org/gitweb?p=manifest.git;a=blob;f=default.xml)
into `~/cord`.

You can specify which gerrit changesets you would like repo to checkout using
the `-b` option on the script [as documented
here](https://github.com/opencord/cord/blob/master/docs/quickstart.md#using-cord-in-a-boxsh-to-download-development-code-from-gerrit).

Once you have done this, if you're not already in the `docker` group, you
should logout and log back into your system to refresh your user account's
group membership. If you don't do this, any docker command you run or ansible
runs for you will fail. You can check your group membership by running
`groups`.

Once you log back in, you may want to run `tmux` to maintain a server-side
session you can reconnect to, in case of network trouble.

All of the commands below assume you're in the `cord/build/platform-install`
directory.

### Creating a development environment on you machine

If you are doing work that does not involve `openstack` you can create a
development environment inside a VM on your local machine. This environment is
mostly designed to do `GUI`, `APIs` and `modeling` related work. It can also be
useful to test a `synchronizer` whose job is synchronizing data using REST
APIs.

To do that we provided a `Vagrant` VM.  From this folder just execute `vagrant
up head-node`. We'll create an Ubuntu based VM with all the `cord` code shared
from your local machine, so that you can make your changes locally and quickly
test the outcome.

Once the `vm` is created you can connect to it with `vagrant ssh head-node` and
then from the `~/cord/build/platform-install` execute the profile you are
interested in.  For instance you can spin up the `frontend` configuration with:
`ansible-playbook -i inventory/frontend deploy-xos-playbook.yml`.

_Note that the `cord-bootstrap.sh` script is automatically invoked by the
provisioning script and the `vagrant` vm requires VirtualBox_

### Credentials

Credentials will be autogenerated and placed in the `credentials/` directory
when the playbooks are run, where the credential name is the filename, and the
contents of the file is the password.

For most profiles the XOS admin user is named `xosadmin@opencord.org`.

### Development Loop

Most profiles are run by specifying an inventory file when running
`ansible-playbook`. For most frontend or mock profiles, you'll want to run the
`deploy-xos-playbook.yml` playbook.

For example, to run the `frontend` config, you would run:

```
ansible-playbook -i inventory/frontend deploy-xos-playbook.yml
```

Assuming it runs without error, you can then explore the environment you've set
up.  When you're ready to tear down your environment, run:

```
ansible-playbook -i inventory/frontend teardown-playbook.yml
```

This will destroy all the docker containers created.  Note that you must run the
`teardown-playbook.yml` using the same inventory file it was created with,
otherwise rogue docker containers may still be running after the teardown.

You can then make changes to code and re-run the same or a different profile.

#### Re-building Containers

By default, container images are not rebuilt if they already exist in the development
environment.  (NOTE: The `xos_core` container is an exception; it is always rebuilt.)
To force the development loop to rebuild and redeploy specific containers, add
`rebuild: true` in the appropriate place as described below.

For service synchronizers, add `rebuild: true` to the service's entry in the `xos_services`
list in the profile manifest.

For GUI extensions, add `rebuild: true` to the extension's entry in the `enabled_gui_extensions`
list in the profile manifest.

For all other containers, add `rebuild: true` to the container's entry in the `docker_images`
list in the XOS repo's `group_vars/all` file.

### Creating a new CORD profile

To create a new CORD profile, you should:

1. Create an inventory file in `inventory/` that defines the `cord_profile`
   variable, with the name of your profile.

```
[all:vars]
cord_profile=my-profile
```

2. Create a .yaml variables file in `profile_manifests/` with the name of your
   profile (ex: `my-profile.yaml`), and populate it with your configuration.

3. To test the profile, run the `deploy-xos-playbook.yml` playbook using your
   inventory profile:
   `ansible-playbook -i inventory/my-profile deploy-xos-playbook.yml`

### Making changes and lint checking your changes

Before commit, please run `./scripts/lintcheck.sh .` in the repo root, which
will perform the same [ansible-lint](https://pypi.python.org/pypi/ansible-lint)
check that Jenkins performs when in review in gerrit.

## Specific profiles notes

### api-test

This profile runs API tests for both the REST and TOSCA APIs. This can be done
in an automated fashion:

`ansible-playbook -i inventory/api-test api-test-playbook.yml`

The XOS credentials in this config are `padmin@vicci.org` and `letmein` (until
the tests are modified to support generated credentials).

### frontend

Builds a basic XOS frontend installation, useful for UI testing and
experimentation.

### mock-rcord, mock-mcord

Builds a Mock R-CORD or Mock M-CORD pod, without running service synchronizers
in a manner similar to frontend.

### opencloud

Used as a part of the [OpenCloud](http://www.opencloud.us/) deployment. Similar
to `rcord`.

### rcord

This is a part of the [R-CORD](https://github.com/opencord/cord) deployment -
start using the steps specified in that repo.

This profile is designed to integrate XOS with physical infrastructure pieces
like MaaS, OpenStack, and ONOS.  See the [CORD-in-a-Box Quick Start
Guide](https://github.com/opencord/cord/blob/master/docs/quickstart.md) for how
to set up a virtual multi-node R-CORD pod on a single host.

If you've already built a CiaB and want to go through the dev loop described
above on the head/production node, when running `ansible-playbook` you must
pass the path to the configuration file that was generated by Gradle during the
build:

```
ansible-playbook -i inventory/head-localhost --extra-vars @../genconfig/config.yml playbook.yml
```

You may find the following shell aliases to be helpful during development:

```
alias xos-teardown="pushd /opt/cord/build/platform-install; ansible-playbook -i inventory/head-localhost --extra-vars @/opt/cord/build/genconfig/config.yml teardown-playbook.yml"
alias xos-launch="pushd /opt/cord/build/platform-install; ansible-playbook -i inventory/head-localhost --extra-vars @/opt/cord/build/genconfig/config.yml launch-xos-playbook.yml"
alias compute-node-refresh="pushd /opt/cord/build/platform-install; ansible-playbook -i /etc/maas/ansible/pod-inventory --extra-vars=@/opt/cord/build/genconfig/config.yml compute-node-refresh-playbook.yml"
```

With these aliases, tearing down XOS and rebuilding it from a fresh database becomes:

```
xos-teardown; xos-launch; compute-node-refresh
```


