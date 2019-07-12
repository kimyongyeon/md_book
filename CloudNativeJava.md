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
        ```
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
            ```
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
            ```
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
                    this.log.info(String.format("registered instances of '%s', serviceId));
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
            ```
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
        ```
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
    - Greeting 서비스
    - 간단한 엣지 서비스
    - 넷플릭스 페인
    - 넷플릭스 주울을 통한 필터링과 프록시
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
