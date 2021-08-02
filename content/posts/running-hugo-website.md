---
title: "Running Hugo Website"
date: 2021-08-01T21:27:19-04:00
tags: ['hugo', 'vps', 'website', 'blog', 'github actions']
---

In order to run a hugo webserver you will need a few things in order first.
I will mention the requirements; however, I will not be going into detail here.

If you would like me to write a post on these pre-requisites. Please let me know in the comments!

---

### Pre-Requisites

| Service | Provider |
| ------- | -------- |
| Virtual Private Server | [Vultr](https://www.vultr.com/?ref=8910980) |
| Domain Name Registrar | [Epik](https://www.epik.com/?affid=ve9na3ca0) |
| DNS | [Cloudflare](https://dash.cloudflare.com/login) |
| Git | [GitHub](https://github.com) |

---

### Domain, VPS, and nginx Configuration

Setup your A Record to point your domain name to your webserver

| type | name | content |
| ---- | ---- | ------- |
| A | @ | vps_ipv4_address |

[nginx install](https://nginx.org/en/linux_packages.html)

Create simple nginx configuration file, /etc/nginx/conf.d/<domain>.conf

```bash
server {
    server_name  <domain-name> www.<domain-name>;
    root         /var/www/<domain-name>/public;

    index index.html;

    location / {
        try_files $uri $uri/ =404;
        add_header Permissions-Policy interest-cohort=();
    }

    listen       80;
    listen       [::]:80;
    return 404; 
}
```

Create a temporary index.html in your website's root directory, /var/www/<domain>/public/index.html
```bash
<html>
<p><domain></p>
</html>
```

Start and Enable nginx
```bash
systemctl enable --now nginx.service
```

If necessary, allow HTTP and HTTPS through firewall
```bash
firewall-cmd --add-service=http --permanent
firewall-cmd --add-service=https --permanent
firewall-cmd reload
```

You should be able to verify access to your webserver over port 80 at this point.

After a successful verification you can continue finishing the configuration.

---

### SSL with acme.sh

For SSL you can find all Let's Encrypt client options [here](https://letsencrypt.org/docs/client-options/)

I recommend [acme.sh](https://github.com/acmesh-official/acme.sh)

Install acme.sh

[get.acme.sh](https://github.com/acmesh-official/get.acme.sh)

Issue cert for your domain
```bash
acme.sh --issue -d <domain> -w /var/lib/nginx/<domain>
```

Install certificate to an nginx directory
```bash
acme.sh --install-cert -d <domain> \
--key-file       /etc/nginx/<domain>/key.pem  \
--fullchain-file /etc/nginx/<domain>/cert.pem \
--reloadcmd     "service nginx force-reload"
```

acme registers a cron job to run daily. This will ensure your certificate is always valid and will restart your webserver.

---

### Create Hugo Website

[Hugo Installation](https://gohugo.io/getting-started/installing)

[Locate a Hugo theme you'd like to use](https://themes.gohugo.io/)

I'm using [PaperMod](https://github.com/adityatelange/hugo-PaperMod)

PaperMod installation instructions are [here](https://github.com/adityatelange/hugo-PaperMod/wiki/Installation#guide)

Once you have followed the installation instructions for your theme you can begin customizing your site.

Again, each theme has their own configuration options; however, I'll explain the general layout of the hugo folder structure to help you understand where to go.

Ensure you are inside the websites directory.

```bash
cd ~/Projects/davidspencer.xyz

tree -L 1 .
.
├── archetypes
├── config.yml
├── content
├── layouts
├── resources
├── static
└── themes
```

Most of the configuration will be done in the config.yml file
You can see my exact configuration at my [repo](https://github.com/Elevate08/davidspencer.xyz)

```yml
baseURL: https://davidspencer.xyz
languageCode: en-us
title: Title of Website
theme: Theme Name
env: production

outputs:
  home:
    - HTML
    - RSS
    - JSON

params:
  assets:
    disableHLJS: true
    disableFingerprinting: false
    favicon: "/favicon.ico>"
    favicon16x16: "/favicon-16x16.png"
    favicon32x32: "/favicon-32x32.png"
    apple_touch_icon: "/apple-touch-icon.png"
    safari_pinned_tab: "/safari-pinned-tab.svg"

  socialIcons:
    - name: "linkedin"
      url: "https://www.linkedin.com/in/davidjspencer/"
    - name: "github"
      url: "https://github.com/Elevate08"
    - name: "email"
      url: "mailto:david@davidspencer.xyz"
    - name: "rss"
      url: "/index.xml"

  editPost:
      URL: "https://github.com/Elevate08/davidspencer.xyz/blob/master/content"
      Text: "Suggest Changes" # edit text
      appendFilePath: true # to append file path to Edit link

  fuseOpts:
    minMatchCharLength: 3
    keys: ["title", "content", "summary"]

menu:
  main:
    - identifier: tags
      name: tags
      url: /tags/
      weight: 20
    - identifier: search
      name: search
      url: /search/
      weight: 30

markup:
  highlight:
    codeFences: true
    guessSyntax: false
    style: native
    tabWidth: 4
```

Once you have some settings in place for your site you can create a new post.

```bash
hugo new posts/first-post.md
```

Using your editor of choice, write up your first post using Markdown.

You can learn about Markdown here
[Markdown Guide](https://www.markdownguide.org/)

Once you have some content created run the following command

```bash
hugo server

Watching for config changes in /home/elevate/Projects/davidspencer.xyz/config.yml
Environment: "development"
Serving pages from memory
Running in Fast Render Mode. For full rebuilds on change: hugo server --disableFastRender
Web Server is available at http://localhost:1313/ (bind address 127.0.0.1)
```

You should now be able to access your hugo website at the location provided.

This is how most development / testing of new content can be done.

Click around and test out your new site.

---

### Deploying Hugo Website using GitHub Actions

Up until now we have just been working locally. You should have a working development environment where you can access your site locally.

To make deployment to your VPS easier and allow you to focus on just creating new content, let's setup some GitHub Actions to do some work for us!

```bash
cd ~/Projects/davidspencer.xyz
git init
```

Usually at this point I'd run ```git add .```; however, you most likely need to configure the git submodule for your theme.

We want to be able to pull a fresh version from the repo every time we deploy.

You should be able to run the following command without error.

```
git submodule add <theme clone url>
```

If you do run into an error let me know in the comments and I'll help you troubleshoot.

Once the theme repo is properly added as a submodule we can continue.

At this time you should login to GitHub and create a new repo. Grab the clone URL.

```bash
git add .
git commit -m "initial upload"
git remote add origin git@github.com:/Elevate08/davidspencer.xyz
git push -u origin master
```

Now to make our github action file, we will need to create a couple directories.

```bash
mkdir -p .github/workflows
touch .github/workflows/main.yml
```

Add the following text to your main.yml file

```yml
name: Hugo to VPS

on:
  push:
    branches:
      - master  # Set a branch to deploy
  pull_request:

jobs:
  deploy:
    runs-on: ubuntu-20.04
    steps:
      - uses: actions/checkout@v2
        with:
          submodules: true  # Fetch Hugo themes (true OR recursive)
          fetch-depth: 0    # Fetch all history for .GitInfo and .Lastmod

      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v2
        with:
          hugo-version: 'latest'
          # extended: true

      - name: Build
        run: hugo --minify

      - name: Deploy
        uses: appleboy/scp-action@master
        with:
          host: ${{ secrets.HOST }}
          port: ${{ secrets.PORT }}
          username: ${{ secrets.USERNAME }}
          key: ${{ secrets.KEY }}
          source: "./public"
          target: "/var/www/davidspencer.xyz"
```

In my workflow, i'm using scp-action to copy the generated website files to my server.

This means we'll need to create some secrets on our repo.

In your repository on GitHub create the following secrets
Settings > Secrets > New repository secret

- HOST
- KEY
- PORT
- USERNAME

I'll assume you have knowledge on SSH and what the secrets should contain.

I'd encourage you to generate a unique key pair and user for this action.

```bash
cd ~/Projects/davidspencer.xyz
git add .
git commit -m "adding github action for automated deployments"
git push
```

You can now watch the action run on GitHub and monitor your VPS for the website files to be copied over.
