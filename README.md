# Ansible Integration in Jenkins

## Create a Droplet on Digital ocean for Jenkins

After creating the server, we will Jenkins as docker container

```
docker run -p 8080:8080 -p 50000:50000 -d -v jenkins_home:/var/jenkins_home -v
/var/run/docker.sock:/var/run/docker.sock -v $(which docker):/usr/bin/docker jenkins/jenkins:lts
```

## Create a Droplet on Digital ocean for for Ansible Control Node

When the server is created, we will Conﬁgured it as Ansible Control Node

```
apt update
apt install ansible -y
apt install python3-pip -y
pip3 install boto3 botocore
```

## Create 2 EC2 instances 

You can either create EC2 instances in the AWS console or you can use Terraform. 
Follow this [Link](https://github.com/hotiaDiallo/terraform-playground/tree/deploy-to-ec2) where i explain how to provision EC2 instances with terraform. 

## Create a Jenkinsfile to configure EC2 instances

To run an ansible playbook, we are going to need the `playbook` itself, a `host file` or a `dynamic inventory` and also a `ansible config file`. That means ansible files should be copied the the remote server.

Note we are going to create some credentials on Jenkins before triggering the pipeline: 
- `ansible-server-key`: for Jenkins to be able to copy files to the ansible server, it need a credentials to ssh to the server;
- `ec2-server-key`: use to copy the content of whatever this private key contains to the ansible server. This key will be use by ansible, from the ansible config file, `private_key_file = ~/server-ssh-key.pem` to be able to ssh to our EC2 instances

Another configuration need to be done in our ansible Control Node. Since we are going to use a dynamic inventory, we need to authenticate to AWS when running the playbook. 
- create .aws folder in the home directory: `mkdir .aws`
- `vim .aws/credentials` and add the following configuration : 
```
[default]
aws_access_key_id = XXXXXX
aws_secret_access_key = XXXXX
region = eu-west-3
```

Now we can complete our jenkinsfile 

### Copy files to ansible server

```
stage("copy files to ansible server") {
    steps {
        script {
            echo "copying all needed files to ansible control node"
            sshagent(['ansible-server-key']) {
                sh "scp -o StrictHostKeyChecking=no ansible/* root@${ANSIBLE_SERVER}:/root"

                withCredentials([sshUserPrivateKey(credentialsId: 'ec2-server-key', keyFileVariable: 'keyfile', usernameVariable: 'user')]) {
                    sh 'scp $keyfile root@$ANSIBLE_SERVER:/root/server-ssh-key.pem'
                }
            }
        }
    }
}
```

### Execute ansible playbook by jenkins

```
stage("execute ansible playbook") {
    steps {
        script {
            echo "calling ansible playbook to configure ec2 instances"
            def remote = [:]
            remote.name = "ansible-server"
            remote.host = ANSIBLE_SERVER
            remote.allowAnyHosts = true

            withCredentials([sshUserPrivateKey(credentialsId: 'ansible-server-key', keyFileVariable: 'keyfile', usernameVariable: 'user')]){
                remote.user = user
                remote.identityFile = keyfile
                sshScript remote: remote, script: "install-prerequisites.sh"
                sshCommand remote: remote, command: "ansible-playbook ec2-playbook.yaml"
            }
        }
    }
}
```


