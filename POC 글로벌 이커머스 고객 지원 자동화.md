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

### 6. Step Function 상태 머신 생성
```
cat > state-machine-definition.json << EOF
{
  "Comment": "고객 리뷰 분석 워크플로우",
  "StartAt": "DetectLanguage",
  "States": {
    "DetectLanguage": {
      "Type": "Task",
      "Resource": "arn:aws:states:::aws-sdk:comprehend:detectDominantLanguage",
      "Parameters": {
        "Text.$": "$.reviewText"
      },
      "ResultPath": "$.languageResult",
      "Next": "CheckLanguage"
    },
    "CheckLanguage": {
      "Type": "Choice",
      "Choices": [
        {
          "Variable": "$.languageResult.Languages[0].LanguageCode",
          "StringEquals": "ko",
          "Next": "AnalyzeSentiment"
        }
      ],
      "Default": "TranslateToKorean"
    },
    "TranslateToKorean": {
      "Type": "Task",
      "Resource": "arn:aws:states:::aws-sdk:translate:translateText",
      "Parameters": {
        "Text.$": "$.reviewText",
        "SourceLanguageCode.$": "$.languageResult.Languages[0].LanguageCode",
        "TargetLanguageCode": "ko"
      },
      "ResultPath": "$.translationResult",
      "Next": "AnalyzeSentiment"
    },
    "AnalyzeSentiment": {
      "Type": "Task",
      "Resource": "arn:aws:states:::aws-sdk:comprehend:detectSentiment",
      "Parameters": {
        "Text.$": "States.Format('{}', $.reviewText)",
        "LanguageCode.$": "$.languageResult.Languages[0].LanguageCode"
      },
      "ResultPath": "$.sentimentResult",
      "Next": "DetectKeyPhrases"
    },
    "DetectKeyPhrases": {
      "Type": "Task",
      "Resource": "arn:aws:states:::aws-sdk:comprehend:detectKeyPhrases",
      "Parameters": {
        "Text.$": "States.Format('{}', $.reviewText)",
        "LanguageCode.$": "$.languageResult.Languages[0].LanguageCode"
      },
      "ResultPath": "$.keyPhrasesResult",
      "Next": "SaveToDynamoDB"
    },
    "SaveToDynamoDB": {
      "Type": "Task",
      "Resource": "arn:aws:states:::dynamodb:putItem",
      "Parameters": {
        "TableName": "ReviewAnalysis-"$My_name"",
        "Item": {
          "ReviewId": {
            "S.$": "$.reviewId"
          },
          "Timestamp": {
            "S.$": "$.State.EnteredTime"
          },
          "OriginalText": {
            "S.$": "$.reviewText"
          },
          "DetectedLanguage": {
            "S.$": "$.languageResult.Languages[0].LanguageCode"
          },
          "Sentiment": {
            "S.$": "$.sentimentResult.Sentiment"
          },
          "PositiveScore": {
            "N.$": "States.Format('{}', $.sentimentResult.SentimentScore.Positive)"
          },
          "NegativeScore": {
            "N.$": "States.Format('{}', $.sentimentResult.SentimentScore.Negative)"
          }
        }
      },
      "ResultPath": "$.dynamoResult",
      "Next": "CheckIfNegative"
    },
    "CheckIfNegative": {
      "Type": "Choice",
      "Choices": [
        {
          "Variable": "$.sentimentResult.Sentiment",
          "StringEquals": "NEGATIVE",
          "Next": "SendSNSAlert"
        }
      ],
      "Default": "Success"
    },
    "SendSNSAlert": {
      "Type": "Task",
      "Resource": "arn:aws:states:::sns:publish",
      "Parameters": {
        "TopicArn": "$TOPIC_ARN",
        "Subject": "⚠️ 부정 리뷰 감지 알림",
        "Message.$": "States.Format('부정 리뷰가 감지되었습니다.\n\n리뷰 ID: {}\n내용: {}\n부정 점수: {}\n언어: {}', $.reviewId, $.reviewText, $.sentimentResult.SentimentScore.Negative, $.languageResult.Languages[0].LanguageCode)"
      },
      "Next": "Success"
    },
    "Success": {
      "Type": "Succeed"
    }
  }
}
EOF

echo "✅ 상태 머신 정의 파일 생성 완료"
```

