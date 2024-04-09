def aws_region='ap-south-2'
def ecr_yt='httpd'
def td_netflix='zm-td'
def svc_netflix='zm-svc'
def cluster_name='zomato-prod'

pipeline {
    agent any

parameters {
        string defaultValue: 'latest', description: 'release image tag ', name: 'release_tag', trim: false   

        booleanParam defaultValue: false, description: 'zomato container deployment', name: 'zomato_deploy'
    }
    stages {
    stage('checkout') {
            steps {
                 git branch: 'main', url: 'https://github.com/DoesntMatterPrivateLimited/pk_zomato.git'
            }
        }
        
  

    stage('update backend task definition'){
            steps {
                script {
                    if (params.zomato_deploy == true) {
                    sh """
                    dockerRepo=`aws ecr describe-repositories --repository-names ${ecr_yt} --region ${aws_region} | grep repositoryUri | cut -d "\\"" -f 4`
                    dockerTag=${params.release_tag}
                     aws ecs describe-task-definition --task-definition ${td_netflix} --region ${aws_region}  | jq '.taskDefinition | del(.requiresAttributes[]) | del(.revision) | del(.taskDefinitionArn) | del(.status) | del(.requiresAttributes) | del(.compatibilities) | del(.registeredAt) | del(.registeredBy) | .containerDefinitions[0].portMappings[0].containerPort = 80' > ${td_netflix}.json
                    sed -i "0,/.*image.*/s##\\\"image\\\": \\\"\${dockerRepo}:\${dockerTag}\\\",#" ${td_netflix}.json
                    echo "updated task definition"
                    aws ecs register-task-definition --family ${td_netflix} --cli-input-json file://${td_netflix}.json --region ${aws_region} > task_update.json
                    echo "registering the task definition"
                    revision=`aws ecs describe-task-definition --task-definition ${td_netflix} --region ${aws_region} | grep revision | tr -s " " | cut -d " " -f 3`
                    echo \$revision
                    echo "value of revision"
                   
                    """
                }
            }
        }}
        stage('zomato_deploy') {
            steps {
                script {
                    if (params.zomato_deploy == true) {
                    sh """
                    echo "copy deployment json file"
                    cp /var/lib/jenkins/workspace/ecs-deploy/zm_deployment.json ./deployment.json
                    taskDefinitionArn1=`aws ecs describe-task-definition --task-definition ${td_netflix} --region ${aws_region} | jq .taskDefinition | jq .taskDefinitionArn`
                    taskDefinitionArn=`echo \$taskDefinitionArn1 | tr -d '"'`
                    sed -i "s#TASK_DEF_ARN#\$taskDefinitionArn#" deployment.json
                    echo "updated deployment yaml file"
                    aws deploy create-deployment --cli-input-json file://deployment.json > deployment_id.txt --region ${aws_region}
                    echo "deployment triggered"
                    
                    """
                }}
            }
        }
    }
}
