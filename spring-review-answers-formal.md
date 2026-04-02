# Spring Review Answers

## 1. Spring Security

Spring Security — это модуль Spring Framework, предназначенный для построения слоя безопасности приложения. Он обеспечивает аутентификацию, авторизацию, защиту HTTP-запросов, управление сессиями, обработку выхода пользователя из системы, защиту от CSRF-атак и интеграцию с различными механизмами входа.

В архитектурном плане Spring Security работает как цепочка servlet-фильтров. Каждый входящий HTTP-запрос проходит через набор security-фильтров до попадания в контроллер. В процессе обработки система извлекает учетные данные пользователя, выполняет аутентификацию, сохраняет результат в `SecurityContext` и затем проверяет права доступа к ресурсу.

К основным возможностям Spring Security относятся:
- form login;
- HTTP Basic;
- JWT-аутентификация;
- OAuth2;
- LDAP;
- защита методов через аннотации;
- разграничение доступа по URL.

Пример базовой конфигурации:

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/public/**").permitAll()
                .requestMatchers("/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .httpBasic(Customizer.withDefaults());

        return http.build();
    }
}
```

Spring Security позволяет централизованно описывать правила доступа и отделять вопросы безопасности от бизнес-логики приложения.

## 2. Что такое авторизация, аутентификация

Аутентификация — это процесс установления личности пользователя. Она отвечает на вопрос: кто именно обращается к системе. Для этого приложение проверяет представленные учетные данные: логин и пароль, токен, сертификат, одноразовый код или иной идентификатор.

Авторизация — это процесс проверки полномочий уже аутентифицированного пользователя. Она отвечает на вопрос: какие действия пользователь имеет право выполнять в системе.

Различие между этими понятиями:
- аутентификация подтверждает личность;
- авторизация определяет границы доступа.

Пример:
- пользователь вводит логин и пароль;
- система проверяет корректность учетных данных;
- после успешного входа система проверяет, имеет ли пользователь роль `ADMIN` для доступа к защищённому ресурсу.

Пример авторизации в Spring Security:

```java
@PreAuthorize("hasRole('ADMIN')")
@DeleteMapping("/users/{id}")
public void deleteUser(@PathVariable Long id) {
    userService.deleteById(id);
}
```

Если пользователь успешно аутентифицирован, но не обладает необходимой ролью, доступ к методу будет запрещён.

## 3. Объекты Principal, Authorities, Authentication

В Spring Security состояние безопасности текущего пользователя описывается несколькими ключевыми объектами.

`Principal` — объект, представляющий пользователя. В простейшем случае это имя пользователя, однако на практике в качестве principal обычно используется объект `UserDetails` или собственный класс пользователя.

`Authorities` — набор прав пользователя. Эти права представлены коллекцией `GrantedAuthority` и могут включать:
- роли, например `ROLE_USER`, `ROLE_ADMIN`;
- отдельные разрешения, например `READ_REPORTS`, `WRITE_USERS`.

`Authentication` — центральный объект системы безопасности, содержащий:
- `principal` — кто пользователь;
- `credentials` — учетные данные;
- `authorities` — права пользователя;
- `authenticated` — признак успешной аутентификации.

После успешной аутентификации объект `Authentication` сохраняется в `SecurityContext`.

Пример получения текущего пользователя в контроллере:

```java
@GetMapping("/profile")
public String profile(Authentication authentication) {
    return authentication.getName();
}
```

Пример получения `Principal`:

```java
@GetMapping("/me")
public String me(Principal principal) {
    return principal.getName();
}
```

Пример получения списка прав:

```java
@GetMapping("/authorities")
public List<String> authorities(Authentication authentication) {
    return authentication.getAuthorities()
        .stream()
        .map(GrantedAuthority::getAuthority)
        .toList();
}
```

Таким образом:
- `Principal` отражает пользователя;
- `Authorities` отражают его права;
- `Authentication` объединяет сведения о пользователе и состоянии его аутентификации.

## 4. Чем отличается InMemoryAuthentication от basicAuthentication

`InMemoryAuthentication` и `Basic Authentication` относятся к разным аспектам безопасности.

`InMemoryAuthentication` — это способ хранения пользователей внутри приложения, в оперативной памяти. Пользователи, пароли и роли задаются в конфигурации и не извлекаются из базы данных или внешнего сервиса.

Пример in-memory-хранилища пользователей:

```java
@Bean
public UserDetailsService userDetailsService() {
    UserDetails user = User.withUsername("user")
        .password("{noop}123")
        .roles("USER")
        .build();

    UserDetails admin = User.withUsername("admin")
        .password("{noop}admin")
        .roles("ADMIN")
        .build();

    return new InMemoryUserDetailsManager(user, admin);
}
```

`Basic Authentication` — это способ передачи логина и пароля в HTTP-запросе через заголовок `Authorization`.

Пример включения HTTP Basic:

```java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    http
        .authorizeHttpRequests(auth -> auth.anyRequest().authenticated())
        .httpBasic(Customizer.withDefaults());

    return http.build();
}
```

Следовательно:
- `InMemoryAuthentication` отвечает за источник пользователей;
- `Basic Authentication` отвечает за механизм передачи учетных данных.

Эти механизмы могут использоваться совместно. В таком случае логин и пароль передаются через HTTP Basic, а проверяются по пользователям, хранящимся в памяти приложения.

## 5. Как мы можем добавить секьюрность к контроллеру

Безопасность контроллера в Spring может быть добавлена несколькими способами.

### Способ 1. Ограничение доступа на уровне URL через `SecurityFilterChain`

Правила задаются централизованно в конфигурации безопасности.

```java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    http
        .authorizeHttpRequests(auth -> auth
            .requestMatchers("/admin/**").hasRole("ADMIN")
            .requestMatchers("/user/**").hasAnyRole("USER", "ADMIN")
            .anyRequest().authenticated()
        )
        .formLogin(Customizer.withDefaults());

    return http.build();
}
```

### Способ 2. Ограничение доступа на уровне метода через аннотации

Для этого используется method security.

```java
@Configuration
@EnableMethodSecurity
public class MethodSecurityConfig {
}
```

Пример защиты метода:

```java
@RestController
public class AdminController {

    @PreAuthorize("hasRole('ADMIN')")
    @GetMapping("/admin/report")
    public String getAdminReport() {
        return "secret report";
    }
}
```

### Дополнительные варианты

Также безопасность может быть реализована:
- через сервисный слой;
- через кастомные security-фильтры;
- через JWT и разбор токена на каждом запросе;
- через `@AuthenticationPrincipal`.

Пример использования `@AuthenticationPrincipal`:

```java
@GetMapping("/current-user")
public String currentUser(@AuthenticationPrincipal UserDetails userDetails) {
    return userDetails.getUsername();
}
```

## 6. Связи таблиц many-to-many, one-to-one

### Связь one-to-one

Связь `one-to-one` означает, что одной записи из первой таблицы соответствует одна запись из второй таблицы. Такая модель используется в случаях, когда данные логически разделены на две сущности, но между ними сохраняется строгое соответствие.

Примеры:
- пользователь и профиль;
- человек и паспорт;
- заказ и детализированная платёжная информация.

На уровне БД такая связь обычно реализуется:
- внешним ключом с ограничением `UNIQUE`;
- общей первичной колонкой.

Пример `@OneToOne`:

```java
@Entity
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @OneToOne(cascade = CascadeType.ALL)
    @JoinColumn(name = "profile_id")
    private Profile profile;
}
```

```java
@Entity
public class Profile {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String phone;
}
```

### Связь many-to-many

Связь `many-to-many` означает, что одной записи из первой таблицы соответствует множество записей из второй таблицы, и наоборот.

Примеры:
- студент и курс;
- пользователь и роль;
- товар и категория.

В реляционной БД такая связь реализуется через промежуточную таблицу.

Пример `@ManyToMany`:

```java
@Entity
public class Student {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToMany
    @JoinTable(
        name = "student_course",
        joinColumns = @JoinColumn(name = "student_id"),
        inverseJoinColumns = @JoinColumn(name = "course_id")
    )
    private Set<Course> courses = new HashSet<>();
}
```

```java
@Entity
public class Course {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String title;
}
```

Если у связи появляются собственные атрибуты, например дата назначения или статус, обычно создают отдельную сущность связи вместо прямого `@ManyToMany`.

## 7. Как работают каскады для таблиц и какие они бывают

Каскады в JPA/Hibernate определяют, какие операции над родительской сущностью автоматически применяются к связанным сущностям.

Основные типы каскадов:
- `PERSIST` — каскадное сохранение;
- `MERGE` — каскадное обновление;
- `REMOVE` — каскадное удаление;
- `REFRESH` — обновление состояния из БД;
- `DETACH` — отсоединение от persistence context;
- `ALL` — все перечисленные типы каскадов.

Пример:

```java
@Entity
public class Order {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OrderItem> items = new ArrayList<>();
}
```

```java
@Entity
public class OrderItem {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne
    @JoinColumn(name = "order_id")
    private Order order;
}
```

Если выполнить сохранение объекта `Order`, связанные объекты `OrderItem` также будут сохранены.

Пример:

```java
Order order = new Order();
OrderItem item = new OrderItem();

item.setOrder(order);
order.getItems().add(item);

orderRepository.save(order);
```

Особое значение имеют:
- `CascadeType.REMOVE` — удаляет дочерние объекты при удалении родителя;
- `orphanRemoval = true` — удаляет дочерний объект, если он удалён из коллекции родителя.

Каскады применяются в тех случаях, когда дочерняя сущность логически принадлежит родителю и не должна существовать отдельно.

## 8. Bootstrap

Bootstrap — это frontend-фреймворк для разработки пользовательских интерфейсов. Он предоставляет готовую адаптивную сетку, стандартные компоненты интерфейса, утилитарные CSS-классы и JavaScript-поведение для типовых UI-элементов.

К основным возможностям Bootstrap относятся:
- grid system;
- кнопки;
- формы;
- таблицы;
- карточки;
- навигационные панели;
- модальные окна;
- выпадающие списки;
- вкладки;
- адаптивная вёрстка.

Пример кнопки и контейнера:

```html
<div class="container mt-4">
  <h1 class="mb-3">User Profile</h1>
  <button class="btn btn-primary">Save</button>
</div>
```

Пример сетки:

```html
<div class="container">
  <div class="row">
    <div class="col-md-6">Left block</div>
    <div class="col-md-6">Right block</div>
  </div>
</div>
```

Преимущества Bootstrap:
- высокая скорость разработки;
- большой набор готовых компонентов;
- предсказуемое поведение интерфейса;
- хорошая документация;
- адаптивность по умолчанию.

Недостатки Bootstrap:
- типовой внешний вид интерфейсов;
- ограниченная гибкость при создании уникального дизайна;
- наличие лишних стилей при частичном использовании;
- зависимость от готовой дизайн-модели.

## 9. Rest-сервисы. Их преимущества и недостатки

REST (*Representational State Transfer*) — это архитектурный стиль, в котором приложение предоставляет ресурсы по URI и позволяет работать с ними через стандартные HTTP-методы.

Пример REST-маршрутов:
- `GET /users` — получение списка пользователей;
- `GET /users/5` — получение пользователя по идентификатору;
- `POST /users` — создание пользователя;
- `PUT /users/5` — полное обновление пользователя;
- `PATCH /users/5` — частичное обновление пользователя;
- `DELETE /users/5` — удаление пользователя.

Пример REST-контроллера:

```java
@RestController
@RequestMapping("/users")
public class UserController {

    @GetMapping
    public List<User> findAll() {
        return userService.findAll();
    }

    @GetMapping("/{id}")
    public User findById(@PathVariable Long id) {
        return userService.findById(id);
    }

    @PostMapping
    public User create(@RequestBody User user) {
        return userService.save(user);
    }

    @DeleteMapping("/{id}")
    public void delete(@PathVariable Long id) {
        userService.deleteById(id);
    }
}
```

Преимущества REST:
- простота архитектуры;
- использование стандартов HTTP;
- удобство интеграции между системами;
- независимость клиента и сервера;
- масштабируемость;
- хорошая поддержка во фреймворках и инструментах.

Недостатки REST:
- возможная избыточность данных;
- необходимость нескольких запросов для получения сложных графов данных;
- отсутствие единого стандарта структуры ответа;
- дополнительные задачи по версионированию API.

## 10. Форматы данных, использующиеся в REST-сервисах

REST не привязан к одному формату передачи данных. Наиболее распространёнными являются:
- `JSON`;
- `XML`;
- `plain text`;
- `HTML`;
- бинарные данные.

На практике основным форматом является `JSON`, поскольку он компактный, легко читается и удобно преобразуется в объекты.

Пример JSON:

```json
{
  "id": 1,
  "name": "Ivan",
  "role": "USER"
}
```

Пример XML:

```xml
<user>
    <id>1</id>
    <name>Ivan</name>
    <role>USER</role>
</user>
```

Формат взаимодействия задаётся HTTP-заголовками:
- `Content-Type` — формат тела запроса;
- `Accept` — ожидаемый формат ответа.

Пример:

```http
Content-Type: application/json
Accept: application/json
```

В Spring преобразование JSON в объект и обратно обычно выполняется через Jackson.

## 11. Что такое `@ResponseBody`, `@RequestBody`, `ResponseEntity`

`@RequestBody` используется для чтения тела HTTP-запроса и преобразования его в Java-объект.

Пример:

```java
@PostMapping("/users")
public User create(@RequestBody User user) {
    return userService.save(user);
}
```

Если клиент отправляет JSON, Spring десериализует его в объект `User`.

`@ResponseBody` указывает Spring, что результат выполнения метода нужно записать непосредственно в тело HTTP-ответа, а не трактовать как имя представления.

Пример:

```java
@GetMapping("/hello")
@ResponseBody
public String hello() {
    return "Hello";
}
```

`ResponseEntity` позволяет полностью управлять HTTP-ответом: телом, статусом и заголовками.

Пример:

```java
@GetMapping("/users/{id}")
public ResponseEntity<User> getUser(@PathVariable Long id) {
    User user = userService.findById(id);
    if (user == null) {
        return ResponseEntity.notFound().build();
    }
    return ResponseEntity.ok(user);
}
```

Пример ответа со статусом `201 Created`:

```java
@PostMapping("/users")
public ResponseEntity<User> createUser(@RequestBody User user) {
    User saved = userService.save(user);
    return ResponseEntity.status(HttpStatus.CREATED).body(saved);
}
```

Следовательно:
- `@RequestBody` читает тело запроса;
- `@ResponseBody` формирует тело ответа;
- `ResponseEntity` предоставляет полный контроль над HTTP-ответом.

## 12. Что такое AJAX / fetch

AJAX (*Asynchronous JavaScript and XML*) — это подход к асинхронному взаимодействию браузера с сервером без полной перезагрузки страницы. Он используется для динамической загрузки данных, отправки форм и обновления интерфейса.

Современным стандартным инструментом для такой работы является `fetch`.

Пример GET-запроса:

```javascript
fetch('/api/users')
  .then(response => response.json())
  .then(data => console.log(data));
```

Пример POST-запроса:

```javascript
fetch('/api/users', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    name: 'Ivan',
    role: 'USER'
  })
})
  .then(response => response.json())
  .then(data => console.log(data));
```

Преимущества AJAX/fetch:
- отсутствие полной перезагрузки страницы;
- более быстрый отклик интерфейса;
- удобная работа с REST API;
- асинхронность.

AJAX — это общий подход, а `fetch` — современный API для реализации этого подхода в браузере.

## 13. Чем аннотация `@RestController` отличается от `@Controller`

`@Controller` — базовая аннотация Spring MVC для контроллеров, которые обрабатывают HTTP-запросы и обычно возвращают представления.

Пример:

```java
@Controller
public class PageController {

    @GetMapping("/home")
    public String home(Model model) {
        model.addAttribute("message", "Welcome");
        return "home";
    }
}
```

В данном случае строка `"home"` интерпретируется как имя представления.

`@RestController` объединяет:
- `@Controller`
- `@ResponseBody`

Поэтому все методы такого класса возвращают данные непосредственно в HTTP-ответ.

Пример:

```java
@RestController
@RequestMapping("/api/users")
public class UserRestController {

    @GetMapping
    public List<User> findAll() {
        return userService.findAll();
    }
}
```

Следовательно:
- `@Controller` используется для MVC-приложений и возврата страниц;
- `@RestController` используется для REST API и возврата данных.

## 14. RestTemplate и его методы

`RestTemplate` — это HTTP-клиент Spring для вызова внешних REST-сервисов.

Основные методы:

`getForObject()` — GET-запрос и возврат только тела ответа.

```java
User user = restTemplate.getForObject(
    "http://localhost:8080/users/1",
    User.class
);
```

`getForEntity()` — GET-запрос и возврат полного HTTP-ответа.

```java
ResponseEntity<User> response = restTemplate.getForEntity(
    "http://localhost:8080/users/1",
    User.class
);
```

`postForObject()` — POST-запрос и возврат тела ответа.

```java
User created = restTemplate.postForObject(
    "http://localhost:8080/users",
    new User(null, "Ivan"),
    User.class
);
```

`postForEntity()` — POST-запрос и возврат полного ответа.

`put()` — PUT-запрос для обновления ресурса.

```java
restTemplate.put(
    "http://localhost:8080/users/1",
    new User(1L, "Updated name")
);
```

`delete()` — DELETE-запрос.

```java
restTemplate.delete("http://localhost:8080/users/1");
```

`exchange()` — универсальный метод, позволяющий указать HTTP-метод, заголовки, тело запроса и тип ответа.

```java
HttpHeaders headers = new HttpHeaders();
headers.setBearerAuth("token");

HttpEntity<Void> entity = new HttpEntity<>(headers);

ResponseEntity<String> response = restTemplate.exchange(
    "http://localhost:8080/users",
    HttpMethod.GET,
    entity,
    String.class
);
```

`patchForObject()` — PATCH-запрос, если клиент поддерживает данный метод.

Дополнительные методы:
- `headForHeaders()`;
- `optionsForAllow()`.

`RestTemplate` используется для интеграции с внешними сервисами, внутренними API и микросервисами.

## 15. HTTP-протокол

HTTP (*HyperText Transfer Protocol*) — это прикладной протокол обмена данными между клиентом и сервером. Он используется веб-приложениями, REST API, браузерами и backend-сервисами.

HTTP работает по модели `request-response`:
- клиент отправляет запрос;
- сервер обрабатывает запрос;
- сервер возвращает ответ.

HTTP является stateless-протоколом. Это означает, что каждый запрос рассматривается независимо. Для хранения состояния используются cookies, session, токены и другие механизмы.

Структура HTTP-запроса:
- метод;
- URI;
- версия протокола;
- заголовки;
- тело запроса.

Пример HTTP-запроса:

```http
POST /users HTTP/1.1
Host: localhost:8080
Content-Type: application/json

{
  "name": "Ivan"
}
```

Структура HTTP-ответа:
- статус-код;
- заголовки;
- тело ответа.

Пример HTTP-ответа:

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "id": 1,
  "name": "Ivan"
}
```

Основные HTTP-методы:
- `GET` — получение ресурса;
- `POST` — создание ресурса;
- `PUT` — полное обновление ресурса;
- `PATCH` — частичное обновление ресурса;
- `DELETE` — удаление ресурса;
- `HEAD` — получение только заголовков;
- `OPTIONS` — получение доступных методов.

Часто используемые статус-коды:
- `200 OK`;
- `201 Created`;
- `204 No Content`;
- `400 Bad Request`;
- `401 Unauthorized`;
- `403 Forbidden`;
- `404 Not Found`;
- `500 Internal Server Error`.

Примеры HTTP-заголовков:
- `Content-Type`;
- `Accept`;
- `Authorization`;
- `Cache-Control`;
- `Set-Cookie`.

HTTPS — это HTTP поверх TLS/SSL, то есть HTTP с шифрованием канала связи.

Пример контроллера, работающего поверх HTTP:

```java
@RestController
@RequestMapping("/api/books")
public class BookController {

    @GetMapping("/{id}")
    public ResponseEntity<Book> getBook(@PathVariable Long id) {
        Book book = bookService.findById(id);
        return ResponseEntity.ok(book);
    }

    @PostMapping
    public ResponseEntity<Book> createBook(@RequestBody Book book) {
        Book saved = bookService.save(book);
        return ResponseEntity.status(HttpStatus.CREATED).body(saved);
    }
}
```

HTTP-протокол является основой работы Spring MVC и REST-сервисов: запрос поступает в приложение, преобразуется в вызов метода контроллера, а результат метода формирует HTTP-ответ.