#### 6-2 상태 머신 생성
```
# Step Functions 상태 머신 생성
STATE_MACHINE_ARN=$(aws stepfunctions create-state-machine \
  --name ReviewAnalysisWorkflow-$MY_NAME \
  --definition file://state-machine-definition.json \
  --role-arn $ROLE_ARN \
  --query 'stateMachineArn' \
  --output text)

echo "✅ Step Functions 상태 머신 생성 완료"
echo "State Machine ARN: $STATE_MACHINE_ARN"
```

### 7. EventBridge 규칙 생성
- S3 리뷰가 업로드되면 자동으로 Step Function 실행하도록 설정

#### 7-1 S3 버킷에 EventBridge 알림 활성화 
```
# S3 버킷에 EventBridge 알림 활성화
aws s3api put-bucket-notification-configuration \
  --bucket $BUCKET_NAME \
  --notification-configuration '{
    "EventBridgeConfiguration": {}
  }'

echo "✅ S3 EventBridge 알림 활성화 완료"
```

#### 7-2 EventBridge 규칙에 대한 IAM 역할 생성
```
 EventBridge 신뢰 정책 생성
cat > eventbridge-trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "events.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

# EventBridge 역할 생성
aws iam create-role \
  --role-name EventBridgeToSQSRole \
  --assume-role-policy-document file://eventbridge-trust-policy.json

# SQS 전송 권한 정책 생성
cat > eventbridge-sqs-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "sqs:SendMessage"
      ],
      "Resource": "$QUEUE_ARN"
    }
  ]
}
EOF

# 정책 연결
aws iam put-role-policy \
  --role-name EventBridgeToSQSRole \
  --policy-name SendToSQS \
  --policy-document file://eventbridge-sqs-policy.json

# 역할 ARN 가져오기
EVENTBRIDGE_ROLE_ARN=$(aws iam get-role \
  --role-name EventBridgeToSQSRole \
  --query 'Role.Arn' \
  --output text)

echo "✅ EventBridge IAM 역할 생성 완료"
sleep 10
```

#### 7-3 EventBridge 규칙 생성
```
# EventBridge 규칙 생성 (S3 객체 생성 이벤트)
aws events put-rule \
  --name S3ReviewUploadRule-$MY_NAME \
  --event-pattern "{
    \"source\": [\"aws.s3\"],
    \"detail-type\": [\"Object Created\"],
    \"detail\": {
      \"bucket\": {
        \"name\": [\"$BUCKET_NAME\"]
      },
      \"object\": {
        \"key\": [{\"prefix\": \"reviews/\"}]
      }
    }
  }"

echo "✅ EventBridge 규칙 생성 완료"
```

#### 7-4 SQS를 타겟으로 추가 
```
# SQS를 EventBridge 타겟으로 추가
aws events put-targets \
  --rule S3ReviewUploadRule \
  --targets "Id"="1","Arn"="$QUEUE_ARN","RoleArn"="$EVENTBRIDGE_ROLE_ARN"

echo "✅ EventBridge 타겟 설정 완료"
```

### 8. 테스트 데이터 wnsql
- 다양한 언어의 고객 리뷰 샘플을 생성

#### 8-1 긍정 리뷰(한국어)
```
cat > review-positive-ko.json << 'EOF'
{
  "reviewId": "review-001",
  "reviewText": "이 제품 정말 만족해요! 배송도 빠르고 품질도 훌륭합니다. 강력 추천합니다!"
}
EOF

echo "✅ 긍정 리뷰 (한국어) 생성 완료"
```

#### 8-2 부정 리뷰(한국어)
```
cat > review-negative-ko.json << 'EOF'
{
  "reviewId": "review-002",
  "reviewText": "배송이 너무 늦어서 실망했어요. 제품 포장도 허술하고 품질이 기대 이하입니다."
}
EOF

echo "✅ 부정 리뷰 (한국어) 생성 완료"
```

#### 8-3 긍정 리뷰(영어)
```
cat > review-positive-en.json << 'EOF'
{
  "reviewId": "review-003",
  "reviewText": "This product is amazing! Fast delivery and great quality. Highly recommended!"
}
EOF

echo "✅ 긍정 리뷰 (영어) 생성 완료"
```

