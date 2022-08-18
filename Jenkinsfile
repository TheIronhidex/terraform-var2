pipeline {
    environment {
	REGION = 'eu-west-3'
        DOCKER_REPO = 'theironhidex'
      }

 agent any
    tools {
       terraform 'terraform20803'
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
                sh "echo -e build_number: ${BUILD_NUMBER}/njob_base_name: ${JOB_BASE_NAME} >> default.yml"
	    }
        }
	
	//stage('Wait 5 minutes') {
            //steps {
                //sleep time:5, unit: 'MINUTES'
            //}
        //}
	    
	stage ("Ansible run image") {
            steps {
                ansiblePlaybook become: true, colorized: true, extras: '-v', disableHostKeyChecking: true, credentialsId: 'jose-ssh', installation: 'ansible210', inventory: 'inventory.hosts', playbook: 'playbook-run-docker.yml'
            }
        }
        	    
	stage('terraform destroy') {
            steps{
		    //withCredentials([
		     //aws(accessKeyVariable: 'AWS_ACCESS_KEY_ID', credentialsId: 'aws-jose', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                sh "terraform destroy --auto-approve"
		    //}
	    }
	}
	    
	    stage('Destroy infras?') {
            steps{
                input "Proceed destroying the infrastructure?"
            }
        }
    }
}
