# Canary Deployment Strategy Documentation

## Overview

Canary deployment is a technique for reducing the risk of introducing a new software version in production by gradually shifting traffic from the original version to the new release. Unlike blue-green deployment which switches all traffic at once, canary deployment exposes the new version to a small subset of users before rolling it out to the entire infrastructure.

## Benefits of Canary Deployment

- **Reduced Risk**: Limited exposure means fewer users are affected if issues are detected
- **Real User Testing**: Tests new version with actual user traffic in production environment
- **Early Issue Detection**: Problems can be identified before affecting all users
- **Progressive Rollout**: Ability to gradually increase traffic to the new version
- **Immediate Rollback**: Quick reversion to stable version if issues arise

## Jenkins Pipeline Implementation

### Required Resources

- **Production Server(s)**: Main server(s) handling most of the production traffic
- **Canary Server(s)**: Server(s) receiving a percentage of production traffic for testing
- **Load Balancer**: Capable of weighted traffic distribution (like AWS ELB/ALB)
- **Monitoring System**: For tracking performance and error metrics (like CloudWatch)

### Pipeline Structure

```
                    ┌───────────────┐
                    │   Get Code    │
                    └───────┬───────┘
                            │
                    ┌───────▼───────┐
                    │     Build     │
                    └───────┬───────┘
                            │
                    ┌───────▼───────┐
                    │   Save to S3  │
                    └───────┬───────┘
                            │
                    ┌───────▼───────────┐
                    │ Deploy to Canary  │
                    └───────┬───────────┘
                            │
                    ┌───────▼───────────┐
                    │Canary Health Check│
                    └───────┬───────────┘
                            │
              ┌─────────────▼─────────────┐
              │Adjust LB for Canary Traffic│
              └─────────────┬─────────────┘
                            │
              ┌─────────────▼─────────────┐
              │ Monitor Canary Performance │
              └─────────────┬─────────────┘
                            │
              ┌─────────────▼─────────────┐
              │  Full Deployment Decision  │
              └─────────────┬─────────────┘
                            │
           ┌───────────────┐│┌───────────────┐
           │   Rollback    ││|Deploy to Prod │
           │    Canary     │││               │
           └───────────────┘│└───────┬───────┘
                            │        │
                            │┌───────▼───────┐
                            ││Switch to Full │
                            ││Prod Traffic   │
                            │└───────────────┘
                            │
                    ┌───────▼───────┐
                    │   Clean up    │
                    └───────────────┘
```

### Key Pipeline Stages

1. **Initialize & Build**
   - Get code from repository
   - Build application
   - Save artifacts to storage (S3)

2. **Deploy to Canary**
   - Deploy new version to canary server(s)
   - Run health checks to verify basic functionality
   - Configure load balancer to direct small percentage (typically 5-20%) of traffic to canary

3. **Monitor & Evaluate**
   - Collect metrics for a predetermined period (5-30 minutes typically)
   - Monitor error rates, response times, CPU/memory usage
   - Compare metrics against predefined thresholds

4. **Decision Point**
   - Automated evaluation based on metrics
   - Optional manual approval step
   - Decision to proceed or rollback

5. **Complete Deployment or Rollback**
   - If successful: Deploy to all production servers and shift 100% traffic
   - If issues detected: Revert load balancer to direct 100% traffic to original version

## Implementation Example

The following Jenkins pipeline demonstrates a canary deployment strategy:

