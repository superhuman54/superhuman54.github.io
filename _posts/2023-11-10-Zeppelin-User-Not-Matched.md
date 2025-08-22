---
layout: post
title: "Zeppelin HDFS 사용자 불일치 Permission Denied 문제"
date: 2023-11-10
categories: [Big Data]
tags: [zeppelin, hdfs, hadoop, spark, troubleshooting]
---

운영 환경에서 Zeppelin을 사용하여 데이터 분석 작업을 수행하던 중, Spark 인터프리터 설정을 변경하고 재시작하자마자 예상치 못한 에러가 발생했다.

<!-- more -->

에러 메시지는 다음과 같았다:

![Exception](https://github.com/user-attachments/assets/a277404a-6023-40d7-8512-323f592d9175)

이상한 점은 Spark 인터프리터가 분명히 `zeppelin` 사용자로 구동되고 있었는데, 갑자기 `admin` 사용자로 전환되어 HDFS에 쓰기 작업을 시도한다는 것이었다. HDFS의 `/user` 경로는 `hdfsadmingroup` 그룹에 대해서만 쓰기 권한이 허용되어 있었고, `admin` 사용자는 이 그룹에 속하지 않았기 때문에 당연히 에러가 발생할 수밖에 없었다.

![/etc/hadoop/conf/hdfs-site.xml](https://github.com/user-attachments/assets/4c49c582-46c4-4743-b7a5-bc5fee871c35)
![/etc/group](https://github.com/user-attachments/assets/5751230c-0109-4b4f-9016-ece9838342a3)



## 원인 분석

문제의 근본적인 원인을 파악하기 위해 에러 스택트레이스를 자세히 분석해보았다. 에러는 클라이언트가 YARN에게 작업을 제출하고, 로컬 리소스를 새로운 디렉토리에 업로드하는 과정에서 새 디렉토리 생성에 실패하면서 발생했다.
```scala 
  // org.apache.spark.yarn.Client

  private def createContainerLaunchContext(): ContainerLaunchContext = {
    logInfo("Setting up container launch context for our AM")
    val pySparkArchives =
      if (sparkConf.get(IS_PYTHON_APP)) {
        findPySparkArchives()
      } else {
        Nil
      }

    val launchEnv = setupLaunchEnv(stagingDirPath, pySparkArchives)
    val localResources = prepareLocalResources(stagingDirPath, pySparkArchives)  
  }
  //...
  
  /**
   * Upload any resources to the distributed cache if needed. If a resource is intended to be
   * consumed locally, set up the appropriate config for downstream code to handle it properly.
   * This is used for setting up a container launch context for our ApplicationMaster.
   * Exposed for testing.
   */
  def prepareLocalResources(
      destDir: Path,
      pySparkArchives: Seq[String]): HashMap[String, LocalResource] = {
    logInfo("Preparing resources for our AM container")
    // Upload Spark and the application JAR to the remote file system if necessary,
    // and add them as local resources to the application master.
    val fs = destDir.getFileSystem(hadoopConf)

    // Used to keep track of URIs added to the distributed cache. If the same URI is added
    // multiple times, YARN will fail to launch containers for the app with an internal
    // error.
    val distributedUris = new HashSet[String]
    // Used to keep track of URIs(files) added to the distribute cache have the same name. If
    // same name but different path files are added multiple time, YARN will fail to launch
    // containers for the app with an internal error.
    val distributedNames = new HashSet[String]

    val replication = sparkConf.get(STAGING_FILE_REPLICATION).map(_.toShort)
    val localResources = HashMap[String, LocalResource]()
    // org.apache.hadoop.security.AccessControlException 발생!!
    FileSystem.mkdirs(fs, destDir, new FsPermission(STAGING_DIR_PERMISSION)) 

    // If preload is enabled, preload the statCache with the files in the directories
    val statCache = if (statCachePreloadEnabled) {
      // Consider only following configurations, as they involve the distribution of multiple files
      var files = sparkConf.get(SPARK_JARS).getOrElse(Nil) ++ sparkConf.get(JARS_TO_DISTRIBUTE) ++
        sparkConf.get(FILES_TO_DISTRIBUTE) ++ sparkConf.get(ARCHIVES_TO_DISTRIBUTE) ++
        sparkConf.get(PY_FILES) ++ pySparkArchives

      getPreloadedStatCache(files)
    } else {
      HashMap[URI, FileStatus]()
    }
    val symlinkCache: Map[URI, Path] = HashMap[URI, Path]()
```

`org.apache.spark.yarn.Client` 클래스에서 `createContainerLaunchContext()`와 `prepareLocalResources()` 메소드를 살펴보니, `prepareLocalResources()` 메소드에서 `stagingDirPath`를 `destDir` 인자값으로 전달받아 디렉토리를 생성하기 위해 `FileSystem.mkdirs()`를 호출하는데, 바로 여기서 에러가 발생하고 있었다.

문제는 이 `destDir` 변수가 `/user/admin`을 담고 있다는 점이었다. 정상적으로 동작한다면 `/user/zeppelin` 값이 설정되어야 하는데, 어디선가 `/user/admin`으로 변경되고 있었다.

```scala
  // org.apache.spark.yarn.Client

   private[spark] val STAGING_DIR = ConfigBuilder("spark.yarn.stagingDir")
    .doc("Staging directory used while submitting applications.")
    .version("2.0.0")
    .stringConf
    .createOptional
  
  // ...

  def submitApplication(): Unit = {
    ResourceRequestHelper.validateResources(sparkConf)

    try {
      launcherBackend.connect()
      yarnClient.init(hadoopConf)
      yarnClient.start()

      if (log.isDebugEnabled) {
        logDebug("Requesting a new application from cluster with %d NodeManagers"
          .format(yarnClient.getYarnClusterMetrics.getNumNodeManagers))
      }

      // Get a new application from our RM
      val newApp = yarnClient.createApplication()
      val newAppResponse = newApp.getNewApplicationResponse()
      this.appId = newAppResponse.getApplicationId()

      // The app staging dir based on the STAGING_DIR configuration if configured
      // otherwise based on the users home directory.
      // `appStagingBaseDir`은 홈디렉토리로 지정되고 어떤 사용자의 홈인지 여기서 결정된다.
      val appStagingBaseDir = sparkConf.get(STAGING_DIR)
        .map { new Path(_, UserGroupInformation.getCurrentUser.getShortUserName) } 
        .getOrElse(FileSystem.get(hadoopConf).getHomeDirectory())
      stagingDirPath = new Path(appStagingBaseDir, getAppStagingDir(appId)) 
      
      // ...
  }
```
더 깊이 추적해보니 `submitApplication()` 메소드에서 `appStagingBaseDir`이 홈디렉토리로 지정되는데, 이때 사용자의 홈디렉토리가 결정되는 것을 발견했다. `spark.yarn.stagingDir` 설정이 없다면 기본적으로 `UserGroupInformation.getCurrentUser.getShortUserName`과 `FileSystem.get(hadoopConf).getHomeDirectory()`로 경로가 생성된다.

핵심은 `getCurrentUser()`에서 현재 사용자를 결정하는 로직이었다. 이 메소드는 `getLoginUser()`를 호출하여 로그인 사용자를 반환하는데, Hadoop의 보안 체계는 [JAAS](https://docs.oracle.com/javase/7/docs/technotes/guides/security/jaas/JAASRefGuide.html)(Java Authentication and Authorization Service)를 사용한다.

```java
  // org.apache.hadoop.security.UserGroupInformation

 * Login a subject with the given parameters.  If the subject is null,
   * the login context used to create the subject will be attached.
   * @param subject to login, null for new subject.
   * @param params for login, null for externally managed ugi.
   * @return UserGroupInformation for subject
   * @throws IOException
   */
  private static UserGroupInformation doSubjectLogin(
      Subject subject, LoginParams params) throws IOException {
    ensureInitialized();
    // initial default login.
    if (subject == null && params == null) {
      params = LoginParams.getDefaults();
    }
    HadoopConfiguration loginConf = new HadoopConfiguration(params);
    try {
      HadoopLoginContext login = newLoginContext(
        authenticationMethod.getLoginAppName(), subject, loginConf); 
      login.login(); // JAAS LoginContext 구현체로 로그인 시도를 하면 LoginModule의 구현체에서 인증이 진행된다.
      UserGroupInformation ugi = new UserGroupInformation(login.getSubject());
      // attach login context for relogin unless this was a pre-existing
      // subject.
      if (subject == null) {
        params.put(LoginParam.PRINCIPAL, ugi.getUserName());
        ugi.setLogin(login);
        ugi.setLastLogin(Time.now());
      }
      return ugi;
    } catch (LoginException le) {
      throw e;
    }
  }
```

JAAS의 `LoginContext`에서 로그인을 시도하면 `LoginModule`에서 두 단계의 인증 과정을 거친다. 첫 번째는 `LoginModule.login()`에서 검증 단계이고, 두 번째는 `LoginModule.commit()`에서 인증 성공 시 주체를 저장하고 갱신하는 단계이다.

```java
// org.apache.hadoop.security.UserGroupInformation

public static class HadoopLoginModule implements LoginModule {
    private Subject subject;

    @Override
    public boolean abort() throws LoginException {
      return true;
    }

    private <T extends Principal> T getCanonicalUser(Class<T> cls) {
      for(T user: subject.getPrincipals(cls)) {
        return user;
      }
      return null;
    }
    
    @Override
    public boolean login() throws LoginException {
      LOG.debug("Hadoop login");
      return true;
    }

    @Override
    public boolean commit() throws LoginException {
      LOG.debug("hadoop login commit");
      // if we already have a user, we are done.
      if (!subject.getPrincipals(User.class).isEmpty()) {
        LOG.debug("Using existing subject: {}", subject.getPrincipals());
        return true;
      }
      Principal user = getCanonicalUser(KerberosPrincipal.class);
      if (user != null) {
        LOG.debug("Using kerberos user: {}", user);
      }
      //If we don't have a kerberos user and security is disabled, check
      //if user is specified in the environment or properties
      if (!isSecurityEnabled() && (user == null)) {
        String envUser = System.getenv(HADOOP_USER_NAME); // 사용자의 이름을 HADOOP_USER_NAME의 환경변수에서 가져온다.
        if (envUser == null) {
          envUser = System.getProperty(HADOOP_USER_NAME);
        }
        user = envUser == null ? null : new User(envUser);
      }
      // use the OS user
      if (user == null) {
        user = getCanonicalUser(OS_PRINCIPAL_CLASS);
        LOG.debug("Using local user: {}", user);
      }
      // if we found the user, add our principal
      if (user != null) {
        LOG.debug("Using user: \"{}\" with name: {}", user, user.getName());

        User userEntry = null;
        try {
          // LoginContext will be attached later unless it's an external
          // subject.
          AuthenticationMethod authMethod = (user instanceof KerberosPrincipal)
            ? AuthenticationMethod.KERBEROS : AuthenticationMethod.SIMPLE;
          userEntry = new User(user.getName(), authMethod, null);
        } catch (Exception e) {
          throw (LoginException)(new LoginException(e.toString()).initCause(e));
        }
        LOG.debug("User entry: \"{}\"", userEntry);

        subject.getPrincipals().add(userEntry);
        return true;
      }
      throw new LoginException("Failed to find user in name " + subject);
    }
  }
```

여기서 중요한 발견을 했다. Hadoop의 `LoginModule` 구현체에서는 인증이 항상 성공하도록 설계되어 있었고, `commit()` 단계에서 주체를 갱신할 때 **`HADOOP_USER_NAME` 환경변수**에서 현재 사용자의 이름을 조회한다는 것이다.

![Spark Interpreter Enviroment Variables](https://github.com/user-attachments/assets/197d143a-6082-42e4-9c59-bd03a734ef3a)

실제로 SparkInterpreter가 실행될 때의 환경변수를 확인해보니 `HADOOP_USER_NAME=admin`으로 설정되어 있었다. 이것이 바로 문제의 원인이었다.

## Zeppelin의 사용자 처리 로직

```java
 // org.apache.zeppelin.interpreter.launcher.SparkInterpreterLauncher

 public Map<String, String> buildEnvFromProperties(InterpreterLaunchContext context) throws IOException {
    Map<String, String> env = super.buildEnvFromProperties(context);
    Properties sparkProperties = new Properties();
    String spMaster = getSparkMaster(context);
    if (spMaster != null) {
      sparkProperties.put(SPARK_MASTER_KEY, spMaster);
    }
    Properties properties = context.getProperties();
    
    // ...
    
    if (isYarnMode(context)) {
      boolean runAsLoginUser = Boolean.parseBoolean(context
              .getProperties()
              .getProperty("zeppelin.spark.run.asLoginUser", "true")); // `zeppelin.spark.run.asLoginUser`의 값에 따라 환경변수 존재 유무가 결정된다.
      String userName = context.getUserName();
      if (runAsLoginUser && !"anonymous".equals(userName)) { 
        env.put("HADOOP_USER_NAME", userName);
      }
    }
  }
```
Zeppelin의 SparkInterpreter 코드를 살펴보니 `buildEnvFromProperties()` 메소드에서 환경변수를 설정하는 로직을 발견할 수 있었다. 이 메소드에서는 `zeppelin.spark.run.asLoginUser` 설정값에 따라 `HADOOP_USER_NAME` 환경변수의 존재 여부가 결정되었다.

즉, YARN 모드에서 실행할 때 `zeppelin.spark.run.asLoginUser`가 `true`로 설정되어 있고 사용자명이 `anonymous`가 아니라면, Zeppelin UI에서 로그인한 사용자의 이름을 `HADOOP_USER_NAME` 환경변수로 설정하는 것이었다. 우리의 경우 Zeppelin UI에서 `admin` 사용자로 로그인했기 때문에, Spark 작업이 실행될 때 `admin` 사용자의 홈디렉토리인 `/user/admin`에 접근하려고 시도했던 것이다.

## 해결 방안

이 문제를 해결하는 방법은 크게 세 가지가 있었다.

### 방법 1: zeppelin.spark.run.asLoginUser 비활성화 (선택한 방법)

![Zeppelin Configurations](https://github.com/user-attachments/assets/7d2d4e93-5bf9-4082-95c9-b16e5386657b)

첫 번째는 Zeppelin에서 로그인 사용자를 Spark 사용자로 사용하는 기능을 비활성화하는 것이었다. Zeppelin UI의 Spark 인터프리터 설정에서 `zeppelin.spark.run.asLoginUser` 옵션을 `false`로 변경했다. 이렇게 하면 `HADOOP_USER_NAME` 환경변수가 설정되지 않아서 기본적으로 Zeppelin 프로세스를 실행하는 시스템 사용자(즉, `zeppelin` 사용자)로 Spark 작업이 실행된다.

### 방법 2: 사용자를 HDFS 그룹에 추가

두 번째 방법은 현재 로그인 사용자를 HDFS 그룹에 포함시키는 것이었다. `/etc/group` 파일에서 `hdfsadmingroup`에 `admin` 사용자를 추가하면 된다. 하지만 이 방법은 보안상 권장되지 않는다.

### 방법 3: 다른 사용자로 로그인

세 번째 방법은 이미 `hdfsadmingroup`에 속한 다른 사용자를 Zeppelin 로그인 계정으로 사용하는 것이었다. 하지만 이 경우 YARN UI나 Spark UI에서 작업을 식별하기 어려워진다는 단점이 있다.

우리 팀의 개발 환경 특성상 첫 번째 방법이 가장 적절했다. 설정 변경 후에는 정상적으로 동작하는 것을 확인할 수 있었다.

## 결론과 교훈

이번 문제를 통해 Zeppelin과 Hadoop 생태계의 복잡한 사용자 인증 체계를 깊이 이해할 수 있었다. 특히 JAAS 기반의 Hadoop 인증 시스템에서 환경변수가 어떻게 사용자 식별에 영향을 미치는지 알게 되었다.

운영 환경에서는 보안 그룹과 사용자 권한을 명확히 분리하여 관리하는 것이 중요하다. 또한 Zeppelin의 `zeppelin.spark.run.asLoginUser` 설정을 사용할 때는 해당 사용자가 실제로 필요한 HDFS 권한을 가지고 있는지 반드시 확인해야 한다.

무엇보다 이런 복잡한 문제가 발생했을 때는 에러 스택트레이스를 차근차근 따라가며 근본 원인을 찾는 것이 중요하다는 것을 다시 한번 깨달았다. 표면적으로는 단순한 권한 문제처럼 보였지만, 실제로는 Zeppelin의 사용자 처리 로직과 Hadoop의 인증 체계가 복합적으로 얽힌 문제였다.
