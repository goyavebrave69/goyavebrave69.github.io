---
sidebar_position: 1
---
# Cours n°1
## SPRING SECURITY BROOOO 🔐🚔

C’est l’heure de jouer les FLICS du web 🙌.
## Étape 1 : Commençons par ajouter la dépendance à notre pom.xml

```xml
<dependencies>
	<!-- ... other dependency elements ... -->
	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-security</artifactId>
	</dependency>
   
</dependencies>
```

## Étape 2: Modèle et service utilisateur

- Définir une classe model User contenant les champs du User ainsi que son repo
- Implémenter une interface UserDetailsService pour charger les détails de l'utilisateur en fonction du nom d'utilisateur

On a dit DDD ! Donc on créer un nouveau package `auth` qui va contenir tout notre code en lien avec la security, et encore une fois à l’intérieur du package nous aurons : application, domain, infrastructure.

### Model 

```js
package com.crud.demo.auth.domain.entity;

import js.util.List;

@Entity
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "first_name", nullable = false)
    private String firstName;

    @Column(name = "last_name", nullable = false)

    private String lastName;

    @Column(name = "username", nullable = false, unique = true)

    private String username;

    @Column(name = "email", nullable = false, unique = true)
    private String email;

    @Column(name = "password", nullable = false)
    private String password;

    @Column(name = "is_enabled", nullable = false)
    private Boolean isEnabled;

    @ManyToOne(cascade = CascadeType.MERGE, fetch = FetchType.EAGER)
    @JoinColumn(name = "role_id")
    private Role roles;
		

// getters and setters
```

Mhhh tiens tiens qu’est-ce que l’on remarque ici ?? Une petite relation je crois hein !
C’est parti ajoutons cette classe `Role`

```js
@Entity
public class Role {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String type;

    public Role() {
    }

    public Role(String type) {
        this.type = type; // ADMIN, USER, ...
    }

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public String getType() {
        return type;
    }

    public void setType(String type) {
        this.type = type;
    }
}
```

### Repository 

```js
package com.crud.demo.auth.infrastructure.repository;

import com.crud.demo.auth.domain.entity.User;
import org.springframework.data.jpa.repository.JpaRepository;

public interface UserRepository  extends JpaRepository<User, Long> { }
```

### UserDetailsServiceImpl
Maintenant on créer un package service à l’intérieur du `domain` avec une class UserDetailsServiceImpl qui implements UserDetailService.

```js
package com.crud.demo.auth.domain.service;

import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.core.userdetails.UsernameNotFoundException;

public class UserDetailsServiceImpl implements UserDetailsService {

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        return null;
    }
}
```

C’est ici que l’on va coder la fonction qui va nous permettre de récupérer l’utilisateur si il existe bien, dans le cas contraire, une erreur sera renvoyée.

Commençons donc par 💉 notre repository  puis allons chercher ce fameux user par son Username

```js
private final UserRepository userRepository;

UserDetailsServiceImpl(UserRepository userRepository) {
      this.userRepository = userRepository;
}

@Override
public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {

      return userRepository.findByUsername(username);
}
```

```js
public interface UserRepository extends JpaRepository<User, Long> {
    User findByUsername(String username);
}
```

Sauf que… Si on regarde juste notre IDE on voit qu’il y a un problème `Required Type : UserDetails` , car ici la fonction attend un return de type UserDetails, là où on retourne directement un User.

Nous allons donc devoir créer une classe qui va nous permettre de construire un objet de type UserDetails avec les informations de mon User.



```js
package com.crud.demo.auth.domain.entity;

import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.core.userdetails.UserDetails;

import js.util.ArrayList;
import js.util.Collection;
import js.util.List;

public class UserPrincipal implements UserDetails {

    private final User user;
    
		UserPrincipal(User user) {
        this.user = user;
    }

    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        List<GrantedAuthority> authorities = new ArrayList<>();
        for (String role: user.getRoles()) {
            authorities.add(new SimpleGrantedAuthority(role));
        }
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

Désormais nous allons return une petite instance de cette classe dans notre loadUserByUsername

```js
@Override
public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {

   return new UserPrincipal(userRepository.findByUsername(username));
}
```