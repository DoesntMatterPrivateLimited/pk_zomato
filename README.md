# $${\color{blue}Project : \color{red} Project-Zomato-App}$$ 

 

### $\color{pink}{Tech / Stack :}$

- Javascript
- AWS
- Docker
- Jenkins
- Trivi
- Owasp
- ECR
- ECS


## $\color{lightblue}{Create \ Ubuntu \ VM \ using \ AWS \ EC2}$


## $\color{green}{Step-1 : Jenkins \ Server \ Setup}$

- Install Java & Jenkins using below commands
````
sudo apt-get update

sudo apt-get install default-jdk17

wget -q -O - https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo apt-key add -

sudo sh -c 'echo deb https://pkg.jenkins.io/debian-stable binary/ > /etc/apt/sources.list.d/jenkins.list'

sudo apt-get update

sudo apt-get install jenkins
sudo systemctl start jenkins
sudo systemctl enable jenkins
sudo systemctl status jenkins
````
- Copy jenkins admin password
````
sudo cat /var/lib/jenkins/secrets/initialAdminPassword       
````
- Open jenkins server in browser using VM public ip
 $\color{blue}{http://public-ip:8080/}$                   

- Create Admin Account & Install Required Plugins in Jenkins

## $\color{green}{Step-2 : Install \ NodeJs}$

````
sudo apt install nodejs
````

## $\color{green}{Step-3 : Run \ Sonar \ Using \ Docker}$

````
docker run -d --name sonarqube -p 9000:9000 sonarqube:lts-community
````
- Enable 9000 port number in security group

- Login into Sonar Server & Generate Sonar Token 

- Go to credentials Add Sonar Token in 'Jenkins Credentials' as Secret Text

 - Manager Jenkins 
 - Credentials 
 - Add Credentials 
 - Select Secret text
 - Enter Sonar Token as secret text 

- Go To  Manage Jenkins -> Plugins -> Available -> Sonar Qube Scanner Plugin -> Install it

- Go To Manage Jenkins -> Configure System -> Sonar Qube Servers -> Add Sonar Qube Server 
		
 1. Name : Sonar-Server
 2. Server URL : http://52.66.247.11:9000/   (Give your sonar server url here)
 3. Add Sonar Server Token			

- Once above steps are completed, then add below stage in the pipeline
````
stage('SonarQube analysis') {
			withSonarQubeEnv('sonar-9.9.3') {
			def mavenHome = tool name: "Maven-3.9.6", type: "maven"
			def mavenCMD = "${mavenHome}/bin/mvn"
			sh "${mavenCMD} sonar:sonar"
    	}
}

````
## $\color{green}{Step-4: Install \ Trivy:}$

````
sudo apt-get install wget apt-transport-https gnupg lsb-release
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy  
````
- Create Trivi Scan Stage in Pipeline
````
stage('TRIVY FS SCAN') {
            steps {
                sh "trivy fs . > trivyfs.txt"
            }
        }
   
````
- To scan image using trivy
```` 
trivy image <imageid>
````

## $\color{green}{Step-4: \ Install \ Dependency-Check \ Plugin}$

- Go to "Dashboard" in your Jenkins web interface.

- Navigate to "Manage Jenkins" → "Manage Plugins."

- Click on the "Available" tab and search for "OWASP Dependency-Check."

- Check the checkbox for "OWASP Dependency-Check" and click on the "Install without restart" button.

- Configure Dependency-Check Tool:

- After installing the Dependency-Check plugin, you need to configure the tool.


- Go to "Dashboard" → "Manage Jenkins" → "Global Tool Configuration."

- Find the section for "OWASP Dependency-Check."

- Add the tool's name, e.g., "DP-Check."

- Save your settings.

````
stage('Install Dependencies') {
            steps {
                sh "npm install"
            }
        }

        stage('OWASP FS SCAN') {
             steps {
                 dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
                 dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
             }
         }
````
## $\color{green}{Step-6 : Setup \ ECR}$

- Go to plugin=AWS credentials,
- Amazon ECR plugin

1. create credentials and add AWS  credentials 
  Install and Configure the AWS CLI:
  If you haven't already, install the AWS CLI and configure it with your AWS access key ID, secret access key, default region, etc. You can do this by running:

````
aws configure
````
2. Create an ECR Repository:
   Use the create-repository command to create a new repository in ECR. You need to specify a name for the repository.

$ aws ecr create-repository --repository-name your-repo-name

3. (Optional) Set Permissions:
By default, the repository will be private. If you want to grant permissions to other AWS accounts or IAM users, you can create a policy and attach it to the repository. For example:

$ aws ecr set-repository-policy --repository-name your-repo-name --policy-text file://ecr-policy.json

  Replace 'ecr-policy.json' with the path to a JSON file containing your policy.

4. Authenticate Docker Client with ECR:
   Before you can push Docker images to your ECR repository, you need to authenticate your Docker client with the ECR registry. You can do this using the get-login-password command to       retrieve an authentication token and then use it to log in to Docker.
````
aws ecr get-login-password --region your-region | docker login --username AWS --password-stdin your-account-id.dkr.ecr.your-region.amazonaws.com
````
  Replace 'your-region' with your AWS region (e.g., 'us-east-1') and 'your-account-id' with your AWS account ID.

5. Push Docker Images to ECR:
   Build your Docker image locally and then tag it with the URI of your ECR repository.
````
 docker build -t your-repo-name .

docker tag your-repo-name:latest your-account-id.dkr.ecr.your-region.amazonaws.com/your-repo-name:latest

docker push your-account-id.dkr.ecr.your-region.amazonaws.com/your-repo-name:latest
````
Replace 'your-repo-name', 'your-account-id', and 'your-region' with appropriate values.

````
stage('ecr login') {
            steps {
                script {
                    docker.withRegistry('https://590183663006.dkr.ecr.ap-south-2.amazonaws.com', 'ecr:ap-south-2:awscreds') {
                        // Inside this block, you can perform Docker-related operations
                    }
                }
            }
        }

stage('ecr image push') {
            steps {
                script {
                    
                    docker.withRegistry ('https://590183663006.dkr.ecr.ap-south-2.amazonaws.com', 'ecr:ap-south-2:awscreds'){
                    docker.image('zomato:latest').push('latest')
                        
                    }
                    
                    }
                }
            }
````
## $\color{green}{Step-7: \ load \ Balancer}$
````
stage('load balancer') {
            steps {
                
            withCredentials([aws(accessKeyVariable: 'AWS_ACCESS_KEY_ID', credentialsId: 'awscreds', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                    sh '''
                    aws elbv2 create-target-group --name tg1 --protocol HTTP --port 80 --vpc-id vpc-0e3a99ba27ae8a71c --target-type ip --ip-address-type ipv4
                    aws elbv2 create-load-balancer --name zomato --subnets subnet-0abfaaa33ce7d7693 subnet-09e4560f5b398c7cb --security-groups sg-072b3bbafdb2ad46e
                    aws elbv2 create-listener --load-balancer-arn arn:aws:elasticloadbalancing:ap-south-2:590183663006:loadbalancer/app/zomato/b41cc28a96a6bf57 --protocol HTTP --port
                    80 --default-actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:ap-south-2:590183663006:targetgroup/tg1/062e93db68af4803

                    
            '''
````

## $\color{green}{Step-8: \ Create \ ECS \ Cluster}$


1. Create a Cluster: Use the create-cluster command to create a new ECS cluster. You'll need to specify the cluster name and optionally the cluster configuration.

2. Configure Task Definition: Define your task definition, which describes how your containers should be launched and run. You can do this using the register-task-definition command.

3. Create a Service: Use the create-service command to create a service within your ECS cluster. The service manages the desired number of tasks and maintains the specified number of           	running copies of your task definition.
````
 stage('ecs') {
            steps {
                withCredentials([aws(accessKeyVariable: 'AWS_ACCESS_KEY_ID', credentialsId: 'awscreds', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                sh '''
                    aws ecs create-cluster --cluster-name zomato
                    aws ecs register-task-definition --cli-input-json file://./zm.json --region ap-south-2
                    '''
}
               
            }
            
        }
        stage('Deploy ECS Service') {
            steps {
                script{
                withCredentials([aws(accessKeyVariable: 'AWS_ACCESS_KEY_ID', credentialsId: 'awscreds', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY')]){
                    def clusterName = 'zomato'
                    def serviceName = 'zservice'
                    def taskDefinition = 'zomatotd:2'
                    def desiredCount = 1
                    def launchType = 'FARGATE'
                    def platformVersion = 'LATEST'
                    def subnets = 'subnet-0abfaaa33ce7d7693'
                    def securityGroups = 'sg-072b3bbafdb2ad46e'
                    def loadBalancerJson = "{\"targetGroupArn\": \"arn:aws:elasticloadbalancing:ap-south-2:590183663006:targetgroup/tg1/062e93db68af4803\", \"containerName\": 
                   \"zomato\", \"containerPort\": 3000}"

                    sh "aws ecs create-service --cluster ${clusterName} --service-name ${serviceName} --task-definition ${taskDefinition} --desired-count ${desiredCount} --launch-type 
                    ${launchType} --platform-version ${platformVersion} --load-balancers '${loadBalancerJson}' --network-configuration 'awsvpcConfiguration={subnets= 
                   [${subnets}],securityGroups=[${securityGroups}],assignPublicIp=ENABLED}'"
                    // sh "aws ecs create-service --cluster ${clusterName} --service-name ${serviceName} --task-definition ${taskDefinition} --desired-count ${desiredCount} --launch- 
                    type ${launchType} --platform-version ${platformVersion} --load-balancers [{\"targetGroupArn\": \"arn:aws:elasticloadbalancing:ap-south- 
                    2:590183663006:targetgroup/tg1/9a198ac210c109cb\", \"containerName\": \"zomato\", \"containerPort\": 80}] --network-configuration 'awsvpcConfiguration={subnets= 
                   [${subnets}],securityGroups=[${securityGroups}],assignPublicIp=ENABLED}'"
                }
                }
            }
        }


````

