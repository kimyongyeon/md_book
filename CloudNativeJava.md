## 1. 기초
---
- 클라우드 네이티브 애플리케이션
- 부트캠프
- 12요소 애플리케이션 설정
- 테스트
- 애플리케이션 마이그레이션

## 2. 웹서비스
---
- REST API
- 라우팅: 클라우드 네이티브 시스템은 동적이다. 
    - 배경: 
        - DNS 방법을 제시하나 단점이 너무 많음.
        - 로드밸런싱의 라우팅 문제: 상태유지, 라운드로빈 방식, JSESSIONID 스티키세션, 하지만 그 이상의 기능이 필요하다면 로드밸런서으로 감당하기는 어렵다. 
        - 여기서 질문하나!
        - JSESSION가 아니라 OAuth 접근 토큰이나 JSON 웹 토큰으로 스티키 세션을 구현해야 한다면 어떻게 해야할까?
        - DNS 로드밸런싱 풀에 있는 모든 노드는 라우팅을 통해 요청 받아서 처리할 수 있어야 함.
        - JVM 플랫폼 IP 주소를 캐싱, DNS 로드밸런싱 기능 무력화 
        - 가상 로드밸런서로 앞에서 말한 여러 제약조건을 극복할 수는 있지만,
            - 모든 중앙화된 로드밸런서는 시스템의 상태나 요청 처리에 필요한 작업량을 알지 못함.
        - ***정리: 라우팅에 DNS가 가장 좋은 해법이라고 할 수는 없으며, 인프라가 아니라 프로그래밍을 통해 더 나은 라우팅 전략을 구현할 수 있도 있다.***
    - DiscoveryClient 추상화
        - 서비스레지스트리
            - 침투적 방식
            - 코드 변경 필요
        - 상용제품 구현체: 클라우드 파운드리, 아파치 주키퍼, 해시코프컨설, 넷플릭스유레카
        - 추상화: 
```java
public interface Discovery {
    String description();
    ServiceInstance getLocalServiceInstnace();
    List<ServiceInstance> getInstance(String var1);
    List<String> getServices();
}
```
- 유레카 서비스 레지스트리 의존관계
    - org.springframework.cloud:spring-cloud-starter-eureka-server
    - ***@EnableEurekaServer***
```java
// EurekaServiceApplication.java
@EnableEurekaServer
@SpringBootApplication
public class EurekaServiceApplication {
    public static void main(String args[]){
        SpringApplication.run(EurekaServiceApplication.class, args);
    }
}
// application.properties
server.port=${PORT:8761}
eureka.client.register-with-eureka=false
eureka.client.fetch-registry=false
eureka.server.enable-self-preservation=false
```

- **클러스터링 구현은 여전히 어렵다. else 지금은 잘된다.**
- 서비스들이 모두 유레카를 통해 서로를 발견하고 통신하므로, 유레카는 REST 서비스의 중추 신경계와도 같다.
    <img src="https://blobscdn.gitbook.com/v0/b/gitbook-28427.appspot.com/o/assets%2F-LE8_fwLnI2gUuguYTDU%2F-LFlqdcGJio_dOX8yoHA%2F-LFlqe4npvdvrtdiNTes%2Feureka-high-level-architecture.png?generation=1529844771835661&alt=media">

- 참고: https://coe.gitbook.io/guide/service-discovery/eureka_2
- 클라이언트 서버 의존관계 
- org.springframework.boot:spring-boot-starter-web : 스프링 웹 어플리케이션을 만들기 위함 
- spring-cloud-starter-eureka

