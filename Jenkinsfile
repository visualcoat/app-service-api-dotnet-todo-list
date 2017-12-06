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
    sh 'mvn clean package'
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
              StrEnvironment="test"              
              StrPrefixCode="aps"
              StrResourceGroupJenkins=$StrPrefixCode"-"${service_name}"-Service-Jenkins"
              StrResourceGroupSolution=$StrPrefixCode"-"${service_name}"-Service-"$StrEnvironment
              StrLocation="WestUS"
              StrStorageAccount=$StrPrefixCode"jenkinsstorage"
              StrContainerName=$StrPrefixCode"armtemplates"
              StrLocalArm="${WORKSPACE}/infrastructure/api"
              StrLocalZip="${WORKSPACE}/api/dist"
              StrFileName="azuredeploy.json"
              StrFileZip=""${service_name}"Service.zip"
       
              az group create --name $StrResourceGroupJenkins --location $StrLocation
              az storage account create --resource-group $StrResourceGroupJenkins --location $StrLocation --sku Standard_LRS --kind Storage --name $StrStorageAccount
              
              connection=$(az storage account show-connection-string --resource-group $StrResourceGroupJenkins --name $StrStorageAccount --query connectionString)
              urlblob=$(az storage account show -g $StrResourceGroupJenkins -n $StrStorageAccount --query primaryEndpoints --output tsv |head -n 2|awk '{print $1}')
              accesskey1=$(az storage account keys list --account-name $StrStorageAccount --resource-group $StrResourceGroupJenkins --output tsv|head -n 1|awk '{print $3}')
              
              az storage container create --name $StrContainerName --public-access off --connection-string $connection
              azcopy --source $StrLocalArm --destination $urlblob$StrContainerName --dest-key $accesskey1 --recursive --quiet
              azcopy --source $StrLocalZip --destination $urlblob$StrContainerName --dest-key $accesskey1 --recursive --quiet
                
                echo "Running Azure Delegator Build"
                expiretime=$(date -u -d '30 minutes' +%Y-%m-%dT%H:%MZ)
                token=$(az storage blob generate-sas --container-name $StrContainerName --name $StrFileName --expiry $expiretime --permissions r --output tsv --connection-string $connection)
                tokenZip=$(az storage blob generate-sas --container-name $StrContainerName --name $StrFileZip --expiry $expiretime --permissions r --output tsv --connection-string $connection)
                urlFile=$(az storage blob url --container-name $StrContainerName --name $StrFileName --output tsv --connection-string $connection)
                urlZip=$(az storage blob url --container-name $StrContainerName --name $StrFileZip --output tsv --connection-string $connection)
                ServicePrincipalAppId=$(az keyvault secret show --name "keyVaultServicePrincipalAppId" --vault-name "ICDelegatorJenkinsKeys" --output tsv|head -n 1|awk '{print $6}')
                ServicePrincipalSecret=$(az keyvault secret show --name "keyVaultServicePrincipalSecret" --vault-name "ICDelegatorJenkinsKeys" --output tsv|head -n 1|awk '{print $6}')
                TemplateStorageAccountAccessKey=$(az keyvault secret show --name "azTmplStorageAccountAccessKey" --vault-name "ICDelegatorJenkinsKeys" --output tsv|head -n 1|awk '{print $6}')
                az group create --name $StrResourceGroupSolution --location $StrLocation
                az group deployment create --resource-group $StrResourceGroupSolution --template-uri "https://raw.githubusercontent.com/visualcoat/app-service-api-dotnet-todo-list/master/azuredeploy.json" --parameters parm_msdeploy_packageurl="$urlZip?$tokenZip" parm_build_environment=$StrEnvironment parm_build_prefixcode=$StrPrefixCode parm_keyVaultServicePrincipalAppId=$ServicePrincipalAppId parm_keyVaultServicePrincipalSecret=$ServicePrincipalSecret parm_azTmplStorageAccountAccessKey=$TemplateStorageAccountAccessKey
            '''
    }
    // get publish settings
    def pubProfilesJson = sh script: "az webapp deployment list-publishing-profiles -g $resourceGroup -n $webAppName", returnStdout: true
    def ftpProfile = getFtpPublishProfile pubProfilesJson
    // upload package
    sh "curl -T target/calculator-1.0.war $ftpProfile.url/webapps/ROOT.war -u '$ftpProfile.username:$ftpProfile.password'"
    // log out
    sh 'az logout'
  }
}
