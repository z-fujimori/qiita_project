# 2026.0707
## RDS作成

AWSコンソールのCloud Shellへクエリを投げていく

```shell
SG_ID=$(aws ec2 create-security-group \
    --group-name postgres-demo-sg \
    --description "Postgres Demo" \
    --vpc-id $VPC_ID \
    --query GroupId \
    --output text)
echo $SG_ID

# 検証用に全開放。必要な接続元にのみしぽる。
aws ec2 authorize-security-group-ingress \
    --group-id $SG_ID \
    --protocol tcp \
    --port 5432 \
    --cidr 0.0.0.0/0
```

```
PASSWORD="*****"
# PostgreSQL作成
aws rds create-db-instance \
    --db-instance-identifier postgres-demo \
    --engine postgres \
    --db-instance-class db.t4g.micro \
    --allocated-storage 20 \
    --master-username postgres \
    --master-user-password "$PASSWORD" \
    --db-name demo_db \
    --vpc-security-group-ids $SG_ID \
    --publicly-accessible \
    --backup-retention-period 0 \
    --no-multi-az
# RDSインスタンスが起動を終えて「利用可能」状態になるのを待つコマンド
aws rds wait db-instance-available \
    --db-instance-identifier postgres-demo
# エンドポイント取得
POSTGRESQL_ENDPOINT=$(aws rds describe-db-instances \
    --db-instance-identifier postgres-demo \
    --query "DBInstances[0].Endpoint.Address" \
    --output text)
echo $POSTGRESQL_ENDPOINT
# 接続
PGPASSWORD=$PASSWORD psql \
  "host=${POSTGRESQL_ENDPOINT} \
   port=5432 \
   dbname=demo_db \
   user=postgres \
   sslmode=require"
```

```
-- table作成
CREATE TABLE fridge_items ( 
  id          INTEGER PRIMARY KEY, 
  item_name   VARCHAR(100) NOT NULL, 
  quantity    INTEGER NOT NULL, 
  updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP, 
  delete_flg  BOOLEAN NOT NULL DEFAULT FALSE 
);
-- データ投入
INSERT INTO fridge_items(id,item_name, quantity)
VALUES
(1,'牛乳',2),
(2,'卵',10),
(3,'プリン',3);
```

---
# 2026.0711
## Openflow
Snowflakeのワークシート
```
USE ROLE ACCOUNTADMIN;

CREATE ROLE IF NOT EXISTS OPENFLOW_ADMIN;

GRANT CREATE OPENFLOW DATA PLANE INTEGRATION
    ON ACCOUNT
    TO ROLE OPENFLOW_ADMIN;

GRANT CREATE OPENFLOW RUNTIME INTEGRATION
    ON ACCOUNT
    TO ROLE OPENFLOW_ADMIN;

GRANT CREATE COMPUTE POOL
    ON ACCOUNT
    TO ROLE OPENFLOW_ADMIN;

-- デフォルトRoleにOPENFLOW_ADMINをセット。
GRANT ROLE OPENFLOW_ADMIN
    TO USER <自分のユーザー名>;

ALTER USER <自分のユーザー名>
    SET DEFAULT_ROLE = OPENFLOW_ADMIN;

ALTER USER <自分のユーザー名>
    SET DEFAULT_SECONDARY_ROLES = ('ALL');
```

---

格納先用意
```
USE ROLE ACCOUNTADMIN;

CREATE WAREHOUSE IF NOT EXISTS OPENFLOW_WH
    WAREHOUSE_SIZE = XSMALL
    AUTO_SUSPEND = 60
    AUTO_RESUME = TRUE
    INITIALLY_SUSPENDED = TRUE;

CREATE DATABASE IF NOT EXISTS OPENFLOW_DB;

CREATE SCHEMA IF NOT EXISTS OPENFLOW_DB.DEMO;

CREATE TABLE IF NOT EXISTS OPENFLOW_DB.DEMO.REFRIGERATOR_ITEMS (
    ID NUMBER,
    ITEM_NAME VARCHAR,
    QUANTITY NUMBER,
    UPDATED_AT TIMESTAMP_NTZ,
    DELETE_FLG BOOLEAN,
    LOADED_AT TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()
);
```

