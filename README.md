Streaming ETL Pipeline

![Untitled](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/caf803c9-9104-4757-a9b7-aa1e52f69bcb)

1. kinesis Data Stream을 통해서 실시간으로 데이터가 들어옴.
   - 들어온 데이터를 실시간으로 S3 Bucket에 저장
   - Glue Job을 실행
2. Glue Job을 통해서 Lambda가 실행되어 데이터를 변형한다.
3. 변형된 데이터를 Glue를 통해 Amazon Redshift에 저장한다.
4. 저장된 데이터는 Redshift Spectrum을 사용해 외부에서 데이터를 쿼리할 수 있고, Athena를 통해서 데이터를 쿼리할 수 도 있다.

VPC와 S3 Gateway Endpoint 생성

### Create Kinesis Data Stream
```
$ aws kinesis create-stream --stream-name etl-kinesis-stream --shard-count 2 --region ap-northeast
-2
```
### Create S3 Bucket
Kinesis Data Stream을 통해서 들어온 값을 저장하기 위한 S3 Bucket을 생성
```
$ aws s3 mb s3://etl-kinesis-s3-pjm1024cl --region ap-northeast-2
```
S3에 저장하는 Firehose를 생성

![Untitled](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/987e8e15-2d47-4a08-94d9-0f522eeec506)
![Untitled](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/d7f8732b-b5e3-4240-b1d9-88926e8d24b9)
![Untitled](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/4e8cd3c6-0ac4-4a84-9c84-449170d53c2e)
![Untitled](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/80ea9a95-a56c-45af-95a1-bf7f6ff39d14)

Kinesis Data Stream에 데이터를 전송
```
import boto3
import uuid
import random
import datetime
import json


dt = datetime.datetime.now()
client = boto3.client('kinesis', region_name='ap-northeast-2')

def put_records(records):

    kinesis_records = []

    for r in records:
        kinesis_records.append(
            {
                    'Data': json.dumps(r).encode('utf-8'),
                    'PartitionKey': 'string_for_partition'
            }
        )
    response = client.put_records(
        Records=kinesis_records,
        StreamName='etl-kinesis-stream'
    )
    return response

def main():
    while True:
        timestamp = dt.timestamp()

        print('start to send')
        data = [
            {
                'time': timestamp,
                'uuid': str(uuid.uuid1()),
                'country': random.choice(['KOREA', 'CHINA', 'JAPAN']),
                'web_sites': random.choice(['https://blog.wsi-korea.org', 'https://security.wsi-korea.internal', 'https://nas.wsi-korea.org', 'https://directory.wsi-korea.internal', 'https://monitoring.wsi-korea.internal:8018']),
                'event': random.choice(['left_click', 'right_click']),
                'action': random.choice(['download', 'upload', 'deleted', 'update', 'replace', 'copy']),
                'ping': str(random.randint(1, 100)) + "ms"
            }
        ]
        response = put_records(data)
        print('response: {}'.format(response))

if __name__ == "__main__":
    main()
```
```
$ pip3 install boto3
$ python3 main.py
```

Lake Location을 생성

![Untitled](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/704e01ab-f1eb-4f98-a75f-ab40cc613a1c)
![Untitled](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/6b852da5-9e75-4dff-80c3-4f38a91df89d)
![Untitled](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/bcc174f8-aca8-462d-88d7-d1606c82eacf)

총 두 개의 Data Lake Locations를 생성하였는지 확인

![Untitled](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/69a5602e-2145-42eb-875a-0e7e26a4705f)

Lake Formation을 활용해 Glue Data Base도 생성

![Untitled](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/ca8ff243-6593-4e48-a64a-61ac4dfb1e07)
![Untitled](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/f97bea0f-8791-47c3-9c3d-1f5821ba6eef)

변형된 데이터를 저장할 DB 생성

![Untitled](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/1f0fdfb6-2c92-4d09-b144-11e74c150608)

Crawler를 생성해서 S3에 올라온 데이터를 크롤링할 수 있도록 생성
   - 보안그룹을 생성

