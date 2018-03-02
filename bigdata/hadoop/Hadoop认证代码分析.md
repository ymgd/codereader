# Hadoop认证代码分析

Hadoop作为分布式系统，服务分布于多台服务器之间，提供多用户的访问机制，却有着极其简单的认证实现逻辑。

JAAS（Java Authentication Authorization Service）完整的提供了一个认证鉴权的框架，在Hadoop这个看似庞大的架构当中，借助这一体系，在一个单独的Java类中，实现了认证的绝大部分逻辑。

本文主要讲述Hadoop中的认证实现方法（鉴权暂时不谈）。

## 一、JAAS
### 1、什么是认证鉴权

认证（Authentication），用于鉴别“张三即是张三”，识别用户的合法性。

鉴权（Authorization），用于鉴别“张三能不能干这事情”，识别用户操作的权限。

### 2、JAAS架构及使用方法

> 以下关于JAAS，参考自：[http://www.cnblogs.com/allenny/archive/2006/02/27/338544.html](http://www.cnblogs.com/allenny/archive/2006/02/27/338544.html)。

#### JAAS的主要实现架构，如下图：

![][1]

图中从上到下各层次：

Application：JAAS使用者使用Java语言编写的应用。

LoginContext：可以看做JAAS为上层Java应用提供的住入口，上层应用通过调用LoginContext提供的方法，与底层的认证支撑体系（账号库等等）对接。

xxxLoginModule：JAAS将底层不同的实际账号认证服务进行抽象，比如RDBMS抽象成RdbmsLoginModule等等。有了LogingModule的抽象能力，让JAAS的使用者可以根据自己的需求选择实际的账号存储体系及账号认证的服务架构，只需要以相对应的LoginModule与上层（LoginContext）相对接上就可以了。

各类实际认证服务：包括LDAP、RDBMS等等，用户也可以自定义Service。

#### JAAS的主要使用方法

Java应用使用JAAS的方法为：

![][2]

整个调用就两步：

1. 创建LoginContext实例lc。

2. 调用lc的login方法实现认证。（认证失败时会抛出LoginException)

需要留意的是，在创建LoginContext实例的时候，传入的"Example"参数，用于定位在配置文件jaas.config中所配好的Login Entry。寻找到名为Example的Login Entry所对应的配置项，获知其所使用的LoginModule为RdbmsLoginModule以及相关的配置参数（其实，就是使用mysql数据库中的信息作为认证依据）。一个LoginContext在认证过程中可以使用多个Login Entry，实现多LoginModule登录的需求。

#### 认证的实际流程

Java应用在调用LoginContext的login方法之后，JAAS实际以预定义的顺序调用LoginModule的若干方法最终完成整个认证的流程。在LoginModule的这些方法中，调用实际的认证服务，就将认证（登录）的流程与实际的认证服务关联起来了。

LoginModule的主要方法：

- initialize () ：创建LoginModule实例时会被构造函数调用。
- login ()：进行验证，调用LoginContext的login方法之后，逐个调用所关联的每个LoginModule的login方法。
- commit ()：当LgoninContext对象接受所有LoginModule对象传回的结果后将调用该方法。该方法将Principal对象和凭证赋给Subject对象。
- abort () 当任何一个LoginModule对象验证失败时都会调用该方法。此时没有任何Principal对象或凭证关联到Subject对象上。
- logout () 删除与Subject对象关联的Principal对象和凭证。

对于一个LoginModule来说，在用户调用LoginContext的login方法之后，实际经历的主要方法调用流程就是：

- 成功时调用序列：initialize->login->commit
- 失败时调用序列：initialize->login->abort

#### 认证过程中的信息存储

在认证的过程当中，当用户进行认证登录之后，其信息需要存储于某个对象当中，以备之后的操作基于该对象提供的方法进行，对象自身携带的信息可以充分说明对象是经过某用户成功认证之后生成的。

Java中，该对象为Subject类的实例，Subject中所包含信息：

