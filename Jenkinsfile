#!groovy

import groovy.json.JsonOutput

def buildSuccess = false
def rosContainer
try {
  node('android') {
    timeout(time: 90, unit: 'MINUTES') {
      // Allocate a custom workspace to avoid having % in the path (it breaks ld)
      ws('/tmp/realm-java') {
        stage('SCM') {
          checkout([
                 $class: 'GitSCM',
                branches: scm.branches,
                gitTool: 'native git',
                extensions: scm.extensions + [
                  [$class: 'CleanCheckout'],
                  [$class: 'SubmoduleOption', recursiveSubmodules: true]
                ],
                userRemoteConfigs: scm.userRemoteConfigs
               ])
        }

        // Toggles for PR vs. Master builds.
        // For PR's, we just build for arm-v7a and run unit tests for the ObjectServer variant
        // A full build is done on `master`.
        // TODO Once Android emulators are available on all nodes, we can switch to x86 builds
        // on PR's for even more throughput.
        def ABIs = ""
        def instrumentationTestTarget = "connectedAndroidTest"

        def buildEnv
        def rosEnv
        stage('Docker build') {
          // Docker image for build
          buildEnv = docker.build 'realm-java:snapshot'
          // Docker image for testing Realm Object Server
          def dependProperties = readProperties file: 'dependencies.list'
          def rosDeVersion = dependProperties["REALM_OBJECT_SERVER_DE_VERSION"]
          rosEnv = docker.build 'ros:snapshot', "--build-arg ROS_DE_VERSION=${rosDeVersion} tools/sync_test_server"
        }

	    rosContainer = rosEnv.run()

        try {
              buildEnv.inside("-e HOME=/tmp " +
                  "-e _JAVA_OPTIONS=-Duser.home=/tmp " +
                  "--privileged " +
                  "-v /dev/bus/usb:/dev/bus/usb " +
                  "-v ${env.HOME}/gradle-cache:/tmp/.gradle " +
                  "-v ${env.HOME}/.android:/tmp/.android " +
                  "-v ${env.HOME}/ccache:/tmp/.ccache " +
                  "-e REALM_CORE_DOWNLOAD_DIR=/tmp/.gradle " +
                  "--network container:${rosContainer.id}") {

                // TODO: add support for running monkey on the example apps
                stage('Build') {
                  withCredentials([[$class: 'FileBinding', credentialsId: 'c0cc8f9e-c3f1-4e22-b22f-6568392e26ae', variable: 'S3CFG']]) {
                    sh "cd realm && chmod +x gradlew && ./gradlew clean assemble -Ps3cfg=${env.S3CFG} -PbuildTargetABIs=${ABIs}"
                  }
                }

                stage('Collect metrics') {
                  collectAarMetrics()
                }

                if (['master', 'next-major'].contains(env.BRANCH_NAME)) {
                  stage('Publish to OJO') {
                    withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: 'bintray', passwordVariable: 'BINTRAY_KEY', usernameVariable: 'BINTRAY_USER']]) {
                      sh "chmod +x gradlew && ./gradlew -PbintrayUser=${env.BINTRAY_USER} -PbintrayKey=${env.BINTRAY_KEY} assemble ojoUpload --stacktrace"
                    }
                  }
                }
              }
        } finally {
              archiveRosLog(rosContainer.id)
              sh "docker logs ${rosContainer.id}"
              rosContainer.stop()
        }
      }
    }
    currentBuild.rawBuild.setResult(Result.SUCCESS)
    buildSuccess = true
  }
} catch(Exception e) {
  currentBuild.rawBuild.setResult(Result.FAILURE)
  buildSuccess = false
  throw e
} finally {
  if (['master', 'releases', 'next-major'].contains(env.BRANCH_NAME) && !buildSuccess) {
    node {
      withCredentials([[$class: 'StringBinding', credentialsId: 'slack-java-url', variable: 'SLACK_URL']]) {
        def payload = JsonOutput.toJson([
	    username: 'Mr. Jenkins',
	    icon_emoji: ':jenkins:',
	    attachments: [[
	        'title': "The ${env.BRANCH_NAME} branch is broken!",
		'text': "<${env.BUILD_URL}|Click here> to check the build.",
		'color': "danger"
	    ]]
	])
        sh "curl -X POST --data-urlencode \'payload=${payload}\' ${env.SLACK_URL}"
      }
    }
  }
}

