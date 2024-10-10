# Linloir's blog

[Blog](https://blog.linloir.cn) about me.

Powered by [Hexo](https://hexo.io/) and [Butterfly](https://github.com/jerryc127/hexo-theme-butterfly).

## Build and Deploy

The repo is equipped with [Gitea Actions](https://docs.gitea.io/en-us/actions/) for CI/CD, which will automatically build the blog and triggers an update to the hosting server

Unbaked raw contents lies in the `main` branch, where all the blogs are written.

After editing the blog and deciding to publish it, create and push a new tag starting with `v` on the `main` branch.

Gitea Actions will take it from there, generating all the files needed, push to the `publish` branch, and calls on the `Caddy-Git` plugin for an update.

## License

All blog posts are licensed under [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/).

The codes in `themes` folder are not modified and inherit the license from their original authors. The work of this blog does not include any codes in `themes` folder.

Workflow of this blog is licensed under [MIT](https://opensource.org/licenses/MIT).

## Credits

- [Hexo](https://hexo.io/) for blog infrastructure
- [Butterfly](https://github.com/jerryc127/hexo-theme-butterfly) for blog theme
- [Gitea](https://gitea.io/) for self-hosted git service
- [Caddy](https://caddyserver.com/) for reverse proxy and web server
- [Caddy-Git](https://github.com/greenpau/caddy-git) for git integration with Caddy which allows me to serve my repo as a website
- [Cloudflare](https://www.cloudflare.com/) for DNS and CDN (providing v4-v6 proxy)
- [Mac Mini](https://www.apple.com/mac-mini/) for hosting the blog in my home network

## Contact

- Email: `jonathanzhang.st@gmail.com` / `3145078758@qq.com`
