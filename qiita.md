## はじめに

最近、WordPressでのコーポレートサイト開発を少しやっています。
以前は[vccw](http://vccw.cc/)を使っていたのですが、環境構築が少し面倒、遅い、モダンな感じで開発したい、という思いからdockerで環境構築したので、その時のメモを残しておきます。
WordPressのことをあまりちゃんと分かっていないので、色々おかしい点があるかもしれませんがご了承ください🙇‍♂️

## できること

まず前提として、本番のwordpressはルートではなくwordpressディレクトリ(サブディレクトリ)にある状態です。

今回の環境で出来るようになることは以下のようになっています。

- DockerでのWordPressローカル開発環境構築
  - ローカル開発環境の構築が誰でも手軽に
  - 起動や停止が早い
- wordmoveを使用し、本番のデータをローカルに反映
- themeファイルをGit管理
- Mailhogを使用し、メールのローカルテスト

## やったこと

コードは[-QP-docker_wordpress](https://github.com/hasehiro0828/-QP-docker_wordpress)に載せています。
基本的な構成は[Docker-composeで最強（自分史上）のWordpress開発環境を作る](https://qiita.com/ryo2132/items/d75e1846aa181676406e)を参考にさせていただいたので、先にご参照ください。
ただ、私の環境ではwordpressをサブディレクトリに設定しているからなのか、このままではうまくいかなかったので少し修正し、Mailhogも使えるようにしました。
また、テーマのGit管理もできるようにしました。

### `docker-compose.yml`の補足

以下が`docker-compose.yml`です。

```yaml:docker-compose.yml
version: '3'
services:
    database:
        image: mysql:5.7
        command:
            - "--character-set-server=utf8"
            - "--collation-server=utf8_unicode_ci"
        ports:
            - "${LOCAL_DB_PORT}:3306"
        restart: on-failure:5
        container_name: "${PRODUCTION_NAME}_db"
        environment:
            MYSQL_USER: wordpress
            MYSQL_DATABASE: wordpress
            MYSQL_PASSWORD: wordpress
            MYSQL_ROOT_PASSWORD: wordpress
    wordpress:
        depends_on:
            - database
        build: #(2)
            context: ./docker
            dockerfile: wordpress.Dockerfile
        container_name: "${PRODUCTION_NAME}_wordpress"
        ports:
            - "${LOCAL_SERVER_PORT}:80"
        restart: on-failure:5
        volumes:
            - ./wordmove/wordpress:/var/www/html/wordpress #(1)
            - ./original_theme:/var/www/html/wordpress/wp-content/themes/original_theme #(3)
            - ./index.php:/var/www/html/index.php #(1)
            - ./.htaccess:/var/www/html/.htaccess #(1)
        working_dir: /var/www/html/wordpress
        environment:
            WORDPRESS_DB_HOST: database:3306
            WORDPRESS_DB_PASSWORD: wordpress
    wordmove:
        tty: true
        depends_on:
            - wordpress
        image: welaika/wordmove
        restart: on-failure:5
        container_name: "${PRODUCTION_NAME}_wordmove"
        volumes:
            - ./config:/home/
            - ~/.ssh:/home/.ssh
            - ./wordmove/wordpress/:/var/www/html/ #(1)
        environment:
            LOCAL_SERVER_PORT: "${LOCAL_SERVER_PORT}"
            PRODUCTION_URL: "${PRODUCTION_URL}"
            PRODUCTION_DIR_PATH: "${PRODUCTION_DIR_PATH}"
            PRODUCTION_DB_NAME: "${PRODUCTION_DB_NAME}"
            PRODUCTION_DB_USER: "${PRODUCTION_DB_USER}"
            PRODUCTION_DB_PASSWORD: "${PRODUCTION_DB_PASSWORD}"
            PRODUCTION_DB_HOST: "${PRODUCTION_DB_HOST}"
            PRODUCTION_DB_PORT: "${PRODUCTION_DB_PORT}"
            PRODUCTION_SSH_HOST: "${PRODUCTION_SSH_HOST}"
            PRODUCTION_SSH_USER: "${PRODUCTION_SSH_USER}"
            PRODUCTION_SSH_PORT: "${PRODUCTION_SSH_PORT}"
            RUBYOPT: -EUTF-8 #invalid byte sequence in US-ASCII (ArgumentError)の回避
    mailhog: #(2)
        container_name: "${PRODUCTION_NAME}_mailhog"
        image: mailhog/mailhog
        ports:
            - "8025:8025"
            - "1025:1025"
```

#### (1)について

wordpressをサブディレクトリにしていることによる変更です。
wordmoveで持ってきたファイル群をwordpressコンテナの`/var/www/html/wordpress`に置いています。
`index.php`と`.htaccess`は私の環境ではこうしないと上手く行かなかったのですが、不要かもしれないです。（ここら辺よく分かってないです。すみません。）

#### (2)について

Mailhogというローカルでメール送信テストができるコンテナを追加しました。
ただ、これだけでは上手くいかないので、[Docker connect Mail catcher with WordPress](https://stackoverflow.com/questions/42153606/docker-connect-mail-catcher-with-wordpress)を参考に以下の`Dockerfile`を作成しました。
wordpressコンテナにmailhog用の設定をしています。

```dockerfile:wordpress.Dockerfile
FROM wordpress:latest
RUN curl --location --output /usr/local/bin/mhsendmail https://github.com/mailhog/mhsendmail/releases/download/v0.2.0/mhsendmail_linux_amd64 && \
    chmod +x /usr/local/bin/mhsendmail

RUN echo 'sendmail_path="/usr/local/bin/mhsendmail --smtp-addr=mailhog:1025 --from=admin@example.com"' > /usr/local/etc/php/conf.d/mailhog.ini

```

#### (3)について

Git管理しているテーマをwordpressのコンテナに入れています。
ローカルのファイルを変更すればDocker内のファイルも変更されるので、ローカル環境でもすぐに確認できます。

## おわりに

以上、簡単にですが、dockerでWordPressの開発環境を構築した際のメモです。
お役に立てましたら幸いです🙇‍♂️



WordPressを使用したベストの開発方法って何なんでしょうか？
やりたいことがある時は調べれば出るのですが、どう開発していくのがいいのか、何を使えばいいのか、どういう構成にすればいいのか、みたいな開発者目線のまとまった情報みたいなのは中々ネットでは見つけられませんね。
良い本などありましたら教えていただけますとありがたいです！

## 参考

- [Docker connect Mail catcher with WordPress](https://stackoverflow.com/questions/42153606/docker-connect-mail-catcher-with-wordpress)
- [Docker-composeで最強（自分史上）のWordpress開発環境を作る](https://qiita.com/ryo2132/items/d75e1846aa181676406e)