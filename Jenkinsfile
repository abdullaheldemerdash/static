pipeline {
      agent any
      stages {
          stage('Upload to AWS.') {
              steps {
               withAWS(credentials: 'aws-static', region: 'us-west-2') {
    // do something
                  s3Upload(file:'index.html', bucket:'aeldemerdash-udacity', path:'index.html')
                  sh 'echo "Hello World"'
                  sh '''
                      echo "Multiline shell steps works too"
                      ls -lah
                      '''
                     }
                    }
                  }
                }
              }
