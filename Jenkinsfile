// IO Prescription Placeholders
def runId
def isSASTEnabled
def isSASTPlusMEnabled
def isSCAEnabled
def isDASTEnabled
def isDASTPlusMEnabled
def isImageScanEnabled
def isNetworkScanEnabled
def isCloudReviewEnabled
def isThreatModelEnabled
def isInfraReviewEnabled
def breakBuild

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
                    
                    ]) {
                        sh 'io --stage io Persona.Type=devsecops Project.Release.Type=minor'
                    }

                 script {
                    // IO-IQ will write the prescription to io_state JSON
                    if (fileExists('io_state.json')) {
                        def prescriptionJSON = readJSON file: 'io_state.json'

                        // Pretty-print Prescription JSON
                        // def prescriptionJSONFormat = JsonOutput.toJson(prescriptionJSON)
                        // prettyJSON = JsonOutput.prettyPrint(prescriptionJSONFormat)
                        // echo("${prettyJSON}")

                        // Use the run Id from IO IQ to get detailed message/explanation on prescription
//                         runId = prescriptionJSON.data.io.run.id
//                         def apiURL = ioServerURL + ioRunAPI + runId
//                         def res = sh(script: "curl --location --request GET ${apiURL} --header 'Authorization: Bearer ${IO_ACCESS_TOKEN}'", returnStdout: true)

//                         def jsonSlurper = new JsonSlurper()
//                         def ioRunJSON = jsonSlurper.parseText(res)
//                         def ioRunJSONFormat = JsonOutput.toJson(ioRunJSON)
//                         def ioRunJSONPretty = JsonOutput.prettyPrint(ioRunJSONFormat)
//                         print("==================== IO-IQ Explanation ======================")
//                         echo("${ioRunJSONPretty}")
//                         print("==================== IO-IQ Explanation ======================")

                        // Update security flags based on prescription
                        isSASTEnabled = prescriptionJSON.data.prescription.security.activities.sast.enabled
                        isSASTPlusMEnabled = prescriptionJSON.data.prescription.security.activities.sastPlusM.enabled
                        isSCAEnabled = prescriptionJSON.data.prescription.security.activities.sca.enabled
                        isDASTEnabled = prescriptionJSON.data.prescription.security.activities.dast.enabled
                        isDASTPlusMEnabled = prescriptionJSON.data.prescription.security.activities.dastPlusM.enabled
                        isImageScanEnabled = prescriptionJSON.data.prescription.security.activities.imageScan.enabled
                        isNetworkScanEnabled = prescriptionJSON.data.prescription.security.activities.NETWORK.enabled
                        isCloudReviewEnabled = prescriptionJSON.data.prescription.security.activities.CLOUD.enabled
                        isThreatModelEnabled = prescriptionJSON.data.prescription.security.activities.THREATMODEL.enabled
                        isInfraReviewEnabled = prescriptionJSON.data.prescription.security.activities.INFRA.enabled
                    } else {
                        error('IO prescription JSON not found.')
                    }
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
                  blackduck(
                      configName: 'poc-bd',
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
                    codeDx(configName: 'poc-codedx', projectId: '1'), 
//                     jira(assignee: 'iouser@synopsys.com', configName: 'poc-jira', issueQuery: 'resolution=Unresolved AND labels in (Security, Defect)', projectKey: 'INSEC'), 
//                     slack(configName: 'poc-slack')
                ]) {
                    sh 'io --stage workflow --state io_state.json'
                }
            }
        }
        
        stage('Security Sign-Off') {
            steps {
                script {
                    if (fileExists('wf-output.json')) {
                        def wfJSON = readJSON file: 'wf-output.json'

                        // If the Workflow Output JSON has a lot of key-values; Jenkins throws a StackOverflow Exception
                        //  when trying to pretty-print the JSON
                        // def wfJSONFormat = JsonOutput.toJson(wfJSON)
                        // def wfJSONPretty = JsonOutput.prettyPrint(wfJSONFormat)
                        // print("======================== IO Workflow Engine Summary ==========================")
                        // print(wfJSONPretty)
                        // print("======================== IO Workflow Engine Summary ==========================")

                        breakBuild = wfJSON.breaker.status
                        print("========================== Build Breaker Status ============================")
                        print("Breaker Status: $breakBuild")
                        print("========================== Build Breaker Status ============================")

                        if (breakBuild) {
                            input message: 'Build-breaker criteria met.'
                        }
                    } else {
                        print('No output from the Workflow Engine. No sign-off required.')
                    }
                }
            }
        } 
    }
}
