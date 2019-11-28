---
title: 传统Java Web非Spring Boot项目从Spring Cloud Eureka中获取服务
---

更新日志：
2018/3/3 21:05:43 新建
2018/3/11 7:11:59 新增注册到Eureka、从Eureka注销、新增Feign，更新配置文件，更新代码


部门项目的技术框架从 ZooKeeper+Dubbo 转型为Spring Cloud 微服务，转型顺利、开发方便、使用良好，于是完全废弃了ZooKeeper+Dubbo，而Web端后台管理界面的项目由于种种原因不希望大规模重构为Spring Boot项目，继续保持原有的SSM框架，并使用http调用微服务接口。为避免将微服务地址写死，这就需要Web项目连接到Spring Cloud Eureka 上通过服务名获取微服务真实地址。

<!--more-->

## 项目依赖 ##
		
		<!-- eureka 服务发现 -->
		<dependency>
  			<groupId>com.netflix.eureka</groupId>
  			<artifactId>eureka-client</artifactId>
  			<version>1.7.0</version>
		</dependency>
		<!-- Ribbon 负载均衡 -->
		<dependency>
    		<groupId>com.netflix.ribbon</groupId>
   			<artifactId>ribbon-core</artifactId>
    		<version>${netflix.ribbon.version}</version>
		</dependency>
        <dependency>
            <groupId>com.netflix.ribbon</groupId>
            <artifactId>ribbon-loadbalancer</artifactId>
            <version>${netflix.ribbon.version}</version>
            <exclusions>  
            	<exclusion>  
            		<groupId>io.reactivex</groupId>
  					<artifactId>rxjava</artifactId>  
        		</exclusion>
        	</exclusions>    
        </dependency>
        <dependency>
            <groupId>com.netflix.ribbon</groupId>
            <artifactId>ribbon-eureka</artifactId>
            <version>${netflix.ribbon.version}</version>
        </dependency>
		<!-- Feign 包装http请求 -->
		<dependency>
    		<groupId>io.github.openfeign</groupId>
    		<artifactId>feign-hystrix</artifactId>
    		<version>${netflix.feign.version}</version>
		</dependency>
		<dependency>
    		<groupId>io.github.openfeign</groupId>
    		<artifactId>feign-ribbon</artifactId>
    		<version>${netflix.feign.version}</version>
		</dependency>
		<!-- <dependency>
    		<groupId>io.github.openfeign</groupId>
    		<artifactId>feign-gson</artifactId>
    		<version>${netflix.feign.version}</version>
		</dependency> -->
		<dependency>
    		<groupId>io.github.openfeign</groupId>
    		<artifactId>feign-slf4j</artifactId>
    		<version>${netflix.feign.version}</version>
		</dependency>
		<dependency>
    		<groupId>io.github.openfeign</groupId>
    		<artifactId>feign-jackson</artifactId>
    		<version>${netflix.feign.version}</version>
		</dependency>
		
		<dependency>
    		<groupId>io.reactivex</groupId>
    		<artifactId>rxjava</artifactId>
    		<version>1.1.1</version>
		</dependency>

这里使用netflix项目下的eureka、ribbon、Feign、Hystrix。
ribbon-开头的项目都是同一个版本号，所以就抽取出$`{netflix.ribbon.version}`统一管理。feign-开头的项目也都是同一个的版本号，抽取${netflix.feign.version}统一管理。

