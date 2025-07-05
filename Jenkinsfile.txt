#!groovy
node {
    def BUILD_NUMBER = env.BUILD_NUMBER
    def RUN_ARTIFACT_DIR="tests/${BUILD_NUMBER}"
    def SFDC
    
    def JWT_KEY_CRED_ID = env.JWT_CRED_ID_DH
    def HUB_ORG1 = env.HUB_ORG_DH1
    def HUB_ORG = env.HUB_ORG_DH
    def SFDC_HOST = env.SFDC_HOST_DH
    def CONNECTED_APP_CONSUMER_KEY1 = env.CONNECTED_APP_CONSUMER_KEY_DH1
    def CONNECTED_APP_CONSUMER_KEY = env.CONNECTED_APP_CONSUMER_KEY_DH
    
    

    println 'KEY IS' 
    println JWT_KEY_CRED_ID
    println HUB_ORG
    println HUB_ORG1
    println SFDC_HOST
    println CONNECTED_APP_CONSUMER_KEY
    println CONNECTED_APP_CONSUMER_KEY1

    def toolbelt = tool 'toolbelt'

    stage('Checkout Source') {
        checkout scm // Checks out the code from the main branch for the code changes
    }
           
    withCredentials([file(credentialsId: JWT_KEY_CRED_ID, variable: 'jwt_key_file')]) {
        stage('Authorize ORG1 Org') {
            echo "JWT Key Credential ID: ${env.JWT_CRED_ID_DH}"
            echo "Hub Org: ${env.HUB_ORG_DH_1}"
	    echo "Connected App Consumer Key: ${env.CONNECTED_APP_CONSUMER_KEY_DH1}"
            echo "SFDC Host: ${env.SFDC_HOST_DH}"
			 
            def checkrc = bat returnStatus: true, script: "${toolbelt}sf org login jwt --instance-url ${SFDC_HOST} --client-id ${CONNECTED_APP_CONSUMER_KEY1} --username ${HUB_ORG1} --jwt-key-file ${jwt_key_file} --setalias ORG1"
            echo "SFDC_HOST: ${SFDC_HOST}"

            // Check for successful authorization of ORG1
		
            if (checkrc != 0) {
                error 'ORG1 org authorization failed'
            } else {
                echo 'ORG1 org authorized successfully'
            }
        }

        // Deploying the code to ORG1
        stage('Push To ORG1') { 
            def rc = bat returnStatus: true, script: "${toolbelt}sf project deploy start --target-org ORG1" 
	    if (rc != 0) {
                error 'Salesforce push to ORG1 org failed.' 
            } else {
                echo 'Salesforce push to ORG1 org successful.'   
            }
        }	
            
	    stage('Authorize DevHub Org') {
         echo "Hub Org: ${env.HUB_ORG_DH}"
         echo "Connected App Consumer Key: ${env.CONNECTED_APP_CONSUMER_KEY_DH}"	
		// Deploying code to ORG1
            def checkrc = bat returnStatus: true, script: "${toolbelt}sf org login jwt --instance-url ${SFDC_HOST} --client-id ${CONNECTED_APP_CONSUMER_KEY} --username ${HUB_ORG} --jwt-key-file ${jwt_key_file} --setalias Devhub"
            
            // Check for successful authorization of Devhub
            if (checkrc != 0) {
                error 'DevHub org authorization failed'
            } else {
                echo 'DevHub org authorized successfully'
            }
        }

        // Deploying code to DevHub
        stage('Push To DevHub') { 
            def rc = bat returnStatus: true, script: "${toolbelt}sf project deploy start --target-org Devhub" 
            if (rc != 0) {
                error 'Salesforce push to DevHub org failed.' 
            }
        }
    }
}