#### 8-4 부정 리뷰(일본어)
```
cat > review-negative-ja.json << 'EOF'
{
  "reviewId": "review-004",
  "reviewText": "配送が遅すぎます。商品の品質も期待外れでした。残念です。"
}
EOF

echo "✅ 부정 리뷰 (일본어) 생성 완료"
```

#### 8-5 긍정 리뷰(중국어)
```
cat > review-positive-zh.json << 'EOF'
{
  "reviewId": "review-005",
  "reviewText": "产品质量非常好！物流很快，包装也很精美。五星好评！"
}
EOF

echo "✅ 긍정 리뷰 (중국어) 생성 완료"
```

### 9. 수동 테스트(Step Function 직접 실행)
- S3 업로드 전에 먼저 Step Function을 수동으로 실행

#### 9-1 긍정 리뷰 테스트
```
# 긍정 리뷰로 Step Functions 실행
EXECUTION_ARN_1=$(aws stepfunctions start-execution \
  --state-machine-arn $STATE_MACHINE_ARN \
  --name execution-positive-ko-$(date +%s) \
  --input file://review-positive-ko.json \
  --query 'executionArn' \
  --output text)

echo "✅ 긍정 리뷰 실행 시작"
echo "Execution ARN: $EXECUTION_ARN_1"

# 실행 상태 확인 (약 10초 후)
echo "⏳ 10초 대기 중..."
sleep 10

aws stepfunctions describe-execution \
  --execution-arn $EXECUTION_ARN_1 \
  --query 'status' \
  --output text
```

#### 9-2 부정 리뷰 테스트(SNS 발송)
```
# 부정 리뷰로 Step Functions 실행
EXECUTION_ARN_2=$(aws stepfunctions start-execution \
  --state-machine-arn $STATE_MACHINE_ARN \
  --name execution-negative-ko-$(date +%s) \
  --input file://review-negative-ko.json \
  --query 'executionArn' \
  --output text)

echo "✅ 부정 리뷰 실행 시작 (SNS 알림이 발송됩니다)"
echo "Execution ARN: $EXECUTION_ARN_2"

# 실행 상태 확인
echo "⏳ 10초 대기 중..."
sleep 10

aws stepfunctions describe-execution \
  --execution-arn $EXECUTION_ARN_2 \
  --query 'status' \
  --output text

echo "⚠️  이메일로 부정 리뷰 알림이 발송되었는지 확인하세요!"
```

#### 9-3 영어 리뷰 테스트
```
# 영어 리뷰로 Step Functions 실행
EXECUTION_ARN_3=$(aws stepfunctions start-execution \
  --state-machine-arn $STATE_MACHINE_ARN \
  --name execution-positive-en-$(date +%s) \
  --input file://review-positive-en.json \
  --query 'executionArn' \
  --output text)

echo "✅ 영어 리뷰 실행 시작"
echo "Execution ARN: $EXECUTION_ARN_3"

sleep 10

aws stepfunctions describe-execution \
  --execution-arn $EXECUTION_ARN_3 \
  --query 'status' \
  --output text
```

### 10. S3 업로드를 통한 자동 실행 테스트
- S3에 리뷰를 업로드하면 자동으로 분석 시작

#### 10-1 S3에 리뷰 업로드
```
# S3에 리뷰 파일들 업로드
aws s3 cp review-positive-ko.json s3://$BUCKET_NAME/reviews/
aws s3 cp review-negative-ko.json s3://$BUCKET_NAME/reviews/
aws s3 cp review-positive-en.json s3://$BUCKET_NAME/reviews/
aws s3 cp review-negative-ja.json s3://$BUCKET_NAME/reviews/
aws s3 cp review-positive-zh.json s3://$BUCKET_NAME/reviews/

echo "✅ 모든 리뷰를 S3에 업로드 완료"
echo "⏳ EventBridge → SQS → Step Functions 자동 실행 대기 중..."
```

#### 10-2 SQS 메시지 확인
```
# SQS 큐에 메시지가 들어왔는지 확인
sleep 5

aws sqs get-queue-attributes \
  --queue-url $QUEUE_URL \
  --attribute-names ApproximateNumberOfMessages

echo "위 숫자가 0보다 크면 메시지가 큐에 들어온 것입니다."
```

#### 10-3 SQS 메시지 수동 처리 스크립트 생성
- EventBridge가 SQS로 메시지를 보내면, 이를 읽어서 Step Function을 생성 하는 스크립트 만든다.

