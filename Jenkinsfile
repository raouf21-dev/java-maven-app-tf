#!/usr/bin/env groovy

library identifier: 'jenkins-shared-library@main', retriever: modernSCM(
  [$class: 'GitSCMSource',
  remote: 'https://github.com/raouf21-dev/jenkins-shared-library.git',
  credentialsId: 'github-credentials'
  ]
)

pipeline {   
  agent any
  tools {
    maven 'Maven'
  }
  environment {
    IMAGE_NAME = 'santana20095/demo-app:java-maven-2.0'
  }
  stages {
    stage("build app") {
      steps {
        script {
          echo 'building application jar...'
          buildJar()
        }
      }
    }
    stage("build image") {
      steps {
        script {
          echo 'building docker image...'
          buildImage(env.IMAGE_NAME)
          dockerLogin()
          dockerPush(env.IMAGE_NAME)
        }
      }
    }
    // stage("provision server") {
    //   environment {
    //     AWS_ACCESS_KEY_ID = credentials('jenkins_aws_access_key_id')
    //     AWS_SECRET_ACCESS_KEY = credentials('jenkins_aws_secret_access_key')
    //     TF_VAR_env_prefix = 'test'
    //   }
    //   steps {
    //     script {
    //       sh 'aws sts get-caller-identity'
    //       dir('terraform') {
    //         sh "terraform init"
    //         sh "terraform apply --auto-approve"
    //         EC2_PUBLIC_IP = sh(
    //           script: "terraform output ec2-public_ip",
    //           returnStdout: true
    //         ).trim()
    //       }
    //     }
    //   }
    // }

    stage("provision server") {
  environment {
    AWS_ACCESS_KEY_ID = credentials('jenkins_aws_access_key_id')
    AWS_SECRET_ACCESS_KEY = credentials('jenkins_aws_secret_access_key')
    TF_VAR_env_prefix = 'test'
  }

  steps {
    script {

      // Verify which IAM user Jenkins is using
      sh 'aws sts get-caller-identity'
      sh '''
      aws s3api get-bucket-location \
        --bucket myapp-tf-s3-bucket
      '''
      // TEST 1: Check bucket policy
      sh 'aws s3api get-bucket-policy --bucket myapp-tf-s3-bucket --region ap-south-1 || true'

      // TEST 2: Check if Jenkins can list the bucket
      sh 'aws s3 ls s3://myapp-tf-s3-bucket --region ap-south-1'

      // TEST 3: Check access to Terraform state
      sh '''
        aws s3api head-object \
          --bucket myapp-tf-s3-bucket \
          --key myapp/state.tfstate \
          --region ap-south-1
      '''

      dir('terraform') {
        sh "terraform init"
        sh "terraform apply --auto-approve"

        EC2_PUBLIC_IP = sh(
          script: "terraform output ec2-public_ip",
          returnStdout: true
        ).trim()
      }
    }
  }
}
    stage("deploy") {
      environment {
        DOCKER_CREDS = credentials('docker-hub-repo')
      }
      steps {
        script {
          echo "waiting for EC2 server to initialize"
          sleep(time: 90, unit: "SECONDS")

          echo 'deploying docker image to EC2...'
          echo "${EC2_PUBLIC_IP}"
          
          def shellCmd = "bash ./server-cmds.sh ${IMAGE_NAME} ${DOCKER_CREDS_USR} ${DOCKER_CREDS_PSW}"
          def ec2Instance = "ec2-user@${EC2_PUBLIC_IP}"

          sshagent(['server-ssh-key']) {
            sh "scp -o StrictHostKeyChecking=no server-cmds.sh ${ec2Instance}:/home/ec2-user"
            sh "scp -o StrictHostKeyChecking=no docker-compose.yaml ${ec2Instance}:/home/ec2-user"
            sh "ssh -o StrictHostKeyChecking=no ${ec2Instance} ${shellCmd}"
          }
        }
      }
    }               
  }
}