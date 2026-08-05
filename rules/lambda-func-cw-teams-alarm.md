## Alarm normal

- Tech stack: python (latest version)
- Env: TEAMS_EMERGENCY_WEBHOOK_URL, TEAMS_HIGH_WEBHOOK_URL (workflow)

```text
import json
import os
import urllib3
from datetime import datetime

http = urllib3.PoolManager()

def lambda_handler(event, context):
    sns_message = json.loads(event['Records'][0]['Sns']['Message'])
    print(sns_message)
    
    alarm_name = sns_message['AlarmName']
    new_state_reason = sns_message['NewStateReason']
    metric_name = sns_message['Trigger']['MetricName']
    service_name = sns_message['Trigger']['Namespace']
    alarm_description = sns_message.get('AlarmDescription', 'No description provided')
    
    dimensions = sns_message.get('Trigger', {}).get('Dimensions', [])
    resource_info = ', '.join([f"{d['name']}: {d['value']}" for d in dimensions]) if dimensions else 'Unknown resource'

    timestamp = sns_message['StateChangeTime']

    try:
        dt = datetime.strptime(timestamp, "%Y-%m-%dT%H:%M:%S.%f%z")
        timestamp = dt.strftime("%Y/%m/%d %H:%M")
    except ValueError:
        # Trường hợp không có milliseconds
        dt = datetime.strptime(timestamp, "%Y-%m-%dT%H:%M:%S%z")
        timestamp = dt.strftime("%Y/%m/%d %H:%M")
    
    priority = 'High'
    # if 'Priority: Risk' in alarm_description:
    #     priority = 'Risk'
    if 'Priority: Emergency' in alarm_description:
        priority = 'Emergency'
    
    theme_colors = {
        'High': '4287f5',
        # 'Risk': 'FFA500',
        'Emergency': 'FF0000'
    }
    theme_color = theme_colors.get(priority, 'FF0000')

    card = {
    "type": "AdaptiveCard",
    "$schema": "http://adaptivecards.io/schemas/adaptive-card.json",
    "version": "1.5",
    "body": [
        {
            "type": "TextBlock",
            "text": f"🚨 AWS Alarm: {alarm_name}",
            "weight": "Bolder",
            "size": "Large",
            "color": "Attention" if priority == "Emergency" else "Warning"
        },
        {
            "type": "FactSet",
            "facts": [
                {"title": "Priority", "value": priority},
                {"title": "Service", "value": service_name},
                {"title": "Metric", "value": metric_name},
                {"title": "Resource", "value": resource_info},
                {"title": "Reason", "value": new_state_reason},
                {"title": "Description", "value": alarm_description},
                {"title": "Time", "value": timestamp}
            ]
        }
    ]
}


    webhook_url = os.environ['TEAMS_WEBHOOK_URL']
    if priority == 'Emergency':
        webhook_url = os.environ['TEAMS_EMERGENCY_WEBHOOK_URL']
    elif priority == 'High':
        webhook_url = os.environ['TEAMS_HIGH_WEBHOOK_URL']

    response = http.request(
        "POST",
        webhook_url,
        body=json.dumps(card),
        headers={'Content-Type': 'application/json'}
    )

    return {
        'statusCode': 200,
        'body': 'Notification sent to Teams'
    }
```

## Budget alarm

- Tech stack: python (latest version)
- Env: TEAMS_WEBHOOK_URL (workflow)

