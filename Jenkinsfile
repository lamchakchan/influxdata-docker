pipeline {
  agent any
  options { skipDefaultCheckout() }

  environment {
    DOCKER_MAINTAINER = credentials("INFLUXDATA_DOCKER_MAINTAINER")
  }

  stages {
    stage('Setup project workspace') {
      steps {
        dir('influxdata-docker') {
          checkout scm
        }
      }
    }

    stage('Build chronograf') {
      steps {
        dir('influxdata-docker') {
          sh './circle-test.sh chronograf'
        }
      }
    }

    stage('Build influxdb') {
      steps {
        dir('influxdata-docker') {
          sh './circle-test.sh influxdb'
        }
      }
    }

    stage('Build kapacitor') {
      steps {
        dir('influxdata-docker') {
          sh './circle-test.sh kapacitor'
        }
      }
    }

    stage('Build telegraf') {
      steps {
        dir('influxdata-docker') {
          sh './circle-test.sh telegraf'
        }
      }
    }

    stage('Update official images') {
      steps {
        dir('official-images') {
          checkout(
            scm: [
              $class: 'GitSCM',
              branches: [[name: '*/master' ]],
              userRemoteConfigs: [[
                credentialsId: 'jenkins-hercules-ssh',
                url: 'git@github.com:influxdata/official-images.git',
              ]],
            ],
            poll: false,
          )

          // Reset master to the value at origin.
          sh "git checkout master && git reset --hard origin/master"
        }

        withDockerContainer(image: "golang:1.9.1-stretch") {
          sh 'cd influxdata-docker; go run update.go -n'
        }
      }
    }

    stage('Push changes to GitHub') {
      when {
        expression {
          return sh(returnStatus: true, script: 'cd official-images; git diff --quiet') != 0
        }
      }

      steps {
        sh "docker pull jsternberg/hub"
        dir('official-images') {
          sshagent(credentials: ['jenkins-hercules-ssh']) {
            sh """
              git -c user.name="Jonathan A. Sternberg" -c user.email="jonathan@influxdb.com" commit -am "Update influxdata images"
              git push origin master
            """
          }

          sh """
            if ! git remote | grep upstream; then
              git remote add upstream git://github.com/docker-library/official-images.git
            else
              git remote set-url upstream git://github.com/docker-library/official-images.git
            fi
          """

          withEnv(["GITHUB_USER=${DOCKER_MAINTAINER_USR}", "GITHUB_TOKEN=${DOCKER_MAINTAINER_PSW}"]) {
            withDockerContainer(image: "jsternberg/hub") {
              sh """
                if ! hub pr show &> /dev/null; then
                  hub pull-request -m "Update influxdata images"
                fi
              """
            }
          }
        }
      }
    }
  }
}