```java
// GreetingsServiceApplication.java
@EnableDiscoveryClient
@SpringBootApplication
public class GreetingsServiceApplication {
    public static void main(String args[]){
        SpringApplication.run(GreetingsServiceApplication.class, args);
    }
}
@RestController
class GreetingsRestController {
    @RequestMapping(method=RequestMethod.GET, value="/hi/{name}")
    Map<String, String> hi(@PathVariable String name, @RequestHeader(value = "X-CNJ-Name", required=false) Optional<String> cn) {
        String resolvedName=cn.orElse(name);
        return Collections.singletonMap("greeting", "Hello, " + resolvedName + "!");
    }
}
// bootstrap.properties
spring.application.name=greetings-service
server.port=${PORT:0}
eureka.intance.instance-id=${spring.application.name}:${spring.application.instance_id:${random.valuie}}
// DiscoveryClientCLR.java
@Compoent
public class DiscoveryClientCLR implements CommandLineRunner {
    private final DiscoveryClient discoveryClient;
    private Log log = LogFactory.getLog(getClass());
    @Autowired
    public DiscoveryClientCLR(DiscoveryClient discoveryClient) {
        this.discoveryClient=discoveryClient;
    }
    @Overrid
    public void run(String... args) throws Exception {
        this.log.info("localServiceInstance");
        this.logServiceInstance(this.discoveryClient.getLocalServiceInstanec());
        String.serviceId="greeting-service";
        this.log.info(String.format("registered instances of '%s'", serviceId));
        this.discoveryClient.getInstances(serivceId).forEach(this::logServiceInstance);
    }
    private void logServiceInstance(ServiceInstance si) {
        String msg = String.format("host=%s, port=%s, serviceId=%s", si.getHost(), si.getPort(), si.getServiceId());
        log.info(msg);
    }
}

```

- 인스턴스를 랜덤하게 하나 선택 가능
- 응답 가중치 전략 선택 가능
- 넷플릭스 리본 
    - 클라이언트 로드밸런싱 라이브러리 
    - 라운드로빈, 응답가중치, 확장성 다양한 로드밸런싱 전략을 지원함.

```java
// RibbonCLR.java
@Component
public class RibbonCLR implements CommandLineRunner {
    private final DiscoveryClient discoveryClient;
    private final Log log = LogFactory.getLog(getClass());

    public RibbonCLR(DiscoveryClient discoveryClient) {
    this.discoveryClient = discoveryClient;
    }

    @Override
    public void run(ApplicationArguments args) throws Exception {

    String serviceId = "greetings-service";

    // <1>
    List<Server> servers = this.discoveryClient.getInstances(serviceId).stream()
        .map(si -> new Server(si.getHost(), si.getPort()))
        .collect(Collectors.toList());

    // <2>
    IRule roundRobinRule = new RoundRobinRule();

    BaseLoadBalancer loadBalancer = LoadBalancerBuilder.newBuilder()
        .withRule(roundRobinRule).buildFixedServerListLoadBalancer(servers);

    IntStream.range(0, 10).forEach(i -> {
    // <3>
    Server server = loadBalancer.chooseServer();
    URI uri = URI.create("http://" + server.getHost() + ":" + server.getPort()
        + "/");
    log.info("resolved service " + uri.toString());
    });
}

// LoadBalancedWebClientConfiguration .java
@Configuration
class LoadBalancedWebClientConfiguration {
    @Bean
    @LoadBalanced
    WebClient webClient() {
    return WebClient.builder().build();
    }
}

// LoadBalancedWebClientRunner.java
@Component
class LoadBalancedWebClientRunner implements ApplicationRunner {
    private final Log log = LogFactory.getLog(getClass());
    private final WebClient client;
    LoadBalancedWebClientRunner(@LoadBalanced WebClient client) {
        this.client = client;
    }

    // <1>
    @Override
    public void run(ApplicationArguments args) throws Exception {
        Map<String, String> variables = Collections.singletonMap("name",
        "Cloud Natives!");
        // <2>
        this.client.get().uri("http://greetings-service/hi/{name}", variables)
            .retrieve().bodyToMono(JsonNode.class).map(x -> x.get("greeting").asText()).subscribe(greeting -> log.info("greeting: " + greeting));
    }
}
```

- 클라우드 파운드리 라우트 서비스
    - 클라이언트 로드밸런싱 
    - 라우팅 동작을 규정하는 중앙의 컴포넌트 역할
    - 라우트 서비스 
        - 궁극적으로는 주어진 호스트와 포트에서 모든 HTTP요청을 받아서 처리하고, 변환하는 포워딩을 담당하는 HTTP 서비스다.
    - 빌드팩, 애플리케이션, 서비스 브로커처럼 개발해서 플랫폼에 플러그인으로 심을 수 있는 클라우드 파운드리의 확장판
    - 클라우드 파운드리는 현재로서의 하나의 요청에서 다른 요청으로 라우트 서비스 체이닝은 지원하지 않는다. 
