# weather-appv2

# Część nieobowiązkowa (3. (max +80%))

## Cel projeku

Celem zadania było zbudowanie obrazu kontenera zgodnego z OCI, zawierającego aplikację opracowaną w części obowiązkowej projektu, z przeznaczeniem na platformy sprzętowe linux/amd64 oraz linux/arm64. Proces budowania miał wykorzystywać zaawansowane funkcje Docker BuildKit, w tym wsparcie dla mount ssh, mount secret, system cache oparty o rejestr (registry) oraz podpisywanie i weryfikację obrazu przy użyciu narzędzia cosign. Obraz został zweryfikowany pod kątem podatności z wykorzystaniem Docker Scout, ograniczając wyniki do luk o poziomie krytycznym i wysokim.

## Wykonane czynności

### Generowanie klucza SSH i konfiguracja połączenia z GitHub

Aby umożliwić pobranie kodu źródłowego z prywatnego repozytorium w trakcie budowania obrazu, wygenerowano parę kluczy SSH i dodano ją do agenta SSH:

```
ssh-keygen -t ed25519 -C "agata.ogrodnik13@gmail.com"
```

```
ssh -T git@github.com
```


```
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519

```

### Utworzenie własnego buildera docker buildx z obsługą kontenerów

```

docker buildx create --name mybuilder --use --driver docker-container
docker buildx inspect --bootstrap


```
### Logowanie do GitHub Container Registry (GHCR)

```
echo TOKEN | docker login ghcr.io -u agatOgr --password-stdin


```

### Budowa obrazu z użyciem buildx, cache i wsparciem dla wielu architektur

```
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  --ssh default \
  --cache-to=type=registry,ref=ghcr.io/agatogr/weather-appv2-cache,mode=max \
  --cache-from=type=registry,ref=ghcr.io/agatogr/weather-appv2-cache \
  -t ghcr.io/agatogr/weather-appv2:latest \
  --push \

```
<img width="952" alt="image" src="https://github.com/user-attachments/assets/0a06c335-bd9e-42a9-ab7c-d495a33f2f3f" />

### Podpisanie obrazu za pomocą cosign

```
cosign generate-key-pair
```


```
cosign sign --key cosign.key ghcr.io/agatogr/weather-appv2:latest
```
### Weryfikacja podpisu cyfrowego obrazu

```
cosign verify --key cosign.pub ghcr.io/agatogr/weather-appv2:latest
```

<img width="737" alt="image" src="https://github.com/user-attachments/assets/2c632c55-8d0d-43aa-97f7-7967c8774cfc" />

### Inspekcja manifestu obrazu

Potwierdzono obecność manifestów dla platform linux/amd64 oraz linux/arm64.

```
% docker buildx imagetools inspect ghcr.io/agatogr/weather-appv2:latest

Name:      ghcr.io/agatogr/weather-appv2:latest
MediaType: application/vnd.oci.image.index.v1+json
Digest:    sha256:af779ad712edd20d0e4688f59e592ea3ae34d12c3c43409cc9dc04d808df6fbd
           
Manifests: 
  Name:        ghcr.io/agatogr/weather-appv2:latest@sha256:9a3dfd07f5beeabf8bad56014730e7c7dcf0ff30e1c32e548db0eb48fe96f9c9
  MediaType:   application/vnd.oci.image.manifest.v1+json
  Platform:    linux/amd64
               
  Name:        ghcr.io/agatogr/weather-appv2:latest@sha256:fded4bce0a4dc45d38b80574d1cc05cf83bbe234020cf476825fe28ddce72ccc
  MediaType:   application/vnd.oci.image.manifest.v1+json
  Platform:    linux/arm64
               
  Name:        ghcr.io/agatogr/weather-appv2:latest@sha256:4d032586ac415dcda793dac2e9e910deb0fe9d1e9e6c674294779e350425f371
  MediaType:   application/vnd.oci.image.manifest.v1+json
  Platform:    unknown/unknown
  Annotations: 
    vnd.docker.reference.digest: sha256:9a3dfd07f5beeabf8bad56014730e7c7dcf0ff30e1c32e548db0eb48fe96f9c9
    vnd.docker.reference.type:   attestation-manifest
               
  Name:        ghcr.io/agatogr/weather-appv2:latest@sha256:3e5cd4158c19be9d1e4786769cda29e97311a2b158905df0160643972cc0ac2d
  MediaType:   application/vnd.oci.image.manifest.v1+json
  Platform:    unknown/unknown
  Annotations: 
    vnd.docker.reference.digest: sha256:fded4bce0a4dc45d38b80574d1cc05cf83bbe234020cf476825fe28ddce72ccc
    vnd.docker.reference.type:   attestation-manifest
agataogrodnik@Mac weather-appv2 % 
```
### Skany bezpieczeństwa z użyciem Docker Scout

