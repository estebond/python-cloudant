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

def getEnvForDest(dest) {
    // Define the matrix environments
    CLOUDANT_ENV = ['DB_HTTP=https', 'DB_HOST=clientlibs-test.cloudant.com', 'DB_PORT=443']
    CONTAINER_ENV = ['DB_HTTP=http', 'DB_PORT=5984']
    def testEnvVars =[]
    if (dest == 'cloudant') {
       testEnvVars.addAll(CLOUDANT_ENV)
    } else {
       testEnvVars.addAll(CONTAINER_ENV)
       switch(dest) {
           case 'library/couchdb:2.1.1':
               testEnvVars.add('DB_PORT=5984')
               testEnvVars.add('ADMIN_PARTY=true')
               break
           case 'library/couchdb:1.7.1':
               testEnvVars.add('DB_PORT=5984')
               testEnvVars.add('ADMIN_PARTY=true')
               break
           case 'ibmcom/cloudant-developer':
               testEnvVars.add('DB_PORT=80')
               break
           default:
               error("Unknown test env")
       }
    }
    return testEnvVars
 }

def test_python(name, dest) {
  // Define the test routine for different python versions
  def testRun = {
    sh "wget -S --spider --retry-connrefused ${env.DB_HTTP}://${env.DB_HOST}:${env.DB_PORT}; done"
    switch(dest) {
      case 'apache/couchdb:2.1.0':
        httpRequest httpMode: 'PUT', url: "${env.DB_HTTP}://${env.DB_HOST}:${env.DB_PORT}/_replicator", authentication: env.CREDS_ID
        httpRequest httpMode: 'PUT', url: "${env.DB_HTTP}://${env.DB_HOST}:${env.DB_PORT}/_users", authentication: env.CREDS_ID
        httpRequest httpMode: 'PUT', url: "${env.DB_HTTP}://${env.DB_HOST}:${env.DB_PORT}/_global_changes", authentication: env.CREDS_ID
        break
      case 'ibmcom/cloudant-developer':
        httpRequest httpMode: 'PUT', url: "${env.DB_HTTP}://${env.DB_HOST}:${env.DB_PORT}/_replicator", authentication: env.CREDS_ID
        break
      default:
        break
    }
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
    def runInfo = [imageName: name, envVars: getEnvForDest(dest), runArgs: '-u 0']
    // Add test suite specific environment variables
    if (dest == 'cloudant') {
      runInfo.envVars.add('CREDS_ID=clientlibs-test')
      runInfo.creds = [usernamePassword(credentialsId: 'clientlibs-test', usernameVariable: 'DB_USER', passwordVariable: 'DB_PASSWORD')]
      runIn.dockerEnv(runInfo, testRun)
    } else {
      // Use the sidecar as the test target host
      runInfo.envVars.add('DB_HOST=$SIDECAR_0')
      // Add credentials for the cloudantdeveloper image
      if (dest == 'ibmcom/cloudant-developer') {
        runInfo.envVars.add('CREDS_ID=cloudant-developer')
        runInfo.creds = [usernamePassword(credentialsId: 'cloudant-developer', usernameVariable: 'DB_USER', passwordVariable: 'DB_PASSWORD')]
      }
      runIn.dockerEnvWithSidecars(runInfo, [[imageName: dest]], testRun)
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
    ['library/couchdb:1.7.1', 'library/couchdb:latest'].each { c ->
      testAxes.put("Python${v}_${c}", {test_python("python:${v}", c)})
    }
  }
  parallel(testAxes)
}