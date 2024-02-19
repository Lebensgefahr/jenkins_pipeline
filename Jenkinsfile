import static groovy.json.JsonOutput.*

def dockerHub = "docker-hub.domain.com"
def dockerHubCredId = "docker-hub.domain.com"

def dockerBuildRepo = "git@gl.domain.com:customers/docker_build.git"
def dockerBuildRepoBranch = "main"
def dockerBuildRepoCredId = "3ca09abb-c2a8-4123-8c47-f798574b0f57"

def ansibleRepo = "git@gl.domain.com:customers/gmc/ansible.git"
def ansibleRepoBranch = "staging"
def ansibleRepoCredId = "3ca09abb-c2a8-4123-8c47-f798574b0f57"

def ansibleVarsFile = './staging/group_vars/all.yml'

def services = [:]

def generatedParams = []

def causes = currentBuild.getBuildCauses()

def mapParams = [
  'DEPLOY':['type':'boolean', 'defaultValue':true, 'description':'Uncheck to build and publish images without deploying'],
  'REBUILD':['type':'boolean', 'defaultValue':false, 'description':'WARNING: All images will be rebuilded and deployed'],
  'app-frontend':['type':'string', 'defaultValue':'app/app-ui/1200', description: ''],
  'app':['type':'string', 'defaultValue':'app/mw-main.multibranch/bugfix%2F90936/6', description: ''],
  'app-ptz':['type':'string', 'defaultValue':'app/ptz-driver.multibranch/master/183', description: ''],
  'app-api':['type':'string', 'defaultValue':'app/api-proxy.multibranch/master/597', description: ''],
  'app-token':['type':'string', 'defaultValue':'app/token-service.multibranch/bugfix%2F69701/22', description: ''],
  'app-url':['type':'string', 'defaultValue':'app/url-service.multibranch/master/52', description: ''],
  'app-video':['type':'string', 'defaultValue':'app/video-service.multibranch/master/169', description: ''],
  'subs':['type':'string', 'defaultValue':'app/subs.multibranch/master/111', description: ''],
  'app-snapshot':['type':'string', 'defaultValue':'app/snapshot-service.multibranch/master/36', description: ''],
  'parser':['type':'string', 'defaultValue':'app/parser/parser.multibranch/staging%2Fgmc/1', description: ''],
  'app-ms':['type':'string', 'defaultValue':'app/ms.multibranch/master/523', description: ''],
  'app-gps':['type':'string', 'defaultValue':'app/gps-service.multibranch/master/45', description: ''],
  'app-amq-client':['type':'string', 'defaultValue':'app/amq-client.multibranch/master/77', description: ''],
  'app-notice':['type':'string', 'defaultValue':'app/notice.multibranch/master/4', description: ''],
  'app-ptz-hc':['type':'string', 'defaultValue':'app/ptz-hc-service.multibranch/master/19', description: '']
]

def checkoutGitRepository(path, url, branch, credentialsId = null, poll = true, timeout = 10, depth = 0){
    dir(path) {
        checkout(
            changelog:true,
            poll: poll,
            scm: [
                $class: 'GitSCM',
                branches: [[name: "*/${branch}"]],
            doGenerateSubmoduleConfigurations: false,
            extensions: [
                [$class: 'CheckoutOption', timeout: timeout],
                [$class: 'CloneOption', depth: depth, noTags: false, reference: '', shallow: depth > 0, timeout: timeout]],
            submoduleCfg: [],
            userRemoteConfigs: [[url: url, credentialsId: credentialsId]]]
        )
    }
}

