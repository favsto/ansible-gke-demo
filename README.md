# Google Kubernetes Engine automation workshop

## Ansible

* log into the [Google Cloud Console](https://console.cloud.google.com)
* open Google Cloud Shell with the button in the right top of the console
* create a new Google Compute Engine instance, this is our working machine:
```bash
gcloud compute instances create ansible-master --zone=europe-west3-b --scopes=https://www.googleapis.com/auth/cloud-platform
```
* you can close the Google Cloud Shell. Come back to the console, enter the Google Compute Engine section
* log into the GCE instance called ansible-master via the SSH button on its right
```bash
# install base packages
sudo su
apt-get install -y git && gcloud source repos clone demo ./workshop

# setup ansible
chmod +x workshop/ansible/ansible_setup.sh
source workshop/ansible/ansible_setup.sh
```
* Ansible will use an SSH key, copy it to your clipboard
```bash
cat /root/.ssh/ansible.pub
```
* paste it into project metadata: project menu > Compute Engine -> Metadata SSH Keys > Edit > Add item
* create a new Service Account: project manu > IAM & admin -> Service accounts > Create service account. Use these parameters, it will download a JSON file locally:
```
service account name: ansible
role: Project > Editor
service account ID: default
furnisk a new private key: JSON
```
* come back to the SSH shell and upload the JSON via the gear icon on right top, and then Upload file
* move the file:
```bash
# replace <sa-key>.json with your service account file name, before running
mv /home/$SUDO_USER/<sa-key>.json /root/$project_id.json
chown root:root /root/$project_id.json
chmod 600 /root/$project_id.json
```
* test the inventory
```bash
ansible -i /usr/local/ansible/inventory/$project_id/ all --list-hosts
```
* create some GCE instances:
```bash
mkdir -p /usr/local/ansible/playbooks

wget --no-check-certificate "https://drive.google.com/uc?export=download&id=12UgG61SEr5MMJ9_sR-SLPKaLhvR3QMi-" -O /usr/local/ansible/playbooks/gce_create_demo.yml

ansible-playbook -i /usr/local/ansible/inventory/$project_id/ /usr/local/ansible/playbooks/gce_create_demo.yml

# choose among all, tag_ubuntu...
ansible -i /usr/local/ansible/inventory/$project_id/ <all, tag_ubuntu, n1-standard-1...> --list-hosts
```
* let's create a brand new playbook
```bash
head /usr/local/ansible/playbooks/gce_create_demo.yml -n 12 > /usr/local/ansible/playbooks/create_k8s_cluster.yml

nano /usr/local/ansible/playbooks/create_k8s_cluster.yml
# add the following tasks:
  - name: "Configuring Google Cloud SDK for Ansible Master..."
    shell: |
      gcloud auth activate-service-account {{srv_acct}} --key-file={{crd_file}} --quiet
      gcloud config set project {{gce_proj}} --quiet
      gcloud services enable container.googleapis.com --quiet
      gcloud services enable containerregistry.googleapis.com --quiet
    run_once: true
    delegate_to: localhost
  - name: "Installing kubectl on Ansible Master..."
    apt: name=kubectl
    run_once: true
    delegate_to: localhost
  - name: "Creating a new Kubernetes cluster..."
    local_action: shell gcloud container clusters create demo-cluster --num-nodes 3 --machine-type n1-standard-1 --zone europe-west3-b
    run_once: true
```
* execute the new playbook through ansible-master
```bash
ansible-playbook -i /usr/local/ansible/inventory/$project_id/ -l ansible-master /usr/local/ansible/playbooks/create_k8s_cluster.yml
```
* check that your cluster is up and running browsing the section Kubernetes Engine into the project.
* close your SSH session

## Kubernetes Engine

* navigate the project and open Google Cloud Shell
* download and enter the folder with the demo application
```bash
gcloud source repos clone demo ./workshop
cd workshop/kubernetes/application
```
* build the first container image (v1) with the demo code and push it to Google Container Registry
```bash
docker build -t gcr.io/$DEVSHELL_PROJECT_ID/hello-node:v1 .
gcloud docker -- push gcr.io/$DEVSHELL_PROJECT_ID/hello-node:v1
```
* grab your Kubernetes cluster credentials
```bash
gcloud container clusters get-credentials demo-cluster --zone europe-west3-b --project $DEVSHELL_PROJECT_ID
```
* deploy your first version
```bash
kubectl run hello-node --image=gcr.io/$DEVSHELL_PROJECT_ID/hello-node:v1 --port=8080
kubectl expose deployment hello-node --type="LoadBalancer"

# please hold, the external IP will be provided...
kubectl get services --watch
```
* write down your provided EXTERNAL IP address; you can try your application navigating https://<EXTERNAL-IP>:8080 or via this recursive command (use another tab of the shell):
```bash
while sleep 1; do curl http://<EXTERNAL-IP>:8080; echo; done
```
* keep this recursive call running if you can, it will show you how the application will change
* scale up the application (notice changes on the recurisve response):
```bash
kubectl scale deployment hello-node --replicas=4
```
* now, make some changes on the file `workshop/kubernetes/application/server.js`, change its version name
* deploy your new version and notice how the rolling update works:
```bash
# create a new application version
docker build -t gcr.io/$DEVSHELL_PROJECT_ID/hello-node:v2 .
gcloud docker -- push gcr.io/$DEVSHELL_PROJECT_ID/hello-node:v2

# rolling update
kubectl set image deployment/hello-node hello-node=gcr.io/$DEVSHELL_PROJECT_ID/hello-node:v2
```

## Google Container Builder

* navigate the project and open Google Cloud Shell
* configure your git for pushing new code:
```bash
git config --global user.email "<YOUR-EMAIL>"
git config --global user.name "Demo User"
```
* study the content of the file `workshop/builder/cloudbuild.yaml`; notice that its content order has been randomized :)
* come back to the Cloud Console and navigate to: Container Registry > Build triggers
* create a new trigger with these parameters:
```
source: Cloud Source Repository
repository: demo

name: workshop
trigger type: branch
branch (regex): .*
build configuration: cloudbuild.yaml
cloudbuild.yaml location: /builder/cloudbuild.yaml

```
* come back to the Cloud Shell and put the content of your cloudbuil.yaml file in the right order (use wathever text editor you prefer) and save
* test your release:
```bash
cd ~/workshop
git add . ; git commit -m "test builder" ; git push
cd builder
```
* see the effects in your project, navigate Container Registry > Build history
* if your build has been processed correctly you will see new changes on your published application; otherwise try a new combination and retry previous commands