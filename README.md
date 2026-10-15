# TP — Authentification JWT avec Spring Boot & Spring Security

API REST sécurisée par **JSON Web Token (JWT)** : inscription, connexion, puis
accès aux routes protégées via l'en-tête `Authorization: Bearer <token>`.
Projet réalisé dans le cadre du cours de sécurité (ENSET Mohammedia).

---

## 1. Technologies

| Outil | Version |
|---|---|
| Java | 21 |
| Spring Boot | 4.1.1 |
| Spring Security | Fournie par Spring Boot |
| Spring Data JPA + H2 | Base de données en mémoire |
| jjwt (`io.jsonwebtoken`) | 0.13.0 |
| Lombok | Fournie par Maven |
| Maven | Wrapper `mvnw` inclus |

---

## 2. Principe de fonctionnement

```text
Client                         API Spring Boot
  │  POST /auth/register            │  enregistre l'utilisateur
  │────────────────────────────────>│  (mot de passe haché avec BCrypt)
  │                                 │
  │  POST /auth/login               │  vérifie email + mot de passe
  │────────────────────────────────>│  génère un JWT signé (HS256)
  │<──────── { "token": "..." } ────│
  │                                 │
  │  GET /api/hello                 │  JwtFilter : lit l'en-tête Authorization,
  │  Authorization: Bearer <token>  │  vérifie signature + expiration,
  │────────────────────────────────>│  place l'utilisateur dans le SecurityContext
  │<──────── 200 OK ────────────────│
```

---

## 3. Structure du projet

```text
src/main/java/ma/enset/tp_jwt_spring
├── config
│   └── SecurityConfig.java
│       # BCrypt, AuthenticationManager et chaîne de filtres
├── controller
│   ├── AuthController.java
│   │   # /auth/register et /auth/login
│   └── UserController.java
│       # /api/hello et /api/profile (protégées)
├── dto
│   ├── AuthResponse.java
│   │   # { token }
│   ├── LoginRequest.java
│   │   # { email, password }
│   └── RegisterRequest.java
│       # { email, password }
├── entity
│   └── AppUser.java
│       # table users : id, email, password, role
├── repository
│   └── UserRepository.java
│       # findByEmail et existsByEmail
├── security
│   └── JwtFilter.java
│       # filtre exécuté à chaque requête
├── service
│   ├── CustomerUserDetailsServiceImpl.java
│   │   # AppUser -> UserDetails
│   └── JwtService.java
│       # génération, lecture et validation du JWT
└── TpJwtSpringApplication.java
```

---

## 4. Configuration (`application.properties`)

```properties
spring.application.name=tp-jwt-spring

spring.datasource.url=jdbc:h2:mem:securitydbdemo
spring.datasource.driverClassName=org.h2.Driver
spring.datasource.username=sa
spring.datasource.password=

spring.jpa.show-sql=true
spring.jpa.hibernate.ddl-auto=update

spring.h2.console.enabled=true
spring.h2.console.path=/h2-console

jwt.secret=<clé HMAC encodée en Base64>
jwt.expiration=3600000        # 1 heure (en millisecondes)
```

> ⚠️ La clé `jwt.secret` présente dans ce projet est une clé de démonstration.
> En production, elle doit être générée aléatoirement et fournie par une
> variable d'environnement, jamais versionnée.

---

## 5. Lancer le projet

### Linux / macOS

```bash
./mvnw spring-boot:run
```

Si le wrapper n'est pas exécutable :

```bash
bash mvnw spring-boot:run
```

### Windows

```bat
mvnw.cmd spring-boot:run
```

L'API démarre sur [http://localhost:8080](http://localhost:8080/).

