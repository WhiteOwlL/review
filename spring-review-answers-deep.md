# Spring Review Answers Deep

## Содержание

1. Spring Security
2. Аутентификация и авторизация
3. Principal, Authorities, Authentication
4. InMemoryAuthentication и Basic Authentication
5. Способы добавить security к контроллеру
6. Связи `one-to-one` и `many-to-many`
7. Каскады в JPA/Hibernate
8. Bootstrap
9. REST-сервисы
10. Форматы данных в REST
11. `@RequestBody`, `@ResponseBody`, `ResponseEntity`
12. AJAX и `fetch`
13. `@RestController` и `@Controller`
14. `RestTemplate`
15. HTTP-протокол

---

## 1. Spring Security

Spring Security — это модуль Spring Framework, который отвечает за безопасность приложения. Он обеспечивает аутентификацию, авторизацию, защиту HTTP-эндпоинтов, защиту методов, управление контекстом безопасности, работу с сессией, logout, защиту от CSRF, интеграцию с разными механизмами входа и возможность подключения внешних identity-провайдеров.

Spring Security не является просто набором аннотаций. Это полноценная инфраструктура, встроенная в жизненный цикл HTTP-запроса.

### Основные задачи Spring Security

- идентификация пользователя;
- проверка учетных данных;
- выдача и хранение сведений о правах пользователя;
- ограничение доступа к URL;
- ограничение доступа к методам;
- управление входом и выходом из системы;
- защита от типовых веб-угроз;
- централизованное описание правил безопасности.

### Архитектурная идея

Spring Security работает как цепочка servlet-фильтров. HTTP-запрос сначала попадает не в контроллер, а проходит через `FilterChain`, внутри которой находятся security-фильтры.

В типовом сценарии происходит следующее:

1. Запрос попадает в security filter chain.
2. Из запроса извлекаются учетные данные или токен.
3. Создаётся объект `Authentication`.
4. `AuthenticationManager` передаёт его подходящему `AuthenticationProvider`.
5. Провайдер выполняет проверку пользователя.
6. При успехе формируется аутентифицированный `Authentication`.
7. Объект помещается в `SecurityContext`.
8. Затем выполняется проверка прав доступа.
9. Только после этого управление передаётся контроллеру.

### Ключевые компоненты

`SecurityFilterChain`  
Описывает правила безопасности для HTTP-запросов.

`AuthenticationManager`  
Оркестратор аутентификации. Передаёт объект `Authentication` провайдерам.

`AuthenticationProvider`  
Проверяет конкретный тип аутентификации. Например, логин/пароль, токен, кастомную схему.

`UserDetailsService`  
Интерфейс загрузки пользователя по username.

`PasswordEncoder`  
Отвечает за безопасное хранение и сравнение паролей.

`SecurityContextHolder`  
Хранит `SecurityContext`, а внутри него — текущий `Authentication`.

### Что обычно настраивают в приложении

- какие URL доступны без входа;
- какие URL доступны только аутентифицированным пользователям;
- какие URL требуют конкретных ролей;
- form login или HTTP Basic;
- stateless или session-based режим;
- CSRF;
- CORS;
- кастомную страницу логина;
- logout;
- обработку `401` и `403`.

### Пример конфигурации

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
                .requestMatchers(HttpMethod.GET, "/articles/**").hasAnyRole("USER", "ADMIN")
                .anyRequest().authenticated()
            )
            .formLogin(Customizer.withDefaults())
            .logout(logout -> logout.logoutUrl("/logout"))
            .httpBasic(Customizer.withDefaults());

        return http.build();
    }
}
```

### Безопасность на уровне HTTP и на уровне методов

Spring Security умеет работать на двух уровнях:

- на уровне маршрутов;
- на уровне методов.

URL-защита удобна для описания внешнего интерфейса приложения. Method security полезна там, где требуется защита бизнес-операций независимо от того, каким способом они вызываются.

### Пример method security

```java
@Configuration
@EnableMethodSecurity
public class MethodSecurityConfig {
}
```

```java
@Service
public class UserService {

    @PreAuthorize("hasRole('ADMIN')")
    public void deleteUser(Long id) {
        // delete logic
    }
}
```

### Session-based и stateless режим

Spring Security может работать в двух основных моделях:

`session-based`  
После логина состояние пользователя хранится в HTTP-сессии. Следующие запросы обрабатываются на основе уже существующей серверной сессии.

`stateless`  
Сервер не хранит состояние. Каждый запрос должен нести все данные для аутентификации, чаще всего в виде JWT.

Пример stateless-настройки:

```java
http
    .sessionManagement(session -> session.sessionCreationPolicy(SessionCreationPolicy.STATELESS));