```
cat > process-queue.sh << 'EOF'
#!/bin/bash

# 환경 변수 확인
if [ -z "$QUEUE_URL" ] || [ -z "$STATE_MACHINE_ARN" ]; then
  echo "❌ 환경 변수가 설정되지 않았습니다."
  echo "다음 명령어를 먼저 실행하세요:"
  echo "export QUEUE_URL=<your-queue-url>"
  echo "export STATE_MACHINE_ARN=<your-state-machine-arn>"
  exit 1
fi

echo "🔄 SQS 메시지 처리 시작..."

# 메시지 수신
MESSAGES=$(aws sqs receive-message \
  --queue-url $QUEUE_URL \
  --max-number-of-messages 10 \
  --wait-time-seconds 5)

# 메시지가 있는지 확인
if [ -z "$(echo $MESSAGES | jq -r '.Messages')" ] || [ "$(echo $MESSAGES | jq -r '.Messages')" = "null" ]; then
  echo "📭 처리할 메시지가 없습니다."
  exit 0
fi

# 각 메시지 처리
echo $MESSAGES | jq -c '.Messages[]' | while read -r message; do
  RECEIPT_HANDLE=$(echo $message | jq -r '.ReceiptHandle')
  BODY=$(echo $message | jq -r '.Body')
  
  # S3 이벤트에서 파일 정보 추출
  BUCKET=$(echo $BODY | jq -r '.detail.bucket.name')
  KEY=$(echo $BODY | jq -r '.detail.object.key')
  
  echo "📄 처리 중: s3://$BUCKET/$KEY"
  
  # S3에서 리뷰 파일 다운로드
  aws s3 cp s3://$BUCKET/$KEY /tmp/review.json
  
  # Step Functions 실행
  EXECUTION_NAME="execution-$(basename $KEY .json)-$(date +%s)"
  
  aws stepfunctions start-execution \
    --state-machine-arn $STATE_MACHINE_ARN \
    --name $EXECUTION_NAME \
    --input file:///tmp/review.json
  
  echo "✅ Step Functions 실행: $EXECUTION_NAME"
  
  # 메시지 삭제
  aws sqs delete-message \
    --queue-url $QUEUE_URL \
    --receipt-handle $RECEIPT_HANDLE
  
  echo "🗑️  메시지 삭제 완료"
  echo "---"
done

echo "✅ 모든 메시지 처리 완료"
EOF

chmod +x process-queue.sh

echo "✅ SQS 메시지 처리 스크립트 생성 완료"
```

#### 10-4 스크립트 실행
```
# 환경 변수 설정 (위에서 생성한 값 사용)
export QUEUE_URL=$QUEUE_URL
export STATE_MACHINE_ARN=$STATE_MACHINE_ARN

# 스크립트 실행
./process-queue.sh

echo "⏳ Step Functions 실행 완료까지 약 20초 대기 중..."
sleep 20
```

### 11. 결과 확인
#### 11-1 DynamoDB 데이터 조회
```
# DynamoDB 테이블의 모든 데이터 조회
echo "=== DynamoDB 저장된 리뷰 분석 결과 ==="
aws dynamodb scan --table-name ReviewAnalysis-$MY_NAME --output table

# 간단한 형식으로 조회
aws dynamodb scan --table-name ReviewAnalysis-$MY_NAME \
  --query 'Items[*].[ReviewId.S, Sentiment.S, PositiveScore.N, NegativeScore.N, DetectedLanguage.S]' \
  --output table
```

#### 11-2 부정 리뷰만 필터링
```
# 부정 리뷰만 조회
echo "=== 부정 리뷰 목록 ==="
aws dynamodb scan --table-name ReviewAnalysis-$MY_NAME \
  --filter-expression "Sentiment = :sentiment" \
  --expression-attribute-values '{":sentiment":{"S":"NEGATIVE"}}' \
  --query 'Items[*].[ReviewId.S, OriginalText.S, NegativeScore.N]' \
  --output table
```

#### 11-3 Step Functions 실행 이력 확인
```
# 최근 실행 이력 조회
echo "=== Step Functions 실행 이력 ==="
aws stepfunctions list-executions \
  --state-machine-arn $STATE_MACHINE_ARN \
  --max-results 10 \
  --query 'executions[*].[name, status, startDate]' \
  --output table
```

