---
title: mybatis+druid多数据源实践
categories: 技术
tags: [java,springboot,mybatis,druid]
---

## start

&emsp;&emsp;最近项目中需要使用`mybatis` + `druid`集成多数据源,网上有很多`mybatis`多数据源的解决方案,但是总感觉不太灵活,于是自己实践了配置方案,虽然期间遇到很多问题,但总算基本实现功能,遗憾的是由于对`spring ioc`源码没有深入的了解,有些地方代码实现的很不优雅.

- - -

<!--more-->

- - -

## content

### 思路

- 加载`druid`配置和`datasource`配置,动态注册`datasource bean`.
- 通过`spring aop`拦截`mybatis mapper`,根据注解设置对应数据源.

### 配置

#### 基本属性配置

**druid:`druid`具体配置参考官网**
```yml 
spring:
  datasource:
    type: com.alibaba.druid.pool.DruidDataSource
    druid:
      initialSize:  ${DATASOURCE_INITIAL_SIZE:20}
      minIdle: ${DATASOURCE_MIN_IDLE:20}
      maxActive: ${DATASOURCE_MAX_ACTIVE:100}
      maxWait: ${DATASOURCE_MAX_WAIT:2000}
      timeBetweenEvictionRunsMillis: 60000
      removeAbandoned: true
      removeAbandonedTimeout: 180
      logAbandoned: true
      minEvictableIdleTimeMillis: 300000
      validationQuery: SELECT 1 FROM DUAL
      testWhileIdle: true
      testOnBorrow: false
      testOnReturn: false
      poolPreparedStatements: true
      maxPoolPreparedStatementPerConnectionSize: 30
      filters: stat
      connectionProperties: druid.stat.mergeSql=true;druid.stat.slowSqlMillis=5000
```

**multiple datasource:(1.testDb1 2.testDb2)**
```yml
spring:
  datasource:
    dataSources:
      # testDb1
      testDb1:
        ip: localhost
        port: 3306
        database: testDb1
        url: jdbc:mysql://${spring.datasource.dataSources.testDb1.ip}:${spring.datasource.dataSources.testDb1.port}/${spring.datasource.dataSources.testDb1.database}?zeroDateTimeBehavior=convertToNull&useUnicode=true&characterEncoding=utf-8&autoReconnect=true&allowMultiQueries=true&useSSL=false
        username: root
        password: root
        driver-class-name: com.mysql.jdbc.Driver
        name: testDb1
      # testDb2
      testDb2:
        ip: localhost
        port: 3306
        database: testDb2
        url: jdbc:mysql://${spring.datasource.dataSources.testDb2.ip}:${spring.datasource.dataSources.testDb2.port}/${spring.datasource.dataSources.testDb2.database}?zeroDateTimeBehavior=convertToNull&useUnicode=true&characterEncoding=utf-8&autoReconnect=true&allowMultiQueries=true&useSSL=false
        username: root
        password: root
        driver-class-name: com.mysql.jdbc.Driver
        name: testDb2
```

#### druid相关代码

**DefaultDruidProperties:用于解析`druid`相关属性配置**
```java
@Getter
@Setter
@Builder
@AllArgsConstructor
@NoArgsConstructor
@EqualsAndHashCode(callSuper = false)
@ToString
@ConfigurationProperties(prefix = DefaultDruidProperties.DRUID_PREFIX)
public class DefaultDruidProperties {

    public static final String DRUID_PREFIX = "spring.datasource.druid";

    private String name;
    private String driverClassName;
    private String url;
    private String username;
    private String password;
    private String initialSize;
    private String minIdle;
    private String maxActive;
    private String maxWait;
    private String timeBetweenEvictionRunsMillis;
    private String minEvictableIdleTimeMillis;
    private String validationQuery;
    private String testWhileIdle;
    private String testOnBorrow;
    private String testOnReturn;
    private String poolPreparedStatements;
    private String maxPoolPreparedStatementPerConnectionSize;
    private String filters;
    private String connectionProperties;
    private String removeAbandoned;
    private String removeAbandonedTimeout;
    private String logAbandoned;

	// 静态工厂方法
    public static DefaultDruidProperties buildProperties(ConfigurableEnvironment environment) {
        DefaultDruidProperties properties = new DefaultDruidProperties();
        if (environment != null) {
            MutablePropertySources propertySources = environment.getPropertySources();
            new RelaxedDataBinder(properties, DefaultDruidProperties.DRUID_PREFIX)
                .bind(new PropertySourcesPropertyValues(propertySources));
        }
        return properties;
    }
}
```

