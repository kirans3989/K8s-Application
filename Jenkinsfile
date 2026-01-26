pipeline {
    agent any

    environment {
        AWS_REGION = "us-east-1"
        EKS_CLUSTER = "demo"
        KUBECONFIG = "${WORKSPACE}/kubeconfig"
    }

    stages {

        stage('Update kubeconfig') {
            steps {
                sh '''
                aws eks update-kubeconfig \
                  --region ${AWS_REGION} \
                  --name ${EKS_CLUSTER} \
                  --kubeconfig ${KUBECONFIG}
                '''
            }
        }

        stage('Install EBS CSI Driver') {
            steps {
                sh '''
                eksctl utils associate-iam-oidc-provider \
                  --cluster ${EKS_CLUSTER} \
                  --approve

                eksctl create iamserviceaccount \
                  --name ebs-csi-controller-sa \
                  --namespace kube-system \
                  --cluster ${EKS_CLUSTER} \
                  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
                  --approve

                eksctl create addon \
                  --name aws-ebs-csi-driver \
                  --cluster ${EKS_CLUSTER}
                '''
            }
        }

        stage('Verify EBS CSI Pods') {
            steps {
                sh '''
                kubectl get pods -n kube-system | grep ebs
                '''
            }
        }

        stage('Create Namespace') {
            steps {
                sh '''
                kubectl create namespace cv-app-prod || true
                '''
            }
        }

        stage('Deploy ConfigMap & Secret') {
            steps {
                sh '''
                kubectl apply -f k8s/configmap.yaml
                kubectl apply -f k8s/secret.yaml
                '''
            }
        }

        stage('Deploy Stateless App') {
            steps {
                sh '''
                kubectl apply -f k8s/deployment.yaml
                kubectl get pods -n cv-app-prod
                '''
            }
        }

        stage('Deploy StorageClass & MySQL') {
            steps {
                sh '''
                kubectl apply -f k8s/sc.yaml
                kubectl apply -f k8s/mysql.yaml
                kubectl get pvc -n cv-app-prod
                '''
            }
        }

        stage('Wait for MySQL Pod') {
            steps {
                sh '''
                kubectl wait \
                  --for=condition=Ready pod/mysql-0 \
                  -n cv-app-prod \
                  --timeout=300s
                '''
            }
        }

        stage('Initialize MySQL Data') {
            steps {
                sh '''
                kubectl exec mysql-0 -n cv-app-prod -- bash -c "
                mysql -uroot -pcvpassword123 <<EOF
                CREATE DATABASE IF NOT EXISTS cvdb;
                USE cvdb;
                CREATE TABLE IF NOT EXISTS demo (id INT, name VARCHAR(50));
                INSERT INTO demo VALUES (1,'commvault');
                INSERT INTO demo VALUES (2,'eks-backup-test');
                SELECT * FROM demo;
                EOF
                "
                '''
            }
        }
    }

    post {
        success {
            echo "EKS app + MySQL deployment completed successfully"
        }
        failure {
            echo "Pipeline failed. Check logs."
        }
    }
}

