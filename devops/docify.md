# docify 

## build docify docker image

```
FROM node:latest
LABEL description="A demo Dockerfile for build Docsify."
WORKDIR /docs
RUN npm install -g docsify-cli@latest
EXPOSE 3000/tcp
ENTRYPOINT docsify serve .
```

##  how to run

```
docker tag 1ed4090a7dfc 10.10.9.29:2443/library/docify:self
docker push 10.10.9.29:2443/library/docify:self
docker run -itp 3000:3000 --name=docsify -v $(pwd):/docs 10.10.9.29:2443/library/docify:self
```
