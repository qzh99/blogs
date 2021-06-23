## 一、windows 安装 SQLite

### 1、下载资源包

https://www.sqlite.org/download.html

sqlite-dll-win64-x64-3360000.zip

sqlite-tools-win32-x86-3360000.zip

### 2、解压

解压两个安装包，复制所有文件到自定义的目录中，比如`D:\soft\sqlite3`

```
sqldiff.exe --比较两个SQLite数据库的差异
sqlite3.dll --Windows平台的SQLite数据库最小依赖，有了这个文件就可以通过API来创建、读写表
sqlite3.exe --windows平台SQLite命令行操作程序
sqlite3_analyzer.exe --数据库分析工具
```

## 二、操作例子

> SQLite支持大部分SQL92语法，使用时参考SQL92语法和SQLite方言，文档地址：https://www.sqlite.org/docs.html

#### 1、 创建或连接到数据库

在`D:\soft\sqlite3`路径中执行cmd命令，切换到控制台；执行`sqlite3`命令切换到sqlite控制台

执行`.open test.db`命令，如果test.db文件存在则会打开数据库，否则则会创建。

```sqlite
D:\soft\sqlite3>sqlite3
SQLite version 3.35.5 2021-04-19 18:32:05
Enter ".help" for usage hints.
Connected to a transient in-memory database.
Use ".open FILENAME" to reopen on a persistent database.
sqlite> .open test.db
sqlite>
```

此时已经创建了test.db文件，可以用`.databases`命令查看

```sqlite
sqlite> .databases
main: D:\soft\sqlite3\test.db r/w
sqlite>
```

#### 2、 创建表

```sqlite
create table animals(id int primary key , name varchar);
```

```sqlite
sqlite> create table animals(id INTEGER PRIMARY KEY, name varchar);
sqlite> .tables
animals
```



####  3、 插入数据

```sqlite
insert into animals(id,name) values(null,'cat');
insert into animals(id,name) values(null,'dog');
insert into animals(id,name) values(null,'pig');
```

#### 4、 查询数据

```sqlite
sqlite> select * from animals;
1|cat
2|dog
3|pig
```

执行命令格式化输出

```sqlite
.header on
.mode column
.timer on
```

结果

```
sqlite> select * from animals;
id  name
--  ----
1   cat
2   dog
3   pig
Run Time: real 0.007 user 0.000000 sys 0.000000
```





## 三、Java API操作SQLite

#### 1、导入jar包

> SQLite 驱动地址 https://github.com/xerial/sqlite-jdbc

```xml
<dependency>
    <groupId>org.xerial</groupId>
    <artifactId>sqlite-jdbc</artifactId>
    <version>3.34.0</version>
</dependency>
```

> note：sqlite-jdbc-3.34.0.jar驱动包中native目录下自带了各平台SQLite的依赖库，所以不需要再安装依赖库，可以之家使用。

#### 2、加载驱动、获取连接

> 驱动加载动作是在类org.sqlite.JDBC加载时通过静态代码块完成的，更多案例参考：https://www.runoob.com/sqlite/sqlite-java.html

```java
import java.sql.*;

public class SQLiteJDBC
{
  public static void main( String args[] )
  {
    Connection c = null;
    try {
      Class.forName("org.sqlite.JDBC");
      c = DriverManager.getConnection("jdbc:sqlite:D:\soft\sqlite3\test.db");
    } catch ( Exception e ) {
      System.err.println( e.getClass().getName() + ": " + e.getMessage() );
      System.exit(0);
    }
    System.out.println("Opened database successfully");
  }
}
```

#### 一个实际的应用

> 以下代码来自华为数据接入服务，dis-agent，仅供参考

