### 시나리오 개요 
글로벌 온라인 쇼핑몰 스타트업에서 다국어 고객 문의를 실시간으로 처리하고, 긍정/부정 리뷰를 자동 분석하여 고객 만족도를 관리하는 시스템을 구축합니다.

```
고객 리뷰 등록 (S3) 
  ↓
EventBridge (이벤트 감지)
  ↓
SQS (메시지 큐잉)
  ↓
Step Functions (워크플로우 시작)
  ↓
1. Comprehend (언어 감지)
2. Translate (한국어 번역)
3. Comprehend (감정 분석 + 핵심 구문)
4. DynamoDB (결과 저장)
5. SNS (부정 리뷰 시 알림)
```

### 사전준비 
#### CloudShell 접속
1. AWS 콘솔에 로그인
2. 우측 상단의 CloudShell 아이콘 클릭
3. Region : US-EAST-1

```
# 현재 리전 확인
aws configure get region

# 리전이 us-east-1이 아니면 설정
aws configure set region us-east-1

# 계정 ID 확인 (나중에 사용)
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
echo "Account ID: $ACCOUNT_ID"

# IAM User Name 확인
MY_NAME=$(aws sts get-caller-identity --query 'Arn' --output text | cut -d '/' -f 2)
echo "My IAM Name: $MY_NAME"
```

### 1. S3 Bucket 생성 
- 고객 리뷰를 저장할 S3 버킷 생성
```
# 고유한 버킷 이름 설정 (본인의 숫자로 변경하세요)*
BUCKET_NAME="ecommerce-reviews-lab-$MY_NAME"

# S3 버킷 생성*
aws s3 mb s3://$BUCKET_NAME

# 버킷 생성 확인*
aws s3 ls | grep ecommerce-reviews

echo "✅ S3 버킷 생성 완료: $BUCKET_NAME"
```

### 2. SQS 큐 생성
- 리뷰 메시지를 순차적으로 처리하기 위한 큐 생성
```
# SQS 표준 큐 생성
QUEUE_URL=$(aws sqs create-queue \
  --queue-name ReviewProcessingQueue-$MY_NAME \
  --attributes VisibilityTimeout=300 \
  --query 'QueueUrl' \
  --output text)

echo "✅ SQS 큐 생성 완료"
echo "Queue URL: $QUEUE_URL"

# 큐 ARN 가져오기 (나중에 사용)
QUEUE_ARN=$(aws sqs get-queue-attributes \
  --queue-url $QUEUE_URL \
  --attribute-names QueueArn \
  --query 'Attributes.QueueArn' \
  --output text)

echo "Queue ARN: $QUEUE_ARN"
```

### 3. SNS 토픽 생성
- 부정 리뷰 감지 시 알림을 받을 SNS 토픽 생성
```
# SNS 토픽 생성
TOPIC_ARN=$(aws sns create-topic \
  --name NegativeReviewAlerts-$MY_NAME \
  --query 'TopicArn' \
  --output text)

echo "✅ SNS 토픽 생성 완료"
echo "Topic ARN: $TOPIC_ARN"

# 이메일 구독 (본인의 이메일로 변경하세요)# 주의: 이메일 확인 메일이 발송되니 반드시 확인해야 합니다!
aws sns subscribe \
  --topic-arn $TOPIC_ARN \
  --protocol email \
  --notification-endpoint YOUR-EMAIL@example.com

echo "⚠️  이메일로 구독 확인 메일이 발송되었습니다. 반드시 확인 버튼을 클릭하세요!"
```

### 4. DynamoDB 테이블 생성
```
# DynamoDB 테이블 생성
aws dynamodb create-table \
  --table-name ReviewAnalysis-$MY_NAME \
  --attribute-definitions \
    AttributeName=ReviewId,AttributeType=S \
    AttributeName=Timestamp,AttributeType=S \
  --key-schema \
    AttributeName=ReviewId,KeyType=HASH \
    AttributeName=Timestamp,KeyType=RANGE \
  --billing-mode PAY_PER_REQUEST

# 테이블 생성 대기 (약 10-20초 소요)
echo "⏳ DynamoDB 테이블 생성 중..."
aws dynamodb wait table-exists --table-name ReviewAnalysis-$MY_NAME

echo "✅ DynamoDB 테이블 생성 완료"

# 테이블 확인
aws dynamodb describe-table \
  --table-name ReviewAnalysis-$MY_NAME \
  --query 'Table.TableStatus' \
  --output text
```

### 5. IAM 역할 생성 (Step Functions용)
#### 신뢰 정책 파일 생성
```
cat > trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "states.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

echo "✅ 신뢰 정책 파일 생성 완료"
```

#### IAM 역할생성
```
# IAM 역할 생성
aws iam create-role \
  --role-name ReviewAnalysisStepFunctionsRole \
  --assume-role-policy-document file://trust-policy.json

echo "✅ IAM 역할 생성 완료"
```

#### 권한 정책 파일 생성 
```
cat > permissions-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "comprehend:DetectDominantLanguage",
        "comprehend:DetectSentiment",
        "comprehend:DetectKeyPhrases"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "translate:TranslateText"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "dynamodb:PutItem"
      ],
      "Resource": "arn:aws:dynamodb:us-east-1:$ACCOUNT_ID:table/ReviewAnalysis-$MY_NAME"
    },
    {
      "Effect": "Allow",
      "Action": [
        "sns:Publish"
      ],
      "Resource": "$TOPIC_ARN"
    },
    {
      "Effect": "Allow",
      "Action": [
        "sqs:ReceiveMessage",
        "sqs:DeleteMessage",
        "sqs:GetQueueAttributes"
      ],
      "Resource": "$QUEUE_ARN"
    }
  ]
}
EOF

echo "✅ 권한 정책 파일 생성 완료"
```

#### 정책을 역할에 연결
```
# 인라인 정책 연결
aws iam put-role-policy \
  --role-name ReviewAnalysisStepFunctionsRole \
  --policy-name ReviewAnalysisPermissions \
  --policy-document file://permissions-policy.json

echo "✅ 권한 정책 연결 완료"

# 역할 ARN 가져오기
ROLE_ARN=$(aws iam get-role \
  --role-name ReviewAnalysisStepFunctionsRole \
  --query 'Role.Arn' \
  --output text)

echo "Role ARN: $ROLE_ARN"

# 역할이 생성되고 전파될 때까지 대기
echo "⏳ IAM 역할 전파 대기 중... (10초)"
sleep 10
```







