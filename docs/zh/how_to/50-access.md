---
weight: 50
sourceSHA: 6de5193b1f91f44e7880f1e9e76832add1847ded06798757e5f9a9c9446aa667
---

# 如何访问MySQL-MGR实例

业务应用访问 MySQL-MGR 实例时有多种连接方式，本节描述了如何选择连接地址，通过连接地址访问到 MySQL-MGR 实例，并举例演示。

## 适用场景

| 访问方式 | 适用场景 |
| --- | --- |
| **集群内访问** | 单主或多主模式下，集群内应用通过 SVC 暴露的端口访问 MGR 实例。 |
| **集群外访问** | 单主或多主模式下，集群外应用通过 NodePort 的端口访问 MGR 实例。|

## 常见客户端程序

举例展示了 JDBC 和 GORM 客户端的程序。其他客户端请自行参考相应框架说明。

使用代码时，请将相关变量修改为您的真实值。变量说明如下：

| 变量 | 说明 |
| --- | --- |
| **host** | 对应访问方式下的主机 IP 或 SVC 名称 |
| **port** | 对应访问方式下的端口号 |
| **user** | MySQL-MGR 用户名 |
| **password** | MySQL-MGR 用户的真实密码 |
| **database** | 需要访问的库名称 |

### JDBC

1. 集群内访问

    ```
    import java.sql.*;
    public class Main {
        static final String DB_URL = "jdbc:mysql://mgr-0:3306,mgr-1:3306,mgr-2:3306/mysql";
        static final String USER = "root";
        static final String PASS = "123456";
        public static void main(String[] args) {
            try {
                Connection conn = DriverManager.getConnection(DB_URL,USER,PASS);
                Statement statement = conn.createStatement();
            }catch(SQLException se){
                se.printStackTrace();
            }
        }
    }
    ```

2. 集群外访问 - 通过 NodePort

    ```
    import java.sql.*;
    public class Main {
        static final String DB_URL = "jdbc:mysql://192.168.1.100:32630,192.168.1.101:32630,192.168.1.102:32630/mysql";
        static final String USER = "root";
        static final String PASS = "123456";
        public static void main(String[] args) {
            try {
                Connection conn = DriverManager.getConnection(DB_URL,USER,PASS);
                Statement statement = conn.createStatement();
            }catch(SQLException e){
                e.printStackTrace();
            }
        }
    }
    ```

### GORM

1. 集群内访问

    ```
    import (
        "gorm.io/driver/mysql"
        "gorm.io/gorm"
        "gorm.io/plugin/dbresolver"
    )

    func main() {
        writeDSN:="root:123456@tcp(mgr-read-write:32631)/mysql"
        readDSN := "root:123456@tcp(mgr-read-only:32630)/mysql"
        db, err := gorm.Open(mysql.Open(writeDSN), &gorm.Config{})
        if err != nil {
            panic(err)
        }
        err = db.Use(dbresolver.Register(dbresolver.Config{
            Replicas:          []gorm.Dialector{mysql.Open(readDSN)},
            Policy:            dbresolver.RandomPolicy{},
            TraceResolverMode: true,
        }))
        if err != nil {
            panic(err)
        }
    }
    ```
    
2. 集群外访问 - 通过 NodePort

    ```
    package main

    import (
        "gorm.io/driver/mysql"
        "gorm.io/gorm"
        "gorm.io/plugin/dbresolver"
    )

    func main() {
        writeDSNs := []string{
            "root:123456@tcp(192.168.18.130:32631)/mysql",
            "root:123456@tcp(192.168.18.131:32631)/mysql",
            "root:123456@tcp(192.168.18.132:32631)/mysql",
        }
        readDSNs := []string{
            "root:123456@tcp(192.168.18.131:32630)/mysql",
            "root:123456@tcp(192.168.18.132:32630)/mysql",
            "root:123456@tcp(192.168.18.133:32630)/mysql",
        }
        db, err := gorm.Open(mysql.Open(writeDSNs[0]), &gorm.Config{})
        if err != nil {
            panic(err)
        }
        sources := make([]gorm.Dialector, 0)
        replicas := make([]gorm.Dialector, 0)
        for _, dsn := range writeDSNs {
            sources = append(sources, mysql.Open(dsn))
        }
        for _, dsn := range readDSNs {
            replicas = append(replicas, mysql.Open(dsn))
        }
        err = db.Use(dbresolver.Register(dbresolver.Config{
            Sources:           sources,
            Replicas:          replicas,
            Policy:            dbresolver.RandomPolicy{},
            TraceResolverMode: true,
        }))
        if err != nil {
            panic(err)
        }
    }
    ```