![image](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/bfbdf2b4-a4ef-43dc-b729-593ecd79ae36)
![Untitled](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/60fe5b26-be0e-4171-ba5e-d6b7110c442b)
![Untitled](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/827f7e08-a1ce-412f-bdb2-60b3762aa117)
![Untitled](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/8b4d0a13-4683-45ef-9a8e-690b424f096e)
![Untitled](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/95f5e985-a878-428b-b21b-76a04610501a)
![Untitled](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/14306a4b-ad8b-400b-ba67-d77bb0590296)

data source를 설정

![Untitled](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/e04a3f4b-405e-4eca-927d-192dc0aae957)
![Untitled](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/e4c836ce-1fbf-4add-9498-a7e16aace51d)
![Untitled](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/b7cd32cb-efb5-42c1-8be6-7d6ecaee78ac)
![Untitled](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/46e64bb7-9ec6-486e-9763-3e0cae989dcf)

Transform Crawler를 생성

![Untitled](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/cffeade0-796e-41d8-a3a9-5eafd6ae76c9)
![Untitled](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/179488f6-94b8-4193-a18d-34b815588439)
![Untitled](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/4e69df23-8d5c-49d2-8544-173c6f3c3aae)
![Untitled](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/5a024fc6-08d2-410e-ac20-5de39f6e4f3e)

### Create Redshift Cluster

![Untitled](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/a26e994b-d558-4b35-84c5-11c1a167bd7d)
![Untitled](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/ab4cc188-c95c-458f-a896-a553144d93d7)
![Untitled](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/430fc567-58f5-4a63-be08-a7bc6c185247)
![Untitled](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/ae2d040d-eb94-4bd4-ac31-b96ea8a0ae53)
![Untitled](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/29b4c3bf-ccea-412a-a911-92f924da7f5b)

Cluster를 생성

### Grant Glue Database & Data Locations
![Untitled](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/6320b351-7021-4236-9844-37bd80ff98d8)
![Untitled](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/f68d5f38-875d-48a6-9795-7bc3ebd197b7)
![Untitled](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/6313dbbb-17e2-4246-a353-d47cc7a6f09f)
![Untitled](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/5bd14064-f9f7-4321-8c11-6f611a428a01)
![Untitled](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/1a88202f-2ccd-4b83-b61a-8e33c012ef8a)
![Untitled](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/b537d14f-75a3-4b92-9d6e-df57204042d0)
![Untitled](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/740306ce-0c80-463d-8204-b0e1ef2a05ee)
![Untitled](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/ec7f0a83-901c-45b7-a3d4-9f43d0aad022)

Run Crawler

etl-db-transform Table에 대한 권한 Grant 

![Untitled](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/4d8028e9-fd2d-40d0-86d5-016de9dfb75e)
![Untitled](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/a51f8412-7fbc-4859-b10f-e78c324c98bc)
![Untitled](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/4c1f9d52-7e82-47d9-ae4e-647e11a585a4)

### Create Glue Job
![Untitled](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/55f10aeb-5477-4742-99ce-a1f9c47da8c4)
![Untitled](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/de836410-e9b7-4f78-becf-2c864e37162e)
![Untitled](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/682f287c-666e-4d7e-bfde-ef38973bfde7)
![Untitled](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/eefd3c90-80a7-4aa9-9d36-a9e71dbf175c)
```
select * from myDataSource WHERE `event` LIKE "left_click"
```
![Untitled](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/220e02c2-f2f5-46ea-b37e-1c79525f7ac3)
![Untitled](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/a5c8a0ac-a33e-43c9-b3d8-6fb0253138d8)
![Untitled](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/226da8f2-c95f-4fdf-a0bd-f75f110b793d)
![Untitled](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/5c6aa3d1-4d70-4b3c-add8-3fc12996f6d2)
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor1",
            "Effect": "Allow",
            "Action": [
                "cloudwatch:PutMetricData",
                "s3:CreateBucket",
                "s3:ListBucket",
                "ec2:DescribeVpcAttribute",
                "glue:*",
                "logs:CreateLogStream",
                "ec2:DescribeNetworkInterfaces",
                "logs:AssociateKmsKey",
                "iam:ListRolePolicies",
                "s3:DeleteObject",
								"glue:*",
                "ec2:DescribeRouteTables",
                "iam:GetRole",
                "s3:PutBucketPublicAccessBlock",
                "ec2:DeleteNetworkInterface",
                "s3:GetBucketAcl",
                "logs:CreateLogGroup",
                "logs:PutLogEvents",
                "ec2:DescribeSecurityGroups",
                "ec2:CreateNetworkInterface",
                "s3:GetObject",
                "s3:PutObject",
								"lakeformation:*",
                "s3:ListAllMyBuckets",
                "ec2:DescribeVpcEndpoints",
                "ec2:DescribeSubnets",
                "s3:GetBucketLocation",
                "iam:GetRolePolicy"
            ],
            "Resource": "*"
        }
    ]
}
```
 IAM Role에는 위와 같은 권한을 부여
SAVE > RUN
S3 버킷에 아래와 같이 파일이 업로드됨

![Untitled](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/f8a44dd5-cd86-4443-8662-5efb42fd52fb)

Crawler ROle의 IAM Role을 수정
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:PutObject"
            ],
            "Resource": [
                "arn:aws:s3:::etl-kinesis-s3-pjm1024cl/*"
            ]
        }
    ]
}
```
Transform Crawler을 돌려서 etl-db-transform에 Table을 생성해준 뒤, Glue Job으로 이동해서 아래와 같이 Target을 추가
 
