# weather-appv2

# Część nieobowiązkowa zadanie 1 (3. (max +80%))

## Cel projeku

Celem zadania było zbudowanie obrazu kontenera zgodnego z OCI, zawierającego aplikację opracowaną w części obowiązkowej projektu, z przeznaczeniem na platformy sprzętowe linux/amd64 oraz linux/arm64. Proces budowania miał wykorzystywać zaawansowane funkcje Docker BuildKit, w tym wsparcie dla mount ssh, mount secret, system cache oparty o rejestr (registry) oraz podpisywanie i weryfikację obrazu przy użyciu narzędzia cosign.

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


```
cosign generate-key-pair
```


```
cosign sign --key cosign.key ghcr.io/agatogr/weather-appv2:latest
```

```
cosign verify --key cosign.pub ghcr.io/agatogr/weather-appv2:latest
```

<img width="737" alt="image" src="https://github.com/user-attachments/assets/2c632c55-8d0d-43aa-97f7-7967c8774cfc" />


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


```
docker scout cves ghcr.io/agatogr/weather-appv2:latest --only-severity critical,high
```
<img width="840" alt="image" src="https://github.com/user-attachments/assets/58417816-0c63-45d5-9cd7-58a63a9cb38d" />

