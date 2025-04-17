
# Oracle DB 23ai 'select ai' setup on a free docker container
This tutorial explains how to setup step-by-step **select ai** on a DB23ai free podman image. If you copy & past all the codes as-is, without changing anything you should arrive to a working env. The password used are already there, but if you want change, align the rest of the commands. The only credential you don't find in the documentation is related to the **OPENAI_API_KEY** that you have to get by yourself from [OpenAI Platform](https://platform.openai.com/settings/organization/api-keys)
To see the official and original documentation, please refer to:
* [Install DB23ai Utils](https://docs.oracle.com/en/database/oracle/oracle-database/23/sutil/installing-dbms_cloud.html)


### Create the container

```
podman run -d --name selectai -p 1521:1521 container-registry.oracle.com/database/free:latest
podman exec selectai ./setPassword.sh Welcome1234##
```

### Install the package DBMS_CLOUD

* connect to the instance:
```
podman exec -it selectai /bin/bash
```

* execute the installation
```
$ORACLE_HOME/perl/bin/perl $ORACLE_HOME/rdbms/admin/catcon.pl -u sys/Welcome1234## -force_pdb_mode 'READ WRITE' -b dbms_cloud_install -d $ORACLE_HOME/rdbms/admin/ -l /tmp catclouduser.sql

$ORACLE_HOME/perl/bin/perl $ORACLE_HOME/rdbms/admin/catcon.pl -u sys/Welcome1234## -force_pdb_mode 'READ WRITE' -b dbms_cloud_install -d $ORACLE_HOME/rdbms/admin/ -l /tmp dbms_cloud_install.sql
```

* check: you should see DBMS_CLOUD_AI* records:
```
podman exec -it selectai sqlplus '/ as sysdba'
select con_id, owner, object_name, status, sharing, oracle_maintained from cdb_objects where object_name like 'DBMS_CLOUD_AI%';
```

* download certificates from [OCI_object_bucket](https://objectstorage.us-phoenix-1.oraclecloud.com/p/KB63IAuDCGhz_azOVQ07Qa_mxL3bGrFh1dtsltreRJPbmb-VwsH2aQ4Pur2ADBMA/n/adwcdemo/b/CERTS/o/dbc_certs.tar)

* upload to the podman instance:
```
podman cp ./dbc_certs.tar selectai:/home/oracle/
```


* in the container with:
```
podman exec -it selectai /bin/bash
```
run:
```
cd  /home/oracle/
mkdir dbc
mv ./dbc_certs.tar ./dbc
cd dbc
tar -xvf ./dbc_certs.tar
```
NOTICE: You will use the tool orapki in: `/opt/oracle/product/23ai/dbhomeFree/bin/orapki`

* Run:
```
mkdir -p /home/oracle/wallets/ssl
cd /home/oracle/wallets/ssl

orapki wallet create -wallet . -pwd YourStrongPassword123 -auto_login
```


* create a script in `/home/oracle/wallets/ssl` named `wallets.sh` with this content:
```
#! /bin/bash
for i in $(ls /home/oracle/dbc/*cer)
do
orapki wallet add -wallet . -trusted_cert -cert $i -pwd YourStrongPassword123
done
```

* change to make executable and run it:
```
chmod 755 ./wallets.sh

./wallets.sh
```


* check:
```
cd /home/oracle/wallets/ssl
orapki wallet display -wallet .
```


* Change `sqlnet.ora` in this way:
```
podman exec -it selectai /bin/bash
cd /opt/oracle/product/23ai/dbhomeFree/network/admin
```

* with vi, add:
```
WALLET_LOCATION=(SOURCE=(METHOD=FILE)(METHOD_DATA=(DIRECTORY=/home/oracle/wallets/ssl)))
```

* modify this script and save as `/home/oracle/dbc/dbc_aces.sql`:
```
@$ORACLE_HOME/rdbms/admin/sqlsessstart.sql
 
-- you must not change the owner of the functionality to avoid future issues
define clouduser=C##CLOUD$SERVICE
 
-- CUSTOMER SPECIFIC SETUP, NEEDS TO BE PROVIDED BY THE CUSTOMER-- - SSL Wallet directory
define sslwalletdir=/home/oracle/wallets/ssl
 
---- UNCOMMENT AND SET THE PROXY SETTINGS VARIABLES IF YOUR ENVIRONMENT NEEDS PROXYS--
 
-- define proxy_uri=<your proxy URI address>
-- define proxy_host=<your proxy DNS name>
-- define proxy_low_port=<your_proxy_low_port>
-- define proxy_high_port=<your_proxy_high_port>
 
-- Create New ACL / ACE s
begin
-- Allow all hosts for HTTP/HTTP_PROXY
    dbms_network_acl_admin.append_host_ace(
        host =>'*',
        lower_port => 443,
        upper_port => 443,
        ace => xs$ace_type(
            privilege_list => xs$name_list('http', 'http_proxy'),
            principal_name => upper('&clouduser'),
            principal_type => xs_acl.ptype_db
            )
        );
--
-- UNCOMMENT THE PROXY SETTINGS SECTION IF YOUR ENVIRONMENT NEEDS PROXYS
--
-- Allow Proxy for HTTP/HTTP_PROXY
-- dbms_network_acl_admin.append_host_ace(
-- host =>'&proxy_host',
-- lower_port => &proxy_low_port,
-- upper_port => &proxy_high_port,
-- ace => xs$ace_type(
-- privilege_list => xs$name_list('http', 'http_proxy'),
-- principal_name => upper('&clouduser'),
-- principal_type => xs_acl.ptype_db));
--
-- END PROXY SECTION
--
 
-- Allow wallet access
    dbms_network_acl_admin.append_wallet_ace(
        wallet_path => 'file:&sslwalletdir',
        ace => xs$ace_type(
            privilege_list =>xs$name_list('use_client_certificates', 'use_passwords'),
            principal_name => upper('&clouduser'),
            principal_type => xs_acl.ptype_db));
end;
/
 
-- Setting SSL_WALLET database property
begin
    if sys_context('userenv', 'con_name') = 'CDB$ROOT' then
        execute immediate 'alter database property set ssl_wallet=''&sslwalletdir''';
--
-- UNCOMMENT THE FOLLOWING COMMAND IF YOU ARE USING A PROXY
--
--        execute immediate 'alter database property set http_proxy=''&proxy_uri''';
    end if;
end;
/
 
@$ORACLE_HOME/rdbms/admin/sqlsessend.sql
```

* execute (skip any questions if no proxy):
```
cd /home/oracle/dbc
sqlplus '/ as sysdba'
@@/home/oracle/dbc/dbc_aces.sql
```

* check the installation, creating  in `home/oracle/dbc/verify.sql`:
```
define clouduser=C##CLOUD$SERVICE
 
-- CUSTOMER SPECIFIC SETUP, NEEDS TO BE PROVIDED BY THE CUSTOMER
-- - SSL Wallet directory and password
define sslwalletdir=/home/oracle/wallets/ssl
define sslwalletpwd=YourStrongPassword123
 
-- In environments w/ a proxy, you need to set the proxy in the verification code
-- define proxy_uri=<your proxy URI address>
 
-- create and run this procedure as owner of the ACLs, which is the future owner
-- of DBMS_CLOUD
 
CREATE OR REPLACE PROCEDURE &clouduser..GET_PAGE(url IN VARCHAR2) AS
    request_context UTL_HTTP.REQUEST_CONTEXT_KEY;
    req UTL_HTTP.REQ;
    resp UTL_HTTP.RESP;
    data VARCHAR2(32767) default null;
    err_num NUMBER default 0;
    err_msg VARCHAR2(4000) default null;
 
BEGIN
 
-- Create a request context with its wallet and cookie table
    request_context := UTL_HTTP.CREATE_REQUEST_CONTEXT(
        wallet_path => 'file:&sslwalletdir',
        wallet_password => '&sslwalletpwd');
 
-- Make a HTTP request using the private wallet and cookie
-- table in the request context
 
-- uncomment if proxy is required
--    UTL_HTTP.SET_PROXY('&proxy_uri', NULL);
 
    req := UTL_HTTP.BEGIN_REQUEST(url => url,request_context => request_context);
    resp := UTL_HTTP.GET_RESPONSE(req);
 
DBMS_OUTPUT.PUT_LINE('valid response');
 
EXCEPTION
    WHEN OTHERS THEN
        err_num := SQLCODE;
        err_msg := SUBSTR(SQLERRM, 1, 3800);
        DBMS_OUTPUT.PUT_LINE('possibly raised PLSQL/SQL error: ' ||err_num||' - '||err_msg);
 
        UTL_HTTP.END_RESPONSE(resp);
        data := UTL_HTTP.GET_DETAILED_SQLERRM ;
        IF data IS NOT NULL THEN
            DBMS_OUTPUT.PUT_LINE('possibly raised HTML error: ' ||data);
        END IF;
END;
/
 
set serveroutput on
BEGIN
    &clouduser..GET_PAGE('https://objectstorage.eu-frankfurt-1.oraclecloud.com');
END;
/
 
set serveroutput off
drop procedure &clouduser..GET_PAGE;
```

* run it:
```
sqlplus '/ as sysdba'
 @@/home/oracle/dbc/verify.sql
```


### Setup for use the DBMS_CLOUD_AI package
* create user
```
podman exec -it selectai sqlplus '/ as sysdba'
```

* run in `sqlplus`: 
```
alter system set vector_memory_size=512M scope=spfile;

alter session set container=FREEPDB1;

CREATE USER "VECTOR" IDENTIFIED BY vector
    DEFAULT TABLESPACE "USERS"
    TEMPORARY TABLESPACE "TEMP";
GRANT "DB_DEVELOPER_ROLE" TO "VECTOR";
ALTER USER "VECTOR" DEFAULT ROLE ALL;
ALTER USER "VECTOR" QUOTA UNLIMITED ON USERS;
EXIT;
```


* grant privileges to VECTOR user for `DBMS_CLOUD` with `sqlplus`:

```
alter session set container=FREEPDB1;

set verify off
 
-- target sample user
define username='VECTOR'
 
REM the following are minimal privileges to use DBMS_CLOUD
REM - this script assumes core privileges
REM - CREATE SESSIONREM - Tablespace quota on the default tablespace for a user
 
REM for creation of external tables, e.g. DBMS_CLOUD.CREATE_EXTERNAL_TABLE()
grant CREATE TABLE to &username;
 
REM for using COPY_DATA()
REM - Any log and bad file information is written into this directory
grant read, write on directory DATA_PUMP_DIR to &username;
 
REM grant as you see fit
grant EXECUTE on dbms_cloud to &username;
grant EXECUTE on dbms_cloud_pipeline to &username;
grant EXECUTE on dbms_cloud_repo to &username;
grant EXECUTE on dbms_cloud_notification to &username;
```


* make an ACL for any external URL connection (as sysdba in the same connection). NOTICE:  you can limit to the single provider, not recommended in production.
  
```
alter session set container=FREEPDB1;

BEGIN
DBMS_NETWORK_ACL_ADMIN.APPEND_HOST_ACE (
  HOST => '*',
  LOWER_PORT => 443,
  UPPER_PORT => 443,
  ACE => xs$ace_type(
  PRIVILEGE_LIST => xs$name_list('http'),
  PRINCIPAL_NAME => 'VECTOR',
  PRINCIPAL_TYPE => xs_acl.ptype_db));
END;
/
```
* grant execution to the package to the user `VECTOR`:

```
GRANT EXECUTE ON DBMS_CLOUD_AI TO VECTOR;
GRANT EXECUTE ON DBMS_CLOUD TO VECTOR;
```

* create the `select ai` profile to use the command:
  
```
podman exec -it selectai sqlplus vector/vector@localhost:1521/FREEPDB1
```

* execute in `sqlplus`. Change `'sk-proj-ADYn3TYr83fD029S....'` with your OpenAI API KEY:

```
BEGIN
DBMS_CLOUD.create_credential(
  credential_name =>'OPENAI_CRED',
  username => 'OPENAI',
  password => 'sk-proj-ADYn3TYr83fD029S....');
END;
/
```

* create the profile, just to check if `select ai` is available:

```
BEGIN
DBMS_CLOUD_AI.create_profile(
  profile_name => 'openai_gpt',
  attributes =>
   ' { "provider": "openai",
       "credential_name": "OPENAI_CRED"
    }'
);
END;
/
```

### Use Select AI.

Before any session you start connecting with `sqlplus` as `vector`:

```
podman exec -it selectai sqlplus vector/vector@localhost:1521/FREEPDB1
```

* select the profile:
```
BEGIN
  DBMS_CLOUD_AI.set_profile(
    profile_name => 'openai_gpt'
  );
END;
/
```

* just to check if `select ai` is available run command like:
```
select ai who is George Washington;

```