**MultiDataSourceProperties:用于解析多数据源相关配置(并支持单数据源)**
```java
@Getter
@Setter
@Builder
@AllArgsConstructor
@NoArgsConstructor
@EqualsAndHashCode(callSuper = false)
@ToString
@ConfigurationProperties(prefix = MultiDataSourceProperties.DATASOURCE_PREFIX)
public class MultiDataSourceProperties {

    public static final String DATASOURCE_PREFIX = "spring.datasource";

    private Map<String, BaseDataSource> dataSources;
    private String driverClassName;
    private String url;
    private String username;
    private String password;
    private String name;

    public void addDataSource(String name, BaseDataSource baseDataSource) {
        if (dataSources == null) {
            dataSources = new HashMap<>();
        }
        dataSources.put(name, baseDataSource);
    }

	// 静态工厂方法
    public static MultiDataSourceProperties buildProperties(ConfigurableEnvironment environment) {
        MultiDataSourceProperties properties = new MultiDataSourceProperties();
        if (environment != null) {
            MutablePropertySources propertySources = environment.getPropertySources();
            new RelaxedDataBinder(properties, MultiDataSourceProperties.DATASOURCE_PREFIX)
                .bind(new PropertySourcesPropertyValues(propertySources));
        }
        return properties;
    }

	@Getter
	@Setter
	@Builder
	@AllArgsConstructor
	@NoArgsConstructor
	@EqualsAndHashCode(callSuper = false)
	@ToString
    public static class BaseDataSource {

        private String driverClassName;
        private String url;
        private String username;
        private String password;
        private String name;
    }
}
```

**BaseDruidDataSourceFactory:`druid`工厂方法**
```java
public class BaseDruidDataSourceFactory {

    private DefaultDruidProperties properties;

    private Map<String, Object> configProperties = new HashMap<>();

    public BaseDruidDataSourceFactory(DefaultDruidProperties properties) {
        this.properties = properties;
        init();
    }

    public DefaultDruidProperties getProperties() {
        return properties;
    }

    public void setProperties(DefaultDruidProperties properties) {
        this.properties = properties;
    }

    private void init() {
        configProperties.put(DruidDataSourceFactory.PROP_NAME, properties.getName());
        configProperties
            .put(DruidDataSourceFactory.PROP_DRIVERCLASSNAME, properties.getDriverClassName());
        configProperties.put(DruidDataSourceFactory.PROP_URL, properties.getUrl());
        configProperties.put(DruidDataSourceFactory.PROP_USERNAME, properties.getUsername());
        configProperties.put(DruidDataSourceFactory.PROP_PASSWORD, properties.getPassword());
        configProperties.put(DruidDataSourceFactory.PROP_INITIALSIZE, properties.getInitialSize());
        configProperties.put(DruidDataSourceFactory.PROP_MINIDLE, properties.getMinIdle());
        configProperties.put(DruidDataSourceFactory.PROP_MAXACTIVE, properties.getMaxActive());
        configProperties.put(DruidDataSourceFactory.PROP_MAXWAIT, properties.getMaxWait());
        configProperties.put(DruidDataSourceFactory.PROP_TIMEBETWEENEVICTIONRUNSMILLIS,
            properties.getTimeBetweenEvictionRunsMillis());
        configProperties.put(
            DruidDataSourceFactory.PROP_MINEVICTABLEIDLETIMEMILLIS,
            properties.getMinEvictableIdleTimeMillis());
        configProperties
            .put(DruidDataSourceFactory.PROP_VALIDATIONQUERY, properties.getValidationQuery());
        configProperties
            .put(DruidDataSourceFactory.PROP_TESTWHILEIDLE, properties.getTestWhileIdle());
        configProperties
            .put(DruidDataSourceFactory.PROP_TESTONBORROW, properties.getTestOnBorrow());
        configProperties
            .put(DruidDataSourceFactory.PROP_TESTONRETURN, properties.getTestOnReturn());
        configProperties.put(
            DruidDataSourceFactory.PROP_POOLPREPAREDSTATEMENTS,
            properties.getPoolPreparedStatements());
        configProperties.put(DruidDataSourceFactory.PROP_FILTERS, properties.getFilters());
        configProperties.put(DruidDataSourceFactory.PROP_CONNECTIONPROPERTIES,
            properties.getConnectionProperties());
        configProperties
            .put(DruidDataSourceFactory.PROP_REMOVEABANDONED, properties.getRemoveAbandoned());
        configProperties.put(DruidDataSourceFactory.PROP_REMOVEABANDONEDTIMEOUT,
            properties.getRemoveAbandonedTimeout());
        configProperties
            .put(DruidDataSourceFactory.PROP_LOGABANDONED, properties.getLogAbandoned());
    }

    public DataSource createDataSource() throws Exception {
        try {
            return DruidDataSourceFactory.createDataSource(configProperties);
        } catch (Exception e) {
            throw new Exception("create druid datasource error", e);
        }
    }

    public Map<String, Object> getConfigProperties() {
        return Collections.unmodifiableMap(configProperties);
    }

    public void refresh() {
        init();
    }
}
```

