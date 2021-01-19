# Jekyll Homepage

## Install prerequisites

### Debian

```
sudo apt install ruby-full
```

* https://nokogiri.org/tutorials/installing_nokogiri.html

```
sudo apt-get install zlib1g-dev liblzma-dev patch
sudo gem install nokogiri -v '1.10.10' --source 'https://rubygems.org/'
```

## Building locally

```
sudo gem install bundler jekyll

git clone git@github.com:nfaction/nfaction.github.io.git

cd nfaction.github.io

bundle install

bundle exec jekyll serve --watch -H 0.0.0.0
```

Open browser at IP address: `http://<IP>:4000`