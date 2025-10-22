# AWS Comprehend 실습 
## 사전 준비사항
```
#Comprehend 배치 작업용 역할 생성 

cat > comprehend-trust-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "comprehend.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

aws iam create-role \
    --role-name ComprehendRole \
    --assume-role-policy-document file://comprehend-trust-policy.json

```

### 1. AWS 계정 및 CLI 계정
```
# AWS CLI 설치 확인
aws --version // 버전확인
# AWS 자격 증명 설정
aws configure
```

### 2. 필요 권한 
필요 IAM 정책 : ComprehendReadOnly, ComprehendFullAccess

### 3. 실습 디렉터리 생성
```
mkdir comprehend-practice
cd comprehend-practice
```

## 실습 1 : 감정 분석
### 1. 예시 리뷰 데이터 준비 
```

# 리뷰 데이터 파일 생성
cat > customer_reviews.txt << 'EOF'
이 제품 정말 만족해요! 배송도 빠르고 품질도 훌륭합니다.
배송이 너무 늦어서 실망했어요. 제품은 괜찮은데 서비스가 아쉽네요.
가격 대비 품질이 좋은 것 같아요. 추천합니다!
포장이 너무 허술해서 제품이 손상되어 왔습니다.
EOF
```

### 2. 감정 분석 실행
```

# 첫 번째 리뷰 감정 분석
aws comprehend detect-sentiment \
    --text "이 제품 정말 만족해요! 배송도 빠르고 품질도 훌륭합니다." \
    --language-code ko

# 두 번째 리뷰 감정 분석
aws comprehend detect-sentiment \
    --text "배송이 너무 늦어서 실망했어요. 제품은 괜찮은데 서비스가 아쉽네요." \
    --language-code ko
```

### 3. 결과 해석
```

{
    "Sentiment": "POSITIVE",
    "SentimentScore": {
        "Positive": 0.9856,
        "Negative": 0.0052,
        "Neutral": 0.0089,
        "Mixed": 0.0003
    }
}
```

## 실습 2 : 언어 감지
### 1. 다국어 리뷰 데이터 준비 
```

cat > multilingual_reviews.txt << 'EOF'
This product is amazing! Fast delivery and great quality.
Este producto es increíble. Lo recomiendo mucho.
この商品は本当に素晴らしいです。配送も早くて満足しています。
이 제품 정말 좋아요! 다음에도 구매할 예정입니다.
EOF
```

### 2. 언어 감지 실행
```

# 영어 리뷰
aws comprehend detect-dominant-language \
    --text "This product is amazing! Fast delivery and great quality."

# 스페인어 리뷰
aws comprehend detect-dominant-language \
    --text "Este producto es increíble. Lo recomiendo mucho."

# 일본어 리뷰
aws comprehend detect-dominant-language \
    --text "この商품は本当に素晴らしいです。配送も早くて満足しています。"

# 한국어 리뷰
aws comprehend detect-dominant-language \
    --text "이 제품 정말 좋아요! 다음에도 구매할 예정입니다."
```

### 3. 결과 해석
```

{
    "Languages": [
        {
            "LanguageCode": "ko",
            "Score": 0.9998
        }
    ]
}
```

## 실습 3 : 개체명 인식
### 1. 브랜드/제품명이 포함된 리뷰 데이터 준비
```

cat > entity_reviews.txt << 'EOF'
삼성 갤럭시 S23을 서울에서 구매했는데 정말 만족합니다.
애플 아이폰 15 프로를 김철수 님께서 추천해주셨어요.
2024년 1월 15일에 주문한 LG 세탁기가 오늘 도착했습니다.
Netflix 구독료 15,000원이 매월 자동 결제되고 있어요.
EOF
```

### 2. 개체명 인식 실행
```

# 첫 번째 리뷰 개체명 인식
aws comprehend detect-entities \
    --text "삼성 갤럭시 S23을 서울에서 구매했는데 정말 만족합니다." \
    --language-code ko

# 두 번째 리뷰 개체명 인식
aws comprehend detect-entities \
    --text "애플 아이폰 15 프로를 김철수 님께서 추천해주셨어요." \
    --language-code ko
```

