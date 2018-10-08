def project = 'ext-ci-project'
def  appName = 'train-schedule'
def  feSvcName = "${appName}-service"
def  imageTag = "gcr.io/${project}/${appName}:latest"

pipeline {
  agent {
    kubernetes {
      label 'train-schedule'
      defaultContainer 'jnlp'
      yaml """
apiVersion: v1
kind: Pod
metadata:
labels:
  component: ci
spec:
  # Use service account that can deploy to all namespaces
  serviceAccountName: cd-jenkins
  containers:
  - name: gcloud
    image: gcr.io/cloud-builders/gcloud
    command:
    - cat
    tty: true
  - name: kubectl
    image: gcr.io/cloud-builders/kubectl
    command:
    - cat
    tty: true
"""
}
  }
  stages {
    stage('Testing Code') {
      steps {
        container('gcloud') {
          sh """
          kubectl get ns testing || kubectl create ns testing
          kubectl --namespace=testing apply -f k8s/services/
	  kubectl --namespace=testing exec -it train-schedule-deployment-89df87947-jcm4b -- npm test
          """
        }
      }
    }
    stage('Build and push image with Container Builder') {
      steps {
        container('gcloud') {
          sh "PYTHONUNBUFFERED=1 gcloud container builds submit -t ${imageTag} ."
        }
      }
    }
  stage('Deploy Staging') {
      //Staging
      when { branch 'master' }
      steps {
        container('kubectl') {
          // Create namespace if it doesn't exist
	  sh("kubectl delete --all pods --namespace=staging --now")
          sh("kubectl get ns staging || kubectl create ns staging")
          sh("kubectl --namespace=staging apply -f k8s/services/")
          sh("echo To access the Staging environment using the following address with port 8080")
          sh("echo http://`kubectl --namespace=staging get service/train-schedule-service -o jsonpath='{.status.loadBalancer.ingress[0].ip}'`:8080 > ${feSvcName}:8080")
          
        }
      }     
    }
  stage('Deploy Production') {
      // Production branch
      when { branch 'master' }
      steps{
        input 'Deploy to Production?'
         milestone(1)
        container('kubectl') {
	  sh("kubectl delete --all pods --namespace=production --now")      
          sh("kubectl get ns ${env.BRANCH_NAME} || kubectl create ns ${env.BRANCH_NAME}")
          sh("kubectl --namespace=production apply -f k8s/services/")
          sh("echo http://`kubectl --namespace=production get service/train-schedule-service -o jsonpath='{.status.loadBalancer.ingress[0].ip}'`:8080 > ${feSvcName}")
	 
        }
      }
    }

 
  }
   post {
    success {
		slackSend (color: '#00FF00', message: "SUCCESSFUL: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' ")  
    }
    failure {
		slackSend (color: '#FF0000', message: "FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'")
    }
		  
   }
}

