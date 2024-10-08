---
layout: single
title:  "Тестирование со Spring Boot"
date:   2024-06-29 18:18:05 +0300
categories: java springboot testing
---

В этом руководстве мы рассмотрим написание тестов с использованием встроенной поддержки фреймворка Spring Boot. Мы охватим модульные тесты, которые могут выполняться в изоляции, а также интеграционные тесты, которые будут загружать контекст Spring перед выполнением тестов.

## 1. Настройка проекта
Приложение, которое мы будем использовать в этой статье, представляет собой API, предоставляющее некоторые базовые операции с ресурсом "Employee". Это типичная многослойная архитектура — вызов API обрабатывается от Контроллера к Сервису и далее к слою хранения данных.

## 2. Зависимости Maven
Сначала давайте добавим наши зависимости для тестирования:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
    <version>3.3.4</version>
</dependency>
<dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <scope>test</scope>
</dependency>
```

Зависимость spring-boot-starter-test является основной и содержит большинство элементов, необходимых для наших тестов.

База данных H2 — это наша база данных в памяти. Она устраняет необходимость в настройке и запуске реальной базы данных для тестирования.

## 3. Интеграционное тестирование с @SpringBootTest

Как следует из названия, интеграционные тесты сосредоточены на интеграции различных слоев приложения. Это также означает, что здесь не используется мокирование.

В идеале мы должны отделять интеграционные тесты от модульных и не запускать их вместе с модульными тестами. Мы можем сделать это, используя другой профиль, чтобы запускать только интеграционные тесты. Причины для этого могут быть следующими: интеграционные тесты требуют много времени и могут нуждаться в реальной базе данных для выполнения.

Однако в этой статье мы не будем на этом сосредотачиваться и вместо этого воспользуемся хранилищем данных H2 в памяти.

Интеграционные тесты требуют запуска контейнера для выполнения тестовых случаев. Поэтому для этого требуется дополнительная настройка — все это легко реализуется в Spring Boot:

```java
@RunWith(SpringRunner.class)
@SpringBootTest(
  webEnvironment = SpringBootTest.WebEnvironment.MOCK,
  classes = Application.class)
@AutoConfigureMockMvc
@TestPropertySource(
  locations = "classpath:application-integrationtest.properties")
public class EmployeeRestControllerIntegrationTest {

    @Autowired
    private MockMvc mvc;

    @Autowired
    private EmployeeRepository repository;

    // write test cases here
}
```

Аннотация @SpringBootTest полезна, когда нам нужно загрузить весь контейнер. Эта аннотация работает, создавая ApplicationContext, который будет использоваться в наших тестах.

Мы можем использовать атрибут webEnvironment аннотации @SpringBootTest для настройки нашей рабочей среды; здесь мы используем WebEnvironment.MOCK, чтобы контейнер работал в имитированной среде сервлетов.

Далее, аннотация @TestPropertySource помогает настроить расположение файлов свойств, специфичных для наших тестов. Обратите внимание, что файл свойств, загруженный с помощью @TestPropertySource, будет переопределять существующий файл application.properties.

Файл application-integrationtest.properties содержит детали для настройки хранилища данных:

```
spring.datasource.url = jdbc:h2:mem:test
spring.jpa.properties.hibernate.dialect = org.hibernate.dialect.H2Dialect
```
Если мы хотим запускать наши интеграционные тесты с использованием MySQL, мы можем изменить указанные выше значения в файле свойств.

Тестовые случаи для интеграционных тестов могут выглядеть аналогично модульным тестам слоя Контроллера:

```java
@Test
public void givenEmployees_whenGetEmployees_thenStatus200()
  throws Exception {

    createTestEmployee("bob");

    mvc.perform(get("/api/employees")
      .contentType(MediaType.APPLICATION_JSON))
      .andExpect(status().isOk())
      .andExpect(content()
      .contentTypeCompatibleWith(MediaType.APPLICATION_JSON))
      .andExpect(jsonPath("$[0].name", is("bob")));
}
```

Разница с модульными тестами слоя Контроллера заключается в том, что здесь ничего не мокируется, и будут выполняться сценарии от начала до конца.

## 4. Конфигурация тестов с помощью @TestConfiguration

Как мы видели в предыдущем разделе, тест, аннотированный @SpringBootTest, загрузит полный контекст приложения, что означает, что мы можем использовать @Autowire для любого бина, который будет обнаружен с помощью сканирования компонентов в нашем тесте:

```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class EmployeeServiceImplIntegrationTest {

    @Autowired
    private EmployeeService employeeService;

    // class code ...
}
```

Однако мы можем захотеть избежать загрузки реального контекста приложения и использовать специальную конфигурацию для тестов. Мы можем достичь этого с помощью аннотации @TestConfiguration. Существует два способа использования этой аннотации. Либо на статическом внутреннем классе в том же тестовом классе, где мы хотим использовать @Autowire для бина:

```java
@RunWith(SpringRunner.class)
public class EmployeeServiceImplIntegrationTest {

