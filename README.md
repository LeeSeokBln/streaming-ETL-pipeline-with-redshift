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
$ aws s3 mb s3://etl-kinesis-s3-seokbin --region ap-northeast-2
```
S3에 저장하는 Firehose를 생성

![image](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/e5abc978-e2f3-4e33-a628-7664774f8000)
![image](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/3e995699-1ace-4052-861b-f9ad630a584f)
![image](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/5e3344f6-0bee-4161-aa80-29f635e79401)
![image](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/7c782ba0-0b47-4221-be19-8eff8e82936c)
![image](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/c3bcea2c-d2f5-4023-8494-4d34400ff6d8)


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

![image](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/475a7222-01f7-4dd9-9767-98ac92e976cd)
![image](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/0ac2ab55-9326-4c6d-86d4-b379abb2624c)


총 두 개의 Data Lake Locations를 생성하였는지 확인

![image](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/a39857c7-9229-43ba-97a3-bc82ffcb003e)

Lake Formation을 활용해 Glue Data Base도 생성

![image](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/c1d22afb-e28d-46b4-9ed1-da8fc1e0bd24)
![image](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/20fcf4fa-7f41-4f20-bc3d-8d0705410da3)


변형된 데이터를 저장할 DB 생성

![image](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/4a3351ca-b8c4-4215-aefc-2dd639e9e5b1)


Crawler를 생성해서 S3에 올라온 데이터를 크롤링할 수 있도록 생성
   - 보안그룹을 생성

![image](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/bfbdf2b4-a4ef-43dc-b729-593ecd79ae36)
![image](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/91fc251b-5671-411c-9f60-54b919dd8155)
![image](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/ee7e6e5e-dd8c-4cd9-85d5-f4ff002e9c9a)
![image](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/5ffbe2e0-6573-4979-ac14-1aef98bdaca8)
![image](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/93ebb87b-10c8-44b4-87d7-ba3aba3c2baf)
![image](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/0e1ebb97-a877-4253-8b44-3d92e366929b)


data source를 설정

![image](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/993dae13-f4e3-4c1e-b799-a5b5958e5c11)
![image](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/57b93af3-5d02-4531-ab83-e6949ebd457c)
![image](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/00d19e7a-18b6-46fc-b98d-e24df48da29a)


Transform Crawler를 생성

![image](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/0a1c2004-dfeb-4d69-ad11-1563b8a2f334)
![image](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/df895ce4-96c0-4186-bd92-a4840b2cde07)
![image](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/f57a243c-b898-4a97-a007-c75f5255097a)
![image](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/cf15aedb-37f2-44f6-89d4-1321cf737612)


### Create Redshift Cluster

![image](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/4d54b3dd-b5d6-4100-a5b1-03f81747ab94)
![image](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/42cf32ba-790c-4a38-9cfb-af209d0a21a5)
![image](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/b9d602ed-da0b-431b-a44b-a67adc6683b6)
![image](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/49e45df0-095f-4486-ba13-fdde907da854)
![image](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/300987d6-af53-470f-af25-d867c0a14970)


Cluster를 생성

### Grant Glue Database & Data Locations
![image](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/cfd33421-a8dd-40b1-8ebc-7140bd3a6df6)
![image](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/05ac1e4d-dc02-4cf2-8be3-36c1d135c014)
![image](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/cfb15f30-3202-47a9-9fac-10d4f3614d82)
![image](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/25e273eb-93e9-4ae3-a2e0-00c8d5b1ccae)
![image](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/1e1a4f07-b572-4dea-9d99-89bfc33be6ee)
![image](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/1a7f1f33-e1b2-43d2-aeed-282bf6369d2a)
![image](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/1fbae189-7ce2-4496-b7ed-b3eb9234fcff)
![image](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/c57bf1b3-e241-4a26-a18f-f2dcc9624d2a)

Run Crawler

etl-db-transform Table에 대한 권한도 똑같이 Grant 

![image](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/4368f850-1e97-42d8-b29b-ccd0aaa7045a)


### Create Glue Job
![image](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/c315db5e-3409-45dd-97a3-9cc3a3fd54b3)
![image](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/83099f3c-42dc-4997-9756-2ea30890d1cc)
![image](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/56d6fc36-6b19-4592-81d2-a7533b176061)

```
select * from myDataSource WHERE `event` LIKE "left_click"
```
![image](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/e714b905-4e01-4d30-a982-31a09614419c)

![image](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/f38a864a-095a-400b-85a1-cba42b2fa0a2)

![Untitled](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/226da8f2-c95f-4fdf-a0bd-f75f110b793d)
![image](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/4c028bb9-6d42-4c1d-9f57-e7de60168802)

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

![image](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/e6b93746-4ac7-4dea-929f-748287929802)

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

![image](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/1c67fb4a-d76b-4542-8ad0-e515f5183e2a)

![Untitled](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/90aa1afe-554f-431f-84fb-2b6ec410d59b)

해당 Lambda IAM Role에 위와 같은 권한부여

### Create EventBridge

![image](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/3e8018ab-c56e-44e8-8d1c-75bc3c327d6a)
![image](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/2808e5b2-1ca7-4243-9943-743b43a0a309)
![image](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/78fe2b84-d79f-471d-beb8-d90a5629c5a4)
![image](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/6ff983ad-533a-43a7-9976-0c3560855c24)
![image](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/b418769c-9ae4-4f5f-9d58-4d6fa00a7550)

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
![image](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/a6b87e8e-69c8-4c9d-8a88-4856824c71a7)


생성

Firehose에 데이터를 넣으면 자동으로 Lambda를 통해서 Glue Crawler가 실행

![Untitled](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/ac58288f-9b6d-43b7-83ae-059e8a4021e7)

Glue Job도 실행

![Untitled](https://github.com/LeeSeokBln/streaming-ETL-pipeline-with-redshift/assets/101256150/f29e8f7c-0f2e-473c-8b57-7047adc931f9)

Athena에서 아래와 같은 SQL 문을 입력하면 Redshift에서 Glue Database에 있는 데이터를 불러와서 조회
```
create external schema spectrums
from data catalog
database 'etl-db-transform'
iam_role 'arn:aws:iam::<계정id>:role/service-role/AWSGlueServiceRole-roles';

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
