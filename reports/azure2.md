# Poročilo o vzpostavitvi CI/CD v projektu skupine Kaput

Člani:
- Luka Kuder (vodja skupine)
- Enej Kramar
- David Guček

## Docker Hub Container Registry

V tem delu smo pripravili Docker sliko naše aplikacije in jo uspešno naložili v Docker Hub z uporabo CLI ukazov. Postopek je potekal na naslednji način:

### 1. Prijava v Docker Hub

Za dostop do zasebnega Docker Hub repozitorija smo se najprej prijavili v svoj račun:

```bash
docker login
```

Po vnosu uporabniškega imena in gesla je bil dostop omogočen.

---

### 2. Gradnja Docker slike

Ustvarili smo dva repozitorija: `spletno-frontend-dev` in `spletno-backend-dev`, ki vsebujeta različne verzije slik (latest, prod, testing).
Z ukazom `docker build` smo ustvarili slike, ki bodo služile kot kontejner za našo aplikacijo:
![img_4.png](img_4.png)

```bash
docker build -t davidgucci/spletno-frontend-dev:latest -f DockerFile.frontend frontend
docker build -t davidgucci/spletno-frontend-dev:prod -f DockerFile.frontend frontend
docker build -t davidgucci/spletno-frontend-dev:testing -f DockerFile.frontend frontend

docker build -t davidgucci/spletno-backend-dev:latest -f DockerFile.frontend frontend
docker build -t davidgucci/spletno-backend-dev:prod -f DockerFile.frontend frontend
docker build -t davidgucci/spletno-backend-dev:testing -f DockerFile.frontend frontend

-f backend/DockerFile.backend backend
```

- `-t` označuje oznako slike (`tag`), ki mora vsebovati tudi naše Docker Hub uporabniško ime (`davidgucci`)
- `.` pomeni, da se `Dockerfile` nahaja v trenutni mapi

---

### 3. Pregled ustvarjenih slik

Sliko smo preverili z ukazom:

```bash
docker images
```

Izpis je pokazal nove slike, ki smo jih pravkar zgradili:

![img_2.png](img_2.png)
![img_3.png](img_3.png)

---

### 4. Nalaganje slike na Docker Hub

Ko je bila slika zgrajena in pravilno označena, smo jo naložili na Docker Hub:

![img.png](img.png)

```bash
docker push davidgucci/spletno-frontend-dev:latest
docker push davidgucci/spletno-frontend-dev:prod
docker push davidgucci/spletno-frontend-dev:testing
docker push davidgucci/spletno-backend-dev:latest
docker push davidgucci/spletno-backend-dev:prod
docker push davidgucci/spletno-backend-dev:testing
```