需要注意：
1. Maven依赖jar包冲突问题：`rxjava`项目在`ribbon-loadbalancer`和`feign-hystrix`依赖的`hystrix-core`中都有使用。当前最新版2.2.4的`ribbon-loadbalancer`使用`rxjava：1.0.9`。而`feign-hystrix`依赖的`hystrix-core`使用`rxjava：1.1.1`。因为依赖冲突，`ribbon-loadbalancer`中的`rxjava：1.0.9`代替掉了`hystrix-core`中的`rxjava：1.1.1`。这样当程序运行时会疯狂报找不到类Error，找不到`rx/Single`，这个类在2.0.9中并没有，2.1.1中有`hystrix-core`用到了，但是由于依赖冲突使用2.0.9的rxjava没有该类，所以报错。
解决办法:`ribbon-loadbalancer`使用 `exclusion` 排除依赖 `rxjava` 即可。
2. feign-core在中央仓库有两个groupId： `com.netflix.feign` 和 `io.github.openfeign` 。groupId`com.netflix.feign`在2016年7月提交到8.18.0后就没有再提交，而groupId`io.github.openfeign`已经在2018年三月份提交到9.6.0。

## 配置文件 ##
### Ribbon配置 ###

	# ribbon.properties
    # xxx-service对应的微服务名
    xxx-service.ribbon.DeploymentContextBasedVipAddresses=xxx-service
    # 固定写法，xxx-service使用的ribbon负载均衡器
    xxx-service.ribbon.NIWSServerListClassName=com.netflix.niws.loadbalancer.DiscoveryEnabledNIWSServerList
    # 每分钟更新xxx-service对应服务的可用地址列表
    xxx-service.ribbon.ServerListRefreshInterval=60000

### Eureka配置 ###
Eureka默认在classpath中寻找eureka-client.properties配置文件

    # 控制是否注册自身到eureka中，本项目虽然不对外提供服务，但需要Eureka监控，在Eureka列表上显示
    eureka.registration.enabled=true
    # eureka相关配置
	# 默认为true，以实现更好的基于区域的负载平衡。
    eureka.preferSameZone=true
	# 是否要使用基于DNS的查找来确定其他eureka服务器
    eureka.shouldUseDns=false
	# 由于shouldUseDns为false，因此我们使用以下属性来明确指定到eureka服务器的路由（eureka Server地址）
    eureka.serviceUrl.default=http://username:password@localhost:8761/eureka/
    eureka.decoderName=JacksonJson
	# 客户识别此服务的虚拟主机名，这里指的是eureka服务本身
	eureka.vipAddress=XXXplatform
	#服务指定应用名，这里指的是eureka服务本身
	eureka.name=XXXlatform
	#服务将被识别并将提供请求的端口
	eureka.port=8080

## 初始化Ribbon、注册Eureka ##
之前初始化Ribbon、Eureka、注册到Eureka和获取地址的方法写在静态代码块和静态方法中，这样在项目停止时无法从Eureka中取消注册，这会使Eureka进入安全模式，死掉的项目一直显示在Eureka列表中。

继承ServletContextListener，重写contextInitialized、contextDestroyed，在应用上下文启动时初始化Ribbon、Eureka、注册到Eureka，应用上下文销毁时注销Eureka。

	import java.io.IOException;
	import javax.servlet.ServletContextEvent;
	import javax.servlet.ServletContextListener;
	import org.slf4j.Logger;
	import org.slf4j.LoggerFactory;
	import com.netflix.appinfo.ApplicationInfoManager;
	import com.netflix.appinfo.InstanceInfo;
	import com.netflix.config.ConfigurationManager;
	import com.netflix.discovery.DefaultEurekaClientConfig;
	import com.netflix.discovery.DiscoveryManager;

	/**
	 * @ClassName: EurekaInitAndRegisterListener
	 * @Description:  服务器启动初始化Ribbon和注册到Eureka Server
	 * @author SuXun
	 * @date 2018年3月7日 上午9:21:12
	 */
	@SuppressWarnings("deprecation")
	public class EurekaInitAndRegisterListener implements ServletContextListener {
		private static final Logger LOGGER = LoggerFactory.getLogger(EurekaInitAndRegisterListener.class);
		/**
		 * 默认的ribbon配置文件名, 该文件需要放在classpath目录下
		 */
		public static final String RIBBON_CONFIG_FILE_NAME = "ribbon.properties";
	
		@Override
		public void contextInitialized(ServletContextEvent sce) {
			LOGGER.info("开始初始化ribbon");
			try {
				// 加载ribbon配置文件
				ConfigurationManager.loadPropertiesFromResources(RIBBON_CONFIG_FILE_NAME);
			} catch (IOException e) {
				e.printStackTrace();
				LOGGER.error("ribbon初始化失败");
				throw new IllegalStateException("ribbon初始化失败");
			}
			LOGGER.info("ribbon初始化完成");
			// 初始化Eureka Client
			LOGGER.info("Eureka初始化完成,正在注册Eureka Server");
			DiscoveryManager.getInstance().initComponent(new MyInstanceConfig(), new DefaultEurekaClientConfig());
			ApplicationInfoManager.getInstance().setInstanceStatus(InstanceInfo.InstanceStatus.UP);
		}
	
		@Override
		public void contextDestroyed(ServletContextEvent sce) {
			DiscoveryManager.getInstance().shutdownComponent();
		}
	}
    
