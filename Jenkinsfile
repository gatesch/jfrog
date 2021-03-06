def gitCommit() {
        sh "git rev-parse HEAD > GIT_COMMIT"
        def gitCommit = readFile('GIT_COMMIT').trim()
        sh "rm -f GIT_COMMIT"
        return gitCommit
    }

def answerQuestion = ''

    node {
        // Checkout source code from Git
        stage 'Checkout'
        checkout scm

        // PHPUnit test
        stage 'Unit Test'
        sh "phpunit --bootstrap src/Email.php tests"

        // Artifactory
        stage 'Artifactory'
        def server = Artifactory.server '101'
        sh "zip php-${gitCommit()}.zip *.php"
        def uploadSpec = """{
         "files": [
           {
             "pattern": "php-${gitCommit()}.zip",
             "target": "reports/"
           }
         ]
         }"""
         server.upload spec: uploadSpec
         server.upload spec: uploadSpec, failNoOp: true

//        sh "curl -u admin:redhat12 -X PUT http://api.tesch.loc/artifactory/reports/php-${gitCommit()}.zip ./php-${gitCommit()}.zip"

        // SonarQube
//        stage 'SCA / SonarQube'
//        sh "/opt/sonar/bin/sonar-scanner -Dsonar.projectKey=php -Dsonar.sources=. -Dsonar.host.url=http://sonar.tesch.loc -Dsonar.login=503a334e716877d70432d29aa4247b212123bc0a"
        

        // Build Docker image
        stage 'Build'
        sh "docker build -t quay.io/gatesch/php:${gitCommit()} ."

        // Login to DTR 
        stage 'Login'
        withCredentials(
            [[
                $class: 'UsernamePasswordMultiBinding',
                credentialsId: 'dtr',
                passwordVariable: 'DTR_PASSWORD',
                usernameVariable: 'DTR_USERNAME'
            ]]
        ){ 
        sh "docker login -u gatesch -p ${env.DTR_PASSWORD}  quay.io"}

        // Push the image 
        stage 'Push'
        sh "docker push quay.io/gatesch/php:${gitCommit()}"

//        clean all
          stage('Deploy test') 

          script {
          answerQ = sh (returnStdout: true, script: "kubectl get ns |grep test |awk '{print \$1}'")
          }

          if ( answerQ != "" ) {
                sh "kubectl set image deployment php-safe -n test php=quay.io/gatesch/php:${gitCommit()}"
           }

           else {
                sh "kubectl create ns test"
        	sh "kubectl create -f limits.yaml"
        	sh "kubectl create deployment php-safe -n test --image=quay.io/gatesch/php:${gitCommit()}"
        	sh "kubectl expose deployment php-safe --port=80 --name=php-service -n test"
        	sh "kubectl create -f php-ingress.yaml"
           }

        // functional test
        stage 'Selenium'
        sh "./selenium-test.py"
    }

        stage('Deploy approval'){
             input "Deploy to prod?"
        }
	
	node {
        stage('Deploy prod')

              script {
                     answerP = sh (returnStdout: true, script: "kubectl get ns |grep prod |awk '{print \$1}'")
          }

          if ( answerP != "" ) {
                sh "kubectl set image deployment php-safe -n prod php=quay.io/gatesch/php:${gitCommit()}"
           }

           else {
                sh "kubectl create ns prod"
                sh "kubectl create -f limits-prod.yaml"
                sh "kubectl create deployment php-safe -n prod --image=quay.io/gatesch/php:${gitCommit()}"
                sh "kubectl expose deployment php-safe --port=80 --name=php-service -n prod"
                sh "kubectl create -f php-prod-ingress.yaml"
                sh "kubectl autoscale deployment php-safe --cpu-percent=50 --min=1 --max=10 -n prod"
           }
	}

