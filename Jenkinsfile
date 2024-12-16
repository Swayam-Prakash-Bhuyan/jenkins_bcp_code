pipeline {
    agent any

    stages {
        stage('Create Backup') {
            steps {
                script {
                    // Define the backup file path
                    def backupFile = "${WORKSPACE}/jenkinsBackup.tar.gz"
                    
                    // Change directory to /var/lib/jenkins and create a backup tar file
                    def tarResult = sh(script: "tar --ignore-failed-read -czvf ${backupFile} -C /var/lib jenkins", returnStatus: true)
                    
                    // Check if tar command was successful (status code 0 means success)
                    if (tarResult != 0) {
                        error "Tar command failed. Aborting the pipeline."
                    }
                }
            }
        }

        stage('Upload Backup to S3') {
            steps {
                script {
                    // Define the backup file path in the workspace
                    def backupFile = "${WORKSPACE}/jenkinsBackup.tar.gz"
                    
                    // Upload the backup to S3 only if the tar command was successful
                    withAWS(credentials: 'jenkins-backup-user', region: 'us-east-1') {
                        s3Upload(
                            acl: 'Private', 
                            bucket: 'jenkins-backup-0', 
                            file: backupFile, 
                            path: 'backup/'
                        )
                    }
                }
            }
        }
    }
}
