# airflow-in-eks-docs
Guide on how to setup eks, fluxcd and airflow in aws

# Repos:
```
airflow dags:
https://github.com/EdwinGuo/airflow-dags

cluster deployment configs:
https://github.com/EdwinGuo/airflow-eks-config

helm charts for airflow
https://github.com/EdwinGuo/airflow-eks-helm-chart
```


# Setup Workstation
```
# Open up a standard AWS AMI ec2 instance, from size perspective, micro or small is enough for the project.

# Update packages of the instance
sudo yum -y update

# Create a python virtual environment
python3 -m venv .sandbox

# Active the python virtual environment
source .sandbox/bin/activate

# Upgrade pip
pip install --upgrade pip

# Download and extract the latest release of eksctl with the following command.
curl --silent --location "https://github.com/weaveworks/eksctl/releases/download/0.49.0/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp

# Move the extracted binary to /usr/local/bin.
sudo mv /tmp/eksctl /usr/local/bin

# Test that your installation was successful with the following command.
eksctl version

# Download the latest release of Kubectl with the command
curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.18.2/bin/linux/amd64/kubectl

# Make the kubectl binary executable.
chmod +x ./kubectl

# Move the binary in to your PATH.
sudo mv ./kubectl /usr/local/bin/kubectl

# Test to ensure the version you installed is up-to-date:
kubectl version --client

# Install Helm3
curl -sSL https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash

# Check the version
helm version --short

# Download the stable repo
helm repo add stable https://charts.helm.sh/stable

################################# EOV

# upgrade aws cli
pip install --upgrade awscli && hash -r

# install some utilities
sudo yum -y install jq gettext bash-completion moreutils

# go the settings, AWS settings and turn off temporary credentials

# remove temporary credentials
rm -vf ${HOME}/.aws/credentials

# configure aws env variables
# The following get-caller-identity example displays information about the IAM identity used to authenticate the request
aws configure
aws sts get-caller-identity
export ACCOUNT_ID=
export AWS_REGION=

# update the file bash_profile and configure aws
echo "export ACCOUNT_ID=${ACCOUNT_ID}" | tee -a ~/.bash_profile
echo "export AWS_REGION=${AWS_REGION}" | tee -a ~/.bash_profile
aws configure set default.region ${AWS_REGION}
aws configure get default.region

################################# EOV

# Press return for all questions by keeping the defaults and empty passphrase.
ssh-keygen -t rsa

################################# EOV

cd airflow-materials-aws

# Create the cluster
eksctl create cluster -f cluster.yml

# Check if the cluster is healthy
kubectl get nodes
kubectl get pods --all-namespaces

################################# EOV

# Installing Flux
# create the flux Kubernetes namespace
kubectl create namespace flux

# add the Flux chart repository to Helm and install Flux.
helm repo add fluxcd https://charts.fluxcd.io

helm upgrade -i flux fluxcd/flux \
--set git.url=git@github.com:EdwinGuo/airflow-eks-config \
--namespace flux

helm upgrade -i helm-operator fluxcd/helm-operator --wait \
--namespace flux \
--set git.ssh.secretName=flux-git-deploy \
--set git.pollInterval=1m \
--set chartsSyncInterval=1m \
--set helm.versions=v3

# Check the install. 3 pods should be running
kubectl get pods -n flux

# Install fluxctl in order to get the SSH key to allow GitHub write access. This allows Flux to keep the configuration in GitHub in sync with the configuration deployed in the cluster.
sudo wget -O /usr/local/bin/fluxctl https://github.com/fluxcd/flux/releases/download/1.19.0/fluxctl_linux_amd64
sudo chmod 755 /usr/local/bin/fluxctl

fluxctl version
fluxctl identity --k8s-fwd-ns flux

mkdir airflow-eks-config/{releases,namespaces}
find airflow-eks-config/ -type d -exec touch {}/.keep \;
cd airflow-eks-config
git add .
git commit -am "directory structure"
git push
```


# Add user to airflow, generate tokens for authentication
```
steps:
https://www.aylakhan.tech/?p=700

defails:

from airflow import models, settings
from airflow.contrib.auth.backends.password_auth import PasswordUser
user = PasswordUser(models.User())
user.username = 'test_user'
user.email = 'test_user@mydomain'
user.password = 'tH1sIsAP@ssw0rd'
session = settings.Session()
user_exists = session.query(models.User.id).filter_by(username=user.username).scalar() is not None
if not user_exists:
   session.add(user)
   session.commit()

session.close()

------ generate token----------
generate the token
import base64

def generate_token(user, password):
	userpass = user + ':' + password
	token = base64.b64encode(userpass.encode()).decode()
	return 'token ' + token
```

# to trigger airflow job with auth
```
 curl -v -X POST http://af15fbba358f24a39b4dee81341e150c-1683699660.us-east-1.elb.amazonaws.com:8080/api/experimental/dags/parallel_dag/dag_runs -H "Authorization: 'token dGVzdF91c2VyOnRIMXNJc0FQQHNzdzByZA=='" -H 'Cache-Control: no-cache' -H 'content-type: application/json'  --insecure -d "{}"

requests.post('http://a9be71af0c7e94ad1acddbcb4119b82a-7275127.us-east-1.elb.amazonaws.com:8080/api/experimental/dags/parallel_dag/dag_runs', data=json.dumps("{}"), auth=("test_user", "tH1sIsAP@ssw0rd"))
```

# How to enable the oidc authentication with AWS eks on pod level
```
https://www.youtube.com/watch?v=bu0M2y2g1m8
```

# How to make helm chart working as github pages
```
https://medium.com/@mattiaperi/create-a-public-helm-chart-repository-with-github-pages-49b180dbb417

git clone https://github.com/${YOUR_NAME}/${YOUR_REPO}.git && cd ${YOUR_REPO}

echo -e “User-Agent: *\nDisallow: /” > robots.txt

helm lint helm-airflow/*

helm package helm-airflow/*

helm repo index --url https://edwinguo.github.io/airflow-eks-helm-chart/ .

git add . && git commit -m “Initial commit” && git push origin master

```


# Frequent used commands
```
kubectl exec -it airflow-dev-webserver-7c88ffdc7-6gdr6 -n dev --container webserver -- /bin/bash

# get the private key and paste to your repo
fluxctl identity --k8s-fwd-ns flux

# after promote job to config or dag repo, you can trigger the job ahead of schedule.
fluxctl sync --k8s-fwd-ns flux

# access logs
kubectl get pods -n flux
kubectl logs helm-operator-<id> -n flux


# helm related
helm list --namespace=dev
helm uninstall airflow-dev -n dev
```
