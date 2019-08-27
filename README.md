# Personio - SRE Challenge

This project consists of a K8s cluster, hosting the following applications:
* **App**: a very simple app in Python which prints Hello from Python. Runs on port 5000
* **Jenkins**: Build with an image containing kubectl, it is designed to run the CI/CD. It also contains permissions to run commands "natively" in the cluster.
* **Influx-Grafana**: A single image containing InfluxDB and Grafana for monitoring the cluster. Two dashboards are pre-configured.

Also Telegraf will be installed on the hosts, providing data for InfluxDB.

There are two ways to deploy this stack:
* **local**: This is for testing. The stack will be deployed on the local machine.
* **k8s**: The stack will be deployed on AWS EC2 nodes.

The instructions are as follow, with comments about improvements at the end of this document.

When using for local testing, run the following command from the ansible folder:

        $ ansible-playbook main.yml -i hosts -e "env=localhost"

This will install minikube and its dependencies. To enable monitoring, it is necessary to configure telegraf by following these steps:

        $ minikube service influx --url

      Paste the output to the line 104 of the file /etc/telegraf/telegraf.conf, so it should be something like this:

        $  urls = ["http://192.168.99.102:30142"]

      The other configuration required is to set the IP of minikube:

        $ minikube ip

      Paste the output to the line 3 of /etc/telegraf/telegraf.d/k8s.conf

        $  url = "http://192.168.99.102:10255"

After that is done, it is possible to use the environment by accessing the URLs available by:

        $ minikube service SERVICE_NAME --url
    
The last configuration needed is for Jenkins, described below in this README.

To use the remote option, first, you'll need to provide the infrastructure. This is obtainable through Kops (https://github.com/kubernetes/kops), a tool recommended in Kubernetes documentation for AWS deploys. To still use tools that are more common the following commands will generate an Terraform configuration. Since it generates the code from a few parameters, the cluster will be named k8s.felipemkp.com. Before anything else, its necessary to create kops user:

aws iam create-group --group-name kops

aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonEC2FullAccess --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonRoute53FullAccess --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/IAMFullAccess --group-name kops
aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonVPCFullAccess --group-name kops

aws iam create-user --user-name kops

aws iam add-user-to-group --user-name kops --group-name kops

aws iam create-access-key --user-name kops


With the output of the last command, it's necessary to run aws configure inserting the credentials that were generate and then  configure them on environment variables:

        $ export AWS_ACCESS_KEY_ID=$(aws configure get aws_access_key_id)
        $ export AWS_SECRET_ACCESS_KEY=$(aws configure get aws_secret_access_key)

Kops utilizes a bucket as a source of truth. Since the it is possible that a resource in Terraform be created before others, it's recommended to manually create the bucket. After that it is necessary to export these two variables:

        $ aws s3api create-bucket --bucket felipemkp-personio-test --region us-east-1
        $ export NAME=k8s.felipemkp.com
        $ export KOPS_STATE_STORE=s3://felipemkp-personio-test

To generate the Terraform files, run:

        $ kops create cluster --zones us-east-1a ${NAME} --node-count 1 --node-size t2.medium --master-size t2.medium --target=terraform

Then, from the generate directory out/terraform, run Terraform itself:

        $ terraform init (in case you don't have the AWS provider plugin)
        $ terraform plan
        $ terraform apply
    
2 instances will be deployed: 1 for the Master node and 1 for the slave. Each one has its own security group with only the required ports available. 

After the creation of the cluster, will be necessary to obtain the external IP addresses and insert on the hosts file locate at ansible/hosts. On [k8s] should be there only the master node. On [telegraf], both machines should be included.

The cluster takes a while to start but after that, you'll need to run the following command:

                $ ansible-playbook main.yml -i hosts -e "env=remote"

The services were already provisioned however, K8s will create the AWS ELBs with random names. Therefore, by running the command below on a node the registered name will be shown. Please keep in mind that DNS takes a while to propagate and be made available.

        $ kubectl get services

The ports and users are as follow:

* **App**: 5000
* **Jenkins**: 8080 - For configuration please read below
* **Grafana**: 3003 - User and password: root

When jenkins first launches, it requires a token generated automatically. To obtain it, run the following command on one of the nodes:

        $ kubectl get pods | grep jenkins | awk '{ print $1 }' | xargs kubectl logs | grep -B2 initialAdminPassword
    
It's advisable to install the recommended plugins. Afterward, you'll need to create a new job of the type Pipeline, and with the following options:

    * Definition: Pipeline Script from SCM
    * SCM: Git
    * Repository: https://github.com/felipemkp/pre_final.git

In the first execution, the build will fail because Jenkins doesn't have yet the Jenkinsfile with the parameters configured. After refreshing the job page, 3 parameters will appear:

    * Version: The tag number for the container of the app
    * Username: Used to authenticate on Docker Hub and upload a new image with updated code
    * Password

The job is quite simple: It downloads the repository containing the application code, builds a new image with it and uploads to DockerHub to then redeploy the pod with the new version represented by the tag. 

## Possible improvements:

    * Better separation of pods by adding more nodes;
    * Automatically configure telegraf when running locally
    * Spread the nodes through AZs, increasing resilience
    * Unit tests when deploying
    * A much more configurable (and generic) Terraform, with variables to define cluster options
    * Better configuration of the K8s cluster, especially regarding network mappings.