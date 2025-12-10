Eli Fadi DevOps 24


Guestbook edited – GitHub Actions + OpenShift - NEW steps dated 251210

<img width="298" height="281" alt="Screenshot 2025-12-10 110822" src="https://github.com/user-attachments/assets/836c1ab6-a833-47a0-9e46-f87224e765ec" />


This repo-EDITING contains my guestbook app from the previous GuestBook OpenShift lab, but wired up with a real CI/CD flow:

- Backend (Go API)
- Frontend (Nginx serving the static UI)
- Postgres and Redis as data + cache
- GitHub Actions builds images and pushes them to GitHub Container Registry (GHCR)
- OpenShift pulls those images and runs everything in my eli-dev project

  

What i did:

1. Move images to GitHub Container Registry

Before this work, the images were built by OpenShift and stored in its internal registry, with image names like:

     image: image-registry.openshift-image-registry.svc:5000/eli-dev/backend:latest
     image: image-registry.openshift-image-registry.svc:5000/eli-dev/frontend:latest


Today i changed the setup so that GitHub builds the images and stores them in GHCR instead.

The new image references in the YAML look like this:

# openshift/backend.yaml
containers:
         - name: backend
         image: ghcr.io/<github-owner>/guestbook-backend:latest
         imagePullPolicy: Always
         ...

<img width="992" height="739" alt="Screenshot 2025-12-10 104520" src="https://github.com/user-attachments/assets/65f71bcc-aeb2-432e-adb3-fcf8a464761d" />

         

# openshift/frontend.yaml
containers:
          - name: frontend
          image: ghcr.io/<github-owner>/guestbook-frontend:latest
          imagePullPolicy: Always
          ...

<img width="987" height="921" alt="Screenshot 2025-12-10 105207" src="https://github.com/user-attachments/assets/fd8cd48e-ee2c-4523-914a-bc1f12d48793" />



In my case fashinski is my GitHub username, so for me it looks like:

    image: ghcr.io/fashinski/guestbook-backend:latest
    image: ghcr.io/fashinski/guestbook-frontend:latest


By doing this, OpenShift no longer depends on its own registry. It just pulls whatever GHCR tag we push from the pipeline.

If the GHCR repos are public, OpenShift can pull them without extra secrets.
If they are private, you d add an imagePullSecret and point the deployments at that secret.




2. Add GitHub Actions workflow
   
I added a workflow file at:

    .github/workflows/guestbook-ci-cd.yml


<img width="1484" height="965" alt="image" src="https://github.com/user-attachments/assets/8ee8f012-9e94-45a5-8a49-fb7af22685eb" />


The workflow has two jobs:

- Job 1 – 
- Build and push images to GHCR
- Triggers on push to the main branch
- Checks out the repo
- Logs in to GHCR using GITHUB_TOKEN
- Builds and pushes backend image:
         ghcr.io/${{ github.repository_owner }}/guestbook-backend:latest
- Builds and pushes frontend image:
         ghcr.io/${{ github.repository_owner }}/guestbook-frontend:latest


So every time I push to main, Github builds fresh images for both services and updates the :latest tag in GHCR.

- Job 2 – 
Deploy to OpenShift
This job runs after the images are built.
What it does:

- Checks out the repo again (so it has the YAML files).
- Installs the OpenShift oc CLI on the GitHub runner:

      - name: Install OpenShift oc client
        uses: redhat-actions/openshift-tools-installer@v1
        with:
          oc: 'latest'


Logs in to my OpenShift cluster using a token stored as repo secrets:

    - name: Login to OpenShift
      uses: redhat-actions/oc-login@v1
      with:
        openshift_server_url: ${{ secrets.OC_SERVER }}
        openshift_token: ${{ secrets.OC_TOKEN }}
        namespace: ${{ secrets.OC_NAMESPACE }}
        insecure_skip_tls_verify: true


Applies the manifests:

    oc apply -f openshift/postgres.yaml  -n "${OC_NAMESPACE}"
    oc apply -f openshift/redis.yaml     -n "${OC_NAMESPACE}"
    oc apply -f openshift/backend.yaml   -n "${OC_NAMESPACE}"
    oc apply -f openshift/frontend.yaml  -n "${OC_NAMESPACE}"


Restart backend and frontend deployments so Openshift pulls the new GHCR images:

    oc rollout restart deployment/backend  -n "${OC_NAMESPACE}"
    oc rollout restart deployment/frontend -n "${OC_NAMESPACE}"

    
    

3. Required secrets

In the GitHub repo, under:

    Settings → Secrets and variables → Actions

I created these secrets:

    OC_SERVER – OpenShift API URL

    OC_TOKEN – my login token (service account or personal token)

    OC_NAMESPACE – the project name in OpenShift (for me: eli-dev)

GITHUB_TOKEN is already provided by Github, so i didn’t need to create that one.


<img width="1426" height="912" alt="Screenshot 2025-12-10 112107" src="https://github.com/user-attachments/assets/deedf52e-7c40-4ac6-b77b-6617569ef7ba" />



4. How the flow works now

End to end flow after all changes:

- I change something in the code or YAML and push to main.

- GitHub Actions:

      Builds backend + frontend images from the repo

      Pushes them to GHCR

- The deploy job:

      Logs in to OpenShift

      Applies the latest YAML manifests

      Restarts the guestbook deployments

- OpenShift pulls the new images from GHCR and runs the updated version of the app.

So the “contract” between the pipeline and the cluster is basically:

- Workflow pushes to:

      ghcr.io/fashinski/guestbook-backend:latest
      ghcr.io/fashinski/guestbook-frontend:latest

- Deployments pull from the same tags, which we wired into backend.yaml and frontend.yaml.



  

5. Manual checks

If I want to double check things from my laptop, i can still use oc:

# check deployments
oc get deploy -n eli-dev

# check pods
oc get pods -n eli-dev

# check route / app
oc get routes -n eli-dev

<img width="1379" height="929" alt="image" src="https://github.com/user-attachments/assets/5014e86b-0cc7-41c4-ab0c-188ef735f260" />



And of course, i can open the route in the browser and post a new guestbok entry to see that everything still talks nicely: frontend → backend → Postgres/Redis.

That’s the story of todays work: i moved image building from OpenShift to GitHub, pointed the YAML to GHCR, and wired it all together with a simple CI/CD pipeline.



<img width="1828" height="887" alt="Screenshot 2025-12-10 123243" src="https://github.com/user-attachments/assets/3ee1e9dc-31ad-4dcd-b9a9-9738479860a6" />

-

<img width="1840" height="774" alt="Screenshot 2025-12-10 123453" src="https://github.com/user-attachments/assets/c221a938-1411-4f79-8933-e363cb7bf0bc" />

-


<img width="1827" height="916" alt="Screenshot 2025-12-10 124004" src="https://github.com/user-attachments/assets/c834b157-40fe-402b-9e28-332365be3a37" />

-


End for job dated 251012

-

------------------------------------------------------------------------------------

-


# OpenShift: Distribuerad Gästbok med cache

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