Runtimeロール作成
```
USE ROLE ACCOUNTADMIN;

CREATE ROLE IF NOT EXISTS OPENFLOW_RUNTIME_POSTGRES;

GRANT USAGE, OPERATE
    ON WAREHOUSE OPENFLOW_WH
    TO ROLE OPENFLOW_RUNTIME_POSTGRES;

GRANT USAGE
    ON DATABASE OPENFLOW_DB
    TO ROLE OPENFLOW_RUNTIME_POSTGRES;

GRANT USAGE
    ON SCHEMA OPENFLOW_DB.DEMO
    TO ROLE OPENFLOW_RUNTIME_POSTGRES;

GRANT SELECT, INSERT, UPDATE, DELETE
    ON ALL TABLES IN SCHEMA OPENFLOW_DB.DEMO
    TO ROLE OPENFLOW_RUNTIME_POSTGRES;

GRANT SELECT, INSERT, UPDATE, DELETE
    ON FUTURE TABLES IN SCHEMA OPENFLOW_DB.DEMO
    TO ROLE OPENFLOW_RUNTIME_POSTGRES;

GRANT CREATE TABLE
    ON SCHEMA OPENFLOW_DB.DEMO
    TO ROLE OPENFLOW_RUNTIME_POSTGRES;
```

Snowflakeからの通信経路を作る。Snowflake-hosted RuntimeからSnowflake外へ通信するには、External Access Integrationが必要。
```
USE ROLE ACCOUNTADMIN;

CREATE DATABASE IF NOT EXISTS OPENFLOW_CONFIG;
CREATE SCHEMA IF NOT EXISTS OPENFLOW_CONFIG.NETWORK;

CREATE OR REPLACE NETWORK RULE
    OPENFLOW_CONFIG.NETWORK.POSTGRES_RULE
    MODE = EGRESS
    TYPE = HOST_PORT
    VALUE_LIST = (
        'your-proxy.proxy-xxxxxxxx.ap-northeast-1.rds.amazonaws.com:5432'
    );

CREATE OR REPLACE EXTERNAL ACCESS INTEGRATION
    OPENFLOW_POSTGRES_EAI
    ALLOWED_NETWORK_RULES = (
        OPENFLOW_CONFIG.NETWORK.POSTGRES_RULE
    )
    ENABLED = TRUE;

GRANT USAGE
    ON INTEGRATION OPENFLOW_POSTGRES_EAI
    TO ROLE OPENFLOW_RUNTIME_POSTGRES;
```

# 2026.07114


