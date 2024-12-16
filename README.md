# Jenkins Backup and Restore Pipeline

This repository provides a set of scripts and configurations to automate the backup and restore process for Jenkins using AWS S3. The pipeline leverages the **Pipeline AWS Steps Plugin** for integration with AWS S3 and requires the appropriate IAM user privileges for both AWS and GCP (in case of cloud-specific operations). The backup files are created, stored in S3, and restored to the Jenkins system as needed.

## Prerequisites

Before running the pipeline, ensure you have the following:

### Jenkins Configuration:
- Jenkins should be configured with the **Pipeline: AWS Steps Plugin** for integration with AWS S3.
- IAM user privileges for AWS should be configured with appropriate permissions (e.g., `s3:Upload`, `s3:Download`, etc.).
- For Google Cloud Platform (GCP) operations, ensure a **Service Account** is configured with the necessary privileges to interact with GCP services.
- The `jenkins` user should have necessary permissions to access the `/var/lib/jenkins` directory and execute commands like `tar`, `chmod`, `chown`, etc.

### Backup Storage:
- **AWS S3** bucket setup: A bucket named `jenkins-backup-0` to store Jenkins backup files.
- Ensure the bucket is properly configured with the correct access control list (ACL), so Jenkins can upload and download backup files.

### Command-Line Tools:
- `aws-cli` for interacting with AWS S3.
- `tar` for compressing and extracting files.
- `systemctl` for restarting Jenkins after the restore process.

## Pipeline Overview

This pipeline automates the following tasks:
1. **Backup Jenkins**: Create a backup of the Jenkins data directory (`/var/lib/jenkins`) and store it in an AWS S3 bucket.
2. **Restore Jenkins**: Download the backup from S3 and restore it on the Jenkins server.

## Pipeline Steps

### 1. **Backup Jenkins Data**:

The pipeline creates a backup of the `/var/lib/jenkins` directory and uploads it to an AWS S3 bucket.

#### Key Commands:
```bash
# Change ownership of Jenkins directory (ensure proper permissions)
sudo chown -R jenkins:jenkins /var/lib/jenkins

# Set appropriate permissions for the Jenkins directory
sudo chmod -R 755 /var/lib/jenkins

# Create a tarball of the Jenkins directory and backup it to S3
tar --ignore-failed-read -czvf jenkinsBackup.tar.gz -C /var/lib jenkins
aws s3 cp jenkinsBackup.tar.gz s3://jenkins-backup-0/backup/
```

### 2. **Restore Jenkins Data**:

To restore the backup, the pipeline downloads the backup file from the S3 bucket, extracts it, and then restarts Jenkins.

#### Key Commands:
```bash
# Download the backup tar file from S3
cd /var/lib/
aws s3 cp s3://jenkins-backup-0/backup/jenkinsBackup.tar.gz ./

# Extract the tar file
tar -xzvf jenkinsBackup.tar.gz -C ./

# Restart Jenkins to apply the changes
sudo systemctl restart jenkins
```

### IAM User Permissions (AWS):
The IAM user associated with Jenkins needs the following permissions for AWS S3:
- `s3:PutObject` for uploading backup files to S3.
- `s3:GetObject` for downloading backup files from S3.
- `s3:ListBucket` to list objects in the S3 bucket.

Here is an example IAM policy for the Jenkins backup user:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:PutObject",
                "s3:GetObject",
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::jenkins-backup-0",
                "arn:aws:s3:::jenkins-backup-0/*"
            ]
        }
    ]
}
```

### GCP Service Account Permissions (for GCP operations):
If you are using Google Cloud Platform for backup, the service account needs similar permissions to interact with GCP resources like Cloud Storage, similar to the AWS IAM policy above.

## Example Jenkins Pipeline Script

Here’s an example of how to integrate this process into a Jenkins pipeline:

```groovy
pipeline {
    agent any

    stages {
        stage('Create Backup') {
            steps {
                script {
                    // Ensure correct ownership and permissions for Jenkins directory
                    sh 'sudo chown -R jenkins:jenkins /var/lib/jenkins'
                    sh 'sudo chmod -R 755 /var/lib/jenkins'

                    // Define the backup file path
                    def backupFile = "${WORKSPACE}/jenkinsBackup.tar.gz"
                    
                    // Create the backup tar file and upload it to S3
                    def tarResult = sh(script: "sudo tar --ignore-failed-read -czvf ${backupFile} -C /var/lib jenkins", returnStatus: true)
                    if (tarResult != 0) {
                        error "Tar command failed. Aborting the pipeline."
                    }
                    
                    // Upload the backup to S3
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

        stage('Restore Backup') {
            steps {
                script {
                    // Download and extract the backup from S3
                    sh 'cd /var/lib/ && aws s3 cp s3://jenkins-backup-0/backup/jenkinsBackup.tar.gz ./'
                    sh 'tar -xzvf /var/lib/jenkinsBackup.tar.gz -C /var/lib/'

                    // Restart Jenkins to apply the restore
                    sh 'sudo systemctl restart jenkins'
                }
            }
        }
    }
}
```

## Troubleshooting

- **Permission Issues**: If the `aws-cli` commands fail due to permission issues, ensure that the Jenkins instance has the correct AWS credentials and IAM permissions for S3 access.
- **File Ownership**: After restoring the backup, double-check that the file ownerships are set correctly, especially if Jenkins fails to start or behaves unexpectedly.
- **Backup Integrity**: If the backup file is not uploading or downloading correctly, verify that the S3 bucket has the correct ACL and that the Jenkins user has write permissions.

## Conclusion

This repository automates the backup and restore process for Jenkins using AWS S3. By following the steps and ensuring proper permissions, you can effectively manage your Jenkins backups in a secure and efficient way.
