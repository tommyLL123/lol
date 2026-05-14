# CS 2031 - DBP

## Desarrollo Basado en Plataformas

**Jorge Villavicencio**

UTEC — Reinventa el mundo

---

# Validaciones Comunes (Bean Validation)

```java
@NotNull        // No puede ser null
@NotBlank       // No puede ser null, vacío o solo espacios
@NotEmpty       // No puede ser null o vacío
@Size(min, max) // Tamaño de String o Collection
@Email          // Debe ser email válido
@Min / @Max     // Valores numéricos mínimos/máximos
@Pattern        // Regex personalizado
@Past / @Future // Validación de fechas
```

---

## Ejemplo: ProductoDTO

```java
public class ProductoDTO {

    // @DecimalMin / @DecimalMax: Valida el valor mínimo y máximo para tipos numéricos (incluido BigDecimal).
    @NotNull(message = "El precio es obligatorio")
    @DecimalMin(value = "0.01", message = "El precio debe ser positivo")
    @DecimalMax(value = "99999.99", message = "El precio es demasiado alto")
    private BigDecimal precio;

    // @Min / @Max: Valida el valor mínimo y máximo para tipos primitivos (int, long, etc.) y sus wrappers.
    @Min(value = 1, message = "El stock mínimo es 1")
    @Max(value = 1000, message = "El stock máximo es 1000")
    private int stock;

    // 3. Validación de formato de datos (email y expresiones regulares)

    // @Email: Valida que el String tenga un formato de email válido.
    @Email(message = "El formato del email es incorrecto")
    @NotBlank(message = "El email del proveedor es obligatorio")
    private String emailProveedor;

    // @Pattern: Permite definir una Expresión Regular personalizada. (Ver ejemplos abajo)
    @Pattern(regexp = "^\\d{3}-\\d{3}-\\d{4}$", message = "El código debe tener el formato XXX-XXX-XXXX")
    private String codigoProducto;

    // 4. Validación de fechas

    // @Past / @Future: Valida que un campo de fecha sea en el pasado o en el futuro.
    @Past(message = "La fecha de fabricación debe ser una fecha pasada")
    private java.time.LocalDate fechaFabricacion;

    // (Getters y Setters)
}
```

---

## ModelMapper

### Dependencia (Maven)

```xml
<dependency>
    <groupId>org.modelmapper</groupId>
    <artifactId>modelmapper</artifactId>
    <version>3.2.4</version>
</dependency>
```

### Configuración

```java
@Configuration
public class ModelMapperConfig {
    @Bean
    public ModelMapper modelMapper() {
        ModelMapper modelMapper = new ModelMapper();
        modelMapper.getConfiguration().setAmbiguityIgnored(true);
        return modelMapper;
    }
}
```

### Uso en un Service

```java
import org.modelmapper.ModelMapper;
import org.springframework.stereotype.Service;

@Service
public class UsuarioService {

    private final ModelMapper modelMapper;

    // Inyección de ModelMapper
    public UsuarioService(ModelMapper modelMapper) {
        this.modelMapper = modelMapper;
    }

    /**
     * Convierte el DTO a la Entidad antes de guardarla en la base de datos.
     * @param registroDTO El DTO de entrada.
     * @return La Entidad Usuario.
     */
    public Usuario convertirDtoAEntidad(UsuarioRegistroDTO registroDTO) {
        // Uso del método map: mapea propiedades con el mismo nombre
        Usuario usuario = modelMapper.map(registroDTO, Usuario.class);
        return usuario;
    }

    /**
     * Convierte la Entidad a un DTO de respuesta.
     * @param usuario La Entidad Usuario.
     * @return El DTO de salida.
     */
    public UsuarioResponseDTO convertirEntidadADto(Usuario usuario) {
        UsuarioResponseDTO responseDTO = modelMapper.map(usuario, UsuarioResponseDTO.class);
        return responseDTO;
    }
}
```

---

# Paginación

## Clases a usar

```java
import org.springframework.data.domain.Pageable;
import org.springframework.data.web.PageableDefault;
```

> Recuerden que esta es una forma elegante de realizar paginación y recomendada! Sin embargo, ustedes son capaces de hacer su propia lógica de paginación de forma "primitiva" con `findAll()`, `count()` y un par de for's ;)

### Repository

```java
public interface ProductRepository extends JpaRepository<Product, Long> {}
```

### Service

```java
@Service
public class ProductService {

    private final ProductRepository productRepository;

    public ProductService(ProductRepository productRepository) {
        this.productRepository = productRepository;
    }

    public Page<Product> getAllProducts(Pageable pageable) {
        return productRepository.findAll(pageable);
    }
}
```