```groovy
pipeline {
    agent any

    environment {
        PROJECT_NAME = 'ProNet.Api'
        SOLUTION_FILE = 'Api.csproj'
        PUBLISH_DIR = 'publish'
        GITHUB_REPO = 'https://github.com/shrifzain/ApiNet.git'
        S3_BUCKET = 'pronet-artifacts'
        MAIN_SERVER = '54.152.102.169'           // Main production server
        CANARY_SERVER = '54.158.110.100'         // Canary server 
        SSH_KEY_ID = 'sheraa-ssh-key'            // Jenkins credential ID for SSH key
        AWS_KEY_ID = 'awss'                      // AWS credential ID
        CANARY_TRAFFIC_PERCENTAGE = '20'         // Initial traffic percentage for canary
    }

    stages {
        stage('Get Code') {
            steps {
                echo 'Getting code from GitHub'
                git url: "${GITHUB_REPO}", branch: 'master', credentialsId: '01143717'
            }
        }

        stage('Build') {
            steps {
                echo 'Building the app'
                sh "dotnet publish ${SOLUTION_FILE} -c Release -o ${PUBLISH_DIR}"
            }
        }

        stage('Save to S3') {
            steps {
                echo 'Saving app to S3'
                withAWS(credentials: "${AWS_KEY_ID}") {
                    sh "aws s3 cp ${PUBLISH_DIR} s3://${S3_BUCKET}/${PROJECT_NAME}/${BUILD_NUMBER}/ --recursive"
                }
            }
        }

        stage('Deploy to Canary') {
            steps {
                echo "Deploying to canary server: ${CANARY_SERVER}"
                // Deploy code to canary server
            }
        }

        stage('Canary Health Check') {
            steps {
                echo 'Running health checks on canary'
                // Run health checks against canary deployment
            }
        }

        stage('Adjust Load Balancer for Canary Traffic') {
            steps {
                echo "Directing ${CANARY_TRAFFIC_PERCENTAGE}% of traffic to canary"
                // Configure load balancer to send percentage of traffic to canary
            }
        }

        stage('Monitor Canary Performance') {
            steps {
                echo 'Monitoring canary performance'
                // Monitor metrics for specified period
            }
        }

        stage('Full Deployment Decision') {
            steps {
                // Decide whether to proceed based on metrics
                // Option for manual approval
            }
        }

        stage('Deploy to Production') {
            when {
                expression { env.ROLLBACK != 'true' }
            }
            steps {
                echo "Deploying to main production server: ${MAIN_SERVER}"
                // Deploy to production servers
            }
        }

        stage('Switch to Full Production Traffic') {
            when {
                expression { env.ROLLBACK != 'true' }
            }
            steps {
                echo "Shifting 100% of traffic to production"
                // Update load balancer to direct all traffic to updated production
            }
        }

        stage('Rollback Canary') {
            when {
                expression { env.ROLLBACK == 'true' }
            }
            steps {
                echo "Rolling back canary deployment"
                // Revert load balancer to original configuration
            }
        }
    }

    post {
        always {
            echo 'Cleaning up'
            cleanWs()
        }
        success {
            echo 'Deployment successful!'
        }
        failure {
            echo 'Deployment failed!'
            // Initiate rollback if not already in rollback mode
        }
    }
}
```

## Key Configuration Parameters

- **Canary Traffic Percentage**: Typically 5-20% of users; adjust based on application criticality
- **Monitoring Duration**: Usually 5-30 minutes; adjust based on traffic volume and risk tolerance
- **Success Metrics**: Define thresholds for:
  - Error rates (e.g., <1%)
  - Response times (e.g., <300ms avg)
  - CPU/memory usage (e.g., <80% utilization)
  - Business metrics (e.g., conversion rates within 2% of baseline)

## AWS Implementation Specifics

### Load Balancer Configuration

To implement canary deployments with AWS resources:

1. **Create Target Groups**: One for main production servers, one for canary servers
2. **Configure ALB/ELB**: Use weighted target groups to control traffic distribution
3. **Modify Weights**: Use AWS CLI to adjust traffic percentages

Example AWS CLI command:
```bash
aws elbv2 modify-listener --listener-arn $LISTENER_ARN --default-actions '[
    {
        "Type": "forward",
        "ForwardConfig": {
            "TargetGroups": [
                {
                    "TargetGroupArn": "$MAIN_TARGET_GROUP",
                    "Weight": 80
                },
                {
                    "TargetGroupArn": "$CANARY_TARGET_GROUP",
                    "Weight": 20
                }
            ]
        }
    }
]'
```

### Monitoring with CloudWatch

1. **Create Custom Metrics**: Track application-specific performance indicators
2. **Set Up Alarms**: Configure thresholds for automatic evaluation
3. **Create Dashboard**: Visualize canary vs. production performance

## Best Practices

1. **Start Small**: Begin with a small percentage (5%) and gradually increase
2. **Proper Segmentation**: Ensure canary traffic is representative of overall user base
3. **Comprehensive Monitoring**: Monitor both technical and business metrics
4. **Automated Rollbacks**: Define clear thresholds for automatic rollbacks
5. **Consistent Environments**: Ensure canary and production environments are identical
6. **Session Stickiness**: Consider maintaining user sessions on same version
7. **Geographic Distribution**: For global applications, consider deploying canaries by region

## Comparison with Blue-Green Deployment

| Feature | Canary Deployment | Blue-Green Deployment |
|---------|------------------|----------------------|
| Traffic Shift | Gradual | All at once |
| Risk Level | Lower | Higher |
| Complexity | More complex | Simpler |
| Infrastructure Needs | Can use existing resources | Requires full duplicate environment |
| Testing with Real Users | Yes, subset of users | Only after full switch |
| Rollback Speed | Very fast | Fast |
| Deployment Time | Longer (monitoring period) | Shorter |

## Conclusion

Canary deployment provides a safer approach to releasing new versions by limiting exposure and allowing early detection of issues. While it requires more complex monitoring and traffic management than blue-green deployment, it offers significant risk reduction for critical applications.