### 3. 결과 해석 
```

{
    "Entities": [
        {
            "Score": 0.9999,
            "Type": "ORGANIZATION",
            "Text": "삼성",
            "BeginOffset": 0,
            "EndOffset": 2
        },
        {
            "Score": 0.8756,
            "Type": "OTHER",
            "Text": "갤럭시 S23",
            "BeginOffset": 3,
            "EndOffset": 9
        }
    ]
}

```

## 실습 4 : 핵심 구문 추출
### 1. 핵심 구문 추출 실행
```

# 상세한 리뷰에서 핵심 구문 추출
aws comprehend detect-key-phrases \
    --text "이 무선 이어폰의 음질이 정말 뛰어나고 배터리 수명도 길어서 만족합니다. 노이즈 캔슬링 기능도 훌륭하고 착용감이 편안해요." \
    --language-code ko

# 부정적 리뷰에서 핵심 구문 추출
aws comprehend detect-key-phrases \
    --text "배송 포장이 너무 허술해서 제품이 파손되어 도착했습니다. 고객 서비스 응답도 늦고 환불 처리가 복잡해요." \
    --language-code ko
```

### 2. 결과 해석
```

{
    "KeyPhrases": [
        {
            "Score": 0.9998,
            "Text": "무선 이어폰",
            "BeginOffset": 2,
            "EndOffset": 7
        },
        {
            "Score": 0.9995,
            "Text": "음질",
            "BeginOffset": 9,
            "EndOffset": 11
        },
        {
            "Score": 0.9989,
            "Text": "배터리 수명",
            "BeginOffset": 18,
            "EndOffset": 23
        }
    ]
}
```

### 대량 텍스트 분석을 위한 배치 작업 
```

# S3 버킷 생성 (버킷 이름은 고유해야 함: trainingN = training1,2,3)
aws s3 mb s3://my-comprehend-bucket-trainingN

# 분석할 텍스트 파일들을 S3에 업로드
aws s3 cp customer_reviews.txt s3://my-comprehend-bucket-trainingN/input/
```

```
#Comprehend 배치 작업용 역할 생성 

cat > comprehend-trust-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "comprehend.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

aws iam create-role \
    --role-name ComprehendRole \
    --assume-role-policy-document file://comprehend-trust-policy.json

```

```
#S3 접근 권한 정책 연결

cat > comprehend-s3-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::my-comprehend-bucket-trainingN",
        "arn:aws:s3:::my-comprehend-bucket-trainingN/*"
      ]
    }
  ]
}
EOF

aws iam put-role-policy \
    --role-name ComprehendRole \
    --policy-name ComprehendS3Access \
    --policy-document file://comprehend-s3-policy.json

```

```
# 감정 분석 배치 작업 시작
aws comprehend start-sentiment-detection-job \
    --job-name "customer-sentiment-analysis" \
    --language-code ko \
    --input-data-config S3Uri=s3://my-comprehend-bucket-trainingN/input/ \
    --output-data-config S3Uri=s3://my-comprehend-bucket-trainingN/output/ \
    --data-access-role-arn arn:aws:iam::YOUR-ACCOUNT-ID:role/ComprehendRole

```

```
# SUBMITTED 상태에서 기다려도 /output 폴더가 생성되지 않을 경우, job의 상태를 확인하기 위한 명령어입니다.
aws comprehend describe-sentiment-detection-job \
--job-id 'YOUR-JOB-ID'

```