```java
package com.huaweicloud.dis.agent.tailing.checkpoints;

import com.google.common.annotations.VisibleForTesting;
import com.google.common.base.Preconditions;
import com.huaweicloud.dis.agent.AgentContext;
import com.huaweicloud.dis.agent.tailing.FileFlow;
import com.huaweicloud.dis.agent.tailing.TrackedFile;
import java.io.File;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.Paths;
import java.nio.file.attribute.FileAttribute;
import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.sql.SQLException;
import java.sql.Statement;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.Map;
import java.util.concurrent.locks.ReentrantLock;
import org.apache.commons.codec.digest.DigestUtils;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class SQLiteFileCheckpointStore implements FileCheckpointStore {
  public String toString() {
    return "SQLiteFileCheckpointStore(dbFile=" + this.dbFile + ", lock=" + this.lock + ", dbQueryTimeoutSeconds=" + this.dbQueryTimeoutSeconds + ", dbConnectionTimeoutSeconds=" + this.dbConnectionTimeoutSeconds + ")";
  }
  
  private static final Logger LOGGER = LoggerFactory.getLogger(SQLiteFileCheckpointStore.class);
  
  static final String DEFAULT_CHECKPOINTS_DIR = "conf" + File.separator;
  
  static final String DEFAULT_CHECKPOINTS_FILE = "since.db";
  
  private static final int DEFAULT_DB_CONNECTION_TIMEOUT_SECONDS = 5;
  
  private static final int DEFAULT_DB_QUERY_TIMEOUT_SECONDS = 30;
  
  private final Path dbFile;
  
  private ReentrantLock lock = new ReentrantLock();
  
  @VisibleForTesting
  final AgentContext agentContext;
  
  @VisibleForTesting
  Connection connection;
  
  private final int dbQueryTimeoutSeconds;
  
  private final int dbConnectionTimeoutSeconds;
  
  public SQLiteFileCheckpointStore(AgentContext agentContext) {
    this.agentContext = agentContext;
    Path configCheckPath = agentContext.checkpointFile();
    if (configCheckPath == null) {
      String dbFileStr = DEFAULT_CHECKPOINTS_DIR + this.agentContext.getAgentName() + "_" + "since.db";
      if (!Files.exists(Paths.get(dbFileStr, new String[0]), new java.nio.file.LinkOption[0]) && "dis-agent".equals(this.agentContext.getAgentName()) && 
        Files.exists(Paths.get(DEFAULT_CHECKPOINTS_DIR + "since.db", new String[0]), new java.nio.file.LinkOption[0])) {
        LOGGER.info("Will use compatible checkpoint file : " + DEFAULT_CHECKPOINTS_DIR + "since.db");
        this.dbFile = Paths.get(DEFAULT_CHECKPOINTS_DIR + "since.db", new String[0]);
      } else {
        LOGGER.info("Will use default checkpoint file : {}", dbFileStr);
        this.dbFile = Paths.get(dbFileStr, new String[0]);
      } 
    } else {
      LOGGER.info("Will use custom checkpoint file : {}", configCheckPath.toString());
      this.dbFile = configCheckPath;
    } 
    this
      .dbQueryTimeoutSeconds = this.agentContext.readInteger("checkpoints.queryTimeoutSeconds", Integer.valueOf(30)).intValue();
    this.dbConnectionTimeoutSeconds = this.agentContext.readInteger("checkpoints.connectionTimeoutSeconds", 
        Integer.valueOf(5)).intValue();
    connect();
    deleteOldData();
  }
  
  private synchronized boolean isConnected() {
    return (this.connection != null);
  }
  
  public synchronized void close() {
    if (isConnected())
      try {
        LOGGER.debug("Closing connection to database {}...", this.dbFile);
        this.connection.close();
      } catch (SQLException e) {
        LOGGER.error("Failed to cleanly close the database {}", this.dbFile, e);
      }  
    this.connection = null;
  }
  
  private synchronized void connect() {
    if (!isConnected()) {
      try {
        Class.forName("org.sqlite.JDBC");
      } catch (ClassNotFoundException e) {
        throw new RuntimeException("Failed to load SQLite driver.", e);
      } 
      try {
        LOGGER.debug("Connecting to database {}...", this.dbFile);
        if (!Files.isDirectory(this.dbFile.getParent(), new java.nio.file.LinkOption[0]))
          Files.createDirectories(this.dbFile.getParent(), (FileAttribute<?>[])new FileAttribute[0]); 
        String connectionString = String.format("jdbc:sqlite:%s", new Object[] { this.dbFile.toString() });
        this.connection = DriverManager.getConnection(connectionString);
        this.connection.setAutoCommit(false);
      } catch (SQLException|java.io.IOException e) {
        throw new RuntimeException("Failed to create or connect to the checkpoint database.", e);
      } 
      try {
        Statement statement = this.connection.createStatement();
        try {
          statement.executeUpdate("create table if not exists FILE_CHECKPOINTS(       flow text,       path text,       fileId text,       lastModifiedTime bigint,       size bigint,       offset bigint,       headerLength int,       headerString text,       lastUpdated datetime,       primary key (flow, fileId))");
        } finally {
          if (Collections.<Statement>singletonList(statement).get(0) != null)
            statement.close(); 
        } 
      } catch (SQLException e) {
        throw new RuntimeException("Failed to configure the checkpoint database.", e);
      } 
    } 
  }
  
  protected boolean ensureConnected() {
    if (isConnected())
      return true; 
    try {
      connect();
      return true;
    } catch (Exception e) {
      LOGGER.error("Failed to open a connection to the checkpoint database.", e);
      return false;
    } 
  }
  
  public synchronized FileCheckpoint saveCheckpoint(TrackedFile file, long offset) {
    Preconditions.checkNotNull(file);
    return createOrUpdateCheckpoint(new FileCheckpoint(file, offset));
  }
  
  public synchronized boolean saveCheckpointBatchAfterSend(List<FileCheckpoint> fileCheckpoints) {
    return createOrUpdateCheckpointBatchByIncrement(fileCheckpoints);
  }
  
  private FileCheckpoint createOrUpdateCheckpoint(FileCheckpoint cp) {
    if (!ensureConnected())
      return null; 
    try {
      PreparedStatement update = this.connection.prepareStatement("update FILE_CHECKPOINTS set path=?, offset=?, lastModifiedTime=?, size=?, headerLength=?, headerString=?, lastUpdated=strftime('%Y-%m-%d %H:%M:%f', 'now') where flow=? and fileId=?");
      try {
        update.setString(1, cp.getFile().getPath().toAbsolutePath().toString());
        update.setLong(2, cp.getOffset());
        update.setLong(3, cp.getFile().getLastModifiedTime());
        update.setLong(4, cp.getFile().getSize());
        update.setInt(5, cp.getFile().getHeaderBytesLength());
        update.setString(6, 
            (cp.getFile().getHeaderBytes() == null) ? cp.getFile().getSha256HeaderStr() : 
            DigestUtils.sha256Hex(cp.getFile().getHeaderBytes()));
        update.setString(7, cp.getFile().getFlow().getId());
        update.setString(8, cp.getFile().getId().toString());
        int affected = update.executeUpdate();
        if (affected == 0) {
          PreparedStatement insert = this.connection.prepareStatement("insert or ignore into FILE_CHECKPOINTS values(?, ?, ?, ?, ?, ?, ?, ?, strftime('%Y-%m-%d %H:%M:%f', 'now'))");
          try {
            insert.setString(1, cp.getFile().getFlow().getId());
            insert.setString(2, cp.getFile().getPath().toAbsolutePath().toString());
            insert.setString(3, cp.getFile().getId().toString());
            insert.setLong(4, cp.getFile().getLastModifiedTime());
            insert.setLong(5, cp.getFile().getSize());
            insert.setLong(6, cp.getOffset());
            insert.setInt(7, cp.getFile().getHeaderBytesLength());
            insert.setString(8, 
                (cp.getFile().getHeaderBytes() == null) ? cp.getFile().getSha256HeaderStr() : 
                DigestUtils.sha256Hex(cp.getFile().getHeaderBytes()));
            affected = insert.executeUpdate();
            if (affected == 1) {
              LOGGER.trace("Created new database checkpoint: {}@{}", cp.getFile(), Long.valueOf(cp.getOffset()));
            } else {
              LOGGER.error("Did not update or create checkpoint because of race condition: {}@{}", cp
                  .getFile(), 
                  Long.valueOf(cp.getOffset()));
              throw new RuntimeException("Race condition detected when setting checkpoint for file: " + cp
                  .getFile().getPath());
            } 
          } finally {
            if (Collections.<PreparedStatement>singletonList(insert).get(0) != null)
              insert.close(); 
          } 
        } else {
          LOGGER.trace("Updated database checkpoint: {}@{}", cp.getFile(), Long.valueOf(cp.getOffset()));
        } 
        this.connection.commit();
        return cp;
      } finally {
        if (Collections.<PreparedStatement>singletonList(update).get(0) != null)
          update.close(); 
      } 
    } catch (SQLException e) {
      LOGGER.error("Failed to create the checkpoint {}@{} in database {}", new Object[] { cp.getFile(), Long.valueOf(cp.getOffset()), this.dbFile, e });
      try {
        this.connection.rollback();
      } catch (SQLException e2) {
        LOGGER.error("Failed to rollback checkpointing transaction: {}@{}", cp.getFile(), Long.valueOf(cp.getOffset()));
        LOGGER.info("Reinitializing connection to database {}", this.dbFile);
        close();
      } 
      return null;
    } 
  }
  
  private boolean createOrUpdateCheckpointBatchByIncrement(List<FileCheckpoint> fileCheckpoints) {
    if (!ensureConnected())
      return false; 
    try {
      this.lock.lock();
      PreparedStatement update = this.connection.prepareStatement("update FILE_CHECKPOINTS set path=?, offset=(select max(?, offset) from FILE_CHECKPOINTS where flow=? and fileId=?), lastModifiedTime=?, size=?, headerLength=?, headerString=?, lastUpdated=strftime('%Y-%m-%d %H:%M:%f', 'now') where flow=? and fileId=?");
    } catch (SQLException e) {
      LOGGER.error("Failed to create the checkpoints {} in database {}", new Object[] { fileCheckpoints, this.dbFile, e });
      try {
        this.connection.rollback();
      } catch (SQLException e2) {
        LOGGER.error("Failed to rollback checkpointing transaction: {}", fileCheckpoints);
        LOGGER.info("Reinitializing connection to database {}", this.dbFile);
        close();
      } 
      return false;
    } finally {
      this.lock.unlock();
    } 
  }
  
  public long getOffsetForFileID(FileFlow<?> flow, String fileID) {
    Preconditions.checkNotNull(flow);
    Preconditions.checkNotNull(fileID);
    if (!ensureConnected())
      return 0L; 
    try {
      PreparedStatement statement = this.connection.prepareStatement("select offset from FILE_CHECKPOINTS where flow=? and fileId=?");
      try {
        statement.setString(1, flow.getId());
        statement.setString(2, fileID);
        statement.setMaxRows(1);
        ResultSet result = statement.executeQuery();
      } finally {
        if (Collections.<PreparedStatement>singletonList(statement).get(0) != null)
          statement.close(); 
      } 
    } catch (SQLException e) {
      LOGGER.error("Failed when getting offset for fileId {} in flow {}", new Object[] { fileID, flow.getId(), e });
      return 0L;
    } 
  }
  
  @Deprecated
  public FileCheckpoint getCheckpointForFlow(FileFlow<?> flow) {
    Preconditions.checkNotNull(flow);
    if (!ensureConnected())
      return null; 
    try {
      PreparedStatement statement = this.connection.prepareStatement("select path, fileId, lastModifiedTime, size, offset, headerLength, headerString from FILE_CHECKPOINTS where flow=? order by lastUpdated desc limit 1");
      try {
        statement.setString(1, flow.getId());
        statement.setMaxRows(1);
        ResultSet result = statement.executeQuery();
      } finally {
        if (Collections.<PreparedStatement>singletonList(statement).get(0) != null)
          statement.close(); 
      } 
    } catch (SQLException e) {
      LOGGER.error("Failed when getting checkpoint for flow {}", flow.getId(), e);
      return null;
    } 
  }
  
  public List<Map<String, Object>> dumpCheckpoints() {
    if (!ensureConnected())
      return Collections.emptyList(); 
    try {
      PreparedStatement statement = this.connection.prepareStatement("select * from FILE_CHECKPOINTS order by lastUpdated desc");
      try {
        ResultSet result = statement.executeQuery();
      } finally {
        if (Collections.<PreparedStatement>singletonList(statement).get(0) != null)
          statement.close(); 
      } 
    } catch (SQLException e) {
      LOGGER.error("Failed when dumping checkpoints from db.", e);
      return Collections.emptyList();
    } 
  }
  
  @VisibleForTesting
  synchronized void deleteOldData() {
    if (!ensureConnected())
      return; 
    try {
      String query = String.format("delete from FILE_CHECKPOINTS where datetime(lastUpdated, '+%d days') < CURRENT_TIMESTAMP", new Object[] { Integer.valueOf(this.agentContext.checkpointTimeToLiveDays()) });
      PreparedStatement statement = this.connection.prepareStatement(query);
      try {
        statement.setQueryTimeout(this.dbQueryTimeoutSeconds);
        int affectedCount = statement.executeUpdate();
        this.connection.commit();
        LOGGER.info("Deleted {} old checkpoints.", Integer.valueOf(affectedCount));
      } finally {
        if (Collections.<PreparedStatement>singletonList(statement).get(0) != null)
          statement.close(); 
      } 
    } catch (SQLException e) {
      try {
        this.connection.rollback();
      } catch (SQLException e2) {
        LOGGER.error("Failed to rollback cleanup transaction.", e2);
        LOGGER.info("Reinitializing connection to database {}", this.dbFile);
        close();
      } 
      LOGGER.error("Failed to delete old checkpoints.", e);
    } 
  }
  
  private boolean updateFileId(TrackedFile trackedFile) {
    if (!ensureConnected())
      return false; 
    try {
      PreparedStatement update = this.connection.prepareStatement("update FILE_CHECKPOINTS set fileId=?, lastModifiedTime=? lastUpdated=strftime('%Y-%m-%d %H:%M:%f', 'now') where flow=? and path=?");
      try {
        update.setString(1, trackedFile.getId().toString());
        update.setLong(2, trackedFile.getLastModifiedTime());
        update.setString(3, trackedFile.getFlow().getId());
        update.setString(4, trackedFile.getPath().toAbsolutePath().toString());
        int affected = update.executeUpdate();
        if (affected == 0) {
          boolean bool = false;
          if (Collections.<PreparedStatement>singletonList(update).get(0) != null)
            update.close(); 
        } 
        LOGGER.trace("Updated database fileId: {}@{}", trackedFile, trackedFile.getId().toString());
        this.connection.commit();
        return true;
      } finally {
        if (Collections.<PreparedStatement>singletonList(update).get(0) != null)
          update.close(); 
      } 
    } catch (SQLException e) {
      LOGGER.error("Failed to update fileId {}@{} in database {}", new Object[] { trackedFile, trackedFile
            
            .getId().toString(), this.dbFile });
      try {
        this.connection.rollback();
      } catch (SQLException e2) {
        LOGGER.error("Failed to rollback transaction: {}@{}", trackedFile, trackedFile.getId().toString());
        LOGGER.info("Reinitializing connection to database {}", this.dbFile);
        close();
      } 
      return false;
    } 
  }
  
  public List<TrackedFile> getAllCheckpointForFlow(FileFlow<?> flow) {
    Preconditions.checkNotNull(flow);
    if (!ensureConnected())
      throw new RuntimeException("Failed to connect to checkpoint database."); 
    List<TrackedFile> trackedFiles = new ArrayList<>();
    try {
      PreparedStatement statement = this.connection.prepareStatement("select path, fileId, lastModifiedTime, size, offset, headerLength, headerString from (select * from FILE_CHECKPOINTS where flow=? order by lastModifiedTime asc ) group by fileId order by lastModifiedTime desc");
      try {
        statement.setString(1, flow.getId());
        ResultSet result = statement.executeQuery();
      } finally {
        if (Collections.<PreparedStatement>singletonList(statement).get(0) != null)
          statement.close(); 
      } 
    } catch (SQLException e) {
      LOGGER.error("Failed when getting checkpoint for flow {}", flow.getId(), e);
      throw new RuntimeException("Failed to configure the checkpoint database.", e);
    } 
  }
  
  public void deleteCheckpointByTrackedFileList(List<TrackedFile> trackedFileList) {
    if (trackedFileList == null || trackedFileList.size() == 0)
      return; 
    if (!ensureConnected())
      return; 
    try {
      this.lock.lock();
      PreparedStatement statement = this.connection.prepareStatement("delete from FILE_CHECKPOINTS where rowid in (select rowid from FILE_CHECKPOINTS where flow=? and fileId=? order by lastModifiedTime asc limit 1)\n");
      try {
        for (TrackedFile trackedFile : trackedFileList) {
          statement.setString(1, trackedFile.getFlow().getId());
          statement.setString(2, trackedFile.getId().toString());
          statement.addBatch();
          LOGGER.debug("delete checkpoint info {}.", trackedFile);
        } 
        statement.executeBatch();
        this.connection.commit();
        LOGGER.info("Delete {} checkpoints", Integer.valueOf(trackedFileList.size()));
      } finally {
        if (Collections.<PreparedStatement>singletonList(statement).get(0) != null)
          statement.close(); 
      } 
    } catch (SQLException e) {
      try {
        this.connection.rollback();
      } catch (SQLException e2) {
        LOGGER.error("Failed to rollback cleanup transaction.", e2);
        LOGGER.info("Reinitializing connection to database {}", this.dbFile);
        close();
      } 
    } finally {
      this.lock.unlock();
    } 
  }
}

```



## 四、参考文档

+ 菜鸟教程（推荐）:https://www.runoob.com/sqlite/sqlite-tutorial.html
+ 中文文档：https://huili.github.io/index.html
+ SQLite快速入门教程：

  - https://www.jianshu.com/p/424a8b143bbb
  - https://www.cnblogs.com/xcsn/p/6050878.html
+ 原理

  - https://www.cnblogs.com/springyangwc/archive/2011/10/11/2207097.html
  - https://www.jianshu.com/p/b47986e7e734
  - https://blog.csdn.net/Tybyqi/article/details/85068915
  - http://blog.itpub.net/26736162/viewspace-2141867/
+ Java API操作SQLite：https://www.cnblogs.com/popfisher/p/5497206.html
+ 主键自增：https://blog.csdn.net/qq_35844043/article/details/97634257

## 五、rust操作SQLite

+ https://github.com/diesel-rs/diesel
+ https://github.com/launchbadge/sqlx
+ https://github.com/rusqlite/rusqlite
+ https://github.com/joaoh82/rust_sqlite
