# What

This will deploy one or more clusters in the cloud, with optional post-install tasks defined by template.

# Supported platforms

## Container
 * Kubernetes
 * Openshift 3.11

## Cloud
 * AWS
 * GCP

1. Install the CLI for your choice of cloud provider:
 * AWS: https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-configure.html
 * GCP: https://cloud.google.com/sdk/docs/quickstarts

2. Install [Vagrant](https://www.vagrantup.com/downloads.html).

3. Install the Vagrant plugin for your choice of cloud provider:
 * AWS: `vagrant plugin install vagrant-aws`
 * GCP: `vagrant plugin install vagrant-google`

Note: For AWS, also need to install a dummy box:
```
vagrant box add dummy https://github.com/mitchellh/vagrant-aws/raw/master/dummy.box
```

4. Clone this repo and cd to it.

5. If running on macOS, install GNU Getopt and Coreutils:
```
brew install gnu-getopt coreutils
echo 'export PATH="/usr/local/opt/gnu-getopt/bin:$PATH"' >> ~/.bash_profile
```

6. Create an environment (in AWS by default):
```
./deploy.sh --init=myDeployment --aws_keypair=CHANGEME
```
This will provision a VPC and some other objects. Be sure to update the name of the keypair.

7. Provision a deployment in the environment:
```
./deploy.sh --env=myDeployment --template=clusterpair
```

8. Connect via SSH:
```
./deploy.sh --env=myDeployment --ssh
```

9. Tear down the deployment:
```
./deploy.sh --env=myDeployment --destroy
```

10. Tear down the environment:
```
./deploy.sh --uninit=myDeployment
```

# DESIGN

Before deploying anything, an environment must be created. For example:
```
./deploy.sh --aws_keypair=CHANGEME --init=foo
./deploy.sh --gcp_keyfile=gcp-key.json --cloud=gcp --init=bar
```
If using GCP, download the JSON service account key after creating the deployment, and copy to the path specified with the --gcp-keyfile parameter:
 * On GCP console, select the Project, click APIs and Services, Credentials, Create Credentials, Service account key, Create. Save the file.

The environments can be listed:
```
$ ./deploy.sh --envs
Environment  Cloud  Created
foo          gcp    2020-01-21 16:25:26
bar          aws    2020-01-21 15:33:25
```

This has provisioned an AWS VPC and a GCP project, along with required networking objects and rules.

Now a deployment can be deployed to an environment.

The `deploy.sh` script sets a number of environment variables:
 * `AWS_EBS` - a list of EBS volumes to be attached to each worker node. This is a space-separate list of type:size pairs, for example: `"gp2:30 standard:20"` will provision a gp2 volume of 30 GB and a standard volume of 20GB
 * `AWS_TYPE` - the AWS machine type for each node
 * `INIT_CLOUD` - the cloud on which to deploy (aws or gcp)
 * `DEP_CLUSTERS` - the number of clusters to deploy
 * `DEP_K8S_VERSION` - the version of Kubernetes to deploy
 * `DEP_NODES` - the number of worker nodes on each cluster
 * `DEP_PLATFORM` - can be set to either k8s or ocp3
 * `DEP_PX_VERSION` - the version of Portworx to install
 * `GCP_DISKS` - similar to AWS_EBS, for example: `"pd-standard:20 pd-ssd:30"`
 * `GCP_TYPE` - the GCP machine type for each node

The defaults are defined in the script.

There are two ways to override these variables. The first is to specify a template with the `--template=...` parameter. For example:
```
$ cat templates/clusterpair
DEP_CLUSTERS=2
DEP_PX_VERSION=2.3.2
DEP_PX_CLUSTER_PREFIX=px-deploy
DEP_SCRIPTS="install-px clusterpair"
```

More on DEP_SCRIPTS below.

The second way to override the defaults is to specify on the command line. See `./deploy -h` for a full list. For example, to deploy into the `foo` environment:
```
./deploy --env=foo --clusters=5 --template=petclinic --nodes=6
```

This example is a mixture of both methods. The template is applied, then the command line parameters are applied, so not only is the template overriding the defaults, but also the parameters are overriding the template.

`DEP_SCRIPTS` is a list of scripts to be executed on each master node. For example:
```
$ cat scripts/clusterpair
(
if [ $cluster != 1 ]; then
  while : ; do
    token=$(ssh -oConnectTimeout=1 -oStrictHostKeyChecking=no node-$cluster-1 pxctl cluster token show 2>/dev/null | cut -f 3 -d " ")
    echo $token | grep -Eq '\w{128}'
    [ $? -eq 0 ] && break
    sleep 5
    echo waiting for portworx
  done
  storkctl generate clusterpair -n default remotecluster-$cluster | sed "/insert_storage_options_here/c\    ip: node-$cluster-1\n    token: $token" >/var/tmp/cp.yaml
  while : ; do
    cat /var/tmp/cp.yaml | ssh -oConnectTimeout=1 -oStrictHostKeyChecking=no master-1 kubectl apply -f -
    [ $? -eq 0 ] && break
    sleep 5
  done
fi
) >&/var/log/vagrant-clusterpair
```

All of the variables above are passed to the script. In addition to these, there are some more variables available:
 * `$cluster` - cluster number
 * `$script` - filename of the script

# BUGS
 * When destroying the clusters as above, it uses the default number of clusters and nodes, so will only destroy master-1, node-1-1, node-1-2 and node-1-3, unless --clusters and --nodes are specified.
 * AWS: When specifying the region, it must match the region set in `$HOME/.aws/config` until https://github.com/mitchellh/vagrant-aws/pull/564 is merged.
 * Can the GCP JSON key key downloaded automatically?
