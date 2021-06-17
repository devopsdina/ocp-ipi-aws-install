# ocp-ipi-aws-install

Example GitHub Actions workflows to:

- Deploy an OpenShift Cluster configured for Windows Containers, including the WMCO, Windows MachineSet and Windows container.
- Configuring the final leg of the SSL certificate to remove the website warning, using certbot and route53.
- Removing the kubeadmin user.
- Removing the OpenShift Cluster from AWS, with the metadata files created during deployment.
- Force removing the OpenShift Cluster from AWS, without any metadata files.

S3 is used for file storage of the installer (which can be sourced from cloud.redhat.com), along with the pull secret.

S3 is used to store metadata for each run of the `deploy-openshift` workflow, assuming the installer completes successfully.

**DISCLAIMER**
This repo is not meant to demonstrate best practices or extensibility.  This repo is meant to show a low barrier to entry for automation in regards to deploying OpenShift with Windows Container support in AWS.

## Workflows
### `deploy-openshift.yml`

- The installer is run in multiple steps to allow for creation of the install-config and the `cluster-network-03-config.yaml` file which is required for windows

- SSH keys are generated during each run for both the cluster and the WMCO (Windows Machine Config Operator).  All keys are sent to the S3 storage bucket.

- A Windows MachineSet is deployed using the public AWS ami: `ami-015451c8bbe8e7650`.  This will result in a `Windows_Server-2019-English-Full-ContainersLatest-2021.04.14` host machine.

- The OpenShift cluster certificate, by default will encrypt traffic.  However it is not signed by a CA so it will appear insecure in the browser.  If you check the cert on [SSL checker](https://www.sslshopper.com/ssl-checker.html), it will show secure until the very end of the chain by default.  This job will finish securing the certificate by using Let's Encrypt via the route53 plugin.

- A demo IIS container is deployed

#### Requirements for `deploy-openshift.yml`

An AWS S3 Storage bucket with the following:

- S3 bucket with permissions set accordingly
- The OpenShift installer in a folder called `macos-installer`
- The _matching_ pull secret for the installer in a folder called `ocp-install-configs`

**The following secrets are set:**

Used to login to AWS:
- AWS_ACCESS_KEY_ID
- AWS_SECRET_ACCESS_KEY

Used for OpenShift configuration

- BASEDOMAIN
    - The base domain, usually your organizations or personal one.  Example: test.mycompany.com
- CIDRCLUSTERNETWORK
- CIDRMACHINENETWORK
- SERVICENETWORK

Used for certificates and tags:

- EMAIL

Used for OCP configuration(s) post deployment:

- OC_USER
- OC_PASSWORD

### `configure-ssl-cert.yml`

**This job still needs to be tested**

Assuming the cluster is in AWS, using route 53 this job will use certbot + let's encrypt to act as a CA to sign the certificate.  The certificate that is provisioned with OpenShift, while it is encrypted, it will show "un-secure" on a browser until signed by a CA due to how the certificate chain works.

#### Requirements for `configure-ssl-cert.yml`

An AWS S3 Storage bucket with the following:

- S3 bucket with permissions set accordingly
- The OpenShift metadata files

The following secrets are used in this workflow:

_Used to login to AWS:_
- AWS_ACCESS_KEY_ID
- AWS_SECRET_ACCESS_KEY

_Used for certificate configuration_
- BASEDOMAIN
    - The base domain, usually your organizations or personal one.  Example: test.mycompany.com
- EMAIL

_Used for oc login to apply certs_:
- OC_USER
- OC_PASSWORD
### `remove-kubeadmin-user`

Removes the kubeadmin user, sets htpasswd as oauth and uses the OC_USER and OC_PASSWORD secrets to configure a new user.

**This job still needs to be tested**

The following secrets are used in this workflow:
- AWS_ACCESS_KEY_ID
- AWS_SECRET_ACCESS_KEY
- BASEDOMAIN
    - The base domain, usually your organizations or personal one.  Example: test.mycompany.com
- OC_INIT_USER
    - existing user (most likely kubeadmin)
- OC_INIT_PASSWORD
    - existing password
- OC_USER
    - New user to be configured
- OC_PASSWORD
    - Password for new user

### `remove-cluster`

This workflow will destroy the OpenShift cluster.  This workflow assumes you have the metadata files from the original deployment in an S3 bucket.

The following secrets are used in this workflow:
_required to pull the OpenShift installer and corresponding metadata files_
- AWS_ACCESS_KEY_ID
- AWS_SECRET_ACCESS_KEY

### `force-remove-cluster`

This workflow will destroy the OpenShift cluster.  This workflow does not rely on any metadata from the original deployment.  It is a destroy hack!

The following secrets are used in this workflow:
_required to pull the OpenShift installer_
- AWS_ACCESS_KEY_ID
- AWS_SECRET_ACCESS_KEY

### `prepull-windows-image`

This workflow will prepull the Windows container image on the MachineSet.  Until the timeout is increased to 30 minutes for pulling an image, all Windows images need to be pull in advance of the actual container deployment

This workflow assumes you have the metadata from the install and the ssh key used to configure the WMCO in S3 storage.

The following secrets are used in this workflow:
_required to pull the OpenShift metadata_
- AWS_ACCESS_KEY_ID
- AWS_SECRET_ACCESS_KEY

### `deploy-netcandystore`

Deploy the [NetCandy Store](http://people.redhat.com/chernand/windows-containers-quickstart/ns-intro/) a mixed environment consisting of Windows Containers and Linux Container using helm.

This application consists of:

- Windows Container running a .NET v4 frontend, which is consuming a backend service.
- Linux Container running a .NET Core backend service, which is using a database.
- Linux Container running a MSSql database.

The following secrets are used in this workflow:
- AWS_ACCESS_KEY_ID
- AWS_SECRET_ACCESS_KEY
- OC_USER
- OC_PASSWORD