```java
// RouteServiceApplication.java
@SpringBootApplication
public class RouteServiceApplication {

        public static void main(String[] args) {
                SpringApplication.run(RouteServiceApplication.class, args);
        }

        // <1>
        @Bean
        RestTemplate restOperations() {
                RestTemplate restTemplate = new RestTemplate(new TrustEverythingClientHttpRequestFactory()); // <2>
                restTemplate.setErrorHandler(new NoErrorsResponseErrorHandler()); // <3>
                return restTemplate;
        }

        private static class NoErrorsResponseErrorHandler extends DefaultResponseErrorHandler {
                @Override
                public boolean hasError(ClientHttpResponse response) throws IOException {
                        return false;
                }
        }

        private static final class TrustEverythingClientHttpRequestFactory extends SimpleClientHttpRequestFactory {
                private static SSLContext getSslContext(TrustManager trustManager) {
                        try {
                                SSLContext sslContext = SSLContext.getInstance("SSL");
                                sslContext.init(null, new TrustManager[]{trustManager}, null);
                                return sslContext;
                        }
                        catch (KeyManagementException | NoSuchAlgorithmException e) {
                                throw new RuntimeException(e);
                        }
                }
                @Override
                protected HttpURLConnection openConnection(URL url, Proxy proxy) throws IOException {

                        HttpURLConnection connection = super.openConnection(url, proxy);

                        if (connection instanceof HttpsURLConnection) {
                                HttpsURLConnection httpsConnection = (HttpsURLConnection) connection;
                                SSLContext sslContext = getSslContext(new TrustEverythingTrustManager());
                                httpsConnection.setSSLSocketFactory(sslContext.getSocketFactory());
                                httpsConnection.setHostnameVerifier((s, session) -> true);
                        }
                        return connection;
                }

        }
        private static final class TrustEverythingTrustManager implements
            X509TrustManager {
                @Override
                public void checkClientTrusted(X509Certificate[] x509Certificates, String s)
                    throws CertificateException {
                }
                @Override
                public void checkServerTrusted(X509Certificate[] x509Certificates, String s)
                    throws CertificateException {
                }
                @Override
                public X509Certificate[] getAcceptedIssuers() {
                        return new X509Certificate[0];
                }
        }
}        

// RoutingRestController.java
@RestController
class RoutingRestController {

        // <1>
        private static final String FORWARDED_URL = "X-CF-Forwarded-Url";

        private static final String PROXY_METADATA = "X-CF-Proxy-Metadata";

        private static final String PROXY_SIGNATURE = "X-CF-Proxy-Signature";

        private final Log logger = LogFactory.getLog(this.getClass());

        private final RestOperations restOperations;

        RoutingRestController(RestTemplate rt) {
                this.restOperations = rt;
        }

        // <2>
        @RequestMapping(headers = {FORWARDED_URL, PROXY_METADATA, PROXY_SIGNATURE})
        ResponseEntity<?> service(RequestEntity<byte[]> incoming) {

                this.logger.info("incoming request: " + incoming);

                HttpHeaders headers = new HttpHeaders();
                headers.putAll(incoming.getHeaders());
                headers.put("X-CNJ-Name", Collections.singletonList("Cloud Natives"));

                // <3>
                URI uri = headers
                    .remove(FORWARDED_URL)
                    .stream()
                    .findFirst()
                    .map(URI::create)
                    .orElseThrow(() -> new IllegalStateException(String.format("No %s header present", FORWARDED_URL)));

                // <4>
                RequestEntity<?> outgoing = new RequestEntity<>(
                    ((RequestEntity<?>) incoming).getBody(), headers, incoming.getMethod(), uri);

                this.logger.info("outgoing request: {}" + outgoing);

                return this.restOperations.exchange(outgoing, byte[].class);
        }
}        
```

- 그외 장점들
    - 요청에 대한 전처리
    - 인증 처리
    - 어댑터를 이용해서 인증 타입을 전환할 수도 있다.
    - 일정량 이상의 요청을 제한하는 로직을 추가할 수도 있다.
    - 상용제품: 애피지

