# TP 2: Authentification avec JWT et Spring Security

Application Spring Boot démontrant une authentification basée sur JSON Web Token
(JWT) avec Spring Security et une base de données H2.

## Prérequis

- Java 21
- Maven (ou le wrapper Maven fourni)

## Démarrage

```bash
./mvnw spring-boot:run
```

Sous Linux, si le wrapper n'est pas exécutable :

```bash
bash mvnw spring-boot:run
```

L'application est disponible sur `http://localhost:8080`.

## Endpoints principaux

- `POST /auth/register` : créer un compte
- `POST /auth/login` : obtenir un token JWT
- Endpoints utilisateur protégés par Spring Security
- Console H2 : `http://localhost:8080/h2-console`

Les exemples de requêtes sont disponibles dans [`requests.http`](requests.http).
