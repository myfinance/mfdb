pipeline {
 agent none

 environment{
   SERVICE_NAME = "mfdb"
   ORGANIZATION_NAME = "myfinance"
   DOCKERHUB_USER = "holgerfischer"

   //Snapshot Version
   VERSION = "0.16.0-alpha.${BUILD_ID}"
   //Release Version
   //VERSION = "0.15.0"

   K8N_IP = "192.168.100.73"
   DB_REPOSITORY_TAG = "${DOCKERHUB_USER}/${ORGANIZATION_NAME}-mfpostgres:${VERSION}"
   MFUPDATE_REPOSITORY_TAG = "${DOCKERHUB_USER}/${ORGANIZATION_NAME}-mfdbupdate:${VERSION}"
   NEXUS_URL = "${K8N_IP}:31001"
   MVN_REPO = "http://${NEXUS_URL}/repository/maven-releases/"
   DOCKER_REPO = "${K8N_IP}:31003/repository/mydockerrepo/"
   TARGET_HELM_REPO = "http://${NEXUS_URL}/repository/myhelmrepo/"
   DEV_NAMESPACE = "mfdev"
   TEST_NAMESPACE = "mftest"
 }
 
 stages{
   stage('preperation'){
    agent {
        docker {
            image 'maven:3.6.3-jdk-11'
        }
    }      
     steps {
       cleanWs()
       git credentialsId: 'github', url: "https://github.com/myfinance/mfdb.git"

     }
   }
   stage('build'){
    agent {
        docker {
            image 'maven:3.6.3-jdk-11'
        }
    }      
     steps {
       sh '''mvn versions:set -DnewVersion=${VERSION}'''
       sh '''mvn clean install'''

     }
   }
   stage('build and push Images'){
    agent {
        docker {
            image 'docker' 
        }
    }        
     steps {
       sh 'docker image build -t ${DB_REPOSITORY_TAG} ./mf-docker-images/target/docker-prep/mfpostgres/'
       sh 'docker image build -t ${MFUPDATE_REPOSITORY_TAG} ./mf-docker-images/target/docker-prep/mfdb/'

       sh 'docker tag ${MFUPDATE_REPOSITORY_TAG} ${DOCKER_REPO}${MFUPDATE_REPOSITORY_TAG}'
       sh 'docker tag ${DB_REPOSITORY_TAG} ${DOCKER_REPO}${DB_REPOSITORY_TAG}'

       sh 'docker push ${DOCKER_REPO}${MFUPDATE_REPOSITORY_TAG}'
       sh 'docker push ${DOCKER_REPO}${DB_REPOSITORY_TAG}'
     }
   }

   stage('deploy to cluster'){
     agent any
     steps {
       //sh 'kubectl delete job.batch/mfupgrade'
       //sh 'envsubst < deploy.yaml | kubectl apply -f -'
       sh 'envsubst < ./helm/mfdb/Chart_template.yaml > ./helm/mfdb/Chart.yaml'
       sh 'helm upgrade -i --cleanup-on-fail mfdb ./helm/mfdb/ -n ${DEV_NAMESPACE} --set repository=${DOCKER_REPO}${DOCKERHUB_USER}/${ORGANIZATION_NAME}-'
       sh 'helm package helm/mfdb -u -d helmcharts/'
       sh 'curl ${TARGET_HELM_REPO} --upload-file helmcharts/mfdb-${VERSION}.tgz -v'
     }
   }
   }
}
