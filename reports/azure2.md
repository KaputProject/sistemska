# Poročilo o vzpostavitvi CI/CD v projektu skupine Kaput

## Docker Hub container registry

# Poročilo o vzpostavitvi CI/CD v projektu skupine Kaput

## Docker Hub Container Registry

V tem delu smo pripravili Docker sliko naše aplikacije in jo uspešno naložili v Docker Hub z uporabo CLI ukazov. Postopek je bil naslednji:

### 1. Prijava v Docker Hub

Za dostop do zasebnega Docker Hub repozitorija smo se najprej prijavili v svoj račun:

```bash
docker login
```

Po vnosu uporabniškega imena in gesla je bil dostop omogočen.

---

### 2. Gradnja Docker slike

Z ukazom `docker build` smo ustvarili sliko z imenom `spletno-frontend-dev`, ki bo služila kot kontejner za našo aplikacijo:

```bash
docker build -t davidgucci/spletno-frontend-dev:latest .
```

- `-t` označuje oznako slike (`tag`), ki mora vsebovati tudi naše Docker Hub uporabniško ime (`davidgucci`)
- `.` pomeni, da se `Dockerfile` nahaja v trenutni mapi

---

### 3. Pregled ustvarjenih slik

Sliko smo preverili z ukazom:

```bash
docker images
```

Izpis je pokazal novo sliko, npr.:

```
REPOSITORY                     TAG        IMAGE ID       CREATED         SIZE
davidgucci/spletno-frontend-dev   latest     abcdef123456   Few seconds ago   2.2GB
```

---

### 4. Nalaganje slike na Docker Hub

Ko je bila slika zgrajena in pravilno označena, smo jo potisnili (push) na Docker Hub:

```bash
docker push davidgucci/spletno-frontend-dev:latest
```

S tem je slika postala dostopna v repozitoriju na naslovu:
[https://hub.docker.com/repository/docker/davidgucci/spletno-frontend-dev](https://hub.docker.com/repository/docker/davidgucci/spletno-frontend-dev)

---




