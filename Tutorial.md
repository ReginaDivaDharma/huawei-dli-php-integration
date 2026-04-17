# Dashboard Visibility: PHP to Data Lake Insight (DLI)

A guide for connecting a PHP application to Huawei Cloud Data Lake Insight (DLI) via Apache Kyuubi, using the Hive ODBC driver.

---

## Architecture Overview

<img width="1320" height="745" alt="image" src="https://github.com/user-attachments/assets/d0f8414a-578f-4ad4-8a07-be6380191703" />


The PHP application connects to DLI through a Hive-compatible ODBC driver. This driver establishes a HiveServer2 (Thrift) session with Apache Kyuubi, which acts as a stable gateway to DLI. Because DLI is serverless, it does not expose a persistent database endpoint , Kyuubi provides that stable endpoint (Port 10009) so PHP can trigger Spark SQL jobs without needing a local Spark environment.

---

## Table of Contents

- [Prerequisites](#prerequisites)
- [Part 1 — Set Up Apache Kyuubi](#part-1--set-up-apache-kyuubi)
- [Part 2 — Connect via ODBC (PHP)](#part-2--connect-via-odbc-php) 

---

## Prerequisites

Before starting, make sure the following are in place:

- **DLI is ready** — you have an active DLI SQL queue and some dummy/test data loaded. A NAT Gateway must also be configured so that DLI can be accessed from VMs on other networks. *(See: [DLI + NAT Gateway Setup Guide](#) — add your link here)*
- **Java 1.8.0** is installed on the server that will run Kyuubi.

Verify your Java version:

```bash
java -version
# Expected: openjdk version "1.8.0_xxx" or similar
```

---

## Part 1 — Set Up Apache Kyuubi

### Step 1 — Download Apache Kyuubi

Download **Kyuubi version 1.7.0** from the official Apache release page and extract it to your server.

```bash
# Example — adjust the download URL if needed
wget https://downloads.apache.org/kyuubi/kyuubi-1.7.0/apache-kyuubi-1.7.0-bin.tgz
tar -xzf apache-kyuubi-1.7.0-bin.tgz
cd apache-kyuubi-1.7.0-bin
```

### Step 2 — Download the DLI Kyuubi Driver

Go to the Huawei DLI SDK page and download the DLI Kyuubi driver:

**URL:** https://uquery-sdk.obs-website.cn-north-1.myhwclouds.com/?locale=en-us

Look for and download:
- `dli-kyuubi-1.1.1`
- *(optionally verify with `dli-kyuubi-1.1.1.sha256`)*

### Step 3 — Place the Driver in Kyuubi

Copy the downloaded driver JAR into Kyuubi's JDBC engines directory. This tells Kyuubi how to communicate with DLI.

```bash
cp dli-kyuubi-1.1.1 /path/to/apache-kyuubi-1.7.0-bin/externals/engines/jdbc/
```

> **Note:** Replace `/path/to/` with the actual path where you extracted Kyuubi.

### Step 4 — Configure Kyuubi
 
Edit the Kyuubi configuration file to point it at your DLI endpoint.
 
```bash
nano /opt/kyuubi/conf/kyuubi-defaults.conf
```
 
Set the following properties. The table below explains each one — replace the placeholder values with your own.
 
#### Frontend (Server Binding)
 
| Property | Example Value | Description |
|---|---|---|
| `kyuubi.frontend.bind.host` | `0.0.0.0` | Listen on all network interfaces |
| `kyuubi.frontend.protocols` | `THRIFT_BINARY,mysql` | Enable Thrift Binary (for ODBC/JDBC) and MySQL protocol |
| `kyuubi.frontend.thrift.binary.bind.port` | `10009` | Port for HiveServer2/Thrift connections |
| `kyuubi.frontend.rest.bind.port` | `10099` | Port for REST API |
 
#### Engine Type
 
| Property | Example Value | Description |
|---|---|---|
| `kyuubi.engine.type` | `jdbc` | Use JDBC as the backend engine type |
| `kyuubi.engine.jdbc.type` | `dli` | Specify DLI as the JDBC engine |
 
#### DLI JDBC Connection
 
| Property | Example Value | Description |
|---|---|---|
| `kyuubi.engine.jdbc.driver.class` | `com.huawei.dli.jdbc.DliDriver` | DLI JDBC driver class |
| `kyuubi.engine.jdbc.connection.url` | `jdbc:dli://<dli-endpoint>/<project_id>` | Your DLI JDBC endpoint URL |
| `kyuubi.engine.jdbc.session.initialize.sql` | `select 1` | Validation query run on session start |
 
#### DLI Authentication & Target
 
| Property | Example Value | Description |
|---|---|---|
| `kyuubi.engine.dli.jdbc.connection.region` | `ap-southeast-4` | Your Huawei Cloud region |
| `kyuubi.engine.dli.jdbc.connection.ak` | `<your-access-key>` | Huawei Cloud Access Key (AK) |
| `kyuubi.engine.dli.jdbc.connection.sk` | `<your-secret-key>` | Huawei Cloud Secret Key (SK) |
| `kyuubi.engine.dli.jdbc.connection.queue` | `queue_spark` | DLI SQL queue name |
| `kyuubi.engine.dli.jdbc.connection.database` | `dummy_data` | Default DLI database to connect to |
| `kyuubi.engine.dli.jdbc.connection.project` | `<your-project-id>` | Huawei Cloud project ID |
 
#### Performance & Behaviour Tuning
 
| Property | Example Value | Description |
|---|---|---|
| `kyuubi.engine.dli.result.line.num.limit` | `50000` | Max rows returned per query |
| `kyuubi.engine.dli.small.file.merge` | `false` | Disable small file merging |
| `kyuubi.engine.dli.bi.type` | `tableau` | BI tool compatibility mode |
| `kyuubi.engine.dli.schema.show.name` | `false` | Whether to show schema name in results |
| `kyuubi.engine.dli.result.cache.enable` | `true` | Enable result caching |
| `kyuubi.engine.dli.boolean.type.to.int` | `false` | Convert boolean values to int |
| `kyuubi.operation.incremental.collect` | `false` | Disable incremental result collection |
| `kyuubi.engine.jdbc.memory` | `5g` | Memory allocated to the JDBC engine |
 
> **Security note:** Avoid committing real AK/SK values to a public repository. Use environment variables or a secrets manager in production.

### Step 5 — Start Kyuubi

```bash
cd /path/to/apache-kyuubi-1.7.0-bin
bin/kyuubi start
```

<img width="1013" height="275" alt="image" src="https://github.com/user-attachments/assets/203a0043-58e8-4a30-aff1-5da2b800ba53" />

You should be seeing this


Check the logs to confirm Kyuubi started and is listening on port 10009:

```bash
tail -f logs/kyuubi-*.log
# Look for: "FrontendService started on ..." or "ThriftBinaryFrontendService"
```

### Step 6 — Verify the Connection

You can do a quick check using the Kyuubi JDBC CLI or `beeline`:

```bash
bin/beeline -u "jdbc:hive2://<kyuubi-host>:10009/" -n <your-username>
# Try: SHOW DATABASES;
```

For Example in this tutorial i will run this command on the terminal to see if my Kyuubi can actually connect DLI 
```bash
/opt/kyuubi/bin/beeline -u "jdbc:hive2://localhost:10009/default" -n root
SHOW DATABASES;
```
Here's the output : 
<img width="1016" height="207" alt="image" src="https://github.com/user-attachments/assets/c8375e59-5bd1-4b3b-a32d-5aeda7337472" />

If you see your DLI databases listed, Kyuubi is successfully connected to DLI. ✅

---

## Part 2 — Connect via ODBC (PHP)
 
This part covers connecting PHP to Kyuubi using unixODBC and the Cloudera Hive ODBC driver.
 
```
PHP → unixODBC → Hive ODBC Driver → Kyuubi (10009) → DLI
```
 
### Step 1 — Install the Hive ODBC Driver
 
Install the Cloudera Hive ODBC driver RPM package:
 
```bash
yum install -y ClouderaHiveODBC-2.6.1.1001-1.x86_64.rpm
```
 
### Step 2 — Find the Driver Path
 
After installation, confirm the location of the driver shared library:
 
```bash
find /opt -name "*hiveodbc*.so"
```
 
You should see a path similar to `/opt/cloudera/hiveodbc/lib/64/libclouderahiveodbc64.so`. Take note of this path — you will need it in the next step.
 
### Step 3 — Register the Driver in `odbcinst.ini`
 
Edit the system ODBC driver registry to register the Hive driver:
 
```bash
vi /etc/odbcinst.ini
```
 
Add the following block:
 
```ini
[Cloudera Hive ODBC Driver 64-bit]
Driver=/opt/cloudera/hiveodbc/lib/64/libclouderahiveodbc64.so
```
 
> The section name in brackets (`[Cloudera Hive ODBC Driver 64-bit]`) is the driver label. It must match exactly what you reference in `odbc.ini` in the next step.
 
### Step 4 — Configure the DSN in `odbc.ini`
 
Create a Data Source Name (DSN) that points to your Kyuubi instance:
 
```bash
vi /etc/odbc.ini
```
 
Add the following block:
 
```ini
[kyuubi_dsn]
Driver=Cloudera Hive ODBC Driver 64-bit
Host=127.0.0.1
Port=10009
Schema=default
HiveServerType=2
AuthMech=2
UID=root
ThriftTransport=1
SSL=0
```
 
The key properties are explained below:
 
| Property | Value | Description |
|---|---|---|
| `Driver` | `Cloudera Hive ODBC Driver 64-bit` | Must match the label in `odbcinst.ini` |
| `Host` | `127.0.0.1` | Kyuubi is running on the same server |
| `Port` | `10009` | Kyuubi Thrift Binary port |
| `Schema` | `default` | Default DLI database/schema to use |
| `HiveServerType` | `2` | HiveServer2 mode (required for Kyuubi) |
| `AuthMech` | `2` | Username-only authentication |
| `UID` | `root` | Username passed to Kyuubi |
| `ThriftTransport` | `1` | Binary transport (matches Kyuubi's Thrift Binary frontend) |
| `SSL` | `0` | SSL disabled (enable if your setup requires it) |
 
### Step 5 — Install the PHP ODBC Extension
 
Install the `php-odbc` package so PHP can make ODBC connections:
 
```bash
yum install -y php-odbc
```
 
Verify the extension is loaded:
 
```bash
php -m | grep -i odbc
# Expected output: odbc
```
 
### Step 6 — Test the ODBC Connection
 
Before testing from PHP, verify the DSN works at the system level using `isql`:
 
```bash
isql -v kyuubi_dsn
```
 
If successful, you will be dropped into an interactive SQL prompt connected to DLI through Kyuubi. Try a quick query:
 
```sql
SHOW DATABASES;
```
 
If this returns your DLI databases, the full ODBC stack is working correctly. ✅ You are now ready to use `odbc_connect()` from PHP to query DLI.
 
---
 
## Troubleshooting
 
| Symptom | Likely Cause | Fix |
|---|---|---|
| Kyuubi fails to start | Java not found or wrong version | Confirm `java -version` returns 1.8.0 |
| `ClassNotFoundException` in logs | Driver JAR not in the right folder | Check `externals/engines/jdbc/` for the DLI JAR |
| Connection refused on port 10009 | Kyuubi not running, or firewall blocking | Check logs; open port 10009 in security group |
| DLI query fails with auth error | Wrong AK/SK in config | Re-check `kyuubi-defaults.conf` credentials |
 
---
 
## References
 
- [Apache Kyuubi Documentation](https://kyuubi.readthedocs.io/)
- [Huawei DLI Product Page](https://www.huaweicloud.com/product/dli.html)
- [DLI SDK Downloads](https://uquery-sdk.obs-website.cn-north-1.myhwclouds.com/?locale=en-us)
- [DLI + NAT Gateway Setup](#) *(add your link here)*
