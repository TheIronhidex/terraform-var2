pipeline {
    environment {
	REGION = 'eu-west-3'
        DOCKER_REPO = 'theironhidex'
	NUMBER_CONTAINERS = '1'
      }

 agent any
    tools {
       terraform 'terraform20803'
    }
    stages {   
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
		                IP_EC2=sh (script: "terraform output public_ip", returnStdout:true).trim()
	                  }
	        }
	    }

        stage('Input of new IPs') {
            steps{
	       sh "echo ${IP_EC2} > inventory.hosts"
            }
        }
	    
	stage('Input of new variables') {
            steps{
                sh """
		cat <<EOT > default.yml
		---
		create_containers: ${NUMBER_CONTAINERS}
		default_container_image: ${env.DOCKER_REPO}/${JOB_BASE_NAME}:${BUILD_NUMBER}
		default_container_command: run -d -p80:80
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
