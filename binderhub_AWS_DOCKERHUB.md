# Instructions for Setting up a BinderHub with AWS and DockerHub

This guide roughly follows the [Zero to BinderHub](https://binderhub.readthedocs.io/en/latest/zero-to-binderhub/) documentation. I found that guide vague in some places (for someone unfamiliar with BinderHub such as myself). Notably, the BinderHub team primarily use Google Cloud and there is less support/documentation for AWS (which is what I primarily use) so I decided to write down my workflow here for reproducibility and in case anybody stumbles across this repo looking for help.

[Binderhub](https://github.com/jupyterhub/binderhub) combines JupyterHub (service to spawn single user Jupyter Notebook servers) and Repo2Docker (for building Docker images from Git repositories) to provide on-demand Jupyter notebooks that do not require authentication.

This documentation focuses on using Amazon Elastic Kubernetes Service (EKS) to create and manage the cluster and DockerHub as the container registry. I'd like to use AWS ECR as the container registry but that [PR remains in progress as of April 2021](https://github.com/jupyterhub/binderhub/pull/1055).

To create a BinderHub, follow the steps below.

1. [Create an EC2 Instance](#1)
2. [Create a Cluster with EKS](#2)
3. [Set up BinderHub](#3)
4. [Secure BinderHub](#4)
5. [](#5)
6. [Tearing It All Down](#6)

## 1. Create an EC2 Instance <a name="1"></a>

I used a t2.micro AWS EC2 instance to help set-up and manage my BinderHub (you could do this locally too if you wanted, but I preferred to have a separate machine for this that others could easily manage too). Follow the steps below to set up an instace:

- Log in to the [AWS console](https://aws.amazon.com/) and go to the EC2 service.
- Make sure you're in your desired region (Canada: `ca-central-1` for me) and click "Instances" in the dashboard.
- Click "Launch Instance" and select a suitable AMI (I choose the "Ubuntu Server 18.04 LTS (HVM), SSD Volume Type").
- In Step 2 (choose an instance type), Step 3 (configure instance) and Step 4 (add storage) leave the defaults.
- Add tags in Step 5 if you wish.
- In Step 6 choose "Create a new security group" and add the following rules:
![security](img/security.png)
- Click "Review and Launch", then click "Launch".
- Connect to the instance with SSH once it's ready.
- Once you're in it's a good idea to update ubuntu: `sudo apt-get update; sudo apt-get upgrade`.

## 2. Create a Cluster with EKS <a name="2"></a>

- From within your EC2 instance, you should now go to the [AWS EKS docs](https://docs.aws.amazon.com/eks/latest/userguide/getting-started-eksctl.html) and install `awscli`, `kubectl` and `eksctl` following the documentation. You'll also need to configure the AWS CLI following [these instructions](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-quickstart.html).
- When you get to the section [Create your Amazon EKS cluster and compute](https://docs.aws.amazon.com/eks/latest/userguide/getting-started-eksctl.html#eksctl-create-cluster), we need to do the following steps. These are required because EKS is no longer natively supporting docker, read more [here](https://github.com/weaveworks/eksctl/issues/942#issuecomment-515867547) and the solution comes from [here](https://gist.github.com/tobemedia/2144d74d232ccce8972613e8ae13b054).
  - run `ssh-keygen` and accept defaults to generate a key pair.
  - Create a file `nano aws_eks_config.yaml`
  - Fill it with the following (but feel free to modify things like the instanceType or Size if you want bigger/smaller capacity):

```yml
# file: aws_eks_config.yml
# AWS EKS ClusterConfig used to setup the BinderHub / JupyterNotebooks K8s cluster
# using a workaround from https://discourse.jupyter.org/t/binder-deployed-in-aws-eks-domain-name-resolution-errors/766/10
# to fix broken DNS resolution
--- 
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: binderhub
  region: ca-central-1
  version: '1.19'

nodeGroups:
  - name: nodes
    instanceType: m5.2xlarge
    minSize: 1
    maxSize: 4
    desiredCapacity: 1
    preBootstrapCommands:
        # Replicate what --enable-docker-bridge does in /etc/eks/bootstrap.sh
        # Enabling the docker bridge network. We have to disable live-restore as it
        # prevents docker from recreating the default bridge network on restart
       - "cp /etc/docker/daemon.json /etc/docker/daemon_backup.json"
       - "echo -e '.bridge=\"docker0\" | .\"live-restore\"=false' >  /etc/docker/jq_script"
       - "jq -f /etc/docker/jq_script /etc/docker/daemon_backup.json | tee /etc/docker/daemon.json"
       - "systemctl restart docker"
```
  - Then run `eksctl create cluster --config-file aws_eks_config.yaml` to create the cluster
  - The cluster usually takes ~15 mins to build, check it has finished by running `kubectl get svc`

>Note that it takes some trial-and-error to select the `instanceType`. You can easily scale the cluster up or down with `eksctl` but unfortunately it's difficult to change the `instanceType` after the cluster is created. For some context one m5.2xlarge was suitable for managing a class of ~30 students doing data science (mostly using `pandas` and `scikit-learn`). One node could handle up to 60 students but it was slow, adding two nodes significantly improved speed in this situation. [This guide](https://tljh.jupyter.org/en/latest/howto/admin/resource-estimation.html) in the Littlest JupyterHub documentation may also be helpful.

## 3. Set Up BinderHub <a name="3"></a>

- From here, just follow all the steps in the Zero-to-Binderhub guide starting from [1.2. Installing Helm](https://binderhub.readthedocs.io/en/latest/zero-to-binderhub/setup-prerequisites.html#installing-helm), and using DockerHub as the container registry.
- **WARNING**: when you get to [3.5. Connect BinderHub and JupyterHub](https://binderhub.readthedocs.io/en/latest/zero-to-binderhub/setup-binderhub.html#connect-binderhub-and-jupyterhub), you'll run the command `kubectl --namespace=<namespace-from-above> get svc proxy-public` to retrieve the JupyterHub "EXTERNAL-IP" address. If you paste this url into a browser. It should display a simple JupyterHub "403 : Forbidden" page like below:
![jupyterhub](img/jupyterhub.png)
- If it doesn't and shows something like the "This site can't be reached" error below, you'll need to patch the LoadBalancer's port forwarding. To do that, go to EC2 in the AWS web console -> Click Load Balancers in the side menu -> Find the load balancer, click on it and check the "Instances" tab -> you'll notice that the status of the Instance ID's is "OutOfService" -> now go to the "Description" tab -> Scroll down to the "Port Configuration" header and note that 80 TCP port, something like "*80 (TCP) forwarding to 32413 (TCP)*" -> Click “Health Check” tab -> Click "Edit Health Check" -> Change the "Ping Port" to the port that 80 (TCP) is being forwarded to, e.g., 32413.
![error](img/error.png)
- Continue on with final steps in the Z2BH guide and you'll have you're own Binderhub:
![binder_insecure](img/binder_insecure.png)
- At the moment this binder is insecure, it works, but you may run into security issues down the line. In the next step we'll secure it for use with HTTPS traffic.

## 4. Secure BinderHub <a name="4"></a>

- We can secure our Binder by following the steps in [the BinderHub docs](https://binderhub.readthedocs.io/en/latest/https.html#secure-with-https). However, these steps are currently (April 2021) specific to Google Cloud and there are some tricks to get it to work on AWS. Read on.
- The first thing you need to do is buy a domain name to serve your binder from, e.g., you might buy the domain `example.com`, and you can then serve your binder at `binder.example.com`. Buy the domain now and we'll use it a little later on. I've tried this with namecheap and Google Domains.
- Next, install `cert-manager` which will manage our TLS certificate. [Installation docs are here](https://cert-manager.io/docs/installation/kubernetes/).
- Now, create a file `binderhub-issuer.yaml` with the following contents:
```yml
apiVersion: cert-manager.io/v1alpha2
kind: Issuer
metadata:
  name: letsencrypt-production
  namespace: binderhub # you can view namespace with: kubectl get svc --all-namespaces
spec:
  acme:
    # You must replace this email address with your own.
    # Let's Encrypt will use this to contact you about expiring
    # certificates, and issues related to your account.
    email: <your-email-address>
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      # Secret resource used to store the account's private key.
      name: letsencrypt-production
    solvers:
    - http01:
        ingress:
          class: nginx
```
- Run `kubectl apply -f binderhub-issuer.yaml` to apply this configuration
- Now we want to create an nginx-based LoadBalancer on AWS. I followed the [Kubernetes Ingress Nginx documentation](https://kubernetes.github.io/ingress-nginx/deploy/#aws) to do this which just required running: `kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.44.0/deploy/static/provider/aws/deploy.yaml`
- Get the external IP of the newly created LoadBalancer: `kubectl -n ingress-nginx get svc ingress-nginx-controller`, think of this as the entry point to our cluster.
- Now we need to create CNAMES in our domain's DNS configuration our Binder and backing JupyterHub that point to this LoadBalancer's IP. How this is done will depend on the provider but you just need to add a new CNAME record, give it a name (any name you want but I recommend, `binder` for binder and `hub.binder` for jupyterhub). When we finish this section, your binder will be available at `binder.<your-domain-name>.com`. Here's an example:
![cnames](img/cnames.png)
 - Now we just need to adjust our existing `nano config.yaml` (in the `binderhub` directory) to account for our new domains and loadbalancer. Also note the inclusion of the `cors` headers which helps by-pass some browser security issues (this took me forever to figure out but now I added a section to the [Zero to Binderhub docs](https://binderhub.readthedocs.io/en/latest/cors.html) on this process):
```yaml
  config:
    BinderHub:
      hub_url: https://<jupyterhub-URL> # e.g. https://hub.binder.example.com
  
  cors: &cors
    allowOrigin: '*'

  jupyterhub:
    custom:
      cors: *cors
    ingress:
      enabled: true
      hosts:
        - <jupyterhub-URL> # e.g. hub.binder.example.com
      annotations:
        kubernetes.io/ingress.class: nginx
        kubernetes.io/tls-acme: "true"
        cert-manager.io/issuer: letsencrypt-production
        https:
          enabled: true
          type: nginx
      tls:
        - secretName: <jupyterhub-URL-with-dashes-instead-of-dots>-tls # e.g. hub-binder-example-com-tls
          hosts:
            - <jupyterhub-URL> # e.g. hub.binder.example.com

  ingress:
    enabled: true
    hosts:
      - <binderhub-URL> # e.g. binder.example.com
    annotations:
      kubernetes.io/ingress.class: nginx
      kubernetes.io/tls-acme: "true"
      cert-manager.io/issuer: letsencrypt-production
      https:
        enabled: true
        type: nginx
    tls:
      - secretName: <binderhub-URL-with-dashes-instead-of-dots>-tls  # e.g. binder-example-com-tls
        hosts:
          - <binderhub-URL> # e.g. binder.example.com
```
- Now just apply the new `config.yaml` and we are done! `helm upgrade binderhub jupyterhub/binderhub --version=0.2.0-n219.hbc17443 -f secret.yaml -f config.yaml`
- Secure!
![secure](img/binder_secure.png)

## 5. Customization <a name="5"></a>

- There's a variety of settings you can configure to customize your BinderHub, for example, to ban or to allocate more resources to specific repositories. These are all documented in the [BinderHub docs](https://binderhub.readthedocs.io/en/latest/customization/index.html).
- One setting I recommend configuring is adding a GitHub API token (by default GitHub only lets you make 60 requests each hour. If you expect more users than this, you should create an API access token to raise your API limit to 5000 requests an hour). This is explained in the [BinderHub docs here](https://binderhub.readthedocs.io/en/latest/zero-to-binderhub/setup-binderhub.html#increase-your-github-api-limit), but briefly, you should create a [new token](https://github.com/settings/tokens/new) with default permissions, then add the following to `secret.yaml`:
```yaml
GitHubRepoProvider:
  access_token: b9cc23fef2072c264d9fadd06adb757808766431
```
- I also found it useful to specify the culling behaviour of the hub (the deletion of inactive pods/users). By default the culling process runs every ten minutes and basically culls any user pods that have been inactive for more than one hour. While this is a defautl setting, I like to explicitly include it in `config.yaml` for clarity. Also, “inactivity” is defined as no response from the user’s browser, but sometimes a user will not be using their computer but will leave their browser open. I therefore like to also impose a max age on a pod to delete pods in this situation. To do this, add the following to `config.yaml`:

```yaml
jupyterhub:
  cull:
    enabled: true
    # maxAge is 5 hours: 5 * 3600 = 18000
    maxAge: 18000
```

## 6. Tearing It All Down <a name="6"></a>

- Follow the instructions in the [AWS docs](https://docs.aws.amazon.com/eks/latest/userguide/delete-cluster.html).

```bash
kubectl get svc --all-namespaces
kubectl delete namespaces <namespace_to_delete>
eksctl delete cluster --name <deployment_name>
```