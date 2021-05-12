# ocp-ipi-aws-install

Example GitHub Actions workflow to deploy OpenShift Cluster in AWS with networking updated to support Windows Containers.

Initial configuration for the cluster is pulled down from S3 storage, along with the installer.

The metadata and files from the deployment are uploaded to s3 to allow for easier deletion of the cluster.