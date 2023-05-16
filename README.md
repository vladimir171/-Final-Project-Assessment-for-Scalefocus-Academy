<!-- Final Project Assessment for Scalefocus Academy -->

# Final Project Assessment for Scalefocus Academy

## Prerequisites

For this project we need to install the necessary tools: MicroK8s, Helm and Jenkins

## MicroK8s

We need to have a Kubernetes cluster up and running. I'm doing the exercise on Ubuntu so in this case i will use microK8s.
To install MicroK8s on Linux we use the following command:

```console
sudo snap install microk8s --classic
```
We can check the status of microk8s and the services with the command:

```console
sudo microk8s status
```
![Screenshot from 2023-05-15 11-37-40](https://github.com/vladimir171/Final-Project-Assessment-for-Scalefocus-Academy/assets/33956834/b01252e9-03ab-418d-9ff3-16b02d3be714)

Helm is enabled by default in Microk8s.
Additionally here is the list of all [Available addons](https://microk8s.io/docs/addons) that can be enabled.

In my case i enabled DNS, hostpath-storage, dashboard, host-access

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

### Installing the Kubernetes plugin and configuration in Jenkins

From the sidebar on the Jenkins Dashboard we select **Manage Jenkins**, then we go to **Manage Plugins**.
From the sidebar we select **Available plugins** and search for **kubernetes**.
We select the checkbox and **Install without restart**, we can restart Jenkins after the installation is finished.

Now we can navigate back to **Manage Jenkins** and go to **Manage Nodes and Clouds**
From the sidebar we selec **Configure Clouds** and from the dropdown Add a new cloud we select **Kubernetes**

![Screenshot from 2023-05-16 13-26-57](https://github.com/vladimir171/Final-Project-Assessment-for-Scalefocus-Academy/assets/33956834/69a73374-59dd-4f75-a0d3-d619b4c81af0)

Here we need to set the following parametars:

Name = kubernetes

Credentials
We go to Add > Jenkins
- Under Kind we select **Secret File** from the dropdown
- Under File we click Browse... and navigate to $HOME/.kube/config and select the config file
- We leave the other settings to default and click Add

We can now select config (certificate config) from the dropdown and **Test Connection**

![Screenshot from 2023-05-16 13-32-34](https://github.com/vladimir171/Final-Project-Assessment-for-Scalefocus-Academy/assets/33956834/90f769cf-4ed3-49bc-b992-00b8bc93b6d9)

Because all the information including url, token, secret, user are contained in the file we don't need to set the Kubernetes URL

Jenkins URL = http://127.0.0.1:8080
Jenkins tunnel = http://127.0.0.1:50000

We add Pod Template with the following settings:

Name = kube
Labels = kubepods

We add Container with the following settings:

Name = jnlp
Docker image = jenkins/jnlp-slave:latest
Working directory = /home/jenkins/

We add Environment Variable
Key = JENKINS_URL
Value = http://127.0.0.1:8080

Pod Retention = Never

We leave the rest to default and click Save

### Creating the pipeline

From the sidebar we select **+ New item**, we enter the name Helm-Wordpress and select Pipeline

![Screenshot from 2023-05-16 13-43-38](https://github.com/vladimir171/Final-Project-Assessment-for-Scalefocus-Academy/assets/33956834/8c9d9815-50b4-4f04-94c3-94596c3abd48)

We go down to **Pipeline** and we can write our first pipeline to check if wp namespace exists, if it doesn't then it creates one

```console
pipeline {
    agent any
    
    stages {
        stage('Check and Create Namespace') {
            steps {
                script {
                    // Check if the 'wp' namespace exists
                    def namespace ='wp'
                    def namespaceExists = false
                    
                    try {
                        sh(returnStdout: true, script: "microk8s kubectl get namespaces ${namespace} | grep -w ${namespace}").trim()
                        namespaceExists = true
                    } catch (Exception e) {
                        echo "Namespace $namespace does not exist."
                    }
                    
                    if (namespaceExists) {
                        echo "Namespace $namespace already exists."
                    } else {
                        echo "Namespace $namespace does not exist. Creating..."
                        sh "microk8s kubectl create namespace $namespace"
                    }
                }
            }
        }
    }
}
```
The job is successfull and we can see the Console Output

![Screenshot from 2023-05-16 13-49-43](https://github.com/vladimir171/Final-Project-Assessment-for-Scalefocus-Academy/assets/33956834/d93d94b9-e7e1-416c-98e3-a1977ea534fa)

Also we can check from terminal to verify that the namespace was created

![Screenshot from 2023-05-16 13-50-28](https://github.com/vladimir171/Final-Project-Assessment-for-Scalefocus-Academy/assets/33956834/a49689a1-5bb0-49ed-a0eb-96cbc9931011)

Now for the next step we check if WordPress exists, if it doesn't then it installs the chart

```console
stage('Check and Install WordPress') {
            steps {
                
                script {
                    withEnv(['KUBECONFIG=/var/snap/microk8s/current/credentials/client.config']) {
                        def podName = sh(
                            script: 'microk8s kubectl get pods -n wp --output=jsonpath="{.items[*].metadata.name}"',
                            returnStdout: true
                            ).trim()
                          
                          if (!podName.contains('my-wordpress')) {
                            sh "helm install my-wordpress --set wordpressUsername=admin --set wordpressPassword=Password123 --set  service.type=ClusterIP oci://registry-1.docker.io/bitnamicharts/wordpress -n wp"
                            }
                    }
                
                }
            }
        }
```
We can see that the build was successful and check the Console Output

![Screenshot from 2023-05-16 14-30-14](https://github.com/vladimir171/Final-Project-Assessment-for-Scalefocus-Academy/assets/33956834/81fef8ed-19ff-40f6-8a66-3275ce768949)

And if we run the build again it won't do any changes

![Screenshot from 2023-05-16 14-31-06](https://github.com/vladimir171/Final-Project-Assessment-for-Scalefocus-Academy/assets/33956834/6dc328cc-a135-4a25-8d3c-6badf2be8a52)

The complete pipeline looks like this:

```console
pipeline {
    agent any
    
    stages {
        stage('Check and Create Namespace') {
            steps {
                script {
                    // Check if the 'wp' namespace exists
                    def namespace ='wp'
                    def namespaceExists = false
                    
                    try {
                        sh(returnStdout: true, script: "microk8s kubectl get namespaces ${namespace} | grep -w ${namespace}").trim()
                        namespaceExists = true
                    } catch (Exception e) {
                        echo "Namespace $namespace does not exist."
                    }
                    
                    if (namespaceExists) {
                        echo "Namespace $namespace already exists."
                    } else {
                        echo "Namespace $namespace does not exist. Creating..."
                        sh "microk8s kubectl create namespace $namespace"
                    }
                }
            }
        }
        
        stage('Check and Deploy WordPress') {
            steps {
                
                script {
                    withEnv(['KUBECONFIG=/var/snap/microk8s/current/credentials/client.config']) {
                        def podName = sh(
                            script: 'microk8s kubectl get pods -n wp --output=jsonpath="{.items[*].metadata.name}"',
                            returnStdout: true
                            ).trim()
                          
                          if (!podName.contains('final-project-wp-scalefocus')) {
                            sh "helm install final-project-wp-scalefocus --set wordpressUsername=admin --set wordpressPassword=Password123 --set  service.type=ClusterIP oci://registry-1.docker.io/bitnamicharts/wordpress -n wp"
                            }
                    }
                
                }
            }
        }
    }
}
```

It deploys wordpress with the name **final-project-wp-scalefocus** using Helm Chart

We can now do

```console
microk8s kubectl get pods -n wp
microk8s kubectl describe pod final-project-wp-scalefocus-wordpress-545864ffd6-6wbgh -n wp
```
This will give us the details of the pod including the IP address and the port to connect

![Screenshot from 2023-05-16 14-42-51](https://github.com/vladimir171/Final-Project-Assessment-for-Scalefocus-Academy/assets/33956834/70ec933e-ae80-4597-a07d-6fa70e054d8d)

We can now open the browser and load the Home page

![Screenshot from 2023-05-16 14-42-31](https://github.com/vladimir171/Final-Project-Assessment-for-Scalefocus-Academy/assets/33956834/22d6c887-46c7-4f5a-a106-b259c01e37f7)


