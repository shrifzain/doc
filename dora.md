# DORA Metrics Calculation: A Step-by-Step Guide

## Introduction

This document explains how to calculate the three key DORA (DevOps Research and Assessment) metrics using data from Jenkins builds, GitHub commits, and pull requests. These metrics provide insights into software delivery performance and help teams identify areas for improvement.

## The Data Sources

We're working with three CSV files:

1. **jenkins_build_history.csv** - Contains deployment data
   - Fields: job_name, build_number, result, timestamp, date, duration_sec, triggered_by, parameters, build_url, commit_author, commit_id, commit_message, branch

2. **shrifzain_ApiNet_commits.csv** - Contains commit history
   - Fields: HA (commit hash), Author, Date, Message, Branch/PR

3. **shrifzain_ApiNet_pull_requests.csv** - Contains pull request information
   - Fields: PR_Number, Title, State, Created_At, Merged_At, Closed_At, Author, Branch

## Metric 1: Deployment Frequency

### Definition
Deployment Frequency measures how often code is successfully deployed to production.

### Step-by-Step Calculation

#### Step 1: Extract Successful Deployments
We first identify all successful deployments from the Jenkins build history:

```python
successful_deployments = jenkins_df[jenkins_df['result'] == 'SUCCESS']
```

From our sample data, we have:
- Build #60: 2025-04-12 21:22:32 (SUCCESS)
- Build #59: 2025-04-12 18:41:18 (SUCCESS)
- Build #58: 2025-04-09 01:35:51 (SUCCESS)

#### Step 2: Determine the Date Range
Calculate the time span covered by our data:

```python
start_date = jenkins_df['date'].min()  # 2025-04-09 01:34:54
end_date = jenkins_df['date'].max()    # 2025-04-12 21:22:32
date_range = (end_date - start_date).days + 1  # 4 days
```

#### Step 3: Calculate Deployment Frequency
Divide the number of successful deployments by the date range:

```python
deployment_frequency = len(successful_deployments) / date_range
# 3 deployments / 4 days = 0.75 deployments per day
```

#### Step 4: Determine Performance Level
Map the result to DORA performance levels:

```python
if deployment_frequency > 1:
    performance_level = "Elite"  # Multiple deployments per day
elif deployment_frequency >= 1/7:
    performance_level = "High"   # Between once per day and once per week
elif deployment_frequency >= 1/30:
    performance_level = "Medium" # Between once per week and once per month
else:
    performance_level = "Low"    # Less than once per month
```

With 0.75 deployments per day, the performance level is **High**.

## Metric 2: Lead Time for Changes

### Definition
Lead Time for Changes measures the time it takes for code to go from commit to running in production.

### Step-by-Step Calculation

#### Step 1: Map Commits to Pull Requests and Deployments
This is the most complex part of the calculation. We need to:
1. Identify which commit was deployed in each successful build
2. Find the pull request associated with that commit
3. Locate the first commit in that pull request
4. Calculate the time difference between the first commit and when it was deployed

```python
for _, deploy in jenkins_df[jenkins_df['result'] == 'SUCCESS'].iterrows():
    commit_hash = deploy['commit_id']
    deployment_time = deploy['date']
    
    # Find matching PR for this commit
    for _, pr in pr_df.iterrows():
        pr_number = pr['PR_Number']
        
        # Find commits associated with this PR
        for _, commit in commits_df.iterrows():
            if f"PR #{pr_number}" in str(commit['Branch/PR']) and commit['HA'] == commit_hash:
                # Find the first commit in this PR
                pr_commits = commits_df[commits_df['Branch/PR'].str.contains(f"PR #{pr_number}", na=False)]
                if not pr_commits.empty:
                    # Get earliest commit time for this PR
                    first_commit_time = min(pr_commits['Date'])
                    lead_time_minutes = (deployment_time - first_commit_time).total_seconds() / 60
                    
                    lead_times.append({
                        'pr_number': pr_number,
                        'first_commit': first_commit_time,
                        'deployment_time': deployment_time,
                        'lead_time_minutes': lead_time_minutes
                    })
```

From our sample data, we found:

