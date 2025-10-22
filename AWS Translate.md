# AWS Translate 실습 
## 사전 준비사항
### 1. AWS CLI 설치 및 구성 
```

# AWS CLI 설치 확인
aws --version

# AWS 자격 증명 설정
aws configure
```

### 2. 필요한 권한 확인
- `translate:TranslateText`
- `translate:StartTextTranslationJob`
- `translate:DescribeTextTranslationJob`
- `translate:ListTextTranslationJobs`

## 실습 1 : 실시간 텍스트 번역
### 1. 기본 번역 명령어
```

# 영어 → 한국어 번역
aws translate translate-text \
    --source-language-code en \
    --target-language-code ko \
    --text "Hello, I have a problem with my recent order. The product arrived damaged."

# 일본어 → 한국어 번역
aws translate translate-text \
    --source-language-code ja \
    --target-language-code ko \
    --text "こんにちは、注文した商品が壊れて届きました。返品したいです。"

# 중국어 → 한국어 번역
aws translate translate-text \
    --source-language-code zh \
    --target-language-code ko \
    --text "您好，我收到的商品有质量问题，我想要退款。"
```


### 2. 자동 언어 감지
```

# 언어 자동 감지 (source-language-code를 'auto'로 설정)
aws translate translate-text \
    --source-language-code auto \
    --target-language-code ko \
    --text "Bonjour, je voudrais retourner mon achat"

```

## 실습 2 : 고객 지원 시스템 시뮬레이션
### 1. 예시 고객 문의 데이터 생성
다음 내용으로 customer_inquiries.txt 파일 생성
```

영어 문의: I received the wrong item. I ordered a blue shirt but got a red one.
일본어 문의: 配送が遅れています。いつ届く予定ですか？
중국어 문의: 我想取消我的订单，还没有发货。
스페인어 문의: El producto no funciona correctamente. ¿Puedo obtener un reembolso?
프랑스어 문의: La taille du vêtement ne me convient pas. Comment puis-je l'échanger?
```

### 2. 배치 번역 스크립트
```

cat > translate_inquiries.sh << 'EOF'
#!/bin/bash
# translate_inquiries.sh

echo "=== 고객 문의 번역 시스템 ==="
echo ""

# 영어 문의 번역
echo "1. 영어 고객 문의:"
english_text="I received the wrong item. I ordered a blue shirt but got a red one."
echo "원문: $english_text"
aws translate translate-text \
    --source-language-code en \
    --target-language-code ko \
    --text "$english_text" \
    --query 'TranslatedText' \
    --output text
echo ""

# 일본어 문의 번역
echo "2. 일본어 고객 문의:"
japanese_text="配送が遅れています。いつ届く予定ですか？"
echo "원문: $japanese_text"
aws translate translate-text \
    --source-language-code ja \
    --target-language-code ko \
    --text "$japanese_text" \
    --query 'TranslatedText' \
    --output text
echo ""

# 중국어 문의 번역
echo "3. 중국어 고객 문의:"
chinese_text="我想取消我的订单，还没有发货。"
echo "원문: $chinese_text"
aws translate translate-text \
    --source-language-code zh \
    --target-language-code ko \
    --text "$chinese_text" \
    --query 'TranslatedText' \
    --output text
echo ""

EOF
```

**실행 방법**
```

chmod +x translate_inquiries.sh
./translate_inquiries.sh
```

### 3. 응답 번역 (한국어 -> 외국어)
```

# 한국어 응답을 영어로 번역
korean_response="죄송합니다. 잘못된 상품을 보내드린 것 같습니다. 교환 처리를 위해 고객센터로 연락주시기 바랍니다."

aws translate translate-text \
    --source-language-code ko \
    --target-language-code en \
    --text "$korean_response"

# 한국어 응답을 일본어로 번역
aws translate translate-text \
    --source-language-code ko \
    --target-language-code ja \
    --text "$korean_response"
```