**MultiDruidDataSourceRegistry:多数据源注册器,根据配置属性动态注册`datasource`**
```java
public class MultiDruidDataSourceRegistry implements BeanDefinitionRegistryPostProcessor {

    private ScopeMetadataResolver scopeMetadataResolver = new AnnotationScopeMetadataResolver();

    private BeanNameGenerator beanNameGenerator = new AnnotationBeanNameGenerator();

    private BaseDruidDataSourceFactory druidDataSourceFactory;

    private MultiDataSourceProperties dataSourceProperties;

    public MultiDruidDataSourceRegistry(
        BaseDruidDataSourceFactory druidDataSourceFactory,
        MultiDataSourceProperties dataSourceProperties) {
        this.druidDataSourceFactory = druidDataSourceFactory;
        this.dataSourceProperties = dataSourceProperties;
    }

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory)
        throws BeansException {
        dataSourceProperties.getDataSources().forEach((name, baseDataSource) -> {
            BeanDefinition definition = beanFactory.getBeanDefinition(getBeanName(name));
            MutablePropertyValues propertyValues = definition.getPropertyValues();
            // inject druid property values
            druidDataSourceFactory.getConfigProperties().forEach(propertyValues::addPropertyValue);
            // inject base datasource property values
            removeBaseProperties(propertyValues);
            resetBaseProperties(propertyValues, baseDataSource, name);
        });
    }

    @Override
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry)
        throws BeansException {
        dataSourceProperties.getDataSources()
            .forEach((name, baseDataSource) -> registerBean(registry, getBeanName(name),
                DruidDataSource.class));
    }

    private void registerBean(BeanDefinitionRegistry registry, String name, Class<?> beanClass) {
        AnnotatedGenericBeanDefinition abd = new AnnotatedGenericBeanDefinition(beanClass);
        ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(abd);
        abd.setScope(scopeMetadata.getScopeName());
        String beanName = (name != null ? name
            : this.beanNameGenerator.generateBeanName(abd, registry));
        AnnotationConfigUtils.processCommonDefinitionAnnotations(abd);
        BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(abd, beanName);
        BeanDefinitionReaderUtils.registerBeanDefinition(definitionHolder, registry);
    }

    private void resetBaseProperties(MutablePropertyValues propertyValues,
        BaseDataSource baseDataSource, String name) {
        propertyValues.addPropertyValue(DruidDataSourceFactory.PROP_NAME,
            Optional.ofNullable(baseDataSource.getName()).orElse(name));
        propertyValues.addPropertyValue(DruidDataSourceFactory.PROP_DRIVERCLASSNAME,
            baseDataSource.getDriverClassName());
        propertyValues.addPropertyValue(DruidDataSourceFactory.PROP_URL, baseDataSource.getUrl());
        propertyValues
            .addPropertyValue(DruidDataSourceFactory.PROP_USERNAME, baseDataSource.getUsername());
        propertyValues
            .addPropertyValue(DruidDataSourceFactory.PROP_PASSWORD, baseDataSource.getPassword());
    }

    private void removeBaseProperties(MutablePropertyValues propertyValues) {
        propertyValues.removePropertyValue(DruidDataSourceFactory.PROP_NAME);
        propertyValues.removePropertyValue(DruidDataSourceFactory.PROP_DRIVERCLASSNAME);
        propertyValues.removePropertyValue(DruidDataSourceFactory.PROP_URL);
        propertyValues.removePropertyValue(DruidDataSourceFactory.PROP_USERNAME);
        propertyValues.removePropertyValue(DruidDataSourceFactory.PROP_PASSWORD);
    }

    private String getBeanName(String name) {
        return "DataSource" + name.substring(0, 1).toUpperCase() + name.substring(1);
    }
}
```