#### 11-4 특정 실행의 상세 결과 확인
```
# 가장 최근 실행의 상세 정보 확인
LATEST_EXECUTION=$(aws stepfunctions list-executions \
  --state-machine-arn $STATE_MACHINE_ARN \
  --max-results 1 \
  --query 'executions[0].executionArn' \
  --output text)

echo "=== 최근 실행 상세 정보 ==="
aws stepfunctions describe-execution \
  --execution-arn $LATEST_EXECUTION

# 실행 히스토리 확인 (각 단계별 실행 과정)
echo "=== 실행 히스토리 ==="
aws stepfunctions get-execution-history \
  --execution-arn $LATEST_EXECUTION \
  --query 'events[*].[timestamp, type]' \
  --output table
```

### 12. 통계 조회 스크립트
```
at > show-statistics.sh << 'EOF'
#!/bin/bash

echo "======================================"
echo "    리뷰 분석 통계 대시보드"
echo "======================================"
echo ""

# 전체 리뷰 수
TOTAL_COUNT=$(aws dynamodb scan --table-name ReviewAnalysis-$MY_NAME --select COUNT --query 'Count' --output text)
echo "📊 전체 분석된 리뷰 수: $TOTAL_COUNT"
echo ""

# 감정별 통계
echo "😊 감정 분석 결과:"
POSITIVE_COUNT=$(aws dynamodb scan --table-name ReviewAnalysis-$MY_NAME \
  --filter-expression "Sentiment = :sentiment" \
  --expression-attribute-values '{":sentiment":{"S":"POSITIVE"}}' \
  --select COUNT --query 'Count' --output text)
echo "  - 긍정 리뷰: $POSITIVE_COUNT개"

NEGATIVE_COUNT=$(aws dynamodb scan --table-name ReviewAnalysis-$MY_NAME \
  --filter-expression "Sentiment = :sentiment" \
  --expression-attribute-values '{":sentiment":{"S":"NEGATIVE"}}' \
  --select COUNT --query 'Count' --output text)
echo "  - 부정 리뷰: $NEGATIVE_COUNT개"

NEUTRAL_COUNT=$(aws dynamodb scan --table-name ReviewAnalysis-$MY_NAME \
  --filter-expression "Sentiment = :sentiment" \
  --expression-attribute-values '{":sentiment":{"S":"NEUTRAL"}}' \
  --select COUNT --query 'Count' --output text)
echo "  - 중립 리뷰: $NEUTRAL_COUNT개"

MIXED_COUNT=$(aws dynamodb scan --table-name ReviewAnalysis-$MY_NAME \
  --filter-expression "Sentiment = :sentiment" \
  --expression-attribute-values '{":sentiment":{"S":"MIXED"}}' \
  --select COUNT --query 'Count' --output text)
echo "  - 혼합 리뷰: $MIXED_COUNT개"
echo ""

# 언어별 통계
echo "🌍 언어별 분포:"
aws dynamodb scan --table-name ReviewAnalysis-$MY_NAME \
  --projection-expression "DetectedLanguage" | \
  jq -r '.Items[].DetectedLanguage.S' | \
  sort | uniq -c | \
  awk '{
    lang=$2
    count=$1
    if (lang=="ko") lang="한국어"
    else if (lang=="en") lang="영어"
    else if (lang=="ja") lang="일본어"
    else if (lang=="zh") lang="중국어"
    printf "  - %s: %d개\n", lang, count
  }'
echo ""

# Step Functions 실행 통계
echo "⚙️  Step Functions 실행 상태:"
aws stepfunctions list-executions \
  --state-machine-arn $STATE_MACHINE_ARN \
  --max-results 50 | \
  jq -r '.executions[].status' | \
  sort | uniq -c | \
  awk '{printf "  - %s: %d개\n", $2, $1}'
echo ""

echo "======================================"
echo "✅ 통계 조회 완료"
echo "======================================"
EOF

chmod +x show-statistics.sh

echo "✅ 통계 조회 스크립트 생성 완료"
```

#### 12-2 통계 실행
```
# 통계 스크립트 실행
./show-statistics.sh
```

<img width="332" height="403" alt="image" src="https://github.com/user-attachments/assets/a1ca5a86-381f-4434-8150-583c8ddbce92" />