### Controller

```java
@RestController
@RequestMapping("/api/products")
public class ProductController {

    private final ProductService productService;

    public ProductController(ProductService productService) {
        this.productService = productService;
    }

    @GetMapping
    public ResponseEntity<Page<Product>> getAllProducts(
            @PageableDefault(page = 0, size = 5) Pageable pageable) {

        Page<Product> productsPage = productService.getAllProducts(pageable);
        return ResponseEntity.ok(productsPage);
    }
}
```

### Ejemplo de response del service anterior

Pero... Solo queremos un par de datos en nuestro caso. ¿Cómo lo pasamos a un DTO?

```json
{
  "content": [
    {
      "id": 1,
      "name": "Smartphone",
      "price": 699.99,
      "stock": 50,
      "category": "Electronics"
    },
    {
      "id": 2,
      "name": "Laptop",
      "price": 1200.50,
      "stock": 25,
      "category": "Electronics"
    }
  ],
  "pageable": {
    "pageNumber": 0,
    "pageSize": 2,
    "sort": {
      "empty": true,
      "sorted": false,
      "unsorted": true
    },
    "offset": 0,
    "paged": true,
    "unpaged": false
  },
  "last": false,
  "totalPages": 3,
  "totalElements": 6,
  "size": 2,
  "number": 0,
  "sort": {
    "empty": true,
    "sorted": false,
    "unsorted": true
  },
  "first": true,
  "numberOfElements": 2,
  "empty": false
}
```

### Solución: usar DTOs

#### ProductDto

```java
public class ProductDto {

    private Long id;
    private String name;
    private double price;
    private int stock;
    private String category;

    // Getters y setters
}
```

#### PagedResponseDto

```java
public class PagedResponseDto<T> {

    private List<T> content;
    private int page;
    private int size;
    private long totalElements;

    public PagedResponseDto(Page<T> pageResult) {
        this.content = pageResult.getContent();
        this.page = pageResult.getNumber();
        this.size = pageResult.getSize();
        this.totalElements = pageResult.getTotalElements();
    }

    // Getters
}
```

#### ProductService (con mapeo a DTO)

```java
@Service
public class ProductService {

    private final ProductRepository productRepository;
    private final ModelMapper modelMapper;

    public ProductService(ProductRepository productRepository,
                          ModelMapper modelMapper) {
        this.productRepository = productRepository;
        this.modelMapper = modelMapper;
    }

    public Page<ProductDto> getAllProducts(Pageable pageable) {
        Page<Product> productsPage = productRepository.findAll(pageable);

        // Transforma la Page de entidades a una Page de DTOs
        return productsPage.map(product -> modelMapper.map(product, ProductDto.class));
    }
}
```

#### ProductController (con PagedResponseDto)

```java
@RestController
@RequestMapping("/api/products")
public class ProductController {

    private final ProductService productService;

    public ProductController(ProductService productService) {
        this.productService = productService;
    }

    @GetMapping
    public ResponseEntity<PagedResponseDto<ProductDto>> getAllProducts(
            @PageableDefault(page = 0, size = 10) Pageable pageable) {

        Page<ProductDto> productsPage = productService.getAllProducts(pageable);

        // Crea la respuesta final usando tu DTO de paginación
        PagedResponseDto<ProductDto> responseDto = new PagedResponseDto<>(productsPage);

        return ResponseEntity.ok(responseDto);
    }
}
```

### ¡Listo! Tenemos el formato que queríamos

```json
{
  "content": [
    {
      "id": 1,
      "name": "Smartphone",
      "price": 699.99,
      "stock": 50,
      "category": "Electronics"
    },
    {
      "id": 2,
      "name": "Laptop",
      "price": 1200.50,
      "stock": 25,
      "category": "Electronics"
    }
  ],
  "page": 0,
  "size": 2,
  "totalElements": 6
}
```

---

# Security

Para generar un jwt secet puedes usar:

```bash
openssl rand -base64 32.
```

## Account implements UserDetails

```java
@Entity
public class Account implements UserDetails {

    // ...

    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        /* Mapea los roles del usuario a GrantedAuthority.
         * Tu entidad Account tiene un método getRoles() que devuelve
         * una lista de strings con los nombres de los roles (ej: ["ADMIN", "USER"]).
         */
        return Account.getRoles().stream()
                .map(role -> new SimpleGrantedAuthority("ROLE_" + role.getName()))
                .collect(Collectors.toList());
    }

    public String getPassword() { return password; }

    @Override
    public String getUsername() { return this.email; }

    // ...
}
```

