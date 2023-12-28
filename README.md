# gpu-cluster

# Setup
- Install mkdocs-material
```
docker pull squidfunk/mkdocs-materials
```
- Create a site
```
docker run --rm -it -v ${PWD}:/docs squidfunk/mkdocs-material new .
```
- Preview
```
docker run --rm -it -p 8000:8000 -v ${PWD}:/docs squidfunk/mkdocs-material
```
- Build the site
```
docker run --rm -it -v ${PWD}:/docs squidfunk/mkdocs-material build
```
- Remove the site build
```
sudo rm site
```