node {

  def parametersFile = "${WORKSPACE}".substring(0, "${WORKSPACE}".lastIndexOf("/"))+'/.'+"${JOB_BASE_NAME}"+'_build_parameters.yml'

  try {
    def rewriteYaml = false
    parameters = readYaml file: "${parametersFile}"
    if (mapParams.size() != parameters.size()) {
      println "The number of parameters in the file is less than the default so overwrite it."
      writeYaml file: "${parametersFile}", data: mapParams, overwrite: true
      parameters = readYaml file: "${parametersFile}"
    }
    params.keySet().each { name ->
      if(params."${name}" instanceof String) {
        if (params."${name}" != parameters."${name}".defaultValue) {
          rewriteYaml = true
          parameters."${name}".defaultValue = params."${name}"
        }
      }
    }
    if(rewriteYaml) {
      writeYaml file: "${parametersFile}", data: parameters, overwrite: true
      parameters = readYaml file: "${parametersFile}"
    }
  } catch (Exception e) {
    params.keySet().each { name ->
      if(params."${name}" && mapParams."${name}"){
        if(params."${name}" instanceof String) {
          if (params."${name}" != mapParams."${name}".defaultValue) {
            mapParams."${name}".defaultValue = params."${name}"
          }
        }
      }
    }
    writeYaml file: "${parametersFile}", data: mapParams
    parameters = readYaml file: "${parametersFile}"
  }

  parameters.keySet().each { name ->
    switch(parameters."${name}".type) {
      case 'string':
        generatedParams += [string(name: "${name}", defaultValue: parameters."${name}".defaultValue, description: parameters."${name}".description),]
        break
      case 'boolean':
        generatedParams += [booleanParam(name: "${name}", defaultValue: parameters."${name}".defaultValue, description: parameters."${name}".description),]
        break
    }
  }
  properties([
    parameters(
      generatedParams
    )
  ])
}

