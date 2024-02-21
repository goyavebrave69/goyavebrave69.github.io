---
sidebar_position: 1
---
# Les Tests en SPRING
### **Tutoriel : Test JUnit pour l'entité Article en Java**

### Étape 1 : Créer une classe de test

Créez une nouvelle classe de test pour l'entité **`Article`**. Cette classe de test contiendra les méthodes de test pour vérifier le comportement de votre entité.

```js
import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertNotNull;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.LocalDateTime;
import java.util.ArrayList;
import java.util.List;

public class ArticleTest {

    private Article article;

    @BeforeEach
    public void setUp() {
        article = new Article();
    }

    // Ajoutez ici vos méthodes de test
}

```

### Étape 2 : Mettre en place l'environnement de test

Utilisez l'annotation **`@BeforeEach`** pour créer une méthode d'initialisation qui sera exécutée avant chaque test. Dans cette méthode, initialisez une instance de l'entité **`Article`**.

```js
@BeforeEach
public void setUp() {
    article = new Article();
}

```

### Étape 3 : Tester l'ID de l'article

Écrivez une méthode de test pour vérifier la configuration et la récupération de l'ID de l'article.

```js
@Test
public void testArticleId() {
    Long id = 1L;
    article.setId(id);
    assertEquals(id, article.getId());
}
```

### Étape 4 : Tester le titre de l'article

Écrivez une méthode de test pour vérifier la configuration et la récupération du titre de l'article.

```js
@Test
public void testArticleTitle() {
    String title = "Test Title";
    article.setTitle(title);
    assertEquals(title, article.getTitle());
}

```

### Étape 5 : Tester la description de l'article

Écrivez une méthode de test pour vérifier la configuration et la récupération de la description de l'article.

```js
@Test
public void testArticleDescription() {
    String description = "Test Description";
    article.setDescription(description);
    assertEquals(description, article.getDescription());
}

```

### Étape 6 : Tester la date de création de l'article

Écrivez une méthode de test pour vérifier si la date de création de l'article est correcte.

```js
@Test
public void testArticleCreatedAt() {
    LocalDateTime createdAt = LocalDateTime.now();
    assertEquals(createdAt.getYear(), article.getCreatedAt().getYear());
    assertEquals(createdAt.getMonth(), article.getCreatedAt().getMonth());
    assertEquals(createdAt.getDayOfMonth(), article.getCreatedAt().getDayOfMonth());
}

```

### Étape 7 : Tester les commentaires de l'article

Écrivez une méthode de test pour vérifier la configuration et la récupération des commentaires de l'article.

```js
@Test
public void testArticleComments() {
    List<Comment> comments = new ArrayList<>();
    Comment comment1 = new Comment();
    Comment comment2 = new Comment();
    comments.add(comment1);
    comments.add(comment2);

    article.setComments(comments);
    assertNotNull(article.getComments());
    assertEquals(2, article.getComments().size());
}

```

### Étape 8 : Exécuter les tests

Exécutez les tests en utilisant votre environnement de développement intégré (IDE) ou en utilisant Maven. Assurez-vous que tous les tests passent avec succès.

## Tester un controller

Désormais, allons créer un test pour notre Controller d’article

```js
@SpringBootTest
@AutoConfigureMockMvc
public class ArticleControllerTest {

    @Autowired
    private MockMvc mockMvc;

	@Test
    public void testGetAllArticles() throws Exception {

        mockMvc
                .perform(MockMvcRequestBuilders.get("/articles")
                        .contentType(MediaType.APPLICATION_JSON))
                .andExpect(MockMvcResultMatchers.status().isOk())
                .andExpect(MockMvcResultMatchers.jsonPath("$[0].title").value("Sujet santée"));
    }
}
```