- 엣지 서비스
    <img src="https://ssup2.github.io/images/theory_analysis/MSA/Service_Type.PNG">
    - Core/Atomic Service : Core Business Logic이나 Atomic한 Business Logic을 수행하는 Service이다.
    - Composite/Integration Service : Core/Atomic Service를 조합하여 구성한 Service이다.
    - API/Edge Service : Core/Atomic Service, Composite/Integration Service를 조합하여 App에게 노출되는 Service이다. API Gateway의 역활도 수행한다.
    <br>
    
    - 역할
        - 클라이언트와 백앤드 서버와의 출입문
        - 인증/인가 같은 보안
        - 일정량 이상의 요청 제한
        - 라우팅(라우팅,필터링,API변환,클라이언트어댑터API,서비스프록시)
        - 횡단관심사

    <br>
    - Greeting 서비스
        엣지 서비스가 풀어야할 문제점 제시.
        
```java
// EurekaServiceApplication.java
// <1>
@EnableEurekaServer
@SpringBootApplication
public class EurekaServiceApplication {
    public static void main(String args[]) {
        SpringApplication.run(EurekaServiceApplication.class, args);
    }
}
```

```
# application.properties
server.port=${PORT:8761}
# bootstrap.properties
spring.application.name=eureka-service
# <1>
eureka.client.register-with-eureka=false
eureka.client.fetch-registry=false
# <2>
eureka.server.enable-self-preservation=false
```

- 1: 자기 자신 등록 안함.
- 2: 서비스 레지스트리에 등록된 노드 중 정해진 시간 안에 생존신호를 보내지 않는 노드의 비율이 높아지면, 유레카는 일단 이를 애플리케이션 문제가 아니라 네트워크 문제라고 가정하고 생존 신호를 보내지 않는 노드를 레지스트리에서 제거하지 않는데, 이를 자기보호 모드라고 한다. 
    
```java
// GreetingsServiceApplication.java
// <1>
@EnableDiscoveryClient
@SpringBootApplication
public class GreetingsServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(GreetingsServiceApplication.class, args);
    }
}      
// DefaultGreetingsRestController.java
@Profile({ "default", "insecure" })
@RestController
@RequestMapping(method = RequestMethod.GET, value = "/greet/{name}")
class DefaultGreetingsRestController {
    @RequestMapping
    Map<String, String> hi(@PathVariable String name) {
        return Collections.singletonMap("greeting", "Hello, " + name + "!");
    }
}          
```

- 간단한 엣지 서비스
```java
@Profile({ "default", "insecure" })
@RestController
@RequestMapping("/api")
class RestTemplateGreetingsClientApiGateway {

    private final RestTemplate restTemplate;

    @Autowired
    RestTemplateGreetingsClientApiGateway(
        @LoadBalanced RestTemplate restTemplate) { // <1>
        this.restTemplate = restTemplate;
    }

    @GetMapping("/resttemplate/{name}")
    Map<String, String> restTemplate(@PathVariable String name) {

        //@formatter:off
        ParameterizedTypeReference<Map<String, String>> type =
            new ParameterizedTypeReference<Map<String, String>>() {};
        //@formatter:on

        ResponseEntity<Map<String, String>> responseEntity = this.restTemplate
        .exchange("http://greetings-service/greet/{name}", HttpMethod.GET, null,
            type, name);
        return responseEntity.getBody();
    }
}
```
- 간단하게 받아서 외부서비스를 호출하는 gateway 구현

- 넷플릭스 페인
    - RPC 사용하지 말것
    - REST 장점을 살려라
    - 넷플릭스 페인을 사용하면 인터페이스와 몇 가지 규약을 정의하는 것만으로 서비스 클라이언트를 쉽게 생성할 수 있다. 

