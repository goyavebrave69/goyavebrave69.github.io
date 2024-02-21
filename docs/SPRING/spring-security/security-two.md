---
sidebar_position: 1
---
# Cours n°2
## Passons à la config JWT 🔑

### Dépendances
Avant tout, ajoutons les Dependencies JWT :

```xml
            <dependency>
			<groupId>io.jsonwebtoken</groupId>
			<artifactId>jjwt-api</artifactId>
			<version>0.11.5</version>
		</dependency>
		<dependency>
			<groupId>io.jsonwebtoken</groupId>
			<artifactId>jjwt-impl</artifactId>
			<version>0.11.5</version>
			<scope>runtime</scope>
	    </dependency>
		<dependency>
			<groupId>io.jsonwebtoken</groupId>
			<artifactId>jjwt-jackson</artifactId>
			<version>0.11.5</version>
			<scope>runtime</scope>
		</dependency>
```

Nous aurons 3 étapes :

- Créer notre controller pour login
- Create a JwtTokenUtil class to generate and validate JWT tokens
- Configure the secret key used for signing and verifying tokens
- Define the expiration time for JWT tokens

### Controller

Notre controller contiendra pour l’instant une seule méthode :

```js
@RestController
public class AuthController {

    private final UserService userService;
    private final JwtTokenUtil jwtTokenUtil;
    private final UserDetailsServiceImpl userDetailsService;

    public AuthController(UserService userService, JwtTokenUtil jwtTokenUtil, UserDetailsServiceImpl userDetailsService) {
        this.userService = userService;
        this.jwtTokenUtil = jwtTokenUtil;
        this.userDetailsService = userDetailsService;
    }

    @PostMapping("/login")
    public ResponseEntity<?> authenticateUser(@RequestBody User loginRequest) throws Exception {
        try {
            userService.login(loginRequest);
            String token = jwtTokenUtil.generateToken(userDetailsService.loadUserByUsername(loginRequest.getUsername()));
            return ResponseEntity.ok(token);
        } catch (BadCredentialsException e) {
            return ResponseEntity.status(HttpStatus.UNAUTHORIZED).build();
        }
    }
}
```
### Les services 

Comme vous le voyez nous avons 3 services, commençons par coder le jwtTokenUtil

```js
@Service
public class JwtTokenUtil {

    private static String secretKey = "Juw34WOQg0Rn397En9Ek+LSH4fDEl4QSBGeN1izonY6xA2/sJLVQs2I5vT5ydJxclQUaiLNc2xqlpcodiEnQ5nGiKCXtBbmO6jkpsxBV/h9HgpzmtkSiahnqolPzE0pPEsEQBa2Sow4pLM1yRahGhKoHUBHEykKL8ADJPyJ4n578th4s5vYAaErhBnJ9rVua42RiQLa8avCo6yiKskfAdKegJvdUv/jkZNrXzeIwvjmVQvoUWvtYDgsKP/8RSBkQ5c0snaDQ/Bl7XaPsp/rk1Cy6FW6pb4p6RMyBwVsFxtMEGkM0rxjpUkinIwRxidkk5aeMU8xjx+IH9D5CIAPZzM9GgzbI7WNHMKQKp8iUkC4=";
    public static final long JWT_TOKEN_VALIDITY = 5 * 60 * 60;

		public String generateToken(UserDetails userDetails) {
        Date now = new Date();
        Date expiryDate = new Date(now.getTime() + JWT_TOKEN_VALIDITY * 1000); // 1 day validity
        return Jwts
                .builder()
                .setSubject(userDetails.getUsername())
                .setIssuedAt(new Date(System.currentTimeMillis()))
                .setExpiration(expiryDate)
                .signWith(getSignInKey(), SignatureAlgorithm.HS256)
                .compact();
    }

		private Key getSignInKey() {
        byte[] keyBytes = Decoders.BASE64.decode(secretKey);

        return Keys.hmacShaKeyFor(keyBytes);
    }

}
```

Le user service :

