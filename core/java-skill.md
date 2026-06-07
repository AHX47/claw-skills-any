# Java Skills — Read/Write Guide

## Reading Java Code
Identify: `public static void main(String[] args)` entry point, Spring Boot `@SpringBootApplication`, class hierarchies via `extends`/`implements`, annotations (`@Override`, `@Autowired`, `@RestController`).

## Modern Java (17+) Writing Standards
```java
// Records — immutable data carriers
record User(Long id, String name, String email) {
    User { // compact constructor — validation
        Objects.requireNonNull(name, "name required");
        if (email == null || !email.contains("@"))
            throw new IllegalArgumentException("Invalid email");
    }
}

// Sealed classes
sealed interface Shape permits Circle, Rectangle {}
record Circle(double radius) implements Shape {}
record Rectangle(double width, double height) implements Shape {}

// Pattern matching
double area(Shape s) {
    return switch (s) {
        case Circle c       -> Math.PI * c.radius() * c.radius();
        case Rectangle r    -> r.width() * r.height();
    };
}
```

## Collections & Streams
```java
import java.util.*;
import java.util.stream.*;

List<String> names = List.of("Ali","Mohamed","Fatima","Ahmed");

// Filter, map, collect
List<String> filtered = names.stream()
    .filter(n -> n.length() > 3)
    .map(String::toUpperCase)
    .sorted()
    .collect(Collectors.toList());

// Group by
Map<Integer, List<String>> byLength = names.stream()
    .collect(Collectors.groupingBy(String::length));

// Statistics
IntSummaryStatistics stats = names.stream()
    .mapToInt(String::length)
    .summaryStatistics();
System.out.println("Max length: " + stats.getMax());
```

## File I/O (NIO.2)
```java
import java.nio.file.*;
import java.nio.charset.StandardCharsets;

// Read
String content = Files.readString(Path.of("file.txt"), StandardCharsets.UTF_8);
List<String> lines = Files.readAllLines(Path.of("file.txt"));

// Write
Files.writeString(Path.of("out.txt"), "Hello", StandardCharsets.UTF_8);
Files.write(Path.of("out.txt"), lines, StandardCharsets.UTF_8,
            StandardOpenOption.CREATE, StandardOpenOption.APPEND);

// Walk directory
try (Stream<Path> walk = Files.walk(Path.of("src"))) {
    walk.filter(p -> p.toString().endsWith(".java"))
        .forEach(System.out::println);
}

// JSON with Jackson
ObjectMapper mapper = new ObjectMapper();
User user = mapper.readValue(Path.of("user.json").toFile(), User.class);
mapper.writeValue(Path.of("out.json").toFile(), user);
```

## Optional — Null Safety
```java
Optional<User> findUser(Long id) {
    return Optional.ofNullable(userMap.get(id));
}

// Usage — never use .get() without check
findUser(42L)
    .map(User::name)
    .filter(name -> !name.isEmpty())
    .ifPresentOrElse(
        name -> System.out.println("Found: " + name),
        ()   -> System.out.println("Not found")
    );
```

## Spring Boot REST API
```java
@RestController
@RequestMapping("/api/users")
@RequiredArgsConstructor
public class UserController {

    private final UserService userService;

    @GetMapping("/{id}")
    public ResponseEntity<UserDto> getUser(@PathVariable Long id) {
        return userService.findById(id)
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public UserDto createUser(@Valid @RequestBody CreateUserRequest req) {
        return userService.create(req);
    }

    @ExceptionHandler(UserNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public Map<String, String> handleNotFound(UserNotFoundException e) {
        return Map.of("error", e.getMessage());
    }
}
```

## Concurrency
```java
import java.util.concurrent.*;

// Virtual threads (Java 21)
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    List<Future<String>> futures = urls.stream()
        .map(url -> executor.submit(() -> fetch(url)))
        .toList();
    futures.forEach(f -> System.out.println(f.get()));
}

// CompletableFuture
CompletableFuture<User> future = CompletableFuture
    .supplyAsync(() -> fetchUser(id))
    .thenApply(u -> enrichUser(u))
    .exceptionally(ex -> User.empty());
```

## Exception Handling
```java
// Custom exceptions
public class AppException extends RuntimeException {
    private final int statusCode;
    public AppException(String msg, int code) {
        super(msg); this.statusCode = code;
    }
}

// Try-with-resources
try (var conn = dataSource.getConnection();
     var stmt = conn.prepareStatement("SELECT * FROM users WHERE id=?")) {
    stmt.setLong(1, userId);
    try (var rs = stmt.executeQuery()) {
        if (rs.next()) return mapRow(rs);
    }
} catch (SQLException e) {
    throw new AppException("DB error: " + e.getMessage(), 500);
}
```

## Testing (JUnit 5 + Mockito)
```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {

    @Mock UserRepository repo;
    @InjectMocks UserService service;

    @Test
    void findById_returnsUser_whenExists() {
        var user = new User(1L, "Ali", "ali@test.com");
        when(repo.findById(1L)).thenReturn(Optional.of(user));

        var result = service.findById(1L);

        assertThat(result).isPresent();
        assertThat(result.get().name()).isEqualTo("Ali");
        verify(repo, times(1)).findById(1L);
    }

    @ParameterizedTest
    @ValueSource(strings = {"", " ", "invalid"})
    void createUser_throws_whenEmailInvalid(String email) {
        assertThrows(AppException.class,
            () -> service.create(new CreateUserRequest("Ali", email)));
    }
}
```
