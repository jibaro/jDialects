(English version see "README-ENGLISH.MD")  
## jDialects
开源协议: [LGPL 2.1](http://www.gnu.org/licenses/lgpl-2.1.html)  

jDialects是一个收集了大多数数据库方言的Java小项目，通常可用来创建分页SQL和建表DDL语句，可根据不同的数据库方言生成不同的SQL。目前jDialects支持75种数据库方言，包括Hibernate中没有的SQLite和Access等。jDialects需要Java7或以上版本支持。  

jDialects起初是为了jSqlBox项目而开发的，但它本身是一个独立的项目(发布包只有90k大小且无其它第三方依赖)，只要用到了SQL，就可以利用它来创建对应数据库的分页SQL和DDL。如果你使用了纯JDBC、JdbcTemplate、DbUtils...等，只要出现原生SQL，并有跨数据库需求的场合(例如单元测试需要同时在H2内存数据库和实际数据库Oracle上运行)，就可以利用这个工具来实现跨数据库的分页和建表。  

jDialects项目的主体部分是由代码生成工具从Hibernate5.2.9版本中抽取而自动生成，这从一定程度上也保证了它的代码质量。代码生成工具见[jDiagen](https://github.com/drinkjava2/jDiagen)项目。    
  
### 如何引入项目?  
下载 "jdialects-1.0.1.jar"并放入项目库目录，或在项目的pom.xml文件中加入：
```
	<dependency>  
		<groupId>com.github.drinkjava2</groupId>  
		<artifactId>jdialects</artifactId>  
		<version>1.0.1</version>  
	</dependency>
```	
(目前正在开发1.0.2-SNAPSHOP版，主要对DLL这块功能进行完善)
### 在程序中使用   
1) 创建跨数据库的分页：  
```
       Dialect d=guessDialect(dataSource);  //根据数据源判断方言类型,  
       //Dialect d=guessDialect(connection);  //或根据连接来判断方言类型  
       //Dialect d=Dialect.MySQL5Dialect;     //或手工指定数据库方言类型 
       String result=d.paginate(3, 10, "select * from users where id=?");  //创建分页SQL  
     
   当方言为MySQL5Dialect, 结果为: "select * from users where id=? limit 20, 10"  
   当方言为Oracle8iDialect, 结果为: "select * from ( select row_.*, rownum rownum_ from ( select * from users where id=? ) row_ ) where rownum_ <= 30 and rownum_ > 20"  
   当方言为Oracle12cDialect, 结果为: "select * from users where id=? offset 20 rows fetch next 10 rows only"  
   当方言为Sybase11Dialect, 抛出DialectExcepiton异常并提示: "Sybase11Dialect" does not support physical pagination  
   ...
```   
     
2) 创建跨数据库的DDL(目前功能比较简陋，正在改进中):  
```
	private static String ddlSQL(Dialect d) {
		return "create table " + d.check("BufferPool") + "("//
				+ d.BIGINT("f1") //
				+ ", " + d.BIT("f2", 5) //
				+ ", " + d.BLOB("f3") //
				+ ", " + d.BOOLEAN("f4") //
				+ ", " + d.INTEGER("f5") //
				+ ", " + d.VARCHAR("f6", 8000) //
				+ ", " + d.NUMERIC("ACCESS_LOCK", 8,2) // 
				+ ")" + d.engine(" DEFAULT CHARSET=utf8");
	}

	public static void main(String[] args) {//运行在不同方言上
		System.out.println(ddlSQL(Dialect.MySQL57InnoDBDialect));
		System.out.println(ddlSQL(Dialect.SQLServer2012Dialect));
		System.out.println(ddlSQL(Dialect.Oracle10gDialect));	
	} 
```	
   本示例运行结果：
```   
   create table BufferPool(f1 bigint, f2 bit, f3 longblob, f4 bit, f5 integer, f6 varchar(8000), ACCESS_LOCK decimal(8,2))engine=innoDB DEFAULT CHARSET=utf8
   create table BufferPool(f1 bigint, f2 bit, f3 varbinary(MAX), f4 bit, f5 int, f6 varchar(MAX), ACCESS_LOCK numeric(8,2))
   create table BufferPool(f1 number(19,0), f2 number(1,0), f3 blob, f4 number(1,0), f5 number(10,0), f6 long, ACCESS_LOCK number(8,2))
```
   示例在运行时会有日志警告输出: "BufferPool"和"ACCESS_LOCK"分别是DB2和Teradata的保留字，这意味着如果运行在DB2Dialect或TeradataDialect数据库上将会有DialectException例外抛出。为保证可移植性，最好将有警告的字段改为其它名称。  
   如果想要跳过保留字检查(不推荐)，也可按以下格式来书写DDL：  