```js
@Service
public class UserService {
    private UserRepository userRepository;
    private BCryptPasswordEncoder bcryptEncoder;
    
    public UserService(UserRepository userRepository, BCryptPasswordEncoder bcryptEncoder) {
        this.userRepository = userRepository;
        this.bcryptEncoder = bcryptEncoder;
    }

    public User login(User user) {
        User userEntity = getUserEntityByUsername(user.getUsername());
        if (!verifyHashedPasswordDuringLogin(user.getPassword(), userEntity.getPassword())) {
            throw new RuntimeException("Le mot de passe est incorrect");
        }
        user.setRoles(userEntity.getRoles());
        
        return user;
    }
    
    public boolean verifyHashedPasswordDuringLogin(String password, String hashedPassword) {
        return bcryptEncoder.matches(password, hashedPassword);
    }

    public User getUserEntityByUsername(String username) {
        try {
            return userRepository.findByUsername(username);
        } catch (Exception e) {
            throw new RuntimeException("L'email n'existe pas");
        }
    }
    
}
```

Le user details service :

```js
@Service
public class UserDetailsServiceImpl implements UserDetailsService {

    private final UserRepository userRepository;

    UserDetailsServiceImpl(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {

        return new UserPrincipal(userRepository.findByUsername(username));
    }
}
```

Mhhh `new UserPrincipal` 🧐

C’est la classe qui va nous servir à construire notre object de type UserDetails

```js
public class UserPrincipal implements UserDetails {

    private final User user;
    public UserPrincipal(User user) {
        this.user = user;
    }
    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        List<GrantedAuthority> authorities = new ArrayList<>();
        authorities.add(new SimpleGrantedAuthority(user.getRoles().getType()));
        return authorities;
    }

    @Override
    public String getPassword() {
        return user.getPassword();
    }

    @Override
    public String getUsername() {
        return user.getUsername();
    }

    @Override
    public boolean isAccountNonExpired() {
        return false;
    }

    @Override
    public boolean isAccountNonLocked() {
        return false;
    }

    @Override
    public boolean isCredentialsNonExpired() {
        return false;
    }

    @Override
    public boolean isEnabled() {
        return user.getEnabled();
    }
}
```
### Configuration de la sécurité 🪪
Maintenant, il va falloir que l’on passe à la config de la securité ! Pour ce faire, nous créons un fichier que l’on appellera `SecurityConfig` à l’intérieur d’application

```js
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    private final JwtAuthenticationFilter jwtAuthenticationFilter;

    public SecurityConfig(JwtAuthenticationFilter jwtAuthenticationFilter) {
        this.jwtAuthenticationFilter = jwtAuthenticationFilter;
    }

    @Bean
    public BCryptPasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
                .authorizeHttpRequests((requests) -> requests
                        .requestMatchers("/register", "/login").permitAll()
                        .requestMatchers("/logout", "/user-admin-data").hasAnyAuthority("USER", "ADMIN")
                        .requestMatchers("/admin", "/only-admin-data").hasAuthority("ADMIN")
                        .requestMatchers("/articles/**")
                        .authenticated())
                .csrf((csrf) -> csrf.csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse()) // You can disable csrf protection by removing this line
                        .ignoringRequestMatchers("/register", "/login")
                        .disable()  // Décommentez pour désactiver en entier la protection CSRF en développement
                )
                .sessionManagement(session -> session
                                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)// Spring Security ne crée pas de session
                        // Nous allons utiliser JWT pour gérer les sessions
                );
        http.addFilterBefore(jwtAuthenticationFilter, UsernamePasswordAuthenticationFilter.class);
        return http.build(); // Très important de retourner le résultat de la méthode build() !
        // Sinon rien de tout ça ne fonctionne !
    }
}
```

Ajoutons des données manuellement en BDD afin de pouvoir effectuer un petit test

Procédons maintenant à la création du jwtAuthenticationFilter :

