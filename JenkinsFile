pipeline {
    agent any
    
    stages {
        stage('Jenkins Backup') {
            steps {
                sh '''
                    cd /var/lib
                    tar -czvf jenkinsBackup.tar.gz jenkins
                '''
            }
        }

        stage('Upload into S3 Bucket') {
            steps {
                script {
                    // Replace <bucket-name> with your S3 bucket name and set up AWS CLI
                    sh '''
                        s3Upload consoleLogLevel: 'INFO', dontSetBuildResultOnFailure: false, dontWaitForConcurrentBuildCompletion: false, entries: [[bucket: 'jenkins-backup-0 ', excludedFile: '', flatten: false, gzipFiles: false, keepForever: false, managedArtifacts: false, noUploadOnFailure: false, selectedRegion: 'us-east-1', showDirectlyInBrowser: false, sourceFile: '/var/lib/jenkinsBackup.tar.gz', storageClass: 'STANDARD', uploadFromSlave: false, useServerSideEncryption: false]], pluginFailureResultConstraint: 'FAILURE', profileName: 's3backup', userMetadata: []
                        #aws s3 cp /var/lib/jenkinsBackup.tar.gz s3://<bucket-name>/jenkinsBackup.tar.gz
                    '''
                }
            }
        }
    }
}