```

### CSRF

CSRF — это атака, при которой злоумышленник заставляет браузер пользователя отправить запрос от его имени.

Spring Security включает CSRF-защиту по умолчанию для stateful-веб-приложений.

Для REST API без cookie-based session CSRF часто отключают:

```java
http.csrf(csrf -> csrf.disable());
```

### CORS

CORS нужен, если frontend и backend находятся на разных origin.

Пример:

```java
@Bean
public CorsConfigurationSource corsConfigurationSource() {
    CorsConfiguration configuration = new CorsConfiguration();
    configuration.setAllowedOrigins(List.of("http://localhost:3000"));
    configuration.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE"));
    configuration.setAllowedHeaders(List.of("*"));

    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/**", configuration);
    return source;
}
```

### PasswordEncoder

Пароли нельзя хранить в открытом виде. Spring Security предоставляет `PasswordEncoder`.

Пример:

```java
@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder();
}
```

Пример использования:

```java
String encoded = passwordEncoder.encode("myPassword");
boolean matches = passwordEncoder.matches("myPassword", encoded);
```

### Часто используемые аннотации

- `@EnableWebSecurity`
- `@EnableMethodSecurity`
- `@PreAuthorize`
- `@PostAuthorize`
- `@Secured`
- `@RolesAllowed`
- `@AuthenticationPrincipal`

### Частые ошибки

- хранение пароля в открытом виде;
- смешение авторизации и аутентификации;
- защита только контроллера без защиты сервисного слоя;
- бездумное отключение CSRF;
- `hasRole("ADMIN")` и `hasAuthority("ADMIN")` используются как одно и то же, хотя семантика различается.

### Краткий вывод

Spring Security — это инфраструктура, которая позволяет централизованно описывать и применять правила безопасности на уровне HTTP-запросов и бизнес-методов.

---

## 2. Что такое авторизация, аутентификация

Аутентификация и авторизация — это два разных этапа обеспечения доступа в систему.

### Аутентификация

Аутентификация — это проверка личности пользователя. Она отвечает на вопрос: кто именно обращается к системе.

Способы аутентификации:

- логин и пароль;
- токен;
- OAuth2;
- сертификат;
- одноразовый код;
- внешний identity provider.

### Авторизация

Авторизация — это проверка прав аутентифицированного пользователя. Она отвечает на вопрос: что именно ему разрешено делать.

Примеры:

- имеет ли пользователь право удалить запись;
- может ли пользователь открыть страницу администратора;
- доступен ли ему определённый endpoint;
- имеет ли он право редактировать сущность другого пользователя.

### Различие

Аутентификация подтверждает личность.  
Авторизация определяет полномочия.

### Жизненный пример

Если пользователь ввёл корректный логин и пароль, он успешно аутентифицирован.  
Если после этого система позволяет ему открывать `/admin/**`, значит авторизация для этого ресурса у него есть.  
Если доступа к `/admin/**` нет, аутентификация могла быть успешной, но авторизация — отрицательной.

### Разница в HTTP-кодах

- `401 Unauthorized` в практическом смысле обычно означает отсутствие или недействительность аутентификации;
- `403 Forbidden` означает, что пользователь установлен, но права доступа недостаточны.

### Пример в коде

```java
@RestController
@RequestMapping("/orders")
public class OrderController {

    @GetMapping("/{id}")
    @PreAuthorize("hasAnyRole('USER', 'ADMIN')")
    public Order getOrder(@PathVariable Long id) {
        return orderService.findById(id);
    }

    @DeleteMapping("/{id}")
    @PreAuthorize("hasRole('ADMIN')")
    public void deleteOrder(@PathVariable Long id) {
        orderService.deleteById(id);
    }
}
```

### Формулировка для ревью

Аутентификация — это установление личности пользователя.  
Авторизация — это проверка его прав доступа к ресурсам и операциям системы.

---

## 3. Principal, Authorities, Authentication

Эти объекты являются базовыми элементами модели безопасности Spring Security.

### Principal

`Principal` — это представление текущего пользователя.

На практике principal может быть:

- строковым username;
- объектом `UserDetails`;
- кастомным доменным объектом;
- объектом, извлечённым из JWT или OAuth2.

Пример:

```java
@GetMapping("/me")
public String me(Principal principal) {
    return principal.getName();
}
```

### Authorities

`Authorities` — это список полномочий пользователя.

Они описываются интерфейсом `GrantedAuthority`.

Примеры authorities:

- `ROLE_USER`
- `ROLE_ADMIN`
- `READ_ORDERS`
- `WRITE_ORDERS`
- `DELETE_USERS`

Пример чтения authorities:

```java
@GetMapping("/authorities")
public List<String> authorities(Authentication authentication) {
    return authentication.getAuthorities()
        .stream()
        .map(GrantedAuthority::getAuthority)
        .toList();
}
```

### Authentication

`Authentication` — центральный объект Spring Security, описывающий текущую аутентификацию.

Внутри него содержатся:

- `principal`
- `credentials`
- `authorities`
- `details`
- `authenticated`

Пример получения `Authentication`:

```java
@GetMapping("/profile")
public String profile(Authentication authentication) {
    return authentication.getName();
}
```

### Где хранится Authentication

Текущий объект аутентификации хранится в `SecurityContext`, а сам контекст доступен через `SecurityContextHolder`.

Пример:

```java
Authentication authentication = SecurityContextHolder
    .getContext()
    .getAuthentication();
```

### Пример кастомного principal

```java
public class CustomUserDetails implements UserDetails {

    private final Long id;
    private final String username;
    private final String password;
    private final Collection<? extends GrantedAuthority> authorities;

    public CustomUserDetails(Long id, String username, String password,
                             Collection<? extends GrantedAuthority> authorities) {
        this.id = id;
        this.username = username;
        this.password = password;
        this.authorities = authorities;
    }

    public Long getId() {
        return id;
    }

    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        return authorities;
    }

    @Override
    public String getPassword() {
        return password;
    }

    @Override
    public String getUsername() {
        return username;
    }

    @Override
    public boolean isAccountNonExpired() {
        return true;
    }

    @Override
    public boolean isAccountNonLocked() {
        return true;
    }

    @Override
    public boolean isCredentialsNonExpired() {
        return true;
    }

    @Override
    public boolean isEnabled() {
        return true;
    }
}
```

Использование:

```java
@GetMapping("/user-id")
public Long currentUserId(@AuthenticationPrincipal CustomUserDetails userDetails) {
    return userDetails.getId();
}
```

### Дополнительный нюанс про роли

Когда используется `hasRole("ADMIN")`, Spring Security ожидает authority вида `ROLE_ADMIN`.

Пример:

```java
new SimpleGrantedAuthority("ROLE_ADMIN");
```

Если используется `hasAuthority("ADMIN")`, тогда нужен authority строго `"ADMIN"`.

### Краткий вывод

`Authentication` — основной объект безопасности.  
`Principal` — пользователь.  
`Authorities` — его полномочия.

---

## 4. Чем отличается InMemoryAuthentication от Basic Authentication

Эти понятия относятся к разным уровням security-конфигурации.

### InMemoryAuthentication

`InMemoryAuthentication` — это способ хранения пользователей внутри приложения, в памяти.

Обычно используется:

- в демо-проектах;
- в учебных приложениях;
- в интеграционных тестах;
- в небольших внутренних сервисах.

Пример:

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

Недостатки:

- пользователи не хранятся постоянно;
- невозможно удобно управлять ими извне;
- не подходит для production-сценариев с реальными пользователями;
- неудобен для масштабирования.

### Basic Authentication

HTTP Basic Authentication — это способ передачи учетных данных через HTTP-заголовок `Authorization`.

Структура заголовка:

```http
Authorization: Basic base64(username:password)
```

Пример включения:

```java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    http
        .authorizeHttpRequests(auth -> auth.anyRequest().authenticated())
        .httpBasic(Customizer.withDefaults());

    return http.build();
}
```

### Ключевое различие

- `InMemoryAuthentication` отвечает за источник пользователей;
- `Basic Authentication` отвечает за способ передачи логина и пароля.

### Как они могут работать вместе

```java
@Configuration
public class SecurityConfig {

    @Bean
    public UserDetailsService userDetailsService() {
        UserDetails user = User.withUsername("user")
            .password("{noop}123")
            .roles("USER")
            .build();

        return new InMemoryUserDetailsManager(user);
    }

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth.anyRequest().authenticated())
            .httpBasic(Customizer.withDefaults());

        return http.build();
    }
}
```

### Важный нюанс безопасности

Base64 не является шифрованием. Поэтому Basic Authentication допустимо использовать только поверх HTTPS.

### Краткий вывод

`InMemoryAuthentication` — это способ хранения пользователей.  
`Basic Authentication` — это схема аутентификации на уровне HTTP.

---

## 5. Как мы можем добавить секьюрность к контроллеру

Защитить контроллер можно несколькими способами.

### Способ 1. Ограничение доступа по URL через `SecurityFilterChain`

Пример:

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

Преимущества:

- централизованное описание правил;
- удобно для защиты всего внешнего API;
- удобно видеть картину доступа по маршрутам.

### Способ 2. Ограничение на уровне метода через `@PreAuthorize`

```java
@Configuration
@EnableMethodSecurity
public class MethodSecurityConfig {
}
```

```java
@RestController
@RequestMapping("/admin")
public class AdminController {

    @PreAuthorize("hasRole('ADMIN')")
    @GetMapping("/report")
    public String report() {
        return "report";
    }
}
```

Преимущества:

- защита располагается рядом с бизнес-операцией;
- можно использовать гибкие выражения;
- можно переносить защиту на сервисный слой.

### Способ 3. `@Secured`

```java
@Secured("ROLE_ADMIN")
public void deleteUser(Long id) {
}
```

`@Secured` проще, чем `@PreAuthorize`, но менее гибкая.

### Способ 4. `@RolesAllowed`

```java
@RolesAllowed("ADMIN")
public void generateReport() {
}
```

Эта аннотация относится к стандарту JSR-250.

### Способ 5. Проверка через `@AuthenticationPrincipal`

Иногда доступ зависит не только от роли, но и от личности пользователя.

```java
@GetMapping("/users/{id}")
public User getUser(@PathVariable Long id,
                    @AuthenticationPrincipal CustomUserDetails currentUser) {
    if (!currentUser.getId().equals(id)) {
        throw new AccessDeniedException("Access denied");
    }
    return userService.findById(id);
}
```

### Способ 6. Перенос защиты в сервис

```java
@Service
public class ReportService {

    @PreAuthorize("hasRole('ADMIN')")
    public Report generate() {
        return new Report();
    }
}
```

Это особенно полезно, если один и тот же сервис вызывается разными контроллерами или другими компонентами.

### Пример SpEL

```java
@PreAuthorize("#id == authentication.principal.id or hasRole('ADMIN')")
public User findUser(Long id) {
    return userService.findById(id);
}
```

### Краткий вывод

Минимум два базовых способа:

- через `SecurityFilterChain`;
- через method security (`@PreAuthorize`, `@Secured`, `@RolesAllowed`).

---

## 6. Связи таблиц many-to-many и one-to-one

В JPA связи между сущностями отражают структуру отношений в реляционной базе данных.

### One-to-one

Связь `one-to-one` означает, что одной записи одной таблицы соответствует одна запись другой таблицы.

Примеры:

- пользователь и профиль;
- человек и паспорт;
- заказ и платежная информация.

### Варианты реализации в БД

- внешний ключ + `UNIQUE`;
- общая первичная колонка;
- отдельная таблица с жёстким соответствием.

### Пример `@OneToOne`

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
    private String address;
}
```

### Двунаправленная one-to-one

```java
@Entity
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @OneToOne(mappedBy = "user", cascade = CascadeType.ALL)
    private Profile profile;
}
```

```java
@Entity
public class Profile {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @OneToOne
    @JoinColumn(name = "user_id")
    private User user;
}
```

Здесь владеющей стороной является `Profile`, потому что именно там указан `@JoinColumn`.

### Нюанс про fetch type

Для `@OneToOne` загрузка по умолчанию может быть `EAGER`, что часто нежелательно. В реальных проектах это бывает причиной лишних запросов и проблем с производительностью.

Пример:

```java
@OneToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "profile_id")
private Profile profile;
```

### Many-to-many

Связь `many-to-many` означает, что одной записи из первой таблицы соответствует множество записей из второй таблицы и наоборот.

Примеры:

- студент и курс;
- пользователь и роль;
- товар и категория.

### Реализация в БД

В реляционной модели для `many-to-many` нужна промежуточная таблица.

Например:

- `students`
- `courses`
- `student_course`

### Пример `@ManyToMany`

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

### Двунаправленная many-to-many

```java
@Entity
public class Course {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToMany(mappedBy = "courses")
    private Set<Student> students = new HashSet<>();
}
```

### Основные проблемы `many-to-many`

- трудно расширять модель;
- трудно добавлять поля в саму связь;
- неудобно управлять каскадами;
- повышается риск неожиданных операций удаления;
- сложнее контролировать SQL.

### Когда лучше не использовать `@ManyToMany`

Если у самой связи есть собственные атрибуты:

- дата назначения;
- статус;
- приоритет;
- автор назначения;
- срок действия.

Тогда лучше ввести отдельную сущность связи.

### Пример отдельной сущности связи

```java
@Entity
public class UserRole {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne
    @JoinColumn(name = "user_id")
    private User user;

    @ManyToOne
    @JoinColumn(name = "role_id")
    private Role role;

    private LocalDate assignedAt;
}
```

### Краткий вывод

`One-to-one` используется для строгого соответствия один к одному.  
`Many-to-many` реализуется через промежуточную таблицу и удобен только в простых случаях.  
Если связь обладает собственной семантикой, лучше использовать отдельную сущность связи.

---

## 7. Как работают каскады для таблиц и какие они бывают

Каскады в JPA/Hibernate определяют, какие операции над родительской сущностью автоматически распространяются на связанные сущности.

### Основные типы каскадов

- `PERSIST`
- `MERGE`
- `REMOVE`
- `REFRESH`
- `DETACH`
- `ALL`

### `PERSIST`

При сохранении родительской сущности сохраняются связанные дочерние сущности.

Пример:

```java
@OneToMany(mappedBy = "order", cascade = CascadeType.PERSIST)
private List<OrderItem> items;
```

### `MERGE`

Если родительский объект мержится в persistence context, связанные сущности тоже будут merge-иться.

### `REMOVE`

При удалении родителя будут удалены и дочерние сущности.

### `REFRESH`

Состояние сущности и связанных сущностей обновляется из базы данных.

### `DETACH`

Сущность и её связи отсоединяются от текущего persistence context.

### `ALL`

Комбинирует все перечисленные типы.

### Базовый пример

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

### Пример сохранения

```java
Order order = new Order();
OrderItem item = new OrderItem();

item.setOrder(order);
order.getItems().add(item);

orderRepository.save(order);
```

Если настроен `CascadeType.PERSIST` или `CascadeType.ALL`, `OrderItem` сохранится вместе с `Order`.

### `orphanRemoval`

`orphanRemoval = true` — это не то же самое, что `CascadeType.REMOVE`.

Разница:

- `REMOVE` срабатывает при удалении родителя;
- `orphanRemoval = true` срабатывает, когда дочерний объект убирают из коллекции и он перестаёт быть связан с родителем.

Пример:

```java
order.getItems().remove(item);
```

Если `orphanRemoval = true`, объект `item` будет удалён из БД.

### Каскады ORM и каскады БД

Нужно различать:

- каскады JPA/Hibernate;
- каскадные действия самой базы данных (`ON DELETE CASCADE`).

JPA-каскады управляют графом объектов в приложении.  
DB-cascade управляет действиями на уровне самой базы.

Это разные механизмы.

### Когда каскады уместны

Когда дочерние объекты логически принадлежат родителю:

- заказ и позиции;
- корзина и элементы корзины;
- документ и строки документа.

### Когда каскады опасны

- роли и пользователи;
- справочники;
- общие сущности, используемые в разных местах;
- `@ManyToMany` без полного понимания последствий.

### Краткий вывод

Каскады нужны для автоматического распространения операций на связанные сущности, но применять их нужно только там, где действительно существует отношение владения.

---

## 8. Bootstrap

Bootstrap — это frontend-фреймворк для быстрой разработки интерфейсов. Он предоставляет готовую CSS-сетку, набор UI-компонентов, утилитарные классы и JavaScript-поведение для типовых элементов.

### Что включает Bootstrap

- grid system;
- кнопки;
- формы;
- таблицы;
- карточки;
- навигацию;
- модальные окна;
- alerts;
- dropdown;
- collapse;
- utilities.

### Для чего используется

- быстрое прототипирование;
- административные панели;
- корпоративные приложения;
- учебные проекты;
- интерфейсы с предсказуемым поведением.

### Пример подключения через CDN

```html
<!doctype html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <link
    href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/css/bootstrap.min.css"
    rel="stylesheet">
  <title>Bootstrap Demo</title>
</head>
<body>
  <div class="container mt-4">
    <h1 class="mb-3">Hello Bootstrap</h1>
    <button class="btn btn-primary">Save</button>
  </div>
</body>
</html>
```

### Grid system

Bootstrap использует 12-колоночную сетку.

Пример:

```html
<div class="container">
  <div class="row">
    <div class="col-md-4">Column 1</div>
    <div class="col-md-4">Column 2</div>
    <div class="col-md-4">Column 3</div>
  </div>
</div>
```

### Пример формы

```html
<form class="container mt-4">
  <div class="mb-3">
    <label for="email" class="form-label">Email</label>
    <input type="email" class="form-control" id="email">
  </div>
  <div class="mb-3">
    <label for="password" class="form-label">Password</label>
    <input type="password" class="form-control" id="password">
  </div>
  <button type="submit" class="btn btn-success">Submit</button>
</form>
```

### Преимущества

- высокая скорость разработки;
- готовые визуальные компоненты;
- адаптивность по умолчанию;
- единообразный стиль;
- хорошая документация;
- понятный входной порог.

### Недостатки

- типовой внешний вид;
- сложнее создать уникальный дизайн;
- возможен избыток классов в HTML;
- при сложной кастомизации может конфликтовать с собственной дизайн-системой.

### Краткий вывод

Bootstrap — удобный инструмент для быстрой адаптивной вёрстки и сборки типовых интерфейсов.

---

## 9. REST-сервисы. Их преимущества и недостатки

REST (*Representational State Transfer*) — это архитектурный стиль построения веб-сервисов, основанный на работе с ресурсами через HTTP.

### Основная идея

В REST каждая сущность рассматривается как ресурс, которому соответствует URI.

Примеры:

- `/users`
- `/users/10`
- `/orders`
- `/orders/15/items`

### Основные ограничения REST

Классический REST включает ряд архитектурных ограничений:

- client-server;
- stateless;
- cacheable;
- uniform interface;
- layered system;
- code on demand как необязательное ограничение.

### Что означает stateless

Сервер не должен хранить контекст клиента между запросами. Каждый запрос должен содержать всю необходимую информацию.

### Стандартные HTTP-методы в REST

- `GET` — получение ресурса;
- `POST` — создание ресурса;
- `PUT` — полная замена ресурса;
- `PATCH` — частичное обновление;
- `DELETE` — удаление.

### Пример REST-контроллера

```java
@RestController
@RequestMapping("/users")
public class UserController {

    @GetMapping
    public List<User> findAll() {
        return userService.findAll();
    }

    @GetMapping("/{id}")
    public ResponseEntity<User> findById(@PathVariable Long id) {
        User user = userService.findById(id);
        if (user == null) {
            return ResponseEntity.notFound().build();
        }
        return ResponseEntity.ok(user);
    }

    @PostMapping
    public ResponseEntity<User> create(@RequestBody User user) {
        User saved = userService.save(user);
        return ResponseEntity.status(HttpStatus.CREATED).body(saved);
    }

    @PutMapping("/{id}")
    public User update(@PathVariable Long id, @RequestBody User user) {
        return userService.update(id, user);
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> delete(@PathVariable Long id) {
        userService.deleteById(id);
        return ResponseEntity.noContent().build();
    }
}
```

### Преимущества REST

- простая модель взаимодействия;
- опора на стандартный HTTP;
- понятная адресация ресурсов;
- независимость клиента и сервера;
- удобство интеграции разных платформ;
- хорошая масштабируемость;
- поддержка кэширования;
- предсказуемость контрактов.

### Недостатки REST

- не всегда удобно получать сложные агрегированные данные;
- возможна избыточность ответов;
- иногда требуется несколько запросов вместо одного;
- не существует одного строгого формата ответа;
- отдельной задачей становится версионирование.

### Пример плохого и хорошего URI

Нежелательно:

- `/getAllUsers`
- `/deleteUserById/10`

Предпочтительно:

- `GET /users`
- `DELETE /users/10`

### Идемпотентность

Для REST и HTTP важно понимать свойства методов:

- `GET` — безопасный и идемпотентный;
- `PUT` — идемпотентный;
- `DELETE` — в общем случае идемпотентный;
- `POST` — неидемпотентный.

### HATEOAS

В строгом понимании REST может включать HATEOAS, когда сервер возвращает не только данные, но и ссылки на допустимые дальнейшие действия. На практике многие REST API ограничиваются JSON-ответами без HATEOAS.

### Краткий вывод

REST — это ресурсно-ориентированный способ построения HTTP API, хорошо подходящий для веб-приложений, интеграций и микросервисной архитектуры.

---

## 10. Форматы данных, использующиеся в REST-сервисах

REST не привязан к одному конкретному формату данных. Формат определяется отдельно через HTTP-заголовки и механизмы content negotiation.

### Наиболее распространённые форматы

- JSON;
- XML;
- plain text;
- HTML;
- binary;
- CSV;
- иногда YAML.

### JSON

Наиболее популярный формат в современных REST API.

Преимущества:

- компактность;
- читаемость;
- удобство для JavaScript;
- хорошая поддержка в Spring через Jackson;
- простой маппинг на DTO.

Пример:

```json
{
  "id": 1,
  "name": "Ivan",
  "role": "USER"
}
```

### XML

Используется в корпоративных системах, старых интеграциях, SOAP-окружениях и некоторых банковских интерфейсах.

Преимущества:

- строгая структура;
- развитая экосистема схем;
- гибкость при валидации и стандартизации.

Недостатки:

- многословность;
- больший объём;
- более сложный парсинг.

Пример:

```xml
<user>
    <id>1</id>
    <name>Ivan</name>
    <role>USER</role>
</user>
```

### Plain text

Используется редко, но может применяться для простых ответов:

```java
@GetMapping("/ping")
public String ping() {
    return "OK";
}
```

### Binary data

REST может передавать файлы:

- изображения;
- PDF;
- архивы;
- документы.

Пример:

```java
@GetMapping("/file")
public ResponseEntity<byte[]> download() {
    byte[] data = ...;
    return ResponseEntity.ok()
        .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=test.pdf")
        .body(data);
}
```

### Content Negotiation

Content negotiation — это выбор формата ответа на основе заголовков запроса.

Используются заголовки:

- `Content-Type`
- `Accept`

Пример:

```http
Content-Type: application/json
Accept: application/json
```

### Как это работает в Spring

Spring использует `HttpMessageConverter`.

Если в приложении есть Jackson, JSON обычно обрабатывается автоматически.  
Если подключены XML-конвертеры, можно отдавать XML.

### Пример контроллера

```java
@GetMapping(value = "/users/{id}", produces = MediaType.APPLICATION_JSON_VALUE)
public User getUser(@PathVariable Long id) {
    return userService.findById(id);
}
```

### DTO и формат ответа

Часто рекомендуется отдавать не сущности JPA напрямую, а DTO. Это позволяет:

- не раскрывать внутреннюю модель;
- контролировать поля;
- избегать циклических ссылок;
- формировать стабильный контракт API.

### Краткий вывод

Формат в REST определяется отдельно от архитектурного стиля. На практике доминирует JSON, но возможны XML, текст и бинарные данные.

---

## 11. Что такое `@ResponseBody`, `@RequestBody`, `ResponseEntity`

Эти элементы относятся к механизму обработки HTTP-тела в Spring MVC / Spring REST.

### `@RequestBody`

`@RequestBody` используется для чтения тела HTTP-запроса и преобразования его в Java-объект.

Пример:

```java
@PostMapping("/users")
public User create(@RequestBody User user) {
    return userService.save(user);
}
```

Если клиент отправил JSON, Spring через `HttpMessageConverter` преобразует его в объект `User`.

### `@RequestBody` + валидация

```java
@PostMapping("/users")
public ResponseEntity<UserDto> create(@Valid @RequestBody UserDto dto) {
    UserDto saved = userService.save(dto);
    return ResponseEntity.status(HttpStatus.CREATED).body(saved);
}
```

### DTO-пример

```java
public class UserDto {

    @NotBlank
    private String name;

    @Email
    private String email;

    // getters/setters
}
```

### `@ResponseBody`

`@ResponseBody` указывает, что возвращаемое значение метода нужно записать в тело HTTP-ответа, а не интерпретировать как имя представления.

Пример:

```java
@GetMapping("/hello")
@ResponseBody
public String hello() {
    return "Hello";
}
```

### Пример возврата объекта

```java
@GetMapping("/users/{id}")
@ResponseBody
public User getUser(@PathVariable Long id) {
    return userService.findById(id);
}
```

### `ResponseEntity`

`ResponseEntity` — это полный HTTP-ответ, представленный объектом Java.

Он позволяет управлять:

- статусом;
- заголовками;
- телом.

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

### Пример `201 Created`

```java
@PostMapping("/users")
public ResponseEntity<User> createUser(@RequestBody User user) {
    User saved = userService.save(user);
    return ResponseEntity.status(HttpStatus.CREATED).body(saved);
}
```

### Пример с заголовками

```java
@GetMapping("/download")
public ResponseEntity<byte[]> download() {
    byte[] data = "file content".getBytes(StandardCharsets.UTF_8);

    return ResponseEntity.ok()
        .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=file.txt")
        .contentType(MediaType.TEXT_PLAIN)
        .body(data);
}
```

### Когда использовать `ResponseEntity`

- когда нужен нестандартный HTTP-статус;
- когда нужно вернуть `404`, `204`, `201`;
- когда нужно добавить заголовки;
- когда требуется полный контроль над ответом.

### Краткий вывод

- `@RequestBody` читает тело запроса;
- `@ResponseBody` формирует тело ответа;
- `ResponseEntity` даёт полный контроль над HTTP-ответом.

---

## 12. Что такое AJAX / fetch

AJAX — это подход к асинхронному взаимодействию клиента и сервера без полной перезагрузки страницы.

### Расшифровка

AJAX — `Asynchronous JavaScript and XML`.

Несмотря на название, сегодня в AJAX чаще используется JSON, а не XML.

### Что даёт AJAX

- обновление части страницы без reload;
- асинхронную отправку форм;
- загрузку данных по требованию;
- улучшение пользовательского интерфейса;
- возможность работы SPA или гибридных веб-интерфейсов.

### Исторический механизм

Ранее AJAX чаще всего реализовывался через `XMLHttpRequest`.

### Современный подход — `fetch`

`fetch` — встроенный браузерный API для выполнения HTTP-запросов.

Пример GET:

```javascript
fetch('/api/users')
  .then(response => response.json())
  .then(data => console.log(data));
```

### Пример POST

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

### Пример с `async/await`

```javascript
async function loadUsers() {
  const response = await fetch('/api/users');
  const users = await response.json();
  console.log(users);
}
```

### Обработка ошибок

```javascript
async function createUser() {
  const response = await fetch('/api/users', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ name: 'Ivan' })
  });

  if (!response.ok) {
    throw new Error('Request failed');
  }

  return response.json();
}
```

### Что нужно учитывать

- CORS;
- `Content-Type`;
- обработку статусов `4xx` и `5xx`;
- сериализацию и десериализацию JSON;
- авторизацию через заголовки.

### Пример запроса с токеном

```javascript
fetch('/api/profile', {
  headers: {
    'Authorization': 'Bearer token-value'
  }
});
```

### Краткий вывод

AJAX обеспечивает асинхронное обновление интерфейса, а `fetch` является стандартным JavaScript-инструментом для такой работы.

---

## 13. Чем аннотация `@RestController` отличается от `@Controller`

Обе аннотации относятся к Spring MVC, но используются в разных сценариях.

### `@Controller`

`@Controller` — базовая аннотация для контроллеров, которые обрабатывают HTTP-запросы и, как правило, возвращают представления.

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

Здесь строка `"home"` — это имя view.  
Далее Spring ищет шаблон `home.html`, `home.jsp` или другое представление в зависимости от настроек.

### `@RestController`

`@RestController` — это комбинация:

- `@Controller`
- `@ResponseBody`

Все методы класса автоматически возвращают данные в тело HTTP-ответа.

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

В этом примере список пользователей будет сериализован в JSON.

### Когда использовать `@Controller`

- серверная HTML-страница;
- Thymeleaf;
- JSP;
- MVC-приложение;
- возврат имени представления.

### Когда использовать `@RestController`

- REST API;
- JSON/XML ответы;
- backend для frontend;
- мобильный клиент;
- внешнее API.

### Комбинирование

В одном приложении могут одновременно быть и `@Controller`, и `@RestController`.

Например:

- административная HTML-панель на `@Controller`;
- REST API для frontend на `@RestController`.

### Пример `@Controller` + `@ResponseBody`

```java
@Controller
public class MixedController {

    @GetMapping("/page")
    public String page() {
        return "page";
    }

    @GetMapping("/api/ping")
    @ResponseBody
    public String ping() {
        return "pong";
    }
}
```

### Краткий вывод

`@Controller` ориентирован на представления.  
`@RestController` ориентирован на возврат данных.

---

## 14. RestTemplate и его методы

`RestTemplate` — это синхронный HTTP-клиент Spring для вызова внешних сервисов.

### Для чего используется

- интеграция с внешними REST API;
- вызов внутренних микросервисов;
- обмен данными между backend-приложениями;
- загрузка внешних ресурсов.

### Основные методы

#### `getForObject()`

Возвращает только тело ответа.

```java
User user = restTemplate.getForObject(
    "http://localhost:8080/users/1",
    User.class
);
```

#### `getForEntity()`

Возвращает полный `ResponseEntity`.

```java
ResponseEntity<User> response = restTemplate.getForEntity(
    "http://localhost:8080/users/1",
    User.class
);
```

#### `postForObject()`

```java
User created = restTemplate.postForObject(
    "http://localhost:8080/users",
    new User(null, "Ivan"),
    User.class
);
```

#### `postForEntity()`

```java
ResponseEntity<User> response = restTemplate.postForEntity(
    "http://localhost:8080/users",
    new User(null, "Ivan"),
    User.class
);
```

#### `put()`

```java
restTemplate.put(
    "http://localhost:8080/users/1",
    new User(1L, "Updated name")
);
```

#### `delete()`

```java
restTemplate.delete("http://localhost:8080/users/1");
```

#### `exchange()`

Самый универсальный метод.

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

#### `patchForObject()`

```java
String result = restTemplate.patchForObject(
    "http://localhost:8080/users/1",
    Map.of("name", "Patched"),
    String.class
);
```

### Дополнительные методы

- `headForHeaders()`
- `optionsForAllow()`

### Пример настройки bean

```java
@Configuration
public class RestTemplateConfig {

    @Bean
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
}
```

### Пример с `HttpEntity`

```java
HttpHeaders headers = new HttpHeaders();
headers.setContentType(MediaType.APPLICATION_JSON);

User user = new User(null, "Ivan");
HttpEntity<User> request = new HttpEntity<>(user, headers);

ResponseEntity<User> response = restTemplate.exchange(
    "http://localhost:8080/users",
    HttpMethod.POST,
    request,
    User.class
);
```

### Обработка ошибок

По умолчанию `RestTemplate` использует `ResponseErrorHandler`.

Можно настроить собственный обработчик:

```java
RestTemplate restTemplate = new RestTemplate();
restTemplate.setErrorHandler(new DefaultResponseErrorHandler());
```

### Ограничения

- синхронная модель;
- поток блокируется на время запроса;
- для новых реактивных сценариев обычно предпочитают `WebClient`.

### Отличие от WebClient

`RestTemplate`:

- синхронный;
- проще для классических приложений;
- широко встречается в legacy-проектах.

`WebClient`:

- современный;
- поддерживает реактивный стиль;
- гибче для сложных сценариев.

### Краткий вывод

`RestTemplate` — это классический клиент Spring для выполнения HTTP-запросов к внешним сервисам.

---

## 15. HTTP-протокол

HTTP — это прикладной протокол обмена данными между клиентом и сервером.

### Базовая модель

Работа строится по схеме:

1. клиент отправляет запрос;
2. сервер его обрабатывает;
3. сервер возвращает ответ.

### Stateless

HTTP является stateless-протоколом. Это означает, что каждый запрос рассматривается независимо.

Если приложению нужно хранить пользовательское состояние, используются:

- cookies;
- session;
- JWT;
- внешние storage-решения.

### Структура HTTP-запроса

- метод;
- URI;
- версия протокола;
- заголовки;
- тело.

Пример:

```http
POST /users HTTP/1.1
Host: localhost:8080
Content-Type: application/json
Accept: application/json

{
  "name": "Ivan"
}
```

### Структура HTTP-ответа

- версия протокола;
- статус-код;
- reason phrase;
- заголовки;
- тело.

Пример:

```http
HTTP/1.1 201 Created
Content-Type: application/json

{
  "id": 1,
  "name": "Ivan"
}
```

### Основные методы

`GET`  
Получение ресурса.

`POST`  
Создание ресурса или выполнение операции.

`PUT`  
Полная замена ресурса.

`PATCH`  
Частичное обновление ресурса.

`DELETE`  
Удаление ресурса.

`HEAD`  
Получение только заголовков.

`OPTIONS`  
Получение информации о доступных методах.

### Safe и idempotent методы

`Safe` — метод не должен изменять состояние ресурса.

Безопасные методы:

- `GET`
- `HEAD`
- `OPTIONS`

`Idempotent` — повторное выполнение даёт тот же логический результат.

Идемпотентные методы:

- `GET`
- `PUT`
- `DELETE`
- `HEAD`
- `OPTIONS`

`POST` обычно неидемпотентный.

### Группы статус-кодов

`1xx` — информационные.  
`2xx` — успешные.  
`3xx` — перенаправления.  
`4xx` — ошибки клиента.  
`5xx` — ошибки сервера.

### Часто используемые статусы

- `200 OK`
- `201 Created`
- `204 No Content`
- `301 Moved Permanently`
- `302 Found`
- `400 Bad Request`
- `401 Unauthorized`
- `403 Forbidden`
- `404 Not Found`
- `405 Method Not Allowed`
- `409 Conflict`
- `415 Unsupported Media Type`
- `500 Internal Server Error`

### Заголовки

Примеры заголовков:

- `Content-Type`
- `Accept`
- `Authorization`
- `Cache-Control`
- `Set-Cookie`
- `Location`
- `Content-Length`
- `User-Agent`
- `Origin`

### Пример `Location`

После создания ресурса можно вернуть URI созданного объекта:

```java
@PostMapping("/users")
public ResponseEntity<User> create(@RequestBody User user) {
    User saved = userService.save(user);
    URI location = URI.create("/users/" + saved.getId());

    return ResponseEntity.created(location).body(saved);
}
```

### Кэширование

HTTP поддерживает кэширование через заголовки:

- `Cache-Control`
- `ETag`
- `Last-Modified`

### Cookies и Session

Cookie — это небольшой фрагмент данных, который сервер передаёт клиенту и который клиент отправляет обратно в следующих запросах.

Session — механизм хранения состояния пользователя на сервере, обычно связанный с session id в cookie.

### HTTPS

HTTPS — это HTTP поверх TLS/SSL.

Он обеспечивает:

- конфиденциальность;
- целостность;
- подлинность сервера.

### Версии протокола

- HTTP/1.1
- HTTP/2
- HTTP/3

HTTP/2 добавил multiplexing и улучшил производительность.  
HTTP/3 строится поверх QUIC.

### Пример контроллера, работающего по HTTP

```java
@RestController
@RequestMapping("/api/books")
public class BookController {

    @GetMapping("/{id}")
    public ResponseEntity<Book> getBook(@PathVariable Long id) {
        Book book = bookService.findById(id);
        if (book == null) {
            return ResponseEntity.notFound().build();
        }
        return ResponseEntity.ok(book);
    }

    @PostMapping
    public ResponseEntity<Book> createBook(@RequestBody Book book) {
        Book saved = bookService.save(book);
        return ResponseEntity.status(HttpStatus.CREATED).body(saved);
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteBook(@PathVariable Long id) {
        bookService.deleteById(id);
        return ResponseEntity.noContent().build();
    }
}
```

### Краткий вывод

HTTP — это основа веб-взаимодействия. Spring MVC и Spring REST строятся поверх HTTP: входящий запрос преобразуется в вызов метода контроллера, а результат выполнения формирует HTTP-ответ.

---

## Итог

Данный документ охватывает основные темы ревью:

- безопасность Spring-приложения;
- базовые сущности Spring Security;
- способы защиты контроллеров;
- связи и каскады в JPA;
- Bootstrap;
- REST и HTTP;
- механизмы работы контроллеров и REST-клиентов.

Для дальнейшего расширения этого конспекта наиболее естественно добавить:

- Spring Bean Lifecycle;
- IoC и DI;
- `@Component`, `@Service`, `@Repository`, `@Controller`;
- Spring MVC request lifecycle;
- exception handling через `@ControllerAdvice`;
- транзакции и `@Transactional`;
- Hibernate fetch strategies;
- lazy/eager loading;
- N+1 problem;
- DTO, mapper, validation;
- JWT flow;
- OAuth2 flow;
- CORS и CSRF в деталях;
- differences between `PUT` and `PATCH`;
- `RestTemplate` vs `WebClient`;
- типовые вопросы по `ResponseEntity` и обработке ошибок.
