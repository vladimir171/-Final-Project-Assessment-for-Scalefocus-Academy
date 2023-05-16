<!-- Final Project Assessment for Scalefocus Academy -->

# Final Project Assessment for Scalefocus Academy

## Prerequisites

For this project we need to install the necessary tools: microK8s, Helm and Jenkins

## microK8s

We need to have a Kubernetes cluster up and running. I'm doing the exercise on Ubuntu so in this case i will use microK8s.
To install microK8s on Linux we use the following command:

```console
sudo snap install microk8s --classic
```
We can check the status of microk8s and the services with the command:

```console
sudo microk8s status
```

Helm is enabled by default.
Additionally here is the list of all [Available addons](https://microk8s.io/docs/addons) that can be enabled.

In my case i enabled DNS, hostpath-storage ...

```console
microk8s enable dns
microk8s enable hostpath-storage
```

Link to the official [Install MicroK8s](https://microk8s.io/?_ga=2.244230917.1064120375.1684140339-1143821281.1684140339) documentation.

MicroK8s creates a group to enable seamless usage of commands which require admin privilege.
In order not to have issues with the **Error: Kubernetes cluster unreachable..** we need to add our current user to the group and gain access to the .kube caching directory.
We can do this by running the following two commands:

```console
sudo usermod -a -G microk8s $USER
sudo chown -f -R $USER ~/.kube
```

Then we need to re-enter the session for the group update to take place:

```console
su - $USER
```
Microk8s uses different path for the configuration file which is located in :

We can create $HOME/.kube/config file with the following command ( We will need the file to connect Kubernetes with Jenkis )

```console
microk8s.kubectl config view --raw > $HOME/.kube/config
```
Using the command microk8s.kubectl config view > $HOME/.kube/config didn't work for me without --raw

## Helm

Because Microk8s comes with preinstalled Helm i only added the repository for Bitnami

```console
helm repo add bitnami https://charts.bitnami.com/bitnami
```

Then i tested if i can successfully do helm install from the provided link my-release oci://registry-1.docker.io/bitnamicharts/wordpress .
Instead of editing the file and providing custom .yaml file, i specified the parameters that i want to edit for my helm install

```console
helm install my-wordpress \
  --set wordpressUsername=admin \
  --set wordpressPassword=Password123 \
  --set service.type=ClusterIP \
    oci://registry-1.docker.io/bitnamicharts/wordpress
```

After the successfull install i deleted the release

```console
helm delete my-wordpress
```

## Jenkins

I chose to install Jenkins outside of the cluster and then connect it with Kubernetes

### Installation of Java

Jenkins requires Java in order to run and we can install it with the following command

```console
sudo apt install openjdk-11-jre
```
### Long Term Support release

We install the LTS release for our project with the steps from the [official documentation](https://www.jenkins.io/doc/book/installing/linux/)

```console
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee \
  /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update
sudo apt-get install jenkins
```

Enable the Jenkins service to start at boot

```console
sudo systemctl enable jenkins
```
Start the Jenkins service

```console
sudo systemctl start jenkins
```
### Unlocking Jenkins

Browse to http://localhost:8080 and unlock using the automatically-generated password

The following command will print the password at console

```console
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```
After this we install the recomended plugins and wait for them to finish

### Installing the Kubernetes plugin and configuration

  