这里有个自定义的类MyInstanceConfig，这个类作用是将注册到Eureka的hostName从主机名换成IP地址加端口号的形式。

	import java.io.IOException;
	import java.net.Inet4Address;
	import java.net.InetAddress;
	import java.net.NetworkInterface;
	import java.net.UnknownHostException;
	import java.util.Enumeration;
	import org.slf4j.Logger;
	import org.slf4j.LoggerFactory;
	import com.netflix.appinfo.MyDataCenterInstanceConfig;

	/**
	 * @ClassName: MyInstanceConfig
	 * @Description:  
	 * @author SuXun
	 * @date 2018年3月7日 上午9:33:31
	 */
	public class MyInstanceConfig extends MyDataCenterInstanceConfig {
		private static final Logger LOG = LoggerFactory.getLogger(MyInstanceConfig.class); 
		@Override
		public String getHostName(boolean refresh) {
	
			try {
				return findFirstNonLoopbackAddress().getHostAddress();
			} catch (Exception e) {
				return super.getHostName(refresh);
			}
		}
		
		public InetAddress findFirstNonLoopbackAddress() {
			InetAddress result = null;
			try {
				int lowest = Integer.MAX_VALUE;
				for (Enumeration<NetworkInterface> nics = NetworkInterface
						.getNetworkInterfaces(); nics.hasMoreElements();) {
					NetworkInterface ifc = nics.nextElement();
					if (ifc.isUp()) {
						LOG.trace("Testing interface: " + ifc.getDisplayName());
						if (ifc.getIndex() < lowest || result == null) {
							lowest = ifc.getIndex();
						}
						else if (result != null) {
							continue;
						}
	
						// @formatter:off
							for (Enumeration<InetAddress> addrs = ifc
									.getInetAddresses(); addrs.hasMoreElements();) {
								InetAddress address = addrs.nextElement();
								if (address instanceof Inet4Address
										&& !address.isLoopbackAddress()) {
									LOG.trace("Found non-loopback interface: "
											+ ifc.getDisplayName());
									result = address;
								}
							}
						// @formatter:on
					}
				}
			}
			catch (IOException ex) {
				LOG.error("Cannot get first non-loopback address", ex);
			}
	
			if (result != null) {
				return result;
			}
	
			try {
				return InetAddress.getLocalHost();
			}
			catch (UnknownHostException e) {
				LOG.warn("Unable to retrieve localhost");
			}
	
			return null;
		}
	}

虽然`DiscoveryManager.getInstance().initComponent()`方法已经被标记为`@Deprecated`了，但是ribbon的`DiscoveryEnabledNIWSServerList`组件代码中依然是通过`DiscoveryManager`来获取EurekaClient对象的。

