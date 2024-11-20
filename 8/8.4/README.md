docker build . --file 8/8.4/Dockerfile.apache --tag ianmgg/php84:latest
docker build . --file 8/8.4/Dockerfile.cli --tag ianmgg/php84:cli
docker build . --file 8/8.4/Dockerfile.apache.dev --tag ianmgg/php84:dev

### Buildx

```
docker buildx build \
    --platform linux/amd64,linux/arm64 \
    --file 8/8.4/Dockerfile.apache \
    --tag ianmgg/php84:latest \
    --push .
```