**DruidConfig:`druid`配置类,声明`druid`需要的相关`bean`,其中多数据源由`MultiDruidDataSourceRegistry`动态注册**
```java
@Configuration
public class DruidConfig implements EnvironmentAware {

    @Autowired
    private Environment environment;

    @Bean
    public MultiDruidDataSourceRegistry registry(DataSourceProperties dataSourceProperties) {
        MultiDataSourceProperties multiDataSourceProperties = MultiDataSourceProperties
            .buildProperties((ConfigurableEnvironment) environment);
        // single datasource
        if (StringUtils.hasText(multiDataSourceProperties.getUrl())) {
            String name = Optional.ofNullable(multiDataSourceProperties.getName())
                .orElse("defaultDb");
            BaseDataSource baseDataSource = new BaseDataSource();
            baseDataSource.setName(name);
            baseDataSource.setDriverClassName(multiDataSourceProperties.getDriverClassName());
            baseDataSource.setUrl(multiDataSourceProperties.getUrl());
            baseDataSource.setUsername(multiDataSourceProperties.getUsername());
            baseDataSource.setPassword(multiDataSourceProperties.getPassword());
            multiDataSourceProperties.addDataSource(name, baseDataSource);
        }
        return new MultiDruidDataSourceRegistry(druidDataSourceFactory(),
            multiDataSourceProperties);
    }

    @Bean
    public BaseDruidDataSourceFactory druidDataSourceFactory() {
        return new BaseDruidDataSourceFactory(DefaultDruidProperties.buildProperties(
            (ConfigurableEnvironment) environment));
    }

    @Bean
    public ServletRegistrationBean druidStatViewServlet() {
        ServletRegistrationBean servletRegistrationBean =
            new ServletRegistrationBean(new StatViewServlet(), "/admin/druid/*");
        servletRegistrationBean.addInitParameter("resetEnable", "true");
        return servletRegistrationBean;
    }

    @Bean
    public FilterRegistrationBean druidStatFilter() {
        FilterRegistrationBean filterRegistrationBean = new FilterRegistrationBean(
            new WebStatFilter());
        filterRegistrationBean.addUrlPatterns("/*");
        filterRegistrationBean.addInitParameter(
            "exclusions", "*.js,*.gif,*.jpg,*.png,*.css,*.ico,/admin/druid/*");
        return filterRegistrationBean;
    }

    @Override
    public void setEnvironment(Environment environment) {
        this.environment = environment;
    }
}
```

&emsp;&emsp;在配置`DruidConfig`时并没有通过`@EnableConfigurationProperties`引入`DefaultDruidProperties`和`MultiDataSourceProperties`,而是通过静态方法`buildProperties()`获取属性配置类,这是因为继承`BeanDefinitionRegistryPostProcessor`的`bean`优先级较高,此时`ConfigurationProperties`并没有被属性注入,所以这里用静态方法获取属性配置对象.

#### mybatis相关代码

**DatabaseContextHolder:通过`ThreadLocal`选择动态数据源**
```java
public class DatabaseContextHolder {

    private static final ThreadLocal<String> CONTEXT_HOLDER = new ThreadLocal<>();

    public static void setDbType(String dbType) {
        Assert.notNull(dbType, "database type requires not null!");
        CONTEXT_HOLDER.set(dbType);
    }

    public static String getDbType() {
        return CONTEXT_HOLDER.get();
    }

    public static void clearDbType() {
        CONTEXT_HOLDER.remove();
    }
}
```

