pipeline {
	options {
		timeout(time: 2, unit: 'HOURS')
		buildDiscarder(logRotator(numToKeepStr:'10'))
		disableConcurrentBuilds(abortPrevious: true)
	}
  agent {
    kubernetes {
      label 'wildwebdeveloper-buildtest-pod'
      defaultContainer 'jnlp'
      yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: container
    image: docker.io/akurtakov/fedora-gtk3-mutter-java-node:java-21-node-20
    imagePullPolicy: Always
    tty: true
    command: [ "uid_entrypoint", "cat" ]
    resources:
      limits:
        memory: "4Gi"
        cpu: "2000m"
      requests:
        memory: "4Gi"
        cpu: "1000m"
  - name: jnlp
    image: 'eclipsecbi/jenkins-jnlp-agent'
    volumeMounts:
    - mountPath: /home/jenkins/.ssh
      name: volume-known-hosts
  volumes:
  - configMap:
      name: known-hosts
    name: volume-known-hosts
"""
    }
  }
	environment {
		NPM_CONFIG_USERCONFIG = "$WORKSPACE/.npmrc"
		MAVEN_OPTS="-Xmx1024m"
		GITHUB_API_CREDENTIALS_ID = 'github-bot-token'
	}
	stages {
		stage('Prepare-environment') {
			steps {
				container('container') {
					sh 'java -version'
					sh 'mvn --version'
					sh 'node --version'
					sh 'npm --version'
					sh 'npm config set cache="$WORKSPACE/npm-cache"'
				}
			}
		}
		stage('Build') {
			steps {
				container('container') {
					withCredentials([string(credentialsId: "${GITHUB_API_CREDENTIALS_ID}", variable: 'GITHUB_API_TOKEN')]) {
						wrap([$class: 'Xvnc', useXauthority: true]) {
							sh """mvn clean verify -B -fae -Dtycho.disableP2Mirrors=true -Ddownload.cache.skip=true -Dmaven.test.error.ignore=true -Dmaven.test.failure.ignore=true ${env.BRANCH_NAME=='master' ? '-Psign': ''} -Dmaven.repo.local=$WORKSPACE/.m2/repository -Dgithub.api.token="${GITHUB_API_TOKEN}" """
						}
					}
				}
			}
			post {
				always {
					junit '*/target/surefire-reports/TEST-*.xml'
					archiveArtifacts artifacts: 'repository/target/repository/**,*/target/work/configuration/*.log,*/target/work/data/.metadata/.log,*/target/work/data/languageServers-log/**,org.eclipse.wildwebdeveloper/target/chrome-debug-adapter/chromeDebugAdapter.zip'
				}
			}
		}
		stage('Deploy') {
			when {
				branch 'master'
			}
			steps {
				sshagent ( ['projects-storage.eclipse.org-bot-ssh']) {
					sh '''
						ssh genie.wildwebdeveloper@projects-storage.eclipse.org rm -rf /home/data/httpd/download.eclipse.org/wildwebdeveloper/snapshots
						ssh genie.wildwebdeveloper@projects-storage.eclipse.org mkdir -p /home/data/httpd/download.eclipse.org/wildwebdeveloper/snapshots
						scp -r repository/target/repository/* genie.wildwebdeveloper@projects-storage.eclipse.org:/home/data/httpd/download.eclipse.org/wildwebdeveloper/snapshots
						ssh genie.wildwebdeveloper@projects-storage.eclipse.org mkdir -p /home/data/httpd/download.eclipse.org/wildwebdeveloper/snapshots/IDEs
						scp -r repository/target/products/*.zip genie.wildwebdeveloper@projects-storage.eclipse.org:/home/data/httpd/download.eclipse.org/wildwebdeveloper/snapshots/IDEs
						scp -r repository/target/products/*.tar.gz genie.wildwebdeveloper@projects-storage.eclipse.org:/home/data/httpd/download.eclipse.org/wildwebdeveloper/snapshots/IDEs
					'''
				}
			}
		}
	}
}
