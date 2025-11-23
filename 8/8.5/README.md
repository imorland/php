docker build . --file 8/8.5/Dockerfile.apache --tag ianmgg/php85:latest
docker build . --file 8/8.5/Dockerfile.cli --tag ianmgg/php85:cli
docker build . --file 8/8.5/Dockerfile.apache.dev --tag ianmgg/php85:dev

### Buildx

```
docker buildx build \
    --platform linux/amd64,linux/arm64 \
    --file 8/8.5/Dockerfile.apache \
    --tag ianmgg/php85:latest \
    --push .
```