```java
// <1>
@FeignClient(serviceId = "greetings-service")
interface GreetingsClient {
    // <2>
    @RequestMapping(method = RequestMethod.GET, value = "/greet/{name}")
    Map<String, String> greet(@PathVariable("name") String name); // <3>
}
- 1: DiscoveryClient 추상화와 넷플릭스 리본이 클라이언트 로드밸런싱을 수행할 수 있도록 서비스 ID를 지정한다.
- 2: 스프링 MVC의 @RequestMapping으로 클라이언트가 어떤 종단점을 호출하는지 지정한다. @RequestMapping은 주로 서버 쪽 구현에 사용해왔겠지만 여기에서는 분명히 클라이언트에 사용되고 있음을 눈여겨 보자.
- 3: 반환값을 지정한다. 어려운 슈퍼 타입 토큰을 패턴을 사용하지 않고도 결과를 Map<String, String>으로 쉽게 매핑할 수 있다.
// <1>
@Profile("feign")
@RestController
@RequestMapping("/api")
class FeignGreetingsClientApiGateway {

    private final GreetingsClient greetingsClient;

    @Autowired
    FeignGreetingsClientApiGateway(GreetingsClient greetingsClient) {
        this.greetingsClient = greetingsClient;
    }

    // <2>
    @GetMapping("/feign/{name}")
    Map<String, String> feign(@PathVariable String name) {
        return this.greetingsClient.greet(name);
    }
}
- 1: 엣지 서비스는 feign 프로파일이 적용될 때만 실행된다.
- 2: 엣지 서비스에 대한 호출은 여전히 REST에 의존하는 메소드 호출이다. 클라이언트는 단순하고 쓰기 쉽다. 페인을 사용할 줄 아는 개발자라면 여기에서 무슨 일이 일어나는지 금방 알아챌 것이다. 
```
- 정리: REST호출로 다른 서비스로부터 데이터를 가져오느느 방법을 알아봄.
    
    - 넷플릭스 주울을 통한 필터링과 프록시
        
        - Zuul in Netflix's cloud Architecture
            <img src="https://blobscdn.gitbook.com/v0/b/gitbook-28427.appspot.com/o/assets%2F-LE8_fwLnI2gUuguYTDU%2F-LE8aLapaOcF3H3eke0t%2F-LE8aMYFy_jwf_Vy95yR%2Fzuul-netflix-cloud-architecture.png?generation=1528095670450003&alt=media">
        
        - Zuul 2.0 Architecture
            <img src="https://blobscdn.gitbook.com/v0/b/gitbook-28427.appspot.com/o/assets%2F-LE8_fwLnI2gUuguYTDU%2F-LFeAkrDtna1r8J8fu9J%2F-LE8aMYIWejnnlftUAtL%2Fzuul-how-it-works.png?generation=1529716086552876&alt=media">
       
    - 주울필터 역할
        - 동적 라우팅 로직 포함
        - 로드밸런싱
        - 서비스 보호 기능 
        - 문제 경로 파악과 디버깅을 지원하는 도구
        - 지오로케이션, 쿠키전파, 토큰 복호화 등을 담당하며서 시스템에 들어오는 요청을 처리할 수 있는 가장 높은 수준의 컨텍스트를 만드는데 사용
        - 횡단관심사
        - 특정 디바이스에서 발생하는 데이터 인코딩 관련 문제
        - 불필요한 헤더 제거
        - URL 인코딩등을 처리하면서 요청/응답의 정규화에도 사용된다.
        - 특정 요청을 격리하는 타켓 라우팅
        - 트래픽 변형
        - 글로벌 라우팅
        - 비정상 노드 및 영역별 무정지 장애 대응 처리
        - 공격탐지 및 방지 기능
            
    - 주울게이트웨이
        - 시스템을 통과하는 데이터의 흐름을 한곳에서 관찰할 수 있는 장소라는 관점에서 일종의 품질 보증 도구 역할도 담당한다고 볼 수 있다.
            
