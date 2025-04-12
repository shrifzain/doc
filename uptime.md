# AWS Elastic Beanstalk Uptime Calculation Guide

This document provides a comprehensive guide to calculating, monitoring, and reporting uptime for AWS Elastic Beanstalk environments. It includes both conceptual explanations and practical implementation details.

## Table of Contents

- [Understanding AWS Environment Health](#understanding-aws-environment-health)
- [Collecting Health Data](#collecting-health-data)
- [Uptime Calculation Approaches](#uptime-calculation-approaches)
- [Real-World Example](#real-world-example)
- [Automated Reporting Implementation](#automated-reporting-implementation)
- [Scheduling Reports](#scheduling-reports)
- [Best Practices](#best-practices)

## Understanding AWS Environment Health

AWS Elastic Beanstalk provides environment health statuses that are represented numerically:

| Value | Status | Description |
|-------|--------|-------------|
| 1 | Green/OK | Environment is fully functional |
| 2 | Yellow/Warning | Environment is experiencing issues but still operational |
| 3 | Red/Error | Environment is not functioning properly |
| 0 | Grey/No Data | No data available |

These health statuses are available as CloudWatch metrics and form the basis for uptime calculations.

## Collecting Health Data

To calculate uptime, you first need to collect environment health data from CloudWatch:

```python
def get_environment_health_data(cloudwatch_client, environment_name, application_name, start_time, end_time):
    """
    Fetch environment health data from CloudWatch
    """
    health_metrics = cloudwatch_client.get_metric_data(
        MetricDataQueries=[
            {
                'Id': 'environment_health',
                'MetricStat': {
                    'Metric': {
                        'Namespace': 'AWS/ElasticBeanstalk',
                        'MetricName': 'EnvironmentHealth',
                        'Dimensions': [
                            {'Name': 'EnvironmentName', 'Value': environment_name},
                            {'Name': 'ApplicationName', 'Value': application_name}
                        ]
                    },
                    'Period': 300,  # 5-minute intervals
                    'Stat': 'Average',
                },
                'ReturnData': True
            }
        ],
        StartTime=start_time,
        EndTime=end_time
    )
    
    return health_metrics['MetricDataResults'][0]['Values'], health_metrics['MetricDataResults'][0]['Timestamps']
```

This function will return an array of health status values at 5-minute intervals within your specified time range.

## Uptime Calculation Approaches

### Basic Uptime Calculation

The simplest approach counts only fully healthy (Green/OK) periods:

```python
def calculate_basic_uptime(health_data):
    """
    Calculate uptime based on fully healthy periods only
    """
    total_points = len(health_data) if health_data else 1
    healthy_points = sum(1 for point in health_data if point == 1)
    
    return (healthy_points / total_points) * 100
```

### Enhanced Uptime Calculation

A more nuanced approach distinguishes between different health states:

```python
def calculate_enhanced_uptime(health_data):
    """
    Calculate different uptime metrics considering all health states
    """
    total_points = len(health_data) if health_data else 1
    
    # Count different states
    green_points = sum(1 for point in health_data if point == 1)   # Fully healthy
    yellow_points = sum(1 for point in health_data if point == 2)  # Degraded but working
    red_points = sum(1 for point in health_data if point == 3)     # Not working
    no_data_points = sum(1 for point in health_data if point == 0) # No data
    
    # Calculate percentages
    metrics = {
        'full_health_percentage': (green_points / total_points) * 100,
        'availability_percentage': ((green_points + yellow_points) / total_points) * 100,
        'outage_percentage': (red_points / total_points) * 100,
        'no_data_percentage': (no_data_points / total_points) * 100,
        'status_breakdown': {
            'green_percentage': (green_points / total_points) * 100,
            'yellow_percentage': (yellow_points / total_points) * 100,
            'red_percentage': (red_points / total_points) * 100,
            'no_data_percentage': (no_data_points / total_points) * 100
        }
    }
    
    return metrics
```

### Tracking Continuous Outages

For SLA reporting, consecutive outage time is often more important than individual data points:

```python
def find_longest_outage(health_data, timestamps):
    """
    Find the longest continuous outage
    """
    current_outage = 0
    longest_outage = 0
    outage_start = None
    longest_outage_start = None
    longest_outage_end = None
    
    for i, point in enumerate(health_data):
        if point == 3:  # Red/Error
            if current_outage == 0:
                outage_start = timestamps[i]
            current_outage += 1
        else:
            if current_outage > longest_outage:
                longest_outage = current_outage
                longest_outage_start = outage_start
                longest_outage_end = timestamps[i-1]
            current_outage = 0
    
    # Check if we ended in an outage
    if current_outage > longest_outage:
        longest_outage = current_outage
        longest_outage_start = outage_start
        longest_outage_end = timestamps[-1]
    
    return {
        'longest_continuous_outage_points': longest_outage,
        'longest_continuous_outage_minutes': longest_outage * 5,  # Assuming 5-minute periods
        'longest_outage_start': longest_outage_start,
        'longest_outage_end': longest_outage_end
    }
```

## Real-World Example

Let's examine a real-world scenario with 24 hours of health data:

```
# Sample of health_data values (288 total points)
health_data = [
    1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1,  # Hour 1: All green
    1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1,  # Hour 2: All green
    1, 1, 2, 2, 2, 1, 1, 1, 1, 1, 1, 1,  # Hour 3: Some yellow (degraded)
    1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1,  # Hour 4: All green
    1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1,  # Hour 5: All green
    1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1,  # Hour 6: All green
    3, 3, 3, 2, 2, 2, 1, 1, 1, 1, 1, 1,  # Hour 7: Brief outage, then degraded
    1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1,  # Hour 8: All green
    # ... remaining 16 hours (192 data points) all green (value 1)
]
```

Analysis of this data:
- Green (status = 1): 273 points
- Yellow (status = 2): 6 points
- Red (status = 3): 3 points
- No data (status = 0): 0 points
- Total points: 282

Uptime calculation results:
- Basic uptime (Green only): 96.81%
- Available uptime (Green + Yellow): 98.94%
- Outage percentage: 1.06%
- Longest continuous outage: 15 minutes

Sample report output:

```json
{
  "date": "2025-04-10",
  "environment_name": "production-web-app",
  "application_name": "my-elastic-beanstalk-app",
  "metrics": {
    "total_data_points": 282,
    "uptime_percentage": 96.81,
    "availability_percentage": 98.94,
    "outage_percentage": 1.06,
    "longest_continuous_outage_minutes": 15,
    "outage_start_time": "2025-04-10T06:00:00Z",
    "outage_end_time": "2025-04-10T06:15:00Z"
  },
  "status": "Degraded Performance",
  "status_breakdown": {
    "green_percentage": 96.81,
    "yellow_percentage": 2.13,
    "red_percentage": 1.06,
    "no_data_percentage": 0.00
  }
}
```

## Automated Reporting Implementation

Below is a complete AWS Lambda function that can be scheduled to run daily to generate uptime reports:

```python
import boto3
import datetime
import json
import os

def lambda_handler(event, context):
    # Configure these variables
    eb_environment_name = os.environ.get('EB_ENVIRONMENT_NAME')
    eb_application_name = os.environ.get('EB_APPLICATION_NAME')
    s3_bucket = os.environ.get('S3_BUCKET_NAME')
    sns_topic_arn = os.environ.get('SNS_TOPIC_ARN')  # Optional for email notifications
    
    # Initialize AWS clients
    cloudwatch = boto3.client('cloudwatch')
    s3 = boto3.client('s3')
    sns = boto3.client('sns')
    
    # Calculate time range (last 24 hours)
    end_time = datetime.datetime.utcnow()
    start_time = end_time - datetime.timedelta(days=1)
    
    # Get environment health metrics
    health_metrics = cloudwatch.get_metric_data(
        MetricDataQueries=[
            {
                'Id': 'environment_health',
                'MetricStat': {
                    'Metric': {
                        'Namespace': 'AWS/ElasticBeanstalk',
                        'MetricName': 'EnvironmentHealth',
                        'Dimensions': [
                            {'Name': 'EnvironmentName', 'Value': eb_environment_name},
                            {'Name': 'ApplicationName', 'Value': eb_application_name}
                        ]
                    },
                    'Period': 300,  # 5-minute intervals
                    'Stat': 'Average',
                },
                'ReturnData': True
            },
            {
                'Id': 'instance_health',
                'MetricStat': {
                    'Metric': {
                        'Namespace': 'AWS/ElasticBeanstalk',
                        'MetricName': 'InstancesHealth',
                        'Dimensions': [
                            {'Name': 'EnvironmentName', 'Value': eb_environment_name},
                            {'Name': 'ApplicationName', 'Value': eb_application_name},
                            {'Name': 'InstanceHealthStatus', 'Value': 'Ok'}
                        ]
                    },
                    'Period': 300,
                    'Stat': 'Average',
                },
                'ReturnData': True
            }
        ],
        StartTime=start_time,
        EndTime=end_time
    )
    
    # Extract data and timestamps
    health_data = health_metrics['MetricDataResults'][0]['Values']
    timestamps = health_metrics['MetricDataResults'][0]['Timestamps']
    total_points = len(health_data) if health_data else 1
    
    # Count different states
    green_points = sum(1 for point in health_data if point == 1)
    yellow_points = sum(1 for point in health_data if point == 2)
    red_points = sum(1 for point in health_data if point == 3)
    no_data_points = sum(1 for point in health_data if point == 0)
    
    # Calculate uptime percentages
    uptime_percentage = (green_points / total_points) * 100
    availability_percentage = ((green_points + yellow_points) / total_points) * 100
    outage_percentage = (red_points / total_points) * 100
    
    # Find longest continuous outage
    current_outage = 0
    longest_outage = 0
    outage_start = None
    longest_outage_start = None
    longest_outage_end = None
    
    for i, point in enumerate(health_data):
        if point == 3:  # Red/Error
            if current_outage == 0:
                outage_start = timestamps[i]
            current_outage += 1
        else:
            if current_outage > longest_outage:
                longest_outage = current_outage
                longest_outage_start = outage_start
                longest_outage_end = timestamps[i-1]
            current_outage = 0
    
    # Check if we ended in an outage
    if current_outage > longest_outage:
        longest_outage = current_outage
        longest_outage_start = outage_start
        longest_outage_end = timestamps[-1]
    
    # Determine status
    if uptime_percentage >= 99.9:
        status = "Operational"
    elif availability_percentage >= 95:
        status = "Degraded Performance"
    else:
        status = "Partial Outage"
    
    # Format timestamps for outages if they exist
    outage_start_str = longest_outage_start.strftime('%Y-%m-%dT%H:%M:%SZ') if longest_outage_start else None
    outage_end_str = longest_outage_end.strftime('%Y-%m-%dT%H:%M:%SZ') if longest_outage_end else None
    
    # Add instance health data
    instance_health = health_metrics['MetricDataResults'][1]['Values']
    avg_healthy_instances = round(sum(instance_health) / len(instance_health), 2) if instance_health else 0
    
    # Generate report
    report = {
        "date": end_time.strftime('%Y-%m-%d'),
        "timestamp": end_time.strftime('%Y-%m-%dT%H:%M:%SZ'),
        "period": {
            "start": start_time.strftime('%Y-%m-%dT%H:%M:%SZ'),
            "end": end_time.strftime('%Y-%m-%dT%H:%M:%SZ')
        },
        "environment_name": eb_environment_name,
        "application_name": eb_application_name,
        "metrics": {
            "total_data_points": total_points,
            "uptime_percentage": round(uptime_percentage, 2),
            "availability_percentage": round(availability_percentage, 2),
            "outage_percentage": round(outage_percentage, 2),
            "average_healthy_instances": avg_healthy_instances,
            "longest_continuous_outage_minutes": longest_outage * 5,  # Assuming 5-minute periods
            "outage_start_time": outage_start_str,
            "outage_end_time": outage_end_str
        },
        "status": status,
        "status_breakdown": {
            "green_percentage": round((green_points / total_points) * 100, 2),
            "yellow_percentage": round((yellow_points / total_points) * 100, 2),
            "red_percentage": round((red_points / total_points) * 100, 2),
            "no_data_percentage": round((no_data_points / total_points) * 100, 2)
        }
    }
    
    # Format report as JSON
    report_json = json.dumps(report, indent=2)
    
    # Save report to S3
    date_str = end_time.strftime('%Y-%m-%d')
    s3_key = f"uptime-reports/{date_str}.json"
    
    s3.put_object(
        Bucket=s3_bucket,
        Key=s3_key,
        Body=report_json,
        ContentType='application/json'
    )
    
    # Optionally send report via SNS
    if sns_topic_arn:
        subject = f"Daily Uptime Report: {eb_environment_name} - {date_str}"
        message = f"""Daily Uptime Report for {eb_environment_name}
        
Period: {start_time.strftime('%Y-%m-%d %H:%M:%S')} to {end_time.strftime('%Y-%m-%d %H:%M:%S')}
Uptime: {round(uptime_percentage, 2)}%
Availability: {round(availability_percentage, 2)}%
Status: {status}
Average Healthy Instances: {avg_healthy_instances}

Full report stored in S3: s3://{s3_bucket}/{s3_key}
"""
        sns.publish(
            TopicArn=sns_topic_arn,
            Subject=subject,
            Message=message
        )
    
    return {
        'statusCode': 200,
        'body': json.dumps({
            'message': 'Daily uptime report generated successfully',
            's3_location': f"s3://{s3_bucket}/{s3_key}"
        })
    }
```

## Scheduling Reports

Reports can be scheduled using CloudWatch Events/EventBridge rules with cron expressions:

### Daily Report (midnight UTC)
```
cron(0 0 * * ? *)
```

### Hourly Report
```
cron(0 * * * ? *)
```

### Twice Daily (9 AM and 9 PM UTC)
```
cron(0 9,21 * * ? *)
```

### Every Weekday at 8 AM UTC
```
cron(0 8 ? * MON-FRI *)
```

To create a schedule using AWS CLI:
```
aws events put-rule --name UptimeReportRule --schedule-expression "cron(0 0 * * ? *)"
aws lambda add-permission --function-name UptimeReportFunction --statement-id DailyReport --action lambda:InvokeFunction --principal events.amazonaws.com --source-arn arn:aws:events:region:account-id:rule/UptimeReportRule
aws events put-targets --rule UptimeReportRule --targets Id=1,Arn=arn:aws:lambda:region:account-id:function:UptimeReportFunction
```

## Best Practices

### For Uptime Calculation
1. **Define service level tiers** - Clearly define what constitutes "available" vs "degraded" vs "down"
2. **Consider maintenance windows** - Exclude planned maintenance from downtime calculations
3. **Use multiple data sources** - Don't rely solely on Elastic Beanstalk health; incorporate application-level health checks
4. **Calculate different metrics** - Report both strict uptime (green only) and availability (green + yellow)
5. **Track continuous outages** - Report both percentage and longest continuous outage

### For Reporting
1. **Store historical data** - Keep all daily reports for trending analysis
2. **Provide context** - Include environment details and configuration changes
3. **Automate alerts** - Set up notifications for uptime falling below thresholds
4. **Visualize trends** - Create dashboards showing uptime over time
5. **Calculate SLA compliance** - Compare uptime with service level agreements

### For Status Pages
1. **Simplify status display** - Use clear terms like "Operational", "Degraded", "Outage"
2. **Show historical uptime** - Display 90-day rolling uptime percentage
3. **Provide transparency** - Include incident history and resolution times
4. **Update in real-time** - Ensure status reflects current conditions
5. **Make it accessible** - Ensure status page has high availability

By following this guide, you can implement a comprehensive uptime monitoring and reporting system for your AWS Elastic Beanstalk environments that provides meaningful insights for both technical teams and business stakeholders.
