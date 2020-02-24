**什么是Mybatis？**

（1）Mybatis是一个半ORM（对象关系映射）框架，它==内部封装了JDBC，开发时只需要关注SQL语句本身，不需要花费精力去处理加载驱动、创建连接、创建statement等繁杂的过程==。程序员直接编写原生态sql，可以严格控制sql执行性能，灵活度高。

（2）MyBatis ==可以使用 XML 或注解来配置和映射原生信息==，将 POJO映射成数据库中的记录，避免了几乎所有的 JDBC 代码和手动设置参数以及获取结果集。

（3）通过xml 文件或注解的方式将要执行的各种 statement 配置起来，并通过java对象和 statement中sql的动态参数进行映射生成最终执行的sql语句，最后由mybatis框架执行sql并将结果映射为java对象并返回。（从执行sql到返回result的过程）。