    @TestConfiguration
    static class EmployeeServiceImplTestContextConfiguration {
        @Bean
        public EmployeeService employeeService() {
            return new EmployeeService() {
                // implement methods
            };
        }
    }

    @Autowired
    private EmployeeService employeeService;
}
```

В качестве альтернативы мы можем создать отдельный класс конфигурации для тестов:

```java
@TestConfiguration
public class EmployeeServiceImplTestContextConfiguration {

    @Bean
    public EmployeeService employeeService() {
        return new EmployeeService() {
            // implement methods
        };
    }
}
```

Классы конфигурации, аннотированные @TestConfiguration, исключаются из сканирования компонентов, поэтому нам нужно явно импортировать их в каждом тесте, где мы хотим использовать @Autowire. Мы можем сделать это с помощью аннотации @Import:

```java
@RunWith(SpringRunner.class)
@Import(EmployeeServiceImplTestContextConfiguration.class)
public class EmployeeServiceImplIntegrationTest {

    @Autowired
    private EmployeeService employeeService;

    // remaining class code
}
```

## 5. Мокирование с помощью @MockBean

Наш код слоя Сервиса зависит от нашего Репозитория:

```java
@Service
public class EmployeeServiceImpl implements EmployeeService {

    @Autowired
    private EmployeeRepository employeeRepository;

    @Override
    public Employee getEmployeeByName(String name) {
        return employeeRepository.findByName(name);
    }
}
```

Однако, чтобы протестировать слой Сервиса, нам не нужно знать или беспокоиться о том, как реализован слой Хранения данных. В идеале мы должны иметь возможность писать и тестировать наш код слоя Сервиса без подключения полного слоя Хранения данных.

Для достижения этой цели мы можем использовать поддержку мокирования, предоставляемую Spring Boot Test.

Давайте сначала взглянем на скелет тестового класса:

```java
@RunWith(SpringRunner.class)
public class EmployeeServiceImplIntegrationTest {

    @TestConfiguration
    static class EmployeeServiceImplTestContextConfiguration {

        @Bean
        public EmployeeService employeeService() {
            return new EmployeeServiceImpl();
        }
    }

    @Autowired
    private EmployeeService employeeService;

    @MockBean
    private EmployeeRepository employeeRepository;

    // write test cases here
}
```

Чтобы проверить класс Сервиса, нам нужно создать экземпляр класса Сервиса и сделать его доступным как @Bean, чтобы мы могли использовать @Autowire в нашем тестовом классе. Мы можем достичь этой конфигурации с помощью аннотации @TestConfiguration.

Еще один интересный момент здесь — это использование @MockBean. Она создает мок для EmployeeRepository, который можно использовать для обхода вызова к реальному EmployeeRepository:

```java
@Before
public void setUp() {
    Employee alex = new Employee("alex");

    Mockito.when(employeeRepository.findByName(alex.getName()))
      .thenReturn(alex);
}
```

Поскольку настройка завершена, тестовый случай будет проще:

```java
@Test
public void whenValidName_thenEmployeeShouldBeFound() {
    String name = "alex";
    Employee found = employeeService.getEmployeeByName(name);

     assertThat(found.getName())
      .isEqualTo(name);
 }
```

## 6. Интеграционное тестирование с помощью @DataJpaTest

Мы будем работать с сущностью под названием Employee (Сотрудник), которая имеет свойства id (идентификатор) и name (имя).

```java
@Entity
@Table(name = "person")
public class Employee {

    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    private Long id;

    @Size(min = 3, max = 20)
    private String name;

    // standard getters and setters, constructors
}
```

А вот и наш репозиторий, использующий Spring Data JPA:

```java
@Repository
public interface EmployeeRepository extends JpaRepository<Employee, Long> {

    public Employee findByName(String name);

}
```

На этом код для слоя Хранения данных завершен. Теперь давайте перейдем к написанию нашего тестового класса.

Сначала создадим каркас нашего тестового класса:

```java
@RunWith(SpringRunner.class)
@DataJpaTest
public class EmployeeRepositoryIntegrationTest {

    @Autowired
    private TestEntityManager entityManager;

    @Autowired
    private EmployeeRepository employeeRepository;

    // write test cases here

}
```

@RunWith(SpringRunner.class) обеспечивает связь между возможностями тестирования Spring Boot и JUnit. Каждый раз, когда мы используем какие-либо функции тестирования Spring Boot в наших тестах JUnit, эта аннотация будет необходима.

@DataJpaTest предоставляет стандартную настройку, необходимую для тестирования слоя Хранения данных:

- конфигурация H2, базы данных в памяти
- настройка Hibernate, Spring Data и DataSource
- выполнение @EntityScan
- включение логирования SQL

Чтобы выполнить операции с базой данных, нам нужны некоторые записи, уже находящиеся в нашей базе данных. Для настройки этих данных мы можем использовать TestEntityManager.

TestEntityManager в Spring Boot является альтернативой стандартному JPA EntityManager и предоставляет методы, которые обычно используются при написании тестов.

EmployeeRepository — это компонент, который мы собираемся тестировать.

Теперь давайте напишем наш первый тестовый случай:

```java
@Test
public void whenFindByName_thenReturnEmployee() {
    // given
    Employee alex = new Employee("alex");
    entityManager.persist(alex);
    entityManager.flush();

    // when
    Employee found = employeeRepository.findByName(alex.getName());

    // then
    assertThat(found.getName())
      .isEqualTo(alex.getName());
}
```

В приведенном выше тесте мы используем TestEntityManager для вставки записи Employee в базу данных и считываем ее с помощью API поиска по имени.

Часть assertThat(…) принадлежит библиотеке Assertj, которая поставляется в комплекте со Spring Boot.

## 7. Модульное тестирование с помощью @WebMvcTest

Наш контроллер зависит от слоя Сервиса; давайте включим только один метод для простоты:

```java
@RestController
@RequestMapping("/api")
public class EmployeeRestController {

