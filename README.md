# OpenShift: Distribuerad Gästbok med cache
Eli Fadi 
I denna labb ska ni bygga och deploya en modern, cloud-native applikation på OpenShift. Applikationen är en gästbok som demonstrerar:

- Multi-tier arkitektur
- Containerisering med multi-stage builds
- Configuration management (ConfigMaps & Secrets)
- Service discovery 
- Caching strategies
- Persistent storage
- External routing

## Arkitektur

```txt
Internet
    ↓
[Route] → [Frontend Service] → [Frontend Pod]
                                      ↓
                            [Backend Service] → [Backend Pod(s)]
                                                  ↓         ↓
                                            [Redis]   [PostgreSQL]
```


Den färdiga applikationen ser ut så här: [screencast.com](https://app.screencast.com/x8uWhUNAMZNQB)
w
## Container images

- registry.access.redhat.com/ubi10/go-toolset:10.0
- registry.access.redhat.com/ubi10-minimal:10.0
- registry.access.redhat.com/ubi10/nginx-126:10.0
- quay.io/kurs/redis:latest
- quay.io/fedora/postgresql-16:latest

## Backend

För att bygga backend behöver ni kunna bygga Golang:

```sh
$ go mod tidy
$ go build -o guestbook-api .
```

Backend är beroende av att PostgreSQL och Redis är igång och fungerar. För att backend-applikationen skall kunna köras behöver följande miljövariabler sättas. Värdena inom paranteserna är standardvärdena och kommer användas om du inte sätter miljövariablerna.

PostgreSQL:

- `DB_HOST` (localhost)
- `DB_PORT` (5432)
- `DB_USER` (guestbook)
- `DB_PASSWORD` (password)
- `DB_NAME` (guestbook)

Redis:

- `REDIS_HOST` (localhost)
- `REDIS_PORT` (6379)
- `REDIS_PASSWORD` ()

Applikationen lyssnar på:

- `PORT` (8080)

API-endpoints:

- `/health` GET
- `/api/entries` GET
- `/api/entries` POST
- `/api/stats` GET

### Testa backend

För att se om backend fungerar som den skall kan du köra följande kommandon:

- Testa om `/health` fungerar

```sh
$ curl localhost:8080/health
```

- Hämta alla inlägg

```sh
$ curl localhost:8080/api/entries
```

- Skapa ett nytt inlägg. `name` är namnet på den som skrivit inlägget och `message` är inlägget. I exemplet nedanför skriver Jonas meddelandet *Jonas testar API!*

```sh
$ curl -X POST localhost:8080/api/entries \
  -H "Content-Type: application/json" \
  -d '{"name":"Jonas","message":"Jonas testar API!"}'
```

- Hämta statistik

```sh
$ curl localhost:8080/api/stats
```

## Frontend

För att nginx på frontend skall kunna hitta backend måste vi ange att den skall använda 
OpenShift-klustrets DNS för namnuppslag. Då räcker det med att vår service heter `backend` 
och ligger i samma project som vi har frontend.

```nginx file=nginx.conf

    resolver dns-default.openshift-dns.svc.cluster.local valid=30s;
    resolver_timeout 5s;

    upstream backend {
        server backend:8080;
    }

```

Hela `nginx.conf`-filen finns här i repot.

## PostgreSQL

- `POSTGRESQL_USER`
- `POSTGRESQL_PASSWORD`
- `POSTGRESQL_DATABASE`
- `/var/lib/pgsql/data` är katalogen där PostgreSQL sparar data.

## Redis

- Sätt `REDIS_PASSWORD` till det lösenord du vill använda. Utan detta kommer inte backend kunna 
kommunicera med Redis!
- `/var/lib/redis/data` är katalogen där Redis sparar sin data.

## Checklist

- [ ] Alla 6 pods körs (2x backend, 2x frontend, 1x postgres, 1x redis)
- [ ] ConfigMaps och Secrets används korrekt
- [ ] Backend kan ansluta till både PostgreSQL och Redis
- [ ] Frontend kan kommunicera med backend via service
- [ ] Route exponerar applikationen externt
- [ ] Cache fungerar (verifiera X-Cache header med `curl -i`)
- [ ] Health checks fungerar
- [ ] Persistent storage används för PostgreSQL
- [ ] Kan skapa och läsa inlägg via webbgränssnittet (frontend applikationen)

## Reflektionsfrågor

1. Varför använder vi multi-stage builds?
2. Vad händer om Redis går ner? Funkar applikationen fortfarande?
3. Hur skulle ni implementera high availability för PostgreSQL?
4. Varför använder vi separate services för backend och frontend?
5. Vad är skillnaden mellan ClusterIP, NodePort och LoadBalancer?
6. Varför bör känsliga data ligga i Secrets istället för ConfigMaps?
7. Hur kan vi implementera horizontal pod autoscaling?

