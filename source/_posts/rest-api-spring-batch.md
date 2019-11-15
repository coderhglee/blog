---
title: rest-api-spring-batch
date: 2019-11-14 15:56:57
description : Spring Batch
category: [java, spring]
tag : [java, spring]

---

Spring Batch에서 제공하는 메타테이블을 사용하지않고 Batch 프로그램을 구동시키기 위한 InMemoryBatchConfigurer Class을 정의한다.

## Batch Meta-Data Schema

![Meta-Data Schema](https://user-images.githubusercontent.com/24283191/69026694-482e4300-0a0f-11ea-8f3f-7b7a793d1623.png)

- 이전에 실행한 Job이 어떤 것들이 있는지
- 최근 실패한 Batch Parameter가 어떤것들이 있고, 성공한 Job은 어떤것들이 있는지
- 다시 실행한다면 어디서 부터 시작하면 될지
- 어떤 Job에 어떤 Step들이 있었고, Step들 중 성공한 Step과 실패한 Step들은 어떤것들이 있는지

## In-Memory Repository

도메인 개체를 데이터베이스에 유지하지 않으려는 2가지 시나리오.

- 한 가지 이유는 속도. 각 커밋 지점에 도메인 개체를 저장하는 데 추가 시간이 걸리기 때문.

- 두번째 특정 작업에 대한 상태를 유지할 필요가 없음.

이러한 이유로 Spring Batch는 작업 저장소의 메모리 내 맵 버전을 제공한다.

``` java
public class InMemoryBatchConfigurer implements BatchConfigurer {

    private PlatformTransactionManager transactionManager;
    private JobRepository jobRepository;
    private JobLauncher jobLauncher;
    private JobExplorer jobExplorer;

    @Override
    public PlatformTransactionManager getTransactionManager() {
        return transactionManager;
    }

    @Override
    public JobRepository getJobRepository() {
        return jobRepository;
    }

    @Override
    public JobLauncher getJobLauncher() {
        return jobLauncher;
    }

    @Override
    public JobExplorer getJobExplorer() {
        return jobExplorer;
    }

    @PostConstruct
    public void initialize() {
        //transactionManager define
        if (this.transactionManager == null) {
            this.transactionManager = new ResourcelessTransactionManager();
        }
        try {
            //jobRepository define
            MapJobRepositoryFactoryBean jobRepositoryFactoryBean =
                    new MapJobRepositoryFactoryBean(this.transactionManager);
            jobRepositoryFactoryBean.afterPropertiesSet();
            this.jobRepository = jobRepositoryFactoryBean.getObject();

            //jobExplorer define
            MapJobExplorerFactoryBean jobExplorerFactoryBean =
                    new MapJobExplorerFactoryBean(jobRepositoryFactoryBean);
            jobExplorerFactoryBean.afterPropertiesSet();
            this.jobExplorer = jobExplorerFactoryBean.getObject();

            //jobLauncher define
            SimpleJobLauncher jobLauncher = new SimpleJobLauncher();
            jobLauncher.setJobRepository(jobRepository);
            jobLauncher.afterPropertiesSet();
            this.jobLauncher = jobLauncher;
        } catch (Exception e) {
            throw new BatchConfigurationException(e);
        }
    }
}
```

## Job

![Job](https://user-images.githubusercontent.com/24283191/69026697-495f7000-0a0f-11ea-963e-a75b7a2ef814.png)

```java
@Configuration
public class BatchConfig {

    @Autowired
    private JobBuilderFactory jobBuilderFactory;
    @Autowired
    private StepBuilderFactory stepBuilderFactory;
    @Autowired
    private InterceptingJobExecution interceptingJobExecution;
    @Autowired
    private RestTemplate restTemplate;

    @Bean
    public Job apiJob() {
        return jobBuilderFactory.get("apiJob")
                .incrementer(new RunIdIncrementer())
                .flow(apiJobStep())
                .end()
                .listener(interceptingJobExecution)
                .build();
    }


    @Bean
    public Step apiJobStep() {
        return this.stepBuilderFactory.get("apiJobStep")
                .<Object, Object>chunk(10) // 최소 Chunk의 사이즈는 10임. 그이하는 Exception
                .reader(reader(restTemplate, null))
                .processor(processor())
                .writer(writer())
                .allowStartIfComplete(true)
                .build();
    }

    /**
     * @return
     * @JobScope는 Job 실행시점에 Bean이 생성됨.
     * org.springframework.batch.item.ItemReader는 인터페이스이다. 구현 클래스는 어노테이션 기반 listner 구성에 대해 실행되지 않음.
     * @Bean 메소드에서 @StepScope를 사용하는 경우 listner 어노테이션을 사용할 수 있도록 구현 클래스를 리턴해야함
     */
    @Bean
    @StepScope
    public JsonReader reader(RestTemplate restTemplate, @Value("#{jobParameters[lastCreatedDate]}") String lastCreatedDate) {
        String requestUrl = String.format(url, lastCreatedDate, limit, desc);

        return new JsonReader(restTemplate, requestUrl);
    }

    @Bean
    public JsonProcessor processor() {
        return new JsonProcessor();
    }

    @Bean
    public JsonWrite writer() {
        return new JsonWrite();
    }

}
```

## Chunk Processing

스프링 배치는 'Chunk 지향'처리 스타일을 사용한다.
Chunk 지향 처리는 한 번에 하나씩 데이터를 읽고 트랜잭션 경계 내에서 작성된 Chunk를 작성하는 것을 말한다.
하나의 Item이 ItemReader에서 읽혀지고 ItemProcessor로 전달된다.
읽은 Item 수가 커밋 간격과 같으면 ItemWriter가 전체 Chunk를 작성한 다음 트랜잭션이 커밋된다.

![chunk-oriented Processing](https://user-images.githubusercontent.com/24283191/69026696-48c6d980-0a0f-11ea-814f-fef33b5711d7.png)

## Reader

API를 호출하고 Json을 jackson 라이브러리를 통하여 Java `List<Object>`로 컨버팅 후 Object로 반환

```java
public class JsonReader implements ItemReader<NewsDto> {

    //요청 url
    private String requestUrl;
    private final RestTemplate restTemplate;

    //list index
    private int index;
    private List<Object> list;


    /**
     * JsonReader 는 StepScope로 설정되어 프로그램 실행때마다 Baen을 재생성 하기때문에 Autowired 어노테이션이 제기능을 못함.
     * 미리 설정한 restTemplate Bean을 사용하기위해 BatchConfig에서 생성자로 전달받음.
     *
     * @param restTemplate
     * @param requestUrl
     */
    public JsonReader(RestTemplate restTemplate,String requestUrl) {
        this.restTemplate = restTemplate;
        this.requestUrl = requestUrl;
        init();
    }

    @Override
    public Object read() {
        Object content = null;

        //리스트 사이즈만큼 content를 하나씩 읽는다.
        if (index < list.size()) {
            content = list.get(index);
            index++;
        }

        return content;
    }

    private void init() {
        //초기 0
        index = 0;
        list = setList();
    }


    private List<Object> setList() {
        //LocalDateTime 파싱을 위한 CustomObjectMapper 정의.
        ObjectMapper objectMapper = new CustomObjectMapper();
        List<Object> objectList = null;

        try {
            ResponseEntity<String> response = restTemplate.getForEntity(requestUrl, String.class);

            log.info("call URL {}", requestUrl);

            JsonNode root = objectMapper.readTree(response.getBody());

            JsonNode news = root.path("res").path("channel").path("news");
            ObjectReader objectReader = objectMapper.readerFor(new TypeReference<List<Object>>() {
            });

            objectList = objectReader.readValue(news);
        } catch (Exception e) {

            log.info(e.toString());
        }

        return objectList;
    }
}
```

## Processor

Database로 insert, update 하기 위해 내용 이미지 분리 및 html 파싱작업  

```java
public class JsonProcessor implements ItemProcessor<Object, Object> {

    @Override
    public Object process(Object item) {
        //item을 받아 자신에게 맞는 Object로 정의한후 리턴.
        return item;
    }
}

```

## Writer

Database로 저장하는 작업.

```java
public class JsonWrite implements ItemWriter<Object> {

    @Override
    @Transactional("oracleTransactionManager")
    public void write(List<? extends Object> items) {
        //Processor에서 정의한 Object를 List<Object>로 받는다 size는 초기에 셋팅한 Chuck단위인 10개씩
        for (Object item : items) {
          //DB insert or update

          //CMS Queue ADD
          cmsDtoQueue.add(new CmsDto(robotNewsContent.getContentId()));
        }
    }
}
```

## EnableScheduling

0 0/1 8-18 * * MON-FRI 월~금 08시 00분 부터 18시 59분 까지 1분마다

``` java
@Component
public class AppScheduler {

    @Autowired
    private Job job;

    @Autowired
    private JobLauncher jobLauncher;

    //JPA Repository
    @Autowired
    private RobotNewsRepository robotNewsRepository;

    //CMS 기사 생성 Service
    @Autowired
    private ArticleService articleService;

    //CMS 기사 생성 금지 Flag
    @Value("${cms.info.enabled}")
    private boolean cmsEnabled;

    //스케쥴러 0 0/1 8-18 * * MON-FRI
    @Scheduled(cron = "${job.scheduled}")
    public void work() {
        //실행 시간 밀리타임.
        Long runTime = System.currentTimeMillis();

        log.info("Job Started at : {}", runTime);

        //마지막 수집된 기사 생성시간 조회
        String lastCreatedDate = robotNewsRepository.getLastCreatedDate();
        //Null 이면 현재 날짜에 00:00:00 으로 가져옴.
        if (lastCreatedDate == null) {
            lastCreatedDate = LocalDateTime.now().format(DateTimeFormatter.ofPattern("yyyy-MM-dd")) + " 00:00:00";
        }
        //JobParameters
        JobParameters param =
                new JobParametersBuilder()
                        .addString("lastCreatedDate", lastCreatedDate)
                        .addLong("runTime", runTime).toJobParameters();
        //Job 실행 Class
        JobExecution execution = null;
        try {
            log.info("Job Param : {}", param.toString());
            execution = jobLauncher.run(job, param);
            log.info("Job finished apiJob with status : {}", execution.getStatus());
        } catch (Exception e) {
            e.printStackTrace();
        }

        if (cmsEnabled) {
            //Job 실행 결과 COMPLETED이면  저장된 Queue Size 만큼 기사생성 Service 호출.
            if (execution.getStatus().equals(BatchStatus.COMPLETED)) {
                for (int i = 0; i < cmsDtoQueue.size(); i++) {
                    articleService.articleTask();
                }
            }
        }
    }
}
```

## @EnableAsync

- CMS에서 기사를 생성하는데 10~20초 정도 소요됨.
- 만약 기사가 5개 생성된다면 최대 100초가 걸림.
- 즉 1분안에 모든 로직이 완료된다는 보장이 없음.

## TaskExecutor

``` java
//최초 생성되는 스레드 사이즈
public static final int CORE_TASK_POOL_SIZE = 5;
//해당 풀에 최대로 유지할 수 있는 스레드 사이즈
public static final int MAX_TASK_POOL_SIZE = 30;
//CorePoll보다 스레드가 많아졌을 경우, 남는 스레드가 없을 경우 큐에 담을수있는 사이즈
public static final int QUEUE_CAPACITY_SIZE = 10;

@Bean(name = "taskExecutor")
public TaskExecutor executor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(CORE_TASK_POOL_SIZE);
    executor.setMaxPoolSize(MAX_TASK_POOL_SIZE);
    executor.setQueueCapacity(QUEUE_CAPACITY_SIZE);
    executor.setThreadNamePrefix("task-pool-");

    executor.initialize();
    return executor;
}
```

## static Queue 생성

``` java
//BasicConfig class
public static final Queue<CmsDto> cmsDtoQueue = new ConcurrentLinkedQueue<>();
```

프로세스를 재시작할때 기사 생성 누락을 방지하기위해 누락된 기사 조회 및 Queue Add

``` java
// 모든 Bean 특성이 설정되면 구성 및 최종 초기화
//InitializingMetaDataBean class
@Component
public class InitializingMetaDataBean implements InitializingBean {

    @Autowired
    private RobotNewsRepository robotNewsRepository;

    @Override
    public void afterPropertiesSet() throws Exception {
        //오늘 날짜에 생성되지 않은 기사를 큐에 초기 셋팅.
        String today = LocalDateTime.now().format(DateTimeFormatter.ofPattern("yyyyMMdd"));
        List<String> contentList = robotNewsRepository.getNotYetCreatedContent(today);
        log.info("InitializingMetaDataBean Queue Size {}",contentList.size());

        for (String contentId : contentList) {
            cmsDtoQueue.add(new CmsDto(contentId));
        }

    }
}
```

비동기 Service Define

``` java

//CMS 기사 생성 DTO
public class CmsDto {
    private String contentId;
    private String cmsId;
    private LocalDateTime expireTime;

    public CmsDto(String contentId) {
        this.contentId = contentId;
    }
}

//CMS 기사 생성 Service
@Async("taskExecutor")
public RepeatStatus articleTask() {
    if (cmsDtoQueue.size() <= 0) return RepeatStatus.FINISHED;
    CmsDto cmsDto = cmsDtoQueue.poll();

    String contentId = cmsDto.getContentId();
    String url = String.format(cmsUrl+"?fname=%s&siteid=%s", contentId, siteId);
    log.info("CMS 기사 생성 시작 siteId {} contentId {}", siteId, contentId);
    log.info("Request URL {}", url);

    ResponseEntity<String> response = null;
    try {
        response = restTemplate.getForEntity(url, String.class);

    } catch (ResourceAccessException e) {
        e.printStackTrace();
    } catch (RestClientException e) {
        e.printStackTrace();
    }
    log.info("Response statusCode {}", response.getStatusCode());

    String cmsId = robotNewsRepository.getExistCmsId(contentId);
    log.info("CMSID IS {}", cmsId);

    if (cmsId != null) {
        cmsDto.setCmsId(cmsId);
    }

    if (response.getStatusCode().equals(HttpStatus.OK) && cmsDto.getCmsId() != null) {
        log.info("CMS 기사 생성 성공 siteId {} contentId {}", siteId, contentId);
    } else {
        //만료시간 10분 추가.
        LocalDateTime failNowTime = LocalDateTime.now();

        if (cmsDto.getExpireTime() != null) {

            if (cmsDto.getExpireTime().isAfter(failNowTime)) {
                log.error("CMS 기사 생성 시간 10분 초과 contentId {} expireTime {}", contentId, cmsDto.getExpireTime());
                return RepeatStatus.FINISHED;
            }
        } else {
            cmsDto.setExpireTime(failNowTime.plusMinutes(10));
            log.error("CMS 기사 생성 실패 siteId {} contentId {} expireTime {} ", siteId, contentId, cmsDto.getExpireTime());
        }
        cmsDtoQueue.add(cmsDto);
    }
    return RepeatStatus.FINISHED;
}
```

## CustomObjectMapper

LocalDateTime을 한꺼번에 포맷팅하기 위해 jackson object mapper를 재정의 한다.

``` java
public class CustomLocalDateTimeDeserializer extends JsonDeserializer<LocalDateTime> {

    private static final String DEFAULT_DATE_FORMAT = "yyyy-MM-dd'T'HH:mm:ss.SSSXXX";
    @Override
    public LocalDateTime deserialize(JsonParser jsonParser, DeserializationContext ctxt) throws IOException, JsonProcessingException {

        String valueAsString = jsonParser.getValueAsString();
        if (StringUtils.isEmpty(valueAsString)) {
            return null;
        }
        try {
            DateTimeFormatter formatter = DateTimeFormatter.ofPattern(DEFAULT_DATE_FORMAT)
                    .withZone(ZoneId.of("UTC"));
            LocalDateTime ldt = LocalDateTime.parse(valueAsString,formatter);
            return ldt;
        }catch(Exception e) {
            return null;
        }

    }
}

public class CustomObjectMapper extends ObjectMapper {
    public CustomObjectMapper() {
        SimpleModule simpleModule = new SimpleModule();
        simpleModule.addDeserializer(LocalDateTime.class, new CustomLocalDateTimeDeserializer());

        registerModule(simpleModule);
        // 없는 필드로 인한 오류 무시
        configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
    }
}
```
