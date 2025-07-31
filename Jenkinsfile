pipeline {
    
agent {
    label 'Agent-1'
}

parameters {
  choice choices: ['blue', 'green'], description: 'Choose the deployment environment: Blue or Green', name: 'DEPLOY_ENV'
  choice choices: ['blue', 'green'], description: 'Choose the docker tag for deployment', name: 'DOCKER_TAG'
  booleanParam description: 'Switch traffic between blue and green', name: 'SWITCH_TRAFFIC'
}

environment {
    SCANNER_HOME= tool 'sonar-scanner'
    IMAGE= 'zookl0/bankapp'
    TAG= "${params.DOCKER_TAG}"
    KUBE_NAMESPACE= 'webapps'
}

tools {
  maven 'maven3'
}

stages{
    stage('Check-Out-Code'){
        steps{
            git branch: 'main', credentialsId: 'Github-Credentials', url: 'https://github.com/tajroshith/bankapp-mysql.git'
        }
    }
    stage('Mvn-Compile'){
        steps{
          withMaven(globalMavenSettingsConfig: 'maven-settings', jdk: '', maven: 'maven3', mavenSettingsConfig: '', traceability: true) {
            sh "mvn compile"
           }
        }
    }
    stage('Mvn-Test'){
        steps{
          withMaven(globalMavenSettingsConfig: 'maven-settings', jdk: '', maven: 'maven3', mavenSettingsConfig: '', traceability: true) {
            sh "mvn test"
           }
        }
    }
    stage('Mvn-Package'){
        steps{
          withMaven(globalMavenSettingsConfig: 'maven-settings', jdk: '', maven: 'maven3', mavenSettingsConfig: '', traceability: true) {
            sh "mvn package -DskipTests=true"
           }
        }
    }
    stage('Trivy-Fs-Scan'){
        steps{
            sh "trivy fs --format table -o trivy-fs-report.html ."
        }
    }
    stage('Sonar-Analysis'){
        steps{
          withSonarQubeEnv('sonar-server') {
            sh """ $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectKey=Bankapp -Dsonar.projectName=Bankapp -Dsonar.java.binaries=target/classes """
           }
        }
    }
    stage('Quality-Gate-Check'){
        steps{
          timeout(time: 1, unit: 'HOURS') {
            waitForQualityGate abortPipeline: false
           }
        }
    }
    stage('Publish-Artifact-To-Nexus'){
        steps{
          withMaven(globalMavenSettingsConfig: 'maven-settings', jdk: '', maven: 'maven3', mavenSettingsConfig: '', traceability: true) {
            sh "mvn deploy -DskipTests=true" 
           }
        }
    }
    stage('DockerBuild'){
        steps{
          withCredentials([string(credentialsId: 'Docker_hub_pwd', variable: 'Dhpwd')]) {
            sh "docker login -u zookl0 -p${Dhpwd}"
           }
            sh "docker build -t $IMAGE:$TAG ."
        }
    }
    stage('Trivy-Image-Scan'){
        steps{
            sh "trivy image --format table -o trivy-image-report.html $IMAGE:$TAG"
        }
    }
    stage('DockerPush'){
        steps{
          withCredentials([string(credentialsId: 'Docker_hub_pwd', variable: 'Dhpwd')]) {
            sh "docker login -u zookl0 -p${Dhpwd}"
           }
            sh "docker push $IMAGE:$TAG"
        }
    }
    stage('Deploy SVC'){
        steps{
          withKubeConfig(caCertificate: '', clusterName: 'Vox-Dev-wp-eks-cluster', contextName: '', credentialsId: 'K8s-sa-token', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://F4CB2287C7B426A5210B191A7CF037EA.sk1.ap-south-1.eks.amazonaws.com') {
            sh "kubectl get svc bankapp-service -n ${KUBE_NAMESPACE} || kubectl apply -f app-service.yml -n ${KUBE_NAMESPACE}"
           }
        }
    }
    stage('Deploy-To-K8s'){
        steps{
          script{
          def deploymentFile = params.DEPLOY_ENV == 'blue' ? 'app-deployment-blue.yml' : 'app-deployment-green.yml'
          withKubeConfig(caCertificate: '', clusterName: 'Vox-Dev-wp-eks-cluster', contextName: '', credentialsId: 'K8s-sa-token', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://F4CB2287C7B426A5210B191A7CF037EA.sk1.ap-south-1.eks.amazonaws.com') {
            sh "kubectl apply -f secret.yml -n ${KUBE_NAMESPACE}"
            sh "kubectl apply -f configmap.yml -n ${KUBE_NAMESPACE}"     
            sh "kubectl apply -f pv.yml -n ${KUBE_NAMESPACE}"  
            sh "kubectl apply -f pvc.yml -n ${KUBE_NAMESPACE}"
            sh "kubectl apply -f mysql-deployment.yml -n ${KUBE_NAMESPACE}"
            sleep 20
            sh "kubectl apply -f ${deploymentFile} -n ${KUBE_NAMESPACE}"
            }
         }
      }
    }
    stage('Switch-Traffic'){
        when {
            expression { params.SWITCH_TRAFFIC }
        }
        steps{
          withKubeConfig(caCertificate: '', clusterName: 'Vox-Dev-wp-eks-cluster', contextName: '', credentialsId: 'K8s-sa-token', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://F4CB2287C7B426A5210B191A7CF037EA.sk1.ap-south-1.eks.amazonaws.com') {
            sh "kubectl set selector service bankapp-service app=bankapp,version=${params.DEPLOY_ENV}"
            }
        }
    }
    stage('Verify-Deployment'){
        steps{
            sh "kubectl get pods -l version=${params.DEPLOY_ENV} -n ${KUBE_NAMESPACE}"
            sh "kubectl get svc bankapp-service -n ${KUBE_NAMESPACE}"
        }
    }
}//Stages Closing
}//Pipeline Closing