## 获取服务地址 ##

	mport java.util.ArrayList;
	import java.util.Collections;
	import java.util.List;
	import org.slf4j.Logger;
	import org.slf4j.LoggerFactory;
	import com.netflix.client.ClientFactory;
	import com.netflix.loadbalancer.DynamicServerListLoadBalancer;
	import com.netflix.loadbalancer.RoundRobinRule;
	import com.netflix.loadbalancer.Server;
	
	/**
	 * @ClassName: AlanServiceAddressSelector
	 * @Description:  获取到目标服务注册在Eureka地址
	 * @author SuXun
	 * @date 2018年3月2日 下午5:23:24
	 */
	public class AlanServiceAddressSelector {
		
		private static final Logger log = LoggerFactory.getLogger(AlanServiceAddressSelector.class);
		private static RoundRobinRule chooseRule = new RoundRobinRule();
	
		/**
		 * 根据轮询策略选择一个地址
		 * @param clientName ribbon.properties配置文件中配置项的前缀名, 如myclient
		 * @return null表示该服务当前没有可用地址
		 */
		public static AlanServiceAddress selectOne(String clientName) {
			// ClientFactory.getNamedLoadBalancer会缓存结果, 所以不用担心它每次都会向eureka发起查询
			@SuppressWarnings("rawtypes")
			DynamicServerListLoadBalancer lb = (DynamicServerListLoadBalancer) ClientFactory
					.getNamedLoadBalancer(clientName);
			Server selected = chooseRule.choose(lb, null);
			if (null == selected) {
				log.warn("服务{}没有可用地址", clientName);
				return null;
			}
			log.debug("服务{}选择结果:{}", clientName, selected);
			return new AlanServiceAddress(selected.getPort(), selected.getHost());
		}
	
		/**
		 * 选出该服务所有可用地址
		 * @param clientName
		 * @return
		 */
		public static List<AlanServiceAddress> selectAvailableServers(String clientName) {
			@SuppressWarnings("rawtypes")
			DynamicServerListLoadBalancer lb = (DynamicServerListLoadBalancer) ClientFactory
					.getNamedLoadBalancer(clientName);
			List<Server> serverList = lb.getServerList(true);
			// List<Server> serverList = lb.getReachableServers();
			if (serverList.isEmpty()) {
				log.warn("服务{}没有可用地址", clientName);
				return Collections.emptyList();
			}
			log.debug("服务{}所有选择结果:{}", clientName, serverList);
			List<AlanServiceAddress> address = new ArrayList<AlanServiceAddress>();
			for (Server server : serverList) {
			address.add(new AlanServiceAddress(server.getPort(), server.getHost()));
			}
			return address;
		}
	}

地址实体类：

    /**
     * @ClassName: AlanServiceAddress
     * @Description:  地址实体类
     * @author SuXun
     * @date 2018年3月2日 下午2:14:17
     */
    public class AlanServiceAddress {
    	private int port;
    	private String host;
    
    	public AlanServiceAddress() {
    	}
    
    	public AlanServiceAddress(int port, String host) {
    		this.port = port;
    		this.host = host;
    	}
    
    	public int getPort() {
    		return port;
    	}
    
    	public void setPort(int port) {
    		this.port = port;
    	}
    
    	public String getHost() {
    		return host;
    	}
    
    	public void setHost(String host) {
    		this.host = host;
    	}
    
    	/**
    	 * 将服务地址转换为 http://主机名:端口/ 的格式
    	 * @return
    	 */
    	@Override
    	public String toString() {
    		StringBuilder sb = new StringBuilder(15 + host.length());
    		sb.append("http://").append(host).append(":").append(port).append("/");
    
    		return sb.toString();
    	}
    }

### 使用方法 ###
    
    // 选择出myclient对应服务全部可用地址
    List<AlanServiceAddress> list = AlanServiceAddressSelector.selectAvailableServers("myclient");
    System.out.println(list);
    
    // 选择出myclient对应服务的一个可用地址(轮询), 返回null表示服务当前没有可用地址
    AlanServiceAddress addr = AlanServiceAddressSelector.selectOne("myclient");
    System.out.println(addr);