## 실습 3 : 고급 기능 활용
### 1. 용어집 사용 
```

# 용어집 파일 생성 (terminology.csv)
cat > terminology.csv << EOF
en,ko
smartphone,스마트폰
tablet,태블릿
warranty,보증서
refund,환불
exchange,교환
EOF

# 용어집 업로드 (CSV + UTF-8 인코딩 지정)
aws translate import-terminology \
  --name ecommerce-terms \
  --merge-strategy OVERWRITE \
  --data-file fileb://terminology.csv \
  --terminology-data Format=CSV


# 용어집을 사용한 번역
aws translate translate-text \
    --source-language-code en \
    --target-language-code ko \
    --text "I need to exchange my smartphone under warranty" \
    --terminology-names ecommerce-terms
```

### 2. 지원 언어 목록 확인
```

# 번역 가능한 언어 목록 조회
aws translate list-languages

# 특정 소스 언어에서 번역 가능한 대상 언어 확인
aws translate list-languages --display-language-code ko
```

## 실습 4 : 대량 문서 번역 (배치 작업)
### 1. S3 버킷 준비
```

# S3 버킷 생성
BUCKET_NAME="my-translate-bucket-$(date +%s)"
aws s3 mb s3://$BUCKET_NAME

# 입력 폴더 생성
aws s3api put-object --bucket $BUCKET_NAME --key input/
aws s3api put-object --bucket $BUCKET_NAME --key output/
```

### 2. 번역할 문서 업로드
reviews.txt 파일 생성
```

This product is amazing! Fast delivery and great quality.
The customer service was excellent and very helpful.
I'm not satisfied with this purchase. The item was damaged.
Great value for money. I would recommend this to others.
The product description was misleading. Very disappointed.
```

S3에 업로드
```
aws s3 cp reviews.txt s3://$BUCKET_NAME/input/
```

### 3. S3 접근 권한 부여 
1) 신뢰 정책 파일 생성
```
cat > trust.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "translate.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF


```

2) 역할 생성
```
aws iam create-role \
--role-name AmazonTranslateServiceRole \
--assume-role-policy-document file://trust.json \
--description "Role assumed by Amazon Translate for batch translation jobs"
```

4) S3 접근 권한을 역할에 붙이기
```
cat > s3policy.json <<'EOF'
{
	"Version": "2012-10-17",
	"Statement": [
		{
			"Effect": "Allow",
			"Action": [
				"s3:ListBucket"
			],
			"Resource": [
				"arn:aws:s3:::my-translate-bucket-N"
			]
		},
		{
			"Effect": "Allow",
			"Action": [
				"s3:GetObject",
				"s3:PutObject",
				"s3:DeleteObject"
			],
			"Resource": [
				"arn:aws:s3:::my-translate-bucket-N/*"
			]
		}
	]
}
EOF
```

```
aws iam put-role-policy \
--role-name AmazonTranslateServiceRole \
--policy-name TranslateS3Access \
--policy-document file://s3policy.json
```

### 4. 배치 번역 작업 시작
```

# IAM 역할 ARN (번역 서비스용)
ROLE_ARN="arn:aws:iam::YOUR-ACCOUNT-ID:role/AmazonTranslateServiceRole"

# 배치 번역 작업 시작
JOB_NAME="review-translation-$(date +%s)"
aws translate start-text-translation-job \
    --job-name $JOB_NAME \
    --source-language-code en \
    --target-language-codes ko \
    --input-data-config S3Uri=s3://$BUCKET_NAME/input/,ContentType=text/plain \
    --output-data-config S3Uri=s3://$BUCKET_NAME/output/ \
    --data-access-role-arn $ROLE_ARN
```

### 5. 작업 상태 확인 
```

# 번역 작업 목록 확인
aws translate list-text-translation-jobs

# 환경변수에 JOB_ID 입력, JOB_ID는 'list-text-translation-jobs'에서 확인한 실제 ID로 변경
JOB_ID="4334a3902b334783591826a5a48f60d5"
echo "작업 ID: $JOB_ID"

# 특정 작업 상태 확인
aws translate describe-text-translation-job --job-id $JOB_ID

# 완료된 번역 결과 다운로드
aws s3 sync s3://$BUCKET_NAME/output/ ./translated_output/
```


