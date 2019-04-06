# Personal website

[![Netlify Status](https://api.netlify.com/api/v1/badges/e3d7ca85-43cf-4223-aad2-310306336541/deploy-status)](https://app.netlify.com/sites/ivomarino/deploys)

https://gohugo.io/hosting-and-deployment/hosting-on-netlify/
	https://gohugo.io/getting-started/quick-start/

docker run --rm -it -v $(pwd):/src klakegg/hugo new site quickstart
cd quickstart
git init
git submodule add https://github.com/budparr/gohugo-theme-ananke.git themes/ananke

docker run --rm -it -v $(pwd):/src klakegg/hugo new posts/my-first-post.md

docker run --rm -it -v $(pwd):/src -p 1313:1313 klakegg/hugo server --log
