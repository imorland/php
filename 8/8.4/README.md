docker build . --file 8/8.4/Dockerfile.apache --tag ianmgg/php84:latest
docker build . --file 8/8.4/Dockerfile.cli --tag ianmgg/php84:cli
docker build . --file 8/8.4/Dockerfile.apache.dev --tag ianmgg/php84:dev