```text
import json
import os
import re
import boto3
from datetime import datetime
import urllib.request
import urllib.error
 
ce_client = boto3.client('ce')
 
def lambda_handler(event, context):
    print("=== [START] Received Lambda Event ===")
    print(json.dumps(event, indent=2))
    print("=====================================")
    
    try:
        for record in event.get('Records', []):
            raw_message = record['Sns']['Message']
            
            # Khởi tạo các biến mặc định
            account_id = 'N/A'
            budget_name = 'N/A'
            alert_type = 'ACTUAL'
            threshold = '0'
            actual_amount = 0.0
            budgeted_amount = 0.0
            
            # --- KIỂM TRA ĐỊNH DẠNG MESSAGE VÀ BÓC TÁCH DỮ LIỆU ---
            if raw_message.strip().startswith('{'):
                # Trường hợp dữ liệu dạng JSON
                try:
                    sns_message = json.loads(raw_message)
                    # Thường JSON test không có accountId trực tiếp trong Message của AWS Budget,
                    # nên ta lấy thử từ arn của SNS record trước, nếu không có mới lấy mặc định
                    topic_arn = record.get('Sns', {}).get('TopicArn', '')
                    arn_parts = topic_arn.split(':')
                    if len(arn_parts) > 4:
                        account_id = arn_parts[4]
                        
                    budget_name = sns_message.get('budgetName', 'N/A')
                    alert_type = sns_message.get('alertType', 'ACTUAL')
                    threshold = str(sns_message.get('alertThreshold', '0')).replace('%', '').strip()
                    actual_amount = float(sns_message.get('actualAmount', 0))
                    budgeted_amount = float(sns_message.get('budgetedAmount', 0))
                except Exception:
                    pass
            else:
                # Trường hợp dữ liệu dạng Văn bản thật từ AWS Budgets
                account_match = re.search(r"AWS Account\s*(\d+)", raw_message)
                b_name_match = re.search(r"Budget Name:\s*(.*)", raw_message)
                b_limit_match = re.search(r"Budgeted Amount:\s*\$(.*)", raw_message)
                a_type_match = re.search(r"Alert Type:\s*(.*)", raw_message)
                a_threshold_match = re.search(r"Alert Threshold:\s*(.*)", raw_message)
                a_amount_match = re.search(r"ACTUAL Amount:\s*\$(.*)", raw_message)
                
                if account_match: account_id = account_match.group(1).strip()
                if b_name_match: budget_name = b_name_match.group(1).strip()
                if a_type_match: alert_type = a_type_match.group(1).strip()
                if a_threshold_match: threshold = a_threshold_match.group(1).strip()
                
                def clean_float(val_str):
                    return float(val_str.replace(',', '').strip()) if val_str else 0.0
                
                if b_limit_match: budgeted_amount = clean_float(b_limit_match.group(1))
                if a_amount_match: actual_amount = clean_float(a_amount_match.group(1))
 
            # Tính toán phần trăm sử dụng ngân sách
            percentage = round((actual_amount / budgeted_amount * 100), 1) if budgeted_amount > 0 else 0
            
            # Quyết định trạng thái dựa trên phần trăm thực tế sử dụng
            emoji = "🚨" if percentage >= 90 else "⚠️"
            status = "CRITICAL" if percentage >= 90 else "WARNING"
            color = "D32F2F" if percentage >= 90 else "FF9800"
            
            top_services = get_top_services()
            
            # TRUYỀN THÊM account_id VÀO ĐÂY
            teams_message = create_rich_teams_message(
                emoji, status, color, account_id, budget_name, alert_type, threshold,
                actual_amount, budgeted_amount, percentage, top_services
            )
            
            send_to_teams(teams_message)
            
    except Exception as e:
        print(f"Error: {str(e)}")
        raise
 
 
def get_top_services():
    try:
        end = datetime.utcnow().strftime('%Y-%m-%d')
        start = datetime.utcnow().replace(day=1).strftime('%Y-%m-%d')
        
        response = ce_client.get_cost_and_usage(
            TimePeriod={'Start': start, 'End': end},
            Granularity='MONTHLY',
            Metrics=['UnblendedCost'],
            GroupBy=[{'Type': 'DIMENSION', 'Key': 'SERVICE'}]
        )
        
        groups = response.get('ResultsByTime', [{}])[0].get('Groups', [])
        
        top = []
        for group in groups:
            service = group['Keys'][0]
            cost = float(group['Metrics']['UnblendedCost']['Amount'])
            top.append({"service": service[:60], "cost": round(cost, 2)})
        
        top.sort(key=lambda x: x['cost'], reverse=True)
        return top[:5]
        
    except Exception as e:
        print(f"❌ Cost Explorer Error: {str(e)}")
        return None
 
 
def create_rich_teams_message(
    emoji,
    status,
    color,
    account_id,
    budget_name,
    alert_type,
    threshold,
    actual,
    budgeted,
    percentage,
    top_services
):
    if top_services:
        top_text = "\n".join(
            [f"• {item['service']}: ${item['cost']:,.2f}" for item in top_services]
        )
    else:
        top_text = "• Unable to retrieve top services"

    now = datetime.now()
    alert_time = now.strftime("%Y/%m/%d %H:%M")
    period = now.strftime("%B %Y")

    return {
        "type": "AdaptiveCard",
        "$schema": "http://adaptivecards.io/schemas/adaptive-card.json",
        "version": "1.5",
        "body": [
            {
                "type": "TextBlock",
                "text": f"{emoji} AWS Budget Alert - {status}",
                "weight": "Bolder",
                "size": "Large",
                "color": "Attention" if status == "CRITICAL" else "Warning"
            },
            {
                "type": "TextBlock",
                "text": budget_name,
                "weight": "Bolder",
                "spacing": "Small"
            },
            {
                "type": "FactSet",
                "facts": [
                    {
                        "title": "AWS Account",
                        "value": account_id
                    },
                    {
                        "title": "Current Spend",
                        "value": f"${actual:,.2f} ({percentage}%)"
                    },
                    {
                        "title": "Budget Limit",
                        "value": f"${budgeted:,.2f}"
                    },
                    {
                        "title": "Threshold",
                        "value": threshold
                    },
                    {
                        "title": "Alert Type",
                        "value": alert_type
                    },
                    {
                        "title": "Period",
                        "value": period
                    },
                    {
                        "title": "Alert Time",
                        "value": alert_time
                    }
                ]
            },
            {
                "type": "TextBlock",
                "text": "Top 5 Cost Drivers",
                "weight": "Bolder",
                "spacing": "Medium"
            },
            {
                "type": "TextBlock",
                "text": top_text,
                "wrap": True
            },
            {
                "type": "TextBlock",
                "text": "Quick Links",
                "weight": "Bolder",
                "spacing": "Medium"
            },
            {
                "type": "TextBlock",
                "wrap": True,
                "text": "[AWS Budgets Console](https://console.aws.amazon.com/billing/home#/budgets)\n[AWS Cost Explorer](https://console.aws.amazon.com/cost-management/home#/cost-explorer)\n[AWS Billing Account](https://console.aws.amazon.com/billing/home#/account)"
            },
            {
                "type": "TextBlock",
                "text": "Recommended Actions",
                "weight": "Bolder",
                "spacing": "Medium"
            },
            {
                "type": "TextBlock",
                "wrap": True,
                "text": "• Check unusual costs in the last 24 hours\n• Review top cost-driving services/resources\n• Consider rightsizing or stopping non-production resources"
            }
        ]
    }
 
 
def send_to_teams(message):
    webhook_url = os.environ.get('TEAMS_WEBHOOK_URL')
    if not webhook_url:
        print("❌ Missing TEAMS_WEBHOOK_URL")
        return
    
    masked_url = webhook_url if len(webhook_url) <= 30 else f"{webhook_url[:20]}...{webhook_url[-10:]}"
    print(f"ℹ️ Sending request to Teams Webhook: {masked_url}")
    
    data = json.dumps(message).encode('utf-8')
    req = urllib.request.Request(
        webhook_url,
        data=data,
        headers={'Content-Type': 'application/json'},
        method='POST'
    )
    
    try:
        with urllib.request.urlopen(req, timeout=15) as resp:
            response_code = resp.getcode()
            response_body = resp.read().decode('utf-8')
            print(f"✅ Teams Response Success - Code: {response_code}")
            print(f"📄 Response Body: {response_body}")
            
    except urllib.error.HTTPError as e:
        error_body = e.read().decode('utf-8')
        print(f"❌ Teams HTTP Error - Code: {e.code}")
        print(f"📄 Error Response Body: {error_body}")
    except Exception as e:
        print(f"❌ Teams Connection Error: {str(e)}")
```
