pipeline {
agent any
environment {
TF_WORKDIR = 'terraform'
AWS_DEFAULT_REGION = "${params.AWS_REGION ?: 'us-east-1'}"
SSH_KEY = credentials('jenkins-key')
SSH_USER = 'ec2-user' // or ubuntu depending on AMI
TF_VAR_alert_email = credentials('alert-email')
}
triggers {
// GitHub webhook triggers pipeline automatically
}
stage('Checkout') {
    steps {
        checkout([$class: 'GitSCM',
            branches: [[name: '*/main']],
            userRemoteConfigs: [[url: 'https://github.com/abhigiri07/static-website-project.git']]
        ])
    }
}


stage('Init Terraform') {
steps {
dir(TF_WORKDIR) {
sh 'terraform init -input=false'
sh 'terraform validate || true'
}
}
}
stage('Plan and Apply') {
steps {
dir(TF_WORKDIR) {
sh 'terraform plan -out=tfplan -input=false'
sh 'terraform apply -input=false tfplan'
}
}
}


stage('Get ALB DNS & Validate') {
steps {
dir(TF_WORKDIR) {
script {
alb = sh(script: "terraform output -raw alb_dns_name", returnStdout: true).trim()
echo "ALB: ${alb}"
sh "curl -Ssf http://${alb}/ || true"
}
}
}
}


stage('Trigger CPU Load Test') {
steps {
script {
// Find one instance from ASG, then ssh and run stress-ng
sh '''
INSTANCE_ID=$(aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names static-web-asg --query "AutoScalingGroups[0].Instances[0].InstanceId" --output text)
IP=$(aws ec2 describe-instances --instance-ids $INSTANCE_ID --query "Reservations[0].Instances[0].PublicIpAddress" --output text)
echo "Instance: $INSTANCE_ID -> $IP"
if [ "$IP" != "None" ]; then
ssh -o StrictHostKeyChecking=no -i ${SSH_KEY} ${SSH_USER}@${IP} "sudo stress-ng --cpu 2 --timeout 180s" || true
else
echo 'No public IP available. Consider using SSM or run stress from another jump host.'
fi
'''
}
}
}

stage('Validate Scaling and Alarm') {
steps {
echo 'Check ASG desired/actual count and CloudWatch for alarm trigger. SNS will send email to subscribed address.'
sh '''
aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names static-web-asg --query 'AutoScalingGroups[0].Instances' --output table
aws cloudwatch describe-alarms --alarm-names high-cpu-alarm || true
'''
}
}
}
}
