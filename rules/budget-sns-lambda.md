# Setup Budget SNS Lambda Function

## Yêu cầu

1. Set role:
   + Cloudwatch Log
   + Cost Explorer
      + Allow: ce:GetCostAndUsage
      + Allow: ce:GetCostAndUsageWithResources

```text
import json
import os
import re
import boto3
from datetime import datetime, timedelta
import urllib.request
import urllib.error

ce_client = boto3.client('ce')


def lambda_handler(event, context):
    print("=== [START] Received Lambda Event ===")
    print(json.dumps(event, indent=2, default=str))
    print("=====================================")

    try:
        for record in event.get('Records', []):
            raw_message = record['Sns']['Message']

            account_id = 'N/A'
            budget_name = 'N/A'
            alert_type = 'ACTUAL'
            threshold = '0'
            actual_amount = 0.0
            budgeted_amount = 0.0

            topic_arn = record.get('Sns', {}).get('TopicArn', '')
            arn_parts = topic_arn.split(':')
            if len(arn_parts) > 4:
                account_id = arn_parts[4]

            # Parse message từ AWS Budget
            if raw_message.strip().startswith('{'):
                try:
                    sns_message = json.loads(raw_message)

                    budget_name = sns_message.get('budgetName', 'N/A')
                    alert_type = sns_message.get('alertType', 'ACTUAL')
                    threshold = str(sns_message.get('alertThreshold', '0')).replace('%', '').strip()

                    actual_amount = clean_float(
                        sns_message.get('actualAmount')
                        or sns_message.get('currentSpend')
                        or sns_message.get('forecastedAmount')
                        or 0
                    )

                    budgeted_amount = clean_float(
                        sns_message.get('budgetedAmount')
                        or sns_message.get('budgetLimit')
                        or 0
                    )

                except Exception as e:
                    print(f"⚠️ JSON parse error: {str(e)}")

            else:
                account_match = re.search(r"AWS Account(?: ID)?:\s*(\d+)", raw_message, re.IGNORECASE)
                b_name_match = re.search(r"Budget Name:\s*(.*)", raw_message, re.IGNORECASE)
                b_limit_match = re.search(
                    r"(?:Budgeted Amount|Budget Limit):\s*\$?([0-9,]+(?:\.[0-9]+)?)",
                    raw_message,
                    re.IGNORECASE
                )
                a_type_match = re.search(r"Alert Type:\s*(.*)", raw_message, re.IGNORECASE)
                a_threshold_match = re.search(r"(?:Alert Threshold|Threshold):\s*(.*)", raw_message, re.IGNORECASE)

                amount_match = re.search(
                    r"(?:Current Spend|ACTUAL Amount|Actual Amount|FORECASTED Amount|Forecasted Amount):\s*\$?([0-9,]+(?:\.[0-9]+)?)",
                    raw_message,
                    re.IGNORECASE
                )

                if account_match:
                    account_id = account_match.group(1).strip()

                if b_name_match:
                    budget_name = b_name_match.group(1).strip()

                if a_type_match:
                    alert_type = a_type_match.group(1).strip()

                if a_threshold_match:
                    threshold = a_threshold_match.group(1).replace('%', '').strip()

                if b_limit_match:
                    budgeted_amount = clean_float(b_limit_match.group(1))

                if amount_match:
                    actual_amount = clean_float(amount_match.group(1))

            current_spend = get_month_to_date_spend()

            print("===== CURRENT SPEND TEST =====")
            print(f"Parsed amount from Budget message: ${actual_amount}")
            print(f"Current spend from Cost Explorer: ${current_spend}")
            print("==============================")

            if current_spend is not None:
                actual_amount = current_spend

            # Tính phần trăm theo current spend thật
            percentage = round((actual_amount / budgeted_amount * 100), 1) if budgeted_amount > 0 else 0

            emoji = "🚨" if percentage >= 90 else "⚠️"
            status = "CRITICAL" if percentage >= 90 else "WARNING"
            color = "D32F2F" if percentage >= 90 else "FF9800"

            top_services = get_top_services()

            teams_message = create_rich_teams_message(
                emoji,
                status,
                color,
                account_id,
                budget_name,
                alert_type,
                threshold,
                actual_amount,
                budgeted_amount,
                percentage,
                top_services
            )

            send_to_teams(teams_message)

        return {
            "statusCode": 200,
            "body": json.dumps({"message": "Processed successfully"})
        }

    except Exception as e:
        print(f"❌ Lambda Error: {str(e)}")
        raise


def clean_float(val):
    try:
        if val is None:
            return 0.0

        if isinstance(val, (int, float)):
            return float(val)

        return float(
            str(val)
            .replace(',', '')
            .replace('$', '')
            .replace('%', '')
            .strip()
        )

    except Exception:
        return 0.0


def get_month_to_date_spend():
    try:
        now = datetime.utcnow()

        start = now.replace(day=1).strftime('%Y-%m-%d')

        # Cost Explorer End date là exclusive,
        end = (now + timedelta(days=1)).strftime('%Y-%m-%d')

        print(f"Cost Explorer TimePeriod Start: {start}")
        print(f"Cost Explorer TimePeriod End: {end}")

        response = ce_client.get_cost_and_usage(
            TimePeriod={
                'Start': start,
                'End': end
            },
            Granularity='MONTHLY',
            Metrics=[
                'UnblendedCost'
            ]
        )

        print("Cost Explorer raw response:")
        print(json.dumps(response, indent=2, default=str))

        amount = response['ResultsByTime'][0]['Total']['UnblendedCost']['Amount']
        return round(float(amount), 2)

    except Exception as e:
        print(f"❌ Get Current Spend Error: {str(e)}")
        return None


def get_top_services():
    try:
        now = datetime.utcnow()

        start = now.replace(day=1).strftime('%Y-%m-%d')

        end = (now + timedelta(days=1)).strftime('%Y-%m-%d')

        response = ce_client.get_cost_and_usage(
            TimePeriod={
                'Start': start,
                'End': end
            },
            Granularity='MONTHLY',
            Metrics=[
                'UnblendedCost'
            ],
            GroupBy=[
                {
                    'Type': 'DIMENSION',
                    'Key': 'SERVICE'
                }
            ]
        )

        groups = response.get('ResultsByTime', [{}])[0].get('Groups', [])

        top = []
        for group in groups:
            service = group['Keys'][0]
            cost = float(group['Metrics']['UnblendedCost']['Amount'])

            if cost > 0:
                top.append({
                    "service": service[:60],
                    "cost": round(cost, 2)
                })

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
    now = datetime.utcnow() + timedelta(hours=9)
    alert_time = now.strftime("%Y-%m-%d %H:%M")
    period = now.strftime("%B %Y")

    top_section = []

    if top_services:
        top_section.append("**Top 5 Cost Drivers**")
        for item in top_services:
            top_section.append(f"• **{item['service']}**: **${item['cost']:,.2f}**")
        top_text = "\n\n".join(top_section)
    else:
        top_text = "**Top 5 Cost Drivers**\n\n• Chưa lấy được dữ liệu"

    return {
        "@type": "MessageCard",
        "@context": "http://schema.org/extensions",
        "themeColor": color,
        "summary": f"AWS Budget Alert - {status}",
        "sections": [
            {
                "activityTitle": f"{emoji} **AWS BUDGET ALERT — {status}**",
                "activitySubtitle": f"**{budget_name}**",
                "facts": [
                    {
                        "name": "Account ID:",
                        "value": f"**{account_id}**"
                    },
                    {
                        "name": "Current Spend:",
                        "value": f"**${actual:,.2f}** ({percentage}% of budget)"
                    },
                    {
                        "name": "Budget Limit:",
                        "value": f"**${budgeted:,.2f}**"
                    },
                    {
                        "name": "Threshold:",
                        "value": f"**{threshold}%**"
                    },
                    {
                        "name": "Alert Type:",
                        "value": alert_type
                    },
                    {
                        "name": "Period:",
                        "value": period
                    },
                    {
                        "name": "Alert Time:",
                        "value": alert_time
                    }
                ]
            },
            {
                "text": top_text
            },
            {
                "text": "**Quick Links**\n\n"
                        "[AWS Budgets Console](https://console.aws.amazon.com/billing/home#/budgets)  \n\n"
                        "[Cost Explorer](https://console.aws.amazon.com/cost-management/home#/cost-explorer)  \n\n"
                        "[Billing Account](https://console.aws.amazon.com/billing/home#/account)"
            },
            {
                "text": "**Action Required**\n\n"
                        "• Kiểm tra chi phí bất thường trong 24h qua  \n"
                        "• Review top services/resources đang tăng mạnh  \n"
                        "• Kiểm tra Marketplace, Region, Access Key nếu chi phí tăng bất thường  \n"
                        "• Consider rightsizing hoặc stop non-production resources"
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
        headers={
            'Content-Type': 'application/json'
        },
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
        raise

    except Exception as e:
        print(f"❌ Teams Connection Error: {str(e)}")
        raise
```