## RDS
ロジック変更。プロキシを挟みます
```
# ==========================================
# 1. セキュリティグループの作成
# ==========================================
SG_ID=$(aws ec2 create-security-group \
    --group-name postgres-demo-sg \
    --description "Postgres Demo" \
    --vpc-id $VPC_ID \
    --query GroupId \
    --output text)
echo "Security Group ID: $SG_ID"

# 検証用に全開放。必要な接続元にのみ絞る。
aws ec2 authorize-security-group-ingress \
    --group-id $SG_ID \
    --protocol tcp \
    --port 5432 \
    --cidr 0.0.0.0/0


# ==========================================
# 2. RDS PostgreSQL インスタンスの作成
# ==========================================
PASSWORD="*****" # 実際のパスワードを設定してください

aws rds create-db-instance \
    --db-instance-identifier postgres-demo \
    --engine postgres \
    --db-instance-class db.t4g.micro \
    --allocated-storage 20 \
    --master-username postgres \
    --master-user-password "$PASSWORD" \
    --db-name demo_db \
    --vpc-security-group-ids $SG_ID \
    --publicly-accessible \
    --backup-retention-period 0 \
    --no-multi-az

# RDSインスタンスが起動を終えて「利用可能」状態になるのを待つ
aws rds wait db-instance-available \
    --db-instance-identifier postgres-demo

# エンドポイント取得
POSTGRESQL_ENDPOINT=$(aws rds describe-db-instances \
    --db-instance-identifier postgres-demo \
    --query "DBInstances[0].Endpoint.Address" \
    --output text)
echo "RDS Endpoint: $POSTGRESQL_ENDPOINT"


# ==========================================
# 3. RDS Proxy の作成 (追加セクション)
# ==========================================

# 3-1. Secrets ManagerにRDSの認証情報を保存
SECRET_ARN=$(aws secretsmanager create-secret \
    --name "postgres-demo-secret" \
    --description "Credentials for postgres-demo" \
    --secret-string "{\"username\":\"postgres\",\"password\":\"$PASSWORD\"}" \
    --query ARN \
    --output text)
echo "Secret ARN: $SECRET_ARN"

# 3-2. RDS Proxy用のIAMロール作成（信頼関係ポリシーの定義）
cat <<EOF > proxy-trust-policy.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "",
      "Effect": "Allow",
      "Principal": {
        "Service": "rds.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

PROXY_ROLE_ARN=$(aws iam create-role \
    --role-name postgres-proxy-role \
    --assume-role-policy-document file://proxy-trust-policy.json \
    --query Role.Arn \
    --output text)
echo "IAM Role ARN: $PROXY_ROLE_ARN"

# Secrets Managerへのアクセス権限をロールに付与
cat <<EOF > proxy-policy.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowSecrets",
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue",
        "secretsmanager:DescribeSecret"
      ],
      "Resource": "$SECRET_ARN"
    }
  ]
}
EOF

aws iam put-role-policy \
    --role-name postgres-proxy-role \
    --policy-name postgres-proxy-policy \
    --policy-document file://proxy-policy.json

# IAMの反映を少し待つ
sleep 10

# DBが所属するサブネット（VPCの既存サブネット）を取得
# ※ここでは簡易的に、DBインスタンスが持つサブネットグループからサブネットIDを取得します
SUBNET_IDS=$(aws rds describe-db-instances \
    --db-instance-identifier postgres-demo \
    --query "DBInstances[0].DBSubnetGroup.Subnets[].SubnetIdentifier" \
    --output text)

# 3-3. RDS Proxy本体の作成
aws rds create-db-proxy \
    --db-proxy-name postgres-demo-proxy \
    --engine-family POSTGRESQL \
    --auth SecretArn=$SECRET_ARN,IAMAuth=DISABLED \
    --role-arn $PROXY_ROLE_ARN \
    --vpc-subnet-ids $SUBNET_IDS \
    --vpc-security-group-ids $SG_ID

# Proxyが利用可能になるのを待つ
echo "Waiting for RDS Proxy to become available..."
aws rds wait db-proxy-available \
    --db-proxy-name postgres-demo-proxy

# 3-4. ProxyのターゲットグループにDBインスタンスを登録
aws rds register-db-proxy-targets \
    --db-proxy-name postgres-demo-proxy \
    --target-group-name default \
    --db-instance-identifiers postgres-demo

# Proxyのエンドポイントを取得
PROXY_ENDPOINT=$(aws rds describe-db-proxies \
    --db-proxy-name postgres-demo-proxy \
    --query "DBProxies[0].Endpoint" \
    --output text)
echo "RDS Proxy Endpoint: $PROXY_ENDPOINT"


# ==========================================
# 4. Proxy経由での接続とテーブル作成
# ==========================================
# 接続先を PROXY_ENDPOINT に変更しています
PGPASSWORD=$PASSWORD psql \
  "host=${PROXY_ENDPOINT} \
   port=5432 \
   dbname=demo_db \
   user=postgres \
   sslmode=require" <<EOF
-- table作成
CREATE TABLE fridge_items ( 
  id          INTEGER PRIMARY KEY, 
  item_name   VARCHAR(100) NOT NULL, 
  quantity    INTEGER NOT NULL, 
  updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP, 
  delete_flg  BOOLEAN NOT NULL DEFAULT FALSE 
);
EOF

# 一時ファイルの削除
rm -f proxy-trust-policy.json proxy-policy.json
```

## Openflow

Network RuleとEAI(ExternalAccessIntegration)

```
USE ROLE ACCOUNTADMIN;

CREATE DATABASE IF NOT EXISTS OPENFLOW_CONFIG;
CREATE SCHEMA IF NOT EXISTS OPENFLOW_CONFIG.NETWORK;

-- NetworkRule
CREATE OR REPLACE NETWORK RULE
    OPENFLOW_CONFIG.NETWORK.POSTGRES_RULE
    MODE = EGRESS
    TYPE = HOST_PORT
    VALUE_LIST = (
        'your-proxy.proxy-xxxxxxxx.ap-northeast-1.rds.amazonaws.com:5432'
    );

-- EAI
CREATE OR REPLACE EXTERNAL ACCESS INTEGRATION
    OPENFLOW_POSTGRES_EAI
    ALLOWED_NETWORK_RULES = (
        OPENFLOW_CONFIG.NETWORK.POSTGRES_RULE
    )
    ENABLED = TRUE;

RuntimeRoleへ権限付与
GRANT USAGE
    ON INTEGRATION OPENFLOW_POSTGRES_EAI
    TO ROLE OPENFLOW_RUNTIME_POSTGRES;
```

error `SQL compilation error: invalid value '[postgres-demo-proxy.proxy-czm4uyg6g1cq.ap-northeast-1.rds.amazonaws.com]' for property 'VALUE_LIST'`






# リソースチェック
## 存在すると料金が発生するもの
1. COMPUTE_WH
    やばい料金かかるから気をつけな
2. RDS
3. RDS Proxy
    月々2,000円くらい
    ```
    aws rds delete-db-proxy --db-proxy-name postgres-demo-proxy
    ```
3. Secrets Manager
    月々60円くらい
    ```
    aws secretsmanager delete-secret \
    --secret-id "postgres-demo-secret" \
    --force-delete-without-recovery
    ```

