# laravel-ci-php
Mục lục
I. Cấu hình Gitlab CI/CD
    1. Cấu hình trên môi trường Staging, Production
    2. Cấu hình Laravel Envoy
    3. Sử dụng docker image
    4. Cấu hình .gitlab-ci.yml
    5. Cài đặt Gitlab-runner
II. Chạy Gitlab CI/CD
    1. Workflow
    2. Kết quả
 

I. Cấu hình Gitlab CI/CD
1. Cấu hình trên môi trường Staging, Production
Tạo user
# Tạo user deployer
sudo adduser deployer

# Cấp quyền deployer user đến thư mục/var/www
sudo setfacl -R -m u:deployer:rwx /var/www
Tạo SSH Key
ssh-keygen -t rsa -b 4096 -C "email@example.com"

Generating public/private rsa key pair.
Enter file in which to save the key (/Users/hoangsang/.ssh/id_rsa): 
/Users/hoangsang/.ssh/id_rsa already exists.
Overwrite (y/n)? y
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /Users/hoangsang/.ssh/id_rsa.
Your public key has been saved in /Users/hoangsang/.ssh/id_rsa.pub.
The key fingerprint is:
SHA256:X/ljjz4+tireNtDCiPC6I3o7S3j2soY5bKvbLMMYEeg email@example.com
The key's randomart image is:
+---[RSA 4096]----+
|.                |
|o                |
|..               |
|.E  .        .   |
| .   o .So .o    |
|..    o ..+...   |
|=++  .    .o  +  |
|BX*.o     ..o.++ |
|*XBBoo   ..oo*=+.|
+----[SHA256]-----+
Thêm SSH Key
# Copy public key đến authorized_keys
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys

# Copy private key đến bộ nhớ tạm
cat ~/.ssh/id_rsa
Thêm Private Key vào Project Gitlab
Setting > CI/CD > Secret Variables

alt

2. Cấu hình Laravel Envoy (hiện tại chỉ hỗ trợ hệ điều MacOS và Linux)
Tạo file Envoy.blade.php trong thư mục gốc của project
Cấu hình file Envoy
@servers(['web' => 'deployer@192.168.1.1'])

@setup
    $app_dir = '/var/www/html/project';
@endsetup

@story('deploy')
    install
@endstory

@task('install')
    cd {{ $app_dir }}
    git stash
    git pull
    php artisan migrate
    php artisan db:seed
@endtask
3. Tạo ra hoặc sử dụng có sẵn 1 docker image
a. Sử dụng docker image có sẵn rexl177/laravel-ci-php
b. Tạo 1 docker image
    Bước 1: Tạo 1 Dockerfile
# Set the base image for subsequent instructions
FROM php:7.1

# Update packages
RUN apt-get update

# Install PHP and composer dependencies
RUN apt-get install -qq git curl libmcrypt-dev libjpeg-dev libpng-dev libfreetype6-dev libbz2-dev

# Clear out the local repository of retrieved package files
RUN apt-get clean

# Install needed extensions
# Here you can install any other extension that you need during the test and deployment process
RUN docker-php-ext-install mcrypt pdo_mysql zip

# Install Composer
RUN curl --silent --show-error https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer

# Install Laravel Envoy
RUN composer global require "laravel/envoy=~1.0"
    Bước 2: Build Dockerfile
    Bước 3: Push image build từ Dockerfile
4. Cấu hình .gitlab-ci.yml file
Tạo file .gitlab-ci.yml nằm trong thư mục gốc của project

Test (test các convention và unittest) Example

image: rexl177/laravel-ci-php

variables:
  MYSQL_ROOT_PASSWORD: root
  MYSQL_USER: homestead
  MYSQL_PASSWORD: secret
  MYSQL_DATABASE: homestead
  DB_HOST: mysql

testing:
  services:
    - mysql:5.7
  script:
    - cd laravel
    - composer install --prefer-dist --no-ansi --no-interaction --no-progress --no-scripts
    - cp .env.example .env
    - php artisan migrate
    - php artisan key:generate
    - php artisan cache:clear
    - php artisan config:clear
    - ./vendor/phpunit/phpunit/phpunit
    - phpcs --standard=PSR2 app/Models
    - phpcs --standard=PSR2 tests
    - phpcs --standard=PSR2 app/Traits
    - phpcs --standard=PSR2 app/Models
    - phpcs --standard=PSR2 app/Services
    - phpcs --standard=PSR2 app/Repositories
    - phpcs --standard=PSR2 app/Http/Controllers
    - phpcs --standard=PSR2 app/Observers
Test (test các convention và unittest) và deploy Example

image: rexl177/laravel-ci-php

variables:
  MYSQL_ROOT_PASSWORD: root
  MYSQL_USER: homestead
  MYSQL_PASSWORD: secret
  MYSQL_DATABASE: homestead
  DB_HOST: mysql

stages:
  - test
  - deploy

testing:
  stage: test
  services:
    - mysql:5.7
  script:
    - cd laravel
    - composer install --prefer-dist --no-ansi --no-interaction --no-progress --no-scripts
    - cp .env.example .env
    - php artisan migrate
    - php artisan key:generate
    - php artisan cache:clear
    - php artisan config:clear
    - ./vendor/phpunit/phpunit/phpunit
    - phpcs --standard=PSR2 app/Models
    - phpcs --standard=PSR2 tests
    - phpcs --standard=PSR2 app/Traits
    - phpcs --standard=PSR2 app/Models
    - phpcs --standard=PSR2 app/Services
    - phpcs --standard=PSR2 app/Repositories
    - phpcs --standard=PSR2 app/Http/Controllers
    - phpcs --standard=PSR2 app/Observers

deploying:
  stage: deploy
  before_script:
    - 'which ssh-agent || ( apk update -y && apk add openssh-client -y )'
    - eval $(ssh-agent -s)
    - ssh-add <(echo "$SSH_PRIVATE_KEY")
    - mkdir -p ~/.ssh
    - '[[ -f /.dockerenv ]] && echo -e "Host *\n\tStrictHostKeyChecking no\n\n" > ~/.ssh/config'
  script:
    - cd laravel
    - envoy run deploy
  environment:
    name: production
    url: 

  only:
    - develop
5. Cài đặt Gitlab-runner
Có thể cài đặt ở local hoặc server

Cài đặt tham khảo tại đây.
Đăng ký Gitlab-runner với Gitlab tham khảo tại đây.
II. Chạy Gitlab CI-CD
1. Flow
Mỗi khi code được thay đổi, Gitlab thông báo cho Gitlab-runner.
Gitlab-runner sẽ pull code từ Gitlab.
Gitlab-runner build code và thực thi các jobs được định nghĩa ở .gitlab-ci.yml
Gitlab-runner sẽ gửi thông báo lên Gitlab.
2. Kết quả (ProjectGitlab > CI/CD > Pipelines)
Kết quả các pipeline



Chi tiết 1 pipeline



Kết quả 1 job của 1 pipeline