![Untitled](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/9761c57c-43d7-4622-9c0d-fee6a79c7295)
![Untitled](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/34d65cdd-0b45-4fda-a7c0-9e4abe47f17a)

### Create Lambda
Lambda Function을 하나 생성
```
import json
import logging
import boto3

logger = logging.getLogger()
logger.setLevel(logging.INFO)

client = boto3.client('glue')

GlueJobName = "etl-glue-job"
originalCrawlerName = "etl-crawler-original"

def lambda_handler(event, context):
    eventType = event['Records'][0]['eventSource']
    
    if eventType == "aws:s3":
        response = client.start_crawler(Name = originalCrawlerName)
    elif eventType == "aws:glue":
        response = client.start_job_run(JobName = GlueJobName)
    return response
```
S3에 Event Trigger를 설정

![Untitled](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/73cb9b6d-57fb-42ed-8bf5-2ec426ed04cb)
![Untitled](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/90aa1afe-554f-431f-84fb-2b6ec410d59b)

해당 Lambda IAM Role에 위와 같은 권한부여

### Create EventBridge

![Untitled](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/3b575fc2-039f-4d0e-8f6f-643015125722)
![Untitled](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/ec56a27f-775c-4676-a4d4-7bf4322df9f1)
![Untitled](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/1d169866-60cc-45fa-ade2-ad1e71a84dd3)
![Untitled](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/1bae69df-eeb5-4e2f-9f60-46b599abab03)
```
{
    "detail-type": [
        "Glue Crawler State Change"
    ],
    "source": [
        "aws.glue"
    ],
    "detail": {
        "crawlerName": [
            "etl-crawler-original"
        ],
        "state": [
            "Succeeded"
        ]
    }
}
```
![Untitled](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/d8324045-ecb7-4b04-a22c-258642fda985)

생성

Firehose에 데이터를 넣으면 자동으로 Lambda를 통해서 Glue Crawler가 실행

![Untitled](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/ac58288f-9b6d-43b7-83ae-059e8a4021e7)

Glue Job도 실행

![Untitled](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/f29e8f7c-0f2e-473c-8b57-7047adc931f9)

Redshift에서 아래와 같은 SQL 문을 입력하면 Redshift에서 Glue Database에 있는 데이터를 불러와서 조회
```
create external schema spectrums
from data catalog
database 'etl-db-transform'
iam_role 'arn:aws:iam::948216186415:role/service-role/AWSGlueServiceRole-roles';

CREATE TABLE transform
diststyle all
AS
SELECT * FROM spectrum.transform;
```

실행하기 전에 AWSGlueServiceRole-roles를 Redshift에 Associate 해주고 아래와 같은 권한을 추가
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": "lakeformation:*",
            "Resource": "*"
        }
    ]
}
```
쿼리

```
SELECT * FROM transform
```
![Untitled](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/819029b5-1955-4c11-80e4-65068fb21792)
