import groovy.json.JsonBuilder
import groovy.json.JsonSlurper
import java.io.File
System.setProperty("hudson.model.DirectoryBrowserSupport.CSP", "")



pipeline {
    agent {
        kubernetes {
            defaultContainer 'jdk'
            yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
    - name: lighthouse-builder
      image: ndoppltfoundationarchacr11154.azurecr.io/ndop_lighthouse_builder:latest
      resources:
        requests:
          memory: "512Mi"
        limits:
          memory: "512Mi"
      imagePullPolicy: Always
      command:
        - sleep
      args:
        - infinity
      securityContext:
        runAsUser: 0
        privileged: true
    - name: jdk
      image: docker.io/eclipse-temurin:20.0.1_9-jdk
      command:
        - cat
      tty: true
    - name: lhci
      image: docker.io/patrickhulce/lhci-client:0.12.0
      command:
        - cat
      tty: true
      securityContext:
        runAsUser: 0
        privileged: true
      volumeMounts:
        - name: m2-cache
          mountPath: /root/.m2
    - name: podman
      image: quay.io/containers/podman:v4.5.1
      command:
        - cat
      tty: true
      securityContext:
        runAsUser: 0
        privileged: true
    - name: kubectl
      image: docker.io/bitnami/kubectl:1.27.3
      command:
        - cat
      tty: true
      securityContext:
        runAsUser: 0
        privileged: true
  volumes:
    - name: m2-cache
      hostPath:
        path: /data/m2-cache
        type: DirectoryOrCreate
  imagePullSecrets:
    - name: kiko
'''
        }
    }

    environment {
        APP_NAME = getPomArtifactId()
        APP_VERSION = getPomVersionNoQualifier()
        APP_CONTEXT_ROOT = '/' // it should be '/' or '<some-context>/'
        APP_LISTENING_PORT = '8080'
        APP_JACOCO_PORT = '6300'
        CONTAINER_REGISTRY_URL = 'docker.io'
        IMAGE_ORG = 'florenzobootcamp' // change it to your own organization at Docker.io!
        IMAGE_NAME = "$IMAGE_ORG/$APP_NAME"
        IMAGE_SNAPSHOT = "$IMAGE_NAME:$APP_VERSION-snapshot-$BUILD_NUMBER" // tag for snapshot version
        IMAGE_SNAPSHOT_LATEST = "$IMAGE_NAME:latest-snapshot" // tag for latest snapshot version
        IMAGE_GA = "$IMAGE_NAME:$APP_VERSION" // tag for GA version
        IMAGE_GA_LATEST = "$IMAGE_NAME:latest" // tag for latest GA version
        EPHTEST_CONTAINER_NAME = "ephtest-$APP_NAME-snapshot-$BUILD_NUMBER"
        EPHTEST_BASE_URL = "http://$EPHTEST_CONTAINER_NAME:$APP_LISTENING_PORT".concat("/$APP_CONTEXT_ROOT".replace('//', '/'))

        // credentials
        KUBERNETES_CLUSTER_CRED_ID = 'k3s-lima-vm-kubeconfig'
        CONTAINER_REGISTRY_CRED = credentials("docker-hub-$IMAGE_ORG")
        LIGHTHOUSE_TOKEN = credentials("ci-lighthouse-token-$APP_NAME")
        LIGHTHOUSE_URL = credentials('ci-lighthouse-url')
    }
    stages {
            stage('Prepare environment') {
                steps {
                    echo '-=- prepare environment -=-'
                    echo "APP_NAME: ${APP_NAME}\nAPP_VERSION: ${APP_VERSION}"
                    echo "the name for the epheremeral test container to be created is: $EPHTEST_CONTAINER_NAME"
                    echo "the base URL for the epheremeral test container is: $EPHTEST_BASE_URL"
                    sh 'java -version'
                    sh './mvnw --version'
                    container('podman') {
                        sh 'podman --version'
                        sh "podman login $CONTAINER_REGISTRY_URL -u $CONTAINER_REGISTRY_CRED_USR -p $CONTAINER_REGISTRY_CRED_PSW"
                    }
                    container('kubectl') {
                        withKubeConfig([credentialsId: "$KUBERNETES_CLUSTER_CRED_ID"]) {
                            sh 'kubectl version'
                        }
                    }
                    script {
                        qualityGates = readYaml file: 'quality-gates.yaml'
                    }
                }
            }

        stage('Compile') {
            steps {
                echo '-=- compiling project -=-'
                sh './mvnw compile'
            }
        }

        stage('Code inspection & quality gate') {
            steps {
                echo '-=- run code inspection & check quality gate -=-'
                withSonarQubeEnv('ci-sonarqube') {
                    sh './mvnw sonar:sonar'
                }
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Mutation tests') {
            steps {
                echo '-=- execute mutation tests -=-'
                sh './mvnw org.pitest:pitest-maven:mutationCoverage'
            }
        }

        stage('Generate BOM') {
            steps {
                sh './mvnw org.cyclonedx:cyclonedx-maven-plugin:makeBom'
            }
        }

            stage('Software composition analysis') {
                steps {
                    echo '-=- run software composition analysis -=-'
                    sh './mvnw dependency-check:check'
                    dependencyCheckPublisher(
                        failedTotalCritical: qualityGates.security.dependencies.critical.failed,
                        unstableTotalCritical: qualityGates.security.dependencies.critical.unstable,
                        failedTotalHigh: qualityGates.security.dependencies.high.failed,
                        unstableTotalHigh: qualityGates.security.dependencies.high.unstable,
                        failedTotalMedium: qualityGates.security.dependencies.medium.failed,
                        unstableTotalMedium: qualityGates.security.dependencies.medium.unstable)
                    script {
                        if (currentBuild.result == 'FAILURE') {
                            error('Dependency vulnerabilities exceed the configured threshold')
                        }
                    }
                }
          }

        stage('Package') {
            steps {
                echo '-=- packaging project -=-'
                sh './mvnw package -DskipTests'
                archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
            }
        }

        stage('Build & push container image') {
            steps {
                echo '-=- build & push container image -=-'
                container('podman') {
                    sh "podman build -t $IMAGE_SNAPSHOT ."
                    sh "podman tag $IMAGE_SNAPSHOT $CONTAINER_REGISTRY_URL/$IMAGE_SNAPSHOT"
                    sh "podman push $CONTAINER_REGISTRY_URL/$IMAGE_SNAPSHOT"
                    sh "podman tag $IMAGE_SNAPSHOT $CONTAINER_REGISTRY_URL/$IMAGE_SNAPSHOT_LATEST"
                    sh "podman push $CONTAINER_REGISTRY_URL/$IMAGE_SNAPSHOT_LATEST"
                }
            }
        }

        stage('Run container image') {
            steps {
                echo '-=- run container image -=-'
                container('kubectl') {
                    withKubeConfig([credentialsId: "$KUBERNETES_CLUSTER_CRED_ID"]) {
                        sh "kubectl run $EPHTEST_CONTAINER_NAME --image=$CONTAINER_REGISTRY_URL/$IMAGE_SNAPSHOT --env=JAVA_OPTS=-javaagent:/jacocoagent.jar=output=tcpserver,address=*,port=$APP_JACOCO_PORT --port=$APP_LISTENING_PORT"
                        sh "kubectl expose pod $EPHTEST_CONTAINER_NAME --port=$APP_LISTENING_PORT"
                        sh "kubectl expose pod $EPHTEST_CONTAINER_NAME --port=$APP_JACOCO_PORT --name=$EPHTEST_CONTAINER_NAME-jacoco"
                    }
                }
            }
        }

        stage('Integration tests') {
            steps {
                echo '-=- execute integration tests -=-'
                sh "curl --retry 10 --retry-connrefused --connect-timeout 5 --max-time 5 ${EPHTEST_BASE_URL}actuator/health"
                sh "./mvnw failsafe:integration-test failsafe:verify -Dtest.target.server.url=$EPHTEST_BASE_URL"
                sh "java -jar target/dependency/jacococli.jar dump --address $EPHTEST_CONTAINER_NAME-jacoco --port $APP_JACOCO_PORT --destfile target/jacoco-it.exec"
                sh 'mkdir target/site/jacoco-it'
                sh 'java -jar target/dependency/jacococli.jar report target/jacoco-it.exec --classfiles target/classes --xml target/site/jacoco-it/jacoco.xml'
                junit 'target/failsafe-reports/*.xml'
                jacoco execPattern: 'target/jacoco-it.exec'
            }
        }

        stage('Performance tests') {
            steps {
                echo '-=- execute performance tests -=-'
                sh "curl --retry 10 --retry-connrefused --connect-timeout 5 --max-time 5 ${EPHTEST_BASE_URL}actuator/health"
                sh "./mvnw jmeter:configure@configuration jmeter:jmeter jmeter:results -Djmeter.target.host=$EPHTEST_CONTAINER_NAME -Djmeter.target.port=$APP_LISTENING_PORT -Djmeter.target.root=$APP_CONTEXT_ROOT"
                perfReport sourceDataFiles: 'target/jmeter/results/*.csv'
            }
        }

        
        stage('test') {
            steps {
                container('lighthouse-builder') {
                    sh "lighthouse-ci ${EPHTEST_BASE_URL}hello" + " --jsonReport --report=. --filename=report.html"
                }
            }
        }
                     
        stage('Promote container image') {
            steps {
                echo '-=- promote container image -=-'
                container('podman') {
                    // when using latest or a non-snapshot tag to deploy GA version
                    // this tag push should trigger the change in staging/production environment
                    sh "podman tag $IMAGE_SNAPSHOT $CONTAINER_REGISTRY_URL/$IMAGE_GA"
                    sh "podman push $CONTAINER_REGISTRY_URL/$IMAGE_GA"
                    sh "podman tag $IMAGE_SNAPSHOT $CONTAINER_REGISTRY_URL/$IMAGE_GA_LATEST"
                    sh "podman push $CONTAINER_REGISTRY_URL/$IMAGE_GA_LATEST"
                }
            }
            post {
                always {
                    echo '-=- stop test container and remove deployment -=-'
                    container('kubectl') {
                        withKubeConfig([credentialsId: "$KUBERNETES_CLUSTER_CRED_ID"]) {
                            sh "kubectl delete pod $EPHTEST_CONTAINER_NAME"
                            sh "kubectl delete service $EPHTEST_CONTAINER_NAME"
                            sh "kubectl delete service $EPHTEST_CONTAINER_NAME-jacoco"
                        }
                    }
                }
            }
        }
}

def getPomVersion() {
    return readMavenPom().version
}

def getPomVersionNoQualifier() {
    return readMavenPom().version.split('-')[0]
}

def getPomArtifactId() {
    return readMavenPom().artifactId
}
