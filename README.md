
resty 一款极简的restful轻量级的web框架
===========

拥有jfinal/activejdbc一样的activerecord的简洁设计，使用更简单的restful框架

restful的api设计，是作为restful的服务端最佳选择（使用场景：客户端和服务端解藕，用于对静态的html客户端（mvvm等），ios，andriod等提供服务端的api接口）

提醒：因框架还在开发第一个正式项目，所以会有细微调整，更新提醒频繁，您可以在个人设置->Notification center->Watching->Email 关闭邮件提醒，感谢您的理解和支持

下载jar包： [Resty jar](https://github.com/Dreampie/resty/releases)
一、独有优点：
-----------

1.极简的route设计:

```java
  @GET("/users/:name")  //在路径中自定义解析的参数 如果有其他符合 也可以用 /users/{name}
  // 参数名就是方法变量名  除路径参数之外的参数也可以放在方法参数里  传递方式 user={json字符串}
  public Map find(String name,User user) {
    // return Lister.of(name);
    return Maper.of("k1", "v1,name:" + name, "k2", "v2");//返回什么数据直接return，完全融入普通方法的方式
  }
```

2.极简的activerecord设计，数据操作只需短短的一行,支持批量保存对象

```java
  //批量保存
  User u1 = new User().set("username", "test").set("providername", "test").set("password", "123456");
  User u2 = new User().set("username", "test").set("providername", "test").set("password", "123456");
  User.dao.save(u1,u2);

  //普通保存
  User u = new User().set("username", "test").set("providername", "test").set("password", "123456");
  u.save();

  //更新
  u.update();
  //条件更新
  User.dao.updateBy(columns,where,paras);
  User.dao.updateAll(columns,paras);

  //删除
  u.deleted();
  //条件删除
  User.dao.deleteBy(where,paras);
  User.dao.deleteAll();

  //查询
  User.dao.findById(id);
  User.dao.findBy(where,paras);
  User.dao.findAll();

  //分页
  User.dao.paginateBy(pageNumber,pageSize,where,paras);
  User.dao.paginateAll(pageNumber,pageSize);
```

3.极简的客户端设计，支持各种请求，文件上传和文件下载（支持断点续传）

```java
  Client client=null;//创建客户端对象
  //启动resty-example项目，即可测试客户端
  String apiUrl = "http://localhost:8081/api/v1.0";
  //如果不需要 使用账号登陆
  //client = new Client(apiUrl);
  //如果有账号权限限制  需要登陆
  client = new Client(apiUrl, "/tests/login", "u", "123");

  //该请求必须  登陆之后才能访问  未登录时返回 401  未认证
  ClientRequest authRequest = new ClientRequest("/users", HttpMethod.GET);
  ResponseData authResult = client.build(authRequest).ask();
  System.out.println(authResult.getData());

  //get
  ClientRequest getRequest = new ClientRequest("/tests", HttpMethod.GET);
  ResponseData getResult = client.build(getRequest).ask();
  System.out.println(getResult.getData());

  //post
  ClientRequest postRequest = new ClientRequest("/tests", HttpMethod.POST);
  postRequest.addParameter("test", Jsoner.toJSONString(Maper.of("a", "谔谔")));
  ResponseData postResult = client.build(postRequest).ask();
  System.out.println(postResult.getData());

  //put
  ClientRequest putRequest = new ClientRequest("/tests/x", HttpMethod.PUT);
  ResponseData putResult = client.build(putRequest).ask();
  System.out.println(putResult.getData());


  //delete
  ClientRequest deleteRequest = new ClientRequest("/tests/a", HttpMethod.DELETE);
  ResponseData deleteResult = client.build(deleteRequest).ask();
  System.out.println(deleteResult.getData());


  //upload
  ClientRequest uploadRequest = new ClientRequest("/tests/resty", HttpMethod.POST);
  uploadRequest.addUploadFiles("resty", ClientTest.class.getResource("/resty.jar").getFile());
  uploadRequest.addParameter("des", "test file  paras  测试笔");
  ResponseData uploadResult = client.build(uploadRequest).ask();
  System.out.println(uploadResult.getData());


  //download  支持断点续传
  ClientRequest downloadRequest = new ClientRequest("/tests/file", HttpMethod.GET);
  downloadRequest.setDownloadFile(ClientTest.class.getResource("/resty.jar").getFile().replace(".jar", "x.jar"));
  ResponseData downloadResult = client.build(downloadRequest).ask();
  System.out.println(downloadResult.getData());
```


4.支持多数据源和嵌套事务（使用场景：需要访问多个数据库的应用，或者作为公司内部的数据中间件向客户端提供数据访问api等）

```java
  // 在resource里使用事务,也就是controller里，rest的世界认为所以的请求都表示资源，所以这儿叫resource
  @GET("/users")
  @Transaction(name = {DS.DEFAULT_DS_NAME, "demo"}) //多数据源的事务，如果你只有一个数据库  直接@Transaction 不需要参数
  public User transaction() {
  //TODO 用model执行数据库的操作  只要有操作抛出异常  两个数据源 都会回滚  虽然不是分布式事务  也能保证代码块的数据执行安全
  }

  // 如果你需要在service里实现事务，通过java动态代理（必须使用接口，jdk设计就是这样）
  public interface UserService {
    @Transaction(name = {DS.DEFAULT_DS_NAME, "demo"})//service里添加多数据源的事务，如果你只有一个数据库  直接@Transaction 不需要参数
    public User save(User u);
  }
  // 在resource里使用service层的 事务
  // @Transaction(name = {DS.DEFAULT_DS_NAME, "demo"})的注解需要写在service的接口上
  // 注意java的自动代理必须存在接口
  // TransactionAspect 是事务切面 ，你也可以实现自己的切面比如日志的Aspect，实现Aspect接口
  // 再private UserService userService = AspectFactory.newInstance(new UserServiceImpl(), new TransactionAspect(),new LogAspect());
  private UserService userService = AspectFactory.newInstance(new UserServiceImpl(), new TransactionAspect());
```

5.极简的权限设计，你只需要实现一个简单接口和添加一个拦截器，即可实现基于url的权限设计

```java
  public void configInterceptor(InterceptorLoader interceptorLoader) {
    //权限拦截器 放在第一位 第一时间判断 避免执行不必要的代码
    interceptorLoader.add(new SecurityInterceptor(new MyAuthenticateService()));
  }

  //实现接口
  public class MyAuthenticateService implements AuthenticateService {
    //登陆时 通过name获取用户的密码和权限信息
    public Principal findByName(String name) {
      DefaultPasswordService defaultPasswordService = new DefaultPasswordService();

      Principal principal = new Principal(name, defaultPasswordService.hash("123"), new HashSet<String>() {{
        add("api");
      }});
      return principal;
    }
    //基础的权限总表  所以的url权限都放在这儿  你可以通过 文件或者数据库或者直接代码 来设置所有权限
    public Set<Credential> loadAllCredentials() {
      Set<Credential> credentials = new HashSet<Credential>();
      credentials.add(new Credential("GET", "/api/v1.0/users**", "users"));
      return credentials;
    }
  }
```

6.极简的缓存设计，可扩展，非常简单即可启用model的自动缓存功能

```java
  public void configConstant(ConstantLoader constantLoader) {
    //启用缓存并在要自动使用缓存的model上  开启缓存@Table(name = "sec_user", cached = true)
    constantLoader.setCacheEnable(true);
  }

  @Table(name = "sec_user", cached = true)
  public class User extends Model<User> {
    public static User dao = new User();

  }
```

7.下载文件，只需要直接return file

```java
  @GET("/files")
  public File file() {
    return new File(path);
  }
```

8.上传文件，通过getFiles，getFile把文件写到服务器

```java
  @POST("/files")
  public UploadedFile file() {
    //Hashtable<String, UploadedFile> uploadedFiles=getFiles();
    return getFile(name);
  }
```

9.当然也是支持传统的web开发，你可以自己实现数据解析，在config里添加自定义的解析模板

```java
  public void configConstant(ConstantLoader constantLoader) {
    // 通过后缀来返回不同的数据类型  你可以自定义自己的  render  如：FreemarkerRender
    //默认已添加json和text的支持，只需要把自定义的Render add即可
    // constantLoader.addRender("json", new JsonRender());
  }
```

极简Restful框架 - Resty 开发群: <a target="_blank" href="http://shang.qq.com/wpa/qunwpa?idkey=8fc9498714ebbc3675cc5a5035858004154ef4645ebc9c128dfd76688d32179b"><img border="0" src="http://pub.idqqimg.com/wpa/images/group.png" alt="极简Restful框架 - Resty" title="极简Restful框架 - Resty"></a>

二、运行example示例：
-----------------

1.运行根目录下的pom.xml->install （命令行： mvn clean install -Dmaven.test.skip=true ，install时跳过测试，因为测试需要连接数据库，没有数据库会失败，把相关的插件安装到本地，功能完善之后发布到maven就不需要这样了）

2.在本地mysql数据库里创建demo,example数据库，对应application.properties的数据库配置

3.运行resty-example下的pom.xml->flyway-maven-plugin:migration，自动根具resources下db目录下的数据库文件生成数据库表结构

4.运行resty-example下的pom.xml->tomcat7-maven-plugin:run,启动example程序

提醒:推荐idea作为开发ide，使用分模块的多module开发

License [MIT](http://opensource.org/licenses/MIT)