```   
	   ddl= "create table BufferPool("//
				+ "f1 "+d.BIGINT() //
				+ ",f2 " + d.BIT(5) //
				+ ",f3 " + d.BLOB() //
				+ ",f4 " + d.BOOLEAN() //
				+ ",f5 " + d.INTEGER() //
				+ ",f6 " + d.VARCHAR(8000) //
				+ ",ACCESS_LOCK " + d.NUMERIC(8,2) // 
				+ ")" + d.engine();
```
   jDialects支持的所有类型定义如下：  
	BOOLEAN  
	DOUBLE  
	FLOAT  
	INTEGER  
	LONG(=BIGINT)  
	SHORT(=SMALLINT)  
	BIGDECIMAL(=NUMERIC)  
	STRING(=VARCHAR)  
	DATE  
	TIME  
	TIMESTAMP  
	BIGINT  
	BINARY  
	BIT  
	BLOB  
	CHAR  
	CLOB  
	DECIMAL  
	LONGNVARCHAR  
	LONGVARBINARY  
	LONGVARCHAR  
	NCHAR  
	NCLOB  
	NUMERIC  
	NVARCHAR  
	REAL  
	SMALLINT  
	TINYINT  
	VARBINARY  
	VARCHAR  
	
  
3) 关于SQL函数  
jDialects暂不支持跨数据库的SQL函数，主要是因为SQL函数在不同的数据库里往往有两种情况，一种是命名和参数完全相同，一种是变化非常大，很多是专有函数，无论前者还是后者，都不太适合用通用的SQL函数来代表。针对这一点，jDialects代码生成工具将75种方言的函数对比写在“DatabaseDialects.xls”这个文件中(还包含了各种方言的分页和类型定义)，如果需要作数据库移植时可以作为速查手册。  

以上即为jDialects全部文档，如有不清楚处可以查看项目源码及单元测试。最后强调一下，jDialects只是个文本变换工具，根据不同的方言对SQL进行不同的变换，它本身不是一个完整的持久化工具，必须配合其它持久化工具如JDBC/DbUtils等使用。

### 附录
以下为目前jDialects支持的75种数据库方言： 
AccessDialect  
Cache71Dialect  
CobolDialect  
CUBRIDDialect  
DataDirectOracle9Dialect  
DB2390Dialect  
DB2400Dialect  
DB2Dialect  
DbfDialect  
DerbyDialect  
DerbyTenFiveDialect  
DerbyTenSevenDialect  
DerbyTenSixDialect  
ExcelDialect  
FirebirdDialect  
FrontBaseDialect  
H2Dialect  
HANAColumnStoreDialect  
HANARowStoreDialect  
HSQLDialect  
Informix10Dialect  
InformixDialect  
Ingres10Dialect  
Ingres9Dialect  
IngresDialect  
InterbaseDialect  
JDataStoreDialect  
MariaDB53Dialect  
MariaDBDialect  
MckoiDialect  
MimerSQLDialect  
MySQL55Dialect  
MySQL57Dialect  
MySQL57InnoDBDialect  
MySQL5Dialect  
MySQL5InnoDBDialect  
MySQLDialect  
MySQLInnoDBDialect  
MySQLMyISAMDialect  
Oracle10gDialect  
Oracle12cDialect  
Oracle8iDialect  
Oracle9Dialect  
Oracle9iDialect  
OracleDialect  
ParadoxDialect  
PointbaseDialect  
PostgresPlusDialect  
PostgreSQL81Dialect  
PostgreSQL82Dialect  
PostgreSQL91Dialect  
PostgreSQL92Dialect  
PostgreSQL93Dialect  
PostgreSQL94Dialect  
PostgreSQL95Dialect  
PostgreSQL9Dialect  
PostgreSQLDialect  
ProgressDialect  
RDMSOS2200Dialect  
SAPDBDialect  
SQLiteDialect  
SQLServer2005Dialect  
SQLServer2008Dialect  
SQLServer2012Dialect  
SQLServerDialect  
Sybase11Dialect  
SybaseAnywhereDialect  
SybaseASE157Dialect  
SybaseASE15Dialect  
SybaseDialect  
Teradata14Dialect  
TeradataDialect  
TextDialect  
TimesTenDialect  
XMLDialect  
