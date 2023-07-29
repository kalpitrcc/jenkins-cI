Jenkins pipeline

@Library("share")
def issueLinks
def testResult
def branchName
def buildVersion
def commitData
def commitHash

// Change this email address field to specify where success and failure notifications are sent
// Use a comma (',') to separate email addresses (eg. 'ianc@xyz.com , bob@yyz.org')

emailTo = 'pankajrawat@hkjc.org.hk'

// Change the Jenkins slave node according to the availability
buildnode = 'DEVBUILD53'

node(buildnode){
    try{
        properties([
          buildDiscarder(
                logRotator(artifactDaysToKeepStr: '10', artifactNumToKeepStr: '10', daysToKeepStr: '10', numToKeepStr: '20')),
            parameters([
            gitParameter(branch:'',
                        branchFilter:'origin/(.*)', 
                        defaultValue:'dev', 
                        description:'', 
                        name:'BRANCH_NAME', 
                        quickFilterEnabled: true, 
                        selectedValue:'DEFAULT', 
                        sortMode:'ASCENDING_SMART', 
                        tagFilter:'*', 
                        listSize: '10',
                        type:'PT_BRANCH_TAG',
                    useRepository:'ssh://git@devgit01.corpdev.hkjc.com:7999/ites/ai.dsw.git'),
            choice(choices: 'SNAPSHOT\nRELEASE', description:'SNAPSHOT OR RELEASE', name: 'snapshotorrelease'),
            booleanParam(defaultValue: true, description: '', name: 'enableSonarQube'),
            booleanParam(defaultValue: true, description: 'enableAquaScan or not', name: 'enableAquaScan'),
            booleanParam(defaultValue: true, description: 'enableBlackDuckScan or not', name: 'enableBlackDuckScan'),
            booleanParam(defaultValue: true, description: 'enableFortifyScan or not', name: 'enableFortifyScan'),
          ]),
        ])
 
  stage ('Clone') {
            //clean workspace if needed
            cleanWs() 
            deleteDir() 
            checkout([$class: 'GitSCM', 
                            branches: [[name: "$BRANCH_NAME"]], 
                            userRemoteConfigs: [[url: 'ssh://git@devgit01.corpdev.hkjc.com:7999/ites/ai.dsw.git']]]
                          )
                
            logOutput = sh returnStdout: true, script: 'git log -n 1'

                  // pass this for print content to jenkins console log
                  // this has print/println/printf methods
                  sCommit.processOut(this, logOutput)
            
            commitData = sCommit.getAll()
            commitHash = commitData.CommitId
            issueLinks = sCommit.getIssueLinks()      
            //branchName = sCommit.getBranchName(this)
            branchName = "${params.BRANCH_NAME}"
      }
      

  stage ('SonarQube analysis') {
    if (params.enableSonarQube){
        withSonarQubeEnv('SonarQube') {
          SONAR_PROJECT = "${env.JOB_NAME.replace('ITES/AI.DSW/','')}"
                  SONAR_VERSION = "${env.BUILD_NUMBER}-${BRANCH_NAME}"
          logOutput = sh ''' /opt/sonar-scanner-3.3.0.1492-linux/bin/sonar-scanner \
          -Dsonar.projectKey=ITES_AI.DSW_ \
        -Dsonar.sources=. \
        '''
        }
     }
   }

  stage("Quality gate") {
    steps {
        waitForQualityGate abortPipeline: true
    }
  }

  stage ('Docker Image Build - SNAPSHOT') {
    when {
            expression { 
                   return params.snapshotorrelease == 'SNAPSHOT'
                }
          }
      	 
    steps {
          sh '''
              #!/bin/bash
              docker build -t ai.dsw-docker-snapshot-local/modeltraining:${BUILD_NUMBER} -f Dockerfile.training .
              ''' 
          }
    steps  {
          sh '''
             #!/bin/bash
             docker push ai.dsw-docker-snapshot-local/modeltraining:${BUILD_NUMBER}
             '''
       }
  }

  stage ('Docker Image Build - RELEASE') {
    when {
            expression { 
                   return params.snapshotorrelease == 'nRELEASE'
                }
          }
    steps {
          sh '''
              #!/bin/bash
              docker build -t ai.dsw-docker-release-local/modeltraining:${BUILD_NUMBER} -f Dockerfile.training .
              ''' 
          }
    steps  {
          sh '''
             #!/bin/bash
             docker push ai.dsw-docker-release-local/modeltraining:${BUILD_NUMBER}
             '''
       }
  }
  

  stage('Downstream Job: Aqua scan'){

        if ("${params.enableAquaScan}" == "true")
        {    
            echo 'Add Aqua Scan'
            echo "Docker Image: ${DOCKER_REPO}"
            build job: 'security/aqua', parameters: 
            [
             string(name: 'DOCKER_IMAGE', value: "${env.DOCKER_REPO}:${BUILD_NUMBER}"),
             string(name: 'JOB_BASE_NAME', value: "${JOB_BASE_NAME}"),
            ], propagate: false, wait: false        
            echo "DOCKER_IMAGE: ${env.DOCKER_REPO}:${BUILD_NUMBER}"
        }
        
    }

  stage('Downstream Job: Fortify scan'){
        if ("${params.enableFortifyScan}" == "true")
        {       
            echo 'Add Fortify Scan'
            build job: 'security/Fortify', parameters: 
            [
              string(name: 'BRANCH_SELECTOR', value: "${BRANCH_NAME}"),
              string(name: 'JOB_BASE_NAME', value: "${JOB_BASE_NAME}"),
                
            ], propagate: false, wait: false
            
        }
      } 

  stage('Downstream Job: BlackDuck Scan'){
        if ("${params.enableBlackDuckScan}" == "true")
        {
            echo 'Add BlackDuck Scan'
            build job: 'security/BlackDuck', parameters: 
            [
                string(name: 'BRANCH_SELECTOR', value: "${BRANCH_NAME}"),
                 string(name: 'JOB_BASE_NAME', value: "${JOB_BASE_NAME}"),
                ], propagate: false, wait: false
            }
        }

    } catch(e) {
        currentBuild.result = "FAILED"
        // notifyFailed()
        throw e
    }
}
  

