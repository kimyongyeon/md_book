## 1. 기초

- 클라우드 네이티브 애플리케이션
- 부트캠프
- 12요소 애플리케이션 설정
- 테스트
- 애플리케이션 마이그레이션

## 2. 웹서비스

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

    ![](https://blobscdn.gitbook.com/v0/b/gitbook-28427.appspot.com/o/assets%2F-LE8_fwLnI2gUuguYTDU%2F-LFlqdcGJio_dOX8yoHA%2F-LFlqe4npvdvrtdiNTes%2Feureka-high-level-architecture.png?generation=1529844771835661&alt=media)

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
    ![](https://ssup2.github.io/images/theory_analysis/MSA/Service_Type.PNG)
    
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
        ![](https://blobscdn.gitbook.com/v0/b/gitbook-28427.appspot.com/o/assets%2F-LE8_fwLnI2gUuguYTDU%2F-LE8aLapaOcF3H3eke0t%2F-LE8aMYFy_jwf_Vy95yR%2Fzuul-netflix-cloud-architecture.png?generation=1528095670450003&alt=media)
        
        
        - Zuul 2.0 Architecture
        ![](https://blobscdn.gitbook.com/v0/b/gitbook-28427.appspot.com/o/assets%2F-LE8_fwLnI2gUuguYTDU%2F-LFeAkrDtna1r8J8fu9J%2F-LE8aMYIWejnnlftUAtL%2Fzuul-how-it-works.png?generation=1529716086552876&alt=media)
        
       
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

- 커스텀 주울 필터
    - pre 필터는 요청이 라우팅 되기 전에 실행된다.
    - routing 필터는 요청의 실제 라우팅을 처리한다. 
    - post 필터는 요청이 라우팅된 후에 실행된다.
    - error필터는 요청 처리중에 에러가 발생하면 실행된다. 

    - 스프링부트 + 스프링 클라우드 = 특별한 필터를 붙인다.
        - AuthenticationHeaderFilter: 프록시 요청을 찾아서 다운스트림으로 전달하기전에 authorization 헤더를 제거하는 필터
        - OAuth2TokenRelayFilter: 들어오는 요청이 유효하다면 OAuth접근 토큰을 전파하는 필터
        - ServletDetectionFilter: HTTP 서블릿 요청이 이미 주울 필터 파이프라인을 통과했는지 추적하는 필터
        - Servlet30WrapperFilter: 들어오는 요청을 서블릿 3.0 데코레이터로 감싸주는 필터
        - FormBodyWrapperFilter
        - DebugFilter
        - SendResponseFilter
        - SendErrorFilter
        - SendForwrdFilter
        - SimleHostRoutingFilter
        - RibbonRoutingFilter
    - 요청 제한기 필터 구현

```java
@Profile("throttled")
@Configuration
class ThrottlingConfiguration {

 @Bean //<1> 초당 몇개의 요청을 허용할지 정한다. 초당 0.1개, 즉 10초에 1개의 요청만을 허용하도록 설정했다.
 RateLimiter rateLimiter() {
  return RateLimiter.create(1.0D / 10.0D);
 }
} 

@Profile("throttled")
@Component
class ThrottlingZuulFilter extends ZuulFilter {

 private final HttpStatus tooManyRequests = HttpStatus.TOO_MANY_REQUESTS;

 private final RateLimiter rateLimiter;

 @Autowired
 public ThrottlingZuulFilter(RateLimiter rateLimiter) {
  this.rateLimiter = rateLimiter;
 }

 // <1> 이 필터는 pre필터이며 요청이 프록시 되기 전에 실행된다.
 @Override
 public String filterType() {
  return "pre";
 }

 // <2> 우선 순위가 가장 높은 필터로서 가능한 한 일찍 실행된다.
 @Override
 public int filterOrder() {
  return Ordered.HIGHEST_PRECEDENCE;
 }

 // <3> 반드시 실행되는 필수 필터지만 요청 정보나 설정에 의해 비활성화할 수도 있다.
 @Override
 public boolean shouldFilter() {
  return true;
 }

 // <4> run 메서드에는 Zuulfilter가 실제 필터링하는 로직이 담겨 있다.
 @Override
 public Object run() {
  try {
   RequestContext currentContext = RequestContext.getCurrentContext();
   HttpServletResponse response = currentContext.getResponse();

   if (!rateLimiter.tryAcquire()) {

    // <5> 주입받은 속도 조절계에 지정된 속도보다 더 빠른 속도로 요청이 들어오면 HTTP 429 - Too may Requests 에러와 에러 메시지를 반환한다. 
    response.setContentType(MediaType.TEXT_PLAIN_VALUE);
    response.setStatus(this.tooManyRequests.value());
    response.getWriter().append(this.tooManyRequests.getReasonPhrase());

    // <6> 요청의 프록시 과정을 명시적으로 중단한다. 명시적으로 중단하지 않으면 요청은 자동으로 프록시 된다. 
    currentContext.setSendZuulResponse(false);

    throw new ZuulException(this.tooManyRequests.getReasonPhrase(),
     this.tooManyRequests.value(), this.tooManyRequests.getReasonPhrase());
   }
  }
  catch (Exception e) {
   ReflectionUtils.rethrowRuntimeException(e);
  }
  return null;
 }
}
```
        
- 엣지 서비스의 보안
    - 엣지 서비스는 잠재적으로 악의적인 공격 의도를 담고 있는 요청을 하면 가장 먼저 방어할 수 있는 최전방 수비대라고 할 수 있다.

![Imgurl](https://blobscdn.gitbook.com/v0/b/gitbook-28427.appspot.com/o/assets%2F-LE8_fwLnI2gUuguYTDU%2F-LFe8LTXotQeoxRm3cy0%2F-LFe4W0zW_oX1ui_26BP%2Fapi-gateway-in-msa.png?generation=1529715453444393&alt=media)

- OAuth
    - 1.0, 1.0a, 2.0 세가지 버전 존재한다.
    - 2.0 기반만 다룸.(1점대는 너무 복잡하고 쓸때없는 기능이 너무 많음 그래서 줄인게 2.0)
    - 토큰 기반 인가의 표준
    - 토큰의 장점
        - 아이디와 패스워드가 노출되는 시간 간격을 줄인다.
        - 클라이언트가 패스워드에 얽매이지 않게 하면서도 잘못된 클라이언트가 올바른 사용자의 계정을 잠글 수 없도록 보장한다. 
        - 특정 사용자를 대신해서 클라이언트를 나타내기도 하고, 특정 사용자에 대한 컨텍스트가 없는 토큰을 나타내기도 한다. 
        - OAuth는 사용자가 누구인지 확인하는 인증 프로토콜이라기보다는 사용자 무슨 권한을 가지고 있는지를 확인하는 인가 프로토콜이다.

![Imgurl](http://www.nextree.co.kr/content/images/2017/08/8.png)

- 스프링 시큐리티

![Imgurl](https://t1.daumcdn.net/cfile/tistory/99A7223C5B6B29F003)  

- 스프링 클라우드 시큐리티

![Imgurl](https://image.slidesharecdn.com/springsecurityoauth2-180905043902/95/next-generation-spring-security-oauth20-15-638.jpg?cb=1536122909)  

## 3. 데이터 통합
Spring Data 프로젝트의 리포지토리를 사용해서 데이터 관리하는 방법을 알아본다.

### 데이터 모델링

도메이 주도 디자인을 통해 해결하고자 하는 문제는 바로 복잡성이다.

#### 관계형 데이터베이스 관리 시스템

최근 SQL 데이터베이스는 사실 다른 모든 영구 저장 기술에 견주어 볼 때 가장 강력한 트랜잭션 기능을 제공한다고 보기 힘들다. 아이러니 하게도 관계모델은 그래프를 사용하는 것이 가장 좋은 방법이다.

SQL 데이터베이스가 다수의 읽기 트랜잭션을 처리할 수 있지만, 이를 위해서는 모델의 비정규화와 고가용성을 일부 포기한 모델의 적용이 필요하다. 

현재 RDBMS를 사용하고 있다면 JPA 구현을 이용한 하이버네이트는 데이터베이스 스키마와 레코드를 매핑하는데 매우 좋은 선택이다. JPA는 개발자에게 RDMBS 자체에 대한 것들보다는 도메인 모델에 더욱 집중할 수 있도록 돕는다. 결국 조금 더 편리한 진화를 위해 제어를 포기하는 것이다.

#### NoSQL

NoSQL 데이터베이스는 특정 사용 사례의 요구를 해결하기 위해 최적화된 다양한 데이터 모델을 제공한다. 특정 데이터 모델의 경계 지어진 컨텍스트의 분리가 필요한 마이크로서비스 구현에 장점으로 작용할 수 있다.

폴리글랏 영속성은 마틴 파울러가 다양한 유형의 데이터 베이스를 사용한 아키텍처를 설명할 때 사용해서 대중화된 용어이다. 

특정 문제를 해결하기 위해 이전처럼 관계형 데이터베이스를 사용하는 대신, 특정 사용 사례에 최적화된 NoSQL을 사용할 수 있다.

### 스프링 데이터

스프링 데이터는 데이터베이스 모델의 특수성을 보존하면서도 데이터 스토어와 상호작용을 위한 포괄적인 추상화을 제공하는 오픈소스 프로젝트다.

스프링 데이터 프로젝트는 RDMS와 NoSQL 데이터베이스 모델을 포함한 12가지 이상의 잘 알려진 데이터베이스를 지원 한다. 

#### 스프링 데이터 애플리케이션의 구조

스프링 데이터를 통해 데이터 계층 접근에 필요한 디자인 패턴을 이해하는 데 많은 도움을 얻을 수 있다. 먼저 데이터 스토어에 접근하기 위한 기본적인 사항부터 알아보자.

#### 도메인 클래스

첫째로 도메인 모델의 객체를 대표하는 엔티티 클래스를 살펴보자. 도메인 클래스는 도메인 데이터의 모델을 함수로 표현한 기본 클래스다. 비공개/세터/게터 

```java
public class User {
    private Long id;
    private String firstName;
    private String lastName;
    private String email;
}
```  

> 도메인 클래스는 애플리케이션 안의 데이터 스토어의 (데이터베이스의 테이블과 같은) 데이터 객체와 엔티티의 매핑을 표현한다. 

#### 리포지토리 

스프링 데이터 애플리케이션에서 데이터 접근에는 리포지토리를 사용한다. CRUD 메소드 시그니처를 정의한 인터페이스에 대해서는 자동으로 그 구현체를 제공하기도 한다. CrudRepository - 구현체

```java
public interface UserRepository extends CrudRepository<User, Long> {}
```

UserRepository 인터페이스를 직접 구현할 필요는 없다. 스프링 데이터는 계약에 의해 구현된 빈을 제공하며, 사용자는 필요하면 언제든 다른 스프링 빈을 주입할 수 있다. 

#### 도메인 데이터를 위한 자바 패키지 구성 

도메인 클래스와 도메인 컨셉을 반영하면서도, 향후 리포지토리의 쉬운 마이그레이션을 고려한 패키지를 구성해야 한다. 최적의 준비 방안은 도메인 클래스와 리포지토리를 그룹화해서 하나의 패키지 안에 하나의 클래스로 관리할 수 있다. 

##### 지원하는 레포지토리 
Repository, CrudRepository, PagingAndSortingRepository, JpaRepository, MongoRepository, CouchbaseRepository 

### JDBC를 사용한 RDBMS 접근 시작해보기 

mysql:mysql-connector-java가 클래스패스에 추가 해야함.  
spring-boot-starter-data-jpa가 존재한다면 JPA 역시 설정한다.  
application.yml 에서 커넥션 정보 설정 한다.

### 스프링 JDBC 지원

JdbcTemplate은 스프링 프레임워크에서 데이터 접근을 처리하기 위한 오래된 방법 중 하나다.  
- execute
- batchUpdate
- query

src/main/resources/schema.sql, data.sql 을 이용하면 코드 몇줄로 테이블 생성과 데이터 생성이 가능하다. 

### 스프링 데이터 예제

- 스프링 데이터 JPA(MySQL)  
- 스프링 데이터 몽고디비  
- 스프링 데이터 네이포제이  
- 스프링 데이터 레디스  

유향관계로 연결된 개념은 도메인 클래스 간의 명시적 관계를 나타낸다. 점선으로 연결된 개념은 리포지토리 조인을 통한 연결성을 나타낸다.  

```sql
select o.* from orders o
inner join account ac on o.accountId = ac.Id
inner join address ad on ad.ccountId = ac.Id
where ad.id = 0
```  

클라우드 네이티브 옷집 온라인 상점서비스  

### 스프링 데이터 JPA

스프링 데이터 JPA 프로젝트는 RDBMS와 SQL을 위한 데이터 관리를 제공한다.  

#### 계정서비스 

계정서비스에 스프링 데이터 JPA를 사용할 것이다. 계정 서비스는 클라우드 네이티브 옷집 도메인 모델의 계정 컨텍스트를 담당한다. 스프링 데이터 JPA는 스프링 데이터 프로젝트 전체에서 무언가 특별한 프로젝트라고 할 수 있다. 스프링 데이터 JPA는 다양한(JDBC 기반) 데이터소스를 하나의 라이브러리를 통해 훌륭하게 지원하는 반면, 다른 스프링 데이터 프로젝트는 하나의 데이터 소스만 지원한다.  

### 스프링 데이터 몽고디비 

스프링 데이터 몽고디비 프로젝트는 몽고디비 데이터 관리 리포지토리를 제공한다.  

#### 주문서비스  

클라우드 네이티브 옷집 도메인 모델에서 주문 컨텍스트를 위한 백엔드 서비스로서 기능할 것이다.  
배송, 주문, 주문서, 제품  

### 스프링 데이터 네이포제이

스프링 데이터 네이포제이 프로젝트는 유명한 그래프 데이터베이스인 네이포제이에 리포지토리 기반의 데이터관리를 제공한다.  

#### 재고서비스 

우리는 여기서 재고 서비스 구성에 있어 디자인과 구현, 두 개의 부분으로 깊이 살펴볼 것이다. 클라우드 네이티브 옷집을 위한 재고 컨텍스트를 표현하기 위한 그래프 데이터 모델을 구성하는 방법에 대해 알아볼 필요가 있다. 판매하는 옷에는 계정 구분을 가질 수 있는데, 이는 일년 중 각 분기의 시작에 웹사이트에서 제공되는 상품의 집합이 계절별로 바뀌는 것이다.  
카탈로그는 계증 구조가 될 수도 있다.  
각 제품은 전 세계의 어디엔가 위치한 창고들을 기준으로 재고에 있는 제품을 연결해야 할수도 있다.  

사이퍼를 사용해서 배송을 위한 각 주소 검색을 위한 질의를 작성할 수 있다.  

```
match (shipement:Shipment)-[r:HAS_ADDRESS]->(address)
where shipment.id = 123
return address
```

배송이 어느 창고에서 출발해야 하는지를 알고자 한다면 어떻게 질의 해야 할까?

```
match (shipment:Shipment)-[r:HAS_ADDRESS {type:"SHIP_FROM"}]->(address)
where shipment.id =123
return address
```  

노드 레이블을 대상으로 하는 리포지토리를 다음과 같이 만들 수 있다.  
 
- 주소  
- 배송  
- 재고  
- 제품  
- 카탈로그  
- 창고  

### 스프링 데이터 레디스 

스프링 데이터 레디스 프로젝트는 레디스 키-값 저장소의 스프링 연동을 제공한다.  

#### 사용자서비스 

이 서비스의 사용자가 HTTP를 통해 원격으로 User 리소스를 관리할 수 있는 기능을 제공한다. 

### 메시징 

메시징은 애플리케이션에서 발생한 이벤트를 다른 프로세스와 네트워크에서 실행 중인 서비스에 전달함으로써 애플리케이션을 다른 서비스와 연결하기 위한 방법이다.  

다양한 형태의 메시징 애플리케이션을 소개

- 이벤트 알림  
- 이벤트를 통한 상태 전송  
- 이벤트 소싱  

서비스 간에 메시지를 저장하고 전달하기 위한 일종의 허브로는 아파치 카프카, 래빗 엠큐, 엑티브엠큐, 엠큐시리즈 와 같은 메시지 브로커 들을 사용한다.  
즉 메시지를 보내는 프로듀서와 메시지를 가져다가 처리하는 컨슈머는 서로 직접 연결해서 메시지를 주고받는 것이 아니라, 메시지 브로커에 접속한다. 오래된 방식의 브로커들은 일반적으로 확장이 어렵지만, 레빗엠큐나 아파치 카프카 같은 도구들은 메시지 처리 규모에 따라 확장이 가능하다.

```
$ cat input.txt | grep ERROR | wc -l > output.txt
```



#### 스프링 인티그레이션을 사용한 이벤트 주도 아키텍처

<u>여기서는 messageChannel을 이용하여 원시적으로 데이터를 이벤트에 흐름에 따라서 어떻게 구현을 할 수 있는지 원리에 대해서 설명한 내용들이다.</u>

MessageChannel 정의는 서로 다른 인티그레이션 플로우를 분리하기 위해 존재한다. 이를 통해 공통적인 기능을 재사용할 수 있을 뿐만 아니라 더 높은 수준의 시스템이 채널을 연결로만 사용할 수 있게 한다. MessageChannel이 바로 그 인터페이스인 것이다.  

```java
@Bean IntegrationFlow etlFlow(@Value("${input-directory:${HOME}/Desktop/in}") File dir) {
    return IntegrationFlows.
    // 1. 스프링 인티그레이션에 File 어뎁터를 설정한다. 어댑터가 유입되는 메시지를 처리하는 방법과 지정한 디렉토리를 몇 밀리초 간격으로 풀링할지 여부를 함께 설정한다.
        from(Files.inboundAdapter(dir).autoCreateDirectory(true),
        consumer -> consumer.poller(spec -> spec.fixedRate(1000)))
    // 2. 수신된 파일을 페이로드로 포워딩하는 메소드
        .handle(File.class, (file, headers) -> {
            log.info("We noticed a new file, " + file);
            return file;
        })
    // 3. MessageChannel 인스턴스를 사용해서, 유입된 파일의 확장자에 따라 하나 또는 2개의 인티그레이션 플로우로 라우트 한다.
        .routeToRecipients(
            spec -> spec.recipient(csv(), msg -> hasExt(msg.getPayload(), ".csv"))
                .recipient(txt(), msg -> hasExt(msg.getPayload(), ".txt"))).get();
        )
}
```

채널은 논리적으로 분리를 제공한다. 

#### 메시지 브로커, 브릿지, 경쟁적 컨슈머 패턴, 이벤트 소싱

실제 서비스를 만들때 쓸만한 기술들을 소개했고, 앞으로 어떤 방향으로 갈지에 대해서도 알려주는 내용들이다. 

#### 스프링 클라우드 스트림

스프링 클라우드 스트림은 스프링 인티그레이션을 기반으로 서비스 간 연동 모델에 채널을 사용하는 것을 핵심으로 한다.  

이 단원에서는 rabbitmq를 이용해서 구독/발행, p2p 방식에 대해서 구현예제를 수록함.

#### 정리

메시징은 복잡한 분산 시스템 체계에서 서비스 간 통신 방법을 제공하는 채널이다. 이 책의 나머지 장에서는 배치 프로세스와 워크플로우 엔진, 그리고 서비스간의 메시징 교차 교환에 대해 더 알아볼 것이다.  

### 배치 처리와 태스크

### 데이터 통합

## 4. 운영환경

### 관측 가능한 시스템 

### 서비스 브로커

### 지속적 전달

## 5. 부록

### 자바 EE와 스프링 부트  

### 한국어판 특별 부록 클라우드 파운드리 환경의 준비와 활용  