## 构建Feign客户端 ##
根据服务名在Eureka获取地址，构建Feign，如果缓存有则返回缓存里的Feign，避免重复构建Feign。

构建Feign方法：

    import java.lang.reflect.Method;
    import java.util.concurrent.ConcurrentHashMap;
    import org.apache.commons.lang3.StringUtils;
    import org.slf4j.LoggerFactory;
    import com.netflix.hystrix.HystrixCommand;
    import com.netflix.hystrix.HystrixCommand.Setter;
    import com.netflix.hystrix.HystrixCommandGroupKey;
    import com.netflix.hystrix.HystrixCommandProperties;
    import com.xxx.platform.common.netflex.eureka.AlanServiceAddress;
    import com.xxx.platform.common.netflex.eureka.AlanServiceAddressSelector;
    import feign.Feign;
    import feign.Logger;
    import feign.Request.Options;
    import feign.Retryer;
    import feign.Target;
    import feign.codec.Decoder;
    import feign.codec.Encoder;
    import feign.hystrix.HystrixFeign;
    import feign.hystrix.SetterFactory;
    import feign.jackson.JacksonDecoder;
    import feign.jackson.JacksonEncoder;
    import feign.slf4j.Slf4jLogger;
    
    /**
     * @ClassName: BaseFeignBuilder
     * @Description: 用于构建Feign
     * @author SuXun
     * @date 2018年3月5日 下午3:50:18
     */
    public class BaseFeignBuilder {
    
    	private static final org.slf4j.Logger log = LoggerFactory.getLogger(BaseFeignBuilder.class);
    	private static ConcurrentHashMap<String, Object> cacheFeignMap = new ConcurrentHashMap<String, Object>();
    	private static ConcurrentHashMap<String, String> cacheAddressMap = new ConcurrentHashMap<String, String>();

    	/**
    	 * 构建HystrixFeign,具有Hystrix提供的熔断和回退功能,JacksonEncoder、JacksonDecoder、Slf4jLogger、Logger.Level.FULL
    	 * @param apiType 使用feign访问的接口类,如MedBodyClient.class
    	 * @param clientName 配置文件中的ribbon client名字
    	 * @param fallback 回退类
    	 * @param url 添加网址
    	 * @return
    	 */
    	public static <T> T buildHystrixFeign(Class<T> apiType, T fallback, String url) {
    		// 之前用GsonEncoder()和GsonDecoder()对Date类型支持不好，改成JacksonEncoder和JacksonDecoder，日期转换正常
    		return buildHystrixFeign(apiType, fallback, url, new JacksonEncoder(), new JacksonDecoder(),
    				new Slf4jLogger(BaseFeignBuilder.class), Logger.Level.FULL);
    	}
    
    	/**
    	 * 构建HystrixFeign,具有Hystrix提供的熔断和回退功能
    	 * @param apiType 使用feign访问的接口类,如MedBodyClient.class
    	 * @param clientName clientName 配置文件中的ribbon client名字
    	 * @param fallback 回退类
    	 * @param url 添加网址
    	 * @param encoder 编码器
    	 * @param decoder 解码器
    	 * @param logger 日志对象
    	 * @param logLevel 日志级别
    	 * @return
    	 */
    	public static <T> T buildHystrixFeign(Class<T> apiType, T fallback, String url, Encoder encoder, Decoder decoder,
    			Logger logger, Logger.Level logLevel) {
    		return HystrixFeign.builder().encoder(encoder).decoder(decoder).logger(logger).logLevel(logLevel)
    				//options添加Feign请求响应超时时间
					.options(new Options(60 * 1000, 60 * 1000)).retryer(Retryer.NEVER_RETRY)
    				.setterFactory(new SetterFactory() {
    					@Override
    					public Setter create(Target<?> target, Method method) {
							//添加Hstrix请求响应超时时间
    						return HystrixCommand.Setter
    								.withGroupKey(HystrixCommandGroupKey.Factory.asKey(apiType.getClass().getSimpleName()))
    								.andCommandPropertiesDefaults(
    										HystrixCommandProperties.Setter().withExecutionTimeoutInMilliseconds(60 * 1000) // 超时配置
    						);
    					}
    				}).target(apiType, url, fallback);
    	}
   
    	/**
		 * 获取HystrixFeign。缓存有在缓存取，缓存没有重新构建Feign
    	 * @param apiType 使用feign访问的接口类,如MedBodyClient.class
    	 * @param clientName clientName 配置文件中的ribbon client名字
    	 * @param separator 添加网址分割
    	 * @return
    	 */
    	@SuppressWarnings("unchecked")
    	public static <T> T getCacheFeign(Class<T> apiType, String clientName, T fallback, String separator) {
    		
			String resultAddress = getResultAddress(clientName);
    
    		String cacheKey = apiType.getName() + "-" + clientName + "-" + fallback.getClass().getName() + "-"
    				+ resultAddress + separator;
    		Object cacheFeign = cacheFeignMap.get(cacheKey);
    		if (cacheFeign == null) {
    			T buildFeign = buildHystrixFeign(apiType, fallback, resultAddress + separator);
    			cacheFeignMap.put(cacheKey, buildFeign);
    			return buildFeign;
    		} else {
    			return (T) cacheFeign;
    		}
    	}
    	/**
		 * 获取服务地址，取不到最新地址在缓存取旧地址，有新地址则返回新地址并刷新缓存
		 * @param clientName
	 	 * @return
	 	 */
    	public static String getResultAddress(String clientName) {
    		String recentAddress = null;
    		AlanServiceAddress alanServiceAddress = AlanServiceAddressSelector.selectOne(clientName);
    		recentAddress = alanServiceAddress == null ? "" : alanServiceAddress.toString();
    		String cacheAddress = cacheAddressMap.get(clientName);
    		String resultAddress = "";
    
    		if (StringUtils.isBlank(recentAddress)) {
    			if (StringUtils.isBlank(cacheAddress)) {
    				log.error("服务" + clientName + "无可用地址");
    				throw new RuntimeException("服务" + clientName + "无可用地址");
    			} else {
    				resultAddress = cacheAddress;
    			}
    		} else {
    			resultAddress = recentAddress;
    			cacheAddressMap.put(clientName, recentAddress);
    		}
    		return resultAddress;
    	}
    }

