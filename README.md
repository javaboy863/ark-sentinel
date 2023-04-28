# 1.什么是ark-sentinel？
&emsp;&emsp;ark-sentinel是ark系列框架中的限流降级组件，基于阿里sentinel组件开发，在sentinel的基础上做了封装和功能的增强。
# 2.ark-sentinel解决了什么问题？
&emsp;&emsp;当依赖的服务出现问题或者影响到核心流程时，需要暂时屏蔽掉，待依赖的服务恢复后再重新调用该服务。
&emsp;&emsp;当流量暴涨时，为了保证服务的稳定，通过对接口的限流，以达到保障系统安全的目的。一旦达到限制的阈值，可以拒绝服务、排队或等待、降级等处理。
# 3.功能增强
- 实现用nacos持久化sentinel的配置规则信息，支持Sentinel与Nacos的双向同步持久化。



# 4.接入ark-sentinel
```java
1、  1、引入pom
<properties>
    <sentinel.version>1.8.6</sentinel.version>
</properties>

<dependency>
    <groupId>com.ark.sentinel</groupId>
    <artifactId>sentinel-core</artifactId>
    <version>${sentinel.version}</version>
</dependency>
<!-- Sentinel 接入控制台 -->
<dependency>
    <groupId>com.ark.sentinel</groupId>
    <artifactId>sentinel-transport-simple-http</artifactId>
    <version>${sentinel.version}</version>
</dependency>

<!-- Sentinel 对 SpringMVC 的支持 -->
<dependency>
    <groupId>com.ark.sentinel</groupId>
    <artifactId>sentinel-spring-webmvc-adapter</artifactId>
    <version>${sentinel.version}</version>
</dependency>

<dependency>
    <groupId>com.ark.sentinel</groupId>
    <artifactId>sentinel-annotation-aspectj</artifactId>
    <version>${sentinel.version}</version>
</dependency>

<dependency>
    <groupId>com.ark.sentinel</groupId>
    <artifactId>sentinel-spring-nacos-client</artifactId>
    <version>${sentinel.version}</version>
</dependency>

<!--nacos数据源-->
<dependency>
    <groupId>com.ark.sentinel</groupId>
    <artifactId>sentinel-datasource-nacos</artifactId>
    <version>${sentinel.version}</version>
</dependency>

<!-- 热点规则  可选-->
<dependency>
    <groupId>com.ark.sentinel</groupId>
    <artifactId>sentinel-parameter-flow-control</artifactId>
    <version>${sentinel.version}</version>
</dependency>
<!--支持Web请求进行流量控制 可选-->
<dependency>
    <groupId>com.ark.sentinel</groupId>
    <artifactId>sentinel-web-servlet</artifactId>
    <version>${sentinel.version}</version>
</dependency>


    
2、初始化各个数据源
    @Configuration
    public class SentinelConfiguration {
      /**
       * Nacos 服务器地址
       */
      @Value("${spring.cloud.nacos.config.server-addr}")
      private String serverAddress;
    
      @Value("${spring.application.name}")
      private String application;
    
      @Value("${spring.profiles.active}")
      private String profile;
    
      // Nacos 配置分组
      public static final String group = "SENTINEL_GROUP";
      // Nacos 命名空间
      public static final String namespace = "";
      //限流规则后缀
      public static final String FLOW_DATA_ID_POSTFIX = "-flow-rules";
      //降级规则后缀
      public static final String DEGRADE_DATA_ID_POSTFIX = "-degrade-rules";
      //热点规则后缀
      public static final String PARAM_DATA_ID_POSTFIX = "-param-rules";
      //系统规则后缀
      public static final String SYSTEM_DATA_ID_POSTFIX = "-system-rules";
      //授权规则后缀
      public static final String AUTH_DATA_ID_POSTFIX = "-auth-rules";
      public static final String GATEWAY_FLOW_DATA_ID_POSTFIX = "-gateway-flow-rules";
      public static final String GATEWAY_API_DATA_ID_POSTFIX = "-gateway-api-rules";
    
      /**
       * 初始化数据源
       */
      @Bean
      public NacosDataSource nacosDataSource(ObjectMapper objectMapper) {
        Properties properties = new Properties();
        properties.setProperty(PropertyKeyConst.SERVER_ADDR, serverAddress);
        properties.setProperty(PropertyKeyConst.NAMESPACE, namespace);
    
        initDegradeRule(properties, objectMapper);
        initParamFlowRule(properties, objectMapper);
        initSystemRule(properties, objectMapper);
        initAuthorityRule(properties, objectMapper);
        return initFlowRule(properties, objectMapper);
      }
    
      private NacosDataSource initFlowRule(Properties properties, ObjectMapper objectMapper) {
        String dataId = application + "-" + profile + FLOW_DATA_ID_POSTFIX; // Nacos 配置集编号
        NacosDataSource<List<FlowRule>> nacosDataSource = new NacosDataSource<>(properties, group, dataId,
            new Converter<String, List<FlowRule>>() { // <X> 转换器，将读取的 Nacos 配置，转换成 FlowRule 数组
              @Override
              public List<FlowRule> convert(String value) {
                try {
                  return Arrays.asList(objectMapper.readValue(value, FlowRule[].class));
                } catch (JsonProcessingException e) {
                  throw new RuntimeException(e);
                }
              }
            });
    
        // 注册到 FlowRuleManager 中
        FlowRuleManager.register2Property(nacosDataSource.getProperty());
        return nacosDataSource;
      }

  private void initDegradeRule(Properties properties, ObjectMapper objectMapper) {
    String dataId = application + "-" + profile + DEGRADE_DATA_ID_POSTFIX;
    NacosDataSource<List<DegradeRule>> nacosDataSource = new NacosDataSource<>(properties, group, dataId,
        new Converter<String, List<DegradeRule>>() {
          @Override
          public List<DegradeRule> convert(String value) {
            try {
              return Arrays.asList(objectMapper.readValue(value, DegradeRule[].class));
            } catch (JsonProcessingException e) {
              throw new RuntimeException(e);
            }
          }
        });
    // 注册到 DegradeRuleManager 中
    DegradeRuleManager.register2Property(nacosDataSource.getProperty());
  }

  private void initParamFlowRule(Properties properties, ObjectMapper objectMapper) {
    String dataId = application + "-" + profile + PARAM_DATA_ID_POSTFIX;
    NacosDataSource<List<ParamFlowRule>> nacosDataSource = new NacosDataSource<>(properties, group, dataId,
        value -> {
          try {
            JSONArray list = JSONArray.parseArray(value);
            if (CollectionUtils.isEmpty(list)) {
              return Arrays.asList(new ParamFlowRule());
            }

            List<ParamFlowRule> paramFlowRuleList = list.stream().map(item -> (JSONObject) item).map(item -> {
              ParamFlowRule paramFlowRule = new ParamFlowRule();
              paramFlowRule.setId(item.getLongValue("id"));
              JSONObject rule = item.getJSONObject("rule");
              paramFlowRule.setGrade(rule.getIntValue("grade"));
              paramFlowRule.setParamIdx(rule.getIntValue("paramIdx"));

              paramFlowRule.setCount(rule.getDoubleValue("count"));
              paramFlowRule.setControlBehavior(rule.getIntValue("controlBehavior"));
              paramFlowRule.setMaxQueueingTimeMs(rule.getIntValue("maxQueueingTimeMs"));
              paramFlowRule.setBurstCount(rule.getIntValue("burstCount"));
              paramFlowRule.setDurationInSec(rule.getLong("durationInSec"));
              paramFlowRule.setClusterMode(rule.getBoolean("clusterMode"));
              paramFlowRule.setResource(rule.getString("resource"));
              paramFlowRule.setLimitApp(rule.getString("limitApp"));

              return paramFlowRule;
            }).collect(Collectors.toList());

            // Arrays.asList(objectMapper.readValue(value, ParamFlowRule[].class));
            return paramFlowRuleList;
          } catch (Exception e) {
            throw new RuntimeException(e);
          }
        });

    // 注册到 ParamFlowRuleManager 中
    ParamFlowRuleManager.register2Property(nacosDataSource.getProperty());
  }



  private void initSystemRule(Properties properties, ObjectMapper objectMapper) {
    String dataId = application + "-" + profile + SYSTEM_DATA_ID_POSTFIX;
    NacosDataSource<List<SystemRule>> nacosDataSource = new NacosDataSource<>(properties, group, dataId,
        value -> {
          try {
            return Arrays.asList(objectMapper.readValue(value, SystemRule[].class));
          } catch (JsonProcessingException e) {
            throw new RuntimeException(e);
          }
        });

    // 注册到 SystemRuleManager 中
    SystemRuleManager.register2Property(nacosDataSource.getProperty());
  }

  private void initAuthorityRule(Properties properties, ObjectMapper objectMapper) {
    String dataId = application + "-" + profile + AUTH_DATA_ID_POSTFIX;
    NacosDataSource<List<AuthorityRule>> nacosDataSource = new NacosDataSource<>(properties, group, dataId,
        value -> {
          try {
            return Arrays.asList(objectMapper.readValue(value, AuthorityRule[].class));
          } catch (JsonProcessingException e) {
            throw new RuntimeException(e);
          }
        });
    // 注册到 AuthorityRuleManager 中
    AuthorityRuleManager.register2Property(nacosDataSource.getProperty());
  }
}


 3、如果要使用@SentinelResource注解的方式，需要注册一个 Spring Bean（可选）
@Configuration
public class SentinelAspectConfiguration {
  @Bean
  public SentinelResourceAspect sentinelResourceAspect(){
    return new SentinelResourceAspect();
  }
}


4 、springmvc项目中如果希望请求的path作为资源保护名称 需注入Bean(可选)
@Configuration
public class FilterConfig {

  @Bean
  public FilterRegistrationBean sentinelFilterRegistration() {
    FilterRegistrationBean<Filter> registration = new FilterRegistrationBean<>();
    registration.setFilter(new CommonFilter());
    registration.addUrlPatterns("/*");
    registration.setName("sentinelFilter");
    registration.setOrder(1);
    return registration;
  }
}


5、jvm中添加启动参数：
    测试环境启动添加参数：-Dcsp.sentinel.dashboard.server=sentinel.ark.net   -Dproject.name=distribution（示例，需要根据实际环境设定）
    参数说明：
    -Dcsp.sentinel.dashboard.server=  “sentinel-dashboard 的 IP”
    -Dproject.name= “项目名”-“环境名”

6、打开控制台配置规则 ：sentinel.xxx.net
    注：需要部署服务到测试环境才能在控制台看到，本机环境不行。


7、限流降级使用规范
    研发中——《业务系统限流降级》规范



```

