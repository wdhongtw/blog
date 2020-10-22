# Space for Personal Blog

This blog is hosted on GitHub Pages and rely totally on
the Ruby gem, `github-page`.

For more detail, see official documents.

- [Working with GitHub Pages - GitHub Help](https://help.github.com/en/github/working-with-github-pages)
- [GitHub Pages | Jekyll](https://jekyllrb.com/docs/github-pages/)

## Setup Locally

Install Ruby and libraries. (Assume using Ubuntu / Debian based distribution)

``` shell
apt install ruby ruby-dev bundler build-essential patch zlib1g-dev liblzma-dev
# zlib1g-dev is required by nokogiri
# see: https://nokogiri.org/tutorials/installing_nokogiri.html
```

Run bundle install in project folder, install gems into `vendor/bundle`.

``` shell
bundle config set --local path 'vendor/bundle'
bundle config set --local without 'test'
bundle install
```

Note that the install command may not run successfully as normal user
in Windows WSL. The following workaround may help.

``` shell
sudo bundle install
sudo chown -R <username>: vendor/bundle .bundle/config Gemfile.lock
```

## Usage

``` shell
bundle exec jekyll serve
```