```java
@Configuration
@EnableZuulProxy
class ZuulConfiguration {

    @Bean
    CommandLineRunner commandLineRunner(RouteLocator routeLocator) { // <1>
        Log log = LogFactory.getLog(getClass());
        return args -> routeLocator.getRoutes().forEach(
        r -> log.info(String.format("%s (%s) %s", r.getId(), r.getLocation(),
            r.getFullPath())));
    }

}
- 1: RouteLocator는 인터페이스인데 구현 클래스에는 몇 가지 재미있는 내용이 담겨 있다. 스프링 클라우드는 설정에 따라 @DisvoeryClient를 알고 있는 RouteLocator 구현 클래스를 주입해준다. 

// # bootstrap.properties
// # <1>
// spring.application.name=greetings-client
// # <2>
// server.port=${PORT:8082}
// security.oauth2.resource.userInfoUri=http://auth-service/uaa/user
// #security.oauth2.resource.loadBalanced=false

// hystrix.command.default.execution.isolation.strategy=SEMAPHORE
// spring.mvc.dispatch-options-request=true

// zuul.routes.hi.path=/lets/**
// zuul.routes.hi.serviceId=greetings-service

@Component
class RoutesListener {

 private final RouteLocator routeLocator;

 private final DiscoveryClient discoveryClient;

 private Log log = LogFactory.getLog(getClass());

 @Autowired
 public RoutesListener(DiscoveryClient dc, RouteLocator rl) {
  this.routeLocator = rl;
  this.discoveryClient = dc;
 }

 // <1> 메커니즘에 의해 발행되는 이벤트 리스닝
 @EventListener(HeartbeatEvent.class)
 public void onHeartbeatEvent(HeartbeatEvent event) {
  this.log.info("onHeartbeatEvent()");
  this.discoveryClient.getServices().stream().map(x -> " " + x)
   .forEach(this.log::info);
 }

 // <2> 주울 메커니즘에 의해 발행되는 이벤트 리스닝
 @EventListener(RoutesRefreshedEvent.class)
 public void onRoutesRefreshedEvent(RoutesRefreshedEvent event) {
  this.log.info("onRoutesRefreshedEvent()");
  this.routeLocator.getRoutes().stream().map(x -> " " + x)
   .forEach(this.log::info);
 }

}

@Profile("cors")
@Component
@Order(Ordered.HIGHEST_PRECEDENCE + 10)
class CorsFilter implements Filter {

 private final Log log = LogFactory.getLog(getClass());

 private final Map<String, List<ServiceInstance>> catalog = new ConcurrentHashMap<>();

 private final DiscoveryClient discoveryClient;

 // <1> 서비스 토폴리지 정보를 얻기 위해 DiscoveryClient를 사용한다. 
 @Autowired
 public CorsFilter(DiscoveryClient discoveryClient) {
  this.discoveryClient = discoveryClient;
  this.refreshCatalog();
 }

 // <2> 서블릿 필터의 doFilter를 오버라이드해서 필요한 필터링 로직을 넣는다. 
 // isClientAllowed() 결과가 true이면 ACCESS_CONTROL_ALLOW_ORIGIN을 응답 헤더에 추가한다. 
 @Override
 public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
  throws IOException, ServletException {
  HttpServletResponse response = HttpServletResponse.class.cast(res);
  HttpServletRequest request = HttpServletRequest.class.cast(req);
  String originHeaderValue = originFor(request);
  boolean clientAllowed = isClientAllowed(originHeaderValue);

  if (clientAllowed) {
   response.setHeader(HttpHeaders.ACCESS_CONTROL_ALLOW_ORIGIN,
    originHeaderValue);
  }

  chain.doFilter(req, res);
 }

 // <3> 요청 클라이언트의 오리진이 다운스트림 서비스별 화이트리스트(코드에는 catalog로 되어 있음)에 포함되어 있으면 true를 반환한다. 
 private boolean isClientAllowed(String origin) {
  if (StringUtils.hasText(origin)) {
   URI originUri = URI.create(origin);
   int port = originUri.getPort();
   String match = originUri.getHost() + ':' + (port <= 0 ? 80 : port);

   this.catalog.forEach((k, v) -> {
    String collect = v
     .stream()
     .map(
      si -> si.getHost() + ':' + si.getPort() + '(' + si.getServiceId() + ')')
     .collect(Collectors.joining());
  });

   boolean svcMatch = this.catalog
    .keySet()
    .stream()
    .anyMatch(
     serviceId -> this.catalog.get(serviceId).stream()
      .map(si -> si.getHost() + ':' + si.getPort())
      .anyMatch(hp -> hp.equalsIgnoreCase(match)));
   return svcMatch;
  }
  return false;
 }

 // <4> DiscoveryClient가 생존신호 이벤트를 받을 때마다 화이트리스트가 포함된 다운스트림서비스를 저장하고 있는 로컬 캐시를 페기하고, 새 정보로 갱신한다. 
 @EventListener(HeartbeatEvent.class)
 public void onHeartbeatEvent(HeartbeatEvent e) {
  this.refreshCatalog();
 }

 private void refreshCatalog() {
  discoveryClient.getServices().forEach(
   svc -> this.catalog.put(svc, this.discoveryClient.getInstances(svc)));
 }

 @Override
 public void init(FilterConfig filterConfig) throws ServletException {
 }

 @Override
 public void destroy() {
 }

 private String originFor(HttpServletRequest request) {
  return StringUtils.hasText(request.getHeader(HttpHeaders.ORIGIN)) ? request
   .getHeader(HttpHeaders.ORIGIN) : request.getHeader(HttpHeaders.REFERER);
 }
}

@RestController
@EnableDiscoveryClient
@SpringBootApplication
public class Html5Client {

 private final LoadBalancerClient loadBalancerClient;

 @Autowired
 Html5Client(LoadBalancerClient loadBalancerClient) {
  this.loadBalancerClient = loadBalancerClient;
 }

 public static void main(String[] args) {
  SpringApplication.run(Html5Client.class, args);
 }

 // <1> 종단점은 다운스트림의 서비스의 greetings-client 인스턴스의 종단점을 반환한다.
 // 이 종단점은 엣지 서비스의 CorsFilter에서 Access-Control-* 헤더를 붙여준 덕분에 다른
 // 오리진에서도 접근할 수 있다. 그리고 LoadBalancerClient를 이용하면 로드밸런서 설정에 따라 인스턴스
 // 정보를 얻어올 수 있다. 클라이언트 로드밸런서는 기본값으로 넷플릭스 리본을 사용하는데, 리본은
 // 기본적으로 라운드로빈 로드밸런싱을 사용한다. 
 //@formatter:off
 @GetMapping(value = "/greetings-client-uri",
         produces = MediaType.APPLICATION_JSON_VALUE)
 //@formatter:on
 Map<String, String> greetingsClientURI() throws Exception {
  return Optional
   .ofNullable(this.loadBalancerClient.choose("greetings-client"))
   .map(si -> Collections.singletonMap("uri", si.getUri().toString()))
   .orElse(null);
 }
}
```

