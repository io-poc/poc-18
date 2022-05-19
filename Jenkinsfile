def isSASTEnabled
def isSASTPlusMEnabled
def isSCAEnabled
def isDASTEnabled
def isDASTPlusMEnabled


pipeline {
    agent any
    stages {
        stage('Checkout Source Code') {
            steps {
                git branch: 'master', url: 'https://github.com/io-poc/poc-18'
            }
        }

        stage('Build Source Code') {
            steps {
                  withMaven {
                      sh '''mvn clean package -Dmaven.test.skip'''
                  }
            }
        }

        stage('IO - Prescription') {
            steps {
                synopsysIO(connectors: [
                    io(
                        configName: 'poc-io',
                        projectName: 'insecure-bank',
                        workflowVersion: '2022.4.1'),
                    github(
                        branch: 'master',
                        configName: 'poc-github',
                        owner: 'io-poc',
                        repositoryName: 'poc-18'),
                    blackduck(
                        configName: 'poc-bd',
                        projectName: 'insecure-bank',
                        projectVersion: '1.0')
                    ]) {
                        sh 'io --stage io Persona.Type=devsecops Project.Release.Type=minor'
                    }

                script {
                    def prescriptionJSON = readJSON file: 'io_state.json'

                    isSASTEnabled = prescriptionJSON.data.prescription.security.activities.sast.enabled
                    isSASTPlusMEnabled = prescriptionJSON.data.prescription.security.activities.sastPlusM.enabled
                    isSCAEnabled = prescriptionJSON.data.prescription.security.activities.sca.enabled
                    isDASTEnabled = prescriptionJSON.data.prescription.security.activities.dast.enabled
                    isDASTPlusMEnabled = prescriptionJSON.data.prescription.security.activities.dastPlusM.enabled
                    isImageScanEnabled = prescriptionJSON.data.prescription.security.activities.imageScan.enabled

                }
            }
        }

        
        /*stage('SAST- RapidScan') { environment {
            OSTYPE='linux-gnu' }
            when {
               expression { isSASTEnabled }
            }
            steps {
                echo 'Running SAST using Sigma - Rapid Scan'
                echo env.OSTYPE
                synopsysIO(connectors: [rapidScan(configName: 'poc-sigma')]) {
                sh 'io --stage execution --state io_state.json' }
            }
        } 
        
        stage('SAST - Coverity') {
          when {
            expression { isSASTEnabled }
          }
          steps {
            echo 'Running SAST using Coverity'
            synopsysIO(connectors: [
            coverity(configName: 'poc-coverity'
            )]) {
              sh 'io --stage execution --state io_state.json'
              }
            }
        }*/

        stage('Manual Code Review ') {
            when {
                expression { isSASTPlusMEnabled }
            }
            steps {
                script {
                    input message: 'Manual source code review triggered by IO. Proceed?'
                }
                echo "Out-of-Band Activity - Manual Code Review triggered & approved"
            }
        } 

        stage('SCA - BlackDuck') {
            when {
                expression { isSCAEnabled }
            }
            steps {
              echo 'Running SCA using BlackDuck'
              synopsysIO(connectors: [
                  blackduck(configName: 'poc-bd',
                  projectName: 'insecure-bank',
                  projectVersion: '1.0')]) {
                  sh 'io --stage execution --state io_state.json'
              }
            }
        } 

        stage('Penetration Test') {
            when {
                expression { isDASTPlusMEnabled }
            }
            steps {
                script {
                    input message: 'Penetration Test triggered by IO. Proceed?'
                }
                echo "Out-of-Band Activity - Penetration Test triggered & approved"
            }
        }

        stage('IO - Workflow') {
            steps {
                echo 'Execute Workflow Stage'
                synopsysIO(connectors: [
                    //codeDx(configName: 'poc-codedx', projectId: '1'), 
                    jira(assignee: 'iouser@synopsys.com', configName: 'poc-jira', issueQuery: 'resolution=Unresolved AND labels in (Security, Defect)', projectKey: 'INSEC'), 
                    slack(configName: 'poc-slack')
                ]) {
                    sh 'io --stage workflow --state io_state.json'
                }
                
                 script {
                    def workflowJSON = readJSON file: 'wf-output.json'
                    print("========================== IO WorkflowEngine Summary ============================")
                    print("Breaker Status: $workflowJSON.breaker.status")
                } 
            }
        }
        
        stage('Security Sign-Off') {
            steps {
                script {

                    def workflowJSON = readJSON file: 'wf-output.json'
                    
                    
                    //Build Breaker
                    if(workflowJSON.breaker.status==true) {
                          //exit 1
                    }
                    
                    codedx_value = workflowJSON.summary.risk_score
                    for(arr in codedx_value){
                        if(arr != null)
                        {   
                            print("Code Dx Score: $arr")
                            if(arr < 80)
                            {
                                input message: 'Code Dx Score did not meet the defined threshold. Do you wish to proceed?'
                            }
                        }
                    }
                    
                }
                echo "Security Sign-Off triggered & approved"
            }
        } 
    }

}