pipeline {
//  agent any
  options {
    disableConcurrentBuilds()
  }
  agent {
    label 'el7-docker'
  }
  stages {
    stage ('Process build parameters') {
      steps {
        script {
          cleanWs()
          params.keySet().each { name ->
            if(params."${name}" instanceof String){
              def path = params."${name}"
              def service = name
              def value = path.split("/")
              switch(value[1]) {
                case "app-ui":
                  services += [(service):['fullProjectName': value[0]+"/"+value[1], 'folderName': value[0], 'projectName': value[1], 'branchName': "master", 'buildNumber': value[2], 'imageExists': false, 'image': '', 'oldImage': '']]
                  break
                case "parser":
                  services += [(service):['fullProjectName': value[0]+"/"+value[1]+"/"+value[2]+"/"+value[3], 'folderName': value[0], 'projectName': value[2], 'branchName': value[3], 'buildNumber': value[4], 'imageExists': false, 'image': '', 'oldImage': '']]
                  break
                default:
                  services += [(service):['fullProjectName': value[0]+"/"+value[1]+"/"+value[2], 'folderName': value[0], 'projectName': value[1], 'branchName': value[2], 'buildNumber': value[3], 'imageExists': false, 'image': '', 'oldImage': '']]
                  break
              }
            }
          }
          println prettyPrint(toJson(services))
        }
      }
    }
    stage ('Clone docker build repository') {
      steps {
        script {
          path = "${WORKSPACE}/docker_build"
          checkoutGitRepository(path, "${dockerBuildRepo}", "${dockerBuildRepoBranch}", "${dockerBuildRepoCredId}")
        }
      }
    }
    stage ('Download artifacts') {
      steps {
        script {
          def artifactDownloading = [:]
          services.keySet().each { service ->
            artifactDownloading[service] = {
              path = "${WORKSPACE}/docker_build/${service}/target"
              dir(path) {
                if (params.REBUILD) {
                  copyArtifacts(projectName: services."${service}".fullProjectName, selector: specific(services."${service}".buildNumber), flatten: true)
                } else {
                  try {
                    withCredentials([usernamePassword(credentialsId: "${dockerHubCredId}", usernameVariable: 'username', passwordVariable: 'password')]) {
                      def url = "https://$username:$password@${dockerHub}/v2/${services."${service}".folderName}/${service}/${services."${service}".branchName.replaceAll("%2F", "/")}/manifests/${services."${service}".buildNumber}"
                      httpRequest("$url")
                      services."${service}".imageExists = true
                    }
                  } catch (Exception e) {
                    copyArtifacts(projectName: services."${service}".fullProjectName, selector: specific(services."${service}".buildNumber), flatten: true)
                  }
                }
              }
            }
          }
          parallel artifactDownloading
          def imageExistsServices = services.findAll { key, values ->
            services."${key}".imageExists == true
          }
          println prettyPrint(toJson(imageExistsServices))
        }
      }
    }
    stage ('Build docker images') {
      steps {
        script {
          def parallelBuild = [:]
          def path = "${WORKSPACE}/docker_build/"
          services.keySet().each { service ->
            parallelBuild[service] = {
              dir(path) {
                services."${service}".image = "${dockerHub}"+"/"+services."${service}".folderName+"/"+"${service}"+"/"+services."${service}".branchName.replaceAll("%2F", "/")+":"+services."${service}".buildNumber
                if(!services."${service}".imageExists) {
                  sh "find ${service}/target -type f"
                  sh(script: "./build.sh build ${service} ${services."${service}".folderName} ${services."${service}".projectName} ${services."${service}".branchName.replaceAll("%2F", "/")} ${services."${service}".buildNumber}")
//                  println "./build.sh build ${service} ${services."${service}".folderName} ${services."${service}".projectName} ${services."${service}".branchName.replaceAll("%2F", "/")} ${services."${service}".buildNumber}"
                }
              }
            }
          }
          parallel parallelBuild
          def imageExistsServices = services.findAll { key, values ->
            services."${key}".imageExists != true
          }
          println prettyPrint(toJson(imageExistsServices))
        }
      }
    }
    stage ('Push docker images') {
      steps {
        script {
          def parallelBuild = [:]
          def path = "${WORKSPACE}/docker_build/"
          services.keySet().each { service ->
            parallelBuild[service] = {
              dir(path) {
                services."${service}".image = "${dockerHub}"+"/"+services."${service}".folderName+"/"+"${service}"+"/"+services."${service}".branchName.replaceAll("%2F", "/")+":"+services."${service}".buildNumber
                if(!services."${service}".imageExists) {
                  sh(script: "./build.sh push ${service} ${services."${service}".folderName} ${services."${service}".projectName} ${services."${service}".branchName.replaceAll("%2F", "/")} ${services."${service}".buildNumber}")
//                  println "./build.sh push ${service} ${services."${service}".folderName} ${services."${service}".projectName} ${services."${service}".branchName.replaceAll("%2F", "/")} ${services."${service}".buildNumber}"
                }
              }
            }
          }
          parallel parallelBuild
          def imageExistsServices = services.findAll { key, values ->
            services."${key}".imageExists != true
          }
          println prettyPrint(toJson(imageExistsServices))
        }
      }
    }
    stage ('Clone deploy repository and rewrite images file') {
      when {
        expression { params.DEPLOY == true }
      }
      steps {
        script {
          def comments = ' -m "'+'Started by: '+causes.userName.toString().replaceAll("[\\[\\]]", "")+' URL: '+env.BUILD_URL+'"'
          path = "${WORKSPACE}/ansible"
          checkoutGitRepository(path, "${ansibleRepo}", "${ansibleRepoBranch}", "${ansibleRepoCredId}")
          dir(path) {
            images = readYaml file: "${ansibleVarsFile}"
            images.portal_images.keySet().each { service ->
              def serviceWOUnderscores = service.replaceAll("_", "-")
              if(services."${serviceWOUnderscores}") {
                if(services."${serviceWOUnderscores}".image != '') {
                  services."${serviceWOUnderscores}".oldImage = images.portal_images."${service}"
                  images.portal_images."${service}" = services."${serviceWOUnderscores}".image
                  if (services."${serviceWOUnderscores}".oldImage != services."${serviceWOUnderscores}".image) {
                    comments += ' -m'+' "SERVICE: '+service+' IMAGE: '+services."${serviceWOUnderscores}".oldImage+' -> '+services."${serviceWOUnderscores}".image+'"'
                  }
                }
              }
            }
            println prettyPrint(toJson(services))
            sh 'git config --global user.email "jenkins@domain.com"'
            sh 'git config --global user.name "jenkins http://ci.domain.com"'
            sh "git checkout ${ansibleRepoBranch} --"

            writeYaml file: "${ansibleVarsFile}", data: images, overwrite: true

            sh "echo $BUILD_NUMBER > .build_number"
            sh "git add ${ansibleVarsFile} .build_number"

            sh "git commit ${comments}"
            withCredentials([sshUserPrivateKey(credentialsId: "${ansibleRepoCredId}", keyFileVariable: 'SSH_KEY')]){
              sh 'GIT_SSH_COMMAND="ssh -i $SSH_KEY" git push'
            }
          }
        }
      }
    }
  }
}
