## FoxLake

### 构建运行

```shell
<option name="ALTERNATIVE_JRE_PATH" value="/Library/Java/JavaVirtualMachines/amazon-corretto-17.jdk/Contents/Home" />

docker start e6356a4b34fa #mysql
docker start 9bd925bf39c9 #minio
mysql -h127.0.0.1 -uroot -proot -P16345 -Ac #connect to MySQL server

git submodule update --init --recursive
mvn -T1C -Dmaven.test.skip=true install

# run foxlake in IDEA

mysql -h127.0.0.1 -P11288 -ufoxlake_root -pfoxlake2023 -Ac
```



### 导入数据

```shell
create table t0 (
    id BIGINT NOT NULL,
    text VARCHAR(255) NOT NULL,
    PRIMARY KEY (id)
) engine = 'columnar@default';

explain COPY INTO t0
FROM 'minio://127.0.0.1:9000/foxlakebucket/sample/data/'
CREDENTIALS = (ACCESS_KEY_ID='ROOTUSER', SECRET_ACCESS_KEY='CHANGEME123')
FILE_FORMAT = (TYPE='CSV', FIELD_DELIMITER='|', RECORD_DELIMITER='\n');


CREATE TABLE IF NOT EXISTS test_null (id BIGINT, s VARCHAR(100), i INTEGER) engine = 'columnar@default';

COPY INTO test_null
FROM 'minio://127.0.0.1:9000/foxlakebucket/sample/test_null/t3.csv'
CREDENTIALS = (ACCESS_KEY_ID='ROOTUSER', SECRET_ACCESS_KEY='CHANGEME123')
FILE_FORMAT = (TYPE='CSV', FIELD_DELIMITER='|', RECORD_DELIMITER='\n');
```



 

### select主流程

```

input -> ...net... -> FrontendCommandHandler.handle() < case:COM_QUERY > -> FrontendConnection.query(data)

-> ServerQueryHandler.queryRaw() -> ServerQueryHandler.executeStatement() < case ServerParse.SELECT >

-> SelectHandler.handle() -> ServerConnection.execute(sql, hasMore) -> ServerConnection.execute()

-> ServerConnection.innerExecute() -> TPreparedStatement.execute() -> TStatement.executeInternal()

-> TPreparedStatement.executeSQL() -> TConnection.executeSQL() 

-> TConnection.executeQuery (生成plan) -> PlanExecutor.execute(plan)

-> PlanExecutor.execByExecPlanNodeByOne(plan) -> ExecutorHelper.execute(plan)

-> ExecutorHelper.executeByCursor(plan) (即火山模型) / .executeCluster(plan) (即MPP)

-> IExecutor.execByExecPlanNode(plan)




```







