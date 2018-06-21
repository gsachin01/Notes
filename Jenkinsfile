#!/bin/groovy

pipeline {
  
  agent {
    node {
      
      label 'build_slave'
     
    }
  }
  parameters {
    choice(
      name: 'veracode_scan',
      choices: 'false\ntrue',
      description: 'Do you want to do the Veracode Scan',
      )
      // booleanParam(defaultValue: false, description: 'veracode_scan', name: 'veracode_scan')
    }
    options {
      timestamps()
    }
    
    
  
  
    
  stages {  
      
      
      stage('Clean Directory') {
        steps{
          
          deleteDir()
          script{        
            REPO = "containers.mycomp.com/myteam/dev/sampleecs"
            //sh "source ~/.ssh/vCreds" 
            //sh "source ~/.ssh/credentials" 
          }
        }
      }
      stage('Checkout') {
        steps{
          checkout scm
        }
      }
      
      stage('Checkout App Code') {
        steps{
          sh "env"
         
          checkout([$class: 'GitSCM', branches: [[name: '*/master']], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[credentialsId: 'mymanAPI', url: 'https://github.mycomp.com/mycomp/myteam-sample-svc-py.git']]])
          sh "ls -lrt"
          sh "chmod -R +x *.sh"
        }
      }
      
      stage('Tar File for Veracode Static Analysis') {
        steps{
          sh "tar -cvzf myteam-sample-svc-py.tar.gz * --exclude='myteam-sample-svc-py.tar.gz'"
          sh "ls -lrt"
          sh "pwd"
        }
      }
      
      stage('Veracode Static Analysis') {
        when {
          expression {
            // Given our default value is true, this should 
            // run if I don't change the parameter from its 
            // default value of true, to false.
            return veracode_scan
          }
        }
        steps{
          // Sandbox Upload and Scan
          echo "Veracode Static Analysis"
          // load  "/home/myman/.ssh/vCreds" 
          
          
          //Getting Veracode Credentials from Param Store
          //sh "/usr/local/bin/aws ssm get-parameter --name /scan/veracode/vid  --region=us-east-1"
          //sh "/usr/local/bin/aws ssm get-parameter --name /scan/veracode/vpass --region=us-east-1"
          //veracode applicationName: 'Spring', canFailJob: true, createSandbox: true, criticality: 'VeryHigh', debug: true, fileNamePattern: '', pHost: '', pPassword: '', pUser: '', replacementPattern: '', sandboxName: 'myteam', scanExcludesPattern: '', scanIncludesPattern: '*', scanName: 'Jenkins_${BUILD_NUMBER}', timeout: 15, uploadExcludesPattern: '', uploadIncludesPattern: '*.tar.gz', useIDkey: true, vid: "${vid}", vkey: "${vkey}"
        }
      }
      stage ('Unit Test'){
        steps{
          echo "Testing"
          sh "./run_tests.sh"
        }
      }
      stage ('Build Docker Image'){
        steps{
          /* Build the Docker image with a Dockerfile, tagging it with the build number */
          sh "pwd"
          sh "docker build -t ${REPO} ."
          sh "echo ${REPO}"
          sh "docker tag ${REPO} ${REPO}:${env.BUILD_NUMBER}"
          
        }
      }
      stage ('Publish'){
        steps{
          /* Push the image to Docker Hub, using credentials we have setup separately on the worker node */        
   
          withDockerRegistry([credentialsId: 'docker', url: 'https://containers.mycomp.com/']) {
             sh "docker push ${REPO}:${env.BUILD_NUMBER}"
          }
          
        }
      }
      stage ('Deploying ECS Service in AAC'){
        agent {
          docker { 
            image 'containers.mycomp.com/myteam/dev/pipeline_terraform' 
           
          }
        }
         steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding',
                                  credentialsId: 'AWS_AAC_CREDS',
                                  accessKeyVariable: '_AWS_ACCESS_KEY_ID',
                                  secretKeyVariable: '_AWS_SECRET_ACCESS_KEY']]) {
                    script {
                        env.AWS_ACCESS_KEY_ID = env._AWS_ACCESS_KEY_ID
                        env.AWS_SECRET_ACCESS_KEY = env._AWS_SECRET_ACCESS_KEY
                    }
                }
          sh "pwd"
          sh "env"
          checkout([$class: 'GitSCM', branches: [[name: '*/master']], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[credentialsId: 'mymanAPI', url: 'https://github.mycomp.com/mycomp/myteam-sample_pipeline_name_bg_service_aac-blueprint-tf.git']]])
          sh "ls -lart"
          sh "terraform -v"
          sh "id; whoami"
          sh "rm -rf .terraform"
          sh "terraform init"
          
          sh "terraform plan -var-file=global.tfvars -var 'action=deploy' -var 'version=${env.BUILD_NUMBER}'"
          sh "terraform apply -var-file=global.tfvars -var 'action=deploy' -var 'version=${env.BUILD_NUMBER}'"
        }
      }
      
      stage ('Testing Inactive Application on AAC'){
        steps{
            sh "sleep 120"
            sh "curl -v -f http://sampleecs-${env.BUILD_NUMBER}.services.mgmt.nonprod.mycomp.com"
        }
      }

        stage ('Activating Application on AAC'){
            agent {
                docker {
                    image 'containers.mycomp.com/myteam/dev/pipeline_terraform'
                }
            }
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding',
                                  credentialsId: 'AWS_AAC_CREDS',
                                  accessKeyVariable: '_AWS_ACCESS_KEY_ID',
                                  secretKeyVariable: '_AWS_SECRET_ACCESS_KEY']]) {
                    script {
                        env.AWS_ACCESS_KEY_ID = env._AWS_ACCESS_KEY_ID
                        env.AWS_SECRET_ACCESS_KEY = env._AWS_SECRET_ACCESS_KEY
                    }
                }
                sh "pwd"
                sh "env"
                checkout([$class: 'GitSCM', branches: [[name: '*/master']], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[credentialsId: 'mymanAPI', url: 'https://github.mycomp.com/mycomp/myteam-sample_pipeline_name_bg_service_aac-blueprint-tf.git']]])
                sh "ls -lart"
                sh "terraform -v"
                sh "id; whoami"
                sh "rm -rf .terraform"
                sh "terraform init"
                sh "terraform plan -var-file=global.tfvars -var 'action=deliver' -var 'version=${env.BUILD_NUMBER}'"
                sh "terraform apply -var-file=global.tfvars -var 'action=deliver' -var 'version=${env.BUILD_NUMBER}'"
            }
      }

      stage ('Testing Active Application on AAC'){
        steps{
            sh "sleep 120"
            sh "curl -v -f http://sampleecs.services.mgmt.nonprod.mycomp.com"
        }
      }
  
  stage ('Deploying ECS Service in ABP'){
        
        agent {
          docker { 
            image 'containers.mycomp.com/myteam/dev/pipeline_terraform' 
          
          }
        }
      steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding',
                                  credentialsId: 'AWS_ABP_CREDS',
                                  accessKeyVariable: '_AWS_ACCESS_KEY_ID',
                                  secretKeyVariable: '_AWS_SECRET_ACCESS_KEY']]) {
                    script {
                        env.AWS_ACCESS_KEY_ID = env._AWS_ACCESS_KEY_ID
                        env.AWS_SECRET_ACCESS_KEY = env._AWS_SECRET_ACCESS_KEY
                    }
                }
            
        
        
          sh "pwd"
          sh "env"
          checkout([$class: 'GitSCM', branches: [[name: '*/master']], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[credentialsId: 'releaseAPI', url: 'https://github.mycomp.com/mycomp/myteam-sample_pipeline_name_bg_service_abp-blueprint-tf.git']]])
          sh "ls -lart"
          sh "terraform -v"
          sh "id; whoami"
          sh "rm -rf .terraform"
          sh "rm terraform.tfstate.backup"
          sh "terraform init"
          
          sh "terraform plan -var-file=global.tfvars -var 'action=deploy' -var 'version=${env.BUILD_NUMBER}'"
          sh "terraform apply -var-file=global.tfvars -var 'action=deploy' -var 'version=${env.BUILD_NUMBER}'"
        }
      }
      
      stage ('Testing Application on ABP'){
        steps{
          
           
            sh "sleep 10"
            echo "Deployed on ABP"
            //sh "curl -v -f http://sampleecs-${env.BUILD_NUMBER}.services.dev.mgmt.nonprod.mycomp.com"
        }
        
      
      }
      
  }  
    
    
    
    post {
      failure {
        // notify users when the Pipeline fails
        mail to: 'first.last@mycomp.com',
        subject: "Failed Pipeline: ${currentBuild.fullDisplayName}",
        body: "Something is wrong with ${env.BUILD_URL}"
      }
    }
  }