def forwardAdbPorts() {
  sh ''' adb reverse tcp:9080 tcp:9080 && adb reverse tcp:9443 tcp:9443 &&
      adb reverse tcp:8888 tcp:8888
  '''
}

def String startLogCatCollector() {
  sh '''adb logcat -c
  adb logcat -v time > "logcat.txt" &
  echo $! > pid
  '''
  return readFile("pid").trim()
}

def stopLogCatCollector(String backgroundPid) {
  sh "kill ${backgroundPid}"
  zip([
    'zipFile': 'logcat.zip',
    'archive': true,
    'glob' : 'logcat.txt'
  ])
  sh 'rm logcat.txt'
}

def archiveRosLog(String id) {
  sh "docker cp ${id}:/tmp/ros-testing-server.log ./ros.log"
  zip([
      'zipFile': 'roslog.zip',
      'archive': true,
      'glob' : 'ros.log'
  ])
  sh 'rm ros.log'
}

def sendMetrics(String metricName, String metricValue, Map<String, String> tags) {
  def tagsString = getTagsString(tags)
  withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: '5b8ad2d9-61a4-43b5-b4df-b8ff6b1f16fa', passwordVariable: 'influx_pass', usernameVariable: 'influx_user']]) {
    sh "echo 'https://greatscott-pinheads-70.c.influxdb.com:8086/write?db=realm' --data-binary '${metricName},${tagsString} value=${metricValue}i' --user 'XXX:XXX'"
  }
}

@NonCPS
def getTagsString(Map<String, String> tags) {
  return tags.collect { k,v -> "$k=$v" }.join(',')
}

def storeJunitResults(String path) {
  step([
	 $class: 'JUnitResultArchiver',
     allowEmptyResults: true,
     testResults: path
   ])
}

def collectAarMetrics() {
  def flavors = ['base', 'objectServer']
  for (def i = 0; i < flavors.size(); i++) {
    def flavor = flavors[i]
    sh """set -xe
      cd realm/realm-library/build/outputs/aar
      unzip -l realm-android-library-${flavor}-release.aar
      unzip realm-android-library-${flavor}-release.aar -d unzipped${flavor}
      ls -ll unzipped${flavor}
      find \$ANDROID_HOME -name dx | sort -r | head -n 1 > dx
      \$(cat dx) --dex --output=temp${flavor}.dex unzipped${flavor}/classes.jar
      cat temp${flavor}.dex | head -c 92 | tail -c 4 | hexdump -e '1/4 \"%d\"' > methods${flavor}
    """

    def methods = readFile("realm/realm-library/build/outputs/aar/methods${flavor}")
    sh "echo Methods: ${methods}, flavor: ${flavor}"
    sendMetrics('methods', methods, ['flavor':flavor])

    def aarFile = findFiles(glob: "realm/realm-library/build/outputs/aar/realm-android-library-${flavor}-release.aar")[0]
    sh "echo AarSize: ${aarFile.length as String}, flavor: ${flavor}"
    sendMetrics('aar_size', aarFile.length as String, ['flavor':flavor])

    def soFiles = findFiles(glob: "realm/realm-library/build/outputs/aar/unzipped${flavor}/**/librealm-jni.so")
    sh "echo files: ${soFiles.size()}"
    for (def j = 0; j < soFiles.size(); j++) {
      def soFile = soFiles[j]
      def abiName = soFile.path.tokenize('/')[-2]
      def libSize = soFile.length as String
      sh "echo soFile: ${soFile}, abiName: ${abiName}, size: ${libSize}"
      sendMetrics('abi_size', libSize, ['flavor':flavor, 'type':abiName])
    }
  }
}

def gradle(String commands) {
  sh "chmod +x gradlew && ./gradlew ${commands} --stacktrace"
}

def gradle(String relativePath, String commands) {
  sh "cd ${relativePath} && chmod +x gradlew && ./gradlew ${commands} --stacktrace"
}
