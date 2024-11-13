docker build . --file 8/8.3/Dockerfile.apache --tag ianmgg/php83:latest
docker build . --file 8/8.3/Dockerfile.cli --tag ianmgg/php83:cli
docker build . --file 8/8.3/Dockerfile.apache.dev --tag ianmgg/php83:dev

### Buildx

```
docker buildx build \
    --platform linux/amd64,linux/arm64 \
    --file 8/8.3/Dockerfile.apache \
    --tag ianmgg/php83:latest \
    --push .
```