## 通用Feign接口 ##
利用Feign继承特性，特殊需求接口只要继承通用接口就可获得访问通用接口的能力.

    import java.util.Map;
    import com.xxx.commons.base.ResultJsonEntity;
    import feign.Headers;
    import feign.Param;
    import feign.RequestLine;
    
    /**
     * @ClassName: BaseFeignClient
     * @Description: Feign基类
     * @author SuXun
     * @date 2018年3月5日 下午1:25:59
     */
    // @Herders里边的键值对冒号后面必须有个空格！
    @Headers({ "Content-Type: application/json", "Accept: application/json" })
    public interface BaseFeignClient {
    
    	@RequestLine("POST /select")
    	ResultJsonEntity select(Object obj);
    
    	@RequestLine("GET /selectAll")
    	ResultJsonEntity selectAll();
    
    	// 因为Example无法被序列化成json，所以参数为Map
    	@RequestLine("POST /selectByExample")
    	ResultJsonEntity selectByExample(Map<String, Object> map);
    
    	@RequestLine("GET /selectByPrimaryKey/{key}")
    	ResultJsonEntity selectByPrimaryKey(@Param("key") String key);
    
    	@RequestLine("POST /insertSelective")
    	ResultJsonEntity insertSelective(Object obj);
    
    	@RequestLine("POST /updateByPrimaryKeySelective")
    	ResultJsonEntity updateByPrimaryKeySelective(Object obj);
    
    	/**
    	 * @param map key包含：int pageNum,int rowNum,T record和查询条件
    	 * @return
    	 */
    	@RequestLine("POST /getPageExampleList")
    	ResultJsonEntity getPageExampleList(Map<String, Object> map);
    
    }

