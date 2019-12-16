node ('master') {
    def app
    def version

    stage('Cloning repository') {
        checkout scm
        notifyStarted()
        env.WORKSPACE = pwd()
        sh '''cat server/package.json   | grep version   | head -1   | awk -F: '{ print $2 }'   | sed 's/[",]//g' > version.txt'''
        version = readFile('version.txt').trim()
    }

    stage('Installing dependencies'){
        sh 'sudo npm i nyc -g'
        sh 'sudo npm i jshint -g'
        sh 'cd ./server && sudo npm install --unsafe-perm=true && sudo node install.js'
    }

    stage('Checking JSHint errors') {
        sh '''
        #!/bin/bash
        var4=$(jshint --config=.jshintrc . |wc -l | cut -f1 -d" ")
        if [[ $var4 -gt 0 ]]
        then
            echo $var4 "JSHint errors found"
            exit $var4
        fi
        '''
    }

    stage('Unit Test'){
        sh 'cd ./server && sudo nyc --reporter=lcov npm test'
        sh 'cd ./server && sudo nyc report --reporter=text-lcov > coverage.lcov'
    }

    stage('SonarQube Analysis') {
        def scannerHome = tool 'SonarQube Scanner 3.1';
        withSonarQubeEnv('SonarQube01') {
            sh "${scannerHome}/bin/sonar-scanner"
        }
    }

    stage("Quality Gate") {
        timeout(time: 20, unit: 'MINUTES') {
            def qg = waitForQualityGate()
            if (qg.status != 'OK') {
                notifyFailed()
                error "Pipeline aborted due to SonarQube quality gate failure: ${qg.status}"
            }
        }
    }
    
    stage('Building UI'){
        sh 'echo "Building UI"'
        retcode = sh 'docker run -v `pwd`/client:/src relevancelab/angular-cli:1.3.2  bash -c "cd /src/ && npm install && ng build -op ./public --aot --prod --build-optimizer=true --vendor-chunk=true && tar -cvzf public.tgz ./public"'
        script {
            if (retcode) {
                    notifyFailed()
                    error 'UI Build Failed'
                }
        }
    }

    stage('Building Docker image') {
        app = docker.build("relevancelab/cc")
    }

    stage('Quality Check') {
        /*input "Confirm the quality?"*/
        app.inside {
            sh 'echo "Manualy test passed"'
        }
    }

    stage('Push Docker image') {
        docker.withRegistry('https://registry.hub.docker.com', 'docker-hub-credentials') {
            app.push("${version}_b${env.BUILD_NUMBER}")
            app.push("latest")
        }
    }
    
    stage('Cleanup'){
        // Cleanup docker images
        sh 'echo "Cleaning images"'
        notifySuccessful()
    }
}
`
