/*
This Jenkinsfile is part of the Pipeline as Code defined in the book. 
It will be used to demostrate how to get started doing DevOps with API Connect v10
It will only be showing how to pull from Git, and then deploy to DEV and running unit tests.
The capabilities are here to deploy to Dev, Test, Staging.
The sample is running on one instance of API Connect therefore we are using multiple catalogs to represent Dev and Test environments
*/

//ibraries
import jenkins.model.*
import hudson.model.*
import hudson.util.Secret
import hudson.AbortException
// java
import java.io.File
import java.text.SimpleDateFormat
import org.apache.commons.lang3.time.DateUtils
// Jenkins

import org.jenkinsci.plugins.scriptsecurity.sandbox.RejectedAccessException

// Provide your GitHub information here
jenkinsfileURL = "https://github.com/janibasha1984/apic_jenkins" 
jenkinsfileBranch = "master"

//Credential objects defined in Jenkins
gitCredentials = "719d79c8-60aa-4d89-97b9-5adedd3d0b81"

//Product yaml file
def product = "weather-product_1.0.0.yaml" //Eg: "Chapter 13 member_1.0.2.yaml"
//Name of the API Product in Chapter 13
def productName = "weather-product" //Eg: "Chapter 13 product reference"
// These are hard coded as an example for API Hooks
def apikey = "706724d1-de96-43a6-9854-928a8ad17b2f"
def apisecret = "86062657054832da7c24420167f095fb7fd6123db6fb022cada76d262a2e5cb8"
def testurl = "https://hub.apicisoa.com/app/api/rest/v1/68d05760-98be-4ee6-b9a4-d4d188d31ee3867/tests/run"
def runtestStatus = "0"

node('JenkinsNode') { //This is the worker node which is defined in Jenkins Master.  I only built one. 
    try{
       bat echo "Workspace: ${env.WORKSPACE}"        

        //Checkout the code 
        GitCheckout(env.WORKSPACE, jenkinsfileURL, jenkinsfileBranch, gitCredentials)

        //Load the apic & jenkins variables from the property files
        def yourBuild = readProperties file: 'environment.properties'
        def jenkins = readProperties file: 'jenkins.properties'

        bat """
            cd ${workspace}
            echo "${yourBuild.devServer}"
            echo "${yourBuild.devOrg}"
            echo "${yourBuild.devRealm}"
            echo "${yourBuild.devCatalog}"
        """
 
        //Provide the option to choose the deployment environment
        def environmentChoices = ['Dev', 'Test', 'Stage'].join('\n')
        def environment = null
        environment = input(message: "Choose the publishing target environment ?",
                    parameters: [choice(choices: environmentChoices, name: 'Environment')])       
        bat """
            echo "${environment}"
        """
        //This method will accept the apic license.
        //Not needed in the container environment as it is already done while building the image.         
        Apic_Initiate()    

        //Publish to dev server
        if(environment == 'Dev'){
            Deploy("${yourBuild.devServer}", jenkins.apicCredentials, product, "${yourBuild.devCatalog}", 
                "${yourBuild.devOrg}", "${yourBuild.devRealm}", apikey, apisecret, testurl)
        }

        //Publish to test server
        if(environment == 'Test'){
            Deploy("${yourBuild.testServer}", jenkins["apicCredentials"], product, "${yourBuild.testCatalog}", 
                "${yourBuild.testOrg}", "${yourBuild.testRealm}")   
       } 
               //Publish to Stage server
        if(environment == 'Stage'){
            Deploy("${yourBuild.stageServer}", jenkins["apicCredentials"], product, "${yourBuild.stageCatalog}", 
                "${yourBuild.stageOrg}", "${yourBuild.stageRealm}")   
       }       

    } catch(RejectedAccessException exe)
    {
        throw exe
    } catch (exe)
    {
        echo "${exe}"
        error("[FAILURE] Failed to publish ${product}")
    }
}    

def Apic_Initiate() {
    //You have to accept license when running apic CLI
    bat "echo n| apic --accept-license"    
    
}

 
def GitCheckout(String workspace, String url, String branch, String credentialsId) {
    withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: 'github-credentials-repo-book',
                      passwordVariable: 'password', usernameVariable: 'user']]) {

        // Checkout the code and navigate to the target directory 
        int urlDelim = url.indexOf("://")
        String gitUrlString = url.substring(0, urlDelim + 3) +
               "\"${user}:${password}\"@" + url.substring(urlDelim + 3);        
        
        //Fetch Jenkins workspace
        directoryName = sh (
            script:"""
            basename ${workspace}
            """,
            returnStdout: true
        )
        
        //Clean up old workspace so you can populate with updated GitHub resources
        bat """
            cd ${workspace}
            cd ..
            rm -fr ${directoryName}
            mkdir ${directoryName}
            cd ${directoryName}
            git clone -b ${branch} ${gitUrlString} ${workspace}
        """
    }
}

