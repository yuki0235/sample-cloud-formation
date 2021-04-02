# Rails環境の構築手順(各リソース作成後以降)

```
# EC2へsshログイン
$ ssh -i ~/.ssh/<private_key> ec2-user@<ipv4_public_ip>

# 各種パッケージをインストール(yum)
$ sudo yum install -y git gcc gcc-c++ make openssl-devel readline-devel glibc-headers libyaml-devel zlib zlib-devel libffi-devel libxml2 libxslt libxml2-devel libxslt-devel mysql mysql-devel

# nginxインストール
$ sudo amazon-linux-extras install -y nginx1

# rbenvの設定
$ git clone git://github.com/rbenv/rbenv.git ~/.rbenv
$ echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.bashrc
$ echo 'eval "$(rbenv init -)"' >> ~/.bashrc
$ source ~/.bashrc

# ruby-buildの設定
$ git clone git://github.com/rbenv/ruby-build.git /tmp/ruby-build
$ cd /tmp/ruby-build
$ sudo ./install.sh

# rubyのインストール
$ rbenv install 2.6.5
$ rbenv global 2.6.5
$ rbenv rehash
$ ruby -v

# bundlerのインストール
$ gem install bundler

# railsのインストール
$ gem install rails -v 6.0.3.4
$ rails -v

# nvmの設定
$ git clone https://github.com/creationix/nvm.git ~/.nvm
$ source ~/.nvm/nvm.sh
$ echo 'if [ -f ~/.nvm/nvm.sh ]; then' >> ~/.bashrc
$ echo '  . ~/.nvm/nvm.sh' >> ~/.bashrc
$ echo 'fi' >> ~/.bashrc

# Node.jsの設定
$ nvm ls-remote
$ nvm install 15.0.0
$ node --version
$ npm --version

# yarnの設定
$ npm install yarn -g
$ yarn --version

# アプリケーションの設定
$ sudo mkdir -p /var/www
$ sudo chown ec2-user:ec2-user /var/www
$ cd /var/www; pwd
$ rails new sapmle-app -d mysql 
# vi config/database.yml
=> hostにRDSのエンドポイントを追加
=> パスワードを追記
$ bundle exec rake db:create
$ bundle exec rake db:migrate
$ rails s -b 0.0.0.0
=> ブラウザから「http://<ec2のIP>:3000」でwelcomeページが確認出来ればOK

# nginxの確認
$ sudo systemctl status nginx
=> 「Active: inactive (dead)」になっている
$ sudo systemctl start nginx
$ sudo systemctl status nginx
=> 「Active: active (running) since ~」になっている
=> ブラウザから「http://<ec2のIP>」で接続出来ればOK

# ALBからの接続確認
=> ブラウザから「http://<ALBのDNS名>」で接続確認出来ればOK（nginxが起動していること）

# カスタムnginx設定ファイルを追加
$ sudo vi /etc/nginx/conf.d/<任意の名前>.conf
=> 以下を記述
================================================================
server {
    listen 80;
    server_name 18.181.48.40;

    root /var/www/sample-app/public;

    location = /favicon.ico {
        access_log off;
        log_not_found off;
    }

    location ~ ^/assets/ {
        gzip_static on;
        expires 1d;
    }

    location / {
        try_files $uri $uri/index.html @sample-app_app;
    }

    location @sample-app_app {
        proxy_pass http://puma;
        #proxy_pass http://unicorn;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Host $http_host;
        proxy_redirect off;
        proxy_connect_timeout 30;
    }

    error_page 404 /404.html;
        location = /404.html {
    }
    error_page 500 502 503 504 /500.html;
        location = /500.html {
    }
}
================================================================

# デフォルトのnginx設定ファイルを修正
$ sudo vi /etc/nginx/nginx.conf
=> serverディレクティブをコメントアウト

# nginx設定ファイルのsyntaxチェック
$ sudo nginx -t

# pumaの設定
$ vi config/puma.rb
=> portの行をコメントアウトして以下の１行を追加
================================================================
bind "unix://#{Rails.root}/tmp/sockets/puma.sock"
================================================================
```
$ touch /var/www/<アプリケーションディレクトリ名>/tmp/sockets/puma.sock
$ chmod 777 /var/www/<アプリケーションディレクトリ名>/tmp/sockets/puma.sock