```js
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtTokenUtil jwtTokenUtil;

    private final UserDetailsServiceImpl userDetailsService;

    JwtAuthenticationFilter(JwtTokenUtil jwtTokenUtil, UserDetailsServiceImpl userDetailsService) {
        this.jwtTokenUtil = jwtTokenUtil;
        this.userDetailsService = userDetailsService;
    }

    @Override
    protected void doFilterInternal(
            @NonNull HttpServletRequest request,
            @NonNull HttpServletResponse response,
            @NonNull FilterChain filterChain
    ) throws ServletException, IOException {
        String authHeader = request.getHeader("Authorization");
        final String jwt;
        final String userEmail;

        if (authHeader == null ||!authHeader.startsWith("Bearer ")) {
            filterChain.doFilter(request, response);
            return;
        }
        jwt = authHeader.substring(7);
        userEmail = jwtTokenUtil.getUsernameFromToken(jwt);

        if (userEmail != null && SecurityContextHolder.getContext().getAuthentication() == null) {
            UserDetails userDetails = this.userDetailsService.loadUserByUsername(userEmail);
            if (jwtTokenUtil.isTokenValid(jwt, userDetails)) {
                UsernamePasswordAuthenticationToken authToken = new UsernamePasswordAuthenticationToken(
                        userDetails,
                        null,
                        userDetails.getAuthorities()
                );
                authToken.setDetails(
                        new WebAuthenticationDetailsSource().buildDetails(request)
                );
                SecurityContextHolder.getContext().setAuthentication(authToken);
            }
        }

        filterChain.doFilter(request, response);
    }
```

## Enregistrer un utilisateur

Voyons voir maintenant pour la partie register, créons un service dédiée.
Pour ce faire, nous allons renommer notre `UserService`  en UserLoginService, car le fait qu’il y ai dedans des methodes pour le login & le register nous ferait aller à l’encontre de notre fameux S de solid (single responsability principle babe !)

Voici le code de notre UserRegistrationService qui va contenir deux fonctions :
* `UserRegistration` qui permet d'appeler notre deuxième fonction afin de hasher notre mdp, d'enregistrer en BDD notre User via le repository, puis de return le User via un DTO
* `GenerateHashedPassword` qui nous permet d'hasher le mot de passe du `user`

### Service register pour User

```js
@Service
public class UserRegistrationService {
    private UserRepository userRepository;
    private BCryptPasswordEncoder bcryptPasswordEncoder;
    private UserMapper userMapper;

    public UserRegistrationService(
        UserRepository userRepository,
        BCryptPasswordEncoder bcryptPasswordEncoder,
        UserMapper userMapper
    ) {
        this.userRepository = userRepository;
        this.bcryptPasswordEncoder = bcryptPasswordEncoder;
        this.userMapper = userMapper;
    }

    public UserDTO UserRegistration(User userData) throws Exception {
        userData.setPassword(GenerateHashedPassword(userData.getPassword()));
        try {
            return userMapper.transformUserEntityInUserDto(userRepository.save(userData), true);
        } catch (Exception e) {
            throw new Exception(e.getMessage());
        }
}

    public String GenerateHashedPassword(String password) {
        return bcryptPasswordEncoder.encode(password);
    }
}
```
### User DTO
Nous avons besoin maintenant de coder notre DTO, alors direction le package DTO pour créer UserMapper : 

```js
@Service
public class UserMapper {

    public UserDTO transformUserEntityInUserDto(User user, Boolean isForResponse) {
        UserDTO userDTO = new UserDTO();
        userDTO.setEmail(user.getEmail());
        userDTO.setFirstName(user.getFirstName());
        userDTO.setLastName(user.getLastName());
        userDTO.setUsername(user.getUsername());
        userDTO.setEnabled(true);
        userDTO.setRoles(user.getRoles());
        userDTO.setPassword(isForResponse ? "" : user.getPassword());

        return userDTO;
    }
}
```

Bon, tout ça c'est bien beau mais j'ai besoin désormais d'une nouvelle route dans mon joli `AuthController` afin d'éxecuter tout cela !

```js 
@PostMapping("/register")
    public ResponseEntity<?> register(@RequestBody User userBody) throws RegistrationErrorException {
        try {
            return ResponseEntity.status(201).body(userRegistrationService.UserRegistration(userBody));
        } catch (Exception e) {
            throw new RegistrationErrorException(e.getMessage());
        }
    }
```

TADAMMM, désormais tout est ok pour pouvoir Créer un nouvel utilisateur puis qu'il se log par la suite 😄


Tu pensais que c'était fini hein ? Je te laisse regarder la suite si tu es fan de 🍪, ça devrait te plaire 😉.