- Principal（可以简单类比为好理解的username，不严谨）
- Credentials(public/private) （可简单类比为password，不严谨）

Subject的中关于对象属性的这些信息，在LoginModule的commit方法中会被赋上正确的值。

## 二、Hadop认证主要实现类
### 1、类定义
Hadoop认证的实现类为org.apache.hadoop.security.UserGroupInformation（[代码链接](https://github.com/apache/hadoop/blob/trunk/hadoop-common-project/hadoop-common/src/main/java/org/apache/hadoop/security/UserGroupInformation.java)）。

从上面对JAAS的简述很容易想象，Hadoop为了定义自己的认证机制，实际就是实现了一个自己的LoginModule，在Java应用调用LoginContext的login方法之后，触发Hadoop自定义的LoginModule中的逻辑。

#### HadoopConfiguration
先看看UserGroupInformation的内部类HadoopConfiguration中有这样一段定义：

```java
private static final AppConfigurationEntry[] SIMPLE_CONF = new AppConfigurationEntry[]{OS_SPECIFIC_LOGIN, HADOOP_LOGIN};
private static final AppConfigurationEntry OS_SPECIFIC_LOGIN =
      new AppConfigurationEntry(OS_LOGIN_MODULE_NAME,
                                LoginModuleControlFlag.REQUIRED,
                                BASIC_JAAS_OPTIONS);
private static final AppConfigurationEntry HADOOP_LOGIN =
      new AppConfigurationEntry(HadoopLoginModule.class.getName(),
                                LoginModuleControlFlag.REQUIRED,
                                BASIC_JAAS_OPTIONS);
```

这里定义了AppConfigurationEntry，实际就相当于前面示例中说的Example所对应的配置文件中的LoginEntry。在Hadoop中，不再使用文件方式定义LoginEntry，而是使用的代码实现的方式。

在SIMPLE_CONF这个AppConfigurationEntry的数组中，包含了OS_SPECIFIC_LOGIN跟HADOOP_LOGIN两个LoginEntry，对于比如HADOOP_LOGIN这样的LoginEntry，其中定义了它所关联的LoginModule是HadoopLoginModule。（实际逻辑执行时，代码通过读取hadoop的配置项hadoop.security.authentication获取当前使用的认证方法，该配置项默认为simple，于是会映射到SIMPLE_CONF这一入口，这段逻辑我们可以自己再仔细推敲）

#### HadoopLoginModule
上面提到的，由LoginEntry包含的HadoopLoginModule，也是UserGroupInformation中的内部类：

```java
public static class HadoopLoginModule implements LoginModule 
```

它就是JAAS中LoginModule的扩展类。

上面提到过，在一次登录过程中，会触发LoginModule的login、commit等方法，我们逐个看看HadoopLoginModule中都做了什么。

1、 initialize

```java
@Override
public void initialize(Subject subject, CallbackHandler callbackHandler,
                       Map<String, ?> sharedState, Map<String, ?> options) {
  this.subject = subject;
}
```

HadoopLoginModule在这里并没有做什么，将初始化时传进来的一个Subject对象的引用赋给了成员属性subject，后面在登录过程中，对subject的修改，会影响到调用initialize者所传进来的找个subject对象（最终找个对象中的值会被使用者获取到）。

2、login及logout

```java
@Override
public boolean login() throws LoginException {
  if (LOG.isDebugEnabled()) {
    LOG.debug("hadoop login");
  }
  return true;
}

@Override
public boolean logout() throws LoginException {
  if (LOG.isDebugEnabled()) {
    LOG.debug("hadoop logout");
  }
  return true;
}

@Override
public boolean abort() throws LoginException {
  return true;
}
```

HadoopLoginModule在login被调用时什么都没做，把事情留给后面的方法。
logout及abort也是如此。

一般的交互式应用，会在LoginModule的login方法中实现给用户弹窗请求输入用户名密码的逻辑，Hadoop作为一个后台批处理系统，使用认证服务的不仅仅是普通的人类用户，其中的一些服务实例比如datanode等，也会使用login，因此没有使用交互式等登录方法。

3、commit

```java
@Override
public boolean commit() throws LoginException {
  if (LOG.isDebugEnabled()) {
    LOG.debug("hadoop login commit");
  }
  // subject对象中已经有了用户信息（登录过）
  if (!subject.getPrincipals(User.class).isEmpty()) {
    if (LOG.isDebugEnabled()) {
      LOG.debug("using existing subject:"+subject.getPrincipals());
    }
    return true;
  }
  Principal user = null;
  // 如果使用Kerberos认证方式
  if (isAuthenticationMethodEnabled(AuthenticationMethod.KERBEROS)) {
    user = getCanonicalUser(KerberosPrincipal.class);
    if (LOG.isDebugEnabled()) {
      LOG.debug("using kerberos user:"+user);
    }
  }
  //如果没有用Kerberos，读取环境变量HADOOP_USER_NAME作为用户名
  if (!isSecurityEnabled() && (user == null)) {
    String envUser = System.getenv(HADOOP_USER_NAME);
    if (envUser == null) {
      envUser = System.getProperty(HADOOP_USER_NAME);
    }
    user = envUser == null ? null : new User(envUser);
  }
  //HADOOP_USER_NAME没有设置，使用当前执行出的操作系统用户名作为hadoop操作的用户名
  if (user == null) {
    user = getCanonicalUser(OS_PRINCIPAL_CLASS);
    if (LOG.isDebugEnabled()) {
      LOG.debug("using local user:"+user);
    }
  }
  //找到用户名，将其添加进subject对象当中
  if (user != null) {
    if (LOG.isDebugEnabled()) {
      LOG.debug("Using user: \"" + user + "\" with name " + user.getName());
    }

    User userEntry = null;
    try {
      userEntry = new User(user.getName());
    } catch (Exception e) {
      throw (LoginException)(new LoginException(e.toString()).initCause(e));
    }
    if (LOG.isDebugEnabled()) {
      LOG.debug("User entry: \"" + userEntry.toString() + "\"" );
    }

    subject.getPrincipals().add(userEntry);
    return true;
  }

  //找不到定义用户名，抛异常
  LOG.error("Can't find user in " + subject);
  throw new LoginException("Can't find user name");
}
```

commit方法是HadoopLoginModule的认证流程所在的主要地方，整个代码流程还是比较好理解的，逐一检测每一种可能的认证方式，一旦某种方式获取到用户名之后，将user变量设为用户名（后面的方式会由于if (user == null) 判断为false而被跳过）。最后在方法的最后，将用户信息设置到subject当中。

我们可以做个简单的尝试来检验我们阅读代码之后的结论，比如这个过程中，hadoop会判断HADOOP_USER_NAME是否定义，来确定当前的用户名，我们在实际的hadoop环境中做如下操作：

![][3]

可以看到，当把HADOOP_USER_NAME设置为用户名chinahadoop之后，再进行后续的操作，用户名都会是chinahadoop。

## 三、Hadoop认证机制触发流程

以上描述了Hadoop认证机制的基本原理。下面具体屡一下Hadoop中一个用户登录（认证）操作流程是如何的。

从一个使用场景开始：

```java
public static FileSystem get(final URI uri, final Configuration conf,
      final String user) throws IOException, InterruptedException {
  String ticketCachePath =
    conf.get(CommonConfigurationKeys.KERBEROS_TICKET_CACHE_PATH);
  UserGroupInformation ugi =
      UserGroupInformation.getBestUGI(ticketCachePath, user);
  return ugi.doAs(new PrivilegedExceptionAction<FileSystem>() {
    @Override
    public FileSystem run() throws IOException {
      return get(uri, conf);
    }
  });
}
```

这是org.apache.hadoop.fs.FileSystem中的get方法，用户通过调用该方法获取FileSystem实例，然后操作hdfs文件系统。

因为hadoop分布式文件系统（hdfs）中对于目录的访问需要根据用户进行权限区分（类似于Linux文件系统），所以在这个get方法的实现中，需要具备区分用户的能力。

方法中的UserGroupInformation.getBestUGI调用便是这样一个入口，根据当前的配置情况，该方法最终返回的UserGroupInformation类对象，包含了当前用户的信息。

继续看UserGroupInformation.getBestUGI实现：

```java
public static UserGroupInformation getBestUGI(
    String ticketCachePath, String user) throws IOException {
  if (ticketCachePath != null) {
    return getUGIFromTicketCache(ticketCachePath, user);
  } else if (user == null) {
    return getCurrentUser();
  } else {
    return createRemoteUser(user);
  }    
}
```

代码中基于性能（cache）、功能完整性（支持多种认证方式）等考虑，会有多种分支处理，这里我们研究我们最主要的关注点。

getBestUGI方法中，当用户未认证时，if (user == null) 条件被满足 ，调用getCurrentUser。继续看看getCurrentUser的实现：

```java
static UserGroupInformation getCurrentUser() throws IOException {
  AccessControlContext context = AccessController.getContext();
  Subject subject = Subject.getSubject(context);
  if (subject == null || subject.getPrincipals(User.class).isEmpty()) {
    return getLoginUser();
  } else {
    return new UserGroupInformation(subject);
  }
}
```

用户未认证之前，subject为null（如前面内容描述，subject在LoginModule的commit中被设置好），调用getLoginUser。getLoginUser代码：

```java
static UserGroupInformation getLoginUser() throws IOException {
  if (loginUser == null) {
    loginUserFromSubject(null);
  }
  return loginUser;
}
```

loginUser为空，继续调用loginUserFromSubject：

```java
@InterfaceAudience.Public
@InterfaceStability.Evolving
public synchronized 
static void loginUserFromSubject(Subject subject) throws IOException {
  ensureInitialized();
  try {
    if (subject == null) {
      subject = new Subject();
    }
    // 创建LoginContext，传入的HadoopConfiguration对象会读入hadoop配置，如果配置
    // 项hadoop.security.authentication的值为simple，会走入我们前面描述的
    // SimpleEntry逻辑。（其他配置项时也会获取相应LogingEntry，我们这里以simple为
    // 研究示例。
    LoginContext login =
        newLoginContext(authenticationMethod.getLoginAppName(), 
                        subject, new HadoopConfiguration());

    // LoginContext的login方法调用，会触发关联到的LoginModule的login、commit等
    // 一系列方法调用
    login.login();
    UserGroupInformation realUser = new UserGroupInformation(subject);
    realUser.setLogin(login);
    realUser.setAuthenticationMethod(authenticationMethod);
    realUser = new UserGroupInformation(login.getSubject());
    // If the HADOOP_PROXY_USER environment variable or property
    // is specified, create a proxy user as the logged in user.
    String proxyUser = System.getenv(HADOOP_PROXY_USER);
    if (proxyUser == null) {
      proxyUser = System.getProperty(HADOOP_PROXY_USER);
    }
    loginUser = proxyUser == null ? realUser : createProxyUser(proxyUser, realUser);

    String fileLocation = System.getenv(HADOOP_TOKEN_FILE_LOCATION);
    if (fileLocation != null) {
      // Load the token storage file and put all of the tokens into the
      // user. Don't use the FileSystem API for reading since it has a lock
      // cycle (HADOOP-9212).
      Credentials cred = Credentials.readTokenStorageFile(
          new File(fileLocation), conf);
      loginUser.addCredentials(cred);
    }
    loginUser.spawnAutoRenewalThreadForUserCreds();
  } catch (LoginException le) {
    LOG.debug("failure to login", le);
    throw new IOException("failure to login", le);
  }
  if (LOG.isDebugEnabled()) {
    LOG.debug("UGI loginUser:"+loginUser);
  } 
}
```

在以上代码加了中文注释的地方，将会触发HadoopLoginModule的login以及commit等方法，执行我们在文章第二部分描述的操作逻辑。


[1]: resources/jaasarch.gif
[2]: resources/jaasusage.gif
[3]: resources/username.png