**DynamicDataSource:动态数据源,通过从`ThreadLocal`获取key选择注册的数据源**
```java
public class DynamicDataSource extends AbstractRoutingDataSource {

    @Override
    protected Object determineCurrentLookupKey() {
        return DatabaseContextHolder.getDbType();
    }
}
```

**DataSourceType:在`Mapper`类上注解,用以区分数据源**
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
public @interface DataSourceType {

    /**
     * datasource type name
     *
     * @return name
     */
    String value();
}
```

**DataSourceAspect:`spring aop`拦截切面**
```java
@Aspect
@Order(-1)
@Component
public class DataSourceAspect {

    @Pointcut("execution(* com.jxg.test.dao..*.*(..))")
    public void jointPointExpression() {

    }

    @Around("jointPointExpression()")
    public Object setDataSourceKey(ProceedingJoinPoint point)
        throws Throwable {
        try {
            Optional.ofNullable(
                AnnotationUtils.findAnnotation(point.getTarget().getClass(), DataSourceType.class))
                .ifPresent(
                    dataSourceType -> DatabaseContextHolder.setDbType(dataSourceType.value()));
            return point.proceed();
        } finally {
            DatabaseContextHolder.clearDbType();
        }
    }
}
```

**MybatisConfig:`mybatis`配置类,由于需要用到`MultiDruidDataSourceRegistry`动态注册的数据源,需要在`DruidConfig`后加载**
```java
@Configuration
@AutoConfigureAfter({DruidConfig.class})
@MapperScan(basePackages = "com.jxg.test.dao", sqlSessionFactoryRef = "sqlSessionFactory")
@EnableTransactionManagement
public class MybatisConfig extends MybatisAutoConfiguration {

    @Autowired
    private List<DruidDataSource> dataSources;

    @Bean
    @Override
    public SqlSessionFactory sqlSessionFactory(DataSource dataSource) throws Exception {
        SqlSessionFactoryBean sqlSessionFactoryBean = new SqlSessionFactoryBean();
        sqlSessionFactoryBean.setDataSource(dataSource);
        ResourcePatternResolver resolver = new PathMatchingResourcePatternResolver();
        sqlSessionFactoryBean.setMapperLocations(
            resolver.getResources("classpath*:mybatis/mapper/**/*Mapper.xml"));
        sqlSessionFactoryBean.setTypeAliasesPackage("com.jxg.test.entity.mybatis");
        sqlSessionFactoryBean
            .setConfigLocation(new ClassPathResource("mybatis/mybatis-config.xml"));
        return sqlSessionFactoryBean.getObject();
    }

    @Bean
    @Primary
    public DynamicDataSource dynamicDataSource() {
        Map<Object, Object> targetDataSources = new HashMap<>();
        dataSources.forEach(source -> targetDataSources.put(source.getName(), source));
        DynamicDataSource dataSource = new DynamicDataSource();
        dataSource.setTargetDataSources(targetDataSources);
        // default datasource
        dataSource.setDefaultTargetDataSource(dataSources.get(0));
        return dataSource;
    }

    @Bean
    public DataSourceTransactionManager transactionManager(DataSource dataSource) {
        return new DataSourceTransactionManager(dataSource);
    }
}
```

#### 使用

**自定义注解**
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@DataSourceType("testDb1")
public @interface TestDb1 {

}

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@DataSourceType("testDb2")
public @interface TestDb2 {

}
```

**Mapper**
```java
@TestDb1
@Mapper
public interface testDb1Mapper {

    List<TestDO> findById(@Param("id") Long id);
}

@TestDb2
@Mapper
public interface testDb2Mapper {

    List<TestDO> findById(@Param("id") Long id);
}
```

**Application:需要去除`DataSourceAutoConfiguration`自动配置**
```java
@EnableDiscoveryClient
@SpringBootApplication
@EnableAutoConfiguration(exclude = DataSourceAutoConfiguration.class)
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

### end

&emsp;&emsp;这篇文章基本贴的就是代码,因为思路比较简单就没有做过多的解释.在写的过程中遇到的一些坑主要是由于对`spring`bean加载的过程不够了解以及对`spring`提供的一些极为好用的util类不熟悉.有时间会深入学习下`spring`.