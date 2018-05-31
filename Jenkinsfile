#!groovy

/*
 * Copyright © 2017, 2018 IBM Corp. All rights reserved.
 *
 * Licensed under the Apache License, Version 2.0 (the "License"); you may not use this file
 * except in compliance with the License. You may obtain a copy of the License at
 *
 * http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software distributed under the
 * License is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND,
 * either express or implied. See the License for the specific language governing permissions
 * and limitations under the License.
 */

 // Get the IP of a docker container
def hostIp(container) {
  sh "docker inspect -f '{{.NetworkSettings.IPAddress}}' ${container.id} > hostIp"
  readFile('hostIp').trim()
}

def getEnvForSuite(name, hostIp) {
  CLOUDANT_ENV = ['DB_HTTP=https', 'DB_HOST=clientlibs-test.cloudant.com', 'DB_PORT=443', 'DB_IGNORE_COMPACTION=true']
  CONTAINER_ENV = ['DB_HTTP=http', 'DB_PORT=5984', 'DB_IGNORE_COMPACTION=false']
  def envVars = [
    'SKIP_DB_UPDATES=1' //Currently disabled
  ]
  switch(name) {
    case 'apache/couchdb:1.7.1':
      envVars.addAll(CONTAINER_ENV)
      envVars.add('DB_PORT=5984')
    case 'apache/couchdb:2.1.0':
      envVars.addAll(CONTAINER_ENV)
      envVars.add('ADMIN_PARTY=true')
      envVars.add("DB_URL=http://${hostIp}:5984")
      break
    case 'cloudant':
      envVars.addAll(CLOUDANT_ENV)
      envVars.add("CLOUDANT_ACCOUNT=${env.DB_USER}")
      envVars.add('RUN_CLOUDANT_TESTS=1')
      break
    case 'ibmcom/cloudant-developer':
      envVars.addAll(CONTAINER_ENV)
      envVars.add('RUN_CLOUDANT_TESTS=1')
      envVars.add('DB_USER=admin')
      envVars.add('DB_PASSWORD=pass')
      envVars.add("DB_URL=http://${hostIp}:80")
      break
    default:
      error("Unknown test suite environment ${suiteName}")
  }
  return envVars
}

def test_python(pythonVersion, name) {
  node {
    def runInfo = [imageName: name, envVars: getEnvForDest(dest)]
    // Add test suite specific environment variables
    if (name == 'cloudant') {
      runInfo.envVars.add('CREDS_ID=clientlibs-test')
      runInfo.creds = [usernamePassword(credentialsId: 'clientlibs-test', usernameVariable: 'DB_USER', passwordVariable: 'DB_PASSWORD')]
      runIn.dockerEnv(runInfo, test_python_exec(pythonVersion, getEnvForSuite(name, null)))
    } else {
      def args
      def port
      switch(name) {
        case 'apache/couchdb:1.7.1':
        case 'apache/couchdb:2.1.0':
          port = '5984'
          args = '-p 5984:5984'
          break
        case 'ibmcom/cloudant-developer':
          port = '8080'
          args = '-p 8080:80'
          break
        default:
          error("Unknown container ${suiteName}")
      }
      docker.image(name).withRun(args) { container ->
        hostIp = hostIp(container)
        sh 'wget -S --spider --retry-connrefused ${env.DB_HTTP}://${env.DB_HOST}:${env.DB_PORT}; done'
        switch(name) {
          case 'apache/couchdb:2.1.0':
            sh 'curl -X PUT localhost:5984/_users'
            sh 'curl -X PUT localhost:5984/_replicator'
            sh 'curl -X PUT localhost:5984/_global_changes'
            break
          case 'ibmcom/cloudant-developer':
            sh 'curl -X PUT localhost:8080/_replicator'
            break
          default:
            break
        }
      // Use the sidecar as the test target host
      runInfo.envVars.add('DB_HOST=$SIDECAR_0')
      // Add credentials for the cloudantdeveloper image
      if (name == 'ibmcom/cloudant-developer') {
        runInfo.envVars.add('CREDS_ID=cloudant-developer')
        runInfo.creds = [usernamePassword(credentialsId: 'cloudant-developer', usernameVariable: 'DB_USER', passwordVariable: 'DB_PASSWORD')]
      }
      runIn.dockerEnvWithSidecars(runInfo, [[imageName: dest]], test_python_exec(pythonVersion, getEnvForSuite(name, hostIp)))
      }
    }
  }
}

// Define the test routine for different python versions
def test_python_exec(pythonVersion, envVars) {
  docker.withRegistry("https://${env.DOCKER_REGISTRY}", 'artifactory') {
    docker.image("${env.DOCKER_REGISTRY}python:${pythonVersion}-alpine").inside('-u 0') {
      // Set up the environment and test
      withEnv(envVars){
        try {
        // Unstash the source in this image
        unstash name: 'source'
        sh """pip install -r requirements.txt
              pip install -r test-requirements.txt
              pylint ./src/cloudant
              nosetests -w ./tests/unit --with-xunit"""
        } finally {
          // Load the test results
          junit 'nosetests.xml'
        }
      }
    }
  }
}

// Start of build
stage('Checkout'){
  // Checkout and stash the source
  node{
    checkout scm
    stash name: 'source'
  }
}
stage('Test'){
  // Run tests in parallel for multiple python versions
  def testAxes = [:]
  ['2.7', '3.6'].each { v ->
    ['apache/couchdb:1.7.1'].each { c ->
      testAxes.put("Python${v}_${c}", {test_python(v, c)})
    }
  }
  parallel(testAxes)
}