### 실습용 스크립트 생성
```

cat > analyze_review.sh << 'EOF'
#!/bin/bash

if [ $# -eq 0 ]; then
    echo "사용법: $0 '분석할 텍스트'"
    echo "예시: $0 '이 제품 정말 좋아요!'"
    exit 1
fi

TEXT="$1"
echo "=== 리뷰 분석 결과 ==="
echo "텍스트: $TEXT"
echo

echo "1. 언어 감지:"
aws comprehend detect-dominant-language --text "$TEXT" --query 'Languages[0].[LanguageCode,Score]' --output table

LANG_CODE=$(aws comprehend detect-dominant-language --text "$TEXT" --query 'Languages[0].LanguageCode' --output text)
echo

echo "2. 감정 분석:"
aws comprehend detect-sentiment --text "$TEXT" --language-code "$LANG_CODE" --query '[Sentiment,SentimentScore.Positive,SentimentScore.Negative]' --output table
echo

echo "3. 개체명 인식:"
aws comprehend detect-entities --text "$TEXT" --language-code "$LANG_CODE" --query 'Entities[*].[Type,Text,Score]' --output table
echo

echo "4. 핵심 구문:"
aws comprehend detect-key-phrases --text "$TEXT" --language-code "$LANG_CODE" --query 'KeyPhrases[*].[Text,Score]' --output table
EOF

chmod +x analyze_review.sh

```

### 스크립트 사용 예시
```

# 긍정적 리뷰 분석
./analyze_review.sh "이 제품 정말 만족해요! 품질도 좋고 배송도 빨라요."

# 부정적 리뷰 분석
./analyze_review.sh "배송이 늦고 포장이 허술해서 실망했습니다."

# 영어 리뷰 분석
./analyze_review.sh "This product is amazing! Great value for money."

```

## 실제 활용 사례
### 1. 고객 피드백 대시보드 생성
```

# 여러 리뷰를 한 번에 분석하여 CSV 파일로 저장
cat > batch_analysis.sh << 'EOF'
#!/bin/bash

echo "Review,Sentiment,Positive_Score,Negative_Score,Key_Phrases" > analysis_results.csv

while IFS= read -r line; do
    if [ -n "$line" ]; then
        SENTIMENT=$(aws comprehend detect-sentiment --text "$line" --language-code ko --query 'Sentiment' --output text)
        POS_SCORE=$(aws comprehend detect-sentiment --text "$line" --language-code ko --query 'SentimentScore.Positive' --output text)
        NEG_SCORE=$(aws comprehend detect-sentiment --text "$line" --language-code ko --query 'SentimentScore.Negative' --output text)
        KEY_PHRASES=$(aws comprehend detect-key-phrases --text "$line" --language-code ko --query 'KeyPhrases[*].Text' --output text | tr '\n' ';')

        echo "\"$line\",\"$SENTIMENT\",\"$POS_SCORE\",\"$NEG_SCORE\",\"$KEY_PHRASES\"" >> analysis_results.csv
    fi
done < customer_reviews.txt

echo "분석 완료! analysis_results.csv 파일을 확인하세요."
EOF

chmod +x batch_analysis.sh
./batch_analysis.sh
```

### 2. 실시간 리뷰 모니터링
```

cat > monitor_reviews.sh << 'EOF'
#!/bin/bash

echo "실시간 리뷰 모니터링을 시작합니다..."
echo "부정적 리뷰가 감지되면 알림이 표시됩니다."

while true; do
    echo "새 리뷰를 입력하세요 (종료하려면 'quit' 입력):"
    read -r review

    if [ "$review" = "quit" ]; then
        break
    fi

    SENTIMENT=$(aws comprehend detect-sentiment --text "$review" --language-code ko --query 'Sentiment' --output text)
    NEG_SCORE=$(aws comprehend detect-sentiment --text "$review" --language-code ko --query 'SentimentScore.Negative' --output text)

    if [ "$SENTIMENT" = "NEGATIVE" ] && (( $(echo "$NEG_SCORE > 0.7" | bc -l) )); then
        echo "🚨 부정적 리뷰 감지! 즉시 대응이 필요합니다."
        echo "부정 점수: $NEG_SCORE"
    else
        echo "✅ 리뷰 감정: $SENTIMENT"
    fi
    echo "---"
done
EOF

chmod +x monitor_reviews.sh

```
```

./monitor_reviews.sh

#예시
#새 리뷰를 입력하세요 (종료하려면 'quit' 입력):
#제품 배송이 너무 느립니다.
```
    

