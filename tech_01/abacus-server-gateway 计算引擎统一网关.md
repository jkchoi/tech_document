## 一、abacus 服务
### 配置单仓多应用

abacus-server 通过“单仓多应用”的方式，实现共用一个 git 代码仓库的同时部署多个应用，不同的应用部署到不同的服务器实现物理隔离。

1. 根据实际情况确认需要的应用数量、应用名称，例如：abacus_trade、abacus_gauss、abacus_batch，在乐效进行“单仓多应用”的配置
2. 在 hippo 中根据上述应用所在的 set，创建对应的 cluster 集群配置，设置各应用需要对外暴露的接口
3. 各应用启动时根据所属机器的 set 加载 hippo 对应集群的服务暴露配置，只暴露指定的接口到 zookeeper，例如：computingTrade、computingGauss、computingBatch
4. **兼容现有计算逻辑**，应用在暴露自身指定的接口的同时，**保持原 computing 接口**，用于支持存量业务场景，后续再与调用方沟通逐步切量到网关的方式
### abacus-server
#### 1.代理接口

![image.png](https://raw.githubusercontent.com/jkchoi/tech_document/main/tech_01/20240429112844.png)

新增代理接口 ComputingProxyService，方法签名同 ComputingService，并在 dubbo-provider.xml 中暴露
```JAVA
/**  
 * 计费引擎接口代理  
 *  
 * @author jkchoi  
 */
 public interface ComputingProxyService {  
  
    /**  
     * 计费引擎接口  
     *  
     * @param computingRequest  
     * @return  
     */  
    ComputingResponse computing(ComputingRequest computingRequest);  
}
```

同时创建三个子接口用于区分 abacus_trade、abacus_gauss、abacus_batch
```JAVA
public interface ComputingProxy4TradeService extends ComputingProxyService {
}

public interface ComputingProxy4GaussService extends ComputingProxyService {
}

public interface ComputingProxy4BatchService extends ComputingProxyService {
}
```

#### 2.代理实现
1. ComputingProxyServiceImpl 通过静态代理实现对 ComputingService 的包装
2. 创建 ComputingProxyServiceImpl 的子类用于区分 abacus_trade、abacus_gauss、abacus_batch
```JAVA
/**  
 * 计费引擎接口代理实现  
 *  
 * @author jkchoi  
 */
@Service("computingProxyService")  
public class ComputingProxyServiceImpl implements ComputingProxyService {  
  
    private final ComputingService computingService;  
  
    public ComputingProxyServiceImpl(ComputingService computingService) {  
        this.computingService = computingService;  
    }  
  
    @Override  
    public ComputingResponse computing(ComputingRequest computingRequest) {  
        return computingService.computing(computingRequest);  
    }  
  
}

/**  
 * 计费引擎接口代理实现-实时计费_交易计费  
 *  
 * @author jkchoi  
 */
@Service("computingProxy4TradeService")  
public class ComputingProxy4TradeServiceImpl extends ComputingProxyServiceImpl implements ComputingProxy4TradeService {  
  
    public ComputingProxy4TradeServiceImpl(ComputingService computingService) {  
        super(computingService);  
    }  
}

@Service("computingProxy4GaussService")  
public class ComputingProxy4GaussServiceImpl extends ComputingProxyServiceImpl implements ComputingProxy4GaussService {  
  
    public ComputingProxy4GaussServiceImpl(ComputingService computingService) {  
        super(computingService);  
    }  
}

@Service("computingProxy4BatchService")  
public class ComputingProxy4BatchServiceImpl extends ComputingProxyServiceImpl implements ComputingProxy4BatchService {  
  
    public ComputingProxy4BatchServiceImpl(ComputingService computingService) {  
        super(computingService);  
    }  
}
```

#### 3.定义服务暴露配置
```properties
[
    {
        # 被替换的接口路径
        "interfaceClassPath": "com.fenqile.abacus.engine.proxy.ComputingProxyService",
        # 原始实现类路径
        "implClassPath": "com.fenqile.abacus.service.engine.proxy.ComputingProxyServiceImpl",
        # 替换接口路径
        "replaceInterfaceClassPath": "com.fenqile.abacus.engine.proxy.ComputingProxy4TradeService",
        # 替换实现类路径
        "replaceImplClassPath": "com.fenqile.abacus.service.engine.proxy.ComputingProxy4TradeServiceImpl"
    }
]
```

```JAVA
/**  
 * 替换bean配置  
 *  
 * @author jkchoi  
 */
@Data  
public class ReplaceBeanProperties {  
  
    /**  
     * 被替换的接口路径  
     */  
    private String interfaceClassPath;  
  
    /**  
     * 原始实现类路径  
     */  
    private String implClassPath;  
  
    /**  
     * 替换接口路径  
     */  
    private String replaceInterfaceClassPath;  
  
    /**  
     * 替换实现类路径  
     */  
    private String replaceImplClassPath;
}
```
#### 4.动态替换dubbo暴露的接口和实现
动态暴露
1. 从hippo加载服务替换配置，为空则不对代理接口 ComputingProxyService 进行替换
2. 在 dubbo 将 ComputingProxyService 暴露注册到 zk 之前，通过 Spring 的 BeanPostProcessor，对暴露的属性进行替换
	- 判断当前 Bean 是否为 dubbo 的 ServiceBean，否则直接 return
	- 判断当前 ServiceBean 的实现的接口是否与 hippo 的配置一致，否则直接 return
	- 判断配置中的原始实现类路径、替换实现类路径是否为空，是则启动失败，否则根据类路径获取原始实现类、替换实现类的 Class 对象
	- 判断当前 ServiceBean 的 ref 属性是否等于原始实现类的 Class，是则从上下文applicationContext 中获取被替换实现类的实例，替换 ServiceBean 的 interface、ref 属性；否则直接 return

![image.png|400](https://raw.githubusercontent.com/jkchoi/tech_document/main/tech_01/20240429113425.png)

```java
/**  
 * dubbo 服务替换后置处理器  
 *  
 * @author jkchoi  
 */
@Component  
public class DubboServiceReplaceBeanPostProcessor implements BeanPostProcessor, ApplicationContextAware {  
  
    private static final Logger LOGGER = LoggerFactory.getLogger(DubboServiceReplaceBeanPostProcessor.class);  
  
    /**  
     * dubbo服务动态替换配置key  
     */    
     private static final String REPLACE_BEAN_CONFIG_HIPPO_KEY = "replace.bean.config";  
  
    private ApplicationContext applicationContext;  
  
    private ConfigUtil mConfigUtil = ApolloInjector.getInstance(ConfigUtil.class);  
  
    private List<ReplaceBeanProperties> replaceBeanPropertiesList = new ArrayList<>();  
  
    @PostConstruct  
    public void init() {  
        String appId = mConfigUtil.getAppId();  
        String apolloEnv = mConfigUtil.getApolloEnv().getName();  
        String apolloDataCenter = mConfigUtil.getDataCenter();  
        String apolloCluster = mConfigUtil.getCluster();  
  
        String replaceConfigInfo = HippoConfigGetUtil.getStringApplication(this.REPLACE_BEAN_CONFIG_HIPPO_KEY);  
        LOGGER.info("init DubboServiceReplaceBeanPostProcessor, appId={}, apolloEnv={}, apolloDataCenter={}, apolloCluster={}, replaceConfigInfo={}",  
                appId, apolloEnv, apolloDataCenter, apolloCluster, replaceConfigInfo);  
        if (StringUtils.isBlank(replaceConfigInfo)) {  
            LOGGER.warn("replace.bean.config is empty, skip replace");  
            return;  
        }  
  
        // 加载替换配置  
        replaceBeanPropertiesList = JSONObject.parseObject(replaceConfigInfo, new TypeReference<List<ReplaceBeanProperties>>(){});  
        LOGGER.info("load replace bean properties: ", replaceBeanPropertiesList);  
    }  
  
    @Override  
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {  
        if (!(bean instanceof ServiceBean)) {  
            return bean;  
        }  
  
        ServiceBean serviceBean = (ServiceBean) bean;  
  
        // 遍历配置进行服务替换  
        for (ReplaceBeanProperties replaceBeanProperties : replaceBeanPropertiesList) {  
            String interfaceClassPath = replaceBeanProperties.getInterfaceClassPath();  
            String implClassPath = replaceBeanProperties.getImplClassPath();  
  
            if (!Objects.equals(serviceBean.getInterface(), interfaceClassPath)) {  
                // 非指定接口则跳过  
                continue;  
            }  
  
            handleReplaceInfo(serviceBean, implClassPath, replaceBeanProperties);  
        }  
  
        return bean;  
    }  
  
    @Override  
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {  
        return bean;  
    }  
  
    @Override  
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {  
        this.applicationContext = applicationContext;  
    }  
  
    /**  
     * 处理替换信息  
     *  
     * @param serviceBean               服务Bean  
     * @param replaceBeanProperties     替换信息  
     * @author jkchoi  
     **/    
     private void handleReplaceInfo(ServiceBean serviceBean, String implClassPath, ReplaceBeanProperties replaceBeanProperties) {  
        String replaceImplClassPath = replaceBeanProperties.getReplaceImplClassPath();  
  
        if (StringUtils.isBlank(implClassPath) || StringUtils.isBlank(replaceImplClassPath)) {  
            LOGGER.error("请检查 dubbo service 替换配置, implClassPath={}, replaceBeanProperties={}", implClassPath, JSONObject.toJSONString(replaceBeanProperties));  
            throw new RuntimeException("请检查 dubbo service 替换配置");  
        }  
  
        // 替换服务实现类  
        replaceRef(serviceBean, implClassPath, replaceImplClassPath);  
    }  
  
    /**  
     * 替换服务实现类  
     *  
     * @param serviceBean  
     * @param originalImplClassPath  
     * @param replaceImplClassPath  
     * @author jkchoi  
     **/    
     private void replaceRef(ServiceBean serviceBean, String originalImplClassPath, String replaceImplClassPath) {  
        Class originalClass;  
        Class replaceClass;  
        try {  
            originalClass = Class.forName(originalImplClassPath);  
            replaceClass = Class.forName(replaceImplClassPath);  
  
            // 检查并替换目标服务实现类  
            if (Objects.equals(serviceBean.getRef().getClass(), originalClass)) {  
                // 创建或获取新的服务实现类实例  
                Object replaceBean = applicationContext.getBean(replaceClass);  
  
                LOGGER.info("ready to replace, ref=[{}] to [{}], impl=[{}] to [{}]",  
                        serviceBean.getInterface(), replaceBean.getClass().getInterfaces()[0], originalClass.getSimpleName(), replaceClass.getSimpleName());  
                // 替换服务接口、引用  
                serviceBean.setInterface(replaceBean.getClass().getInterfaces()[0]);  
                serviceBean.setRef(replaceBean);  
            }  
        } catch (ClassNotFoundException e) {  
            LOGGER.error("请检查 dubbo service 替换配置", e);  
            throw new RuntimeException(e);  
        } catch (Exception e) {  
            LOGGER.error("替换服务实现失败", e);  
            throw new RuntimeException(e);  
        }  
    }  
  
}
```

## 二、网关

### 2.1 dubbo-consumer.xml
订阅 ComputingProxy4TradeService、ComputingProxy4GaussService、ComputingProxy4BatchService

```XML
<!-- computing trade 接口 -->  
<dubbo:reference id="computingProxy4TradeService"  
                 interface="com.fenqile.abacus.engine.proxy.ComputingProxy4TradeService"  
                 timeout="60000"  
                 check="false"  
                 protocol="fsof"  
                 group="*"  
                 version="1.0.0"  
                 openfastjsonsupport="true"  
                 retries="2"  
/>  
  
<!-- computing trade 接口 -->  
<dubbo:reference id="computingProxy4GaussService"  
                 interface="com.fenqile.abacus.engine.proxy.ComputingProxy4GaussService"  
                 timeout="60000"  
                 check="false"  
                 protocol="fsof"  
                 group="*"  
                 version="1.0.0"  
                 openfastjsonsupport="true"  
                 retries="2"  
/>  
  
<!-- computing trade 接口 -->  
<dubbo:reference id="computingProxy4BatchService"  
                 interface="com.fenqile.abacus.engine.proxy.ComputingProxy4BatchService"  
                 timeout="60000"  
                 check="false"  
                 protocol="fsof"  
                 group="*"  
                 version="1.0.0"  
                 openfastjsonsupport="true"  
                 retries="2"  
/>
```

### 2.2 dubbo-provider.xml
对外提供网关接口

```XML
<dubbo:service interface="com.fenqile.abacus.gateway.service.ComputingGatewayService"  
               timeout="10000"  
               ref="computingGatewayService"  
               protocol="fsof"  
               group="default"  
               version="1.0.0"  
               openfastjsonsupport="true"  
/>
```

通过请求参数的 sceneId 场景id路由转发到对应的接口
在 `ComputingProxy4Service` 的基础上，使用 `ComputingGatewayRequest`、`ComputingGatewayResponse` 封装请求和返回，方便后续做一定的扩展
```JAVA
/**  
 * 计费引擎接口实现（路由&转发）  
 *  
 * @author jkchoi  
 */
@Service("computingGatewayService")  
public class ComputingGatewayServiceImpl implements ComputingGatewayService {  
  
    private static final Logger LOGGER = LoggerFactory.getLogger(ComputingGatewayServiceImpl.class);  
  
    private final ComputingRouteConfig computingRouteConfig;  
  
    public ComputingGatewayServiceImpl(ComputingRouteConfig computingRouteConfig) {  
        this.computingRouteConfig = computingRouteConfig;  
    }  
  
    @Override  
    public ComputingGatewayResponse computing(ComputingGatewayRequest computingGatewayRequest) {  
        long start = System.currentTimeMillis();  
  
        LOGGER.info("computingGatewayRequest: {}", JSON.toJSONString(computingGatewayRequest));  
        if (Objects.isNull(computingGatewayRequest)) {  
            LOGGER.warn("请求参数为空, 直接返回空响应");  
            return new ComputingGatewayResponse();  
        }  
  
        // 根据请求获取场景id  
        Integer sceneId = computingRouteConfig.getSceneId(computingGatewayRequest);  
        ComputingGatewayResponse computingGatewayResponse = new ComputingGatewayResponse();  
        try {  
            // 获取对应的计算服务  
            ComputingProxyService computingProxyService = computingRouteConfig.getComputingProxyService(sceneId);  
  
            // 从网关请求转为对应服务的请求，目前这里直接复制  
            ComputingRequest computingRequest = new ComputingRequest();  
            BeanUtils.copyProperties(computingGatewayRequest, computingRequest);  
  
            // 路由&转发  
            ComputingResponse computingResponse = computingProxyService.computing(computingRequest);  
            if (Objects.nonNull(computingResponse)) {  
                BeanUtils.copyProperties(computingResponse, computingGatewayResponse);  
            }  
        } catch (BizException e) {  
            LOGGER.error("计费异常", e);  
            ReportUtils.sendReport(ComputingRouterConstant.REPORT_TAG_COMPUTING_ERROR, sceneId, false);  
            throw e;  
        } catch (Exception e) {  
            LOGGER.error("计费异常", e);  
            ReportUtils.sendReport(ComputingRouterConstant.REPORT_TAG_COMPUTING_ERROR, sceneId, false);  
            throw new BizException(BizExceptionEnum.SERVER_ERROR);  
        }  
  
        LOGGER.info("cost={}ms, sceneId={}, computingGatewayResponse={}", (System.currentTimeMillis() - start), sceneId, JSON.toJSONString(computingGatewayResponse));  
        ReportUtils.sendReport(ComputingRouterConstant.REPORT_TAG_COMPUTING_SUCCESS, sceneId, true);  
        return computingGatewayResponse;  
    }
  
}
```

### 2.3 路由配置加载
![image.png](https://raw.githubusercontent.com/jkchoi/tech_document/main/tech_01/20240429134044.png)
定义路由配置类 ComputingRouteConfig

1. 通过 Spring 依赖注入将构建服务id与 ComputingProxyService 的映射
2. 从 hippo 加载路由配置，**如果为空**则启动失败
3. 将路由配置转为对象，构建以 **key=场景id，value=服务id** 的路由配置 Map
4. 启动 hippo 配置变更监听，实时更新路由配置 Map  
    通过 AtomicReference 更新整个路由配置 Map，保证在网关并发读取的情况下安全的更新。

#### 2.3.1 Spring Service 注入到 Map
通过 Spring 将实现了 ComputingProxyService 接口的服务注入到 computingProxyServiceMap，记录服务id与实现类的映射关系
```java
/**  
 * 服务id与实现类的映射关系  
 */  
private final Map<String, ComputingProxyService> computingProxyServiceMap;  
  
public ComputingRouteConfig(Map<String, ComputingProxyService> computingProxyServiceMap) {  
    this.computingProxyServiceMap = computingProxyServiceMap;  
}
```


#### 2.3.2 刷新场景id与服务id的映射关系

代码如下：
```JAVA
/**  
 * 场景id与服务id的映射关系  
 */  
private final AtomicReference<Map<Integer, String>> sceneIdsMapRef = new AtomicReference<>();

/**  
 * 刷新路由配置  
 *  
 * @param value         路由配置字符串  
 * @author jkchoi  
 **/
private void refreshRouteConfig(String value) {  
    LOGGER.info("准备刷新建路由配置 value: {}", value);  
    if (StringUtils.isBlank(value)) {  
        String errorMsg = "不支持使用空的路由配置";  
        throw new BizException(BizExceptionEnum.ROUTE_CONFIG_REFRESH_ERROR, errorMsg);  
    }  
  
    Map<Integer, String> oldRouteConfig = sceneIdsMapRef.get();  
    // 原本使用 ConcurrentHashMap，考虑到网关只会对该 Map 进行只读操作，所以改成 HashMap
    Map<Integer, String> newRouteConfig = new HashMap<>();  
    try {  
        List<ComputingRouteConfigProperties> propertiesList = JSON.parseObject(value, new TypeReference<List<ComputingRouteConfigProperties>>() {});  
        for (ComputingRouteConfigProperties properties : propertiesList) {  
            properties.getSceneIdList().forEach(sceneId -> newRouteConfig.put(sceneId, properties.getServiceId()));  
        }  
  
        LOGGER.info("刷新路由配置 oldRouteConfig：{}, newRouteConfig: {}", JSONObject.toJSONString(oldRouteConfig), JSONObject.toJSONString(newRouteConfig));  
        sceneIdsMapRef.compareAndSet(oldRouteConfig, newRouteConfig);  
    } catch (Exception e) {  
        String errorMsg = "请检查配置 value: " + value;  
        LOGGER.error(errorMsg, e);  
        ReportUtils.sendReport(ComputingRouterConstant.REPORT_TAG_REFRESH_ROUTER_CONFIG_ERROR, false);  
        throw new BizException(BizExceptionEnum.ROUTE_CONFIG_REFRESH_ERROR, errorMsg);  
    }  
  
    ReportUtils.sendReport(ComputingRouterConstant.REPORT_TAG_REFRESH_ROUTER_CONFIG_ERROR, true);  
}
```
1. 通过 `ConcurrentHashMap` 记录场景id与服务id的映射关系，实现线程安全的并发读写
2. 使用 `AtomicReference<ConcurrentHashMap<String, String>>` 的目的是为了在多线程环境下原子性地更新整个缓存映射表（CAS）
> 原本使用 ConcurrentHashMap，考虑到网关只会对该 Map 进行只读操作，所以改成 HashMap

##### 为什么使用 `AtomicReference`
虽然 `ConcurrentHashMap` 本身已经支持并发读写，但它并不能保证对整个引用对象（即整个 Map 实例）的替换操作是原子性的。
在某些场景下，如果需要一次性替换整个缓存内容，而不是单个条目的修改，使用 `AtomicReference` 就显得更为合适。

例如，在配置发生变更时，可能希望创建一个新的 `ConcurrentHashMap` 并将所有新的配置项加载到其中，然后通过原子性地更新 `AtomicReference` 中的引用，确保其他线程能够看到完整的、最新的配置集合。

总结来说
   - 对于内部元素（配置项）的并发控制，确实可以直接依赖于 `ConcurrentHashMap`；
   - 而对于整体容器实例的替换操作，使用 `AtomicReference` 可以提供额外的原子性保障。

因此，具体是否有必要使用 `AtomicReference`，取决于你的应用场景和需求。


#### 2.3.3 监听配置变更
通过 `ConfigChangeListener` 监听 Hippo 配置中心（底层即 Apollo）的配置变更，并触发刷新场景id与服务id的映射关系，实现路由配置的动态更新

```JAVA
private final Config config = ConfigService.getAppConfig();

// 监听配置变更  
config.addChangeListener(changeEvent -> {  
    for (String key : changeEvent.changedKeys()) {  
        if (Objects.equals(ROUTE_CONFIG_KEY, key)) {  
            ConfigChange change = changeEvent.getChange(key);  
            refreshRouteConfig(change.getNewValue());  
        }  
    }  
});
```


#### 2.3.4 根据场景id获取计算服务
根据场景id，路由获取对应的 dubbo 计算服务

```JAVA
/**  
 * 根据场景id获取计算服务  
 *  
 * @param sceneId           场景id  
 * @return                  场景对应的计算服务  
 * @author jkchoi  
 **/
public ComputingProxyService getComputingProxyService(Integer sceneId) {  
    String serviceId = sceneIdsMapRef.get().get(sceneId);  
    LOGGER.info("sceneId={}, serviceId={}", sceneId, serviceId);  
  
    if (Objects.isNull(serviceId) || Objects.isNull(computingProxyServiceMap.get(serviceId))) {  
        String errorMsg = String.format("当前场景 sceneId=%d 未匹配到对应的 dubbo 服务，请检查配置", sceneId);  
        throw new BizException(BizExceptionEnum.ROUTE_COMPUTING_ERROR, errorMsg);  
    }  
  
    return computingProxyServiceMap.get(serviceId);  
}

/**  
 * 获取场景id  
 * * @param request       请求  
 * @return              场景id  
 * @author jkchoi  
 **/
public Integer getSceneId(ComputingGatewayRequest request) {  
    int sceneId = request.getSceneId();  
    String sceneBizId = request.getSceneBizId();  
  
    if (sceneId <= 0) {  
        // 如果 sceneId 无效，则根据场景业务id 获取 sceneId        
        sceneId = ConfigService.getConfig(ComputingRouterConstant.SCENE_MAP_NAMESPACE).getIntProperty(sceneBizId, 0);  
        LOGGER.info("req.sceneId={}, try to get sceneId from namespace={} by sceneBizId={}, current.sceneId={}",  
                request.getSceneId(), ComputingRouterConstant.SCENE_MAP_NAMESPACE, sceneBizId, sceneId);  
    }  
  
    return sceneId;  
}
```

#### 2.3.5 完整示例

```JAVA
/**  
 * 计费路由配置  
 *  
 * @author jkchoi  
 */
@Configuration  
public class ComputingRouteConfig {  
  
    private static final Logger LOGGER = LoggerFactory.getLogger(ComputingRouteConfig.class);  
  
    private final Config config = ConfigService.getAppConfig();  
  
    private ConfigUtil mConfigUtil = ApolloInjector.getInstance(ConfigUtil.class);  
  
    /**  
     * 服务id与实现类的映射关系  
     */  
    private final Map<String, ComputingProxyService> computingProxyServiceMap;  
  
    /**  
     * 场景id与服务id的映射关系  
     */  
    private final AtomicReference<Map<Integer, String>> sceneIdsMapRef = new AtomicReference<>();  
  
    public ComputingRouteConfig(Map<String, ComputingProxyService> computingProxyServiceMap) {  
        this.computingProxyServiceMap = computingProxyServiceMap;  
    }  
  
    /**  
     * 初始化  
     */  
    @PostConstruct  
    public void init() {  
        String appId = mConfigUtil.getAppId();  
        String apolloEnv = mConfigUtil.getApolloEnv().getName();  
        String apolloDataCenter = mConfigUtil.getDataCenter();  
        String apolloCluster = mConfigUtil.getCluster();  
        LOGGER.info("init ComputingRouteConfig, appId={}, apolloEnv={}, apolloDataCenter={}, apolloCluster={}", appId, apolloEnv, apolloDataCenter, apolloCluster);  
  
        if (MapUtils.isEmpty(computingProxyServiceMap)) {  
            throw new BizException(BizExceptionEnum.INIT_ERROR, "未找到有效的 ComputingProxyService，请检查配置");  
        }  
  
        for (Map.Entry<String, ComputingProxyService> entry : computingProxyServiceMap.entrySet()) {  
            LOGGER.info("build computingProxyServiceMap, serviceId: {}, service interface: {}", entry.getKey(), entry.getValue().getClass().getInterfaces());  
        }  
  
        // 刷新场景id与服务id的映射关系  
        String routeConfigStr = HippoConfig.getProperty(ComputingRouterConstant.ROUTE_CONFIG_KEY, StringUtils.EMPTY);  
        refreshRouteConfig(routeConfigStr);  
  
        // 监听配置变更  
        config.addChangeListener(changeEvent -> {  
            for (String key : changeEvent.changedKeys()) {  
                if (Objects.equals(ComputingRouterConstant.ROUTE_CONFIG_KEY, key)) {  
                    ConfigChange change = changeEvent.getChange(key);  
                    refreshRouteConfig(change.getNewValue());  
                }  
            }  
        });  
    }  
  
    /**  
     * 根据场景id获取计算服务  
     *  
     * @param sceneId           场景id  
     * @return                  场景对应的计算服务  
     * @author jkchoi  
     **/    
    public ComputingProxyService getComputingProxyService(Integer sceneId) {  
        String serviceId = sceneIdsMapRef.get().get(sceneId);  
        LOGGER.info("sceneId={}, serviceId={}", sceneId, serviceId);  
  
        if (Objects.isNull(serviceId) || Objects.isNull(computingProxyServiceMap.get(serviceId))) {  
            String errorMsg = String.format("当前场景 sceneId=%d 未匹配到对应的 dubbo 服务，请检查配置", sceneId);  
            throw new BizException(BizExceptionEnum.ROUTE_COMPUTING_ERROR, errorMsg);  
        }  
  
        return computingProxyServiceMap.get(serviceId);  
    }  
  
    /**  
     * 获取场景id  
     *     
     * @param request       请求  
     * @return              场景id  
     * @author jkchoi  
     **/    
    public Integer getSceneId(ComputingGatewayRequest request) {  
        int sceneId = request.getSceneId();  
        String sceneBizId = request.getSceneBizId();  
  
        if (sceneId <= 0) {  
            // 如果 sceneId 无效，则根据场景业务id 获取 sceneId            
            sceneId = ConfigService.getConfig(ComputingRouterConstant.SCENE_MAP_NAMESPACE).getIntProperty(sceneBizId, 0);  
            LOGGER.info("req.sceneId={}, try to get sceneId from namespace={} by sceneBizId={}, current.sceneId={}",  
                    request.getSceneId(), ComputingRouterConstant.SCENE_MAP_NAMESPACE, sceneBizId, sceneId);  
        }  
  
        return sceneId;  
    }  
  
    /**  
     * 刷新路由配置  
     *  
     * @param value         路由配置字符串  
     * @author jkchoi  
     **/    
    private void refreshRouteConfig(String value) {  
        LOGGER.info("准备刷新建路由配置 value: {}", value);  
        if (StringUtils.isBlank(value)) {  
            String errorMsg = "不支持使用空的路由配置";  
            throw new BizException(BizExceptionEnum.ROUTE_CONFIG_REFRESH_ERROR, errorMsg);  
        }  
  
        Map<Integer, String> oldRouteConfig = sceneIdsMapRef.get();  
        Map<Integer, String> newRouteConfig = new HashMap<>();  
        try {  
            List<ComputingRouteConfigProperties> propertiesList = JSON.parseObject(value, new TypeReference<List<ComputingRouteConfigProperties>>() {});  
            for (ComputingRouteConfigProperties properties : propertiesList) {  
                properties.getSceneIdList().forEach(sceneId -> newRouteConfig.put(sceneId, properties.getServiceId()));  
            }  
  
            LOGGER.info("刷新路由配置 oldRouteConfig：{}, newRouteConfig: {}", JSONObject.toJSONString(oldRouteConfig), JSONObject.toJSONString(newRouteConfig));  
            sceneIdsMapRef.compareAndSet(oldRouteConfig, newRouteConfig);  
        } catch (Exception e) {  
            String errorMsg = "请检查配置 value: " + value;  
            LOGGER.error(errorMsg, e);  
            ReportUtils.sendReport(ComputingRouterConstant.REPORT_TAG_REFRESH_ROUTER_CONFIG_ERROR, false);  
            throw new BizException(BizExceptionEnum.ROUTE_CONFIG_REFRESH_ERROR, errorMsg);  
        }  
  
        ReportUtils.sendReport(ComputingRouterConstant.REPORT_TAG_REFRESH_ROUTER_CONFIG_ERROR, true);  
    }  
  
}
```


### 2.4 计费场景路由
![image.png](https://raw.githubusercontent.com/jkchoi/tech_document/main/tech_01/20240429134417.png)
1. 根据场景id获取对应的计费服务 Service
	- 如果 sceneId 无法路由到 serviceId 则尝试使用 sceneBizId 进行路由 serviceId
	- **serviceId 为空** 或者 serviceId **无法映射到计算服务 Service**，则抛异常并上报雷神
2. 路由转发
	- 入参转换  
	    当前网关的请求参数与计费服务的参数一致，可以在此增加网关特有的参数或者逻辑
	- 使用路由到的 Service 发起计费请求
	- 出参转换  
	    当前网关的返回参数与计费服务的参数一致，可以在此增加网关特有的参数或者逻辑

```java
/**  
 * 计费引擎接口实现（路由&转发）  
 *  
 * @author jkchoi  
 */
@Service("computingGatewayService")  
public class ComputingGatewayServiceImpl implements ComputingGatewayService {  
  
    private static final Logger LOGGER = LoggerFactory.getLogger(ComputingGatewayServiceImpl.class);  
  
    private final ComputingRouteConfig computingRouteConfig;  
  
    public ComputingGatewayServiceImpl(ComputingRouteConfig computingRouteConfig) {  
        this.computingRouteConfig = computingRouteConfig;  
    }  
  
    @Override  
    public ComputingGatewayResponse computing(ComputingGatewayRequest computingGatewayRequest) {  
        long start = System.currentTimeMillis();  
  
        LOGGER.info("computingGatewayRequest: {}", JSON.toJSONString(computingGatewayRequest));  
        if (Objects.isNull(computingGatewayRequest)) {  
            LOGGER.warn("请求参数为空, 直接返回空响应");  
            return new ComputingGatewayResponse();  
        }  
  
        // 根据请求获取场景id  
        Integer sceneId = computingRouteConfig.getSceneId(computingGatewayRequest);  
        ComputingGatewayResponse computingGatewayResponse = new ComputingGatewayResponse();  
        try {  
            // 获取对应的计算服务  
            ComputingProxyService computingProxyService = computingRouteConfig.getComputingProxyService(sceneId);  
  
            // 从网关请求转为对应服务的请求，目前这里直接复制  
            ComputingRequest computingRequest = new ComputingRequest();  
            BeanUtils.copyProperties(computingGatewayRequest, computingRequest);  
  
            // 路由&转发  
            ComputingResponse computingResponse = computingProxyService.computing(computingRequest);  
            if (Objects.nonNull(computingResponse)) {  
                BeanUtils.copyProperties(computingResponse, computingGatewayResponse);  
            }  
        } catch (BizException e) {  
            LOGGER.error("计费异常", e);  
            ReportUtils.sendReport(ComputingRouterConstant.REPORT_TAG_COMPUTING_ERROR, sceneId, false);  
            throw e;  
        } catch (Exception e) {  
            LOGGER.error("计费异常", e);  
            ReportUtils.sendReport(ComputingRouterConstant.REPORT_TAG_COMPUTING_ERROR, sceneId, false);  
            throw new BizException(BizExceptionEnum.SERVER_ERROR);  
        }  
  
        LOGGER.info("cost={}ms, sceneId={}, computingGatewayResponse={}", (System.currentTimeMillis() - start), sceneId, JSON.toJSONString(computingGatewayResponse));  
        ReportUtils.sendReport(ComputingRouterConstant.REPORT_TAG_COMPUTING_SUCCESS, sceneId, true);  
        return computingGatewayResponse;  
    }  
  
}
```

