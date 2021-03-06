# Instructions for Setting up a BinderHub with AWS and ECR

This guide roughly follows the [Zero to BinderHub](https://binderhub.readthedocs.io/en/latest/zero-to-binderhub/) documentation. I found that guide vague in some places (for someone unfamiliar with BinderHub such as myself). Notably, the BinderHub team primarily use Google Cloud and there is less support/documentation for AWS (which is what I primarily use) so I decided to write down my workflow here for reproducibility and in case anybody stumbles across this repo looking for help.

Recall that [Binderhub](https://github.com/jupyterhub/binderhub) combines JupyterHub (spawning single user Jupyter Notebook servers) and Repo2Docker (for building Docker images from Git repositories) to provide on-demand Jupyter notebooks that do not require authentication.

This documentation focuses on using Amazon Elastic Kubernetes Service (EKS) to create and manage the cluster and Amazon Elastic Container Registry (ECR) as a container registry.

This documentation is a WIP - check back soon!