pipeline {
    environment {
        GIT_REPO = 'https://github.com/TheIronhidex/terraform-var2'
        GIT_BRANCH = 'main'
	REGION = 'eu-west-3'
        DOCKER_REPO = 'theironhidex'
        CONTAINER_PORT = '87'
	NUMBER_CONTAINERS = '1'
      }

 agent any
    tools {
       terraform 'terraform20803'
    }
    stages {   
        stage ("Get Code") {
            steps {
                git branch: "${env.GIT_BRANCH}", url: "${env.GIT_REPO}"
            }
        }

        stage ("Build Image") {
            steps {
                sh "docker build -t ${env.DOCKER_REPO}/${JOB_BASE_NAME}:${BUILD_NUMBER} ."
            }
        }

        stage ("Publish Image") {
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-hub-jose', passwordVariable: 'docker_pass', usernameVariable: 'docker_user')]) {
                    sh "docker login -u $docker_user -p $docker_pass"
                    sh "docker push ${env.DOCKER_REPO}/${JOB_BASE_NAME}:${BUILD_NUMBER}"
                }
            }
        }

        stage('terraform format check') {
            steps{
                sh 'terraform fmt'
            }
        }
	    
        stage('terraform Init') {
            steps{
                sh 'terraform init'
            }
        }
	    
        stage('Build infras?') {
            steps{
                input "Proceed building the infrastructure?"
            }
        }
        
        stage('terraform apply') {
            steps{
	     withCredentials([
		     aws(accessKeyVariable: 'AWS_ACCESS_KEY_ID', credentialsId: 'aws-jose', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                sh """
		 terraform apply -var=\"region=${env.REGION}\" \
                -var=\"access_key=${AWS_ACCESS_KEY_ID}\" \
                -var=\"secret_key=${AWS_SECRET_ACCESS_KEY}\" \
                --auto-approve
                   """
	            }
		            script {
		                PUBLIC_IP_EC2 = sh (returnStdout: true, script: "terraform output instance public_ip").trim()
	                  }
	        }
	    }

        stage('Input of new IPs') {
            steps{
                sh "echo $PUBLIC_IP_EC2 > inventory.hosts"
            }
	    script {
		IMAGE = "${env.DOCKER_REPO}/${JOB_BASE_NAME}:${BUILD_NUMBER}"
	                  }
        }
	    
	stage('Input of new variables) {
            steps{
                sh """
		cat <<EOT > default.yml
		---
		create_containers: ${NUMBER_CONTAINERS}
		default_container_image: ${IMAGE}
		EOT
		   """
            }
        }
	
	stage('Wait 5 minutes') {
            steps {
                sleep time:5, unit: 'MINUTES'
            }
        }      
	    
	stage ("Ansible run image") {
            steps {
                ansiblePlaybook become: true, colorized: true, extras: '-v', disableHostKeyChecking: true, credentialsId: 'jose-ssh', installation: 'ansible210', inventory: 'inventory.hosts', playbook: 'playbook-run-docker.yml'
            }
        }
        
	    stage('Destroy infras?') {
            steps{
                input "Proceed destroying the infrastructure?"
            }
        }
	    
        stage('Executing Terraform Destroy') {
            steps{
                sh "terraform destroy --auto-approve"
            }
        }
    }   
}