For PR #2:
- First commit: 2025-04-12T19:04:11Z (Update Program.cs)
- Deployed to production: 2025-04-12 21:22:32 (Build #60)
- Lead time: 2 hours 18 minutes (138 minutes)

For PR #1:
- First commit: 2025-04-12T17:16:53Z (add cae Program.cs)
- Deployed to production: 2025-04-12 18:41:18 (Build #59)
- Lead time: 1 hour 24 minutes (84 minutes)

#### Step 2: Calculate Average Lead Time
Add up all lead times and divide by the number of changes:

```python
avg_lead_time_minutes = sum(item['lead_time_minutes'] for item in lead_times) / len(lead_times)
# (138 + 84) / 2 = 111 minutes = 1 hour 51 minutes
```

#### Step 3: Determine Performance Level
Map the result to DORA performance levels:

```python
if avg_lead_time_hours < 1:
    performance_level = "Elite"         # Less than one hour
elif avg_lead_time_hours < 24:
    performance_level = "Elite (slightly above threshold)" # Between one hour and one day
elif avg_lead_time_hours < 168:         # 7 days
    performance_level = "High"          # Between one day and one week
elif avg_lead_time_hours < 720:         # 30 days
    performance_level = "Medium"        # Between one week and one month
else:
    performance_level = "Low"           # More than one month
```

With an average lead time of 1 hour 51 minutes, the performance level is **Elite (slightly above threshold)**.

## Metric 3: Change Failure Rate

### Definition
Change Failure Rate measures the percentage of deployments that cause a failure in production.

### Step-by-Step Calculation

#### Step 1: Count Total and Failed Deployments
Identify the total number of deployments and how many failed:

```python
total_deployments = len(jenkins_df)
failed_deployments = len(jenkins_df[jenkins_df['result'] == 'FAILURE'])
```

From our sample data:
- Total deployments: 4 (Builds #57, #58, #59, #60)
- Failed deployments: 1 (Build #57)

#### Step 2: Calculate Failure Rate
Divide the number of failed deployments by the total deployments:

```python
change_failure_rate = (failed_deployments / total_deployments) * 100
# (1 / 4) * 100% = 25%
```

#### Step 3: Determine Performance Level
Map the result to DORA performance levels:

```python
if change_failure_rate <= 15:
    performance_level = "Elite"   # 0-15%
elif change_failure_rate <= 30:
    performance_level = "High"    # 16-30%
elif change_failure_rate <= 45:
    performance_level = "Medium"  # 31-45%
else:
    performance_level = "Low"     # 46-60% or higher
```

With a change failure rate of 25%, the performance level is **High**.

## Putting It All Together: The Python Script

The provided Python script automates all of these calculations. Here's a breakdown of its structure:

1. **Data Loading Function**: Reads the CSV files and converts date strings to datetime objects
2. **Metric Calculation Functions**: One for each DORA metric
   - `calculate_deployment_frequency()`
   - `calculate_lead_time()`
   - `calculate_change_failure_rate()`
3. **Main Function**: Orchestrates the overall process and displays the results

## Summary of Results

From our sample data, the team's performance is:

| Metric | Value | Performance Level |
|--------|-------|------------------|
| Deployment Frequency | 0.75 per day | High |
| Lead Time for Changes | 1 hour 51 minutes | Elite (slightly above threshold) |
| Change Failure Rate | 25% | High |

## Recommendations for Improvement

Based on these metrics, here are some recommendations:

1. **Deployment Frequency**: To reach Elite level, implement:
   - More automation in testing and deployment
   - Smaller, more frequent code changes
   - Feature flags to decouple deployment from feature release

2. **Lead Time for Changes**: The team is already near Elite level, but could:
   - Further optimize code review processes
   - Enhance CI/CD pipelines to reduce waiting time
   - Consider implementing trunk-based development

3. **Change Failure Rate**: To move from High to Elite level:
   - Increase automated test coverage
   - Implement more thorough code reviews
   - Consider chaos engineering practices to identify potential failure points

## Conclusion

The DORA metrics provide valuable insights into the software delivery performance of a team. By consistently tracking and working to improve these metrics, teams can enhance their delivery capabilities and provide more value to their customers.

Regular monitoring of these metrics will help the team identify trends and measure the impact of process improvements over time. The goal should be to gradually move all metrics toward the Elite performance level.