```
docker scout cves ghcr.io/agatogr/weather-appv2:latest --only-severity critical,high
```

Obraz został przeskanowany za pomocą narzędzia Docker Scout pod kątem znanych podatności o wysokim i krytycznym poziomie zagrożenia (--only-severity critical,high). Analiza obrazu ghcr.io/agatogr/weather-appv2:latest dla platformy linux/arm64 wykazała, że żaden z 294 zainstalowanych pakietów nie zawierał znanych luk bezpieczeństwa. Oznacza to, że obraz jest aktualny i bezpieczny do użycia.

<img width="840" alt="image" src="https://github.com/user-attachments/assets/58417816-0c63-45d5-9cd7-58a63a9cb38d" />


## Plik Dockerfile

```
# Używamy rozszerzonego frontendu BuildKit (v1.4), aby móc korzystać z funkcji takich jak --mount=type=ssh
# Dzięki temu możemy pobierać repozytorium prywatne przez SSH
#syntax=docker/dockerfile:1.4

# --- Etap 1: pobieranie repo przez SSH ---
FROM alpine:latest AS fetch_repo

# Ustawiamy katalog roboczy, w którym sklonujemy repozytorium
WORKDIR /repo

# Instalujemy Git oraz klienta SSH (potrzebne do pobierania repo z GitHuba przez SSH)
RUN apk add --no-cache openssh-client git

# Konfigurujemy zaufane hosty SSH, aby nie trzeba było potwierdzać połączenia z GitHubem
RUN mkdir -p -m 0600 ~/.ssh && ssh-keyscan github.com >> ~/.ssh/known_hosts

# Montujemy klucz SSH z zewnątrz i klonujemy repozytorium z aplikacją z GitHuba
RUN --mount=type=ssh git clone --branch main git@github.com:agatOgr/weather-appv2.git .

# --- Etap 2: budowanie aplikacji Node.js ---
FROM node:22-alpine3.20 AS builder

# Tworzymy katalog roboczy w obrazie buildera
WORKDIR /app

# Kopiujemy potrzebne pliki z wcześniej pobranego repozytorium
COPY --from=fetch_repo /repo/package.json .
COPY --from=fetch_repo /repo/package-lock.json .
COPY --from=fetch_repo /repo/server.js .
COPY --from=fetch_repo /repo/public ./public

# Instalujemy zależności aplikacji
RUN npm install

# Instalujemy dodatkowy pakiet do uruchamiania procesów podrzędnych
RUN npm install cross-spawn@7.0.5

# --- Etap 3: finalny obraz aplikacji ---
FROM node:22-alpine3.20

# Dodajemy metadane (np. autor obrazu) zgodne ze specyfikacją OCI
LABEL org.opencontainers.image.authors="Agata Ogrodnik <agata.ogrodnik13@gmail.com>"

# Tworzymy katalog roboczy dla aplikacji w obrazie produkcyjnym
WORKDIR /app

# Kopiujemy zbudowaną aplikację z etapu buildera
COPY --from=builder /app .

# Otwieramy port 3000 (na którym nasłuchuje nasza aplikacja)
EXPOSE 3000

# Dodajemy prosty mechanizm sprawdzania stanu aplikacji (czy działa serwer HTTP)
HEALTHCHECK --interval=5s --timeout=3s --start-period=3s --retries=2 \
  CMD wget -q --spider http://localhost:3000/ || exit 1

# Domyślne polecenie uruchamiające aplikację
CMD ["node", "server.js"]

```

