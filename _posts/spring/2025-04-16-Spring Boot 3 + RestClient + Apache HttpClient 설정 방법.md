---
title: Spring Boot 3 + RestClient + Apache HttpClient 설정 방법
author: hkh7670
date: 2025-04-16 00:00:00 +0900
categories: [Spring]
tags: [Spring, Spring Boot]
---

### build.gradle에 의존성 추가
- 최신 http client 5 버전을 추가한다.
```groovy
// Apache HttpClient 5
implementation 'org.apache.httpcomponents.client5:httpclient5:5.4.3'
```

### RestClientConfig 설정 코드
- HttpClient를 Bean으로 등록하기 위한 Config 클래스

```java
@Slf4j
@Configuration
public class TMDBRestClientConfig {

    private final String API_READ_ACCESS_TOKEN;
    private final String API_KEY;

    public TMDBRestClientConfig(
        @Value("${tmdb.api-read-access-token}")
        String API_READ_ACCESS_TOKEN,
        @Value("${tmdb.api-key}")
        String API_KEY
    ) {
        this.API_READ_ACCESS_TOKEN = API_READ_ACCESS_TOKEN;
        this.API_KEY = API_KEY;
    }

    private static final ObjectMapper objectMapper = new ObjectMapper();
    private static final String BASE_URL = "https://api.themoviedb.org/3";
    private static final int CONNECTION_TIMEOUT = 5;
    private static final int READ_TIMEOUT = 25;
    private static final int MAX_TOTAL_CONNECTIONS = 100;
    private static final int MAX_PER_ROUTE = 20;

    @Bean
    public TMDBRestClient tmdbRestClient() {
        return HttpServiceProxyFactory
            .builderFor(RestClientAdapter.create(getTMDBRestClient()))
            .build()
            .createClient(TMDBRestClient.class);
    }

    private RestClient getTMDBRestClient() {
        return RestClient.builder()
            .baseUrl(BASE_URL)
            .defaultHeaders(httpHeaders -> {
                httpHeaders.add(HttpHeaders.AUTHORIZATION, getAuthorizationHeaderValue());
                httpHeaders.setContentType(MediaType.APPLICATION_JSON);
            })
            .requestFactory(getClientHttpRequestFactory())
            .defaultStatusHandler(HttpStatusCode::is4xxClientError, default4xxErrorHandler())
            .defaultStatusHandler(HttpStatusCode::is5xxServerError, default5xxErrorHandler())
            .build();
    }

    private String getAuthorizationHeaderValue() {
        return "Bearer " + this.API_READ_ACCESS_TOKEN;
    }

    private ClientHttpRequestFactory getClientHttpRequestFactory() {
        return new HttpComponentsClientHttpRequestFactory(createApacheHttpClient());
    }

    private HttpClient createApacheHttpClient() {
        RequestConfig requestConfig = RequestConfig.custom()
            .setConnectionRequestTimeout(CONNECTION_TIMEOUT, TimeUnit.SECONDS)
            .setResponseTimeout(READ_TIMEOUT, TimeUnit.SECONDS)
            .build();

        PoolingHttpClientConnectionManager connectionManager = new PoolingHttpClientConnectionManager();
        connectionManager.setMaxTotal(MAX_TOTAL_CONNECTIONS); // 전체 커넥션 수 최대값
        connectionManager.setDefaultMaxPerRoute(MAX_PER_ROUTE); // 도메인(호스트) 당 최대 커넥션 수

        return HttpClients.custom()
            .setConnectionManager(connectionManager)
            .setDefaultRequestConfig(requestConfig)
            .evictIdleConnections(Timeout.ofSeconds(30)) // 일정 시간 이상 유휴 커넥션 정리 (메모리 누수 방지)
            .build();
    }

    private ResponseSpec.ErrorHandler default4xxErrorHandler() {
        return (req, res) -> {
            Map<String, Object> errorBody = getErrorBody(res);
            HttpStatusCode statusCode = res.getStatusCode();
            printErrorLog(statusCode, errorBody);
            switch (statusCode) {
                case HttpStatus.UNAUTHORIZED ->
                    throw new ApiErrorException(ErrorCode.AUTHENTICATION_FAIL);
                case HttpStatus.NOT_FOUND -> throw new ApiErrorException(ErrorCode.NOT_FOUND);
                default -> throw new ApiErrorException(ErrorCode.BAD_REQUEST);
            }
        };
    }

    private ResponseSpec.ErrorHandler default5xxErrorHandler() {
        return (req, res) -> {
            Map<String, Object> errorBody = getErrorBody(res);
            HttpStatusCode statusCode = res.getStatusCode();
            printErrorLog(statusCode, errorBody);
            throw new ApiErrorException(ErrorCode.EXTERNAL_SERVER_ERROR);
        };
    }

    private Map<String, Object> getErrorBody(ClientHttpResponse res) {
        Map<String, Object> errorBody;
        try {
            errorBody = objectMapper.readValue(
                res.getBody(),
                new TypeReference<>() {
                }
            );
        } catch (Exception e) {
            log.error("Failed to parse TMDBRestClient error response body", e);
            throw new ApiErrorException(ErrorCode.EXTERNAL_SERVER_ERROR);
        }
        return errorBody;
    }

    private void printErrorLog(
        HttpStatusCode statusCode,
        Map<String, Object> errorBody
    ) {
        log.error("TMDBRestClient Error Status Code: {}", statusCode.value());
        log.error("TMDBRestClient Error Body: {}", errorBody);
    }

}

```


### 사용 예제
- 설정한 값에 맞게 Bean 생성 후 서비스 레이어에서 RestClient 인터페이스를 주입하여 사용
- Interface

```java
@Component
@HttpExchange
public interface TMDBRestClient {

    @GetExchange(value = "/movie/popular")
    TMDBPopularMoviePageResponse getPopularMovies(
        @RequestParam(name = "language", defaultValue = "ko-KR") String language,
        @RequestParam(name = "page", defaultValue = "1") Integer page
    );

}
```

- Service 코드

```java
@Slf4j
@Service
@RequiredArgsConstructor
public class TMDBService {

    private final TMDBRestClient tmdbRestClient;

    public PopularMoviePageResponse getPopularMovies(int page) {
        return PopularMoviePageResponse.from(
            tmdbRestClient.getPopularMovies(null, page)
        );
    }
}
```