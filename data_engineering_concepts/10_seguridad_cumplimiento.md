# 10. Seguridad y Cumplimiento

## 📋 Tabla de Contenidos
- [Fundamentos de Seguridad](#fundamentos-de-seguridad)
- [Encriptación de Datos](#encriptación-de-datos)
- [Gestión de Credenciales](#gestión-de-credenciales)
- [Network Security](#network-security)
- [Autenticación y Autorización](#autenticación-y-autorización)
- [Auditoría y Logging](#auditoría-y-logging)
- [Compliance Frameworks](#compliance-frameworks)
- [Incident Response](#incident-response)

---

## Fundamentos de Seguridad

### CIA Triad

```
┌─────────────────────────────────────────────┐
│            CIA TRIAD                        │
├─────────────────────────────────────────────┤
│                                              │
│  ┌──────────────────────────────────────┐  │
│  │      CONFIDENTIALITY                 │  │
│  │  (Confidencialidad)                  │  │
│  │  • Solo usuarios autorizados         │  │
│  │  • Encriptación                      │  │
│  │  • Access controls                   │  │
│  └──────────────────────────────────────┘  │
│                                              │
│  ┌──────────────────────────────────────┐  │
│  │      INTEGRITY                       │  │
│  │  (Integridad)                        │  │
│  │  • Datos no modificados sin permiso  │  │
│  │  • Checksums, hashing                │  │
│  │  • Version control                   │  │
│  └──────────────────────────────────────┘  │
│                                              │
│  ┌──────────────────────────────────────┐  │
│  │      AVAILABILITY                    │  │
│  │  (Disponibilidad)                    │  │
│  │  • Datos accesibles cuando necesarios│  │
│  │  • High availability, backups        │  │
│  │  • Disaster recovery                 │  │
│  └──────────────────────────────────────┘  │
└─────────────────────────────────────────────┘
```

### Defense in Depth (Defensa en Capas)

```
┌────────────────────────────────────────────────┐
│         SECURITY LAYERS                        │
├────────────────────────────────────────────────┤
│                                                 │
│  Layer 7: Application Security                │
│           └─ Input validation, SQL injection   │
│                                                 │
│  Layer 6: Data Security                       │
│           └─ Encryption, masking, tokenization │
│                                                 │
│  Layer 5: Application Layer                   │
│           └─ API security, authentication      │
│                                                 │
│  Layer 4: Endpoint Security                   │
│           └─ Antivirus, EDR                    │
│                                                 │
│  Layer 3: Network Security                    │
│           └─ Firewalls, VPN, network segmentation │
│                                                 │
│  Layer 2: Perimeter Security                  │
│           └─ IDS/IPS, WAF                      │
│                                                 │
│  Layer 1: Physical Security                   │
│           └─ Data center access controls       │
└────────────────────────────────────────────────┘
```

### Principio de Least Privilege

```python
# ❌ MAL: Usuario con permisos excesivos
GRANT ALL PRIVILEGES ON DATABASE analytics TO etl_user;

# ✅ BIEN: Solo permisos necesarios
GRANT SELECT, INSERT ON analytics.sales TO etl_user;
GRANT SELECT ON analytics.customers TO etl_user;
```

---

## Encriptación de Datos

### Tipos de Encriptación

| Tipo | Cuándo | Dónde | Ejemplo |
|------|--------|-------|---------|
| **At Rest** | Datos almacenados | Disco, S3, DB | AES-256 |
| **In Transit** | Datos moviéndose | Network | TLS 1.3 |
| **In Use** | Datos en memoria | RAM (raro) | Homomorphic encryption |

### 1. Encryption at Rest

#### AWS S3 Encryption

```python
import boto3

s3 = boto3.client('s3')

# ===== SERVER-SIDE ENCRYPTION (SSE) =====

# SSE-S3: S3 maneja las llaves
s3.put_object(
    Bucket='my-bucket',
    Key='sensitive-data.csv',
    Body=data,
    ServerSideEncryption='AES256'
)

# SSE-KMS: AWS Key Management Service (más control)
s3.put_object(
    Bucket='my-bucket',
    Key='pii-data.csv',
    Body=data,
    ServerSideEncryption='aws:kms',
    SSEKMSKeyId='arn:aws:kms:us-east-1:123456789:key/abc-123'
)

# SSE-C: Customer-provided keys (tú manejas las llaves)
import base64
import os

encryption_key = os.urandom(32)  # 256-bit key
key_b64 = base64.b64encode(encryption_key).decode()

s3.put_object(
    Bucket='my-bucket',
    Key='ultra-sensitive.csv',
    Body=data,
    SSECustomerAlgorithm='AES256',
    SSECustomerKey=key_b64,
    SSECustomerKeyMD5=base64.b64encode(hashlib.md5(encryption_key).digest()).decode()
)

# ===== CLIENT-SIDE ENCRYPTION =====
# Encriptar antes de subir
from cryptography.fernet import Fernet

# Generar key
key = Fernet.generate_key()
cipher = Fernet(key)

# Encriptar datos
plaintext = b"Sensitive customer data"
ciphertext = cipher.encrypt(plaintext)

# Subir encriptado
s3.put_object(
    Bucket='my-bucket',
    Key='encrypted-data.bin',
    Body=ciphertext
)

# Descargar y desencriptar
response = s3.get_object(Bucket='my-bucket', Key='encrypted-data.bin')
ciphertext = response['Body'].read()
plaintext = cipher.decrypt(ciphertext)

# ===== BUCKET-LEVEL ENCRYPTION (default) =====
s3.put_bucket_encryption(
    Bucket='my-bucket',
    ServerSideEncryptionConfiguration={
        'Rules': [
            {
                'ApplyServerSideEncryptionByDefault': {
                    'SSEAlgorithm': 'aws:kms',
                    'KMSMasterKeyID': 'arn:aws:kms:us-east-1:123456789:key/abc-123'
                },
                'BucketKeyEnabled': True  # Reduce costos de KMS
            }
        ]
    }
)
```

#### Database Encryption

```python
# ===== SNOWFLAKE: Encriptación automática =====
# Snowflake encripta TODOS los datos automáticamente (AES-256)
# Gratis, sin configuración

# ===== REDSHIFT: Encryption =====
# Crear cluster encriptado
import boto3

redshift = boto3.client('redshift')

redshift.create_cluster(
    ClusterIdentifier='my-encrypted-cluster',
    NodeType='dc2.large',
    MasterUsername='admin',
    MasterUserPassword='SecurePass123!',
    Encrypted=True,  # Encriptación habilitada
    KmsKeyId='arn:aws:kms:us-east-1:123456789:key/abc-123'
)

# ===== POSTGRESQL: Transparent Data Encryption (TDE) =====
# pgcrypto extension para column-level encryption
"""
CREATE EXTENSION pgcrypto;

-- Encriptar columna
CREATE TABLE customers (
    customer_id SERIAL PRIMARY KEY,
    email VARCHAR(255),
    ssn_encrypted BYTEA  -- Columna encriptada
);

-- Insertar con encriptación
INSERT INTO customers (email, ssn_encrypted)
VALUES (
    'user@example.com',
    pgp_sym_encrypt('123-45-6789', 'encryption_key')
);

-- Query desencriptando
SELECT
    email,
    pgp_sym_decrypt(ssn_encrypted, 'encryption_key') as ssn
FROM customers;
"""

# ===== COLUMN-LEVEL ENCRYPTION (Python) =====
from cryptography.fernet import Fernet
import psycopg2

# Setup
key = Fernet.generate_key()
cipher = Fernet(key)

# Insertar datos encriptados
conn = psycopg2.connect(...)
cursor = conn.cursor()

ssn = "123-45-6789"
ssn_encrypted = cipher.encrypt(ssn.encode())

cursor.execute("""
    INSERT INTO customers (email, ssn_encrypted)
    VALUES (%s, %s)
""", ('user@example.com', ssn_encrypted))

# Query y desencriptar
cursor.execute("SELECT ssn_encrypted FROM customers WHERE email = %s", ('user@example.com',))
ssn_encrypted = cursor.fetchone()[0]
ssn_decrypted = cipher.decrypt(ssn_encrypted).decode()
print(f"SSN: {ssn_decrypted}")
```

### 2. Encryption in Transit

```python
# ===== TLS/SSL para conexiones =====

# Snowflake (TLS habilitado por default)
import snowflake.connector

conn = snowflake.connector.connect(
    user='username',
    password='password',
    account='myaccount',
    # TLS 1.2+ siempre habilitado
)

# PostgreSQL con SSL
import psycopg2

conn = psycopg2.connect(
    host='postgres.example.com',
    port=5432,
    database='analytics',
    user='user',
    password='pass',
    sslmode='require',  # Requiere SSL
    sslrootcert='/path/to/ca-cert.pem'
)

# Redshift con SSL
import psycopg2

conn = psycopg2.connect(
    host='mycluster.abc123.us-east-1.redshift.amazonaws.com',
    port=5439,
    database='analytics',
    user='admin',
    password='pass',
    sslmode='require'
)

# ===== API CALLS con HTTPS =====
import requests

# ✅ BIEN: HTTPS
response = requests.get('https://api.example.com/data')

# ❌ MAL: HTTP (sin encriptación)
# response = requests.get('http://api.example.com/data')

# Verificar certificado SSL
response = requests.get('https://api.example.com/data', verify=True)

# ===== S3 Transfer con SSL =====
import boto3

s3 = boto3.client(
    's3',
    use_ssl=True,  # Forzar HTTPS
    config=boto3.session.Config(signature_version='s3v4')
)
```

### 3. Hashing (para passwords, checksums)

```python
import hashlib
import secrets

# ===== PASSWORD HASHING =====
# ❌ NUNCA usar MD5 o SHA1 para passwords
# password_hash = hashlib.md5(password.encode()).hexdigest()  # INSEGURO

# ✅ USAR bcrypt, argon2, o scrypt
import bcrypt

# Hash password
password = "user_password_123"
salt = bcrypt.gensalt(rounds=12)  # Cost factor (más alto = más seguro pero lento)
hashed = bcrypt.hashpw(password.encode(), salt)

print(f"Hashed: {hashed}")

# Verificar password
input_password = "user_password_123"
if bcrypt.checkpw(input_password.encode(), hashed):
    print("Password correcto")
else:
    print("Password incorrecto")

# ===== FILE INTEGRITY (checksums) =====
# SHA-256 para verificar integridad de archivos
def calculate_file_hash(file_path: str) -> str:
    """Calcular SHA-256 de archivo"""
    sha256_hash = hashlib.sha256()

    with open(file_path, 'rb') as f:
        # Leer en chunks (para archivos grandes)
        for byte_block in iter(lambda: f.read(4096), b""):
            sha256_hash.update(byte_block)

    return sha256_hash.hexdigest()

# Uso
hash1 = calculate_file_hash('data.csv')
# ... transferir archivo ...
hash2 = calculate_file_hash('data_downloaded.csv')

if hash1 == hash2:
    print("✅ Archivo íntegro")
else:
    print("❌ Archivo corrupto o modificado")

# ===== DATA ANONYMIZATION (one-way hash) =====
def anonymize_pii(value: str) -> str:
    """
    Hash irreversible de PII para analytics
    (mantiene unicidad, pero no reversible)
    """
    # Agregar salt para prevenir rainbow tables
    salt = "company_secret_salt_2025"
    return hashlib.sha256((value + salt).encode()).hexdigest()

# Ejemplo
email = "user@example.com"
hashed_email = anonymize_pii(email)
print(f"Original: {email}")
print(f"Hashed: {hashed_email}")

# Mismo email siempre produce mismo hash (útil para joins)
# Pero no puede revertirse a email original
```

---

## Gestión de Credenciales

### ❌ NUNCA hacer esto

```python
# NUNCA hardcodear credenciales
db_password = "MySecretPassword123!"
api_key = "sk_live_abc123xyz"

# NUNCA commitear secrets a git
# .env file con secrets → debe estar en .gitignore

# NUNCA logear credenciales
print(f"Connecting with password: {password}")  # NO!
```

### ✅ Mejores Prácticas

#### 1. AWS Secrets Manager

```python
import boto3
import json

secrets_client = boto3.client('secretsmanager', region_name='us-east-1')

# ===== CREAR SECRET =====
secrets_client.create_secret(
    Name='prod/database/credentials',
    SecretString=json.dumps({
        'username': 'admin',
        'password': 'SecurePassword123!',
        'host': 'db.example.com',
        'port': 5432
    })
)

# ===== RECUPERAR SECRET =====
def get_database_credentials():
    """Obtener credenciales de DB desde Secrets Manager"""
    response = secrets_client.get_secret_value(SecretId='prod/database/credentials')
    secret = json.loads(response['SecretString'])
    return secret

# Uso
creds = get_database_credentials()

import psycopg2
conn = psycopg2.connect(
    host=creds['host'],
    port=creds['port'],
    database='analytics',
    user=creds['username'],
    password=creds['password']
)

# ===== ROTAR SECRET automáticamente =====
# Lambda function para rotar password
def lambda_handler(event, context):
    secret_arn = event['SecretId']
    token = event['ClientRequestToken']
    step = event['Step']

    if step == "createSecret":
        # Generar nuevo password
        new_password = secrets.token_urlsafe(32)

        # Guardar en pending
        secrets_client.put_secret_value(
            SecretId=secret_arn,
            ClientRequestToken=token,
            SecretString=json.dumps({'password': new_password}),
            VersionStages=['AWSPENDING']
        )

    elif step == "setSecret":
        # Actualizar password en database
        # ...

    elif step == "testSecret":
        # Verificar que nuevo password funciona
        # ...

    elif step == "finishSecret":
        # Marcar como current
        secrets_client.update_secret_version_stage(
            SecretId=secret_arn,
            VersionStage='AWSCURRENT',
            MoveToVersionId=token
        )

# Configurar rotación automática (cada 30 días)
secrets_client.rotate_secret(
    SecretId='prod/database/credentials',
    RotationLambdaARN='arn:aws:lambda:us-east-1:123:function:rotate-secret',
    RotationRules={'AutomaticallyAfterDays': 30}
)
```

#### 2. Environment Variables

```python
import os
from dotenv import load_dotenv

# Cargar .env file (local development)
load_dotenv()

# Acceder a variables
db_password = os.getenv('DB_PASSWORD')
api_key = os.getenv('API_KEY')

if not db_password:
    raise ValueError("DB_PASSWORD not set")

# .env file (NUNCA commitear esto):
"""
DB_PASSWORD=SecurePassword123
API_KEY=sk_live_abc123
SNOWFLAKE_ACCOUNT=myaccount
SNOWFLAKE_USER=etl_user
"""

# .gitignore (SIEMPRE agregar):
"""
.env
*.env
secrets/
credentials.json
"""
```

#### 3. AWS Parameter Store

```python
import boto3

ssm = boto3.client('ssm')

# ===== GUARDAR PARÁMETRO =====
ssm.put_parameter(
    Name='/prod/database/host',
    Value='db.prod.example.com',
    Type='String'
)

ssm.put_parameter(
    Name='/prod/database/password',
    Value='SecurePassword123!',
    Type='SecureString',  # Encriptado con KMS
    KeyId='alias/aws/ssm'
)

# ===== RECUPERAR PARÁMETRO =====
def get_parameter(name: str, decrypt: bool = True) -> str:
    """Obtener parámetro de SSM"""
    response = ssm.get_parameter(Name=name, WithDecryption=decrypt)
    return response['Parameter']['Value']

# Uso
db_host = get_parameter('/prod/database/host')
db_password = get_parameter('/prod/database/password')

# ===== RECUPERAR MÚLTIPLES PARÁMETROS =====
def get_parameters_by_path(path: str) -> dict:
    """Obtener todos los parámetros bajo un path"""
    response = ssm.get_parameters_by_path(
        Path=path,
        Recursive=True,
        WithDecryption=True
    )

    return {
        param['Name'].split('/')[-1]: param['Value']
        for param in response['Parameters']
    }

# Obtener todas las configs de database
db_config = get_parameters_by_path('/prod/database')
# {'host': '...', 'password': '...', 'port': '5432'}
```

#### 4. Airflow Connections

```python
from airflow.hooks.base import BaseHook

# ===== CREAR CONNECTION en Airflow UI =====
# Connection ID: snowflake_prod
# Connection Type: Snowflake
# Host: myaccount.snowflakecomputing.com
# Schema: analytics
# Login: etl_user
# Password: ***
# Extra: {"warehouse": "ETL_WH", "role": "ETL_ROLE"}

# ===== USAR CONNECTION en DAG =====
from airflow import DAG
from airflow.providers.snowflake.operators.snowflake import SnowflakeOperator

dag = DAG('etl_with_secure_connection', ...)

load_task = SnowflakeOperator(
    task_id='load_to_snowflake',
    snowflake_conn_id='snowflake_prod',  # No hardcoded credentials!
    sql='COPY INTO sales FROM @my_stage',
    dag=dag
)

# ===== ACCEDER programáticamente =====
def get_snowflake_connection():
    """Obtener credenciales de Snowflake desde Airflow"""
    connection = BaseHook.get_connection('snowflake_prod')

    return {
        'account': connection.host.split('.')[0],
        'user': connection.login,
        'password': connection.password,
        'warehouse': connection.extra_dejson.get('warehouse'),
        'role': connection.extra_dejson.get('role')
    }

# Uso
from snowflake.connector import connect

creds = get_snowflake_connection()
conn = connect(
    account=creds['account'],
    user=creds['user'],
    password=creds['password'],
    warehouse=creds['warehouse'],
    role=creds['role']
)
```

---

## Network Security

### VPC (Virtual Private Cloud)

```
┌──────────────────────────────────────────────────┐
│              AWS VPC ARCHITECTURE                │
├──────────────────────────────────────────────────┤
│                                                   │
│  Internet Gateway                                │
│         │                                         │
│         ▼                                         │
│  ┌────────────────────────────────────────┐     │
│  │  Public Subnet (10.0.1.0/24)           │     │
│  │  ┌──────────┐      ┌──────────┐        │     │
│  │  │  NAT     │      │ Bastion  │        │     │
│  │  │ Gateway  │      │   Host   │        │     │
│  │  └──────────┘      └──────────┘        │     │
│  └────────────────────────────────────────┘     │
│         │                                         │
│         ▼                                         │
│  ┌────────────────────────────────────────┐     │
│  │  Private Subnet (10.0.2.0/24)          │     │
│  │  ┌──────────┐      ┌──────────┐        │     │
│  │  │    EC2   │      │  EMR     │        │     │
│  │  │ Instances│      │ Cluster  │        │     │
│  │  └──────────┘      └──────────┘        │     │
│  └────────────────────────────────────────┘     │
│         │                                         │
│         ▼                                         │
│  ┌────────────────────────────────────────┐     │
│  │  Data Subnet (10.0.3.0/24)             │     │
│  │  ┌──────────┐      ┌──────────┐        │     │
│  │  │ Redshift │      │   RDS    │        │     │
│  │  │ Cluster  │      │ Database │        │     │
│  │  └──────────┘      └──────────┘        │     │
│  └────────────────────────────────────────┘     │
└──────────────────────────────────────────────────┘
```

### Security Groups

```python
import boto3

ec2 = boto3.client('ec2')

# ===== CREAR SECURITY GROUP =====
response = ec2.create_security_group(
    GroupName='data-pipeline-sg',
    Description='Security group for data pipeline',
    VpcId='vpc-abc123'
)

security_group_id = response['GroupId']

# ===== REGLAS INBOUND =====
# Permitir SSH solo desde bastion host
ec2.authorize_security_group_ingress(
    GroupId=security_group_id,
    IpPermissions=[
        {
            'IpProtocol': 'tcp',
            'FromPort': 22,
            'ToPort': 22,
            'UserIdGroupPairs': [
                {'GroupId': 'sg-bastion-123'}  # Solo desde bastion
            ]
        }
    ]
)

# Permitir PostgreSQL desde subnet específica
ec2.authorize_security_group_ingress(
    GroupId=security_group_id,
    IpPermissions=[
        {
            'IpProtocol': 'tcp',
            'FromPort': 5432,
            'ToPort': 5432,
            'IpRanges': [
                {'CidrIp': '10.0.2.0/24', 'Description': 'Private subnet'}
            ]
        }
    ]
)

# ===== REGLAS OUTBOUND =====
# Permitir HTTPS saliente (para APIs externas)
ec2.authorize_security_group_egress(
    GroupId=security_group_id,
    IpPermissions=[
        {
            'IpProtocol': 'tcp',
            'FromPort': 443,
            'ToPort': 443,
            'IpRanges': [
                {'CidrIp': '0.0.0.0/0', 'Description': 'HTTPS outbound'}
            ]
        }
    ]
)
```

### VPN y PrivateLink

```python
# ===== AWS PRIVATELINK (acceder a S3 sin internet) =====
# Crear VPC endpoint para S3
ec2.create_vpc_endpoint(
    VpcId='vpc-abc123',
    ServiceName='com.amazonaws.us-east-1.s3',
    RouteTableIds=['rtb-123456'],
    VpcEndpointType='Gateway'
)

# Ahora tráfico a S3 va por red privada de AWS (no internet)

# ===== VPN CONNECTION =====
# Conectar on-premise data center a AWS VPC
# 1. Crear Customer Gateway (tu router)
cgw = ec2.create_customer_gateway(
    Type='ipsec.1',
    PublicIp='203.0.113.1',  # IP pública de tu data center
    BgpAsn=65000
)

# 2. Crear Virtual Private Gateway
vgw = ec2.create_vpn_gateway(Type='ipsec.1')
ec2.attach_vpn_gateway(VpcId='vpc-abc123', VpnGatewayId=vgw['VpnGateway']['VpnGatewayId'])

# 3. Crear VPN Connection
vpn = ec2.create_vpn_connection(
    Type='ipsec.1',
    CustomerGatewayId=cgw['CustomerGateway']['CustomerGatewayId'],
    VpnGatewayId=vgw['VpnGateway']['VpnGatewayId']
)

# Ahora on-premise puede acceder recursos en VPC de forma segura
```

---

## Autenticación y Autorización

### OAuth 2.0 y JWT

```python
from datetime import datetime, timedelta
import jwt
import requests

# ===== GENERAR JWT TOKEN =====
def generate_jwt_token(user_id: str, secret_key: str) -> str:
    """Generar JWT token para usuario"""
    payload = {
        'user_id': user_id,
        'exp': datetime.utcnow() + timedelta(hours=1),  # Expira en 1 hora
        'iat': datetime.utcnow(),
        'scope': ['read:data', 'write:data']
    }

    token = jwt.encode(payload, secret_key, algorithm='HS256')
    return token

# ===== VERIFICAR JWT TOKEN =====
def verify_jwt_token(token: str, secret_key: str) -> dict:
    """Verificar y decodificar JWT token"""
    try:
        payload = jwt.decode(token, secret_key, algorithms=['HS256'])
        return payload
    except jwt.ExpiredSignatureError:
        raise Exception("Token expirado")
    except jwt.InvalidTokenError:
        raise Exception("Token inválido")

# ===== API AUTHENTICATION =====
def call_protected_api(endpoint: str, token: str):
    """Llamar API con autenticación Bearer token"""
    headers = {
        'Authorization': f'Bearer {token}',
        'Content-Type': 'application/json'
    }

    response = requests.get(endpoint, headers=headers)

    if response.status_code == 401:
        raise Exception("No autorizado")

    return response.json()

# Uso
secret = 'my-secret-key-keep-safe'
token = generate_jwt_token('user_123', secret)

data = call_protected_api('https://api.example.com/data', token)

# ===== OAUTH 2.0 CLIENT CREDENTIALS FLOW =====
def get_oauth_token(client_id: str, client_secret: str, token_url: str) -> str:
    """Obtener access token usando client credentials"""
    response = requests.post(
        token_url,
        data={
            'grant_type': 'client_credentials',
            'client_id': client_id,
            'client_secret': client_secret,
            'scope': 'read write'
        }
    )

    if response.status_code != 200:
        raise Exception(f"Failed to get token: {response.text}")

    return response.json()['access_token']

# Uso
access_token = get_oauth_token(
    client_id='my-client-id',
    client_secret='my-client-secret',
    token_url='https://auth.example.com/oauth/token'
)
```

### IAM (Identity and Access Management)

```python
import boto3

iam = boto3.client('iam')

# ===== CREAR IAM ROLE =====
assume_role_policy = {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "glue.amazonaws.com"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}

response = iam.create_role(
    RoleName='GlueETLRole',
    AssumeRolePolicyDocument=json.dumps(assume_role_policy),
    Description='Role for Glue ETL jobs'
)

role_arn = response['Role']['Arn']

# ===== ATTACH POLICIES =====
# Managed policy (predefinido por AWS)
iam.attach_role_policy(
    RoleName='GlueETLRole',
    PolicyArn='arn:aws:iam::aws:policy/service-role/AWSGlueServiceRole'
)

# Custom inline policy
s3_policy = {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:PutObject"
            ],
            "Resource": "arn:aws:s3:::my-data-bucket/*"
        },
        {
            "Effect": "Allow",
            "Action": "s3:ListBucket",
            "Resource": "arn:aws:s3:::my-data-bucket"
        }
    ]
}

iam.put_role_policy(
    RoleName='GlueETLRole',
    PolicyName='S3AccessPolicy',
    PolicyDocument=json.dumps(s3_policy)
)

# ===== PRINCIPLE OF LEAST PRIVILEGE =====
# ❌ MAL: Permisos demasiado amplios
bad_policy = {
    "Effect": "Allow",
    "Action": "s3:*",  # Todos los permisos de S3
    "Resource": "*"    # En todos los recursos
}

# ✅ BIEN: Permisos específicos
good_policy = {
    "Effect": "Allow",
    "Action": [
        "s3:GetObject",
        "s3:PutObject"
    ],
    "Resource": [
        "arn:aws:s3:::my-data-bucket/etl-input/*",
        "arn:aws:s3:::my-data-bucket/etl-output/*"
    ]
}

# ===== ASUMIR ROL (for cross-account access) =====
sts = boto3.client('sts')

assumed_role = sts.assume_role(
    RoleArn='arn:aws:iam::123456789:role/GlueETLRole',
    RoleSessionName='etl-session'
)

credentials = assumed_role['Credentials']

# Usar credenciales temporales
s3_client = boto3.client(
    's3',
    aws_access_key_id=credentials['AccessKeyId'],
    aws_secret_access_key=credentials['SecretAccessKey'],
    aws_session_token=credentials['SessionToken']
)
```

---

## Auditoría y Logging

### AWS CloudTrail

```python
import boto3
from datetime import datetime, timedelta

cloudtrail = boto3.client('cloudtrail')

# ===== HABILITAR CLOUDTRAIL =====
cloudtrail.create_trail(
    Name='data-pipeline-audit',
    S3BucketName='my-cloudtrail-logs',
    IncludeGlobalServiceEvents=True,
    IsMultiRegionTrail=True,
    EnableLogFileValidation=True  # Integrity checking
)

cloudtrail.start_logging(Name='data-pipeline-audit')

# ===== QUERY CLOUDTRAIL LOGS =====
def get_s3_access_logs(bucket_name: str, days: int = 7):
    """
    Obtener todos los accesos a un bucket S3 en últimos N días
    """
    events = []

    paginator = cloudtrail.get_paginator('lookup_events')

    for page in paginator.paginate(
        LookupAttributes=[
            {
                'AttributeKey': 'ResourceName',
                'AttributeValue': bucket_name
            }
        ],
        StartTime=datetime.utcnow() - timedelta(days=days),
        EndTime=datetime.utcnow()
    ):
        events.extend(page['Events'])

    return [
        {
            'time': event['EventTime'],
            'user': event.get('Username', 'N/A'),
            'event': event['EventName'],
            'ip': event.get('SourceIPAddress', 'N/A')
        }
        for event in events
    ]

# Obtener accesos a bucket sensible
s3_logs = get_s3_access_logs('my-pii-data-bucket', days=30)

for log in s3_logs:
    print(f"[{log['time']}] {log['user']} - {log['event']} from {log['ip']}")

# ===== DETECTAR ACCESOS SOSPECHOSOS =====
def detect_suspicious_access(bucket_name: str):
    """
    Detectar patrones sospechosos en accesos a S3
    """
    logs = get_s3_access_logs(bucket_name, days=1)

    # Detectar accesos desde IPs inusuales
    ip_counts = {}
    for log in logs:
        ip = log['ip']
        ip_counts[ip] = ip_counts.get(ip, 0) + 1

    # Alertar si una IP tiene > 1000 requests en 1 día
    for ip, count in ip_counts.items():
        if count > 1000:
            send_security_alert(
                f"Suspicious activity: {count} requests from {ip} to {bucket_name}"
            )

    # Detectar descargas masivas
    download_events = [log for log in logs if log['event'] == 'GetObject']
    if len(download_events) > 10000:
        send_security_alert(
            f"Mass download detected: {len(download_events)} files from {bucket_name}"
        )
```

### Application Logging

```python
import logging
import json
from datetime import datetime

# ===== CONFIGURAR LOGGING SEGURO =====
# NUNCA logear credenciales o PII

class SecureFormatter(logging.Formatter):
    """
    Formatter que redacta información sensible
    """
    SENSITIVE_FIELDS = ['password', 'api_key', 'token', 'ssn', 'credit_card']

    def format(self, record):
        # Redactar campos sensibles en mensaje
        if isinstance(record.msg, dict):
            record.msg = self._redact_sensitive(record.msg)

        return super().format(record)

    def _redact_sensitive(self, data: dict) -> dict:
        """Reemplazar valores sensibles con '***'"""
        redacted = data.copy()

        for key in redacted:
            if any(sensitive in key.lower() for sensitive in self.SENSITIVE_FIELDS):
                redacted[key] = '***REDACTED***'

        return redacted

# Setup logger
logger = logging.getLogger('etl_pipeline')
logger.setLevel(logging.INFO)

handler = logging.StreamHandler()
handler.setFormatter(SecureFormatter(
    '%(asctime)s - %(name)s - %(levelname)s - %(message)s'
))
logger.addHandler(handler)

# ===== STRUCTURED LOGGING =====
def log_etl_event(event_type: str, details: dict):
    """
    Log estructurado para eventos de ETL
    """
    log_entry = {
        'timestamp': datetime.utcnow().isoformat(),
        'event_type': event_type,
        'details': details
    }

    logger.info(json.dumps(log_entry))

# Uso
log_etl_event('job_started', {
    'job_name': 'sales_daily_etl',
    'execution_date': '2025-01-15',
    'source': 's3://data/sales/'
})

log_etl_event('data_quality_check', {
    'table': 'sales',
    'rows_processed': 150000,
    'rows_failed': 15,
    'failure_rate': 0.01
})

# ===== AUDIT LOG para accesos a PII =====
def log_pii_access(user: str, table: str, columns: list, query: str):
    """
    Registrar accesos a datos PII (compliance)
    """
    audit_entry = {
        'timestamp': datetime.utcnow().isoformat(),
        'event_type': 'PII_ACCESS',
        'user': user,
        'table': table,
        'columns': columns,
        'query_hash': hashlib.sha256(query.encode()).hexdigest()  # No logear query completa
    }

    # Enviar a sistema de auditoría centralizado
    send_to_audit_system(audit_entry)

    # También logear localmente
    logger.warning(f"PII ACCESS: {user} accessed {table}.{','.join(columns)}")

# Uso en query wrapper
def execute_query(query: str, user: str):
    """Ejecutar query con audit logging"""
    # Detectar si query accede PII
    pii_tables = detect_pii_tables_in_query(query)

    if pii_tables:
        for table, columns in pii_tables.items():
            log_pii_access(user, table, columns, query)

    # Ejecutar query
    result = connection.execute(query)
    return result
```

### Centralized Logging (ELK Stack)

```python
from elasticsearch import Elasticsearch
from datetime import datetime

# ===== ENVIAR LOGS A ELASTICSEARCH =====
es = Elasticsearch(['https://elasticsearch.example.com:9200'])

def send_log_to_elk(log_entry: dict):
    """Enviar log a Elasticsearch"""
    es.index(
        index=f"etl-logs-{datetime.utcnow().strftime('%Y.%m.%d')}",
        body=log_entry
    )

# Uso
log_entry = {
    '@timestamp': datetime.utcnow().isoformat(),
    'level': 'INFO',
    'pipeline': 'sales_etl',
    'message': 'ETL completed successfully',
    'rows_processed': 150000,
    'duration_seconds': 320
}

send_log_to_elk(log_entry)

# ===== BUSCAR LOGS =====
def search_error_logs(pipeline_name: str, hours: int = 24):
    """Buscar logs de errores en últimas N horas"""
    query = {
        "query": {
            "bool": {
                "must": [
                    {"match": {"pipeline": pipeline_name}},
                    {"match": {"level": "ERROR"}},
                    {
                        "range": {
                            "@timestamp": {
                                "gte": f"now-{hours}h",
                                "lte": "now"
                            }
                        }
                    }
                ]
            }
        },
        "sort": [{"@timestamp": "desc"}]
    }

    results = es.search(index="etl-logs-*", body=query)
    return results['hits']['hits']

# Obtener errores recientes
errors = search_error_logs('sales_etl', hours=24)
for error in errors:
    print(f"[{error['_source']['@timestamp']}] {error['_source']['message']}")
```

---

## Compliance Frameworks

### SOC 2 (Service Organization Control 2)

**5 Trust Service Criteria:**

1. **Security** - Protección contra acceso no autorizado
2. **Availability** - Sistema disponible según SLA
3. **Processing Integrity** - Procesamiento completo y preciso
4. **Confidentiality** - Información confidencial protegida
5. **Privacy** - PII manejado según políticas

**Controles para Data Engineering:**

```python
# ===== CONTROL: Access Reviews =====
def generate_access_review_report():
    """
    Generar reporte de accesos para SOC 2 audit
    """
    import boto3

    iam = boto3.client('iam')

    # Listar usuarios y sus permisos
    users = iam.list_users()['Users']

    report = []
    for user in users:
        username = user['UserName']

        # Obtener grupos
        groups = iam.list_groups_for_user(UserName=username)['Groups']

        # Obtener policies attached
        policies = iam.list_attached_user_policies(UserName=username)['AttachedPolicies']

        # Último acceso
        access_key_last_used = iam.get_access_key_last_used(
            AccessKeyId=user.get('AccessKeyId', 'N/A')
        ) if 'AccessKeyId' in user else None

        report.append({
            'username': username,
            'groups': [g['GroupName'] for g in groups],
            'policies': [p['PolicyName'] for p in policies],
            'last_used': access_key_last_used.get('AccessKeyLastUsed', {}).get('LastUsedDate') if access_key_last_used else 'Never'
        })

    return report

# Generar reporte mensual
report = generate_access_review_report()
# Enviar a compliance team para review

# ===== CONTROL: Change Management =====
def log_schema_change(table: str, change: str, approved_by: str):
    """
    Logear cambios de schema (SOC 2 requirement)
    """
    change_log = {
        'timestamp': datetime.utcnow().isoformat(),
        'table': table,
        'change': change,
        'approved_by': approved_by,
        'executed_by': os.getenv('USER')
    }

    # Guardar en audit trail
    with open('/var/log/schema_changes.log', 'a') as f:
        f.write(json.dumps(change_log) + '\n')

# Uso antes de ALTER TABLE
log_schema_change(
    table='customers',
    change='ADD COLUMN loyalty_points INT',
    approved_by='john.doe@company.com'
)

# Ejecutar cambio
conn.execute('ALTER TABLE customers ADD COLUMN loyalty_points INT')
```

### PCI DSS (Payment Card Industry Data Security Standard)

**Para datos de tarjetas de crédito:**

```python
# ===== TOKENIZATION (en vez de guardar números reales) =====
import secrets

def tokenize_credit_card(card_number: str) -> str:
    """
    Tokenizar número de tarjeta (guardar en vault seguro)
    """
    # Generar token único
    token = f"tok_{secrets.token_urlsafe(32)}"

    # Guardar mapping en vault encriptado (no en DB principal)
    save_to_secure_vault(token, card_number)

    # Retornar token (esto va en DB)
    return token

def detokenize(token: str) -> str:
    """
    Recuperar número real desde vault (solo cuando necesario)
    """
    # Requiere permisos especiales
    if not has_pci_access(current_user()):
        raise PermissionError("No PCI access")

    return get_from_secure_vault(token)

# Uso
card_number = "4532-1234-5678-9010"
token = tokenize_credit_card(card_number)  # "tok_abc123..."

# Guardar solo token en DB
cursor.execute("""
    INSERT INTO orders (customer_id, payment_token, amount)
    VALUES (%s, %s, %s)
""", (customer_id, token, amount))

# ===== MASKING para display =====
def mask_credit_card(card_number: str) -> str:
    """Mostrar solo últimos 4 dígitos"""
    return f"****-****-****-{card_number[-4:]}"

# Display en UI
print(f"Card: {mask_credit_card(card_number)}")  # "Card: ****-****-****-9010"

# ===== AUDIT LOG para accesos a PCI data =====
def log_pci_access(user: str, action: str, card_token: str):
    """Registrar acceso a datos PCI (compliance requirement)"""
    audit_entry = {
        'timestamp': datetime.utcnow().isoformat(),
        'user': user,
        'action': action,
        'card_token': card_token,
        'ip_address': get_client_ip()
    }

    # Log permanente (retención 1 año mínimo)
    send_to_compliance_log(audit_entry)
```

---

## Incident Response

### Security Incident Response Plan

```python
from enum import Enum
from dataclasses import dataclass
from datetime import datetime

class IncidentSeverity(Enum):
    LOW = 1        # Minor issue, no data exposed
    MEDIUM = 2     # Potential vulnerability
    HIGH = 3       # Data exposure possible
    CRITICAL = 4   # Active breach, PII exposed

@dataclass
class SecurityIncident:
    incident_id: str
    severity: IncidentSeverity
    description: str
    detected_at: datetime
    affected_systems: list
    affected_data: list
    responder: str

# ===== INCIDENT DETECTION =====
def detect_anomalous_query(query: str, user: str) -> bool:
    """
    Detectar queries sospechosas
    """
    suspicious_patterns = [
        'SELECT * FROM customers',  # Full table scan de PII
        'DELETE FROM',
        'DROP TABLE',
        'UNION SELECT'  # Posible SQL injection
    ]

    for pattern in suspicious_patterns:
        if pattern.upper() in query.upper():
            # Crear incidente
            incident = SecurityIncident(
                incident_id=f"INC-{datetime.utcnow().strftime('%Y%m%d%H%M%S')}",
                severity=IncidentSeverity.HIGH,
                description=f"Suspicious query detected: {pattern}",
                detected_at=datetime.utcnow(),
                affected_systems=['production_database'],
                affected_data=['customers_table'],
                responder='security@company.com'
            )

            # Alertar
            alert_security_team(incident)

            # Bloquear query
            return True

    return False

# ===== INCIDENT RESPONSE WORKFLOW =====
def handle_data_breach(incident: SecurityIncident):
    """
    Manejar incidente de data breach
    """
    print(f"🚨 INCIDENT DETECTED: {incident.incident_id}")
    print(f"Severity: {incident.severity.name}")

    if incident.severity == IncidentSeverity.CRITICAL:
        # 1. CONTENCIÓN
        print("Step 1: CONTAINMENT")
        # - Bloquear acceso a sistemas afectados
        revoke_all_access(incident.affected_systems)
        # - Aislar sistemas comprometidos
        isolate_systems(incident.affected_systems)

        # 2. EVALUACIÓN
        print("Step 2: ASSESSMENT")
        # - ¿Qué datos fueron expuestos?
        affected_records = count_affected_records(incident.affected_data)
        print(f"  Affected records: {affected_records}")
        # - ¿Cuándo ocurrió?
        # - ¿Cómo ocurrió?

        # 3. NOTIFICACIÓN
        print("Step 3: NOTIFICATION")
        # - Alertar stakeholders internos
        notify_executives(incident)
        # - Si PII expuesto: notificar usuarios afectados (GDPR requirement)
        if 'pii' in incident.affected_data:
            notify_affected_users(affected_records)
        # - Notificar autoridades si requerido (72 horas GDPR)
        if affected_records > 500:
            notify_data_protection_authority(incident)

        # 4. REMEDIACIÓN
        print("Step 4: REMEDIATION")
        # - Patchear vulnerabilidad
        # - Cambiar credenciales comprometidas
        rotate_all_credentials()
        # - Restaurar desde backup limpio si necesario

        # 5. RECOVERY
        print("Step 5: RECOVERY")
        # - Restaurar servicios
        # - Verificar integridad de datos

        # 6. POST-MORTEM
        print("Step 6: POST-MORTEM")
        # - ¿Qué salió mal?
        # - ¿Cómo prevenir en futuro?
        schedule_post_mortem(incident)

# ===== ALERTING =====
def alert_security_team(incident: SecurityIncident):
    """Alertar al equipo de seguridad"""
    if incident.severity in [IncidentSeverity.HIGH, IncidentSeverity.CRITICAL]:
        # PagerDuty
        trigger_pagerduty_alert(
            title=f"Security Incident: {incident.description}",
            severity=incident.severity.name,
            details=incident.__dict__
        )

    # Slack
    send_slack_alert(
        channel='#security-incidents',
        message=f"""
🚨 Security Incident Detected

ID: {incident.incident_id}
Severity: {incident.severity.name}
Description: {incident.description}
Affected Systems: {', '.join(incident.affected_systems)}
        """
    )

    # Email
    send_email(
        to=['security@company.com', 'ciso@company.com'],
        subject=f'[{incident.severity.name}] Security Incident: {incident.incident_id}',
        body=f"""
A security incident has been detected.

Incident ID: {incident.incident_id}
Severity: {incident.severity.name}
Description: {incident.description}
Detected At: {incident.detected_at}

Immediate action required.
        """
    )
```

---

## 🎯 Preguntas de Entrevista

**P: ¿Cómo asegurarías datos sensibles (PII) en un data warehouse?**

R:
1. **Encriptación**: At-rest (AES-256) y in-transit (TLS 1.3)
2. **Access Control**: RBAC, solo usuarios autorizados acceden PII
3. **Column Masking**: Usuarios sin permiso ven datos masked
4. **Audit Logging**: Log de todos los accesos a PII
5. **Data Classification**: Etiquetar columnas PII en metadata
6. **Retention Policies**: Auto-delete después de período requerido
7. **Network Security**: VPC privado, no expuesto a internet

**P: Explica cómo manejarías credenciales en un pipeline ETL**

R:
1. **NUNCA hardcodear**: No passwords en código
2. **Secrets Manager**: AWS Secrets Manager o Parameter Store
3. **Environment Variables**: Para dev local (.env en .gitignore)
4. **IAM Roles**: Para servicios AWS (Glue, Lambda)
5. **Rotation**: Rotar credenciales automáticamente cada 30-90 días
6. **Least Privilege**: Solo permisos necesarios
7. **Audit**: Log cuando se acceden secrets

**P: Tu pipeline expone accidentalmente 10,000 registros con PII. ¿Qué haces?**

R:
1. **Contención inmediata**: Eliminar datos expuestos, revocar accesos
2. **Evaluación**: ¿Qué datos? ¿Cuántos usuarios? ¿Cuándo ocurrió?
3. **Notificación**:
   - Stakeholders internos inmediatamente
   - Usuarios afectados (24-72 horas, según regulación)
   - Autoridades si >500 usuarios (GDPR: 72 horas)
4. **Remediación**: Fix del bug, cambiar credenciales
5. **Prevención**: ¿Cómo prevenir futuro? Mejores controles, validaciones
6. **Documentación**: Incident report completo

---

**Siguiente:** [11. Modelado para BI y Analytics](11_modelado_bi_analytics.md)
