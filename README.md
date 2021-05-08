# etcd.io

This repository houses all of the assets used to build the etcd docs and website available at https://etcd.io.

## Run the site locally

### Prerequisites

To build and serve the site, you'll need these tools:

- **[Hugo, extended edition][hugo-install]**; match the version specified in
  [netlify.toml](netlify.toml)
- **Node**, the latest [LTS release][]. Like Netlify, we use **[nvm][]**, the
  Node Version Manager, to install and manage Node versions:
  ```console
  $ nvm install --lts
  $ nvm use --lts
  ```

### Setup

Once you've installed the [prerequisites](#prerequisites), get local packages:

```bash
npm install
```

### Running

Once the [setup](#setup) has completed, you can locally serve the site:

```bash
npm run serve
```

#### Docker

You can also run the site locally using [Docker](https://docker.com):

```bash
make docker-serve
```

## Publishing the site

The site is published automatically by [Netlify](https://netlify.com). Any time
changes are pushed to the main branch, the site is built and deployed.

### Preview builds

Any time you submit a pull request to this repository, Netlify will publish a
[preview
build](https://www.netlify.com/blog/2016/07/20/introducing-deploy-previews-in-netlify/)
of the changes in that pull request. You can find a link to the preview build in
the checks section of the pull request, under **netlify/etcd/deploy-preview**.

## 怎么发布一个新版的etcd文档？

跟着下面的步骤来添加一个新的etcd文档像这样：vX.Y:

* 复制 [content/docs/next](content/docs/next) 到
  `content/docs/vX.Y`, 这个 `vX.Y` 就是最新的etcd版本. 例如:

    ```bash
    cp -r content/docs/next content/docs/v3.5
    ```

* 找到文件夹下的文件 `_index.md`， 把frontmatter部分的版本改成你想要的内容:
  ```
  ---
  title: etcd version X.Y
  weight: 1000
  cascade:
    version: vX.Y
  ---
  ```
* 在配置文件[config.toml]的屬性`params.versions.all`中添加剛才的版本.
* 如果它是最新版本, 就改变这个参数`params.versions.latest`.
* 提交PR.

## Troubleshooting

If you have an issue with updating the documentation, file an issue against this
repo.

[hugo-install]: https://gohugo.io/getting-started/installing
[LTS release]: https://nodejs.org/en/about/releases/
[nvm]: https://github.com/nvm-sh/nvm/blob/master/README.md#installing-and-updating
