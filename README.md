# weather-appv2


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

```

docker buildx create --name mybuilder --use --driver docker-container
docker buildx inspect --bootstrap


```


```
echo TOKEN | docker login ghcr.io -u agatOgr --password-stdin


```

```
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  --ssh default \
  --cache-to=type=registry,ref=ghcr.io/agatogr/weather-appv2-cache,mode=max \
  --cache-from=type=registry,ref=ghcr.io/agatogr/weather-appv2-cache \
  -t ghcr.io/agatogr/weather-appv2:latest \
  --push \

```


```
cosign generate-key-pair
```
