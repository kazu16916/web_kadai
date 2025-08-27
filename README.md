# Kadai ページ表示方法

## 1. EC2 インスタンスの作成

## 2. EC2 インスタンスへ PC から SSH でログイン
```bash
ssh ec2-user@{IPアドレス} -i {秘密鍵ファイルのパス}
```

## 3. Vim のインストール
```bash
sudo yum install vim -y
```

## 4. Vim 設定の変更
```bash
vim ~/.vimrc
```

内容：
```vim
set number
set expandtab
set tabstop=2
set shiftwidth=2
set autoindent
```

## 5. screen のインストール
```bash
sudo yum install screen -y
```

## 6. screen の利用
タブ分け（docker compose, php 編集画面, sql 入力画面など）がおすすめ。

## 7. screen 設定の変更
```bash
vim ~/.screenrc
```

内容：
```
hardstatus alwayslastline "%{= bw}%-w%{= wk}%n%t*%{-}%+w"
```

## 8. Docker のインストール
```bash
sudo yum install -y docker
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -a -G docker ec2-user
```

## 9. Docker Compose のインストール
```bash
sudo mkdir -p /usr/local/lib/docker/cli-plugins/
sudo curl -SL https://github.com/docker/compose/releases/download/v2.36.0/docker-compose-linux-x86_64   -o /usr/local/lib/docker/cli-plugins/docker-compose
sudo chmod +x /usr/local/lib/docker/cli-plugins/docker-compose
docker compose version
```

## 10. プロジェクトディレクトリの作成
```bash
mkdir dockertest
cd dockertest
```

## 11. docker-compose.yml 作成
```bash
vim compose.yml
```

内容：
```yaml
services:
  web:
    image: nginx:latest
    ports:
      - 80:80
    volumes:
      - ./nginx/conf.d/:/etc/nginx/conf.d/
      - ./public/:/var/www/public/
      - image:/var/www/upload/image/
    depends_on:
      - php

  php:
    container_name: php
    build:
      context: .
      target: php
    volumes:
      - ./public/:/var/www/public/
      - image:/var/www/upload/image/

  mysql:
    container_name: mysql
    image: mysql:8.4
    environment:
      MYSQL_DATABASE: example_db
      MYSQL_ALLOW_EMPTY_PASSWORD: 1
      TZ: Asia/Tokyo
    volumes:
      - mysql:/var/lib/mysql
    command: >
      mysqld
      --character-set-server=utf8mb4
      --collation-server=utf8mb4_unicode_ci
      --max_allowed_packet=4MB

volumes:
  mysql:
  image:
```

## 12. Nginx 設定
```bash
mkdir -p nginx/conf.d
vim nginx/conf.d/default.conf
```

内容：
```nginx
server {
    listen       0.0.0.0:80;
    server_name  _;
    charset      utf-8;

    root /var/www/public;

    client_max_body_size 6M;

    location ~ \.php$ {
        fastcgi_pass php:9000;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME /var/www/public$fastcgi_script_name;
        include       fastcgi_params;
    }

    location /image/ {
        root /var/www/upload;
    }
}
```

## 13. PHP Dockerfile 作成
```bash
vim Dockerfile
```

内容：
```dockerfile
FROM php:8.4-fpm-alpine AS php

RUN docker-php-ext-install pdo_mysql

RUN install -o www-data -g www-data -d /var/www/upload/image/

RUN docker-php-ext-install fileinfo && docker-php-ext-enable fileinfo

COPY uploads.ini /usr/local/etc/php/conf.d/uploads.ini
```

## 14. MySQL テーブル作成
```bash
docker compose exec mysql mysql example_db
```

```sql
CREATE TABLE `example_db`.`bbs_entries` (
  `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
  `body` TEXT NOT NULL,
  `image_filename` VARCHAR(255),
  `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

## 15. kadai.php 作成
```bash
vim public/kadai.php
```

内容（一部抜粋）：
```php
<?php
$dbh = new PDO('mysql:host=mysql;dbname=example_db', 'root', '');

if (isset($_POST['body'])) {
  $image_filename = null;
  if (isset($_FILES['image']) && !empty($_FILES['image']['tmp_name'])) {
    if (preg_match('/^image\//', mime_content_type($_FILES['image']['tmp_name'])) !== 1) {
      header("HTTP/1.1 302 Found");
      header("Location: ./kadai.php");
      return;
    }
    $pathinfo = pathinfo($_FILES['image']['name']);
    $extension = $pathinfo['extension'];
    $image_filename = strval(time()) . bin2hex(random_bytes(25)) . '.' . $extension;
    $filepath =  '/var/www/upload/image/' . $image_filename;
    move_uploaded_file($_FILES['image']['tmp_name'], $filepath);
  }

  $insert_sth = $dbh->prepare("INSERT INTO bbs_entries (body, image_filename) VALUES (:body, :image_filename)");
  $insert_sth->execute([
    ':body' => $_POST['body'],
    ':image_filename' => $image_filename,
  ]);

  header("HTTP/1.1 302 Found");
  header("Location: ./kadai.php");
  return;
}
?>
```

## 16. コンテナ起動
```bash
docker compose up --build
```
（必要に応じて SSH 再ログイン推奨）

## 17. ページ確認
```
http://{IPアドレス}/kadai.php
```
