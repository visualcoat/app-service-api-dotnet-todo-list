import groovy.json.JsonSlurper

def getFtpPublishProfile(def publishProfilesJson) {
  def pubProfiles = new JsonSlurper().parseText(publishProfilesJson)
  for (p in pubProfiles)
    if (p['publishMethod'] == 'FTP')
      return [url: p.publishUrl, username: p.userName, password: p.userPWD]
}

node {
  stage('init') {
    checkout scm
  }
  
  stage('build') {
 //   sh 'mvn clean package'
  }
  
  stage('deploy') {
    def resourceGroup = 'APS-DEV-ResGroup' 
    def webAppName = 'APSDevApp'
    // login Azure
    withCredentials([azureServicePrincipal('sp-jenkin')]) {
      sh '''
        az login --service-principal -u $AZURE_CLIENT_ID -p $AZURE_CLIENT_SECRET -t $AZURE_TENANT_ID
        az account set -s $AZURE_SUBSCRIPTION_ID
      '''
        sh '''
az group create --name "SL-DEV-ToDoList" --location "WestUS"
az group deployment create --resource-group "SL-DEV-ToDoList" --template-uri "https://raw.githubusercontent.com/visualcoat/app-service-api-dotnet-todo-list/master/azuredeploy.json" --parameters siteName="aps-dev-todo-list" hostingPlanName="free" siteLocation="West US" 
            '''
    }
    // get publish settings
    //def pubProfilesJson = sh script: "az webapp deployment list-publishing-profiles -g $resourceGroup -n $webAppName", returnStdout: true
    //def ftpProfile = getFtpPublishProfile pubProfilesJson
    // upload package
    //sh "curl -T target/calculator-1.0.war $ftpProfile.url/webapps/ROOT.war -u '$ftpProfile.username:$ftpProfile.password'"
    // log out
    sh 'az logout'
  }
}
