# Cours n°3 🍪
## Miam miam, des bons cookies à consommer 😋

Imagine que tu es à la tête d'une agence spatiale ultra-secrète 🚀. Les tokens, c'est comme les badges d'accès 🎫 qui permettent à tes astronautes (les utilisateurs) de monter à bord de ta station spatiale (ton application web). Si tu envoies ces badges d'accès dans le corps de la réponse, c'est un peu comme si tu diffusais leurs fréquences 📡 sur les ondes publiques. Les pirates de l'espace 🏴‍☠️, toujours à l'affût, pourraient intercepter ces fréquences et s'infiltrer incognito.

Par contre, opter pour envoyer ces badges d'accès via des cookies 🍪, c'est comme utiliser un système de téléportation sécurisé 🛸 directement vers la poche de l'astronaute. Avec des mesures de sécurité comme HttpOnly et Secure flag, c'est comme ajouter un champ de force 🛡️ autour de la téléportation, rendant presque impossible pour les pirates de l'espace de mettre la main sur ces précieux badges.

En clair, utiliser des cookies pour transmettre les tokens, c'est choisir la voie de la haute technologie pour garder ton équipage et ta station spatiale en sécurité. C'est futuriste 🌌, c'est astucieux 🧠, et ça assure que ta mission se déroulera sans accroc. 🚀🌠

## Génération du cookie
Après ce blabla métaphorique passons au concrêt, nous allons donc modifier la façon dont nous renvoyons les cookies, et cela en vidant le body qui sera désormais vide, pour passer les cookies au travers du header.

```js
@PostMapping(value = "/login", produces = MediaType.APPLICATION_JSON_VALUE)
public ResponseEntity<?> login(@RequestBody User userBody) throws Exception {
    try {
        userLoginService.login(userBody);
        Token token = jwtTokenService.generateToken(userDetailsService.loadUserByUsername(userBody.getUsername()));

        ResponseCookie jwtCookie = ResponseCookie.from("token", token.getToken())
            .httpOnly(true)   // Marquer le cookie comme HttpOnly pour la sécurité
            //.secure(true)     // Marquer le cookie comme sécurisé (transmis uniquement via HTTPS)
            .path("/")        // Le cookie est accessible pour l'ensemble du domaine
            .maxAge(24 * 60 * 60) // Définir la durée de vie du cookie (exemple : 24 heures)
            .sameSite("Strict") // Politique SameSite pour le cookie
            .build();

        return ResponseEntity.ok()
            .header(HttpHeaders.SET_COOKIE, jwtCookie.toString())
            .build();
    } catch (BadCredentialsException e){
        return ResponseEntity.status(HttpStatus.UNAUTHORIZED).build();
    }
}
```
### Entity Token
Nous avons crée une nouvelle Entity `Token` qui nous sert à représenter ce dernier dans un objet (POO tmtc), en revanche elle ne sera pas liée à la BDD mais nous servira uniquement à l'intérieur de notre code.

```js
public class Token {
    private String token;

    public String getToken() {
        return token;
    }

    public void setToken(String token) {
        this.token = token;
    }
```
### Modification du service
Utilisons donc cette Entity afin de renvoyer le token au travers de celle-ci après sa génération dans la methode `generateToken`.

```js
public Token generateToken(UserDetails userDetails) {
    Date now = new Date();
    Token token = new Token();
    token.setToken(Jwts.
        builder()
            .setSubject(userDetails.getUsername())
            .setIssuedAt(new Date(System.currentTimeMillis()))
            .setExpiration(new Date(now.getTime() + JWT_TOKEN_VALIDITY * 1000))
            .signWith(getSignInKey(), SignatureAlgorithm.HS256)
            .compact()
    );
    return token;
}
```



Retournons dans notre controller.

```js
    return ResponseEntity.ok().header(HttpHeaders.SET_COOKIE, jwtCookie.toString()).build();
```
Cette ligne de code nous permet de retourner dans le header le TOKEN et de retourner tout cela dans le response. 
Ainsi, mon utilisateur recevra automatiquement le cookie dans la response, qui sera récupéré et stocké par son navigateur.


### Génération du Cookie 🍪

Je voudrais maintenant que vous observiez cette partie du code en me disant quel crime a été commis ?

```js
    ResponseCookie jwtCookie = ResponseCookie.from("token", token.getToken())
        .httpOnly(true)   // Marquer le cookie comme HttpOnly pour la sécurité
        //.secure(true)     // Marquer le cookie comme sécurisé (transmis uniquement via HTTPS)
        .path("/")        // Le cookie est accessible pour l'ensemble du domaine
        .maxAge(24 * 60 * 60) // Définir la durée de vie du cookie (exemple : 24 heures)
        .sameSite("Strict") // Politique SameSite pour le cookie
        .build();
```

## 🔪🩸🚨Un meurtre est en cours et il doit être empêché ! 

Gros comme une maison, l'erreur est flagrante, il ne faut absolument pas mettre ce bout de code dans le controller, mais plutôt dans un service dédié, alors au boulot jeune Super-Héros, il faut empêcher ce meurtre de se produire !
Je n'allais quand même pas tout faire à ta place non 😉.
