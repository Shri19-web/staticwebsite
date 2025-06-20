# staticwebsite
pipeline {
    agent any
    
    parameters {
        string(name: 'S3_BUCKET', defaultValue: 'your-static-website-bucket', description: 'S3 bucket name for deployment')
        string(name: 'AWS_REGION', defaultValue: 'us-east-1', description: 'AWS region for S3 bucket')
        booleanParam(name: 'ENABLE_CLOUDFRONT_INVALIDATION', defaultValue: false, description: 'Enable CloudFront cache invalidation')
        string(name: 'CLOUDFRONT_DISTRIBUTION_ID', defaultValue: '', description: 'CloudFront distribution ID (if invalidation enabled)')
    }
    
    environment {
        AWS_DEFAULT_REGION = "${params.AWS_REGION}"
        AWS_CREDENTIALS_ID = 'aws-credentials' // Configure this in Jenkins credentials
    }
    
    stages {
        stage('Checkout') {
            steps {
                echo 'Checking out source code...'
                checkout scm
            }
        }
        
        stage('Validate Files') {
            steps {
                script {
                    echo 'Validating HTML files...'
                    sh '''
                        # Check if index.html exists
                        if [ ! -f "index.html" ]; then
                            echo "Error: index.html not found!"
                            exit 1
                        fi
                        
                        # Basic HTML validation
                        if ! grep -q "<!DOCTYPE html>" index.html; then
                            echo "Warning: HTML5 DOCTYPE not found"
                        fi
                        
                        echo "File validation completed successfully"
                    '''
                }
            }
        }
        
        stage('Build') {
            steps {
                echo 'Building static website...'
                script {
                    // For a static site, we might want to minify CSS/JS or optimize images
                    sh '''
                        # Create build directory
                        mkdir -p dist
                        
                        # Copy all files to dist directory
                        cp -r * dist/ 2>/dev/null || true
                        
                        # Remove build artifacts from dist
                        rm -rf dist/dist dist/Jenkinsfile dist/.git* dist/README.md 2>/dev/null || true
                        
                        # List files to be deployed
                        echo "Files to be deployed:"
                        find dist -type f -name "*.html" -o -name "*.css" -o -name "*.js" -o -name "*.png" -o -name "*.jpg" -o -name "*.gif" -o -name "*.ico"
                    '''
                }
            }
        }
        
        stage('Test') {
            steps {
                echo 'Running basic tests...'
                script {
                    sh '''
                        # Check if HTML files are valid
                        for file in dist/*.html; do
                            if [ -f "$file" ]; then
                                echo "Testing $file..."
                                # Basic syntax check
                                if ! grep -q "</html>" "$file"; then
                                    echo "Warning: $file may not be properly closed"
                                fi
                            fi
                        done
                        
                        echo "All tests passed!"
                    '''
                }
            }
        }
        
        stage('Deploy to S3') {
            steps {
                script {
                    echo "Deploying to S3 bucket: ${params.S3_BUCKET}"
                    withCredentials([aws(credentialsId: env.AWS_CREDENTIALS_ID, region: env.AWS_DEFAULT_REGION)]) {
                        sh '''
                            # Sync files to S3 bucket
                            aws s3 sync dist/ s3://${S3_BUCKET}/ \
                                --delete \
                                --cache-control "public, max-age=31536000" \
                                --exclude "*.html" \
                                --exclude "*.xml" \
                                --exclude "*.json"
                            
                            # Upload HTML files with shorter cache control
                            aws s3 sync dist/ s3://${S3_BUCKET}/ \
                                --delete \
                                --cache-control "public, max-age=0, must-revalidate" \
                                --include "*.html" \
                                --include "*.xml" \
                                --include "*.json"
                            
                            # Set proper content types
                            aws s3 cp s3://${S3_BUCKET}/ s3://${S3_BUCKET}/ \
                                --recursive \
                                --metadata-directive REPLACE \
                                --content-type "text/html" \
                                --exclude "*" \
                                --include "*.html"
                            
                            echo "Deployment to S3 completed successfully!"
                        '''
                    }
                }
            }
        }
        
        stage('Configure S3 Website') {
            steps {
                script {
                    echo 'Configuring S3 bucket for static website hosting...'
                    withCredentials([aws(credentialsId: env.AWS_CREDENTIALS_ID, region: env.AWS_DEFAULT_REGION)]) {
                        sh '''
                            # Configure S3 bucket for static website hosting
                            aws s3 website s3://${S3_BUCKET}/ \
                                --index-document index.html \
                                --error-document error.html
                            
                            # Set bucket policy for public read access
                            cat > bucket-policy.json << EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "PublicReadGetObject",
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::${S3_BUCKET}/*"
        }
    ]
}
EOF
                            
                            aws s3api put-bucket-policy \
                                --bucket ${S3_BUCKET} \
                                --policy file://bucket-policy.json
                            
                            # Enable public access block configuration
                            aws s3api put-public-access-block \
                                --bucket ${S3_BUCKET} \
                                --public-access-block-configuration \
                                BlockPublicAcls=false,IgnorePublicAcls=false,BlockPublicPolicy=false,RestrictPublicBuckets=false
                            
                            echo "S3 website configuration completed!"
                            echo "Website URL: http://${S3_BUCKET}.s3-website-${AWS_REGION}.amazonaws.com"
                        '''
                    }
                }
            }
        }
        
        stage('CloudFront Invalidation') {
            when {
                expression { params.ENABLE_CLOUDFRONT_INVALIDATION && params.CLOUDFRONT_DISTRIBUTION_ID }
            }
            steps {
                script {
                    echo 'Invalidating CloudFront cache...'
                    withCredentials([aws(credentialsId: env.AWS_CREDENTIALS_ID, region: env.AWS_DEFAULT_REGION)]) {
                        sh '''
                            aws cloudfront create-invalidation \
                                --distribution-id ${CLOUDFRONT_DISTRIBUTION_ID} \
                                --paths "/*"
                            
                            echo "CloudFront invalidation initiated successfully!"
                        '''
                    }
                }
            }
        }
    }
    
    post {
        always {
            echo 'Cleaning up workspace...'
            cleanWs()
        }
        success {
            echo 'Pipeline executed successfully!'
            script {
                if (env.SLACK_WEBHOOK_URL) {
                    slackSend(
                        channel: '#deployments',
                        color: 'good',
                        message: "✅ Website deployed successfully to S3!\nBucket: ${params.S3_BUCKET}\nBranch: ${env.BRANCH_NAME}\nBuild: ${env.BUILD_NUMBER}"
                    )
                }
            }
        }
        failure {
            echo 'Pipeline failed!'
            script {
                if (env.SLACK_WEBHOOK_URL) {
                    slackSend(
                        channel: '#deployments',
                        color: 'danger',
                        message: "❌ Website deployment failed!\nBucket: ${params.S3_BUCKET}\nBranch: ${env.BRANCH_NAME}\nBuild: ${env.BUILD_NUMBER}"
                    )
                }
            }
        }
    }
}