## JwtService

```java
@Component
public class JwtService {

    @Value("${jwt.secret}")
    private String secret;

    @Value("${jwt.expiration-access}")
    private Long accessTokenExpiration;

    private Key getSigningKey() {
        return Keys.hmacShaKeyFor(secret.getBytes());
    }

    public String generateToken(UserDetails userDetails) {
        Date now = new Date();
        return Jwts.builder()
                .subject(userDetails.getUsername())
                .claim("roles",
                        userDetails.getAuthorities().stream()
                                .map(GrantedAuthority::getAuthority).toList())
                .issuedAt(now)
                .expiration(new Date(now.getTime() + accessTokenExpiration))
                .signWith(getSigningKey())
                .compact();
    }

    public boolean isTokenValid(String token) {
        try {
            Jwts.parser()
                    .verifyWith((SecretKey) getSigningKey())
                    .build()
                    .parseSignedClaims(token);

            return true;
        } catch (JwtException | IllegalArgumentException e) {
            // Token is invalid or expired
            return false;
        }
    }

    public String extractUsername(String token) {
        return Jwts.parser()
                .verifyWith((SecretKey) getSigningKey())
                .build()
                .parseSignedClaims(token)
                .getPayload()
                .getSubject();
    }
}
```

## JwtAuthorizationFilter

```java
@Component
public class JwtAuthorizationFilter extends OncePerRequestFilter {
    // ...
    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain)
            throws ServletException, IOException {

        final String authHeader = request.getHeader("Authorization");

        if (StringUtils.hasText(authHeader) && StringUtils.startsWithIgnoreCase(authHeader, "Bearer ")) {
            String token = authHeader.substring(7);

            if (jwtService.isTokenValid(token)) {
                String username = jwtService.extractUsername(token);

                if (StringUtils.hasText(username)
                        && SecurityContextHolder.getContext().getAuthentication() == null) {

                    UserDetails userDetails = userDetailsService.loadUserByUsername(username);
                    SecurityContext context = SecurityContextHolder.createEmptyContext();
                    UsernamePasswordAuthenticationToken authToken = new UsernamePasswordAuthenticationToken(
                            userDetails, null, userDetails.getAuthorities());
                    authToken.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
                    context.setAuthentication(authToken);
                    SecurityContextHolder.setContext(context);
                }
            }
        }
        // follow the chain
        filterChain.doFilter(request, response);
    }
}
```

## SecurityConfig

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity(securedEnabled = true)
public class SecurityConfig {

    private final AccountService userDetailsService;
    private final JwtAuthorizationFilter jwtFilter;

    public SecurityConfig(AccountService userDetailsService, JwtAuthorizationFilter jwtFilter) {
        this.userDetailsService = userDetailsService;
        this.jwtFilter = jwtFilter;
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    @Bean
    public DaoAuthenticationProvider authenticationProvider() {
        DaoAuthenticationProvider provider = new DaoAuthenticationProvider(userDetailsService);
        provider.setPasswordEncoder(passwordEncoder());
        return provider;
    }

    @Bean
    public AuthenticationManager authenticationManager() {
        return new ProviderManager(List.of(authenticationProvider()));
    }
    // ...SecurityFilterChain filterChain(HttpSecurity http)
}
```

### SecurityFilterChain

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    return http
            .csrf(AbstractHttpConfigurer::disable)
            .sessionManagement(
                    manager -> manager.sessionCreationPolicy(STATELESS)
            )
            .authorizeHttpRequests(auth -> auth
                    .requestMatchers("/auth/**").permitAll()
                    .anyRequest().authenticated()
            )
            .authenticationProvider(authenticationProvider())
            .addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class)
            .build();
}
```

## AccountService (UserDetailsService)

```java
@Service
public class AccountService implements UserDetailsService {
    private final AccountRepository accountRepository;

    public AccountService(AccountRepository accountRepository) {
        this.accountRepository = accountRepository;
    }

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        return accountRepository
                .findByEmail(username)
                .orElseThrow();
    }
}
```

## Autorización por rol/authority

### A nivel de configuración (HttpSecurity)

```java
http
    .authorizeHttpRequests((authorize) -> authorize
        .requestMatchers("/resource/**").hasAuthority("USER")
        .anyRequest().authenticated()
    )
```

### A nivel de método (`@PreAuthorize`)

```java
@PreAuthorize("hasRole('USER') or hasRole('ADMIN')")
@GetMapping("/change-password/{id}")
public ResponseEntity<?> changePassword(@PathVariable("id") Long userId) {
    // TODO
}
```