Petit hic 😕 on se prend une erreur en pleine tronche :’(

Eh oui, notre application est sécurisée donc il faut avoir un token

Pour se faire, allons essayer dans un premier temps de manière brutale en récupérant le token en dur via insomnia

Super, maintenant de manière plus propre, on va récupérer directement le token depuis notre code :

```js
public String loginHelper() throws Exception {
        JSONObject jo = new JSONObject();
        jo.put("username", "goyave");
        jo.put("password", "admin");
        ;

        ResultActions resultActions = mockMvc.perform(
                        MockMvcRequestBuilders.post("/login")
                                .content(jo.toString())
                                .contentType(MediaType.APPLICATION_JSON)
                )
                .andDo(MockMvcResultHandlers.print());
        ;

        MvcResult result = resultActions.andReturn();

        return result.getResponse().getContentAsString();
    }
```

ajoutons maintenant la ligne de code dans notre test afin d’utiliser le token

```js
@Test
    public void testGetAllArticles() throws Exception {

        mockMvc
                .perform(MockMvcRequestBuilders.get("/articles")
                        .header("Authorization", "Bearer " + loginHelper())
                        .contentType(MediaType.APPLICATION_JSON))
                .andExpect(MockMvcResultMatchers.status().isOk())
                .andExpect(MockMvcResultMatchers.jsonPath("$[0].title").value("Sujet santée"));
    }
```

Exo : créer un test de login et registration ainsi qu’un test pour vérifier que l’accès est bien vérouillé

### Tester création d’article

```js
@Test
    public void testCreateArticle() throws Exception {

        JSONObject article = new JSONObject();
        article.put("title", "article de test unitaire");
        article.put("description", "article de test unitaire");

        mockMvc
                .perform(MockMvcRequestBuilders.post("/articles")
                        .header("Authorization", "Bearer " + loginHelper())
                        .content(article.toString())
                        .contentType(MediaType.APPLICATION_JSON))
                .andExpect(MockMvcResultMatchers.status().isOk())
                .andExpect(MockMvcResultMatchers.jsonPath("$.title").value("article de test unitaire"))
                .andExpect(MockMvcResultMatchers.jsonPath("$.description").value("article de test unitaire"));
    }
```

Allons créer une BDD spéciale pour nos tests, nous ne voulons pas que nos tests impactent notre vraie BDD, pour ce faire nous allons utiliser h2 database, commençons donc par les dépendances

```js
<dependency>
			<groupId>com.h2database</groupId>
			<artifactId>h2</artifactId>
			<scope>test</scope>
</dependency>
```

À l’image de notre dossier main, nous allons également faire un dossier ressources dans lequel nous allons mettre un [application.properties](http://application.properties) spécial pour nos tests, un petit copier coller ni vu ni connu 🙄

```js
spring.datasource.url=jdbc:h2:mem:testdb
spring.datasource.driver-class-name=org.h2.Driver
spring.datasource.username=sa
spring.datasource.password=password
spring.jpa.defer-datasource-initialization=true
spring.jpa.hibernate.ddl-auto=update
spring.jpa.properties.hibernate.format_sql=true
spring.h2.console.enabled=true
```

Afin d’éviter un problème de syntaxe, nous allons rajouter ceci dans notre Entity User

`@Table(name = "`user`")`car h2 database a comme mot réservé user

Nous allons créer un fichier pour tester notre authentication avec les annotations suivantes

```js
@SpringBootTest
@AutoConfigureMockMvc
@TestPropertySource(
        locations = "classpath:application-test.properties"
)
public class AuthControllerTest {
}
```

Commençons par tester notre methode register :

```js
@Autowired
private MockMvc mockMvc;

    @Test
    public void testRegister() throws Exception {
        JSONObject jsonUser = new JSONObject();
        jsonUser.put("username", "goyaveeoui");
        jsonUser.put("lastName", "brave");
        jsonUser.put("firstName", "goyave");
        jsonUser.put("password", "admin");
        jsonUser.put("email", "goyaveeoui@goyave.com");
        jsonUser.put("enabled", true);

        mockMvc
                .perform(MockMvcRequestBuilders.post("/register")
                        .content(jsonUser.toString())
                        .contentType(MediaType.APPLICATION_JSON))
                .andExpect(MockMvcResultMatchers.status().isCreated())
                .andExpect(MockMvcResultMatchers.jsonPath("$.username").value("goyaveeoui"))
                .andExpect(MockMvcResultMatchers.jsonPath("$.lastName").value("brave"))
                .andExpect(MockMvcResultMatchers.jsonPath("$.firstName").value("goyave"))
                .andExpect(MockMvcResultMatchers.jsonPath("$.email").value("goyaveeoui@goyave.com"))
                .andExpect(MockMvcResultMatchers.jsonPath("$.enabled").value(true))
                ;
    }
```

De cette manière on va pouvoir s’assurer que le User se créer bien.

Sachez qu’avec cette config, notre BDD est actualisée à chaque lancement de test, ce qui veut dire que les data ne sont pas persistées post-test.

Maintenant, occupons nous de tester notre login :

```js
@Test
    public void testLogin() throws Exception {

        JSONObject jsonUser = new JSONObject();
        jsonUser.put("username", "johndoe");
        jsonUser.put("password", "johndoe");

        MvcResult result = mockMvc
                .perform(MockMvcRequestBuilders.post("/login")
                        .content(jsonUser.toString())
                        .contentType(MediaType.APPLICATION_JSON))
                .andExpect(MockMvcResultMatchers.status().isOk())
                .andReturn();

        assertNotNull(result.getResponse().getContentType());
    }
```

Le problème, c’est qu’il ne trouvera pas cet utilisateur dans la BDD, comment faire 🧐 ?

Nous allons créer un fichier SQL qui se lancera automatiquement à chaque test, ce fichier SQL contiendra une requête qui va nous permettre d’insert un USER avant chaque lancement de test, malynx le lynx ;)

```js
INSERT INTO role (type) VALUES ('admin');

INSERT INTO "user" (email, first_name, is_enabled, last_name, password, username, role_id)
VALUES ('john@doe.com', 'John', true, 'Doe', '$2a$12$S/VXri58yQkBCEIk9CnRtOSXZLFEa03dd5gJ5YwfqFH8wR6Lbfq8S', 'johndoe', 1);
```

maintenant je vais pouvoir aller tester ça

```js
@Test
    public void testLogin() throws Exception {

        JSONObject jsonUser = new JSONObject();
        jsonUser.put("username", "johndoe");
        jsonUser.put("password", "johndoe");

        MvcResult result = mockMvc
                .perform(MockMvcRequestBuilders.post("/login")
                        .content(jsonUser.toString())
                        .contentType(MediaType.APPLICATION_JSON))
                .andExpect(MockMvcResultMatchers.status().isOk())
                .andReturn();

        assertNotNull(result.getResponse().getContentType());
    }
```