## 通用回退类 ##
因为使用HystrixClient.build()，使得Feign拥有熔断器、回退的功能。这里根据通用接口实现的回退类。
这里的ResultJsonEntity、ResultEnum、ResultJsonUtil用于返回平台无关的json数据。

    import java.util.Map;
    import org.slf4j.Logger;
    import org.slf4j.LoggerFactory;
    import com.xxx.commons.base.ResultJsonEntity;
    import com.xxx.commons.enums.ResultEnum;
    import com.xxx.commons.util.ResultJsonUtil;
    
    /**
     * @ClassName: BaseFeignClientFallback
     * @Description:  Feign基类的回退类
     * @author SuXun
     * @date 2018年3月5日 下午5:04:13
     */
    public class BaseFeignClientFallback implements BaseFeignClient {
    
    	protected static final Logger LOG = LoggerFactory.getLogger(BaseFeignClientFallback.class);
    
    	@Override
    	public ResultJsonEntity select(Object obj) {
    		LOG.error("{} select 出错 进入熔断 ", this.getClass().getName());
    		return ResultJsonUtil.returnResult(ResultEnum.FAIL);
    	}
    
    	@Override
    	public ResultJsonEntity selectAll() {
    		LOG.error("{} selectAll 出错 进入熔断", this.getClass().getName());
    		return ResultJsonUtil.returnResult(ResultEnum.FAIL);
    	}
    
    	@Override
    	public ResultJsonEntity selectByExample(Map<String, Object> map) {
    		LOG.error("{} selectByMap 出错 进入熔断 ", this.getClass().getName());
    		return ResultJsonUtil.returnResult(ResultEnum.FAIL);
    	}
    
    	@Override
    	public ResultJsonEntity selectByPrimaryKey(String key) {
    		LOG.error("{} selectByPrimaryKey 出错 进入熔断 ", this.getClass().getName());
    		return ResultJsonUtil.returnResult(ResultEnum.FAIL);
    	}
    
    	@Override
    	public ResultJsonEntity insertSelective(Object obj) {
    		LOG.error("{} insertSelective 出错 进入熔断 ", this.getClass().getName());
    		return ResultJsonUtil.returnResult(ResultEnum.FAIL);
    	}
    
    	@Override
    	public ResultJsonEntity updateByPrimaryKeySelective(Object obj) {
    		LOG.error("{} updateByPrimaryKeySelective 出错 进入熔断 ", this.getClass().getName());
    		return ResultJsonUtil.returnResult(ResultEnum.FAIL);
    	}
    
    	@Override
    	public ResultJsonEntity getPageExampleList(Map<String, Object> map) {
    		LOG.error("{} getPageExampleList 出错 进入熔断 ", this.getClass().getName());
    		return ResultJsonUtil.returnResult(ResultEnum.FAIL);
    	}
    }
## FeignClient使用方法 ##
为了使Feign拥有负载均衡的能力，需要在 `@ModelAttribute` 注解的方法重复调用 `getCacheFeign`，`getCacheFeign` 方法可以获取最新的地址，根据地址构建Feign或者在缓存取出Feign。

	private BaseFeignClient xxxClient = null;
	@ModelAttribute
	public xxxEntity get(@RequestParam(required = false) String id) {
		xxxClient = BaseFeignBuilder.getCacheFeign(BaseFeignClient.class,
				"xxx-service", new BaseFeignClientFallback(), "xxx");
	}

## 总结 ##
到此为止完成了将老旧WEB项目接入微服务获取服务。这里由于是手动获取Eureka中的地址，看起来还不够优雅。Feign可以结合Ribbon使用，通过传入服务名找地址，之前实现后发现需要两次访问才可以正确访问到服务，两次中必然有一次返回找不到地址，所以没有使用。
