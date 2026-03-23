// CD Jenkinsfile for Repo B (Infrastructure-as-Code)
// What this pipeline does:
// 1) Checks out Repo B from GitHub
// 2) Accepts a manual input for the Docker image tag to deploy
// 3) Runs Ansible to provision/configure an EC2 instance and deploy the container
// 4) Verifies the application responds on port 80

pipeline {
  agent any

  // --- Manual "Push-Button" parameter ---
  parameters {
    string(name: 'IMAGE_TAG', defaultValue: 'v1.0.3', description: 'Docker image tag to deploy (e.g., v1.0.3)')
  }

  environment {
    // Docker image repository (must be lowercase)
    DOCKERHUB_IMAGE_REPO = 'fyooie11/devopsassignment2fastapi-dockerfile'

    // Where the Ansible playbook expects to find the SSH private key file (we create it at runtime)
    ANSIBLE_SSH_KEY_PATH = "${env.WORKSPACE}/.ssh_key.pem"

    // Use a virtual environment so pip/Ansible installs work without sudo and without PEP 668 errors.
    VENV_PATH = "${env.WORKSPACE}/.venv"

    // NOTE:
    // Update these AWS variables to match your environment, or pass them via Jenkins as extra-vars.
    AWS_REGION = 'us-east-1'
    INSTANCE_TYPE = 't2.micro'
    AMI_ID = 'ami-0b6c6ebed2801a5cb' // Ubuntu Noble (example; you must verify the correct AMI)
    SUBNET_ID = 'PUT_SUBNET_ID_HERE' // required for EC2 provisioning
    SECURITY_GROUP_ID = 'PUT_SECURITY_GROUP_ID_HERE' // must allow inbound 80 (and 22 if you use SSH)
    KEY_NAME = 'PUT_KEY_PAIR_NAME_HERE' // EC2 key pair name for the instance
    SSH_USER = 'ubuntu'

    // Deployment options
    CONTAINER_NAME = 'fastapi-app'
    HOST_PORT = '80'
    CONTAINER_PORT = '80'
  }

  stages {
    stage('Checkout Repo B') {
      steps {
        // Get the playbooks + Jenkinsfile from the repo itself
        checkout scm
      }
    }

    stage('Install Ansible dependencies') {
      steps {
        // Install Ansible and the AWS collection on the Jenkins agent (controller/agent machine).
        // If your Jenkins machine already has Ansible, this will be quick / cached.
        sh '''
          set -e
          # Create venv if it doesn't exist (avoids system-wide pip restrictions).
          if [ ! -d "$VENV_PATH" ]; then
            python3 -m venv "$VENV_PATH"
          fi

          "$VENV_PATH/bin/pip" install -q --upgrade pip
          "$VENV_PATH/bin/pip" install -q ansible boto3 botocore
          "$VENV_PATH/bin/ansible-galaxy" collection install -q amazon.aws community.general
        '''
      }
    }

    stage('Prepare SSH key for Ansible') {
      steps {
        // This expects you to create a Jenkins credential containing your EC2 SSH private key.
        // Create a credential of kind: "SSH Username with private key"
        // credentialsId must be: ec2-ssh-key
        withCredentials([
          sshUserPrivateKey(
            credentialsId: 'ec2-ssh-key',
            keyFileVariable: 'SSH_KEY_FILE',
            usernameVariable: 'SSH_USERNAME'
          )
        ]) {
          sh '''
            set -e
            # Copy the key to a predictable path for Ansible
            cp "$SSH_KEY_FILE" "$ANSIBLE_SSH_KEY_PATH"
            chmod 600 "$ANSIBLE_SSH_KEY_PATH"
          '''
        }
      }
    }

    stage('Provision EC2 + Deploy Docker image') {
      steps {
        // Docker Hub credentials for docker login
        // Expects you already created: dockerhub-credentials (username/password) or update the ID below.
        withCredentials([
          usernamePassword(
            // Use the credentialsId that exists in your Jenkins.
            // (Your current Docker Hub credential ID is a UUID.)
            credentialsId: 'd6898e21-9ec6-44cc-bef8-31166a0ffd5a',
            usernameVariable: 'DOCKERHUB_USER',
            passwordVariable: 'DOCKERHUB_PASS'
          )
        ]) {
          sh '''
            set -e

            IMAGE_FULL="${DOCKERHUB_IMAGE_REPO}:${IMAGE_TAG}"

            echo "Deploying image: $IMAGE_FULL"

            # Run the Ansible playbook locally; it will provision EC2 and then configure the new instance.
            # We pass sensitive runtime values via --extra-vars.
            "$VENV_PATH/bin/ansible-playbook" \\
              -i localhost, -c local \\
              playbooks/provision_and_deploy.yml \\
              --extra-vars "image_full=$IMAGE_FULL aws_region=$AWS_REGION instance_type=$INSTANCE_TYPE ami_id=$AMI_ID subnet_id=$SUBNET_ID security_group_id=$SECURITY_GROUP_ID key_name=$KEY_NAME ssh_user=$SSH_USER ansible_ssh_private_key_file=$ANSIBLE_SSH_KEY_PATH dockerhub_user=$DOCKERHUB_USER dockerhub_password=$DOCKERHUB_PASS container_name=$CONTAINER_NAME host_port=$HOST_PORT container_port=$CONTAINER_PORT"
          '''
        }
      }
    }
  }

  post {
    success {
      echo 'CD completed successfully.'
    }
    failure {
      echo 'CD failed. Check the Ansible output in the console log.'
    }
  }
}

