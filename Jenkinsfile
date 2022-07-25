

def scan_type
 def host
 def SendEmailNotification(String result) {
  
    // config 
    def to = emailextrecipients([
           requestor()
    ])
    
    // set variables
    def subject = "${env.JOB_NAME} - Build #${env.BUILD_NUMBER} ${result}"
    def content = '${JELLY_SCRIPT,template="html"}'

    // send email
    if(to != null && !to.isEmpty()) {
        env.ForEmailPlugin = env.WORKSPACE
        emailext mimeType: 'text/html',
        body: '${FILE, path="/var/lib/jenkins/workspace/zap-email/report.html"}',
        subject: currentBuild.currentResult + " : " + env.JOB_NAME,
        to: to, attachLog: true
    }
}
			
 pipeline {
    agent any
	parameters {
         	choice  choices: ["Baseline", "APIS", "Full"],
                 	description: 'Type of scan that is going to perform inside the container',
                 	name: 'SCAN_TYPE'
		booleanParam defaultValue: true,
                 	description: 'Parameter to know if wanna generate report.',
                 	name: 'GENERATE_REPORT'
     	}
	tools {
		maven 'Maven'
	}
	
	environment {
		PROJECT_ID = 'tech-rnd-project'
                CLUSTER_NAME = 'jenkins-jen-cluster'
                LOCATION = 'asia-south1-a'
                CREDENTIALS_ID = 'kubernetes'	
	}
	
    stages {
	    stage('Scm Checkout') {
		    steps {
			    checkout scm
		    }
	    }
	    stage('Build') {
		    steps {
			    sh 'mvn clean'
		    }
	    }
			    
	    stage('Test') {
		    steps {
			    echo "Testing..."
			    sh 'mvn test'
		    }
	    }
	    
	    stage('Build Docker Image') {
		    steps {
			    sh 'whoami'
			    script {
				    myimage = docker.build("fazilniveus/devops:${env.BUILD_ID}")
			    }
		    }
	    }
	    
	    stage("Push Docker Image") {
		    steps {
			    script {
				    echo "Push Docker Image"
				    withCredentials([string(credentialsId: 'dockerhub', variable: 'dockerhub')]) {
            				sh "docker login -u fazilniveus -p ${dockerhub}"
				    }
				        myimage.push("${env.BUILD_ID}")
				    
			    }
		    }
	    }
	    
	    stage('Deploy to K8s') {
		    steps{
			    echo "Deployment started ..."
			    sh 'ls -ltr'
			    sh 'pwd'
				sh "sed -i 's/tagversion/${env.BUILD_ID}/g' deployment.yaml"
				echo "Start deployment of deployment.yaml"
				step([$class: 'KubernetesEngineBuilder', projectId: env.PROJECT_ID, clusterName: env.CLUSTER_NAME, location: env.LOCATION, manifestPattern: 'deployment.yaml', credentialsId: env.CREDENTIALS_ID, verifyDeployments: true])
			    	echo "Deployment Finished ..."
			    sh '''
			    '''
			    
		    }
	    }
	    stage('Zap Installation') {
                    steps {
				    
			   
                        sh 'echo "Hello World"'
			sh '''
			    echo "Pulling up last OWASP ZAP container --> Start"
			    docker pull owasp/zap2docker-stable
			    
			    echo "Starting container --> Start"
			    docker run -dt --name owasp \
    			    owasp/zap2docker-stable \
    			    /bin/bash
			    
			    
			    echo "Creating Workspace Inside Docker"
			    docker exec owasp \
    			    mkdir /zap/wrk
			'''
			    }
		    }
	    stage('Scanning target on owasp container') {
             steps {
                 script {
		     sh 'sleep 10'
			sh 'gcloud container clusters get-credentials jenkins-jen-cluster --zone asia-south1-a --project tech-rnd-project'
			sh 'kubectl get pods'	
			sh 'kubectl get service myapp > intake.txt'
			sh """
			
				awk '{print \$4}' intake.txt > extract.txt
                        """
			IP = sh (
        			script: 'grep -Eo "(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\\.(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\\.(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\\.(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)" extract.txt > finalout.txt && ip=$(cat finalout.txt) && aa="http://${ip}" && echo $aa',
        			returnStdout: true
    			).trim()
    			echo "Git committer email: ${IP}"
		 
	      
	    	     
			 
                       scan_type = "${params.SCAN_TYPE}"
                       echo "----> scan_type: $scan_type"
			 
			
		       
			 
                       if(scan_type == "Baseline"){
                           sh """
                               docker exec owasp \
                               zap-baseline.py \
                               -t ${IP} \
                               -r report.html \
                               -I
                           """
                       }
                       else if(scan_type == "APIS"){
                           sh """
                               docker exec owasp \
                               zap-api-scan.py \
                               -t ${IP}\
                               -r report.html \
                               -I
                           """
                       }
                       else if(scan_type == "Full"){
                           sh """
                               docker exec owasp \
                               zap-full-scan.py \
                               -t ${IP}\
                               //-x report.html
                               -I
                            """
                           //-x report-$(date +%d-%b-%Y).xml
                       }
                       else{
                           echo "Something went wrong..."
                       }
			sh '''
				docker cp owasp:/zap/wrk/report.html ${WORKSPACE}/report.html
				echo ${WORKSPACE}
				docker stop owasp
                     	docker rm owasp
			'''
			SendEmailNotification("SUCCESSFUL")
				    
		  }
	     }
	}	

    }
 }