La console H2 est disponible sur
[http://localhost:8080/h2-console](http://localhost:8080/h2-console) avec :

- **JDBC URL** : `jdbc:h2:mem:securitydbdemo`
- **Utilisateur** : `sa`
- **Mot de passe** : vide

---

## 6. Règles de sécurité

| Route | Accès |
|---|---|
| `POST /auth/register` | Public |
| `POST /auth/login` | Public |
| `/h2-console/**` | Public, uniquement pour le développement |
| `/api/**` | **Authentifié**, JWT valide obligatoire |
| Autres routes | Public |

L'API est **stateless** : aucune session serveur n'est utilisée. La protection
CSRF est désactivée pour cette API REST et le `JwtFilter` est exécuté avant
`UsernamePasswordAuthenticationFilter`.

---

## 7. Tests avec Postman ou un client HTTP

Un fichier [`requests.http`](requests.http) est fourni à la racine pour
rejouer les requêtes.

### 7.1 Inscription — `POST /auth/register`

**URL** : `http://localhost:8080/auth/register`

**Headers** :

```http
Content-Type: application/json
```

**Body** :

```json
{
  "email": "user@enset.ma",
  "password": "1234"
}
```

**Réponse attendue** : `200 OK`

```text
Utilisateur cree
```

Le mot de passe est enregistré sous forme de hash BCrypt et n'est jamais
stocké en clair.

### 7.2 Inscription avec un email déjà utilisé

Rejouer la même requête.

**Réponse attendue** : `400 Bad Request`

```text
Email deja utilise
```

### 7.3 Vérification en base avec la console H2

Dans la console H2, exécuter :

```sql
SELECT * FROM USERS;
```

La colonne `PASSWORD` doit contenir un hash BCrypt commençant généralement
par `$2a$` ou `$2b$`, et non le mot de passe original.

### 7.4 Connexion — `POST /auth/login`

**URL** : `http://localhost:8080/auth/login`

**Body** :

```json
{
  "email": "user@enset.ma",
  "password": "1234"
}
```

**Réponse attendue** : `200 OK`

```json
{
  "token": "eyJhbGciOiJIUzI1NiJ9...."
}
```

### 7.5 Connexion avec un mauvais mot de passe

Utiliser un mot de passe incorrect dans la requête de connexion.

**Résultat attendu** : échec d'authentification et aucun token JWT délivré.

### 7.6 Accès sans token — `GET /api/hello`

```http
GET http://localhost:8080/api/hello
```

**Réponse attendue** : `401 Unauthorized` ou `403 Forbidden`, selon le
gestionnaire d'exception utilisé par Spring Security.

### 7.7 Accès avec token — `GET /api/hello`

```http
GET http://localhost:8080/api/hello
Authorization: Bearer <TOKEN>
```

**Réponse attendue** : `200 OK`

```text
Bienvenue dans l'API securisee
```

Dans Postman, sélectionner **Authorization > Bearer Token**, puis coller le
token reçu lors de la connexion.

### 7.8 Profil de l'utilisateur connecté — `GET /api/profile`

```http
GET http://localhost:8080/api/profile
Authorization: Bearer <TOKEN>
```

**Réponse attendue** : `200 OK`

```text
Utilisateur connecte : user@enset.ma
```

### 7.9 Token invalide ou modifié

Modifier un caractère du token ou utiliser un token expiré.

**Résultat attendu** : accès refusé. Le `JwtFilter` n'authentifie pas la
requête lorsque la signature ou l'expiration du token est invalide.

### 7.10 Décodage du token avec jwt.io

Le token peut être décodé sur [jwt.io](https://jwt.io/). On y retrouve :

- le header avec l'algorithme `HS256` ;
- le payload avec `sub` (l'email), `iat` et `exp` ;
- aucune information de mot de passe.

Ne jamais publier la clé secrète sur un outil public avec un token de
production.

---

## 8. Tests automatisés

Pour exécuter les tests :

```bash
bash mvnw test
```

Le projet contient un test de démarrage de l'application dans
[`src/test/java/ma/enset/tp_jwt_spring/TpJwtSpringApplicationTests.java`](src/test/java/ma/enset/tp_jwt_spring/TpJwtSpringApplicationTests.java).

---

## 9. Détail des composants

- **`AppUser`** — entité JPA correspondant à la table `users`, avec un email
  unique, un mot de passe haché et le rôle `USER`.
- **`CustomerUserDetailsServiceImpl`** — charge l'utilisateur par email et le
  convertit en `UserDetails` Spring Security.
- **`JwtService`** — fournit la génération, l'extraction des claims et la
  validation du JWT. La clé HMAC est décodée depuis le secret Base64.
- **`JwtFilter`** — extrait le Bearer token, le valide et renseigne le
  `SecurityContext`.
- **`SecurityConfig`** — configure `BCryptPasswordEncoder`,
  `AuthenticationManager`, la politique `STATELESS` et l'ordre des filtres.
- **`AuthController`** — gère `/auth/register` et `/auth/login`.
- **`UserController`** — expose les endpoints protégés `/api/hello` et
  `/api/profile`.