- 제이쿼리를 사용하는 자바스크립트 코드 
```html
<!doctype html>
<html lang="en">
<head>
    <meta charset="utf-8"/>
    <meta http-equiv="X-UA-Compatible" content="IE=edge"/>
    <title>Demo</title>
    <meta name="description" content=""/>
    <meta name="viewport" content="width=device-width"/>
    <base href="/"/>
</head>
<body>

<div id="message"></div>

<script src="/webjars/jquery/jquery.min.js"></script>
<script>
    // <1> 현재 어플리케이션의 /greeting-client-uri를 호출해서 greetings-client의 URI를 얻어온다.
    var greetingsClientUrl = location.protocol + "//" + window.location.host
        + "/greetings-client-uri";
    $.ajax({url: greetingsClientUrl}).done(function (data) {
        var nameToGreet = window.prompt("who would you like to greet?");
        var greetingsServiceUrl = data['uri'] + "/lets/greet/" + nameToGreet;
        console.log('greetingsServiceUrl: ' + greetingsServiceUrl);
        // <2> 받아온 URI에 있는 서버에 크로스 오리진 요청을 보낸다. 
        $.ajax({url: greetingsServiceUrl}).done(function (greeting) {
            $("#message").html(greeting['greeting']);
        });
    });
</script>
</body>
</html>
```
        
- 엣지 서비스의 보안

- OAuth
- 스프링 시큐리티
- 스프링 클라우드 시큐리티

## 3. 데이터 통합
---
- 데이터 관리
- 메시징
- 배치 처리와 태스크
- 데이터 통합

## 4. 운영환경
---
- 관측 가능한 시스템
- 서비스 브로커
- 지속적 전달
