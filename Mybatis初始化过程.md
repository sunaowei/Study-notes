# Mybatis 初始化过程

>  Mybatis的初始化就是将Mybatis的配置信息加载到`Configuration`这个类中，以便后面的使用。

Mybatis的初始化流程，下面是一段Mybatis从加载配置到执行SQL语句的完整过程的代码(该代码来自Mybatis的测试类)

![Mybatis测试代码1](http://orw70g1os.bkt.clouddn.com/1521025129.png)

下面我们来一步一步的分析

1. 读取配置文件

   我们可以看到首先使用的是`Resources`这个类读取的配置文件，以流的形式读取到程序中，返回一个`Reader`。

2. 加载配置文件

   这一步将上面读取的`reader`作为参数传递给`SqlSessionFactoryBuilder`用于构建`SqlSessionFactory`。![SqlSessionFactoryBuilder](http://orw70g1os.bkt.clouddn.com/1521026253.png)该类提供了许多重载的方法供我们选择。

   ![](http://orw70g1os.bkt.clouddn.com/1521026596.png)

   此类的核心就是将我们传入的`Reader`或者`InputStream`通过`XMLConfigBuilder`类的`parse`方法，构造成`Configuration`，然后通过第91行的build方法，构造一个`SqlSessionFactory`并返回。

3. 配置文件解析过程

   ```java
   XMLConfigBuilder parser = new XMLConfigBuilder(inputStream, environment, properties);
   return build(parser.parse());
   ```

   配置文件解析的入口就是在这里。根据读取配置文件时传入的`Reader`或者`InputStream`分别调用`XMLConfigBuilder`不同的构造器，实例化出`XMLConfigBuilder`,然后调用`XMLConfigBuilder`类中的`parse()`方法,构造出我们需要的`Configuration`。

   ```java
     public XMLConfigBuilder(InputStream inputStream, String environment, Properties props) {
       this(new XPathParser(inputStream, true, props, new XMLMapperEntityResolver()), environment, props);
     }
   ```

   这是构建`XMLConfigBuilder`的具体方法。这个地方的构造过程如下

   * 先实例化`XMLMapperEntityResolver`,然后将其作为实例化`XPathParser`的参数之一，最后实例化`XMLConfigBuilder`

   * 实例化`XMLMapperEntityResolver`的过程很简单，调用的是其无参构造函数，`XMLMapperEntityResolver`里面包含了一些Mybatis的DTD。该类对外暴露了一个`public InputSource resolveEntity(String publicId, String systemId)`的方法，用于寻找根据`publicId`和`systemId`,寻找对应的DTD文件。

   * 实例化`XPathParser`

     ```java
       public XPathParser(InputStream inputStream, boolean validation, Properties variables, EntityResolver entityResolver) {
         commonConstructor(validation, variables, entityResolver);
         this.document = createDocument(new InputSource(inputStream));
       }
     ```

     `inputStream`就是我们传入的配置文件的流的信息，entityResolver是我们刚刚实例化的`XMLMapperEntityResolver`。首先调用了`commonConstructor`

     ```java
       private void commonConstructor(boolean validation, Properties variables, EntityResolver entityResolver) {
         this.validation = validation;
         this.entityResolver = entityResolver;
         this.variables = variables;
          // 实例化一个XPath工厂，用于生产XPath
         XPathFactory factory = XPathFactory.newInstance();
         this.xpath = factory.newXPath();
       }
     ```

     执行完`commonConstructor`之后就是创建`Document`了,这一块没啥重点，解析XML成Document对象而已。

     ```java
       private Document createDocument(InputSource inputSource) {
         // important: this must only be called AFTER common constructor
         try {
           // 创建一个用于创建DocumentBuilder的工厂，并给工厂设置一些参数
           DocumentBuilderFactory factory = DocumentBuilderFactory.newInstance();
           factory.setValidating(validation);

           factory.setNamespaceAware(false);
           factory.setIgnoringComments(true);
           factory.setIgnoringElementContentWhitespace(false);
           factory.setCoalescing(false);
           factory.setExpandEntityReferences(true);

           // 通过工厂创建一个用于创建Document的builder
           DocumentBuilder builder = factory.newDocumentBuilder();
           // 设置一些参数
           builder.setEntityResolver(entityResolver);
           builder.setErrorHandler(new ErrorHandler() {
             @Override
             public void error(SAXParseException exception) throws SAXException {
               throw exception;
             }

             @Override
             public void fatalError(SAXParseException exception) throws SAXException {
               throw exception;
             }

             @Override
             public void warning(SAXParseException exception) throws SAXException {
             }
           });
           // 解析输入的流创建Document
           return builder.parse(inputSource);
         } catch (Exception e) {
           throw new BuilderException("Error creating document instance.  Cause: " + e, e);
         }
       }

     ```

   * 都创建完了，我们再回到真正执行`XMLConfigBuilder`创建的方法。

     ```java
       private XMLConfigBuilder(XPathParser parser, String environment, Properties props) {
         // 通过父类调用Configuration的无参构造方法
         super(new Configuration());
         ErrorContext.instance().resource("SQL Mapper Configuration");
         this.configuration.setVariables(props);
         // 确保只执行一次，调用parse()方法时，会将其设置为true
         this.parsed = false;
         this.environment = environment;
         // 这个是重点，后面会用到
         this.parser = parser;
       }
     ```

     至于`new Configuration()`具体干了啥，我们可以看一下源码。

     ```java
       public Configuration() {
         typeAliasRegistry.registerAlias("JDBC", JdbcTransactionFactory.class);
         typeAliasRegistry.registerAlias("MANAGED", ManagedTransactionFactory.class);

         typeAliasRegistry.registerAlias("JNDI", JndiDataSourceFactory.class);
         typeAliasRegistry.registerAlias("POOLED", PooledDataSourceFactory.class);
         typeAliasRegistry.registerAlias("UNPOOLED", UnpooledDataSourceFactory.class);

         typeAliasRegistry.registerAlias("PERPETUAL", PerpetualCache.class);
         typeAliasRegistry.registerAlias("FIFO", FifoCache.class);
         typeAliasRegistry.registerAlias("LRU", LruCache.class);
         typeAliasRegistry.registerAlias("SOFT", SoftCache.class);
         typeAliasRegistry.registerAlias("WEAK", WeakCache.class);

         typeAliasRegistry.registerAlias("DB_VENDOR", VendorDatabaseIdProvider.class);

         typeAliasRegistry.registerAlias("XML", XMLLanguageDriver.class);
         typeAliasRegistry.registerAlias("RAW", RawLanguageDriver.class);

         typeAliasRegistry.registerAlias("SLF4J", Slf4jImpl.class);
         typeAliasRegistry.registerAlias("COMMONS_LOGGING", JakartaCommonsLoggingImpl.class);
         typeAliasRegistry.registerAlias("LOG4J", Log4jImpl.class);
         typeAliasRegistry.registerAlias("LOG4J2", Log4j2Impl.class);
         typeAliasRegistry.registerAlias("JDK_LOGGING", Jdk14LoggingImpl.class);
         typeAliasRegistry.registerAlias("STDOUT_LOGGING", StdOutImpl.class);
         typeAliasRegistry.registerAlias("NO_LOGGING", NoLoggingImpl.class);

         typeAliasRegistry.registerAlias("CGLIB", CglibProxyFactory.class);
         typeAliasRegistry.registerAlias("JAVASSIST", JavassistProxyFactory.class);

         languageRegistry.setDefaultDriverClass(XMLLanguageDriver.class);
         languageRegistry.register(RawLanguageDriver.class);
       }
     ```

     我们可以看到，这个无参的构造方法主要是注册了一系列的别名，这个我们另行分析，我们先往下看。

   * `XMLConfigBuilder`实例化完成以后就是调用其parse()方法完成一系列配置文件的解析，和对Configuration的装配。

     ```java
       public Configuration parse() {
         // 保证只被解析一次。
         if (parsed) {
           throw new BuilderException("Each XMLConfigBuilder can only be used once.");
         }
         parsed = true;
         // 解析根节点(至于这个evalNode是如何执行的，可以参照XPath解析xml相关的文章)
         XNode xNode = parser.evalNode("/configuration");
         // 解析根节点下面的子节点
         parseConfiguration(xNode);
         return configuration;
       }
     ```

     ```java
      private void parseConfiguration(XNode root) {
         try {
           //issue #117 read properties first
           propertiesElement(root.evalNode("properties"));
           Properties settings = settingsAsProperties(root.evalNode("settings"));
           loadCustomVfs(settings);
           typeAliasesElement(root.evalNode("typeAliases"));
           pluginElement(root.evalNode("plugins"));
           objectFactoryElement(root.evalNode("objectFactory"));
           objectWrapperFactoryElement(root.evalNode("objectWrapperFactory"));
           reflectorFactoryElement(root.evalNode("reflectorFactory"));
           settingsElement(settings);
           // read it after objectFactory and objectWrapperFactory issue #631
           environmentsElement(root.evalNode("environments"));
           databaseIdProviderElement(root.evalNode("databaseIdProvider"));
           typeHandlerElement(root.evalNode("typeHandlers"));
           mapperElement(root.evalNode("mappers"));
         } catch (Exception e) {
           throw new BuilderException("Error parsing SQL Mapper Configuration. Cause: " + e, e);
         }
       }
     ```

     这是解析Configuration下面的11个子节点，并装配到Configuration中，具体解析过程可以点进去详细查看。

     执行完整个parse()的过程，返回`Configuration`整个初始化流程也就结束了。