### 개요
https://github.com/Great-Stone/vault-client-app-development-guide 의 개발 가이드를 참고하여 PostgreSQL 으로 사용 가능하도록 수정한 가이드입니다.
https://github.com/Great-Stone/vault-client-app-development-guide/blob/main/README_KO.md

### 디렉토리
```
Development-Guide/   
├── README.md                           # 이 파일 (전체 가이드 개요)   
├── samples/                            # 애플리케이션 샘플   
│   ├── java-pure-app/                  # Java Vault 클라이언트 예제   
│   ├── python-app/                     # Python Vault 클라이언트 예제   
│   └── java-web-springboot-app/        # Spring Boot 웹 애플리케이션 예제   
```
### 사전 환경 설정
#### 기본 가정
•	VAULT_ADDR, VAULT_TOKEN 환경 변수가 설정되어 있어야 합니다.   
•	PostgreSQL 데이터베이스가 이미 구축되어 있어야 합니다.   
•	Vault에서 사용할 DB 관리자 계정이 준비되어 있어야 합니다.   


   
1. DB 관리자 계정 준비
Vault Database Secrets Engine은 데이터베이스 계정을 직접 생성, 삭제, 교체하기 때문에 Vault가 데이터베이스에 접속할 수 있는 전용 관리자 계정이 반드시 필요합니다.   
해당 계정은 다음과 같은 권한을 보유해야 합니다.   
•	ROLE 생성 및 삭제   
•	비밀번호 변경 (ALTER ROLE)   
•	권한 부여 및 회수 (GRANT / REVOKE)   
•	Dynamic 계정 회수 시 OWNED OBJECT 정리   

```
psql -U postgres -d mydb

CREATE ROLE vaultadmin WITH LOGIN PASSWORD 'vaultadminpw';
ALTER ROLE vaultadmin CREATEROLE CREATEDB;

```
2. Static Database 계정 생성
```
psql -U postgres -d mydb
CREATE ROLE "my-vault-app-static" WITH LOGIN PASSWORD 'initial-password';
```
3. Vault Policy 생성
```
vault policy write myapp-templated-policy - <<'EOF'
# 앱별 전용 KV 경로 (identity.entity.name 기반)
path "{{identity.entity.name}}-kv/data/*" {
  capabilities = ["read", "list"]
}
# Database Dynamic creds
path "{{identity.entity.name}}-database/creds/*" {
  capabilities = ["read", "list", "create", "update"]
}
# Database Static creds
path "{{identity.entity.name}}-database/static-creds/*" {
  capabilities = ["read", "list"]
}
# 토큰 갱신 권한
path "auth/token/renew-self" {
  capabilities = ["update"]
}
# 토큰 조회 권한
path "auth/token/lookup-self" {
  capabilities = ["read"]
}
# Lease 조회 권한 (Database Dynamic 시크릿 TTL 확인용)
path "sys/leases/lookup" {
  capabilities = ["update"]
}
# Lease 갱신 권한 (Database Dynamic 시크릿 갱신용)
path "sys/leases/renew" {
  capabilities = ["update"]
}
EOF
```

4. Identity Entity 생성 및 Policy 연결
```
vault write identity/entity 
name="my-vault-app" 
policies="myapp-templated-policy"

vault read -field=id identity/entity/name/my-vault-app
```


5.Entity Alias 생성 (AppRole ↔ Entity 연결)
```
vault read -field=accessor sys/auth/approle
vault write identity/entity-alias \
name="<ROLE_ID>" \
canonical_id="<ENTITY_ID>" \
mount_accessor="<APPROLE_ACCESSOR>"

```

6.Database Secrets Engine 설정
```
vault secrets enable -path=my-vault-app-database database
vault write my-vault-app-database/config/postgres-demo \
plugin_name=postgresql-database-plugin \
connection_url="postgresql://{{username}}:{{password}}@db_ip:5432/mydb?sslmode=disable" \
allowed_roles="db-demo-dynamic,db-demo-static" \
username="vaultadmin" \
password="vaultadminpw
```

7. Dynamic Role 생성
```
vault write my-vault-app-database/roles/db-demo-dynamic \
db_name=postgres-demo \
creation_statements="
CREATE ROLE "{{name}}" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}';
GRANT CONNECT ON DATABASE mydb TO "{{name}}";
GRANT USAGE ON SCHEMA public TO "{{name}}";
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO "{{name}}";
ALTER DEFAULT PRIVILEGES IN SCHEMA public
GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO "{{name}}";
" 
revocation_statements="
REASSIGN OWNED BY "{{name}}" TO vaultadmin;
DROP OWNED BY "{{name}}";
" 
default_ttl=1m 
max_ttl=24h

vault read my-vault-app-database/creds/db-demo-dynamic
```

8. Static Role 생성
```
vault write my-vault-app-database/static-roles/db-demo-static \
db_name=postgres-demo \
username="my-vault-app-static" \
rotation_period=1h \
rotation_statements="ALTER ROLE "my-vault-app-static" WITH PASSWORD '{{password}}';"

vault read my-vault-app-database/static-creds/db-demo-static
```

9. KV Secret Engine 설정
```
vault secrets enable -path=my-vault-app-kv kv-v2 

vault kv put my-vault-app-kv/database \
api_key="myapp-api-key-123456" \
database_url="postgresql://db_ip:5432/mydb" \
database_username="vaultadmin" \
database_password="vaultadminpw"
```

### 프로그래밍 언어 예제
```bash
# Java 예제
cd samples/java-pure-app
mvn clean package
java -jar target/vault-java-app.jar

# Python 예제
cd samples/python-app
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
python vault_app.py

# Spring Boot 웹 예제
cd samples/java-web-springboot-app
./gradlew bootRun
# 웹 브라우저에서 http://localhost:8080/vault-web 접속
```
