pipeline{
    agent any
    environment{ // set global variables as per requirement
        REPOSITORY_URI = " "
        REPOSITORY_NAME = " "
        REGION_NAME = " "
        AWS_ACCESS_KEY_ID = credentials
        AWS_SECRET_ACCESS_KEY= credentials
        CREDENTIAL_ID = " "
        DEPLOY_LOCATION = " "
        IS_DEPLOYMENT = " "
        PR = env.BUILD_TAG.replaceALL('%2F' , '-')
        BUILD_IMAGES = true

        // K8S VARIABLES //

        ENV_NAME = " "
        NAMESPACE = " "
        DEPLOY_ENVIRONMENT = " "
        DEPLOY_REPLICAS = " "
        CERT_ARN = " "
        OS_ENV_DEPLOY = " "  // Environment of Deployment
        ALB_TAG = " "
        MEM_REQ = " "
        CPU_REQ = " "
        MEM_LIMIT = " "
        CPU_LIMIT = " "
         

    }
    stages{
        stage('Select Environment'){
            steps{
                script{
                    echo "Branch Name: ${env.BRANCH_NAME}"
                    if('develop'.equals(env.BRANCH_NAME)){
                        echo "Dev/Stage Deployments"
                        DEPLOY_LOCATION = "dev"        // refers to environment for deployment
                        ENV_NAME = "develop"           // refers to branch name 
                    }
                    if(env.BRANCH_NAME.startsWith('release')){
                        echo "QA/Production Deployments"
                        DEPLOY_LOCATION = "release"     // refers to environment for deployment
                        ENV_NAME = "release/CICD"        // refers to branch name
                    }
                }
            }
        }
        stage('Build Image & Push to ECR'){
            steps{
                script{
                    sh 'aws configure set region <name>'
                    echo "${PR}"
                    sh "aws codebuild create-project --region <name> --name '${PR}' --vpc-config vpcId= <value>, subnets=<value>, securityGroupIds=<value>  --source type=BITBUCKET, location=<url>,buildspec=<path to buildspec.yaml file>  --source-version='${BRANCH_NAME}' --artifacts type=NO_ARTIFACTS --environment type=LINUX_CONTAINER, image='image format', computeType=BUILD_GENERAL1_SMALL, privilegedMode=true --service-role <value> "
                    awsCodeBuild credentialsType: 'keys', projectName: "${PR}", region: ' ', sourceVersion: '${BRANCH_NAME}', soruceControlType: 'project'
                    sh "aws codebuild ==region <name> delete-project --name '${PR}'"
                }
            }
        }
        stage('Scanning Images for Vulnerabilities'){
            when{
                expression{
                    return DEPLOY_LOCATION.equals("release")
                }
            }
            steps{
                script{
                    sh "aws ecr wait image-scan-complete --repository-name ${REPOSITORY_NAME} --image-id imageTag=${PR} --region ${REGION_NAME}"
                    sh "aws ecr wait describe-image-scan-findings --repository-name ${REPOSITORY_NAME} --image-id imageTag=${PR} --region ${REGION_NAME}"
                }
            }
        }
        stage('Dev Deployment'){
            when{
                expression{
                    return DEPLOY_LOCATION.equals("dev")
                }
            }
            environment{
                CREDENTIAL_ID = ' '
                NAMESPACE = ' '
                DEPLOY_ENVIRONMENT = "dev"
                DEPLOY_REPLICAS = 
                NODE_AFFINITY = ' '
                CERT_ARN = " "
                OS_ENV_DEPLOY = "Development"
                ALB_TAG = " "
            }
            steps{
                script{
                    if (DEPLOY_LOCATION.equals("dev")){
                        withKubeConfig(credentialsId: "${CREDENTIAL_ID}",serverUrl: ''){
                            sh """
                            kubectl config current-context
                            git checkout ${BRANCH_NAME}
                            envsubst <K8S/deploy-var.yaml> K8S/deploy.yaml
                            cat K8S/deploy.yaml | sed 's|OS_ENV_DEPLOY|${OS_ENV_DEPLOY}|g; s/NAMESPACE/${NAMESPACE}/g; s/DEPLOY_ENVIRONMENT/${DEPLOY_ENVIRONMENT}/g; s/DEPLOY_REPLICAS/${DEPLOY_REPLICAS}/g; s/NODE_AFFINITY/${NODE_AFFINITY}/g; s|CERT_ARN|${CERT_ARN}|g; s|ALB_TAG|${ALB_TAG}|g' > k8s-dev-file.yaml
                            cat k8s-dev-file.yaml
                            kubectl apply -f k8s-dev-file.yaml
                            """
                        }
                    }
                }
            }
        }


//       _                   
//      | |                  
//   ___| |_ __ _  __ _  ___ 
//  / __| __/ _` |/ _` |/ _ \
//  \__ \ || (_| | (_| |  __/
//  |___/\__\__,_|\__, |\___|
//                 __/ |     
//                |___/      


stage('Stage Approval'){
    when{
        expression{
            refers DEPLOY_LOCATION.equals("dev")
        }
    }
    steps{
        script{
            try{
                timeout(time: 6, unit: 'HOURS'){
                    APPROVED = input(
                        id : 'Stage',
                        messsage: 'Stage Deployment',
                        parameters: [
                            booleanParam(
                                defaultValue: false,
                                description: '',
                                name: 'Stage Approved'
                            )
                        ]
                    )
                    if (APPROVED){
                        echo 'Approved for Stage'
                        CREDENTIAL_ID = ' '
                        NAMESPACE = ' '
                        DEPLOY_ENVIRONMENT = "Stage"
                        DEPLOY_REPLICAS = 
                        NODE_AFFINITY = ' '
                        CERT_ARN = " "
                        OS_ENV_DEPLOY = "Stage"
                        ALB_TAG = " "
                    } else{
                        DEPLOY_ENVIRONMENT = "none"
                    }
                }
            } catch (err){
                DEPLOY_ENVIRONMENT = "none"
            }
        }
    }
}
stage('Stage Deployment'){
    when{
        expression{
            return DEPLOY_ENVIRONMENT.equals("stage")
        }
    }
    steps{
                script{
                    if (DEPLOY_LOCATION.equals("dev")){
                        withKubeConfig(credentialsId: "${CREDENTIAL_ID}",serverUrl: ''){
                            sh """
                            kubectl config current-context
                            git checkout ${BRANCH_NAME}
                            envsubst <K8S/deploy-var.yaml> K8S/deploy.yaml
                            cat K8S/deploy.yaml | sed 's|OS_ENV_DEPLOY|${OS_ENV_DEPLOY}|g; s/NAMESPACE/${NAMESPACE}/g; s/DEPLOY_ENVIRONMENT/${DEPLOY_ENVIRONMENT}/g; s/DEPLOY_REPLICAS/${DEPLOY_REPLICAS}/g; s/NODE_AFFINITY/${NODE_AFFINITY}/g; s|CERT_ARN|${CERT_ARN}|g; s|ALB_TAG|${ALB_TAG}|g' > k8s-stage-file.yaml
                            cat k8s-stage-file.yaml
                            kubectl apply -f k8s-stage-file.yaml
                            """
                        }
                    }
                }
            }
        }

//    __ _  __ _ 
//   / _` |/ _` |
//  | (_| | (_| |
//   \__, |\__,_|
//      | |      
//      |_|      


stage('QA Approval'){
    when{
        expression{
            refers DEPLOY_LOCATION.equals("release")
        }
    }
    steps{
        script{
            try{
                timeout(time: 6, unit: 'HOURS'){
                    APPROVED = input(
                        id : 'Qa',
                        messsage: 'QA Deployment',
                        parameters: [
                            booleanParam(
                                defaultValue: false,
                                description: '',
                                name: 'QA Approved'
                            )
                        ]
                    )
                    if (APPROVED){
                        echo 'Approved for Qa'
                        CREDENTIAL_ID = ' '
                        NAMESPACE = ' '
                        DEPLOY_ENVIRONMENT = "Qa"
                        DEPLOY_REPLICAS = 
                        NODE_AFFINITY = ' '
                        CERT_ARN = " "
                        OS_ENV_DEPLOY = "QA"
                        ALB_TAG = " "
                    } else{
                        DEPLOY_ENVIRONMENT = "none"
                    }
                }
            } catch (err){
                DEPLOY_ENVIRONMENT = "none"
            }
        }
    }
}
stage('QA Deployment'){
    when{
        expression{
            return DEPLOY_ENVIRONMENT.equals("Qa")
        }
    }
    steps{
                script{
                    if (DEPLOY_LOCATION.equals("release")){
                        withKubeConfig(credentialsId: "${CREDENTIAL_ID}",serverUrl: ''){
                            sh """
                            kubectl config current-context
                            git checkout ${BRANCH_NAME}
                            envsubst <K8S/deploy-var.yaml> K8S/deploy.yaml
                            cat K8S/deploy.yaml | sed 's|OS_ENV_DEPLOY|${OS_ENV_DEPLOY}|g; s/NAMESPACE/${NAMESPACE}/g; s/DEPLOY_ENVIRONMENT/${DEPLOY_ENVIRONMENT}/g; s/DEPLOY_REPLICAS/${DEPLOY_REPLICAS}/g; s/NODE_AFFINITY/${NODE_AFFINITY}/g; s|CERT_ARN|${CERT_ARN}|g; s|ALB_TAG|${ALB_TAG}|g' > k8s-qa-file.yaml
                            cat k8s-qa-file.yaml
                            kubectl apply -f k8s-qa-file.yaml
                            """
                        }
                    }
                }
            }
        }

//                       _ 
//                      | |
//   _ __  _ __ ___   __| |
//  | '_ \| '__/ _ \ / _` |
//  | |_) | | | (_) | (_| |
//  | .__/|_|  \___/ \__,_|
//  | |                    
//  |_|                    

stage('Prod Approval'){
    when{
        expression{
            refers DEPLOY_LOCATION.equals("release")
        }
    }
    steps{
        script{
            try{
                timeout(time: 6, unit: 'HOURS'){
                    APPROVED = input(
                        id : 'Prod',
                        messsage: 'Prod Deployment',
                        parameters: [
                            booleanParam(
                                defaultValue: false,
                                description: '',
                                name: 'Prod Approved'
                            )
                        ]
                    )
                    if (APPROVED){
                        echo 'Approved for Prod'
                        CREDENTIAL_ID = ' '
                        NAMESPACE = ' '
                        DEPLOY_ENVIRONMENT = "Prod"
                        DEPLOY_REPLICAS = 
                        NODE_AFFINITY = ' '
                        CERT_ARN = " "
                        OS_ENV_DEPLOY = "Production"
                        ALB_TAG = " "
                    } else{
                        DEPLOY_ENVIRONMENT = "none"
                    }
                }
            } catch (err){
                DEPLOY_ENVIRONMENT = "none"
            }
        }
    }
}
stage('Prod Deployment'){
    when{
        expression{
            return DEPLOY_ENVIRONMENT.equals("Prod")
        }
    }
    steps{
                script{
                    if (DEPLOY_LOCATION.equals("release")){
                        withKubeConfig(credentialsId: "${CREDENTIAL_ID}",serverUrl: ''){
                            sh """
                            kubectl config current-context
                            git checkout ${BRANCH_NAME}
                            envsubst <K8S/deploy-var.yaml> K8S/deploy.yaml
                            cat K8S/deploy.yaml | sed 's|OS_ENV_DEPLOY|${OS_ENV_DEPLOY}|g; s/NAMESPACE/${NAMESPACE}/g; s/DEPLOY_ENVIRONMENT/${DEPLOY_ENVIRONMENT}/g; s/DEPLOY_REPLICAS/${DEPLOY_REPLICAS}/g; s/NODE_AFFINITY/${NODE_AFFINITY}/g; s|CERT_ARN|${CERT_ARN}|g; s|ALB_TAG|${ALB_TAG}|g' > k8s-prod-file.yaml
                            cat k8s-prod-file.yaml
                            kubectl apply -f k8s-prod-file.yaml
                            """
                        }
                    }
                }
            }
        }

    }
}