# Kadai ページ表示方法

## エラー情報 
## ”Can't open file for writing”と出力が出る可能性があるので 
```bash
sudo chown -R ec2-user:ec2-user public/
```
## など現在書き込もうとしているディレクトリを書き込み権限追加してください 

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
## SSHを一度ログアウトし、もう一度ログイン
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


```
<?php
$dbh = new PDO('mysql:host=mysql;dbname=example_db', 'root', '');

if (isset($_POST['body'])) {
  // POSTで送られてくるフォームパラメータ body がある場合

  $image_filename = null;
  if (isset($_FILES['image']) && !empty($_FILES['image']['tmp_name'])) {
    // アップロードされた画像がある場合
    if (preg_match('/^image\//', mime_content_type($_FILES['image']['tmp_name'])) !== 1) {
      // アップロードされたものが画像ではなかった場合処理を強制的に終了
      header("HTTP/1.1 302 Found");
      header("Location: ./kadai.php");
      return;
    }

    // 元のファイル名から拡張子を取得
    $pathinfo = pathinfo($_FILES['image']['name']);
    $extension = $pathinfo['extension'];
    // 新しいファイル名を決める。他の投稿の画像ファイルと重複しないように時間+乱数で決める。
    $image_filename = strval(time()) . bin2hex(random_bytes(25)) . '.' . $extension;
    $filepath =  '/var/www/upload/image/' . $image_filename;
    move_uploaded_file($_FILES['image']['tmp_name'], $filepath);
  }

  // insertする
  $insert_sth = $dbh->prepare("INSERT INTO bbs_entries (body, image_filename) VALUES (:body, :image_filename)");
  $insert_sth->execute([
    ':body' => $_POST['body'],
    ':image_filename' => $image_filename,
  ]);

  // 処理が終わったらリダイレクトする
  // リダイレクトしないと，リロード時にまた同じ内容でPOSTすることになる
  header("HTTP/1.1 302 Found");
  header("Location: ./kadai.php");
  return;
}

// いままで保存してきたものを取得
$select_sth = $dbh->prepare('SELECT * FROM bbs_entries ORDER BY created_at DESC');
$select_sth->execute();
?>

<!-- フォームのPOST先はこのファイル自身にする -->
<form method="POST" action="./kadai.php" enctype="multipart/form-data">
  <textarea name="body" required></textarea>
  <div style="margin: 1em 0;">
    <input type="file" accept="image/*" name="image" id="imageInput">
  </div>
  <button type="submit">送信</button>
</form>

<hr>

<?php foreach($select_sth as $entry): ?>
  <dl style="margin-bottom: 1em; padding-bottom: 1em; border-bottom: 1px solid #ccc;">
    <dt>ID</dt>
    <dd><?= $entry['id'] ?></dd>
    <dt>日時</dt>
    <dd><?= $entry['created_at'] ?></dd>
    <dt>内容</dt>
    <dd>
      <?= nl2br(htmlspecialchars($entry['body'])) // 必ず htmlspecialchars() すること ?>
      <?php if(!empty($entry['image_filename'])): // 画像がある場合は img 要素を使って表示 ?>
      <div>
        <img src="/image/<?= $entry['image_filename'] ?>" style="max-height: 10em;">
      </div>
      <?php endif; ?>
    </dd>
  </dl>
<?php endforeach ?>

<script>
document.addEventListener("DOMContentLoaded", () => {
  const imageInput = document.getElementById("imageInput");
  imageInput.addEventListener("change", () => {
    if (imageInput.files.length < 1) {
      // 未選択の場合
      return;
    }
    if (imageInput.files[0].size > 5 * 1024 * 1024) {
      // ファイルが5MBより多い場合
      alert("5MB以下のファイルを選択してください。");
      imageInput.value = "";
    }
  });
});
</script>

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
