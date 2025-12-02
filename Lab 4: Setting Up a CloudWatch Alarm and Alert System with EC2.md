# Lab 4: Setting Up a CloudWatch Alarm and Alert System with EC2

Welcome to AWS Lab 4 of the SRE Workshop! This hands-on activity introduces you to monitoring Amazon EC2 instances using Amazon CloudWatch. You'll launch a simple EC2 instance, set up alarms for key metrics (like high CPU usage), and configure alerts to notify you via email or SMS when something goes wrong. We'll keep it personalized by using "lab4-{workshoper name}" in resource names (replace {workshoper name} with your actual name, e.g., "lab4-john-server" if your name is John). Optionally, run a sample app on the instance, but the focus is on beginner-friendly monitoring and alerting.

This guide is designed for absolute beginners with an AWS Free Tier account. Each step includes why we're doing it, screenshots aren't possible here but I'll describe where to click, and troubleshooting tips. We'll use the AWS Management Console for simplicity—no coding required unless you add the optional app.

**Time Estimate:** 45-60 minutes (plus 10-15 for optional app setup).  
**Goals:**  
- Launch an EC2 instance and understand basic monitoring.  
- Create CloudWatch alarms to detect issues (e.g., high CPU).  
- Set up notifications (alerts) via email or SMS.  
- Enable AWS Systems Manager for secure, SSH-less access to your instance.  
- Simulate an issue to trigger an alert and see it in action.  
- Clean up to avoid costs.

**Prerequisites:**  
- AWS account (free tier eligible—sign up at aws.amazon.com if needed).  
- Basic familiarity with AWS Console (from previous labs or tutorials).  
- Email address for notifications (and phone number if you want SMS).  
- Optional: SSH client (like Terminal on macOS/Linux, PuTTY on Windows) if adding the app.

**Important Notes:**  
- Costs: EC2 t2.micro is free tier (750 hours/month), but stop/delete after lab to avoid charges (~$0.01/hour if running).  
- Regions: Use us-east-1 for simplicity, but any works.
- Personalization: Throughout this lab, replace "{workshoper name}" with your name (e.g., "john" for John) to make resource names unique.

## Step 1: Explore CloudWatch Monitoring for EC2
**Why?** CloudWatch automatically collects metrics from EC2 (e.g., CPU, network)—no setup needed!

1. Console > EC2
2. Select your instance (by ID or name tag).  
3. Select Monitoring Tab
<img width="2832" height="948" alt="image" src="https://github.com/user-attachments/assets/70e64ac0-4115-418f-ab3c-710f0afd3f49" />


**Explanation:** Metrics update every 5 minutes (basic) or 1 minute (detailed—enable for $0.01/metric/month). Logs are in CloudWatch Logs if you added the app.

**Troubleshooting:** No metrics? Wait 5-10 minutes after launch. Enable detailed monitoring in EC2 > Instance settings > Monitoring.

## Step 2: Create a CloudWatch Alarm
**Why?** Alarms watch metrics and trigger actions (like notifications) when thresholds are breached—e.g., alert if CPU > 70%.

1. Console > CloudWatch > Alarms > Create alarm.
<img width="3334" height="1706" alt="image" src="https://github.com/user-attachments/assets/94fa0038-caaf-4fda-afad-c7e1655719b1" />

<img width="3336" height="1202" alt="image" src="https://github.com/user-attachments/assets/ad1b4ecd-908d-4446-b9db-51c8082963f0" />

2. Select metric: Search "CPUUtilization" > EC2 > Per-Instance > Your instance.
<img width="3102" height="1536" alt="image" src="https://github.com/user-attachments/assets/d4793f02-91cf-4149-93ad-6779138dae62" />
<img width="3080" height="1520" alt="image" src="https://github.com/user-attachments/assets/a0d8c243-e02a-47c0-9fce-c95377a3c356" />

3. Statistic: Average, Period: 1 minute (enable detailed if needed).  
4. Conditions: Static > Greater > 70 (for high CPU).
<img width="3278" height="1574" alt="image" src="https://github.com/user-attachments/assets/f131191c-1c25-4c9a-ba93-e2637297e1f7" />


5. Additional: Treat missing data as "missing" (default).  
6. Actions: In alarm > Create new SNS topic (next step). Name alarm: `HighCPU-lab4-{workshoper name}-server`.  
7. Create.

**Explanation:** Alarm states: OK (normal), ALARM (breached), INSUFFICIENT_DATA (not enough info).

## Step 3: Set Up Alerts (Notifications)
**Why?** Alarms alone don't notify—use Amazon SNS (Simple Notification Service) for email/SMS.

1. In alarm creation (or SNS Console > Topics > Create topic): Standard type, Name: `lab4-{workshoper name}-alerts`.  
2. Create subscription: Protocol: Email, Endpoint: your-email@example.com. Confirm via email link.  
   - For SMS: Protocol: SMS, Endpoint: +1yourphonenumber (international format).  
   - For Slack: Use Application integration (webhook URL as HTTP endpoint).
  <img width="2730" height="1374" alt="image" src="https://github.com/user-attachments/assets/031e712b-8c8e-4a33-9fa1-2dd3a8fe75a5" />
 
3. Back in alarm: Add notification > Alarm state trigger: In alarm > Send to SNS topic `lab4-{workshoper name}-alerts`.  
4. Finish alarm.

**Explanation:** SNS sends messages when alarm triggers. Test by subscribing and publishing a test message in SNS.

**Troubleshooting:** No confirmation email? Check spam. SMS not working? Verify phone in SNS > Mobile > Text messaging.

## Step 4: Simulate an Issue and Test the Alert
**Why?** See the system in action—generate load to trigger the alarm.

1. Access the instance via Session Manager (from Step 2): Start a session > Run commands.  
2. Install stress: `sudo yum install stress -y`.  
3. Run: `stress --cpu 2 --timeout 300` (high CPU for 5 minutes).  
4. Monitor: CloudWatch Metrics—watch CPU spike >70%.  
5. Wait 1-2 minutes: Alarm goes to ALARM state.  
6. Check email/SMS: Receive notification like "ALARM: HighCPU-lab4-{workshoper name}-server".  
7. Resolve: The stress stops automatically. CPU drops—alarm back to OK, optional resolved notification.

**Explanation:** This simulates a real issue (e.g., app overload). For a sample app, high traffic could trigger it. Using Session Manager keeps it secure.

**Troubleshooting:** No trigger? Ensure detailed monitoring enabled. No notification? Check SNS subscriptions status (Confirmed?).

## Step 5: Cleanup to Avoid Costs
**Why?** Free tier has limits—delete resources.

1. EC2 > Instances > Terminate `lab4-{workshoper name}-server`.  
2. CloudWatch > Alarms > Delete.  
3. SNS > Topics > Delete `lab4-{workshoper name}-alerts` (delete subscriptions first).  
4. IAM > Roles > Delete `ssm-lab4-{workshoper name}-role`.  
5. Delete key pair if not needed.

You've completed AWS Lab 4! You now know how to monitor EC2 with CloudWatch alarms, alerts, and secure access via Systems Manager. Reflect: How does SSM make management easier? (No keys, audited sessions.) Save as `aws-lab4-guide.md`. Ready for more?