S tem je slika postala dostopna na naslovu:  
[https://hub.docker.com/repository/docker/davidgucci/spletno-frontend-dev](https://hub.docker.com/repository/docker/davidgucci/spletno-frontend-dev)
Zdaj imamo repozitorija `spletno-frontend-dev` in `spletno-backend-dev`, ki vsebujeta različne verzije slik (latest, prod, testing).
---

## GitHub Actions Workflows

Za avtomatizacijo CI/CD smo uporabili GitHub Actions in ustvarili dva ločena workflowa za testno in produkcijsko okolje.

### Struktura projekta

```
/spletno
├── backend/
│   └── DockerFile.backend
├── frontend/
│   └── DockerFile.frontend
├── docker-compose.yml
├── Caddyfile
└── .github/workflows/
    ├── deploy-test.yml
    └── deploy-prod.yml
```

---

### Namen workflowov

Vzpostavljena sta bila naslednja workflowa:

- `deploy-dev.yml` → ob vsakem pushu na `develop` vejo
- `deploy-prod.yml` → ob vsakem pushu na `main` vejo

Ob vsakem sproženju se izvede:
1. gradnja Docker slik (frontend in backend),
2. nalaganje slik na Docker Hub,
3. pošiljanje webhook obvestila strežniku.

---

### `deploy-dev.yml`

```yaml
name: Deploy to Test Server

on:
  push:
    branches: [develop]

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout kodo
        uses: actions/checkout@v3

      - name: Prijava v Docker Hub
        run: echo "${{ secrets.DOCKER_PASSWORD }}" | docker login -u "${{ secrets.DOCKER_USERNAME }}" --password-stdin

      - name: Build frontend image (test)
        run: docker build -t ${{ secrets.DOCKER_USERNAME }}/spletno-frontend-dev:testing -f frontend/DockerFile.frontend frontend

      - name: Build backend image (test)
        run: docker build -t ${{ secrets.DOCKER_USERNAME }}/spletno-backend-dev:testing -f backend/DockerFile.backend backend

      - name: Push frontend image (test)
        run: docker push ${{ secrets.DOCKER_USERNAME }}/spletno-frontend-dev:testing

      - name: Push backend image (test)
        run: docker push ${{ secrets.DOCKER_USERNAME }}/spletno-backend-dev:testing

      - name: Pošlji webhook na test strežnik
        if: success()
        uses: distributhor/workflow-webhook@v3
        with:
          webhook_url: ${{ secrets.WEBHOOK_URL_TEST }}
          method: POST
          body: '{ "environment": "testing", "status": "updated" }'
          headers: |
            X-SECRET-TOKEN: ${{ secrets.SECRET_TOKEN }}
```

### `deploy-prod.yml`

```yaml
name: Deploy to Production Server

on:
  push:
    branches: [main]

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout kodo
        uses: actions/checkout@v3

      - name: Prijava v Docker Hub
        run: echo "${{ secrets.DOCKER_PASSWORD }}" | docker login -u "${{ secrets.DOCKER_USERNAME }}" --password-stdin

      - name: Build frontend image (prod)
        run: docker build -t ${{ secrets.DOCKER_USERNAME }}/spletno-frontend-dev:prod -f frontend/DockerFile.frontend frontend

      - name: Build backend image (prod)
        run: docker build -t ${{ secrets.DOCKER_USERNAME }}/spletno-backend-dev:prod -f backend/DockerFile.backend backend

      - name: Push frontend image (prod)
        run: docker push ${{ secrets.DOCKER_USERNAME }}/spletno-frontend-dev:prod

      - name: Push backend image (prod)
        run: docker push ${{ secrets.DOCKER_USERNAME }}/spletno-backend-dev:prod

      - name: Pošlji webhook na production strežnik
        if: success()
        uses: distributhor/workflow-webhook@v3
        with:
          webhook_url: ${{ secrets.WEBHOOK_URL_PROD }}
          method: POST
          body: '{ "environment": "production", "status": "updated" }'
          headers: |
            X-SECRET-TOKEN: ${{ secrets.SECRET_TOKEN }}
```
---

### Ustvarjanje GitHub Personal Access Tokena (PAT) za prijavo

GitHub Personal Access Token (PAT) je varnostni ključ, ki omogoča varno in avtomatizirano prijavo v GitHub API ter izvajanje ukazov, kot so kloniranje repozitorijev, potiskanje sprememb ali dostop do zasebnih repozitorijev, brez potrebe po vnosu uporabniškega imena in gesla. Token smo ustvarili znotraj nastavitev GitHub računa, kjer lahko natančno določiš obseg dovoljenj, ki jih token vsebuje, kar pomeni, da lahko omejiš njegove pravice le na tiste funkcije, ki jih tvoja aplikacija ali skripta potrebuje. Uporaba PAT-ja je varnejša kot shranjevanje gesla, saj ga lahko kadar koli prekličeš ali zamenjaš brez vpliva na tvoj glavni račun, poleg tega pa je token enostavno integrirati v avtomatizirane procese, kot so GitHub Actions ali skripte za neprekinjeno integracijo in dostavo (CI/CD).

### Ustvarjanje Docker Personal Access Tokena (PAT) za prijavo

Docker Personal Access Token je poseben varnostni žeton, ki omogoča varno prijavo v Docker Hub brez uporabe gesla. Uporablja se predvsem v avtomatiziranih okoljih, kot so CI/CD sistemi ali skripte, ki morajo samodejno dostopati do Docker repozitorijev za poteg ali objavo slik. Token smo ustvarili znotraj varnostnih nastavitev Docker Hub računa, kjer smo ga poimenovali in mu dodelili ustrezne pravice. Uporaba tokena omogoča boljši nadzor in varnost, saj se geslo računa ne deli z zunanjimi orodji, hkrati pa lahko token enostavno shranite kot okoljsko spremenljivko in ga vključite v avtomatizirane procese, kar bistveno zmanjša tveganje nepooblaščenega dostopa in omogoča fleksibilnejše upravljanje dostopa do vaših Docker virov.
![img_1.png](img_1.png)

### GitHub Secrets uporabljeni

| Ime                | Namen                                |
|--------------------|--------------------------------------|
| `DOCKER_USERNAME`  | Uporabniško ime za Docker Hub        |
| `DOCKER_PASSWORD`  | Geslo za Docker Hub (PAT)            |
| `WEBHOOK_URL_TEST` | Webhook URL za testni strežnik       |
| `WEBHOOK_URL`      | Webhook URL za produkcijski strežnik |

Secrets se dodajo pod:  
**Repo > Settings > Secrets and variables > Actions > New secret**
![img_5.png](img_5.png)

---

### Dodatni možni workflowi

#### Testiranje (Continuous Integration)
- `test.yml` workflow:
    - `npm test` za backend
    - `npm run test` za frontend

#### Linting (preverjanje kode)
- ESLint (frontend)
- flake8 (backend)

#### Formatiranje
- Prettier (JS/TS)
- Husky (pre-commit hook)

#### Varnostne prakse
- `npm audit`
- `docker scan`

#### Obvestila
- Discord/Slack webhook za status `build`/`deploy`

---

## GitHub Actions Workflow

### Namen

Vzpostavljen je bil GitHub Actions workflow za avtomatsko gradnjo Docker slik, njihov prenos na Docker Hub ter sprožitev webhook sporočila, ki omogoča avtomatski deploy na strežniku.
![img_6.png](img_6.png)

### Struktura workflowa (`.github/workflows/deploy-prod.yml`)

```yaml
name: Deploy Production

on:
  push:
    branches:
      - main

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Log in to Docker Hub
        uses: docker/login-action@v2
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Build and push backend image
        run: |
          docker build -t ${{ secrets.DOCKER_USERNAME }}/kaput-backend:latest -f backend/DockerFile.backend .
          docker push ${{ secrets.DOCKER_USERNAME }}/kaput-backend:latest

      - name: Build and push frontend image
        run: |
          docker build -t ${{ secrets.DOCKER_USERNAME }}/kaput-frontend:latest -f frontend/DockerFile.frontend .
          docker push ${{ secrets.DOCKER_USERNAME }}/kaput-frontend:latest

  trigger-webhook:
    needs: build-and-push
    runs-on: ubuntu-latest
    steps:
      - name: Pošlji webhook na production strežnik
        uses: distributhor/workflow-webhook@v3
        with:
          webhook_url: ${{ secrets.WEBHOOK_URL_PROD }}
          method: POST
          body: '{ "environment": "production", "status": "updated" }'
          headers: |
            X-SECRET-TOKEN: ${{ secrets.SECRET_TOKEN }}
```

### Struktura workflowa (`.github/workflows/deploy-test.yml`)

```yaml
name: Deploy to Test Server

on:
  push:
    branches: [develop]

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout kodo
        uses: actions/checkout@v3

      - name: Prijava v Docker Hub
        run: echo "${{ secrets.DOCKER_PASSWORD }}" | docker login -u "${{ secrets.DOCKER_USERNAME }}" --password-stdin

      - name: Build frontend image (test)
        run: docker build -t ${{ secrets.DOCKER_USERNAME }}/spletno-frontend-dev:testing -f frontend/DockerFile.frontend frontend

      - name: Build backend image (test)
        run: docker build -t ${{ secrets.DOCKER_USERNAME }}/spletno-backend-dev:testing -f backend/DockerFile.backend backend

      - name: Push frontend image (test)
        run: docker push ${{ secrets.DOCKER_USERNAME }}/spletno-frontend-dev:testing

      - name: Push backend image (test)
        run: docker push ${{ secrets.DOCKER_USERNAME }}/spletno-backend-dev:testing

      - name: Pošlji webhook na test strežnik
        if: success()
        uses: distributhor/workflow-webhook@v3
        with:
          webhook_url: ${{ secrets.WEBHOOK_URL_TEST }}
          method: POST
          body: '{ "environment": "testing", "status": "updated" }'
          headers: |
            X-SECRET-TOKEN: ${{ secrets.SECRET_TOKEN }}

```
### Job za testiranje zaledja

Ta test preverja delovanje /users/echo endpointa. Pošlje podatke prek POST metode in pričakuje, da strežnik vrne enake podatke znotraj objekta received. Cilj je zagotoviti, da endpoint pravilno prejme in vrne poslani JSON objekt.

```
const request = require('supertest');
const app = require('../server');

describe('Tech for users/echo', () => {
it('Should echo back the data sent', async () => {
const testData = { message: 'User Luka K.', number: 42 };

        const res = await request(app)
            .post('/users/echo')
            .send(testData);

        expect(res.statusCode).toBe(200);
        expect(res.body).toEqual({ received: testData });
    });
});
```

### Možne razširitve workflowa

- **Avtomatsko testiranje**: Dodaj job za `npm test`, `pytest`, `jest` ipd.
- **Lintanje**: Dodaj linting korake (`eslint`, `black`, `flake8`, `prettier`...).
- **CI za več branch-ev**: Dodaj testiranje na vseh branchih, deploy pa samo za `main`.
- **Varnostne preverbe**: Uporabi `npm audit`, `safety`, `trivy`, `bandit` ipd.

---