    @Autowired
    private EmployeeService employeeService;

    @GetMapping("/employees")
    public List<Employee> getAllEmployees() {
        return employeeService.getAllEmployees();
    }
}
```

Поскольку мы сосредоточены только на коде контроллера, естественно, что мы будем использовать мокирование кода слоя Сервиса для наших модульных тестов:

```java
@RunWith(SpringRunner.class)
@WebMvcTest(EmployeeRestController.class)
public class EmployeeRestControllerIntegrationTest {

    @Autowired
    private MockMvc mvc;

    @MockBean
    private EmployeeService service;

    // write test cases here
}
```

Для тестирования контроллеров мы можем использовать @WebMvcTest. Он автоматически настраивает инфраструктуру Spring MVC для наших модульных тестов.

В большинстве случаев @WebMvcTest будет ограничен инициализацией одного контроллера. Мы также можем использовать его вместе с @MockBean, чтобы предоставить мокированные реализации для любых необходимых зависимостей.

@WebMvcTest также автоматически настраивает MockMvc, который предлагает мощный способ простого тестирования MVC контроллеров без запуска полного HTTP-сервера.

Сказав это, давайте напишем наш тестовый случай:

```java
@Test
public void givenEmployees_whenGetEmployees_thenReturnJsonArray()
  throws Exception {

    Employee alex = new Employee("alex");

    List<Employee> allEmployees = Arrays.asList(alex);

    given(service.getAllEmployees()).willReturn(allEmployees);

    mvc.perform(get("/api/employees")
      .contentType(MediaType.APPLICATION_JSON))
      .andExpect(status().isOk())
      .andExpect(jsonPath("$", hasSize(1)))
      .andExpect(jsonPath("$[0].name", is(alex.getName())));
}
```

Вызов метода get(…) можно заменить другими методами, соответствующими HTTP-методам, таким как put(), post() и т. д. Обратите внимание, что мы также устанавливаем тип содержимого в запросе.

MockMvc является гибким, и мы можем создавать любые запросы с его помощью.

## 8. Автоматически настраиваемые тесты

Одной из удивительных особенностей автоматической настройки аннотаций Spring Boot является то, что они помогают загружать части полного приложения и тестировать специфические слои кодовой базы.

В дополнение к вышеупомянутым аннотациям, вот список нескольких широко используемых аннотаций:

- **@WebFluxTest**: Мы можем использовать аннотацию @WebFluxTest для тестирования контроллеров Spring WebFlux. Она часто используется вместе с @MockBean для предоставления мокированных реализаций необходимых зависимостей.
- **@JdbcTest**: Мы можем использовать аннотацию @JdbcTest для тестирования JPA приложений, но она предназначена для тестов, которые требуют только DataSource. Аннотация настраивает встроенную базу данных в памяти и JdbcTemplate.
- **@JooqTest**: Для тестирования, связанного с jOOQ, мы можем использовать аннотацию @JooqTest, которая настраивает DSLContext.
- **@DataMongoTest**: Для тестирования приложений MongoDB полезна аннотация @DataMongoTest. По умолчанию она настраивает встроенную базу данных MongoDB в памяти, если драйвер доступен через зависимости, настраивает MongoTemplate, сканирует классы с аннотацией @Document и настраивает репозитории Spring Data MongoDB.
- **@DataRedisTest**: Упрощает тестирование приложений Redis. Она сканирует классы с аннотацией @RedisHash и по умолчанию настраивает репозитории Spring Data Redis.
- **@DataLdapTest**: Настраивает встроенный LDAP в памяти (если доступен), настраивает LdapTemplate, сканирует классы с аннотацией @Entry и по умолчанию настраивает репозитории Spring Data LDAP.
- **@RestClientTest**: Обычно мы используем аннотацию @RestClientTest для тестирования REST-клиентов. Она автоматически настраивает различные зависимости, такие как поддержка Jackson, GSON и Jsonb; настраивает RestTemplateBuilder и по умолчанию добавляет поддержку MockRestServiceServer.
- **@JsonTest**: Инициализирует контекст приложения Spring только с теми бинами, которые необходимы для тестирования сериализации JSON.

[оригинал](https://www.baeldung.com/spring-boot-testing)
