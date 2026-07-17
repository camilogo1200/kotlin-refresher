# Spring annotation and architectural role guide

| Annotation | Project role | Notes |
|---|---|---|
| `@SpringBootApplication` | Bootstrap/composition root | Keep business rules outside it |
| `@RestController` | HTTP input adapter | DTO mapping and status/headers |
| `@RestControllerAdvice` | HTTP error adapter | Maps transport-neutral failures to Problem Details |
| `@Service` | Application use-case service/interactor | Instantiate directly in unit tests |
| `@Repository` | Persistence output adapter | Spring Data interfaces are already discovered proxies |
| `@Component` | Other adapter/infrastructure | Prefer a narrower stereotype when one exists |
| `@Configuration` / `@Bean` | Bootstrap wiring | Dispatcher, client, clock, semaphore, mapper configuration |
| `@ConfigurationProperties` | Typed runtime configuration | Immutable and validated |
| `@Transactional` | Local transaction boundary | Proxy-based; no distributed transaction semantics |
| `@KafkaListener` | Kafka input-adapter method | Delegate immediately to an input port |
| `@ApplicationModuleListener` | In-process module-event adapter | Keep module behavior behind use cases |
| `@Table`, `@Id`, `@MappedCollection`, `@Version` | Data adapter mapping | Never place in domain |
| `@GetExchange` / `@HttpExchange` | Technical HTTP client interface | Keep behind an output port |

## Naming

```text
CreateBookUseCase             input port
CreateBookService             @Service implementation / interactor
BookRepository                output port
JdbcBookRepository            @Repository implementation
PriceProvider                 output port
WebClientPriceProvider        @Component implementation
```

Avoid `IRepository`, `ServiceImpl`, and technology names in business ports.

