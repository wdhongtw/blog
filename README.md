# Space for Personal Blog

This blog is hosted on GitHub Pages and rely totally on
the Ruby gem, `github-page`.

For more detail, see official documents.

- [Working with GitHub Pages - GitHub Help](https://help.github.com/en/github/working-with-github-pages)
- [GitHub Pages | Jekyll](https://jekyllrb.com/docs/github-pages/)

## Setup Locally

Assume that Ubuntu 18.04 is used.

Install Ruby and libraries.

``` shell
apt install ruby bundler zlib1g-dev
# zlib1g-dev is required by nokogiri
```

Run bundle install in project folder, install gems into `vendor/bundle`.

``` shell
bundle install --path vendor/bundle --without test
```

Note that the command above may not run successfully as normal user
in Windows WSL. The following workaround may help.

``` shell
sudo bundle install --path vendor/bundle --without test
sudo chown -R <username>: vendor/bundle .bundle/config Gemfile.lock
```

## Usage

``` shell
bundle exec jekyll serve
```