//Run the publish API to designated API Connect v10 manager.  
def Deploy(String server, String creds, String product, String catalog, String org, String realm, String apikey, String apisecret, String testurl, 
    String space="", String versioning = "false", String stagingProduct = "", String stagingProductname = "", String newVersion = "", 
    String productPlanMapFile = "") {
        //Login to APIM Server
        try {
            echo "Attempting Login to ${server}"                     
            Login(server, creds, realm)
            //Method Requiring Script Approval Here
        } catch(RejectedAccessException exe)
        {
            throw exe                                 
        } catch(exe){
            echo "Failed to Login to ${server} with Status Code: ${exe}"
            throw exe                       
        }  

        if (versioning == "false") {
            //Publish to APIM server
            def status = Publish(product, catalog, org, server, space)
            //If the status code is non zero i.e. publish failed, abort the pipeline.
            if(status != 0) 
            {
                currentBuild.result = "FAILED"
                error("[FAILURE] Failed to publish ${product}")
            }  
            else {
                def testStatus = Runtest(apikey,  apisecret,  testurl) 
                echo "I have returned from running tests"
                if(testStatus != 0) 
                {
                    currentBuild.result = "FAILED"
                    error("[FAILURE] Unit Tests Failed after publishing ${product}")
                }  
            }
        }
        else {
            //Stage the new product version            
            def status = Stage(stagingProduct, catalog, org, server, space)
            //If the status code is non zero i.e. staging failed, abort the pipeline.
            if(status != 0) 
            {
                currentBuild.result = "FAILED"
                error("[FAILURE] Failed to stage the product ${stagingProduct} ${newVersion}")
            } 

            //Replace the existing product version with the new version 
            status = Replace(stagingProductname, newVersion, productPlanMapFile, catalog, org, server, space)
            if(status != 0) 
            {
                currentBuild.result = "FAILED"
                error("[FAILURE] Failed to replace the product ${productName} with the new version ${newVersion}")
            }             
        }



        //Logout from APIM server
        logoutFailed = false
        try {
            Logout(server)
            echo "Logged out of ${server}"
            //Method Requiring Script Approval Here
        } catch(RejectedAccessException exe)
        {
            throw exe            
        } catch (exe) {
            echo "Failed to Log out with Status Code: ${exe}, check Environment manually."
            logoutFailed = trued
            throw exe
        }    
}

//Login to APIM  server
def Login(String server, String creds, String realm){
    def usernameVar, passwordVar
    withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: "${creds}",
    usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD']]) {                        
        usernameVar = env.USERNAME
        passwordVar = env.PASSWORD
        
      
        bat "apic login --server ${server} --username ${usernameVar} --password ${passwordVar} --realm ${realm}"        
    } 

}

//Logout from APIM server
def Logout(String server){
    sh "apic logout -s ${server}"
}

//Publish artifacts to APIM server

def Publish(String product, String catalog, String org, String server, String space = ""){
    echo "Publishing product ${product}"
    if (!space.trim()) {
        def status = bat script: "apic products:publish ${product} --catalog ${catalog} --org ${org} --server ${server}", 
            returnStatus: true  
        if (status == 0) {                            
            return status             
        }
    }
    else {
        def status = bat script: "apic products:publish --scope space ${product} --space ${space} --catalog ${catalog} --org ${org} --server ${server}", 
            returnStatus: true  
        if (status == 0) {            
            return status             
        }                    
    }    
} 

// Run Unit Tests

// def Runtest(String apikey, String apisecret, String testurl) {
 //   echo "Publishing product on ${testurl}"

    // def status = sh(script: 'curl -k -X POST \
         //   -H X-API-Key:706724d1-de96-43a6-9854-928a8ad17b2f \
        //    -H X-API-Secret:86062657054832da7c24420167f095fb7fd6123db6fb022cada76d262a2e5cb8 -H Content-Type:application/json -d " { "options": {"allAssertions": true,"JUnitFormat": true},
          //  "variables": { string: string, }}" https://hub.apicisoa.com/app/api/rest/v1/68d05760-98be-4ee6-b9a4-d4d188d31ee3867/tests/run', returnStatus: true)
     //   if (status == 0) {            
       //     return status             
       // } 
 //   def results = new XmlParser().parseText(xml)
 //       println "errors = ${results.attribute("errors")}"
 //   runtestStatus = ${results.attribute("errors")}
  //  if (${results.attribute("errors")} == "0") {
        



//}

//Stage the artifacts to Stage catalog on api manager
def Stage(String product, String catalog, String org, String server, String space = "") {
    echo "Staging product ${product}"
    if (!space.trim()) {
        def status = bat script: "apic products:publish --stage ${product} --catalog ${catalog} --org ${org} --server ${server}", 
            returnStatus: true  
        return status  
    }
    else {
        def status = bat script: "apic products:publish --stage --scope space ${product} --space ${space} --catalog ${catalog} --org ${org} --server ${server}", 
            returnStatus: true  
        return status          
    }     
}
