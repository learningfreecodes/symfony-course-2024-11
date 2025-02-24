# Установка и "Hello, world"

## Устанавливаем Symfony
1. Инициализируем проект командой `composer create-project symfony/skeleton otus_project "7.1.*"`
2. Добавляем в .gitignore служебный каталог PhpStorm
    ```gitignore
    .idea/*
    ```

## Добавляем WorldController с экшеном hello

1. Создаём класс `App\Controller\WorldController`
    ```php
    <?php

    namespace App\Controller;

    use Symfony\Component\HttpFoundation\Response;

    class WorldController
    {
        public function hello(): Response
        {
            return new Response('<html><body><h1><b>Hello,</b> <i>world</i>!</h1></body></html>');
        }
    }
    ```
2. В файле `config/routes.yaml` добавляем описание endpoint'а
    ```yaml
    hello_world:
        path: /world/hello
        controller: App\Controller\WorldController::hello
    ```
3. Выполняем команду `symfony serve`
4. Заходим по адресу `http://localhost:8000`, видим приветственную страницу Symfony
5. Заходим по адресу `http://localhost:8000/world/hello`, видим результат работы нашего контроллера

## Добавляем Docker

1. Создаём файл `docker-compose.yml`
    ```yaml
    version: '3.7'
    
    services:
    
      php-fpm:
        build: docker
        container_name: 'php'
        ports:
          - '9000:9000'
        volumes:
          - ./:/app
        working_dir: /app
    
      nginx:
        image: nginx
        container_name: 'nginx'
        working_dir: /app
        ports:
          - '7777:80'
        volumes:
          - ./:/app
          - ./docker/nginx.conf:/etc/nginx/conf.d/default.conf
    ```
2. Создаём файл `docker\nginx.conf`
    ```
    server {
        listen 80;
     
        server_name localhost;
        error_log  /var/log/nginx/error.log;
        access_log /var/log/nginx/access.log;
        root /app/public;
     
        rewrite ^/index\.php/?(.*)$ /$1 permanent;
     
        try_files $uri @rewriteapp;
     
        location @rewriteapp {
            rewrite ^(.*)$ /index.php/$1 last;
        }
     
        # Deny all . files
        location ~ /\. {
            deny all;
        }
     
        location ~ ^/index\.php(/|$) {
            fastcgi_split_path_info ^(.+\.php)(/.*)$;
            include fastcgi_params;
            fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
            fastcgi_param PATH_INFO $fastcgi_path_info;
            fastcgi_index index.php;
            send_timeout 1800;
            fastcgi_read_timeout 1800;
            fastcgi_pass php:9000;
        }
    }
    ```
3. Создаём файл `docker\Dockerfile`
    ```dockerfile
    FROM php:8.3-fpm-alpine
    
    # Install dev dependencies
    RUN apk update \
        && apk upgrade --available \
        && apk add --virtual build-deps \
            autoconf \
            build-base \
            icu-dev \
            libevent-dev \
            openssl-dev \
            zlib-dev \
            libzip \
            libzip-dev \
            zlib \
            zlib-dev \
            bzip2 \
            git \
            libpng \
            libpng-dev \
            libjpeg \
            libjpeg-turbo-dev \
            libwebp-dev \
            freetype \
            freetype-dev \
            postgresql-dev \
            linux-headers \
            curl \
            wget \
            bash
    
    # Install Composer
    RUN curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/bin --filename=composer
    
    # Install PHP extensions
    RUN docker-php-ext-configure gd --with-freetype=/usr/include/ --with-jpeg=/usr/include/
    RUN docker-php-ext-install -j$(getconf _NPROCESSORS_ONLN) \
        intl \
        gd \
        bcmath \
        pdo_pgsql \
        sockets \
        zip
    RUN pecl channel-update pecl.php.net \
        && pecl install -o -f \
            redis \
            event \
        && rm -rf /tmp/pear \
        && echo "extension=redis.so" > /usr/local/etc/php/conf.d/redis.ini \
        && echo "extension=event.so" > /usr/local/etc/php/conf.d/event.ini
     ```
4. Запускаем контейнеры командой `docker-compose up -d`
5. Заходим по адресу `http://localhost:7777/world/hello`, видим результат работы нашего контроллера



# DI и сервисы

Запускаем контейнеры командой `docker-compose up -d`

## Добавляем сервис со строковым параметром

1. Создаём класс `App\Domain\Service\GreeterService`
    ```php
    <?php
    
    namespace App\Domain\Service;
    
    class GreeterService
    {
        private string $greet;
    
        public function __construct(string $greet)
        {
            $this->greet = $greet;
        }
    
        public function greet(string $name): string
        {
            return $this->greet.', '.$name.'!';
        }
    }
    ```
2. Исправляем класс `App\Controller\WorldController`
    ```php
    <?php
    
    namespace App\Controller;
    
    use App\Domain\Service\GreeterService;
    use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
    use Symfony\Component\HttpFoundation\Response;
    
    class WorldController extends AbstractController
    {
        public function __construct(private readonly GreeterService $greeterService)
        {
        }
    
        public function hello(): Response
        {
            return new Response("<html><body>{$this->greeterService->greet('world')}</body></html>");
        }
    }
    ```
3. Заходим по адресу `http://localhost:7777/world/hello`, видим ошибку

## Добавляем инъекцию параметра

1. Добавляем в файле `config/services.yaml` новую службу
    ```yaml
    App\Domain\Service\GreeterService:
        arguments:
            $greet: 'Hello'
    ```
2. Заходим по адресу `http://localhost:7777/world/hello`, видим сообщение

## Добавляем FormatService

1. Создаём класс `App\Domain\Service\FormatService`
    ```php
    <?php
    
    namespace App\Domain\Service;
    
    class FormatService
    {
        private ?string $tag;
    
        public function __construct()
        {
            $this->tag = null;
        }
    
        public function setTag(string $tag): self
        {
            $this->tag = $tag;
    
            return $this;
        }
    
        public function format(string $contents): string
        {
            return ($this->tag === null) ? $contents : "<{$this->tag}>$contents</{$this->tag}>";
        }
    }
    ```
2. Создаём класс `App\Domain\Service\FormatServiceFactory`
    ```php
    <?php
    
    namespace App\Domain\Service;
    
    class FormatServiceFactory
    {
        public static function strongFormatService(): FormatService
        {
            return (new FormatService())->setTag('strong');
        }
    
        public function citeFormatService(): FormatService
        {
            return (new FormatService())->setTag('cite');
        }
    
        public function headerFormatService(int $level): FormatService
        {
            return (new FormatService())->setTag("h$level");
        }
    }
    ```
3. Исправляем класс `App\Controller\WorldController`
    ```php
    <?php
    
    namespace App\Controller;
    
    use App\Domain\Service\FormatService;
    use App\Domain\Service\GreeterService;
    use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
    use Symfony\Component\HttpFoundation\Response;
    
    class WorldController extends AbstractController
    {
        public function __construct(
            private readonly FormatService $formatService,
            private readonly GreeterService $greeterService,
        )
        {
        }
    
        public function hello(): Response
        {
            $result = $this->formatService->format($this->greeterService->greet('world'));
    
            return new Response("<html><body>$result</body></html>");
        }
    }
    ```
4. Добавляем новые сервисы форматтеров в файле `config/services.yaml`
    ```yaml
    strong_formatter:
      class: App\Domain\Service\FormatService
      factory: ['App\Domain\Service\FormatServiceFactory', 'strongFormatService']
    
    cite_formatter:
      class: App\Domain\Service\FormatService
      factory: ['@App\Domain\Service\FormatServiceFactory', 'citeFormatService']
    
    main_header_formatter:
      class: App\Domain\Service\FormatService
      factory: ['@App\Domain\Service\FormatServiceFactory', 'headerFormatService']
      arguments: [1]
    ```
5. Заходим по адресу `http://localhost:7777/world/hello`, видим, что никакого форматирования не произошло

## Добавляем инъекцию конкретного форматтера

1. В файле `config/services.yaml` добавляем инъекцию конкретного форматтера в контроллер
    ```yaml
    App\Controller\WorldController:
      arguments:
        $formatService: '@cite_formatter'
    ```
2. Заходим по адресу `http://localhost:7777/world/hello`, видим, что применилось форматирование

## Добавляем сервисы с тэгами и проход компилятора

1. Создаём класс `App\Domain\Service\MessageService`
    ```php
    <?php
    
    namespace App\Domain\Service;
    
    class MessageService
    {
        /** @var GreeterService[] */
        private array $greeterServices;
        /** @var FormatService[] */
        private array $formatServices;
    
        public function __construct()
        {
            $this->greeterServices = [];
            $this->formatServices = [];
        }
    
        public function addGreeter(GreeterService $greeterService): void
        {
            $this->greeterServices[] = $greeterService;
        }
    
        public function addFormatter(FormatService $formatService): void
        {
            $this->formatServices[] = $formatService;
        }
    
        public function printMessages(string $name): string
        {
            $result = '';
            foreach ($this->greeterServices as $greeterService) {
                $current = $greeterService->greet($name);
                foreach ($this->formatServices as $formatService) {
                    $current = $formatService->format($current);
                }
                $result .= $current;
            }
    
            return $result;
        }
    }
    ```
2. Исправляем класс `App\Controller\WorldController`
    ```php
    <?php
    
    namespace App\Controller;
    
    use App\Domain\Service\FormatService;
    use App\Domain\Service\MessageService;
    use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
    use Symfony\Component\HttpFoundation\Response;
    
    class WorldController extends AbstractController
    {
        public function __construct(
            private readonly FormatService $formatService,
            private readonly MessageService $messageService,
        )
        {
        }
    
        public function hello(): Response
        {
            $result = $this->formatService->format($this->messageService->printMessages('world'));
    
            return new Response("<html><body>$result</body></html>");
        }
    }
    ```
3. В файл `config/services.yaml`
    1. Убираем сервис `App\Domain\Service\GreeterService`
    2. Добавляем новые сервисы
         ```yaml
         hello_greeter:
           class: App\Domain\Service\GreeterService
           arguments:
             $greet: 'Hello'
           tags: ['app.greeter_service']
         
         greetings_greeter:
           class: App\Domain\Service\GreeterService
           arguments:
             $greet: 'Greetings'
           tags: ['app.greeter_service']
         
         hi_greeter:
           class: App\Domain\Service\GreeterService
           arguments:
             $greet: 'Hi'
           tags: ['app.greeter_service']
         
         list_formatter:
           class: App\Domain\Service\FormatService
           calls:
             - [setTag, ['ol']]
         
         list_item_formatter:
           class: App\Domain\Service\FormatService
           calls:
             - [setTag, ['li']]
           tags: ['app.formatter_service']
         ```
    3. Добавляем тэг `app.formatter_service` для сервисов `cite_formatter` и `strong_formatter`
    4. Исправляем описание сервиса `App\Controller\WorldController`
         ```yaml
         App\Controller\WorldController:
           arguments:
             $formatService: '@list_formatter'
         ```
4. Создаём класс `App\Application\Symfony\GreeterPass`
    ```php
    <?php
    
    namespace App\Application\Symfony;
    
    use App\Domain\Service\MessageService;
    use Symfony\Component\DependencyInjection\Compiler\CompilerPassInterface;
    use Symfony\Component\DependencyInjection\ContainerBuilder;
    use Symfony\Component\DependencyInjection\Reference;
    
    class GreeterPass implements CompilerPassInterface
    {
        public function process(ContainerBuilder $container): void
        {
            if (!$container->has(MessageService::class)) {
                return;
            }
            $messageService = $container->findDefinition(MessageService::class);
            $greeterServices = $container->findTaggedServiceIds('app.greeter_service');
            foreach ($greeterServices as $id => $tags) {
                $messageService->addMethodCall('addGreeter', [new Reference($id)]);
            }
        }
    }
    ```
5. Создаём класс `App\Application\Symfony\FormatterPass`
    ```php
    <?php
    
    namespace App\Application\Symfony;
    
    use App\Domain\Service\MessageService;
    use Symfony\Component\DependencyInjection\Compiler\CompilerPassInterface;
    use Symfony\Component\DependencyInjection\ContainerBuilder;
    use Symfony\Component\DependencyInjection\Reference;
    
    class FormatterPass implements CompilerPassInterface
    {
        public function process(ContainerBuilder $container): void
        {
            if (!$container->has(MessageService::class)) {
                return;
            }
            $messageService = $container->findDefinition(MessageService::class);
            $formatterServices = $container->findTaggedServiceIds('app.formatter_service');
            foreach ($formatterServices as $id => $tags) {
                $messageService->addMethodCall('addFormatter', [new Reference($id)]);
            }
        }
    }
    ```
6. В класс `App\Kernel` добавляем новый метод `build`
    ```php
    protected function build(ContainerBuilder $container): void
    {
        $container->addCompilerPass(new FormatterPass());
        $container->addCompilerPass(new GreeterPass());
    }
    ```
7. Заходим по адресу `http://localhost:7777/world/hello`, видим нумерованный список из трёх приветствий с
   форматированием

## Добавляем приоритеты к тэгам

1. В файле `config/services.yaml` изменяем описание тэгов для сервисов приветствий
    ```yaml
    hello_greeter:
      class: App\Domain\Service\GreeterService
      arguments:
        $greet: 'Hello'
      tags:
        - { name: 'app.greeter_service', priority: 3 }
    
    greetings_greeter:
      class: App\Domain\Service\GreeterService
      arguments:
        $greet: 'Greetings'
      tags:
        - { name: 'app.greeter_service', priority: 2 }
    
    hi_greeter:
      class: App\Domain\Service\GreeterService
      arguments:
        $greet: 'Hi'
      tags:
        - { name: 'app.greeter_service', priority: 1 }
    ```
2. Исправляем класс `App\Application\Symfony\GreeterPass`
    ```php
    <?php
    
    namespace App\Application\Symfony;
    
    use App\Domain\Service\MessageService;
    use Symfony\Component\DependencyInjection\Compiler\CompilerPassInterface;
    use Symfony\Component\DependencyInjection\ContainerBuilder;
    use Symfony\Component\DependencyInjection\Reference;
    
    class GreeterPass implements CompilerPassInterface
    {
        public function process(ContainerBuilder $container): void
        {
            if (!$container->has(MessageService::class)) {
                return;
            }
            $messageService = $container->findDefinition(MessageService::class);
            $greeterServices = $container->findTaggedServiceIds('app.greeter_service');
            uasort($greeterServices, static fn(array $tag1, array $tag2) => $tag1[0]['priority'] - $tag2[0]['priority']);
            foreach ($greeterServices as $id => $tags) {
                $messageService->addMethodCall('addGreeter', [new Reference($id)]);
            }
        }
    }
    ```
3. Заходим по адресу `http://localhost:7777/world/hello`, видим пересортированный список



# Doctrine ORM

Запускаем docker-контейнеры командой `docker-compose up -d`

## Устанавливаем требуемые пакеты, добавляем контейнер с СУБД, добавляем миграцию и выполняем её

1. Заходим в контейнер `php` командой `docker exec -it php sh`. Дальнейшие команды выполняются из контейнера
2. Устанавливаем Doctrine ORM командой `composer require doctrine/orm`
3. Устанавливаем пакет для работы с Doctrine командой `composer require doctrine/doctrine-bundle`, не соглашаемся на
   обновление `docker-compose.yml`
4. Устанавливаем пакет для работы с миграциями командой `composer require doctrine/doctrine-migrations-bundle`
5. Выходим из контейнера и останавливаем все контейнеры командой `docker-compose stop`
6. Создаём файл `migrations/Version20240813150927.php`
    ```php
    <?php
    
    namespace DoctrineMigrations;
    
    use Doctrine\DBAL\Schema\Schema;
    use Doctrine\Migrations\AbstractMigration;
    
    class Version20240813150927 extends AbstractMigration
    {
        public function up(Schema $schema): void
        {
            $this->addSql('CREATE TABLE tweet (id BIGSERIAL NOT NULL, author_id BIGINT DEFAULT NULL, text VARCHAR(140) NOT NULL, created_at TIMESTAMP(0) WITHOUT TIME ZONE NOT NULL, updated_at TIMESTAMP(0) WITHOUT TIME ZONE DEFAULT NULL, PRIMARY KEY(id))');
            $this->addSql('CREATE TABLE "user" (id BIGSERIAL NOT NULL, login VARCHAR(32) NOT NULL, created_at TIMESTAMP(0) WITHOUT TIME ZONE NOT NULL, updated_at TIMESTAMP(0) WITHOUT TIME ZONE DEFAULT NULL, PRIMARY KEY(id))');
            $this->addSql('ALTER TABLE tweet ADD CONSTRAINT tweet__author_id__fk FOREIGN KEY (author_id) REFERENCES "user" (id) NOT DEFERRABLE INITIALLY IMMEDIATE');
            $this->addSql('CREATE TABLE subscription (id BIGSERIAL NOT NULL, author_id BIGINT DEFAULT NULL, follower_id BIGINT DEFAULT NULL, created_at TIMESTAMP(0) WITHOUT TIME ZONE NOT NULL, updated_at TIMESTAMP(0) WITHOUT TIME ZONE DEFAULT NULL, PRIMARY KEY(id))');
            $this->addSql('CREATE INDEX subscription__author_id__idx ON subscription (author_id)');
            $this->addSql('CREATE INDEX subscription__follower_id__idx ON subscription (follower_id)');
            $this->addSql('CREATE UNIQUE INDEX subscription__author_id__follower_id__uniq ON subscription (author_id, follower_id)');
            $this->addSql('ALTER TABLE subscription ADD CONSTRAINT subscription__author_id__fk FOREIGN KEY (author_id) REFERENCES "user" (id) NOT DEFERRABLE INITIALLY IMMEDIATE');
            $this->addSql('ALTER TABLE subscription ADD CONSTRAINT subscription__follower_id__fk FOREIGN KEY (follower_id) REFERENCES "user" (id) NOT DEFERRABLE INITIALLY IMMEDIATE');
            $this->addSql('CREATE TABLE author_follower (author_id BIGINT DEFAULT NULL, follower_id BIGINT DEFAULT NULL, PRIMARY KEY(author_id, follower_id))');
            $this->addSql('CREATE INDEX author_follower__author_id__idx ON author_follower (author_id)');
            $this->addSql('CREATE INDEX author_follower__follower_id__idx ON author_follower (follower_id)');
            $this->addSql('CREATE UNIQUE INDEX author_follower__author_id__follower_id__uniq ON author_follower (author_id, follower_id)');
            $this->addSql('ALTER TABLE author_follower ADD CONSTRAINT author_follower__author_id__fk FOREIGN KEY (author_id) REFERENCES "user" (id) NOT DEFERRABLE INITIALLY IMMEDIATE');
            $this->addSql('ALTER TABLE author_follower ADD CONSTRAINT author_follower__follower_id__fk FOREIGN KEY (follower_id) REFERENCES "user" (id) NOT DEFERRABLE INITIALLY IMMEDIATE');
            $this->addSql('CREATE TABLE feed (id BIGSERIAL NOT NULL, reader_id BIGINT DEFAULT NULL, tweets JSON DEFAULT NULL, created_at TIMESTAMP(0) WITHOUT TIME ZONE NOT NULL, updated_at TIMESTAMP(0) WITHOUT TIME ZONE DEFAULT NULL, PRIMARY KEY(id))');
            $this->addSql('CREATE UNIQUE INDEX feed__reader_id__uniq ON feed (reader_id)');
            $this->addSql('ALTER TABLE feed ADD CONSTRAINT feed__reader_id__fk FOREIGN KEY (reader_id) REFERENCES "user" (id) NOT DEFERRABLE INITIALLY IMMEDIATE');
        }
    
        public function down(Schema $schema): void
        {
            $this->addSql('DROP TABLE author_follower');
            $this->addSql('DROP TABLE subscription');
            $this->addSql('DROP TABLE tweet');
            $this->addSql('DROP TABLE "user"');
            $this->addSql('DROP TABLE feed');
        }
    }
    ```
7. В файле `docker-compose.yml`
    1. В секцию `services` добавляем
        ```yaml
        postgres:
          image: postgres:15
          ports:
            - 15432:5432
          container_name: 'postgresql'
          working_dir: /app
          restart: always
          environment:
            POSTGRES_DB: 'twitter'
            POSTGRES_USER: 'user'
            POSTGRES_PASSWORD: 'password'
          volumes:
            - dump:/app/dump
            - postgresql:/var/lib/postgresql/data
        ```
    2. Добавляем секцию `volumes`
        ```yaml
        volumes:
          dump:
          postgresql:
        ```
8. В файле `.env` настраиваем переменную `DATABASE_URL` для доступа к БД
    ```shell
    DATABASE_URL="postgresql://user:password@postgresql:5432/twitter?serverVersion=15&charset=utf8"
    ```
9. Перезапускаем контейнеры командой `docker-compose up -d`
10. Заходим в контейнер `php` командой `docker exec -it php sh`, дальнейшие команды выполняются из контейнера
11. В контейнере выполняем команду `php bin/console doctrine:migrations:migrate`
12. Подключаемся к БД и проверяем, что таблицы были созданы

## Добавляем сущность (Entity) пользователя и метод его создания

1. В файле `config/packages/doctrine.yaml` исправляем секцию `doctrine.orm.mappings.App`
    ```yaml
    App:
        type: attribute
        is_bundle: false
        dir: '%kernel.project_dir%/src/Domain/Entity'
        prefix: 'App\Domain\Entity'
        alias: App
    ```
2. Добавляем класс `App\Domain\Entity\User`
    ```php
    <?php
    
    namespace App\Domain\Entity;
    
    use DateTime;
    use Doctrine\ORM\Mapping as ORM;
    
    #[ORM\Table(name: '`user`')]
    #[ORM\Entity]
    class User
    {
        #[ORM\Column(name: 'id', type: 'bigint', unique: true)]
        #[ORM\Id]
        #[ORM\GeneratedValue(strategy: 'IDENTITY')]
        private ?int $id = null;
    
        #[ORM\Column(type: 'string', length: 32, nullable: false)]
        private string $login;
    
        #[ORM\Column(name: 'created_at', type: 'datetime', nullable: false)]
        private DateTime $createdAt;
    
        #[ORM\Column(name: 'updated_at', type: 'datetime', nullable: false)]
        private DateTime $updatedAt;
    
        public function getId(): int
        {
            return $this->id;
        }
    
        public function setId(int $id): void
        {
            $this->id = $id;
        }
    
        public function getLogin(): string
        {
            return $this->login;
        }
    
        public function setLogin(string $login): void
        {
            $this->login = $login;
        }
    
        public function getCreatedAt(): DateTime {
            return $this->createdAt;
        }
    
        public function setCreatedAt(): void {
            $this->createdAt = new DateTime();
        }
    
        public function getUpdatedAt(): DateTime {
            return $this->updatedAt;
        }
    
        public function setUpdatedAt(): void {
            $this->updatedAt = new DateTime();
        }
    
        public function toArray(): array
        {
            return [
                'id' => $this->id,
                'login' => $this->login,
                'createdAt' => $this->createdAt->format('Y-m-d H:i:s'),
                'updatedAt' => $this->updatedAt->format('Y-m-d H:i:s'),
            ];
        }
    }
    ```
3. Добавляем класс `App\Domain\Service\UserService`
    ```php
    <?php
    
    namespace App\Domain\Service;
    
    use App\Domain\Entity\User;
    
    class UserService
    {
        public function create(string $login): User
        {
            $user = new User();
            $user->setLogin($login);
            $user->setCreatedAt();
            $user->setUpdatedAt();
    
            return $user;
        }
    }
    ```
4. Исправляем класс `App\Controller\WorldController`
    ```php
    <?php
    
    namespace App\Controller;
    
    use App\Domain\Service\UserService;
    use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
    use Symfony\Component\HttpFoundation\Response;
    
    class WorldController extends AbstractController
    {
        public function __construct(
            private readonly UserService $userService,
        )
        {
        }
    
        public function hello(): Response
        {
            $user = $this->userService->create('My user');
    
            return $this->json($user->toArray());
        }
    }
    ```
5. В файле `config/services.yaml` убираем описание сервиса `App\Controller\WorldController`
6. Заходим по адресу `http://localhost:7777/world/hello`, видим данные нашего пользователя и `id = null`

## Добавляем сохранение пользователя в БД

1. Добавляем интерфейс `App\Domain\Entity\EntityInterface`
    ```php
    <?php
    
    namespace App\Domain\Entity;
    
    interface EntityInterface
    {
        public function getId(): int;
    }
    ```
2. Имплемпентируем этот интерфейс в классе `App\Domain\Entity\User`
3. Добавляем класс `App\Infrastructure\Repository\AbstractRepository`
    ```php
    <?php
    
    namespace App\Infrastructure\Repository;
    
    use App\Domain\Entity\EntityInterface;
    use Doctrine\ORM\EntityManagerInterface;
    
    /**
     * @template T
     */
    abstract class AbstractRepository
    {
        public function __construct(protected readonly EntityManagerInterface $entityManager)
        {
        }
    
        protected function flush(): void
        {
            $this->entityManager->flush();
        }
    
        /**
         * @param T $entity
         */
        protected function store(EntityInterface $entity): int
        {
            $this->entityManager->persist($entity);
            $this->flush();
    
            return $entity->getId();
        }
    }
    ```
4. Добавляем класс `App\Infrastructure\Repository\UserRepository`
    ```php
    <?php
    
    namespace App\Infrastructure\Repository;
    
    /**
     * @extends AbstractRepository<User>
     */
    class UserRepository extends AbstractRepository
    {
        public function create(User $user): int
        {
            return $this->store($user);
        }
    }
    ```
5. Исправляем класс `App\Domain\Service\UserService`
    ```php
    <?php
    
    namespace App\Domain\Service;
    
    use App\Domain\Entity\User;
    use App\Infrastructure\Repository\UserRepository;
    
    class UserService
    {
        public function __construct(private readonly UserRepository $userRepository)
        {
        }
    
        public function create(string $login): User
        {
            $user = new User();
            $user->setLogin($login);
            $user->setCreatedAt();
            $user->setUpdatedAt();
            $this->userRepository->create($user);
    
            return $user;
        }
    }
    ```
6. Заходим по адресу `http://localhost:7777/world/hello`, видим данные нашего пользователя с заполненным id
7. Проверяем, что запись в БД также создалась

## Добавляем сущность твита и создаём пару твитов для пользователя

1. Создаём класс `App\Domain\Entity\Tweet`
    ```php
    <?php
    
    namespace App\Domain\Entity;
    
    use DateTime;
    use Doctrine\ORM\Mapping as ORM;
    
    #[ORM\Table(name: 'tweet')]
    #[ORM\Entity]
    class Tweet implements EntityInterface
    {
        #[ORM\Column(name: 'id', type: 'bigint', unique: true)]
        #[ORM\Id]
        #[ORM\GeneratedValue(strategy: 'IDENTITY')]
        private ?int $id = null;
    
        #[ORM\ManyToOne(targetEntity: 'User', inversedBy: 'tweets')]
        #[ORM\JoinColumn(name: 'author_id', referencedColumnName: 'id')]
        private User $author;
    
        #[ORM\Column(type: 'string', length: 140, nullable: false)]
        private string $text;
    
        #[ORM\Column(name: 'created_at', type: 'datetime', nullable: false)]
        private DateTime $createdAt;
    
        #[ORM\Column(name: 'updated_at', type: 'datetime', nullable: false)]
        private DateTime $updatedAt;
    
        public function getId(): int
        {
            return $this->id;
        }
    
        public function setId(int $id): void
        {
            $this->id = $id;
        }
    
        public function getAuthor(): User
        {
            return $this->author;
        }
    
        public function setAuthor(User $author): void
        {
            $this->author = $author;
        }
    
        public function getText(): string
        {
            return $this->text;
        }
    
        public function setText(string $text): void
        {
            $this->text = $text;
        }
    
        public function getCreatedAt(): DateTime {
            return $this->createdAt;
        }
    
        public function setCreatedAt(): void {
            $this->createdAt = new DateTime();
        }
    
        public function getUpdatedAt(): DateTime {
            return $this->updatedAt;
        }
    
        public function setUpdatedAt(): void {
            $this->updatedAt = new DateTime();
        }
    
        public function toArray(): array
        {
            return [
                'id' => $this->id,
                'login' => $this->author->getLogin(),
                'createdAt' => $this->createdAt->format('Y-m-d H:i:s'),
                'updatedAt' => $this->updatedAt->format('Y-m-d H:i:s'),
            ];
        }
    }
    ```
2. Исправляем класс `App\Domain\Entity\User`
    1. Добавляем новое поле `tweets` и конструктор
        ```php
        #[ORM\OneToMany(targetEntity: Tweet::class, mappedBy: 'author')]
        private Collection $tweets;
    
        public function __construct()
        {
            $this->tweets = new ArrayCollection();
        }
        ```
    2. Исправляем метод `toArray`
        ```php
        public function toArray(): array
        {
            return [
               'id' => $this->id,
               'login' => $this->login,
               'createdAt' => $this->createdAt->format('Y-m-d H:i:s'),
               'updatedAt' => $this->updatedAt->format('Y-m-d H:i:s'),
               'tweets' => array_map(static fn(Tweet $tweet) => $tweet->toArray(), $this->tweets->toArray()),
            ];
        }        
        ```
3. Создаём класс `App\Infrastructure\Repository\TweetRepository`
    ```php
    <?php
    
    namespace App\Infrastructure\Repository;
    
    use App\Domain\Entity\Tweet;
    
    /**
     * @extends AbstractRepository<Tweet>
     */
    class TweetRepository extends AbstractRepository
    {
        public function create(Tweet $tweet): int
        {
            return $this->store($tweet);
        }
    }
    ```
4. Создаём класс `App\Domain\Service\TweetService`
    ```php
    <?php
    
    namespace App\Domain\Service;
    
    use App\Domain\Entity\Tweet;
    use App\Domain\Entity\User;
    use App\Infrastructure\Repository\TweetRepository;
    
    class TweetService
    {
        public function __construct(private readonly TweetRepository $tweetRepository)
        {
        }
    
        public function postTweet(User $author, string $text): void
        {
            $tweet = new Tweet();
            $tweet->setAuthor($author);
            $tweet->setText($text);
            $tweet->setCreatedAt();
            $tweet->setUpdatedAt();
            $this->tweetRepository->create($tweet);
        }
    }
    ```
5. Создаём класс `App\Domain\Service\UserBuilderService`
    ```php
    <?php
    
    namespace App\Domain\Service;
    
    use App\Domain\Entity\User;
    
    class UserBuilderService
    {
        public function __construct(
            private readonly TweetService $tweetService,
            private readonly UserService $userService,
        ) {
        }
    
        /**
         * @param string[] $texts
         */
        public function createUserWithTweets(string $login, array $texts): User
        {
            $user = $this->userService->create($login);
            foreach ($texts as $text) {
                $this->tweetService->postTweet($user, $text);
            }
    
            return $user;
        }
    }
    ```
6. Исправляем класс `App\Controller\WorldController`
    ```php
    <?php
    
    namespace App\Controller;
    
    use App\Domain\Service\UserBuilderService;
    use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
    use Symfony\Component\HttpFoundation\Response;
    
    class WorldController extends AbstractController
    {
        public function __construct(private readonly UserBuilderService $userBuilderService)
        {
        }
    
        public function hello(): Response
        {
            $user = $this->userBuilderService->createUserWithTweets(
                'J.R.R. Tolkien',
                ['The Hobbit', 'The Lord of the Rings']
            );
    
            return $this->json($user->toArray());
        }
    }
    ```
7. Заходим по адресу `http://localhost:7777/world/hello`, видим данные нашего пользователя с пустым списком твитов
8. Проверяем, что в БД твиты появились

## Исправляем проблему с помощью обновления сущности через EntityManager

1. Добавляем в класс `App\Infrastructure\Repository\AbstractRepository` новый метод:
    ```php
    /**
     * @param T $entity
     * @throws ORMException
     */
    public function refresh(EntityInterface $entity): void
    {
        $this->entityManager->refresh($entity);
    }
    ```
2. Добавляем в классе `App\Domain\Service\UserService` метод `refresh`
    ```php
    public function refresh(User $user): void
    {
        $this->userRepository->refresh($user);
    }
    ```
3. Исправляем в классе `App\Service\UserBuilderService` метод `createUserWithTweets`
    ```php
    /**
     * @param string[] $texts
     */
    public function createUserWithTweets(string $login, array $texts): User
    {
        $user = $this->userService->create($login);
        foreach ($texts as $text) {
            $this->tweetService->postTweet($user, $text);
        }
        $this->userService->refresh($user);

        return $user;
    }
    ```
4. Заходим по адресу `http://localhost:7777/world/hello`, видим твиты появились в данных пользователя

## Исправляем проблему с помощью работы с коллекцией

1. В классе `App\Entity\User` добавляем новый метод `addTweet`
    ```php
    public function addTweet(Tweet $tweet): void
    {
        if (!$this->tweets->contains($tweet)) {
            $this->tweets->add($tweet);
        }
    }
    ```
2. В классе `App\Domain\Service\TweetService` исправляем метод `postTweet`
    ```php
    public function postTweet(User $author, string $text): void
    {
        $tweet = new Tweet();
        $tweet->setAuthor($author);
        $tweet->setText($text);
        $tweet->setCreatedAt();
        $tweet->setUpdatedAt();
        $author->addTweet($tweet);
        $this->tweetRepository->create($tweet);
    }
    ```
3. В классе `App\Service\UserBuilderService` возвращаем предыдущую версию метода `createUserWithTweets`
    ```php
    /**
     * @param string[] $texts
     */
    public function createUserWithTweets(string $login, array $texts): User
    {
        $user = $this->userService->create($login);
        foreach ($texts as $text) {
            $this->tweetService->postTweet($user, $text);
        }

        return $user;
    }
    ```
4. Заходим по адресу `http://localhost:7777/world/hello`, видим, что твиты в данных всё ещё присутствуют

## Добавляем самоссылающуюся связь многие-ко-многим к сущности пользователя

1. Исправляем класс `App\Domain\Entity\User`
    1. Добавляем два новых поля `followers` и `authors` и инициализируем их в конструкторе
        ```php
        #[ORM\ManyToMany(targetEntity: 'User', mappedBy: 'followers')]
        private Collection $authors;
 
        #[ORM\ManyToMany(targetEntity: 'User', inversedBy: 'authors')]
        #[ORM\JoinTable(name: 'author_follower')]
        #[ORM\JoinColumn(name: 'author_id', referencedColumnName: 'id')]
        #[ORM\InverseJoinColumn(name: 'follower_id', referencedColumnName: 'id')]
        private Collection $followers;
    
        public function __construct()
        {
            $this->tweets = new ArrayCollection();
            $this->authors = new ArrayCollection();
            $this->followers = new ArrayCollection();
        }
        ```
    2. Добавляем метод `addFollower`
        ```php
        public function addFollower(User $follower): void
        {
            if (!$this->followers->contains($follower)) {
                $this->followers->add($follower);
            }
        }
        ```
    3. Исправляем метод `toArray`
        ```php
        public function toArray(): array
        {
            return [
                'id' => $this->id,
                'login' => $this->login,
                'createdAt' => $this->createdAt->format('Y-m-d H:i:s'),
                'updatedAt' => $this->updatedAt->format('Y-m-d H:i:s'),
                'tweets' => array_map(static fn(Tweet $tweet) => $tweet->toArray(), $this->tweets->toArray()),
                'followers' => array_map(static fn(User $user) => $user->getLogin(), $this->followers->toArray()),
                'authors' => array_map(static fn(User $user) => $user->getLogin(), $this->authors->toArray()),
            ];
        }
        ```
2. Добавляем в класс `App\Infrastructure\Repository\UserRepository` новый метод `subscribeUser`
    ```php 
    public function subscribeUser(User $author, User $follower): void
    {
        $author->addFollower($follower);
        $this->flush();
    }
    ```
3. Добавляем в класс `App\Domain\Service\UserService` новый метод `subscribeUser`
    ```php
    public function subscribeUser(User $author, User $follower): void
    {
        $this->userRepository->subscribeUser($author, $follower);
    }
    ```
4. Добавляем в класс `App\Service\UserBuilderService` новый метод `createUserWithFollower`
    ```php
    /**
     * @return User[]
     */
    public function createUserWithFollower(string $login, string $followerLogin): array
    {
        $user = $this->userService->create($login);
        $follower = $this->userService->create($followerLogin);
        $this->userService->subscribeUser($user, $follower);

        return [$user, $follower];
    }
    ```
5. В классе `App\Controller\WorldController` исправляем метод `hello`
    ```php
    public function hello(): Response
    {
        $users = $this->userBuilderService->createUserWithFollower(
            'J.R.R. Tolkien',
            'Ivan Ivanov'
        );

        return $this->json(array_map(static fn(User $user) => $user->toArray(), $users));
    }
    ```
6. Заходим по адресу `http://localhost:7777/world/hello`, видим, что поле `followers` заполнилось, а вот поле
   `authors` - нет

## Исправляем возникшую проблему

1. Добавляем в класс `App\Domain\Entity\User` новый метод `addAuthor`
    ```php
    public function addAuthor(User $author): void
    {
        if (!$this->authors->contains($author)) {
            $this->authors->add($author);
        }
    }
    ```
2. Исправляем в классе `App\Infrastructure\Repository\UserRepository` метод `subscribeUser`
    ```php
    public function subscribeUser(User $author, User $follower): void
    {
        $author->addFollower($follower);
        $follower->addAuthor($author);
        $this->entityManager->flush();
    }
    ```
3. Заходим по адресу `http://localhost:7777/world/hello`, видим, что оба поля `followers` и `authors` заполнились

## Добавляем сущность подписки (альтернативная реализация связи многие-ко-многим)

1. Добавляем класс `App\Domain\Entity\Subscription`
    ```php
    <?php
    
    namespace App\Domain\Entity;
    
    use DateTime;
    use Doctrine\ORM\Mapping as ORM;
    
    #[ORM\Table(name: 'subscription')]
    #[ORM\Entity]
    class Subscription implements EntityInterface
    {
        #[ORM\Column(name: 'id', type: 'bigint', unique: true)]
        #[ORM\Id]
        #[ORM\GeneratedValue(strategy: 'IDENTITY')]
        private int $id;
    
        #[ORM\ManyToOne(targetEntity: 'User', inversedBy: 'subscriptionFollowers')]
        #[ORM\JoinColumn(name: 'author_id', referencedColumnName: 'id')]
        private User $author;
    
        #[ORM\ManyToOne(targetEntity: 'User', inversedBy: 'subscriptionAuthors')]
        #[ORM\JoinColumn(name: 'follower_id', referencedColumnName: 'id')]
        private User $follower;
    
        #[ORM\Column(name: 'created_at', type: 'datetime', nullable: false)]
        private DateTime $createdAt;
    
        #[ORM\Column(name: 'updated_at', type: 'datetime', nullable: false)]
        private DateTime $updatedAt;
    
        public function getId(): int
        {
            return $this->id;
        }
    
        public function setId(int $id): void
        {
            $this->id = $id;
        }
    
        public function getAuthor(): User
        {
            return $this->author;
        }
    
        public function setAuthor(User $author): void
        {
            $this->author = $author;
        }
    
        public function getFollower(): User
        {
            return $this->follower;
        }
    
        public function setFollower(User $follower): void
        {
            $this->follower = $follower;
        }
    
        public function getCreatedAt(): DateTime {
            return $this->createdAt;
        }
    
        public function setCreatedAt(): void {
            $this->createdAt = new DateTime();
        }
    
        public function getUpdatedAt(): DateTime {
            return $this->updatedAt;
        }
    
        public function setUpdatedAt(): void {
            $this->updatedAt = new DateTime();
        }
    }
    ```
2. Исправляем класс `App\Domain\Entity\User`
    1. Добавляем два новых поля `subscriptionAuthors` и `subscriptionFollowers` и инициализируем их в конструкторе
        ```php
        #[ORM\OneToMany(mappedBy: 'follower', targetEntity: 'Subscription')]
        private Collection $subscriptionAuthors;
 
        #[ORM\OneToMany(mappedBy: 'author', targetEntity: 'Subscription')]
        private Collection $subscriptionFollowers;
        
        public function __construct()
        {
            $this->tweets = new ArrayCollection();
            $this->authors = new ArrayCollection();
            $this->followers = new ArrayCollection();
            $this->subscriptionAuthors = new ArrayCollection();
            $this->subscriptionFollowers = new ArrayCollection();
        }
        ```
    2. Добавляем два новых метода `addSubscriptionAuthor` и `addSubscriptionFollower`
        ```php
        public function addSubscriptionAuthor(Subscription $subscription): void
        {
            if (!$this->subscriptionAuthors->contains($subscription)) {
                $this->subscriptionAuthors->add($subscription);
            }
        }
        
        public function addSubscriptionFollower(Subscription $subscription): void
        {
            if (!$this->subscriptionFollowers->contains($subscription)) {
                $this->subscriptionFollowers->add($subscription);
            }
        }
        ```
    3. Исправляем метод `toArray`
        ```php
        public function toArray(): array
        {
            return [
                'id' => $this->id,
                'login' => $this->login,
                'createdAt' => $this->createdAt->format('Y-m-d H:i:s'),
                'updatedAt' => $this->updatedAt->format('Y-m-d H:i:s'),
                'tweets' => array_map(static fn(Tweet $tweet) => $tweet->toArray(), $this->tweets->toArray()),
                'followers' => array_map(
                    static fn(User $user) => ['id' => $user->getId(), 'login' => $user->getLogin()],
                    $this->followers->toArray()
                ),
                'authors' => array_map(
                    static fn(User $user) => ['id' => $user->getId(), 'login' => $user->getLogin()],
                    $this->authors->toArray()
                ),
                'subscriptionFollowers' => array_map(
                    static fn(Subscription $subscription) => [
                        'subscriptionId' => $subscription->getId(),
                        'userId' => $subscription->getFollower()->getId(),
                        'login' => $subscription->getFollower()->getLogin(),
                    ],
                    $this->subscriptionFollowers->toArray()
                ),
                'subscriptionAuthors' => array_map(
                    static fn(Subscription $subscription) => [
                        'subscriptionId' => $subscription->getId(),
                        'userId' => $subscription->getAuthor()->getId(),
                        'login' => $subscription->getAuthor()->getLogin(),
                    ],
                    $this->subscriptionAuthors->toArray()
                ),
            ];
        }
        ```
3. Создаём класс `App\Infrastructure\Repository\SubscriptionRepository`
    ```php
    <?php
    
    namespace App\Infrastructure\Repository;
    
    use App\Domain\Entity\Subscription;
    
    /**
     * @extends AbstractRepository<Subscription>
     */
    class SubscriptionRepository extends AbstractRepository
    {
        public function create(Subscription $subscription): int
        {
            return $this->store($subscription);
        }
    }
    ```
4. Добавляем класс `App\Domain\Service\SubscriptionService`
    ```php
    <?php
    
    namespace App\Domain\Service;
    
    use App\Domain\Entity\Subscription;
    use App\Domain\Entity\User;
    use App\Infrastructure\Repository\SubscriptionRepository;
    
    class SubscriptionService
    {
        public function __construct(private readonly SubscriptionRepository $subscriptionRepository)
        {
        }
    
        public function addSubscription(User $author, User $follower): void
        {
            $subscription = new Subscription();
            $subscription->setAuthor($author);
            $subscription->setFollower($follower);
            $subscription->setCreatedAt();
            $subscription->setUpdatedAt();
            $author->addSubscriptionFollower($subscription);
            $follower->addSubscriptionAuthor($subscription);
            $this->subscriptionRepository->create($subscription);
        }
    }
    ```
5. В классе `App\Domain\Service\UserBuilderService`
    1. Добавляем инъекцию `App\Domain\Service\SubscriptionService`
    2. Исправляем метод `createUserWithFollower`
        ```php
        /**
         * @return User[]
         */
        public function createUserWithFollower(string $login, string $followerLogin): array
        {
            $user = $this->userService->create($login);
            $follower = $this->userService->create($followerLogin);
            $this->userService->subscribeUser($user, $follower);
            $this->subscriptionService->addSubscription($user, $follower);
    
            return [$user, $follower];
        }
        ```
6. Заходим по адресу `http://localhost:7777/world/hello`, видим, что значения полей `subscriptionId` и `userId`
   отличаются

## Добавляем поиск по логину

1. Добавляем в класс `App\Infrastructure\Repository\UserRepository` метод `findUsersByLogin`
    ```php
    /**
     * @return User[]
     */
    public function findUsersByLogin(string $name): array
    {
        return $this->entityManager->getRepository(User::class)->findBy(['login' => $name]);
    }
    ```
2. Добавляем в класс `App\Domain\Service\UserService` метод `findUsersByLogin`
    ```php
    /**
     * @return User[]
     */
    public function findUsersByLogin(string $login): array
    {
        return $this->userRepository->findUsersByLogin($login);
    }
    ```
3. Исправляем класс `App\Controller\WorldController`
    ```php
    <?php
    
    namespace App\Controller;
    
    use App\Domain\Entity\User;
    use App\Domain\Service\UserService;
    use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
    use Symfony\Component\HttpFoundation\Response;
    
    class WorldController extends AbstractController
    {
        public function __construct(private readonly UserService $userService)
        {
        }
    
        public function hello(): Response
        {
            $users = $this->userService->findUsersByLogin('Ivan Ivanov');
    
            return $this->json(array_map(static fn(User $user) => $user->toArray(), $users));
        }
    }
    ```
3. Заходим по адресу `http://localhost:7777/world/hello`, видим список добавленных нами ранее пользователей

## Добавляем поиск по критерию

1. Добавляем в класс `App\Infrastructure\Repository\UserRepository` метод `findUsersByLoginWithCriteria`
    ```php
    /**
     * @return User[]
     */
    public function findUsersByLoginWithCriteria(string $login): array
    {
        $criteria = Criteria::create();
        $criteria->andWhere(Criteria::expr()?->eq('login', $login));
        $repository = $this->entityManager->getRepository(User::class);

        return $repository->matching($criteria)->toArray();
    }
    ```
2. Добавляем в класс `App\Domain\Service\UserService` метод `findUsersByCriteria`
    ```php
    /**
     * @return User[]
     */
    public function findUsersByLoginWithCriteria(string $login): array
    {
        return $this->userRepository->findUsersByLoginWithCriteria($login);
    }
    ```
3. В классе `App\Controller\WorldController` исправляем метод `hello`
    ```php
    public function hello(): Response
    {
        $users = $this->userService->findUsersByLoginWithCriteria('J.R.R. Tolkien');

        return $this->json(array_map(static fn(User $user) => $user->toArray(), $users));
    }
    ```
4. Заходим по адресу `http://localhost:7777/world/hello`, видим список добавленных нами ранее пользователей



# Doctrine Migrations

Запускаем контейнеры командой `docker-compose up -d`

## Удаляем ручную миграцию и создаём автоматическую

1. Заходим в контейнер `php` командой `docker exec -it php sh`. Дальнейшие команды выполняются из контейнера
2. Очищаем БД, удаляя все таблицы запросом
    ```sql
    DROP TABLE author_follower;
    DROP TABLE subscription;
    DROP TABLE tweet;
    DROP TABLE feed;
    DROP TABLE "user";
    DROP TABLE doctrine_migration_versions;
    ```
3. Удаляем файл `migrations/Version20230305155514.php`
4. Генерируем новый файл миграции командой `php bin/console doctrine:migrations:diff`
5. Открываем сгенерированный файл и видим в методе `down` команду `$this->addSql('CREATE SCHEMA public');`

## Добавляем EventListener и перегенерируем миграцию

1. Создаём класс `App\Application\Doctrine\PostGenerateSchemaEventListener`
    ```php
    <?php
    
    namespace App\Application\Doctrine;
    
    use Doctrine\Bundle\DoctrineBundle\Attribute\AsDoctrineListener;
    use Doctrine\DBAL\Schema\SchemaException;
    use Doctrine\ORM\Tools\Event\GenerateSchemaEventArgs;
    use Doctrine\ORM\Tools\ToolEvents;
    
    #[AsDoctrineListener(event: ToolEvents::postGenerateSchema, connection: 'default')]
    class PostGenerateSchemaEventListener
    {
        /**
         * @throws SchemaException
         */
        public function postGenerateSchema(GenerateSchemaEventArgs $args): void
        {
            $schema = $args->getSchema();
            if (!$schema->hasNamespace('public')) {
                $schema->createNamespace('public');
            }
        }
    }
     ```
2. Удаляем неправильно сгенерированный файл и перегенерируем его командой `php bin/console doctrine:migrations:diff`
3. Видим, что в сгенерированном файле ненужная команда не появилась

## Исправляем атрибуты, перегенерируем миграцию и правим вручную то, что нельзя получить автоматически

1. Обращаем внимание, что имена индексов в миграции сгенерированы автоматически
2. Добавляем к классу `App\Entity\Tweet` атрибут
    ```php
    #[ORM\Index(name: 'tweet__author_id__ind', columns: ['author_id'])]
    ```
3. Добавляем к классу `App\Entity\Subscription` атрибуты
    ```php
    #[ORM\Index(name: 'subscription__author_id__ind', columns: ['author_id'])]
    #[ORM\Index(name: 'subscription__follower_id__ind', columns: ['follower_id'])]
    ```
4. Удаляем неправильно сгенерированный файл и перегенерируем его командой
   `php bin/console doctrine:migrations:diff`
5. Исправляем в сгенерированной миграции вручную оставшиеся автоматически сгенерированными имена индексов и имена
   внешних ключей
    1. в функции `up`:
        ```php
        public function up(Schema $schema) : void
        {
           $this->addSql('CREATE TABLE subscription (id BIGINT GENERATED BY DEFAULT AS IDENTITY NOT NULL, created_at TIMESTAMP(0) WITHOUT TIME ZONE NOT NULL, updated_at TIMESTAMP(0) WITHOUT TIME ZONE NOT NULL, author_id BIGINT DEFAULT NULL, follower_id BIGINT DEFAULT NULL, PRIMARY KEY(id))');
           $this->addSql('CREATE INDEX subscription__author_id__ind ON subscription (author_id)');
           $this->addSql('CREATE INDEX subscription__follower_id__ind ON subscription (follower_id)');
           $this->addSql('CREATE TABLE tweet (id BIGINT GENERATED BY DEFAULT AS IDENTITY NOT NULL, text VARCHAR(140) NOT NULL, created_at TIMESTAMP(0) WITHOUT TIME ZONE NOT NULL, updated_at TIMESTAMP(0) WITHOUT TIME ZONE NOT NULL, author_id BIGINT DEFAULT NULL, PRIMARY KEY(id))');
           $this->addSql('CREATE INDEX tweet__author_id__ind ON tweet (author_id)');
           $this->addSql('CREATE TABLE "user" (id BIGINT GENERATED BY DEFAULT AS IDENTITY NOT NULL, login VARCHAR(32) NOT NULL, created_at TIMESTAMP(0) WITHOUT TIME ZONE NOT NULL, updated_at TIMESTAMP(0) WITHOUT TIME ZONE NOT NULL, PRIMARY KEY(id))');
           $this->addSql('CREATE TABLE author_follower (author_id BIGINT NOT NULL, follower_id BIGINT NOT NULL, PRIMARY KEY(author_id, follower_id))');
           $this->addSql('CREATE INDEX author_follower__author_id__ind ON author_follower (author_id)');
           $this->addSql('CREATE INDEX author_follower__follower_id__ind ON author_follower (follower_id)');
           $this->addSql('ALTER TABLE subscription ADD CONSTRAINT subscription__author_id__fk FOREIGN KEY (author_id) REFERENCES "user" (id) NOT DEFERRABLE INITIALLY IMMEDIATE');
           $this->addSql('ALTER TABLE subscription ADD CONSTRAINT subscription__follower_id__fk FOREIGN KEY (follower_id) REFERENCES "user" (id) NOT DEFERRABLE INITIALLY IMMEDIATE');
           $this->addSql('ALTER TABLE tweet ADD CONSTRAINT tweet__author_id__fk FOREIGN KEY (author_id) REFERENCES "user" (id) NOT DEFERRABLE INITIALLY IMMEDIATE');
           $this->addSql('ALTER TABLE author_follower ADD CONSTRAINT author_follower__author_id__fk FOREIGN KEY (author_id) REFERENCES "user" (id) NOT DEFERRABLE INITIALLY IMMEDIATE');
           $this->addSql('ALTER TABLE author_follower ADD CONSTRAINT author_follower__follower_id__fk FOREIGN KEY (follower_id) REFERENCES "user" (id) NOT DEFERRABLE INITIALLY IMMEDIATE');
        }
        ```
    2. в функции `down`:
        ```php
        public function down(Schema $schema) : void
        {
           $this->addSql('ALTER TABLE subscription DROP CONSTRAINT author_follower__follower_id__fk');
           $this->addSql('ALTER TABLE subscription DROP CONSTRAINT author_follower__author_id__fk');
           $this->addSql('ALTER TABLE tweet DROP CONSTRAINT tweet__author_id__fk');
           $this->addSql('ALTER TABLE author_follower DROP CONSTRAINT subscription__follower_id__fk');
           $this->addSql('ALTER TABLE author_follower DROP CONSTRAINT subscription__author_id__fk');
           $this->addSql('DROP TABLE subscription');
           $this->addSql('DROP TABLE tweet');
           $this->addSql('DROP TABLE "user"');
           $this->addSql('DROP TABLE author_follower');
        }
        ```

## Выполняем миграции и возвращаем соответствие схемы и БД

1. Выполняем миграцию командой `php bin/console doctrine:migrations:migrate`
2. Ещё раз генерируем миграцию, выравнивающую схему БД с описаниями Entity командой
   `php bin/console doctrine:migrations:diff`
3. Заходим в сгенерированный файл и видим, что имена индексов для отношения many-to-many переопределить не удаётся
4. Накатываем миграцию командой `php bin/console doctrine:migrations:migrate`
5. Проверяем в БД, что имена индексов изменились
6. Откатываем миграцию командой `php bin/console doctrine:migrations:migrate VERSION`, где VERSION - FQN класса с
   первой миграцией, создающей все таблицы (с экранированием обратного слэша)
7. Проверяем в БД, что имена индексов снова стали осмысленными
8. Снова накатываем последнюю миграцию командой `php bin/console doctrine:migrations:migrate`
9. Ещё раз генерируем выравнивающую миграцию командой `php bin/console doctrine:migrations:diff`, видим ошибку,
   говорящую о том, что расхождений больше нет

## Валидируем схему и игнорирум таблицу с версиями миграций

1. Выполняем команду `php bin/console doctrine:schema:validate`, видим ошибки
2. Выполняем команду `php bin/console doctrine:schema:update --dump-sql`, видим удаление таблицы с историей миграций
3. В файле `config/packages/doctrine.yaml` добавляем в секцию `doctrine.dbal` новое поле
    ```yaml
    schema_filter: ~^(?!doctrine_)~
    ```
4. Ещё раз выполняем команду `php bin/console doctrine:schema:validate`, видим успешный результат

## Добавляем EventListener для заполнения мета-полей

1. Создаём интерфейс `App\Domain\Entity\HasMetaTimestampsInterface`
    ```php
    <?php
   
    namespace App\Domain\Entity;
   
    interface HasMetaTimestampsInterface
    {
        public function setCreatedAt(): void;
   
        public function setUpdatedAt(): void;
    }
    ```
2. Реализуем созданный интерфейс в классе `App\Domain\Entity\User` (нужные методы уже есть)
3. Создаём класс `App\Application\Doctrine\MetaTimestampsPrePersistEventListener`
    ```php
    <?php
    
    namespace App\Application\Doctrine;
    
    use App\Domain\Entity\HasMetaTimestampsInterface;
    use Doctrine\Bundle\DoctrineBundle\Attribute\AsDoctrineListener;
    use Doctrine\ORM\Events;
    use Doctrine\Persistence\Event\LifecycleEventArgs;
    
    #[AsDoctrineListener(event: Events::prePersist, connection: 'default')]
    class MetaTimestampsPrePersistEventListener
    {
        public function prePersist(LifecycleEventArgs $event): void
        {
            $entity = $event->getObject();
    
            if ($entity instanceof HasMetaTimestampsInterface) {
                $entity->setCreatedAt();
                $entity->setUpdatedAt();
            }
        }
    }
    ```
4. В классе `App\Domain\Service\UserService` исправляем метод `create`
    ```php
    public function create(string $login): User
    {
        $user = new User();
        $user->setLogin($login);
        $this->userRepository->create($user);
   
        return $user;
    }
    ```
5. В классе `App\Controller\WorldController` исправляем метод `hello`
    ```php
    public function hello(): Response
    {
        $user = $this->userService->create('J.R.R. Tolkien');

        return $this->json($user->toArray());
    }
    ```
6. Заходим по адресу `http://localhost:7777/world/hello`, видим данные нашего пользователя с проставленным временем
   создания и редактирования

## Добавляем атрибуты Entity Lifecycle для сущности User

1. Удаляем класс `App\Application\Doctrine\MetaTimestampsPrePersistEventListener`
2. В классе `App\Domain\Entity\User`
    1. Добавляем атрибут класса
        ```php
        #[ORM\HasLifecycleCallbacks]
        ```
    2. Исправляем методы `setCreatedAt` и `setUpdatedAt`, добавляя к каждому атрибут
        ```php
        #[ORM\PrePersist]
        ```
3. Заходим по адресу `http://localhost:7777/world/hello`, видим, что время в пользователе снова проставилось

## Добавляем редактирование для сущности User

1. В классе `App\Domain\Entity\User` добавляем для метода `setUpdatedAt` атрибут
    ```php
    #[ORM\PreUpdate]
    ```
2. В классе `App\Infrastructure\Repository\UserRepository` добавляем новые методы `find` и `updateLogin`
    ```php
    public function find(int $userId): ?User
    {
        $repository = $this->entityManager->getRepository(User::class);
        /** @var User|null $user */
        $user = $repository->find($userId);

        return $user;
    }
   
    public function updateLogin(User $user, string $login): void
    {
        $user->setLogin($login);
        $this->flush();
    }
    ```
3. В классе `App\Domain\Service\UserService` добавляем новый метод `updateUserLogin`
    ```php
    public function updateUserLogin(int $userId, string $login): ?User
    {
        $user = $this->userRepository->find($userId);
        if (!($user instanceof User)) {
            return null;
        }
        $this->userRepository->updateLogin($user, $login);

        return $user;
    }
    ```
4. В классе `App\Controller\WorldController` исправляем метод `hello` (в вызове `updateUserLogin` используем любой ID,
   который реально существует в БД)
    ```php
    public function hello(): Response
    {
        $user = $this->userService->updateUserLogin(1, 'My new user');
        [$data, $code] = $user === null ? [null, Response::HTTP_NOT_FOUND] : [$user->toArray(), Response::HTTP_OK];

        return $this->json($data, $code);
    }
    ```
5. Заходим по адресу `http://localhost:7777/world/hello`, видим, что поле `updatedAt` обновилось

## Добавляем использование QueryBuilder для select-запроса

1. В классе `App\Infrastructure\Repository\UserRepository` добавляем новый метод `findUsersByLoginWithQueryBuilder`
    ```php
    public function findUsersByLoginWithQueryBuilder(string $login): array
    {
        $queryBuilder = $this->entityManager->createQueryBuilder();
        $queryBuilder->select('u')
            ->from(User::class, 'u')
            ->andWhere($queryBuilder->expr()->like('u.login',':userLogin'))
            ->setParameter('userLogin', "%$login%");

        return $queryBuilder->getQuery()->getResult();
    }
    ```
2. В класс `App\Domain\Service\UserService` добавляем новый метод `findUsersByLoginWithQueryBuilder`
    ```php
    public function findUsersByLoginWithQueryBuilder(string $login): array
    {
        return $this->userRepository->findUsersByLoginWithQueryBuilder($login);
    }
    ```
3. В классе `App\Controller\WorldController` исправляем метод `hello`
    ```php
    public function hello(): Response
    {
        $users = $this->userService->findUsersByLoginWithQueryBuilder('Tolkien');
    
        return $this->json(array_map(static fn(User $user) => $user->toArray(), $users));
    }
    ```
4. Заходим по адресу `http://localhost:7777/world/hello`, видим найденные записи

## Добавляем использование QueryBuilder для update-запроса

1. В класс `App\Infrastructure\Repository\UserRepository` добавляем новый метод `updateUserLoginWithQueryBuilder`
    ```php
    public function updateUserLoginWithQueryBuilder(int $userId, string $login): void
    {
        $queryBuilder = $this->entityManager->createQueryBuilder();
        $queryBuilder->update(User::class,'u')
            ->set('u.login', ':userLogin')
            ->where($queryBuilder->expr()->eq('u.id', ':userId'))
            ->setParameter('userId', $userId)
            ->setParameter('userLogin', $login);

        $queryBuilder->getQuery()->execute();
    }
    ```
2. В класс `App\Domain\Service\UserService` добавляем новый метод `updateUserLoginWithQueryBuilder`
    ```php
    public function updateUserLoginWithQueryBuilder(int $userId, string $login): ?User
    {
        $user = $this->userRepository->find($userId);
        if (!($user instanceof User)) {
            return null;
        }
        $this->userRepository->updateUserLoginWithQueryBuilder($user->getId(), $login);
        
        return $user;
    }
    ```
3. В классе `App\Controller\WorldController` исправляем метод `hello` (используем любой ID, который реально существует
   в БД)
    ```php
    public function hello(): Response
    {
        /** @var User $user */
        $user = $this->userService->updateUserLoginWithQueryBuilder(1, 'User is updated');

        return $this->json($user->toArray());
    }
    ```
4. Заходим по адресу `http://localhost:7777/world/hello`, видим запись со старым логином и старым временем обновления
5. Проверяем, что в БД запись обновилась

## Исправляем QueryBuilder для update-запроса

1. В классе `App\Domain\Service\UserService` исправляем метод `updateUserLoginWithQueryBuilder`
    ```php

    public function updateUserLoginWithQueryBuilder(int $userId, string $login): ?User
    {
        $user = $this->userRepository->find($userId);
        if (!($user instanceof User)) {
            return null;
        }
        $this->userRepository->updateUserLoginWithQueryBuilder($user->getId(), $login);
        $this->userRepository->refresh($user);

        return $user;
    }
    ```
2. В классе `App\Controller\WorldController` исправляем метод `hello` (используем любой ID, который реально существует
   в БД)
    ```php
    public function hello(): Response
    {
        /** @var User $user */
        $user = $this->userService->updateUserLoginWithQueryBuilder(1, 'User is updated again');

        return $this->json($user->toArray());
    }
    ```
3. Заходим по адресу `http://localhost:7777/world/hello`, видим запись с новым логином, но старым временем обновления

## Добавляем DBAL QueryBuilder для update-запроса

1. В класс `App\Infrastructure\Repository\UserRepository` добавляем новый метод `updateUserLoginWithDBALQueryBuilder`
    ```php
    /**
     * @throws \Doctrine\DBAL\Exception
     */
    public function updateUserLoginWithDBALQueryBuilder(int $userId, string $login): void
    {
        $queryBuilder = $this->entityManager->getConnection()->createQueryBuilder();
        $queryBuilder->update('"user"')
            ->set('login', ':userLogin')
            ->where($queryBuilder->expr()->eq('id', ':userId'))
            ->setParameter('userId', $userId)
            ->setParameter('userLogin', $login);

        $queryBuilder->executeStatement();
    }
    ```
2. В класс `App\Domain\Service\UserService` добавляем новый метод `updateUserLoginWithDBALQueryBuilder`
    ```php
    public function updateUserLoginWithDBALQueryBuilder(int $userId, string $login): ?User
    {
        $user = $this->userRepository->find($userId);
        if (!($user instanceof User)) {
            return null;
        }
        $this->userRepository->updateUserLoginWithDBALQueryBuilder($user->getId(), $login);
        $this->userRepository->refresh($user);

        return $user;
    }
    ```
3. В классе `App\Controller\WorldController` исправляем метод `hello`
    ```php
    public function hello(): Response
    {
        /** @var User $user */
        $user = $this->userService->updateUserLoginWithDBALQueryBuilder(1, 'User is updated by DBAL');

        return $this->json($user->toArray());
    }
    ```
4. Заходим по адресу `http://localhost:7777/world/hello`, видим запись с новым логином

## Загружаем связанные таблицы через QueryBuilder

1. В класс `App\Infrastructure\Repository\UserRepository` добавляем новый метод `findUserWithTweetsWithQueryBuilder`
    ```php
    /**
     * @throws NonUniqueResultException
     */
    public function findUserWithTweetsWithQueryBuilder(int $userId): array
    {
        $queryBuilder = $this->entityManager->createQueryBuilder();
        $queryBuilder->select('u')
            ->from(User::class, 'u')
            ->where($queryBuilder->expr()->eq('u.id', ':userId'))
            ->setParameter('userId', $userId);
    
        return $queryBuilder->getQuery()->getOneOrNullResult(AbstractQuery::HYDRATE_ARRAY);
    }
    ```
2. В класс `App\Domain\Service\UserService` добавляем новый метод `findUserWithTweetsWithQueryBuilder`
    ```php
    public function findUserWithTweetsWithQueryBuilder(int $userId): array
    {
        return $this->userRepository->findUserWithTweetsWithQueryBuilder($userId);
    }
    ```
3. Исправляем класс `App\Controller\WorldController`
    ```php
    <?php
    
    namespace App\Controller;
    
    use App\Domain\Service\UserService;
    use App\Domain\Service\UserBuilderService;
    use Doctrine\ORM\NonUniqueResultException;
    use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
    use Symfony\Component\HttpFoundation\Response;
    
    class WorldController extends AbstractController
    {
        public function __construct(
            private readonly UserService $userService,
            private readonly UserBuilderService $userBuilderService,
        ) {
        }
    
        /**
         * @throws NonUniqueResultException
         */
        public function hello(): Response
        {
            $user = $this->userBuilderService->createUserWithTweets(
                'Charles Dickens',
                ['Oliver Twist', 'The Christmas Carol']
            );
            $userData = $this->userService->findUserWithTweetsWithQueryBuilder($user->getId());
    
            return $this->json($userData);
        }
    }
    ```
4. Заходим по адресу `http://localhost:7777/world/hello`, видим, что твиты не подгружаются

## Исправляем запрос для получения связанных таблиц

1. В классе `App\Infrastructure\Repository\UserRepository` исправляем метод `findUserWithTweetsWithQueryBuilder`
    ```php
    /**
     * @throws NonUniqueResultException
     */
    public function findUserWithTweetsWithQueryBuilder(int $userId): array
    {
        $queryBuilder = $this->entityManager->createQueryBuilder();
        $queryBuilder->select('u', 't')
            ->from(User::class, 'u')
            ->leftJoin('u.tweets', 't')
            ->where($queryBuilder->expr()->eq('u.id', ':userId'))
            ->setParameter('userId', $userId);
    
        return $queryBuilder->getQuery()->getOneOrNullResult(AbstractQuery::HYDRATE_ARRAY);
    }
    ```
2. Заходим по адресу `http://localhost:7777/world/hello`, видим, что твиты подгружаются

## Добавляем использование DBAL QueryBuilder для связанных таблиц

1. В класс `App\Infrastructure\Repository\UserRepository` добавляем новый метод `findUserWithTweetsWithDBALQueryBuilder`
    ```php
    /**
     * @throws \Doctrine\DBAL\Exception
     */
    public function findUserWithTweetsWithDBALQueryBuilder(int $userId): array
    {
        $queryBuilder = $this->entityManager->getConnection()->createQueryBuilder();
        $queryBuilder->select('u', 't')
            ->from('"user"', 'u')
            ->leftJoin('u', 'tweet', 't', 'u.id = t.author_id')
            ->where($queryBuilder->expr()->eq('u.id', ':userId'))
            ->setParameter('userId', $userId);
    
        return $queryBuilder->executeQuery()->fetchAllNumeric();
    }
    ```
2. В класс `App\Domain\Service\UserService` добавляем новый метод `findUserWithTweetsWithDBALQueryBuilder`
    ```php
    public function findUserWithTweetsWithDBALQueryBuilder(int $userId): array
    {
        return $this->userRepository->findUserWithTweetsWithDBALQueryBuilder($userId);
    }
    ```
3. В классе `App\Controller\WorldController` исправляем метод `hello`
    ```php
    /**
     * @throws \Doctrine\DBAL\Exception
     */
    public function hello(): Response
    {
        $user = $this->userBuilderService->createUserWithTweets(
            'Charles Dickens',
            ['Oliver Twist', 'The Christmas Carol']
        );
        $userData = $this->userService->findUserWithTweetsWithDBALQueryBuilder($user->getId());

        return $this->json($userData);
    }
    ```
4. Заходим по адресу `http://localhost:7777/world/hello`, видим результат в виде JSON-строк



# Doctrine. Дополнительные возможности

Запускаем контейнеры командой `docker-compose up -d`

## Добавляем soft-delete через фильтр

1. Добавляем интерфейс `App\Domain\Entity\SoftDeleteableInterface`
    ```php
    <?php
   
    namespace App\Domain\Entity;

    use DateTime;

    interface SoftDeletableInterface
    {
        public function getDeletedAt(): ?DateTime;
   
        public function setDeletedAt(): void;
    }
    ```
2. В классе `App\Domain\Entity\User` имплементируем интерфейс, добавляя новое поле, геттер и сеттер
    ```php
    #[ORM\Column(name: 'deleted_at', type: 'datetime', nullable: true)]
    private ?DateTime $deletedAt = null;

    public function getDeletedAt(): ?DateTime
    {
        return $this->deletedAt;
    }

    public function setDeletedAt(): void
    {
        $this->deletedAt = new DateTime();
    }
    ```
3. Добавляем класс `App\Application\Doctrine\SoftDeletedFilter`
    ```php
    <?php
    
    namespace App\Application\Doctrine;
    
    use App\Domain\Entity\SoftDeletableInterface;
    use Doctrine\ORM\Mapping\ClassMetadata;
    use Doctrine\ORM\Query\Filter\SQLFilter;
    
    class SoftDeletedFilter extends SQLFilter
    {
        public function addFilterConstraint(ClassMetadata $targetEntity, $targetTableAlias): string
        {
            if (!$targetEntity->reflClass->implementsInterface(SoftDeletableInterface::class)) {
                return '';
            }
    
            return $targetTableAlias.'.deleted_at IS NULL';
        }
    }
    ```
4. В файле `config/packages/doctrine.yaml` добавляем в секцию `doctrine.orm` подсекцию `filters`
    ```yaml
    filters:
        soft_delete_filter:
            class: App\Application\Doctrine\SoftDeletedFilter
            enabled: true
    ```
5. В классе `App\Infrastructure\Repository\UserRepository` добавляем метод `remove`
    ```php
    public function remove(User $user): void
    {
        $user->setDeletedAt();
        $this->flush();
    }
    ```
6. В классе `App\Domain\Service\UserService` добавляем метод `removeById`
    ```php
    public function removeById(int $userId): void
    {
        $user = $this->userRepository->find($userId);
        if ($user instanceof User) {
            $this->userRepository->remove($user);
        }
    }
    ```
7. Исправляем класс `App\Controller\WorldController`
    ```php
    <?php
    
    namespace App\Controller;
    
    use App\Domain\Entity\User;
    use App\Domain\Service\UserService;
    use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
    use Symfony\Component\HttpFoundation\Response;
    
    class WorldController extends AbstractController
    {
        public function __construct(
            private readonly UserService $userService,
        ) {
        }
    
        public function hello(): Response
        {
            $user = $this->userService->create('Jack London');
            $this->userService->removeById($user->getId());
            $usersByLogin = $this->userService->findUsersByLogin($user->getLogin());
    
            return $this->json(['users' => array_map(static fn (User $user) => $user->toArray(), $usersByLogin)]);
        }
    }
    ```
8. Входим в контейнер командой `docker exec -it php sh`, дальнейшие команды будут выполняться из контейнера
9. Выполняем команду `php bin/console doctrine:migrations:diff`
10. Проверяем полученную миграцию и применяем её командой `php bin/console doctrine:migrations:migrate`
11. Заходим по адресу `http://localhost:7777/world/hello`, видим в ответе пустой массив, но в БД пользователь
    присутствует

## Добавляем параметризацию для фильтра

1. Исправляем класс `App\Application\Doctrine\SoftDeletedFilter`
    ```php
    <?php
    
    namespace App\Application\Doctrine;
    
    use App\Domain\Entity\SoftDeletableInterface;
    use Doctrine\ORM\Mapping\ClassMetadata;
    use Doctrine\ORM\Query\Filter\SQLFilter;
    
    class SoftDeletedFilter extends SQLFilter
    {
        public function addFilterConstraint(ClassMetadata $targetEntity, $targetTableAlias): string
        {
            if (!$targetEntity->reflClass->implementsInterface(SoftDeletableInterface::class)) {
                return '';
            }
    
        return $this->getParameter('checkTime') ? 
            '('.$targetTableAlias.'.deleted_at IS NULL OR '.$targetTableAlias.'.deleted_at >= current_timestamp)' :
            $targetTableAlias.'.deleted_at IS NULL';
        }
    }
    ```
2. В файле `config/packages/doctrine.yaml` исправляем секцию `doctrine.orm.filters`
    ```yaml
    filters:
        soft_delete_filter:
            class: App\Application\Doctrine\SoftDeletedFilter
            enabled: true
            parameters:
              checkTime: true
    ```
3. Добавляем интерфейс `App\Domain\Entity\SoftDeleteableInFutureInterface`
    ```php
    <?php
   
    namespace App\Domain\Entity;
   
    use DateInterval;
   
    interface SoftDeletableInFutureInterface
    {
        public function setDeletedAtInFuture(DateInterval $dateInterval): void;
    }
    ```
4. В классе `App\Domain\Entity\User` имплементируем интерфейс
    ```php
    public function setDeletedAtInFuture(DateInterval $dateInterval): void
    {
        if ($this->deletedAt === null) {
            $this->deletedAt = new DateTime();
        }
        $this->deletedAt = $this->deletedAt->add($dateInterval);
    }
    ```
5. В классе `App\Infrastructure\Repository\UserRepository` добавляем метод `removeInFuture`
    ```php
    public function removeInFuture(User $user, DateInterval $dateInterval): void
    {
        $user->setDeletedAtInFuture($dateInterval);
        $this->flush();
    }
    ```
6. В классе `App\Domain\Service\UserService` добавляем метод `removeByIdInFuture`
    ```php
    public function removeByIdInFuture(int $userId, DateInterval $dateInterval): void
    {
        $user = $this->userRepository->find($userId);
        if ($user instanceof User) {
            $this->userRepository->removeInFuture($user, $dateInterval);
        }
    }
    ```
7. Исправляем класс `App\Controller\WorldController`
    ```php
    <?php
    
    namespace App\Controller;
    
    use App\Domain\Entity\User;
    use App\Domain\Service\UserService;
    use DateInterval;
    use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
    use Symfony\Component\HttpFoundation\Response;
    
    class WorldController extends AbstractController
    {
        public function __construct(
            private readonly UserService $userService,
        ) {
        }
    
        public function hello(): Response
        {
            $user = $this->userService->create('Terry Pratchett');
            $this->userService->removeByIdInFuture($user->getId(), DateInterval::createFromDateString('+5 min'));
            $usersByLogin = $this->userService->findUsersByLogin($user->getLogin());

            return $this->json(['users' => array_map(static fn (User $user) => $user->toArray(), $usersByLogin)]);
        }
    }
    ```
8. Заходим по адресу `http://localhost:7777/world/hello`, видим в ответе пользователя, хотя поле `deleted_at` заполнено
9. В классе `App\Controller\WorldController` исправляем метод `hello`
    ```php
    public function hello(): Response
    {
        $user = $this->userService->create('William Shakespeare');
        $this->userService->removeById($user->getId());
        $usersByLogin = $this->userService->findUsersByLogin($user->getLogin());

        return $this->json(['users' => array_map(static fn (User $user) => $user->toArray(), $usersByLogin)]);
    }
    ```
10. Заходим по адресу `http://localhost:7777/world/hello`, видим пустой массив

## Отключаем фильтр в коде

1. В классе `App\Infrastructure\Repository\UserRepository` добавляем метод `findUsersByLoginWithDeleted`
    ```php
    /**
     * @return User[]
     */
    public function findUsersByLoginWithDeleted(string $name): array
    {
        $filters = $this->entityManager->getFilters();
        if ($filters->isEnabled('soft_delete_filter')) {
            $filters->disable('soft_delete_filter');
        }
        return $this->entityManager->getRepository(User::class)->findBy(['login' => $name]);
    }
    ```
2. В классе `App\Domain\Service\UserService` добавляем метод `findUsersByLoginWithDeleted`
    ```php
    /**
     * @return User[]
     */
    public function findUsersByLoginWithDeleted(string $login): array
    {
        return $this->userRepository->findUsersByLoginWithDeleted($login);
    }
    ```
3. В классе `App\Controller\WorldController` исправляем метод `hello`
    ```php
    public function hello(): Response
    {
        $user = $this->userService->create('William Shakespeare');
        $this->userService->removeById($user->getId());
        $usersByLogin = $this->userService->findUsersByLoginWithDeleted($user->getLogin());

        return $this->json(['users' => array_map(static fn (User $user) => $user->toArray(), $usersByLogin)]);
    }
    ```
4. Заходим по адресу `http://localhost:7777/world/hello`, видим удалённые записи

## Добавляем кастомный тип для Doctrine

1. Добавляем класс `App\Domain\ValueObject\CommunicationChannel`
    ```php
    <?php
    
    namespace App\Domain\ValueObject;
    
    use RuntimeException;
    
    class CommunicationChannel
    {
        private const EMAIL = 'email';
        private const PHONE = 'phone';
        private const ALLOWED_VALUES = [self::PHONE, self::EMAIL];
    
        private function __construct(private readonly string $value)
        {
        }
    
        public static function fromString(string $value): self
        {
            if (!in_array($value, self::ALLOWED_VALUES, true)) {
                throw new RuntimeException('Invalid communication channel value');
            }
    
            return new self($value);
        }
    
        public function getValue(): string
        {
            return $this->value;
        }
    }
    ```
2. Добавляем класс `App\Application\Doctrine\Types\CommunicationChannelType`
    ```php
    <?php
    
    namespace App\Application\Doctrine\Types;
    
    use App\Domain\ValueObject\CommunicationChannel;
    use Doctrine\DBAL\Platforms\AbstractPlatform;
    use Doctrine\DBAL\Types\Exception\ValueNotConvertible;
    use Doctrine\DBAL\Types\Type;
    use RuntimeException;
    
    class CommunicationChannelType extends Type
    {
        public function convertToPHPValue($value, AbstractPlatform $platform): ?CommunicationChannel
        {
            if ($value === null) {
                return null;
            }
    
            if (is_string($value)) {
                try {
                    return CommunicationChannel::fromString($value);
                } catch (RuntimeException) {
                }
            }
    
            throw ValueNotConvertible::new($value, $this->getName());
        }
    
        public function convertToDatabaseValue($value, AbstractPlatform $platform): ?string
        {
            if ($value === null) {
                return null;
            }
    
            if ($value instanceof CommunicationChannel) {
                return $value->getValue();
            }
    
            throw ValueNotConvertible::new($value, $this->getName());
        }
    
        public function getSQLDeclaration(array $column, AbstractPlatform $platform): string
        {
            return $platform->getStringTypeDeclarationSQL($column);
        }
    
        public function getName()
        {
            return 'communicationChannel';
        }
    }
    ```
3. В файле `config/packages/doctrine.yaml` добавляем в секцию `doctrine.dbal` подсекцию `types`
    ```yaml
    types:
        communicationChannel: App\Application\Doctrine\Types\CommunicationChannelType
    ```
4. В классе `App\Domain\Entity\User`
    1. добавляем новое поле, геттер и сеттер
         ```php
         #[ORM\Column(name: 'communication_channel', type: 'communicationChannel', nullable: true)]
         private ?CommunicationChannel $communicationChannel = null;
  
         public function getCommunicationChannel(): ?CommunicationChannel
         {
             return $this->communicationChannel;
         }
  
         public function setCommunicationChannel(?CommunicationChannel $communicationChannel): void
         {
             $this->communicationChannel = $communicationChannel;
         }
         ```
    2. исправляем метод `toArray`
         ```php
         public function toArray(): array
         {
             return [
                 'id' => $this->id,
                 'login' => $this->login,
                 'createdAt' => $this->createdAt->format('Y-m-d H:i:s'),
                 'updatedAt' => $this->updatedAt->format('Y-m-d H:i:s'),
                 'communicationChannel' => $this->communicationChannel->getValue(),
                 'tweets' => array_map(static fn(Tweet $tweet) => $tweet->toArray(), $this->tweets->toArray()),
                 'followers' => array_map(
                     static fn(User $user) => ['id' => $user->getId(), 'login' => $user->getLogin()],
                     $this->followers->toArray()
                 ),
                 'authors' => array_map(
                     static fn(User $user) => ['id' => $user->getId(), 'login' => $user->getLogin()],
                     $this->authors->toArray()
                 ),
                 'subscriptionFollowers' => array_map(
                     static fn(Subscription $subscription) => [
                         'subscriptionId' => $subscription->getId(),
                         'userId' => $subscription->getFollower()->getId(),
                         'login' => $subscription->getFollower()->getLogin(),
                     ],
                     $this->subscriptionFollowers->toArray()
                 ),
                 'subscriptionAuthors' => array_map(
                     static fn(Subscription $subscription) => [
                         'subscriptionId' => $subscription->getId(),
                         'userId' => $subscription->getAuthor()->getId(),
                         'login' => $subscription->getAuthor()->getLogin(),
                     ],
                     $this->subscriptionAuthors->toArray()
                 ),
             ];
         }      
         ```
5. В классе `App\Domain\Service\UserService` исправляем метод `create`
    ```php
    public function create(string $login, string $communicationChannel): User
    {
        $user = new User();
        $user->setLogin($login);
        $user->setCommunicationChannel(CommunicationChannel::fromString($communicationChannel));
        $this->userRepository->create($user);

        return $user;
    }
    ```
6. В классе `App\Controller\WorldController` исправляем метод `hello`
    ```php
    public function hello(): Response
    {
        $user = $this->userService->create('Howard Lovecraft', 'email');

        return $this->json(['user' => $user->toArray()]);
    }
    ```
7. Выполняем команду `php bin/console doctrine:migrations:diff`
8. Проверяем полученную миграцию и применяем её командой `php bin/console doctrine:migrations:migrate`
9. Заходим по адресу `http://localhost:7777/world/hello`, видим установленный канал связи

## Заменяем кастомный тип на перечисление

1. Добавляем перечисление `App\Domain\ValueObject\CommunicationChannelEnum`
    ```php
    <?php
    
    namespace App\Domain\ValueObject;
    
    enum CommunicationChannelEnum: string
    {
        case Email = 'email';
        case Phone = 'phone';
    }
    ```
2. В классе `App\Domain\Entity\User`
    1. Исправляем описание поля `$communicationChannel`, его геттера и сеттера
         ```php
         #[ORM\Column(name: 'communication_channel', type: 'string', nullable: true, enumType: CommunicationChannelEnum::class)]
         private ?CommunicationChannelEnum $communicationChannel = null;
   
         public function getCommunicationChannel(): ?CommunicationChannelEnum
         {
             return $this->communicationChannel;
         }
 
         public function setCommunicationChannel(?CommunicationChannelEnum $communicationChannel): void
         {
             $this->communicationChannel = $communicationChannel;
         }
         ```
    2. Исправляем метод `toArray`
         ```php
         public function toArray(): array
         {
             return [
                 'id' => $this->id,
                 'login' => $this->login,
                 'createdAt' => $this->createdAt->format('Y-m-d H:i:s'),
                 'updatedAt' => $this->updatedAt->format('Y-m-d H:i:s'),
                 'communicationChannel' => $this->communicationChannel->value,
                 'tweets' => array_map(static fn(Tweet $tweet) => $tweet->toArray(), $this->tweets->toArray()),
                 'followers' => array_map(
                     static fn(User $user) => ['id' => $user->getId(), 'login' => $user->getLogin()],
                     $this->followers->toArray()
                 ),
                 'authors' => array_map(
                     static fn(User $user) => ['id' => $user->getId(), 'login' => $user->getLogin()],
                     $this->authors->toArray()
                 ),
                 'subscriptionFollowers' => array_map(
                     static fn(Subscription $subscription) => [
                         'subscriptionId' => $subscription->getId(),
                         'userId' => $subscription->getFollower()->getId(),
                         'login' => $subscription->getFollower()->getLogin(),
                     ],
                     $this->subscriptionFollowers->toArray()
                 ),
                 'subscriptionAuthors' => array_map(
                     static fn(Subscription $subscription) => [
                         'subscriptionId' => $subscription->getId(),
                         'userId' => $subscription->getAuthor()->getId(),
                         'login' => $subscription->getAuthor()->getLogin(),
                     ],
                     $this->subscriptionAuthors->toArray()
                 ),
             ];
         }
         ```
3. В классе `App\Domain\Service\UserService` исправляем метод `create`
    ```php
    public function create(string $login, string $communicationChannel): User
    {
        $user = new User();
        $user->setLogin($login);
        $user->setCommunicationChannel(CommunicationChannelEnum::from($communicationChannel));
        $this->userRepository->create($user);

        return $user;
    }
    ```
4. Выполняем команду `php bin/console doctrine:migrations:diff`
5. Проверяем полученную миграцию и применяем её командой `php bin/console doctrine:migrations:migrate`
6. Заходим по адресу `http://localhost:7777/world/hello`, видим установленный канал связи

## Используем наследование сущностей с разными таблицами

1. Добавляем класс `App\Domain\Entity\EmailUser`
    ```php
    <?php
    
    namespace App\Domain\Entity;
    
    use Doctrine\ORM\Mapping as ORM;
    
    #[ORM\Table(name: '`email_user`')]
    #[ORM\Entity]
    class EmailUser extends User
    {
        #[ORM\Column(type: 'string', nullable: false)]
        private string $email;
    
        public function getEmail(): string
        {
            return $this->email;
        }
    
        public function setEmail(string $email): void
        {
            $this->email = $email;
        }
        
        public function toArray(): array
        {
            return parent::toArray() + ['email' => $this->email];
        }
    }
    ```
2. Добавляем класс `App\Domain\Entity\PhoneUser`
    ```php
    <?php
    
    namespace App\Domain\Entity;
    
    use Doctrine\ORM\Mapping as ORM;
    
    #[ORM\Table(name: 'phone_user')]
    #[ORM\Entity]
    class PhoneUser extends User
    {
        #[ORM\Column(type: 'string', nullable: false)]
        private string $phone;
    
        public function getPhone(): string
        {
            return $this->phone;
        }
    
        public function setPhone(string $phone): void
        {
            $this->phone = $phone;
        }
    
        public function toArray(): array
        {
            return parent::toArray() + ['phone' => $this->phone];
        }
    }
    ```
3. Исправляем класс `App\Domain\Entity\User`
    ```php
    <?php
    
    namespace App\Domain\Entity;
    
    use App\Domain\ValueObject\CommunicationChannelEnum;
    use DateInterval;
    use DateTime;
    use Doctrine\Common\Collections\ArrayCollection;
    use Doctrine\Common\Collections\Collection;
    use Doctrine\ORM\Mapping as ORM;
    
    #[ORM\Table(name: '`user`')]
    #[ORM\Entity]
    #[ORM\HasLifecycleCallbacks]
    #[ORM\InheritanceType('JOINED')]
    #[ORM\DiscriminatorColumn(name: 'communication_channel', type: 'string', enumType: CommunicationChannelEnum::class)]
    #[ORM\DiscriminatorMap(
        [
            CommunicationChannelEnum::Email->value => EmailUser::class,
            CommunicationChannelEnum::Phone->value => PhoneUser::class,
        ]
    )]
    class User implements EntityInterface, HasMetaTimestampsInterface, SoftDeletableInterface, SoftDeletableInFutureInterface
    {
        #[ORM\Column(name: 'id', type: 'bigint', unique: true)]
        #[ORM\Id]
        #[ORM\GeneratedValue(strategy: 'IDENTITY')]
        private ?int $id = null;
    
        #[ORM\Column(type: 'string', length: 32, nullable: false)]
        private string $login;
    
        #[ORM\Column(name: 'created_at', type: 'datetime', nullable: false)]
        private DateTime $createdAt;
    
        #[ORM\Column(name: 'updated_at', type: 'datetime', nullable: false)]
        private DateTime $updatedAt;
    
        #[ORM\OneToMany(targetEntity: Tweet::class, mappedBy: 'author')]
        private Collection $tweets;
    
        #[ORM\ManyToMany(targetEntity: 'User', mappedBy: 'followers')]
        private Collection $authors;
    
        #[ORM\ManyToMany(targetEntity: 'User', inversedBy: 'authors')]
        #[ORM\JoinTable(name: 'author_follower')]
        #[ORM\JoinColumn(name: 'author_id', referencedColumnName: 'id')]
        #[ORM\InverseJoinColumn(name: 'follower_id', referencedColumnName: 'id')]
        private Collection $followers;
    
        #[ORM\OneToMany(mappedBy: 'follower', targetEntity: 'Subscription')]
        private Collection $subscriptionAuthors;
    
        #[ORM\OneToMany(mappedBy: 'author', targetEntity: 'Subscription')]
        private Collection $subscriptionFollowers;
    
        #[ORM\Column(name: 'deleted_at', type: 'datetime', nullable: true)]
        private ?DateTime $deletedAt = null;
    
        public function __construct()
        {
            $this->tweets = new ArrayCollection();
            $this->authors = new ArrayCollection();
            $this->followers = new ArrayCollection();
            $this->subscriptionAuthors = new ArrayCollection();
            $this->subscriptionFollowers = new ArrayCollection();
        }
    
        public function getId(): int
        {
            return $this->id;
        }
    
        public function setId(int $id): void
        {
            $this->id = $id;
        }
    
        public function getLogin(): string
        {
            return $this->login;
        }
    
        public function setLogin(string $login): void
        {
            $this->login = $login;
        }
    
        public function getCreatedAt(): DateTime {
            return $this->createdAt;
        }
    
        #[ORM\PrePersist]
        public function setCreatedAt(): void {
            $this->createdAt = new DateTime();
        }
    
        public function getUpdatedAt(): DateTime {
            return $this->updatedAt;
        }
    
        #[ORM\PrePersist]
        #[ORM\PreUpdate]
        public function setUpdatedAt(): void {
            $this->updatedAt = new DateTime();
        }
    
        public function getDeletedAt(): ?DateTime
        {
            return $this->deletedAt;
        }
    
        public function setDeletedAt(): void
        {
            $this->deletedAt = new DateTime();
        }
    
        public function setDeletedAtInFuture(DateInterval $dateInterval): void
        {
            if ($this->deletedAt === null) {
                $this->deletedAt = new DateTime();
            }
            $this->deletedAt = $this->deletedAt->add($dateInterval);
        }
    
        public function addTweet(Tweet $tweet): void
        {
            if (!$this->tweets->contains($tweet)) {
                $this->tweets->add($tweet);
            }
        }
    
        public function addFollower(User $follower): void
        {
            if (!$this->followers->contains($follower)) {
                $this->followers->add($follower);
            }
        }
    
        public function addAuthor(User $author): void
        {
            if (!$this->authors->contains($author)) {
                $this->authors->add($author);
            }
        }
    
        public function addSubscriptionAuthor(Subscription $subscription): void
        {
            if (!$this->subscriptionAuthors->contains($subscription)) {
                $this->subscriptionAuthors->add($subscription);
            }
        }
    
        public function addSubscriptionFollower(Subscription $subscription): void
        {
            if (!$this->subscriptionFollowers->contains($subscription)) {
                $this->subscriptionFollowers->add($subscription);
            }
        }
    
        public function toArray(): array
        {
            return [
                'id' => $this->id,
                'login' => $this->login,
                'createdAt' => $this->createdAt->format('Y-m-d H:i:s'),
                'updatedAt' => $this->updatedAt->format('Y-m-d H:i:s'),
                'tweets' => array_map(static fn(Tweet $tweet) => $tweet->toArray(), $this->tweets->toArray()),
                'followers' => array_map(
                    static fn(User $user) => ['id' => $user->getId(), 'login' => $user->getLogin()],
                    $this->followers->toArray()
                ),
                'authors' => array_map(
                    static fn(User $user) => ['id' => $user->getId(), 'login' => $user->getLogin()],
                    $this->authors->toArray()
                ),
                'subscriptionFollowers' => array_map(
                    static fn(Subscription $subscription) => [
                        'subscriptionId' => $subscription->getId(),
                        'userId' => $subscription->getFollower()->getId(),
                        'login' => $subscription->getFollower()->getLogin(),
                    ],
                    $this->subscriptionFollowers->toArray()
                ),
                'subscriptionAuthors' => array_map(
                    static fn(Subscription $subscription) => [
                        'subscriptionId' => $subscription->getId(),
                        'userId' => $subscription->getAuthor()->getId(),
                        'login' => $subscription->getAuthor()->getLogin(),
                    ],
                    $this->subscriptionAuthors->toArray()
                ),
            ];
        }
    }
    ```
4. В классе `App\Domain\Service\UserService` удаляем метод `create` и добавляем методы `createWithPhone` и
   `createWithEmail`
    ```php
    public function createWithPhone(string $login, string $phone): User
    {
        $user = new PhoneUser();
        $user->setLogin($login);
        $user->setPhone($phone);
        $this->userRepository->create($user);

        return $user;
    }

    public function createWithEmail(string $login, string $email): User
    {
        $user = new EmailUser();
        $user->setLogin($login);
        $user->setEmail($email);
        $this->userRepository->create($user);

        return $user;
    }
    ```
5. В классе `App\Controller\WorldController` исправляем метод `hello`
    ```php
    public function hello(): Response
    {
        $this->userService->createWithPhone('Phone user', '+1234567890');
        $this->userService->createWithEmail('Email user', 'my@mail.ru');
        $users = $this->userService->findUsersByLoginWithQueryBuilder('user');

        return $this->json(
            ['users' => array_map(static fn (User $user) => $user->toArray(), $users)]
        );
    }
    ```
6. Выполняем команду `php bin/console doctrine:migrations:diff`
7. Очищаем таблицу `user` в БД, т.к. иначе миграция не сможет примениться
8. Проверяем полученную миграцию и применяем её командой `php bin/console doctrine:migrations:migrate`
9. Заходим по адресу `http://localhost:7777/world/hello`, видим двух пользователей с разными каналами связи

## Используем наследование сущностей в общей таблице

1. Исправляем в классе `App\Domain\Entity\User` значение атрибута `ORM\InheritanceType` на `SINGLE_TABLE`
2. Выполняем команду `php bin/console doctrine:migrations:diff`
3. Очищаем таблицу `user` в БД, т.к. иначе миграция не сможет примениться
4. Проверяем полученную миграцию и применяем её командой `php bin/console doctrine:migrations:migrate`
5. Заходим по адресу `http://localhost:7777/world/hello`, видим двух пользователей с разными каналами связи




# Контроллеры и маршрутизация

Запускаем контейнеры командой `docker-compose up -d`

## Добавляем CRUD-методы для работы с пользователем

1. Удаляем класс `App\Controller\WorldController`
2. Добавляем класс `App\Controller\Web\CreateUser\v1\Manager`
    ```php
    <?php
    
    namespace App\Controller\Web\CreateUser\v1;
    
    use App\Domain\Entity\User;
    use App\Domain\Service\UserService;
    
    class Manager
    {
        public function __construct(private readonly UserService $userService)
        {
        }
    
        public function create(string $login, ?string $phone = null, ?string $email = null): ?User
        {
            if ($phone !== null) {
                return $this->userService->createWithPhone($login, $phone);
            }
            
            if ($email !== null) {
                return $this->userService->createWithEmail($login, $email);
            }
            
            return null;
        }
    }
    ```
3. Добавляем класс `App\Controller\Web\CreateUser\v1\Controller`
    ```php
    <?php
    
    namespace App\Controller\Web\CreateUser\v1;
    
    use Symfony\Component\HttpFoundation\JsonResponse;
    use Symfony\Component\HttpFoundation\Request;
    use Symfony\Component\HttpFoundation\Response;
    use Symfony\Component\HttpKernel\Attribute\AsController;
    use Symfony\Component\Routing\Attribute\Route;
    
    #[AsController]
    class Controller
    {
        public function __construct(private readonly Manager $manager) {
        }
    
        #[Route(path: 'api/v1/user', methods: ['POST'])]
        public function __invoke(Request $request): Response
        {
            $login = $request->request->get('login');
            $phone = $request->request->get('phone');
            $email = $request->request->get('email');
            $user = $this->manager->create($login, $phone, $email);
            if ($user === null) {
                return new JsonResponse(null, Response::HTTP_BAD_REQUEST);
            }
    
            return new JsonResponse($user->toArray());
        }
    }    
    ```
4. В классе `App\Infrastructure\Repository\UserRepository` добавляем метод `findAll`
    ```php
    /**
     * @return User[]
     */
    public function findAll(): array
    {
        return $this->entityManager->getRepository(User::class)->findAll();
    }
    ```
5. В классе `App\Domain\Service\UserService` добавляем методы `findUserById` и `findAll`
    ```php
    public function findUserById(int $id): ?User
    {
        return $this->userRepository->find($id);
    }

    /**
     * @return User[]
     */
    public function findAll(): array
    {
        return $this->userRepository->findAll();
    }
    ```
6. Добавляем класс `App\Controller\Web\GetUser\v1\Manager`
    ```php
    <?php
    
    namespace App\Controller\Web\GetUser\v1;
    
    use App\Domain\Entity\User;
    use App\Domain\Service\UserService;
    
    class Manager
    {
        public function __construct(private readonly UserService $userService)
        {
        }
    
        public function getUserById(int $userId): ?User
        {
            return $this->userService->findUserById($userId);
        }

        /**
         * @return User[]
         */
        public function getAllUsers(): array
        {
            return $this->userService->findAll();
        }
    }    
    ```
7. Добавляем класс `App\Controller\Web\GetUser\v1\Controller`
    ```php
    <?php
    
    namespace App\Controller\Web\GetUser\v1;
    
    use App\Domain\Entity\User;
    use Symfony\Component\HttpFoundation\JsonResponse;
    use Symfony\Component\HttpFoundation\Request;
    use Symfony\Component\HttpFoundation\Response;
    use Symfony\Component\HttpKernel\Attribute\AsController;
    use Symfony\Component\Routing\Attribute\Route;
    
    #[AsController]
    class Controller
    {
        public function __construct(private readonly Manager $manager) {
        }
    
        #[Route(path: 'api/v1/user', methods: ['GET'])]
        public function __invoke(Request $request): Response
        {
            $userId = $request->query->get('id');
            if ($userId === null) {
                return new JsonResponse(array_map(static fn (User $user): array => $user->toArray(), $this->manager->getAllUsers()));
            }
            $user = $this->manager->getUserById($userId);
            if ($user instanceof User) {
                return new JsonResponse($user->toArray());
            }
    
            return new JsonResponse(null, Response::HTTP_NOT_FOUND);
        }
    }
    ```
8. Добавляем класс `App\Controller\Web\UpdateUserLogin\v1\Manager`
    ```php
    <?php
    
    namespace App\Controller\Web\UpdateUserLogin\v1;
    
    use App\Domain\Entity\User;
    use App\Domain\Service\UserService;
    
    class Manager
    {
        public function __construct(private readonly UserService $userService)
        {
        }
    
        public function updateUserLogin(int $userId, string $login): bool
        {
            $user = $this->userService->updateUserLogin($userId, $login);
            
            return $user instanceof User;
        }
    }
    ```
9. Добавляем класс `App\Controller\Web\UpdateUserLogin\v1\Controller`
    ```php
    <?php
    
    namespace App\Controller\Web\UpdateUserLogin\v1;
    
    use Symfony\Component\HttpFoundation\JsonResponse;
    use Symfony\Component\HttpFoundation\Request;
    use Symfony\Component\HttpFoundation\Response;
    use Symfony\Component\HttpKernel\Attribute\AsController;
    use Symfony\Component\Routing\Attribute\Route;
    
    #[AsController]
    class Controller
    {
        public function __construct(private readonly Manager $manager) {
        }
    
        #[Route(path: 'api/v1/user', methods: ['PATCH'])]
        public function __invoke(Request $request): Response
        {
            $userId = $request->request->get('id');
            $login = $request->request->get('login');
            $result = $this->manager->updateUserLogin($userId, $login);
    
            if ($result) {
                return new JsonResponse(['success' => true]);
            }
    
            return new JsonResponse(null, Response::HTTP_NOT_FOUND);
        }
    }
    ```
10. В классе `App\Domain\Service\UserService` исправляем метод `removeById`
    ```php
    public function removeById(int $userId): bool
    {
        $user = $this->userRepository->find($userId);
        if ($user instanceof User) {
            $this->userRepository->remove($user);
           
            return true;
        }
       
        return false;
    }
    ```
11. Добавляем класс `App\Controller\Web\DeleteUser\v1\Manager`
    ```php
    <?php
    
    namespace App\Controller\Web\DeleteUser\v1;
    
    use App\Domain\Service\UserService;
    
    class Manager
    {
        public function __construct(private readonly UserService $userService)
        {
        }
    
        public function deleteUserById(int $userId): bool
        {
            return $this->userService->removeById($userId);
        }
    }
    ```
12. Добавляем класс `App\Controller\Web\DeleteUser\v1\Controller`
    ```php
    <?php
    
    namespace App\Controller\Web\DeleteUser\v1;
    
    use Symfony\Component\HttpFoundation\JsonResponse;
    use Symfony\Component\HttpFoundation\Request;
    use Symfony\Component\HttpFoundation\Response;
    use Symfony\Component\HttpKernel\Attribute\AsController;
    use Symfony\Component\Routing\Attribute\Route;
    
    #[AsController]
    class Controller
    {
        public function __construct(private readonly Manager $manager) {
        }
    
        #[Route(path: 'api/v1/user', methods: ['DELETE'])]
        public function __invoke(Request $request): Response
        {
            $userId = $request->query->get('id');
            $result = $this->manager->deleteUserById($userId);
            if ($result) {
                return new JsonResponse(['success' => true]);
            }
    
            return new JsonResponse(null, Response::HTTP_NOT_FOUND);
        }
    }
    ```
13. Заходим в контейнер `php` командой `docker exec -it php sh`. Дальнейшие команды выполняются из контейнера
14. Выполняем команду `php bin/console debug:router`, видим список наших endpoint'ов из контроллера
15. Выполняем запрос Add user из Postman-коллекции, видим, что пользователь добавился
16. Выполняем запрос Delete user из Postman-коллекции с id из результата предыдущего запроса, видим, что пользователь
    удалился

## Добавляем инъекцию id в метод контроллера

1. В классе `App\Controller\Web\DeleteUser\v1\Controller` исправляем метод `__invoke`
    ```php
    #[Route(path: 'api/v1/user/{id}', requirements: ['id' => '\d+'], methods: ['DELETE'])]
    public function __invoke(int $id): Response
    {
        $result = $this->manager->deleteUserById($id);
        if ($result) {
            return new JsonResponse(['success' => true]);
        }

        return new JsonResponse(null, Response::HTTP_NOT_FOUND);
    }
    ```
2. Ещё раз выполняем запрос Add user из Postman-коллекции, чтобы создать пользователя
3. Выполняем запрос Delete user by id из Postman-коллекции с id из результата предыдущего запроса, видим, что
   пользователь удалился

## Исправляем запрос Patch user

1. Ещё раз выполняем запрос Add user из Postman-коллекции, чтобы создать пользователя
2. Пробуем отправить запрос Patch user из Postman-коллекции для созданного в предыдущем запросе пользователя, видим
   ошибку 500
3. Переносим в Postman-коллекции в запросе Patch user параметры из тела в строку запроса
4. Исправляем в классе `App\Controller\Web\UpdateUserLogin\v1\Controller` метод `__invoke`
    ```php
    #[Route(path: 'api/v1/user', methods: ['PATCH'])]
    public function __invoke(Request $request): Response
    {
        $userId = $request->query->get('id');
        $login = $request->query->get('login');
        $result = $this->manager->updateUserLogin($userId, $login);

        if ($result) {
            return new JsonResponse(['success' => true]);
        }

        return new JsonResponse(null, Response::HTTP_NOT_FOUND);
    }
    ```
5. Ещё раз пробуем отправить запрос Patch user из Postman-коллекции, логин обновляется

## Делаем инъекцию сущности в метод контроллера по id

1. Устанавливаем пакет `symfony/expression-language` командой `composer require symfony/expression-language`
   (понадобится для использования параметра `expr` атрибута `MapEntity`)
2. В классе `App\Domain\Service\UserService` добавляем метод `remove`
    ```php
    public function remove(User $user): void
    {
        $this->userRepository->remove($user);
    }
    ```
3. В классе `App\Controller\Web\DeleteUser\v1\Manager` добавляем метод `deleteUser`
    ```php
    public function deleteUser(User $user): void
    {
        $this->userService->remove($user);
    }
    ```
4. Исправляем класс `App\Controller\Web\DeleteUser\v1\Controller`
    ```php
    <?php
    
    namespace App\Controller\Web\DeleteUser\v1;
    
    use App\Domain\Entity\User;
    use Symfony\Bridge\Doctrine\Attribute\MapEntity;
    use Symfony\Component\HttpFoundation\JsonResponse;
    use Symfony\Component\HttpFoundation\Response;
    use Symfony\Component\HttpKernel\Attribute\AsController;
    use Symfony\Component\Routing\Attribute\Route;
    
    #[AsController]
    class Controller
    {
        public function __construct(private readonly Manager $manager) {
        }
    
        #[Route(path: 'api/v1/user/{id}', requirements: ['id' => '\d+'], methods: ['DELETE'])]
        public function __invoke(#[MapEntity(id: 'id')] User $user): Response
        {
            $this->manager->deleteUser($user);
    
            return new JsonResponse(['success' => true]);
        }
    }
    ```
5. Выполняем запрос Delete user из Postman-коллекции с несуществующим id, видим ошибку 404
6. Выполняем запрос Delete user из Postman-коллекции с id существующего пользователя, видим, что он удалился

## Делаем инъекцию сущности в метод контроллера по полю login

1. В классе `User` добавляем атрибут
    ```php
    #[ORM\UniqueConstraint(name: 'user__login__uniq', columns: ['login'], options: ['where' => '(deleted_at IS NULL)'])]
    ```
2. Выполняем команду `php bin/console doctrine:migrations:diff`
3. Проверяем сгенерированную миграцию и применяем её с помощью команды `php bin/console doctrine:migrations:migrate`
4. Добавляем класс `App\Controller\Web\GetUserByLogin\v1\Controller`
    ```php
    <?php
    
    namespace App\Controller\Web\GetUserByLogin\v1;
    
    use App\Domain\Entity\User;
    use Symfony\Bridge\Doctrine\Attribute\MapEntity;
    use Symfony\Component\HttpFoundation\JsonResponse;
    use Symfony\Component\HttpFoundation\Response;
    use Symfony\Component\HttpKernel\Attribute\AsController;
    use Symfony\Component\Routing\Attribute\Route;
    
    #[AsController]
    class Controller
    {
        #[Route(path: '/api/v1/get-user-by-login/{login}', methods: ['GET'])]
        public function getUserByLoginAction(#[MapEntity(mapping: ['login' => 'login'])] User $user): Response
        {
            return new JsonResponse(['user' => $user->toArray()], Response::HTTP_OK);
        }
    }
    ```
5. Выполняем запрос Get user by login из Postman-коллекции с несуществующим логином, видим ошибку 404
6. Выполняем запрос Get user by login из Postman-коллекции с существующим логином, видим успешный ответ

## Делаем инъекцию сущности в метод контроллера с помощью expression language

1. В классе `App\Domain\Service\UserService` добавляем метод `updateLogin`
    ```php
    public function updateLogin(User $user, string $login): void
    {
        $this->userRepository->updateLogin($user, $login);
    }
    ```
2. Исправляем класс `App\Controller\Web\UpdateUserLogin\v1\Manager`
    ```php
    <?php
    
    namespace App\Controller\Web\UpdateUserLogin\v1;
    
    use App\Domain\Entity\User;
    use App\Domain\Service\UserService;
    
    class Manager
    {
        public function __construct(private readonly UserService $userService)
        {
        }
    
        public function updateLogin(User $user, string $login): void
        {
            $this->userService->updateLogin($user, $login);
        }
    }    
    ```
3. Исправляем класс `App\Controller\Web\UpdateUserLogin\v1\Controller`
    ```php
    <?php
    
    namespace App\Controller\Web\UpdateUserLogin\v1;
    
    use App\Domain\Entity\User;
    use Symfony\Bridge\Doctrine\Attribute\MapEntity;
    use Symfony\Component\HttpFoundation\JsonResponse;
    use Symfony\Component\HttpFoundation\Request;
    use Symfony\Component\HttpFoundation\Response;
    use Symfony\Component\HttpKernel\Attribute\AsController;
    use Symfony\Component\Routing\Attribute\Route;
    
    #[AsController]
    class Controller
    {
        public function __construct(private readonly Manager $manager) {
        }
    
        #[Route(path: 'api/v1/user/{id}', methods: ['PATCH'])]
        public function __invoke(#[MapEntity(expr: 'repository.find(id)')] User $user, Request $request): Response
        {
            $login = $request->query->get('login');
            $this->manager->updateLogin($user, $login);
    
            return new JsonResponse(['success' => true]);
        }
    }
    ```
4. Выполняем запрос Patch user by id из Postman-коллекции с несуществующим шв, видим ошибку 404
5. Выполняем запрос Patch user by id из Postman-коллекции с существующим логином, видим успешный ответ



# Компонент HttpFoundation

Запускаем контейнеры командой `docker-compose up -d`

## Добавляем ссылку на аватар к пользователю

1. В классе `App\Domain\Entity\User`
    1. добавляем новое поле, геттер и сеттер
        ```php
        #[ORM\Column(type: 'string', nullable: true)]
        private ?string $avatarLink = null;
 
        public function getAvatarLink(): ?string
        {
            return $this->avatarLink;
        }
 
        public function setAvatarLink(?string $avatarLink): void
        {
            $this->avatarLink = $avatarLink;
        }
        ```
    2. исправляем метод `toArray`
        ```php
        public function toArray(): array
        {
            return [
                'id' => $this->id,
                'login' => $this->login,
                'avatar' => $this->avatarLink,
                'createdAt' => $this->createdAt->format('Y-m-d H:i:s'),
                'updatedAt' => $this->updatedAt->format('Y-m-d H:i:s'),
                'tweets' => array_map(static fn(Tweet $tweet) => $tweet->toArray(), $this->tweets->toArray()),
                'followers' => array_map(
                    static fn(User $user) => ['id' => $user->getId(), 'login' => $user->getLogin()],
                    $this->followers->toArray()
                ),
                'authors' => array_map(
                    static fn(User $user) => ['id' => $user->getId(), 'login' => $user->getLogin()],
                    $this->authors->toArray()
                ),
                'subscriptionFollowers' => array_map(
                    static fn(Subscription $subscription) => [
                        'subscriptionId' => $subscription->getId(),
                        'userId' => $subscription->getFollower()->getId(),
                        'login' => $subscription->getFollower()->getLogin(),
                    ],
                    $this->subscriptionFollowers->toArray()
                ),
                'subscriptionAuthors' => array_map(
                    static fn(Subscription $subscription) => [
                        'subscriptionId' => $subscription->getId(),
                        'userId' => $subscription->getAuthor()->getId(),
                        'login' => $subscription->getAuthor()->getLogin(),
                    ],
                    $this->subscriptionAuthors->toArray()
                ),
            ];
        }
        ```
2. Выполняем команду `php bin/console doctrine:migrations:diff`
3. Проверяем сгенерированную миграцию и применяем её с помощью команды `php bin/console doctrine:migrations:migrate`
4. Добавляем класс `App\Infrastructure\Storage\LocalFileStorage`
    ```php
    <?php
    
    namespace App\Infrastructure\Storage;
    
    use Symfony\Component\HttpFoundation\File\File;
    use Symfony\Component\HttpFoundation\File\UploadedFile;
    
    class LocalFileStorage
    {
        public function storeUploadedFile(UploadedFile $uploadedFile): File
        {
            $fileName = sprintf('%s.%s', uniqid('image', true), $uploadedFile->getClientOriginalExtension());
                
            return $uploadedFile->move('upload', $fileName);
        }
    }
    ```
5. Добавляем класс `App\Domain\Service\FileService`
    ```php
    <?php
    
    namespace App\Domain\Service;
    
    use App\Infrastructure\Storage\LocalFileStorage;
    use Symfony\Component\HttpFoundation\File\File;
    use Symfony\Component\HttpFoundation\File\UploadedFile;
    
    class FileService
    {
        public function __construct(private readonly LocalFileStorage $localFileStorage)
        {
        }
    
        public function storeUploadedFile(UploadedFile $uploadedFile): File
        {
            return $this->localFileStorage->storeUploadedFile($uploadedFile);
        }
    }
    ```
6. В классе `App\Infrastructure\Repository\UserRepository` добавляем метод `updateAvatarLink`
    ```php
    public function updateAvatarLink(User $user, string $avatarLink): void
    {
        $user->setAvatarLink($avatarLink);
        $this->flush();
    }
    ```
7. В классе `App\Domain\Service\UserService` добавляем метод `updateAvatarLink`
    ```php 
    public function updateAvatarLink(User $user, string $avatarLink): void
    {
        $this->userRepository->updateAvatarLink($user, $avatarLink);
    }
    ```
8. Добавляем класс `App\Controller\Web\UpdateUserAvatarLink\v1\Manager`
    ```php
    <?php
    
    namespace App\Controller\Web\UpdateUserAvatarLink\v1;
    
    use App\Domain\Entity\User;
    use App\Domain\Service\FileService;
    use App\Domain\Service\UserService;
    use Symfony\Component\HttpFoundation\File\UploadedFile;
    
    class Manager
    {
        public function __construct(
            private readonly FileService $fileService,
            private readonly UserService $userService,
        ) {
        }
    
        public function updateUserAvatarLink(User $user, UploadedFile $file): void
        {
            $this->fileService->storeUploadedFile($file);
            $this->userService->updateAvatarLink($user, $file->getRealPath());
        }
    }
    ```
9. Добавляем класс `App\Controller\Web\UpdateUserAvatarLink\v1\Controller`
    ```php
    <?php
    
    namespace App\Controller\Web\UpdateUserAvatarLink\v1;
    
    use App\Domain\Entity\User;
    use Symfony\Bridge\Doctrine\Attribute\MapEntity;
    use Symfony\Component\HttpFoundation\JsonResponse;
    use Symfony\Component\HttpFoundation\Request;
    use Symfony\Component\HttpFoundation\Response;
    use Symfony\Component\HttpKernel\Attribute\AsController;
    use Symfony\Component\Routing\Attribute\Route;
    
    #[AsController]
    class Controller
    {
        public function __construct(private readonly Manager $manager)
        {
        }
    
        #[Route(path: '/api/v1/update-user-avatar-link/{id}', methods: ['POST'])]
        public function getUserByLoginAction(#[MapEntity(id: 'id')] User $user, Request $request): Response
        {
            $this->manager->updateUserAvatarLink($user, $request->files->get('image'));
            
            return new JsonResponse(['user' => $user->toArray()], Response::HTTP_OK);
        }
    }
    ```
10. Выполняем запрос Update user avatar link из Postman-коллекции v2, видим исходный путь к файлу в каталоге tmp
11. Заходим в контейнер `php` командой `docker exec -it php sh`. Дальнейшие команды выполняются из контейнера
12. Проверяем, что файл появился в каталоге `public/upload` с новым именем и исходным расширением

## Исправим получение пути

1. В классе `App\Controller\Web\v1\UpdateUserAvatarLink\Manager` исправим метод `uploadUserAvatarLink`
    ```php
    public function updateUserAvatarLink(User $user, UploadedFile $uploadedFile): void
    {
        $file = $this->fileService->storeUploadedFile($uploadedFile);
        $this->userService->updateAvatarLink($user, $file->getRealPath());
    }
    ```
2. Ещё раз выполняем запрос Upload file из Postman-коллекции v2, видим корректный путь к файлу

## Сделаем ссылку открываемой

1. Исправляем класс `App\Controller\Web\UpdateUserAvatarLink\v1\Manager`
    ```php
    <?php
    
    namespace App\Controller\Web\UpdateUserAvatarLink\v1;
    
    use App\Domain\Entity\User;
    use App\Domain\Service\FileService;
    use App\Domain\Service\UserService;
    use Symfony\Component\HttpFoundation\File\UploadedFile;
    
    class Manager
    {
        public function __construct(
            private readonly FileService $fileService,
            private readonly UserService $userService,
            private readonly string $baseUrl,
            private readonly string $uploadPrefix,
        ) {
        }
    
        public function updateUserAvatarLink(User $user, UploadedFile $uploadedFile): void
        {
            $file = $this->fileService->storeUploadedFile($uploadedFile);
            $path = $this->baseUrl . str_replace($this->uploadPrefix, '', $file->getRealPath());
            $this->userService->updateAvatarLink($user, $path);
        }
    }
     ```
2. В файле `config/services.yaml`
    1. добавляем в секцию `parameters` новые параметры
        ```yaml
        parameters:
          baseUrl: 'http://localhost:7777'
          uploadPrefix: '/app/public'
        ```
    2. добавляем описание сервиса `App\Controller\Web\UpdateUserAvatarLink\v1\Manager`
        ```yaml
        App\Controller\Web\UpdateUserAvatarLink\v1\Manager:
            arguments:
                $baseUrl: '%baseUrl%'
                $uploadPrefix: '%uploadPrefix%'
        ```
3. Ещё раз выполняем запрос Upload file из Postman-коллекции v2, открываем ссылку, видим картинку

## Добавляем ExceptionListener

1. Добавляем интерфейс `App\Controller\Exception\HttpCompliantExceptionInterface`
    ```php
    <?php
    
    namespace App\Controller\Exception;
    
    interface HttpCompliantExceptionInterface
    {
        public function getHttpCode(): int;
        
        public function getHttpResponseBody(): string;
    }
    ```
2. Добавляем класс `App\Controller\Exception\DeprecatedException`
    ```php
    <?php
    
    namespace App\Controller\Exception;
    
    use Exception;
    use Symfony\Component\HttpFoundation\Response;
    
    class DeprecatedException extends Exception implements HttpCompliantExceptionInterface
    {
        public function getHttpCode(): int
        {
            return Response::HTTP_GONE;
        }
    
        public function getHttpResponseBody(): string
        {
            return 'This API method is deprecated';
        }
    }
    ```
3. В классе `App\Controller\Web\UpdateUserAvatarLink\v1\Manager` исправляем метод `updateUserAvatarLink`
    ```php
    public function updateUserAvatarLink(User $user, UploadedFile $uploadedFile): void
    {
        throw new DeprecatedException();
        $file = $this->fileService->storeUploadedFile($uploadedFile);
        $path = $this->baseUrl . str_replace($this->uploadPrefix, '', $file->getRealPath());
        $this->userService->updateAvatarLink($user, $path);
    }
    ```
4. Добавляем класс `App\Application\EventListener\KernelExceptionEventListener`
    ```php
    <?php
    
    namespace App\Application\EventListener;
    
    use App\Controller\Exception\HttpCompliantExceptionInterface;
    use Symfony\Component\HttpFoundation\JsonResponse;
    use Symfony\Component\HttpFoundation\Response;
    use Symfony\Component\HttpKernel\Event\ExceptionEvent;
    
    class KernelExceptionEventListener
    {
        public function onKernelException(ExceptionEvent $event): void
        {
            $exception = $event->getThrowable();
    
            if ($exception instanceof HttpCompliantExceptionInterface) {
                $event->setResponse($this->getHttpResponse($exception->getHttpResponseBody(), $exception->getHttpCode()));
            }
        }
    
        private function getHttpResponse($message, $code): Response {
            return new JsonResponse(['message' => $message], $code);
        }
    }
    ```
5. В файл `config/services.yaml` добавляем описание нового сервиса
    ```yaml
    App\Application\EventListener\KernelExceptionEventListener:
        tags:
            - { name: kernel.event_listener, event: kernel.exception }
6. Выполняем запрос Update user avatar link из Postman-коллекции v2, видим код ответа 410 и наше сообщение об ошибке

## Добавляем работу с EventDispatcher

1. Добавляем класс `App\Domain\Event\CreateUserEvent`
    ```php
    <?php
    
    namespace App\Domain\Event;
    
    class CreateUserEvent
    {
        public ?int $id = null;
   
        public function __construct(
            public readonly string $login,
            public readonly ?string $phone = null,
            public readonly ?string $email = null,
        ) {
        }
    }
    ```
2. Добавляем класс `App\Domain\EventSubscriber\UserEventSubscriber`
    ```php
    <?php
    
    namespace App\Domain\EventSubscriber;
    
    use App\Domain\Event\CreateUserEvent;
    use App\Domain\Service\UserService;
    use Symfony\Component\EventDispatcher\EventSubscriberInterface;
    
    class UserEventSubscriber implements EventSubscriberInterface
    {
        public function __construct(private readonly UserService $userService)
        {
        }
    
        public static function getSubscribedEvents(): array
        {
            return [
                CreateUserEvent::class => 'onCreateUser'
            ];
        }
    
        public function onCreateUser(CreateUserEvent $event): void
        {
            $user = null;
            
            if ($event->phone !== null) {
                $user = $this->userService->createWithPhone($event->login, $event->phone);
            } elseif ($event->email !== null) {
                $user = $this->userService->createWithEmail($event->login, $event->email);
            }
            
            $event->id = $user?->getId();
        }
    }
    ```
3. Исправляем класс `App\Controller\Web\CreateUser\v1\Manager`
    ```
    <?php
    
    namespace App\Controller\Web\CreateUser\v1;
    
    use App\Domain\Entity\User;
    use App\Domain\Event\CreateUserEvent;
    use App\Domain\Service\UserService;
    use Symfony\Component\EventDispatcher\EventDispatcherInterface;
    
    class Manager
    {
        public function __construct(
            private readonly UserService $userService,
            private readonly EventDispatcherInterface $eventDispatcher,
        ) {
        }
    
        public function create(string $login, ?string $phone = null, ?string $email = null): ?User
        {
            $event = new CreateUserEvent($login, $phone, $email);
            $this->eventDispatcher->dispatch($event);
    
            return $event->id === null ? null : $this->userService->findUserById($event->id);
        }
    } 
    ```
4. Выполняем запрос Add user из Postman-коллекции v2, видим успешный ответ




# Компонент HttpFoundation

Запускаем контейнеры командой `docker-compose up -d`

## Добавляем входящий DTO

1. Заходим в контейнер командой `docker exec -it php sh`, дальнейшие команды выполняются из контейнера.
2. Устанавливаем пакеты `symfony/serializer-pack` и `symfony/validator`
3. Добавляем класс `App\Controller\Web\CreateUser\v1\Input`
    ```php
    <?php
    
    namespace App\Controller\Web\CreateUser\v1\Input;
    
    use Symfony\Component\Validator\Constraints as Assert;
    
    class CreateUserDTO
    {
        public function __construct(
            #[Assert\NotBlank]
            public readonly string $login,
            public readonly ?string $email,
            public readonly ?string $phone,
        ) {    
        }
    }
    ```
4. В классе `App\Controller\Web\CreateUser\v1\Manager` исправляем метод `create`
    ```php
    public function create(CreateUserDTO $createUserDTO): ?User
    {
        $event = new CreateUserEvent($createUserDTO->login, $createUserDTO->phone, $createUserDTO->email);
        $event = $this->eventDispatcher->dispatch($event);

        return $event->id === null ? null : $this->userService->findUserById($event->id);
    }
    ```
5. В классе `App\Controller\Web\CreateUser\v1\Controller` исправляем метод `__invoke`
    ```php
    #[Route(path: 'api/v1/user', methods: ['POST'])]
    public function __invoke(#[MapRequestPayload] CreateUserDTO $createUserDTO): Response
    {
        $user = $this->manager->create($createUserDTO);
        if ($user === null) {
            return new JsonResponse(null, Response::HTTP_BAD_REQUEST);
        }

        return new JsonResponse($user->toArray());
    }
    ```
6. Выполняем запрос Add user из Postman-коллекции v2 с заполненными данными, видим успешный ответ
7. Выполняем запрос Add user из Postman-коллекции v2 без логина, видим ошибку 422

## Добавляем обработку ошибки

1. Исправляем класс `App\Application\EventListener\KernelExceptionEventListener`
    ```php
    <?php
    
    namespace App\Application\EventListener;
    
    use App\Controller\Exception\HttpCompliantExceptionInterface;
    use Symfony\Component\HttpFoundation\JsonResponse;
    use Symfony\Component\HttpFoundation\Response;
    use Symfony\Component\HttpKernel\Event\ExceptionEvent;
    use Symfony\Component\HttpKernel\Exception\HttpExceptionInterface;
    use Symfony\Component\Validator\ConstraintViolation;
    use Symfony\Component\Validator\Exception\ValidationFailedException;
    
    class KernelExceptionEventListener
    {
        public function onKernelException(ExceptionEvent $event): void
        {
            $exception = $event->getThrowable();
    
            if ($exception instanceof HttpCompliantExceptionInterface) {
                $event->setResponse($this->getHttpResponse($exception->getHttpResponseBody(), $exception->getHttpCode()));
            } elseif ($exception instanceof HttpExceptionInterface && $exception->getPrevious() instanceof ValidationFailedException) {
                $event->setResponse($this->getValidationFailedResponse($exception->getPrevious()));
            }
        }
    
        private function getHttpResponse($message, $code): Response {
            return new JsonResponse(['message' => $message], $code);
        }
        
        private function getValidationFailedResponse(ValidationFailedException $exception): Response {
            $response = [];
            foreach ($exception->getViolations() as $violation) {
                $response[$violation->getPropertyPath()] = $violation->getMessage();
            }
            return new JsonResponse($response, Response::HTTP_BAD_REQUEST);
        }
    }    
    ```
2. Выполняем запрос Add user из Postman-коллекции v2 без логина, видим JSON-ответ и ошибку 400

## Добавляем условную валидацию

1. К классу `App\Controller\Web\CreateUser\v1\Input\CreateUserDTO` добавляем атрибут
    ```php
    #[Assert\Expression(
        expression: '(this.email === null and this.phone !== null) or (this.phone === null and this.email !== null)',
        message: 'Either email or phone should be provided'
    )]
    ```
2. Выполняем запрос Add user из Postman-коллекции v2 только с телефоном, видим успешный ответ
3. Выполняем запрос Add user из Postman-коллекции v2 без е-мейла и телефона, видим JSON-ответ и ошибку 400, но не видим
   название свойства
4. Исправляем класс `App\Application\EventListener\KernelExceptionEventListener`
    ```php
    <?php
    
    namespace App\Application\EventListener;
    
    use App\Controller\Exception\HttpCompliantExceptionInterface;
    use Symfony\Component\HttpFoundation\JsonResponse;
    use Symfony\Component\HttpFoundation\Response;
    use Symfony\Component\HttpKernel\Event\ExceptionEvent;
    use Symfony\Component\HttpKernel\Exception\HttpExceptionInterface;
    use Symfony\Component\Validator\Exception\ValidationFailedException;
    
    class KernelExceptionEventListener
    {
        private const DEFAULT_PROPERTY = 'error';
        
        public function onKernelException(ExceptionEvent $event): void
        {
            $exception = $event->getThrowable();
    
            if ($exception instanceof HttpCompliantExceptionInterface) {
                $event->setResponse($this->getHttpResponse($exception->getHttpResponseBody(), $exception->getHttpCode()));
            } elseif ($exception instanceof HttpExceptionInterface && $exception->getPrevious() instanceof ValidationFailedException) {
                $event->setResponse($this->getValidationFailedResponse($exception->getPrevious()));
            }
        }
    
        private function getHttpResponse($message, $code): Response {
            return new JsonResponse(['message' => $message], $code);
        }
    
        private function getValidationFailedResponse(ValidationFailedException $exception): Response {
            $response = [];
            foreach ($exception->getViolations() as $violation) {
                $property = empty($violation->getPropertyPath()) ? self::DEFAULT_PROPERTY : $violation->getPropertyPath();
                $response[$property] = $violation->getMessage();
            }
            return new JsonResponse($response, Response::HTTP_BAD_REQUEST);
        }
    }
    ```
5. Выполняем запрос Add user из Postman-коллекции v2 с е-мейлом и телефоном, видим JSON-ответ и ошибку 400 с общим полем
   error

## Добавляем Model на слое домена

1. Добавляем класс `App\Domain\Model\CreateUserModel`
    ```php
    <?php
    
    namespace App\Domain\Model;
    
    use App\Domain\ValueObject\CommunicationChannelEnum;
    
    class CreateUserModel
    {
        public function __construct(
            public readonly string $login,
            public readonly string $communicationMethod,
            public readonly CommunicationChannelEnum $communicationChannel,
        ) {
        }
    }    
    ```
2. В классе `App\Domain\Entity\EmailUser` исправляем метод `setEmail`
    ```php
    public function setEmail(string $email): self
    {
        $this->email = $email;
        
        return $this;
    }
    ```
3. В классе `App\Domain\Entity\PhoneUser` исправляем метод `setPhone`
    ```php
    public function setPhone(string $phone): self
    {
        $this->phone = $phone;
        
        return $this;
    }
    ```
4. В классе `App\Domain\Service\UserService` добавляем метод `create`
    ```php
    public function create(CreateUserModel $createUserModel): User
    {
        $user = match($createUserModel->communicationChannel) {
            CommunicationChannelEnum::Email => (new EmailUser())->setEmail($createUserModel->communicationMethod),
            CommunicationChannelEnum::Phone => (new PhoneUser())->setPhone($createUserModel->communicationMethod),
        };
        $user->setLogin($createUserModel->login);
        $this->userRepository->create($user);

        return $user;
    }
    ```
5. Исправляем класс `App\Controller\Web\CreateUser\v1\Manager`
    ```php
    <?php
    
    namespace App\Controller\Web\CreateUser\v1;
    
    use App\Controller\Web\CreateUser\v1\Input\CreateUserDTO;
    use App\Domain\Entity\User;
    use App\Domain\Model\CreateUserModel;
    use App\Domain\Service\UserService;
    use App\Domain\ValueObject\CommunicationChannelEnum;
    
    class Manager
    {
        public function __construct(
            private readonly UserService $userService,
        ) {
        }
    
        public function create(CreateUserDTO $createUserDTO): User
        {
            $communicationMethod = $createUserDTO->phone ?? $createUserDTO->email;
            $communicationChannel = $createUserDTO->phone === null ? CommunicationChannelEnum::Email : CommunicationChannelEnum::Phone;
            $createUserModel = new CreateUserModel($createUserDTO->login, $communicationMethod, $communicationChannel);
    
            return $this->userService->create($createUserModel);
        }
    }    
    ```
6. В классе `App\Controller\Web\CreateUser\v1\Controller` исправляем метод `__invoke`
    ```php
    #[Route(path: 'api/v1/user', methods: ['POST'])]
    public function __invoke(#[MapRequestPayload] CreateUserDTO $createUserDTO): Response
    {
        $user = $this->manager->create($createUserDTO);

        return new JsonResponse($user->toArray());
    }
    ```
7. Выполняем запрос Add user из Postman-коллекции v2 с корректными данными, видим успешный ответ

## Добавляем ограничения в БД и валидацию на них

1. В классе `App\Domain\Entity\PhoneUser` исправляем поле `phone`
    ```php
    #[ORM\Column(type: 'string', length: 20, nullable: false)]
    private string $phone;
    ```
2. Выполняем команду `php bin/console doctrine:migrations:diff`
3. Проверяем сгенерированную миграцию и применяем её с помощью команды `php bin/console doctrine:migrations:migrate`
4. В классе `App\Controller\Web\CreateUser\v1\Input\CreateUserDTO` исправляем конструктор
    ```php
    public function __construct(
        #[Assert\NotBlank]
        public readonly string $login,
        public readonly ?string $email,
        #[Assert\Length(max: 20)]
        public readonly ?string $phone,
    ) {
    }
    ```
5. Выполняем запрос Add user из Postman-коллекции v2 с длинным телефоном, видим ошибку с кодом 400

## Добавляем валидацию и фабрику для моделей

1. Исправляем класс `App\Domain\Model\CreateUserModel`
    ```php
    <?php
    
    namespace App\Domain\Model;
    
    use App\Domain\ValueObject\CommunicationChannelEnum;
    
    use Symfony\Component\Validator\Constraints as Assert;
    
    class CreateUserModel
    {
        public function __construct(
            #[Assert\NotBlank]
            public readonly string $login,
            #[Assert\NotBlank]
            #[Assert\When(
                expression: "this.communicationChannel.value === 'phone'",
                constraints: [new Assert\Length(max: 20)]
            )]
            public readonly string $communicationMethod,
            public readonly CommunicationChannelEnum $communicationChannel,
        ) {
        }
    }
    ```
2. Добавляем класс `App\Domain\Service\ModelFactory`
    ```php
    <?php
    
    namespace App\Domain\Service;
    
    use Symfony\Component\Validator\Exception\ValidationFailedException;
    use Symfony\Component\Validator\Validator\ValidatorInterface;
    
    /**
     * @template T
     */    
    class ModelFactory
    {
        public function __construct(
            private readonly ValidatorInterface $validator
        ) {
        }
        
        /**
         * @param class-string $modelClass
         * @return T
         */
        public function makeModel(string $modelClass, ...$parameters)
        {
            $model = new $modelClass(...$parameters);
            $violations = $this->validator->validate($model);
            if ($violations->count() > 0) {
                throw new ValidationFailedException($parameters, $violations);
            }
            
            return $model;
        }
    }    
    ```
3. Исправляем класс `App\Controller\Web\CreateUser\v1\Manager`
    ```php
    <?php
    
    namespace App\Controller\Web\CreateUser\v1;
    
    use App\Controller\Web\CreateUser\v1\Input\CreateUserDTO;
    use App\Domain\Entity\User;
    use App\Domain\Model\CreateUserModel;
    use App\Domain\Service\ModelFactory;
    use App\Domain\Service\UserService;
    use App\Domain\ValueObject\CommunicationChannelEnum;
    
    class Manager
    {
        public function __construct(
            /** @var ModelFactory<CreateUserModel> */
            private readonly ModelFactory $modelFactory,
            private readonly UserService $userService,
        ) {
        }
    
        public function create(CreateUserDTO $createUserDTO): User
        {
            $communicationMethod = $createUserDTO->phone ?? $createUserDTO->email;
            $communicationChannel = $createUserDTO->phone === null ? CommunicationChannelEnum::Email : CommunicationChannelEnum::Phone;
            $createUserModel = $this->modelFactory->makeModel(CreateUserModel::class, $createUserDTO->login, $communicationMethod, $communicationChannel);
    
            return $this->userService->create($createUserModel);
        }
    }
    ```
4. В классе `App\Controller\Web\CreateUser\v1\Input\CreateUserDTO` убираем условие на поле `$phone`
5. Выполняем запрос Add user из Postman-коллекции v2 с длинным телефоном, видим ошибку с кодом 500

## Исправляем обработку исключений

1. В классе `\App\Application\EventListener\KernelExceptionEventListener` исправляем метод `onKernelException`
    ```php
    public function onKernelException(ExceptionEvent $event): void
    {
        $exception = $event->getThrowable();

        if ($exception instanceof HttpCompliantExceptionInterface) {
            $event->setResponse($this->getHttpResponse($exception->getHttpResponseBody(), $exception->getHttpCode()));
        } else {
            if ($exception instanceof HttpExceptionInterface) {
                $exception = $exception->getPrevious();
            }
            if ($exception instanceof ValidationFailedException) {
                $event->setResponse($this->getValidationFailedResponse($exception));
            }
        }
    }
    ```
2. Выполняем запрос Add user из Postman-коллекции v2 с длинным телефоном, видим ошибку на поле из модели с кодом 400
3. В классе `App\Controller\Web\CreateUser\v1\Input\CreateUserDTO` возвращаем обратно условие на поле `$phone`

## Добавляем исходящий DTO

1. Добавляем класс `App\Controller\Web\CreateUser\v1\Output\CreatedUserDTO`
   ```php
   <?php
   
   namespace App\Controller\Web\CreateUser\v1\Output;
   
   class CreatedUserDTO
   {
       public function __construct(
           public readonly int $id,
           public readonly string $login,
           public readonly ?string $avatarLink,
           public readonly string $createdAt,
           public readonly string $updatedAt,
           public readonly ?string $phone,
           public readonly ?string $email,
       ) {
       }
   }
   ```
2. Исправляем класс `App\Controller\Web\CreateUser\v1\Manager`
    ```php
    <?php
    
    namespace App\Controller\Web\CreateUser\v1;
    
    use App\Controller\Web\CreateUser\v1\Input\CreateUserDTO;
    use App\Controller\Web\CreateUser\v1\Output\CreatedUserDTO;
    use App\Domain\Entity\EmailUser;
    use App\Domain\Entity\PhoneUser;
    use App\Domain\Entity\User;
    use App\Domain\Model\CreateUserModel;
    use App\Domain\Service\ModelFactory;
    use App\Domain\Service\UserService;
    use App\Domain\ValueObject\CommunicationChannelEnum;
    
    class Manager
    {
        public function __construct(
            /** @var ModelFactory<CreateUserModel> */
            private readonly ModelFactory $modelFactory,
            private readonly UserService $userService,
        ) {
        }
    
        public function create(CreateUserDTO $createUserDTO): CreatedUserDTO
        {
            $communicationMethod = $createUserDTO->phone ?? $createUserDTO->email;
            $communicationChannel = $createUserDTO->phone === null ? CommunicationChannelEnum::Email : CommunicationChannelEnum::Phone;
            $createUserModel = $this->modelFactory->makeModel(CreateUserModel::class, $createUserDTO->login, $communicationMethod, $communicationChannel);
            $user = $this->userService->create($createUserModel);
    
            return new CreatedUserDTO(
                $user->getId(),
                $user->getLogin(),
                $user->getAvatarLink(),
                $user->getCreatedAt()->format('Y-m-d H:i:s'),
                $user->getUpdatedAt()->format('Y-m-d H:i:s'),
                $user instanceof PhoneUser ? $user->getPhone() : null,
                $user instanceof EmailUser ? $user->getEmail() : null,
            );
        }
    }
    ```
3. Исправляем класс `App\Controller\Web\CreateUser\v1\Controller`
    ```php
    <?php
    
    namespace App\Controller\Web\CreateUser\v1;
    
    use App\Controller\Web\CreateUser\v1\Input\CreateUserDTO;
    use Symfony\Component\HttpFoundation\JsonResponse;
    use Symfony\Component\HttpFoundation\Response;
    use Symfony\Component\HttpKernel\Attribute\AsController;
    use Symfony\Component\HttpKernel\Attribute\MapRequestPayload;
    use Symfony\Component\Routing\Attribute\Route;
    use Symfony\Component\Serializer\Encoder\JsonEncoder;
    use Symfony\Component\Serializer\SerializerInterface;
    
    #[AsController]
    class Controller
    {
        public function __construct(
            private readonly Manager $manager,
            private readonly SerializerInterface $serializer
        ) {
        }
    
        #[Route(path: 'api/v1/user', methods: ['POST'])]
        public function __invoke(#[MapRequestPayload] CreateUserDTO $createUserDTO): Response
        {
            $user = $this->manager->create($createUserDTO);
    
            return new JsonResponse($this->serializer->serialize($user, JsonEncoder::FORMAT), Response::HTTP_OK, [], true);
        }
    }    
    ```

## Выносим сериализацию из контроллера

1. Добавляем маркерный интерфейс `App\Controller\DTO\OutputDTOInterface`
    ```php
    <?php
    
    namespace App\Controller\DTO;
    
    interface OutputDTOInterface
    {
    }    
    ```
2. Имплементируем добавленный интерфейс в классе `App\Controller\Web\CreateUser\v1\Output\CreatedUserDTO`
3. Добавляем класс `App\Application\EventListener\KernelViewEventListener`
    ```php
    <?php
    
    namespace App\Application\EventListener;
    
    use App\Controller\DTO\OutputDTOInterface;
    use Symfony\Component\HttpFoundation\JsonResponse;
    use Symfony\Component\HttpFoundation\Response;
    use Symfony\Component\HttpKernel\Event\ViewEvent;
    use Symfony\Component\Serializer\Normalizer\AbstractObjectNormalizer;
    use Symfony\Component\Serializer\SerializerInterface;
    
    class KernelViewEventListener
    {
        public function __construct(private readonly SerializerInterface $serializer)
        {
        }
    
        public function onKernelView(ViewEvent $event): void
        {
            $dto = $event->getControllerResult();
    
            if ($dto instanceof OutputDTOInterface) {
                $event->setResponse($this->getDTOResponse($dto));
            }
        }
    
        private function getDTOResponse($data): Response {
            $serializedData = $this->serializer->serialize($data, 'json', [AbstractObjectNormalizer::SKIP_NULL_VALUES => true]);
    
            return new JsonResponse($serializedData, Response::HTTP_OK, [], true);
        }
    }    
    ```
4. В файле `config/services.yaml` добавляем описание нового сервиса
    ```yaml
    App\Application\EventListener\KernelViewEventListener:
        tags:
            - { name: kernel.event_listener, event: kernel.view }
    ```
5. Исправляем класс `App\Controller\Web\CreateUser\v1\Controller`
    ```php
    <?php
    
    namespace App\Controller\Web\CreateUser\v1;
    
    use App\Controller\Web\CreateUser\v1\Input\CreateUserDTO;
    use App\Controller\Web\CreateUser\v1\Output\CreatedUserDTO;
    use Symfony\Component\HttpKernel\Attribute\AsController;
    use Symfony\Component\HttpKernel\Attribute\MapRequestPayload;
    use Symfony\Component\Routing\Attribute\Route;
    
    #[AsController]
    class Controller
    {
        public function __construct(
            private readonly Manager $manager,
        ) {
        }
    
        #[Route(path: 'api/v1/user', methods: ['POST'])]
        public function __invoke(#[MapRequestPayload] CreateUserDTO $createUserDTO): CreatedUserDTO
        {
            return $this->manager->create($createUserDTO);
        }
    } 
    ```
6. Выполняем запрос Add user из Postman-коллекции v2 с корректными данными, видим успешный ответ




# Twig и Symfony Forms

Запускаем контейнеры командой `docker-compose up -d`

## Выводим список пользователей с помощью Twig

1. Заходим в контейнер `php` командой `docker exec -it php sh`. Дальнейшие команды выполняются из контейнера
2. Устанавливаем пакет `symfony/twig-bundle`
3. Создаём файл `templates/user-list.twig`
    ```html
    <!DOCTYPE html>
    <html>
    <head>
        <meta charset="UTF-8">
        <title>User list</title>
    </head>
    <body>
    <ul id="user.list">
        {% for user in users %}
            <li>{{ user.id }}. {{ user.login|lower }} {{ user.communicationChannel|upper }} ({{ user.communicationMethod }}) </li>
        {% endfor %}
    </ul>
    </body>
    </html>
    ```
4. Добавляем класс `App\Controller\Web\RenderUserList\v1\Manager`
    ```php
    <?php
    
    namespace App\Controller\Web\RenderUserList\v1;
    
    use App\Domain\Entity\EmailUser;
    use App\Domain\Entity\PhoneUser;
    use App\Domain\Entity\User;
    use App\Domain\Service\UserService;
    use App\Domain\ValueObject\CommunicationChannelEnum;
    
    class Manager
    {
        public function __construct(private readonly UserService $userService) {
        }
        
        public function getUserListData(): array
        {
            $mapper = static function (User $user): array {
                $result = [
                    'id' => $user->getId(),
                    'login' => $user->getLogin(),
                    'communicationChannel' => null,
                    'communicationMethod' => null,
                ];
                if ($user instanceof PhoneUser) {
                    $result['communicationChannel'] = CommunicationChannelEnum::Phone->value;
                    $result['communicationMethod'] = $user->getPhone();
                }
                if ($user instanceof EmailUser) {
                    $result['communicationChannel'] = CommunicationChannelEnum::Email->value;
                    $result['communicationMethod'] = $user->getEmail();
                }
                
                return $result;
            };
            
            return array_map($mapper, $this->userService->findAll());
        }
    }    
    ```
5. Добавляем класс `App\Controller\Web\RenderUserList\v1\Controller`
    ```php
    <?php
    
    namespace App\Controller\Web\RenderUserList\v1;
    
    use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
    use Symfony\Component\HttpFoundation\Response;
    use Symfony\Component\Routing\Attribute\Route;
    
    class Controller extends AbstractController
    {
        public function __construct(private readonly Manager $userManager)
        {
        }
    
        #[Route(path: '/api/v1/get-user-list', methods: ['GET'])]
        public function __invoke(): Response
        {
            return $this->render('user-list.twig', ['users' => $this->userManager->getUserListData()]);
        }
    } 
    ```
6. Заходим по адресу `http://localhost:7777/api/v1/get-user-list`, видим список пользователей

## Добавляем наследование в шаблоны

1. Переименовываем файл `templates/base.html.twig` в `layout.twig`
2. Исправляем файл `templates/user-list.twig`
    ```html
    {% extends 'layout.twig' %}

    {% block title %}
    User list
    {% endblock %}
    {% block body %}
    <ol id="user.list">
        {% for user in users %}
            <li>{{ user.id }}. {{ user.login|lower }} {{ user.communicationChannel|upper }} ({{ user.communicationMethod }}) </li>
        {% endfor %}
    </ol>
    {% endblock %}
    ```
3. Обновляем страницу в браузере, видим, что список стал нумерованным

## Добавляем повторяющиеся блоки

1. Исправляем файл `templates/layout.twig`
    ```html
    <!DOCTYPE html>
    <html>
        <head>
            <meta charset="UTF-8">
            <title>{% block title %}Welcome!{% endblock %}</title>
            {# Run `composer require symfony/webpack-encore-bundle`
               and uncomment the following Encore helpers to start using Symfony UX #}
            {% block stylesheets %}
                {#{{ encore_entry_link_tags('app') }}#}
            {% endblock %}
    
            {% block javascripts %}
                {#{{ encore_entry_script_tags('app') }}#}
            {% endblock %}
        </head>
        <body>
            {% block body %}{% endblock %}
            {% block footer %}{% endblock %}
        </body>
    </html>
    ```
2. Исправляем файл `templates/user-list.twig`
    ```html
    {% extends 'layout.twig' %}

    {% block title %}
    User list
    {% endblock %}
    {% block body %}
    <ol id="user.list">
         {% for user in users %}
            <li>{{ user.id }}. {{ user.login|lower }} {{ user.communicationChannel|upper }} ({{ user.communicationMethod }}) </li>
         {% endfor %}
    </ol>
    {% endblock %}
    {% block footer %}
    <h1>Footer</h1>
    {{ block('body') }}
    <h1>Repeat twice</h1>
    {{ block('body') }}
    {% endblock %}
    ```
3. Обновляем страницу в браузере, видим, что список выводится трижды с заголовками между списками

## Добавляем таблицу в шаблон

1. Создаём файл `templates/user-table.twig`
    ```html
    {% extends 'layout.twig' %}
    
    {% block title %}
        User table
    {% endblock %}
    {% block body %}
        <table>
            <tbody>
            <tr><th>ID</th><th>Логин</th><th>Канал коммуникации</th><th>Адрес</th></tr>
            {% for user in users %}
                <tr><td>{{ user.id }}</td><td>{{ user.login }}</td><td>{{ user.communicationChannel }}</td><td>({{ user.communicationMethod }})</td></tr>
            {% endfor %}
            </tbody>
        </table>
    {% endblock %}
    ```
2. Исправляем в классе `App\Controller\Web\RenderUserList\v1\Controller` метод `__invoke`
    ```php
    #[Route(path: '/api/v1/get-user-list', methods: ['GET'])]
    public function __invoke(): Response
    {
        return $this->render('user-table.twig', ['users' => $this->userManager->getUserListData()]);
    }
    ```
3. Обновляем страницу в браузере, видим, что вместо списка выведена таблица

## Добавляем макрос

1. Создаём файл `templates/macros.twig`
    ```html
    {% macro user_table_body(users) %}
         {% for user in users %}
             <tr><td>{{ user.id }}</td><td>{{ user.login }}</td><td>{{ user.communicationChannel }}</td><td>({{ user.communicationMethod }})</td></tr>
         {% endfor %}
    {% endmacro %}
    ```
2. Исправляем файл `templates/user-table.twig`:
    ```html
    {% extends 'layout.twig' %}

    {% import 'macros.twig' as macros %}

    {% block title %}
    User table
    {% endblock %}
    {% block body %}
    <table>
         <tbody>
            <tr><th>ID</th><th>Логин</th><th>Канал коммуникации</th><th>Адрес</th></tr>
            {{ macros.user_table_body(users) }}
         </tbody>
    </table>
    {% endblock %}
    ```
3. Обновляем страницу в браузере, видим, что таблица всё ещё отображается

## Добавляем собственные расширения

1. Добавляем класс `App\Application\Twig\MyTwigExtension`
    ```php
    <?php
    
    namespace App\Application\Twig;
    
    use Twig\Extension\AbstractExtension;
    use Twig\TwigFilter;
    use Twig\TwigFunction;
    
    class MyTwigExtension extends AbstractExtension
    {
        public function getFilters(): array
        {
            return [
                new TwigFilter('my_twice', static fn ($val) => $val.$val),
            ];
        }

        public function getFunctions(): array
        {
            return [
                new TwigFunction('my_greet', static fn ($val) => 'Hello, '.$val),
            ];
        }
    }
    ```
2. Исправляем файл `templates/macros.twig`
    ```html
    {% macro user_table_body(users) %}
        {% for user in users %}
            <tr><td>{{ user.id|my_twice }}</td><td>{{ user.login }}</td><td>{{ user.communicationChannel }}</td><td>({{ user.communicationMethod }})</td></tr>
        {% endfor %}
    {% endmacro %}
    ```
3. Исправляем файл `templates/user-table.twig`
    ```html
    {% extends 'layout.twig' %}
    
    {% import 'macros.twig' as macros %}
    
    {% block title %}
        {{ my_greet('User') }}
    {% endblock %}
    {% block body %}
        <table>
            <tbody>
            <tr><th>ID</th><th>Логин</th><th>Канал коммуникации</th><th>Адрес</th></tr>
            {{ macros.user_table_body(users) }}
            </tbody>
        </table>
    {% endblock %}
    ```
4. Обновляем страницу в браузере, видим работу функции и фильтра в расширении

## Устанавливаем Webpack Encore и подключаем bootstrap

1. Устанавливаем пакет `symfony/webpack-encore-bundle`
2. Устанавливаем `yarn` в контейнере (либо добавляем в образ, если планируем использовать дальше) командой
   `apk add yarn`
3. Устанавливаем зависимости командой `yarn install`
4. Устанавливаем загрузчик для работы с SASS командой `yarn add sass sass-loader@^14.0.0 --dev`
5. Устанавливаем bootstrap командой `yarn add @popperjs/core bootstrap --dev`
6. Переименовываем файл `assets/styles/app.css` в `app.scss` и исправляем его
    ```scss
    @import "~bootstrap/scss/bootstrap";
    
    body {
        background-color: lightgray;
    }
    ```
7. Исправляем файл `assets/app.js`
    ```js
    import './styles/app.scss';
    require('bootstrap');
    ```
8. Исправляем файл `webpack.config.js`, убираем комментарий в строке 57 (`//.enableSassLoader()`)
9. Исправляем файл `templates/user-table.twig`
    ```html
    {% extends 'layout.twig' %}
   
    {% import 'macros.twig' as macros %}
   
    {% block title %}
    {{ my_greet('User') }}
    {% endblock %}
    {% block body %}
    <table class="table table-hover">
        <tbody>
            <tr><th>ID</th><th>Логин</th><th>Канал коммуникации</th><th>Адрес</th></tr>
            {{ macros.user_table_body(users) }}
        </tbody>
    </table>
    {% endblock %}
    ```
10. В файле `templates/layout.twig` убираем комментарии с вызовов макросов для загрузки CSS и JS
     ```html
     <!DOCTYPE html>
     <html>
         <head>
             <meta charset="UTF-8">
             <title>{% block title %}Welcome!{% endblock %}</title>
             {# Run `composer require symfony/webpack-encore-bundle`
                and uncomment the following Encore helpers to start using Symfony UX #}
             {% block stylesheets %}
                 {{ encore_entry_link_tags('app') }}
             {% endblock %}
    
             {% block javascripts %}
                 {{ encore_entry_script_tags('app') }}
             {% endblock %}
         </head>
         <body>
             {% block body %}{% endblock %}
             {% block footer %}{% endblock %}
         </body>
     </html>   
     ```
11. Выполняем сборку для dev-окружения командой `yarn encore dev`
12. Видим собранные файлы в директории `public/build`
13. Выполняем сборку для prod-окружения командой `yarn encore production`
14. Видим собранные файлы в директории `public/build`, которые обфусцированы и содержат хэш в имени
15. Обновляем страницу в браузере, видим, что таблица отображается в boostrap-стиле

## Добавляем форму для создания и редактирования пользователя

1. Заходим в контейнер `php` командой `docker exec -it php sh`. Дальнейшие команды выполняются из контейнера
2. Устанавливаем пакет `symfony/form`
3. В классе `App\Domain\Entity\User` добавляем поля и геттеры/сеттеры для них
    ```php
    #[ORM\Column(type: 'string', nullable: false)]
    private string $password;

    #[ORM\Column(type: 'integer', nullable: false)]
    private int $age;

    #[ORM\Column(type: 'boolean', nullable: false)]
    private bool $isActive;

    public function getPassword(): string
    {
        return $this->password;
    }

    public function setPassword(string $password): void
    {
        $this->password = $password;
    }

    public function getAge(): int
    {
        return $this->age;
    }

    public function setAge(int $age): void
    {
        $this->age = $age;
    }

    public function isActive(): bool
    {
        return $this->isActive;
    }

    public function setIsActive(bool $isActive): void
    {
        $this->isActive = $isActive;
    }
    ```
4. Выполняем команду `php bin/console doctrine:migrations:diff`
5. Очищаем таблицу `user`, чтобы миграция смогла примениться
6. Проверяем сгенерированную миграцию и применяем её с помощью команды `php bin/console doctrine:migrations:migrate`
7. Исправляем класс `App\Domain\Model\CreateUserModel`
    ```php
    <?php
    
    namespace App\Domain\Model;
    
    use App\Domain\ValueObject\CommunicationChannelEnum;
    
    use Symfony\Component\Validator\Constraints as Assert;
    
    class CreateUserModel
    {
        public function __construct(
            #[Assert\NotBlank]
            public readonly string $login,
            #[Assert\NotBlank]
            #[Assert\When(
                expression: "this.communicationChannel.value === 'phone'",
                constraints: [new Assert\Length(max: 20)]
            )]
            public readonly string $communicationMethod,
            public readonly CommunicationChannelEnum $communicationChannel,
            public readonly string $password = 'myPass',
            public readonly int $age = 18,
            public readonly bool $isActive = true,
        ) {
        }
    }
    ```
8. В классе `App\Domain\Service\UserService` исправляем метод `create`
    ```php
    public function create(CreateUserModel $createUserModel): User
    {
        $user = match($createUserModel->communicationChannel) {
            CommunicationChannelEnum::Email => (new EmailUser())->setEmail($createUserModel->communicationMethod),
            CommunicationChannelEnum::Phone => (new PhoneUser())->setPhone($createUserModel->communicationMethod),
        };
        $user->setLogin($createUserModel->login);
        $user->setPassword($createUserModel->password);
        $user->setAge($createUserModel->age);
        $user->setIsActive($createUserModel->isActive);
        $this->userRepository->create($user);

        return $user;
    }
    ```
9. Добавляем класс `App\Controller\Form\PhoneUserType`
    ```php
    <?php
    
    namespace App\Controller\Form;
    
    use App\Domain\Entity\PhoneUser;
    use Symfony\Component\Form\AbstractType;
    use Symfony\Component\Form\Extension\Core\Type\CheckboxType;
    use Symfony\Component\Form\Extension\Core\Type\IntegerType;
    use Symfony\Component\Form\Extension\Core\Type\PasswordType;
    use Symfony\Component\Form\Extension\Core\Type\SubmitType;
    use Symfony\Component\Form\Extension\Core\Type\TextType;
    use Symfony\Component\Form\FormBuilderInterface;
    use Symfony\Component\OptionsResolver\OptionsResolver;
    
    class PhoneUserType extends AbstractType
    {
        public function buildForm(FormBuilderInterface $builder, array $options): void
        {
            $builder
                ->add('login', TextType::class, [
                    'label' => 'Логин пользователя',
                    'attr' => [
                        'data-time' => time(),
                        'placeholder' => 'Логин пользователя',
                        'class' => 'user-login',
                    ],
                ]);
    
            if ($options['isNew'] ?? false) {
                $builder->add('password', PasswordType::class, [
                    'label' => 'Пароль пользователя',
                ]);
            }
    
            $builder
                ->add('phone', TextType::class, [
                    'label' => 'Телефон',
                ])
                ->add('age', IntegerType::class, [
                    'label' => 'Возраст',
                ])
                ->add('isActive', CheckboxType::class, [
                    'required' => false,
                ])
                ->add('submit', SubmitType::class);
        }
    
        public function configureOptions(OptionsResolver $resolver): void
        {
            $resolver->setDefaults([
                'data_class' => PhoneUser::class,
                'empty_data' => new PhoneUser(),
                'isNew' => false,
            ]);
        }
    
        public function getBlockPrefix(): string
        {
            return 'save_user';
        }
    }
     ```
10. В классе `App\Domain\Service\UserService` добавляем метод `createFromForm`
     ```php
     public function processFromForm(User $user): void
     {
         $this->userRepository->create($user);
     }
     ```
11. Добавляем класс `App\Controller\web\PhoneUserForm\v1\Manager`
     ```php
     <?php
    
     namespace App\Controller\Web\PhoneUserForm\v1;
    
     use App\Controller\Form\PhoneUserType;
     use App\Domain\Entity\User;
     use App\Domain\Service\UserService;
     use Symfony\Component\Form\FormFactoryInterface;
     use Symfony\Component\HttpFoundation\Request;
    
     class Manager
     {
         public function __construct(
             private readonly UserService $userService,
             private readonly FormFactoryInterface $formFactory,
         ) {
         }
        
         public function getFormData(Request $request, ?User $user = null): array
         {
             $isNew = $user === null;
             $form = $this->formFactory->create(PhoneUserType::class, $user, ['isNew' => $isNew]);
             $form->handleRequest($request);
    
             if ($form->isSubmitted() && $form->isValid()) {
                 /** @var User $user */
                 $user = $form->getData();
                 $this->userService->processFromForm($user);
             }
    
             return [
                 'form' => $form,
                 'isNew' => $isNew,
                 'user' => $user,
             ];
         }
     }
     ```
12. Добавляем класс `App\Controller\Web\PhoneUserForm\v1\CreateController`
     ```php
     <?php
    
     namespace App\Controller\Web\PhoneUserForm\v1;
    
     use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
     use Symfony\Component\HttpFoundation\Request;
     use Symfony\Component\HttpFoundation\Response;
     use Symfony\Component\Routing\Attribute\Route;
    
     class CreateController extends AbstractController
     {
         public function __construct(private readonly Manager $manager)
         {
         }
    
         #[Route(path: '/api/v1/create-phone-user', methods: ['GET', 'POST'])]
         public function manageUserAction(Request $request): Response
         {
             return $this->render('phone-user.twig', $this->manager->getFormData($request));
         }
     }    
     ```
13. Добавляем класс `App\Controller\Web\PhoneUserForm\v1\EditController`
     ```php
     <?php
    
     namespace App\Controller\Web\PhoneUserForm\v1;
    
     use App\Domain\Entity\User;
     use Symfony\Bridge\Doctrine\Attribute\MapEntity;
     use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
     use Symfony\Component\HttpFoundation\Request;
     use Symfony\Component\HttpFoundation\Response;
     use Symfony\Component\Routing\Attribute\Route;
    
     class EditController extends AbstractController
     {
         public function __construct(private readonly Manager $manager)
         {
         }
    
         #[Route(path: '/api/v1/update-phone-user/{id}', methods: ['GET', 'POST'])]
         public function manageUserAction(Request $request, #[MapEntity(id: 'id')] User $user): Response
         {
             return $this->render('phone-user.twig', $this->manager->getFormData($request, $user));
         }
     }
     ```
14. Добавляем файл `src/templates/phone-user.twig`
     ```html
     {% extends 'layout.twig' %}
   
     {% block body %}
         <div>
             {{ form(form) }}
         </div>
     {% endblock %}
     ```
15. В браузере переходим по адресу `http://localhost:7777/api/v1/create-phone-user`, видим форму
16. Вводим данные, отправляем и проверяем базу. Данные должны быть сохранены
17. Берем в качестве ID последнего созданного пользователя, переходим по адресу `http://localhost:7777/api/v1/update-phone-user/ID`,
    проверяем работоспособность сохранения.

## Добавляем boostrap в форму

1. Исправляем файл `templates/phone-user.twig`
    ```html
    <!DOCTYPE html>
    <html>
    <head>
        <meta charset="UTF-8">
        {% block head_css %}
            <link rel="stylesheet" href="https://stackpath.bootstrapcdn.com/bootstrap/1.1.1/css/bootstrap.min.css" integrity="sha384-Vkoo8x4CGsO3+Hhxv8T/Q5PaXtkKtu6ug5TOeNV6gBiFeWPGFN9MuhOf23Q9Ifjh" crossorigin="anonymous">
        {% endblock %}
    </head>
    <body>
    {% form_theme form 'bootstrap_4_layout.html.twig' %}
    <div style="width:50%;margin-left:10px;margin-top:10px">
        {{ form(form) }}
    </div>
    {% block head_js %}
        <script src="https://code.jquery.com/jquery-3.4.1.slim.min.js" integrity="sha384-J6qa4849blE2+poT4WnyKhv5vZF5SrPo0iEjwBvKU7imGFAV0wwj1yYfoRSJoZ+n" crossorigin="anonymous"></script>
        <script src="https://cdn.jsdelivr.net/npm/popper.js@1.16.0/dist/umd/popper.min.js" integrity="sha384-Q6E9RHvbIyZFJoft+2mJbHaEWldlvI9IOYy5n3zV9zzTtmI3UksdQRVvoxMfooAo" crossorigin="anonymous"></script>
        <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/bootstrap@4.4.1/dist/css/bootstrap.min.css" integrity="sha384-Vkoo8x4CGsO3+Hhxv8T/Q5PaXtkKtu6ug5TOeNV6gBiFeWPGFN9MuhOf23Q9Ifjh" crossorigin="anonymous">
    {% endblock %}
    </body>
    </html>
    ```
2. Переходим по адресу `http://localhost:7777/api/v1/create-phone-user`, видим более красивый вариант формы

## Меняем HTTP-метод для редактирования

1. В классе `App\Controller\Web\PhoneUserForm\v1\EditController` исправляем атрибут для метода `__invoke`
    ```php
    #[Route(path: '/api/v1/update-phone-user/{id}', methods: ['GET', 'PATCH'])]
    ```
2. В классе `App\Controller\Form\PhoneUserType` исправляем метод `buildForm`
    ```php
    public function buildForm(FormBuilderInterface $builder, array $options): void
    {
        $builder
            ->add('login', TextType::class, [
                'label' => 'Логин пользователя',
                'attr' => [
                    'data-time' => time(),
                    'placeholder' => 'Логин пользователя',
                    'class' => 'user-login',
                ],
            ]);

        if ($options['isNew'] ?? false) {
            $builder->add('password', PasswordType::class, [
                'label' => 'Пароль пользователя',
            ]);
        }

        $builder
            ->add('phone', TextType::class, [
                'label' => 'Телефон',
            ])
            ->add('age', IntegerType::class, [
                'label' => 'Возраст',
            ])
            ->add('isActive', CheckboxType::class, [
                'required' => false,
            ])
            ->add('submit', SubmitType::class)
            ->setMethod($options['isNew'] ? 'POST' : 'PATCH');
    }
    ```
3. Исправляем файл `public/index.php`
    ```php
    <?php
   
    use App\Kernel;
    use Symfony\Component\HttpFoundation\Request;
       
    require_once dirname(__DIR__).'/vendor/autoload_runtime.php';
       
    return function (array $context) {
        Request::enableHttpMethodParameterOverride();

        return new Kernel($context['APP_ENV'], (bool) $context['APP_DEBUG']);
    };
    ```
4. Берем в качестве ID последнего созданного пользователя, переходим по адресу `http://localhost:7777/api/v1/update-phone-user/ID`,
   проверяем работоспособность сохранения.

## Добавляем DTO с валидацией

1. Добавляем класс `App\Controller\Web\UserForm\v1\Input\CreateUserDTO`
    ```php
    <?php
    
    namespace App\Controller\Web\UserForm\v1\Input;
    
    use Symfony\Component\Validator\Constraints as Assert;
    
    #[Assert\Expression(
        expression: '(this.email === null and this.phone !== null) or (this.phone === null and this.email !== null)',
        message: 'Eiteher email or phone should be provided',
    )]
    class CreateUserDTO
    {
        public function __construct(
            #[Assert\NotBlank]
            public ?string $login = null,
            public ?string $email = null,
            #[Assert\Length(max: 20)]
            public ?string $phone = null,
            #[Assert\NotBlank]
            public ?string $password = '',
            public ?int $age = 18,
            public ?bool $isActive = false,
        ) {
        }
    }
    ```
2. Добавляем класс `App\Controller\Form\UserType`
    ```php
    <?php
    
    namespace App\Controller\Form;
    
    use App\Controller\Web\UserForm\v1\Input\CreateUserDTO;
    use Symfony\Component\Form\AbstractType;
    use Symfony\Component\Form\Extension\Core\Type\CheckboxType;
    use Symfony\Component\Form\Extension\Core\Type\IntegerType;
    use Symfony\Component\Form\Extension\Core\Type\PasswordType;
    use Symfony\Component\Form\Extension\Core\Type\SubmitType;
    use Symfony\Component\Form\Extension\Core\Type\TextType;
    use Symfony\Component\Form\FormBuilderInterface;
    use Symfony\Component\OptionsResolver\OptionsResolver;
    
    class UserType extends AbstractType
    {
        public function buildForm(FormBuilderInterface $builder, array $options): void
        {
            $builder
                ->add('login', TextType::class, [
                    'label' => 'Логин пользователя',
                    'attr' => [
                        'data-time' => time(),
                        'placeholder' => 'Логин пользователя',
                        'class' => 'user-login',
                    ],
                ]);
    
            if ($options['isNew'] ?? false) {
                $builder->add('password', PasswordType::class, [
                    'label' => 'Пароль пользователя',
                ]);
            }
    
            $builder
                ->add('phone', TextType::class, [
                    'label' => 'Телефон',
                    'required' => false,
                ])
                ->add('email', TextType::class, [
                    'label' => 'E-mail',
                    'required' => false,
                ])
                ->add('age', IntegerType::class, [
                    'label' => 'Возраст',
                ])
                ->add('isActive', CheckboxType::class, [
                    'required' => false,
                ])
                ->add('submit', SubmitType::class)
                ->setMethod($options['isNew'] ? 'POST' : 'PATCH');
        }
    
        public function configureOptions(OptionsResolver $resolver): void
        {
            $resolver->setDefaults([
                'data_class' => CreateUserDTO::class,
                'empty_data' => new CreateUserDTO(),
                'isNew' => false,
            ]);
        }
    
        public function getBlockPrefix(): string
        {
            return 'save_user';
        }
    }
    ```
3. Добавляем класс `App\Controller\Web\UserForm\v1\Manager`
   ```php
   <?php
    
   namespace App\Controller\Web\UserForm\v1;
    
   use App\Controller\Form\UserType;
   use App\Controller\Web\UserForm\v1\Input\CreateUserDTO;
   use App\Domain\Entity\EmailUser;
   use App\Domain\Entity\PhoneUser;
   use App\Domain\Entity\User;
   use App\Domain\Model\CreateUserModel;
   use App\Domain\Service\ModelFactory;
   use App\Domain\Service\UserService;
   use App\Domain\ValueObject\CommunicationChannelEnum;
   use Symfony\Component\Form\FormFactoryInterface;
   use Symfony\Component\HttpFoundation\Request;
    
   class Manager
   {
       public function __construct(
           private readonly UserService $userService,
           private readonly FormFactoryInterface $formFactory,
           private readonly ModelFactory $modelFactory,
       ) {
       }
    
       public function getFormData(Request $request, ?User $user = null): array
       {
           $isNew = $user === null;
           $formData = $isNew ? null : new CreateUserDTO(
               $user->getLogin(),
               $user instanceof EmailUser ? $user->getEmail() : null,
               $user instanceof PhoneUser ? $user->getPhone() : null,
               $user->getPassword(),
               $user->getAge(),
               $user->isActive(),
           );
           $form = $this->formFactory->create(UserType::class, $formData, ['isNew' => $isNew]);
           $form->handleRequest($request);
    
           if ($form->isSubmitted() && $form->isValid()) {
               /** @var CreateUserDTO $createUserDTO */
               $createUserDTO = $form->getData();
               $communicationMethod = $createUserDTO->phone ?? $createUserDTO->email;
               $communicationChannel = $createUserDTO->phone === null ? CommunicationChannelEnum::Email : CommunicationChannelEnum::Phone;
               $createUserModel = $this->modelFactory->makeModel(
                   CreateUserModel::class,
                   $createUserDTO->login,
                   $communicationMethod,
                   $communicationChannel,
                   $createUserDTO->password,
                   $createUserDTO->age,
                   $createUserDTO->isActive,
               );
               $user = $this->userService->create($createUserModel);
           }
    
           return [
               'form' => $form,
               'isNew' => $isNew,
               'user' => $user,
           ];
       }
   }
   ```
4. Добавляем класс `App\Controller\Web\UserForm\v1\CreateController`
    ```php
    <?php
    
    namespace App\Controller\Web\UserForm\v1;
    
    use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
    use Symfony\Component\HttpFoundation\Request;
    use Symfony\Component\HttpFoundation\Response;
    use Symfony\Component\Routing\Attribute\Route;
    
    class CreateController extends AbstractController
    {
        public function __construct(private readonly Manager $manager)
        {
        }
    
        #[Route(path: '/api/v1/create-user', methods: ['GET', 'POST'])]
        public function manageUserAction(Request $request): Response
        {
            return $this->render('phone-user.twig', $this->manager->getFormData($request));
        }
    }
    ```
5. Добавляем класс `App\Controller\Web\UserForm\v1\EditController`
    ```php
    <?php
    
    namespace App\Controller\Web\UserForm\v1;
    
    use App\Domain\Entity\User;
    use Symfony\Bridge\Doctrine\Attribute\MapEntity;
    use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
    use Symfony\Component\HttpFoundation\Request;
    use Symfony\Component\HttpFoundation\Response;
    use Symfony\Component\Routing\Attribute\Route;
    
    class EditController extends AbstractController
    {
        public function __construct(private readonly Manager $manager)
        {
        }
    
        #[Route(path: '/api/v1/update-user/{id}', methods: ['GET', 'PATCH'])]
        public function manageUserAction(Request $request, #[MapEntity(id: 'id')] User $user): Response
        {
            return $this->render('phone-user.twig', $this->manager->getFormData($request, $user));
        }
    }
    ```
6. Переходим по адресу `http://localhost:7777/api/v1/create-user`, создаём пользователя
7. Берем в качестве ID последнего созданного пользователя, переходим по адресу `http://localhost:7777/api/v1/update-user/ID`,
   проверяем работоспособность сохранения.



# Авторизация и аутентификация

Запускаем контейнеры командой `docker-compose up -d`

## Добавляем пререквизиты

1. Заходим в контейнер `php` командой `docker exec -it php sh`. Дальнейшие команды выполняются из контейнера
2. Устанавливаем пакет `symfony/security-bundle`
3. Устанавливаем в dev-режиме пакет `symfony/maker-bundle`
4. В файле `config/packages/security.yaml`
    1. Исправляем секцию `providers`
        ```yaml
        providers:
            app_user_provider:
                entity:
                    class: App\Domain\Entity\User
                    property: login
        ```
    2. В секции `firewalls.main` заменяем `provider: users_in_memory` на `provider: app_user_provider`
5. Добавляем перечисление `App\Domain\ValueObject\RoleEnum`
    ```php
    <?php
    
    namespace App\Domain\ValueObject;
    
    enum RoleEnum: string
    {
        case ROLE_USER = 'ROLE_USER';
    } 
    ```
6. В классе `App\Domain\Entity\User`
    1. добавляем поле `$roles`, а также геттер и сеттер для него
        ```php
        #[ORM\Column(type: 'json', length: 1024, nullable: false)]
        private array $roles = [];
 
        /**
         * @return string[]
         */
        public function getRoles(): array
        {
            $roles = $this->roles;
            // guarantee every user at least has ROLE_USER
            $roles[] = RoleEnum::ROLE_USER->value;
 
            return array_unique($roles);
        }
 
        /**
         * @param string[] $roles
         */
        public function setRoles(array $roles): void
        {
            $this->roles = $roles;
        }
        ```
    2. имплементируем `Symfony\Component\Security\Core\User\UserInterface`
        ```php
        public function eraseCredentials(): void
        {
        }
     
        public function getUserIdentifier(): string
        {
            return $this->login;
        }
        ``` 
    3. имплементируем `Symfony\Component\Security\Core\User\PasswordAuthenticatedUserInterface` (нужный метод уже есть)
7. Генерируем миграцию командой `php bin/console doctrine:migrations:diff`
8. Проверяем сгенерированную миграцию и применяем её с помощью команды `php bin/console doctrine:migrations:migrate`
9. Исправляем класс `App\Domain\Model\CreateUserModel`
    ```php
    <?php
    
    namespace App\Domain\Model;
    
    use App\Domain\ValueObject\CommunicationChannelEnum;
    
    use Symfony\Component\Validator\Constraints as Assert;
    
    class CreateUserModel
    {
        /**
         * @param string[] $roles
         */
        public function __construct(
            #[Assert\NotBlank]
            public readonly string $login,
            #[Assert\NotBlank]
            #[Assert\When(
                expression: "this.communicationChannel.value === 'phone'",
                constraints: [new Assert\Length(max: 20)]
            )]
            public readonly string $communicationMethod,
            public readonly CommunicationChannelEnum $communicationChannel,
            public readonly string $password = 'myPass',
            public readonly int $age = 18,
            public readonly bool $isActive = true,
            public readonly array $roles = [],
        ) {
        }
    } 
    ```
10. В классе `App\Domain\Service\UserService`
    1. добавляем инъекцию `UserPasswordEncoderInterface`
        ```php
        public function __construct(
            private readonly UserRepository $userRepository,
            private readonly UserPasswordHasherInterface $userPasswordHasher,
        ) {
        }
        ```
    2. исправляем метод `create`
        ```php
        public function create(CreateUserModel $createUserModel): User
        {
            $user = match($createUserModel->communicationChannel) {
                CommunicationChannelEnum::Email => (new EmailUser())->setEmail($createUserModel->communicationMethod),
                CommunicationChannelEnum::Phone => (new PhoneUser())->setPhone($createUserModel->communicationMethod),
            };
            $user->setLogin($createUserModel->login);
            $user->setPassword($this->userPasswordHasher->hashPassword($user, $createUserModel->password));
            $user->setAge($createUserModel->age);
            $user->setIsActive($createUserModel->isActive);
            $user->setRoles($createUserModel->roles);
            $this->userRepository->create($user);
    
            return $user;
        }
        ```
11. Добавляяем класс `App\Controller\Web\CreateUser\v2\Input\CreateUserDTO`
    ```php
    <?php
    
    namespace App\Controller\Web\CreateUser\v2\Input;
    
    use Symfony\Component\Validator\Constraints as Assert;
    
    #[Assert\Expression(
        expression: '(this.email === null and this.phone !== null) or (this.phone === null and this.email !== null)',
        message: 'Eiteher email or phone should be provided',
    )]
    class CreateUserDTO
    {
        public function __construct(
            #[Assert\NotBlank]
            public readonly string $login,
            public readonly ?string $email,
            #[Assert\Length(max: 20)]
            public readonly ?string $phone,
            #[Assert\NotBlank]
            public readonly string $password,
            #[Assert\NotBlank]
            public readonly int $age,
            #[Assert\NotNull]
            public readonly bool $isActive,
            /** @var string[] $roles */
            public readonly array $roles,
        ) {
        }
    }
    ```
12. Добавляем класс `App\Controller\Web\CreateUser\v2\Output\CreatedUserDTO`
    ```php
    <?php
    
    namespace App\Controller\Web\CreateUser\v2\Output;
    
    use App\Controller\DTO\OutputDTOInterface;
    
    class CreatedUserDTO implements OutputDTOInterface
    {
        public function __construct(
            public readonly int $id,
            public readonly string $login,
            public readonly ?string $avatarLink,
            /** @var string[] $roles */
            public readonly array $roles,
            public readonly string $createdAt,
            public readonly string $updatedAt,
            public readonly ?string $phone,
            public readonly ?string $email,
        ) {
        }
    }
    ```
13. Добавляем класс `App\Controller\Web\CreateUser\v2\Manager`
    ```php
    <?php
    
    namespace App\Controller\Web\CreateUser\v2;
    
    use App\Controller\Web\CreateUser\v2\Input\CreateUserDTO;
    use App\Controller\Web\CreateUser\v2\Output\CreatedUserDTO;
    use App\Domain\Entity\EmailUser;
    use App\Domain\Entity\PhoneUser;
    use App\Domain\Model\CreateUserModel;
    use App\Domain\Service\ModelFactory;
    use App\Domain\Service\UserService;
    use App\Domain\ValueObject\CommunicationChannelEnum;
    
    class Manager
    {
        public function __construct(
            /** @var ModelFactory<CreateUserModel> */
            private readonly ModelFactory $modelFactory,
            private readonly UserService $userService,
        ) {
        }
    
        public function create(CreateUserDTO $createUserDTO): CreatedUserDTO
        {
            $communicationMethod = $createUserDTO->phone ?? $createUserDTO->email;
            $communicationChannel = $createUserDTO->phone === null ? CommunicationChannelEnum::Email : CommunicationChannelEnum::Phone;
            $createUserModel = $this->modelFactory->makeModel(
                CreateUserModel::class,
                $createUserDTO->login,
                $communicationMethod,
                $communicationChannel,
                $createUserDTO->password,
                $createUserDTO->age,
                $createUserDTO->isActive,
                $createUserDTO->roles
            );
            $user = $this->userService->create($createUserModel);
    
            return new CreatedUserDTO(
                $user->getId(),
                $user->getLogin(),
                $user->getAvatarLink(),
                $user->getRoles(),
                $user->getCreatedAt()->format('Y-m-d H:i:s'),
                $user->getUpdatedAt()->format('Y-m-d H:i:s'),
                $user instanceof PhoneUser ? $user->getPhone() : null,
                $user instanceof EmailUser ? $user->getEmail() : null,
            );
        }
    }
    ```
14. Добавляем класс `App\Controller\Web\CreateUser\v2\Controller`
    ```php
    <?php
    
    namespace App\Controller\Web\CreateUser\v2;
    
    use App\Controller\Web\CreateUser\v2\Input\CreateUserDTO;
    use App\Controller\Web\CreateUser\v2\Output\CreatedUserDTO;
    use Symfony\Component\HttpKernel\Attribute\AsController;
    use Symfony\Component\HttpKernel\Attribute\MapRequestPayload;
    use Symfony\Component\Routing\Attribute\Route;
    
    #[AsController]
    class Controller
    {
        public function __construct(
            private readonly Manager $manager,
        ) {
        }
    
        #[Route(path: 'api/v2/user', methods: ['POST'])]
        public function __invoke(#[MapRequestPayload] CreateUserDTO $createUserDTO): CreatedUserDTO
        {
            return $this->manager->create($createUserDTO);
        }
    }
    ```
15. Выполняем запрос Add user v2 из Postman-коллекции v3, видим, что пользователь добавлен в БД и пароль захэширован

## Добавляем форму логина

1. В файле `config/packages/security.yaml` в секции `firewall.main` добавляем `security:false`
2. Генерируем форму логина командой `php bin/console make:security:form-login`, не создавая Logout URL и тесты
3. Переименовываем класс `App\Controller\SecurityController` в `App\Controller\Web\Login\v1\Controller`
4. В файле `src/templates/security/login.html.twig` исправляем наследование на базовый шаблон `layout.twig`
5. Переходим по адресу `http://localhost:7777/login` и вводим логин/пароль пользователя, которого создали при проверке
   API. Видим, что после нажатия на `Sign in` ничего не происходит.

## Включаем security

1. Убираем в файле `config/packages/security.yaml` в секции `firewall.main` строку `security:false`
2. Ещё раз переходим по адресу `http://localhost:7777/login` и вводим логин/пароль пользователя, после нажатия на
   `Sign in` получаем перенаправление на корневую страницу с установленной сессией в куки

## Добавляем перенаправление

1. В файле `config/packages/security.yaml` исправляем секцию `security.firewalls.main.form_login:
    ```yaml
    form_login:
        login_path: app_login
        check_path: app_login
        enable_csrf: true
        default_target_path: app_web_renderuserlist_v1__invoke
    ```
2. Проверяем, что всё заработало

## Добавляем авторизацию для ROLE_ADMIN

1. В файле `config/packages/security.yaml` в секцию `access_control` добавляем условие
     ```yaml
     - { path: ^/api/v1/user, roles: ROLE_ADMIN, methods: [DELETE] }
     ```
2. Выполняем запрос Add user из Postman-коллекции v3 с другим значением логина, запоминаем id добавленного пользователя
3. Выполняем запрос Delete user by id из Postman-коллекции v3 с userId добавленного пользователя, добавив Cookie
   `PHPSESSID`, которую можно посмотреть в браузере после успешного логина. Проверяем, что возвращается ответ 403 с
   сообщением `Access denied`
4. Добавляем роль `ROLE_ADMIN` пользователю в БД, перелогиниваемся, чтобы получить корректную сессию и проверяем, что
   стал возвращаться ответ 200 и пользователь удалён из БД

## Добавляем авторизацию для ROLE_VIEW

1. В файле `config/packages/security.yaml` в секции `access_control` добавляем условие
     ```yaml
     - { path: ^/api/v1/get-user-list, roles: ROLE_VIEW, methods: [GET] }
     ```
2. Перезагружаем страницу в браузере, видим, что возвращается ответ 403 с сообщением `Access denied`

## Добавляем иерархию ролей

1. В классе `App\Controller\Web\RenderUserList\v1\Manager` исправляем метод `getUserListData`
    ```php
    public function getUserListData(): array
    {
        $mapper = static function (User $user): array {
            $result = [
                'id' => $user->getId(),
                'login' => $user->getLogin(),
                'communicationChannel' => null,
                'communicationMethod' => null,
                'roles' => $user->getRoles(),
            ];
            if ($user instanceof PhoneUser) {
                $result['communicationChannel'] = CommunicationChannelEnum::Phone->value;
                $result['communicationMethod'] = $user->getPhone();
            }
            if ($user instanceof EmailUser) {
                $result['communicationChannel'] = CommunicationChannelEnum::Email->value;
                $result['communicationMethod'] = $user->getEmail();
            }

            return $result;
        };

        return array_map($mapper, $this->userService->findAll());
    }
    ```
2. Исправляем шаблон `templates/user-table.twig`
    ```html
    {% extends 'layout.twig' %}
       
    {% import 'macros.twig' as macros %}
    
    {% block title %}
        {{ my_greet('User') }}
    {% endblock %}
    {% block body %}
        <table class="table table-hover">
            <tbody>
            <tr><th>ID</th><th>Логин</th><th>Роли</th><th>Канал коммуникации</th><th>Адрес</th></tr>
            {{ macros.user_table_body(users) }}
            </tbody>
        </table>
    {% endblock %}
    ```
3. В шаблоне `templates/macros.twig` исправляем макрос `user_table_body`
    ```html
    {% macro user_table_body(users) %}
        {% for user in users %}
            <tr>
                <td>{{ user.id|my_twice }}</td>
                <td>{{ user.login }}</td>
                <td>
                    <ul>
                    {% for role in user.roles %}
                        <li>{{ role }}</li>
                    {% endfor %}
                    </ul>    
                </td>
                <td>{{ user.communicationChannel }}</td>
                <td>({{ user.communicationMethod }})</td>
            </tr>
        {% endfor %}
    {% endmacro %}    
    ```
4. Добавляем в файл `config/packages/security.yaml` секцию `security.role_hierarchy`
     ```yaml
     role_hierarchy:
         ROLE_ADMIN: ROLE_VIEW
     ```
5. Ещё раз перезагружаем страницу. Видим список с ролями и видим, что у пользователя нет роли `ROLE_VIEW`.

## Добавляем авторизацию через атрибут

1. В классе `App\Controller\Web\GetUser\v1\Controller` добавляем атрибут на метод `__invoke`
    ```php
    #[IsGranted('ROLE_GET_LIST')]
    ```
2. Выполняем запрос Get user из Postman-коллекции v3, видим ошибку 403

## Добавляем авторизацию через код

1. В классе `App\Controller\Web\RenderUserList\v1\Controller` исправляем метод `__invoke`
    ```php
    #[Route(path: '/api/v1/get-user-list', methods: ['GET'])]
    public function __invoke(): Response
    {
        $this->denyAccessUnlessGranted('ROLE_GET_LIST');
        
        return $this->render('user-table.twig', ['users' => $this->userManager->getUserListData()]);
    }
    ```
2. Перезагружаем страницу, видим ошибку 403.
3. В файле `config/packages/security.yaml` исправляем секцию `security.role_hierarchy`
    ```yaml
    role_hierarchy:
        ROLE_ADMIN:
            - ROLE_VIEW
            - ROLE_GET_LIST
    ```
4. Перезагружаем страницу, видим таблицу с пользователями.
5. Выполняем запрос Get user из Postman-коллекции v3, видим успешный ответ.

## Добавляем Voter

1. Добавляем класс `App\Application\Security\Voter\UserVoter`
    ```php
    <?php
    
    namespace App\Application\Security\Voter;
    
    use App\Domain\Entity\User;
    use Symfony\Component\Security\Core\Authentication\Token\TokenInterface;
    use Symfony\Component\Security\Core\Authorization\Voter\Voter;
    
    class UserVoter extends Voter
    {
        public const DELETE = 'delete';
    
        protected function supports(string $attribute, $subject): bool
        {
            return $attribute === self::DELETE && ($subject instanceof User);
        }
    
        protected function voteOnAttribute(string $attribute, $subject, TokenInterface $token): bool
        {
            $user = $token->getUser();
            if (!$user instanceof User) {
                return false;
            }
    
            /** @var User $subject */
            return $user->getId() !== $subject->getId();
        }
    }
    ```
2. Добавляем класс `App\Controller\Exception\AccessDeniedException`
    ```php
    <?php
    
    namespace App\Controller\Exception;
    
    use Exception;
    use Symfony\Component\HttpFoundation\Response;
    
    class AccessDeniedException extends Exception implements HttpCompliantExceptionInterface
    {
        public function getHttpCode(): int
        {
            return Response::HTTP_FORBIDDEN;
        }
    
        public function getHttpResponseBody(): string
        {
            return 'Access denied';
        }
    }    
    ```
3. Исправляем класс `App\Controller\Web\DeleteUser\v1\Manager`
    ```php
    <?php
    
    namespace App\Controller\Web\DeleteUser\v1;
    
    use App\Application\Security\Voter\UserVoter;
    use App\Controller\Exception\AccessDeniedException;
    use App\Domain\Entity\User;
    use App\Domain\Service\UserService;
    use Symfony\Component\Security\Core\Authorization\AuthorizationCheckerInterface;
    
    class Manager
    {
        public function __construct(
            private readonly UserService $userService,
            private readonly AuthorizationCheckerInterface $authorizationChecker,
        ) {
        }
    
        /**
         * @throws AccessDeniedException
         */
        public function deleteUser(User $user): void
        {
            if (!$this->authorizationChecker->isGranted(UserVoter::DELETE, $user)) {
                throw new AccessDeniedException();
            }
            $this->userService->remove($user);
        }
    }
    ```
4. Выполняем запрос Add user из Postman-коллекции v3 с новым значением логина, запоминаем id добавленного пользователя
5. Выполняем запрос Delete user by id из Postman-коллекции v3 сначала с идентификатором добавленного пользователя, потом
   с идентификатором залогиненного пользователя. Проверяем, что в первом случае ответ 200, во втором - 403

## Изменяем стратегию для Voter'ов

1. Выполняем запрос Add user v3 из Postman-коллекции v3 с новым значением логина, запоминаем id добавленного
   пользователя
2. В файл `config/packages/security.yaml` добавляем секцию `security.access_decision_manager`
     ```yaml
     access_decision_manager:
         strategy: consensus
     ```
3. Добавляем класс `App\Application\Security\Voter\FakeUserVoter`
     ```php
     <?php
    
     namespace App\Application\Security\Voter;
    
     use App\Domain\Entity\User;
     use Symfony\Component\Security\Core\Authentication\Token\TokenInterface;
     use Symfony\Component\Security\Core\Authorization\Voter\Voter;
    
     class FakeUserVoter extends Voter
     {
         protected function supports(string $attribute, $subject): bool
         {
             return $attribute === UserVoter::DELETE && ($subject instanceof User);
         }
    
         protected function voteOnAttribute(string $attribute, $subject, TokenInterface $token): bool
         {
             return false;
         }
     }
     ```
4. Добавляем класс `App\Application\Security\Voter\DummyUserVoter`
     ```php
     <?php
    
     namespace App\Application\Security\Voter;
        
     use App\Domain\Entity\User;
     use Symfony\Component\Security\Core\Authentication\Token\TokenInterface;
     use Symfony\Component\Security\Core\Authorization\Voter\Voter;
    
     class DummyUserVoter extends Voter
     {
         protected function supports(string $attribute, $subject): bool
         {
             return $attribute === UserVoter::DELETE && ($subject instanceof User);
         }
    
         protected function voteOnAttribute(string $attribute, $subject, TokenInterface $token): bool
         {
             return false;
         }
     }
     ```
5. Выполняем запрос Add user из Postman-коллекции v3 с новым значением логина, запоминаем id добавленного пользователя
6. Выполняем запрос Delete user by id из Postman-коллекции v3 с идентификатором добавленного пользователя, видим ошибку
   403
7. Удаляем класс `App\Application\Security\Voter\DummyUserVoter`
8. Ещё раз выполняем запрос Delete user by id из Postman-коллекции v3 с идентификатором добавленного пользователя, видим
   успешный ответ



# Stateless API

Запускаем контейнеры командой `docker-compose up -d`

## Добавляем генерацию токена

1. Входим в контейнер командой `docker exec -it php sh`. Дальнейшие команды выполняются из контейнера
2. В файле `config/packages/security.yaml` добавляем в секцию `firewalls` после секции `dev`
    ```yaml
    token:
        pattern: ^/api/v1/get-token
        security: false
    ``` 
3. В класс `App\Domain\Entity\User` добавляем поле `$token` и стандартные геттер/сеттер для него
    ```php
    #[ORM\Column(type: 'string', length: 32, unique: true, nullable: true)]
    private ?string $token = null;

    public function getToken(): ?string
    {
        return $this->token;
    }

    public function setToken(?string $token): void
    {
        $this->token = $token;
    }
    ```
4. Генерируем миграцию командой `php bin/console doctrine:migrations:diff`
5. Проверяем сгенерированную миграцию и применяем её с помощью команды `php bin/console doctrine:migrations:migrate`
6. В классе `App\Infrastructure\Repository\UserRepository` добавляем новый метод `updateUserToken`
    ```php 
    public function updateUserToken(User $user): string
    {
        $token = base64_encode(random_bytes(20));
        $user->setToken($token);
        $this->flush();
        
        return $token;
    }
    ```
7. В классе `App\Domain\Service\UserService` добавляем новые методы `findUserByLogin` и `updateUserToken`
    ```php
    public function findUserByLogin(string $login): ?User
    {
        $users = $this->userRepository->findUsersByLogin($login);
        
        return $users[0] ?? null;
    }

    public function updateUserToken(string $login): ?string
    {
        $user = $this->findUserByLogin($login);
        if ($user === null) {
            return null;
        }
        
        return $this->userRepository->updateUserToken($user);
    }
    ```
8. Добавляем класс `App\Application\Security\AuthService`
    ```php
    <?php
    
    namespace App\Application\Security;
    
    use App\Domain\Service\UserService;
    use Symfony\Component\PasswordHasher\Hasher\UserPasswordHasherInterface;
    
    class AuthService
    {
        public function __construct(
            private readonly UserService $userService,
            private readonly UserPasswordHasherInterface $passwordHasher,
        ) {
        }
    
        public function isCredentialsValid(string $login, string $password): bool
        {
            $user = $this->userService->findUserByLogin($login);
            if ($user === null) {
                return false;
            }
    
            return $this->passwordHasher->isPasswordValid($user, $password);
        }
    
        public function getToken(string $login): ?string
        {
            return $this->userService->updateUserToken($login);
        }
    }
    ```
9. Добавляем класс `App\Controller\Exception\UnauthorizedException`
    ```php
    <?php
    
    namespace App\Controller\Exception;
    
    use Exception;
    use Symfony\Component\HttpFoundation\Response;
    
    class UnauthorizedException extends Exception implements HttpCompliantExceptionInterface
    {
        public function getHttpCode(): int
        {
            return Response::HTTP_UNAUTHORIZED;
        }
    
        public function getHttpResponseBody(): string
        {
            return 'Unauthorized';
        }
    }
    ```
10. Добавляем класс `App\Controller\Web\GetToken\v1\Manager`
    ```php
    <?php
    
    namespace App\Controller\Web\GetToken\v1;
    
    use App\Application\Security\AuthService;
    use App\Controller\Exception\AccessDeniedException;
    use App\Controller\Exception\UnauthorizedException;
    use Symfony\Component\HttpFoundation\Request;
    
    class Manager
    {
        public function __construct(private readonly AuthService $authService)
        {
        }
    
        /**
         * @throws AccessDeniedException
         * @throws UnauthorizedException
         */
        public function getToken(Request $request): string
        {
            $user = $request->getUser();
            $password = $request->getPassword();
            if (!$user || !$password) {
                throw new UnauthorizedException();
            }
            if (!$this->authService->isCredentialsValid($user, $password)) {
                throw new AccessDeniedException();
            }
    
            return $this->authService->getToken($user);
        }
    } 
    ```
11. Добавляем класс `App\Controller\Web\GetToken\v1\Controller`
    ```php
    <?php
    
    namespace App\Controller\Web\GetToken\v1;
    
    use Symfony\Component\HttpFoundation\JsonResponse;
    use Symfony\Component\HttpFoundation\Request;
    use Symfony\Component\HttpFoundation\Response;
    use Symfony\Component\HttpKernel\Attribute\AsController;
    use Symfony\Component\Routing\Attribute\Route;
    
    #[AsController]
    class Controller
    {
        public function __construct(private readonly Manager $manager) {
        }
    
        #[Route(path: 'api/v1/get-token', methods: ['POST'])]
        public function __invoke(Request $request): Response
        {
            return new JsonResponse(['token' => $this->manager->getToken($request)]);
        }
    } 
    ```
12. Выполняем запрос Add user v2 из Postman-коллекции v4, чтобы получить в БД пользователя
13. Выполняем запрос Get token из Postman-коллекции v4 без авторизации, получаем ошибку 401
14. Выполняем запрос Get token из Postman-коллекции v4 с неверными реквизитами, получаем ошибку 403
15. Выполняем запрос Get token из Postman-коллекции v4 с верными реквизитами, получаем токен. Проверяем, что в БД токен
    тоже сохранился.

## Добавляем аутентификатор с помощью токена

1. В классе `App\Infrastructure\Repository\UserRepository` добавляем метод `findUserByToken`
    ```php 
    public function findUserByToken(string $token): ?User
    {
        /** @var User|null $user */
        $user = $this->entityManager->getRepository(User::class)->findOneBy(['token' => $token]); 
        
        return $user;
    }
    ```
2. В классе `App\Domain\Service\UserService` добавляем метод `findUserByToken`
    ```php
    public function findUserByToken(string $token): ?User
    {
        return $this->userRepository->findUserByToken($token);
    }
    ```
3. Добавляем класс `App\Application\Security\ApiTokenAuthenticator`
    ```php
    <?php
    
    namespace App\Application\Security;
    
    use App\Controller\Exception\AccessDeniedException;
    use App\Controller\Exception\UnauthorizedException;
    use App\Domain\Service\UserService;
    use Symfony\Component\HttpFoundation\Request;
    use Symfony\Component\HttpFoundation\Response;
    use Symfony\Component\Security\Core\Authentication\Token\TokenInterface;
    use Symfony\Component\Security\Core\Exception\AuthenticationException;
    use Symfony\Component\Security\Http\Authenticator\AbstractAuthenticator;
    use Symfony\Component\Security\Http\Authenticator\Passport\Badge\UserBadge;
    use Symfony\Component\Security\Http\Authenticator\Passport\Passport;
    use Symfony\Component\Security\Http\Authenticator\Passport\SelfValidatingPassport;
    
    class ApiTokenAuthenticator extends AbstractAuthenticator
    {
        public function __construct(private readonly UserService $userService)
        {
        }
    
        public function supports(Request $request): ?bool
        {
            return true;
        }
    
        public function authenticate(Request $request): Passport
        {
            $authorization = $request->headers->get('Authorization');
            $token = str_starts_with($authorization, 'Bearer ') ? substr($authorization, 7) : null;
            if ($token === null) {
                throw new UnauthorizedException();
            }
    
            return new SelfValidatingPassport(
                new UserBadge($token, fn($token) => $this->userService->findUserByToken($token))
            );
        }
    
        public function onAuthenticationSuccess(Request $request, TokenInterface $token, string $firewallName): ?Response
        {
            return null;
        }
    
        public function onAuthenticationFailure(Request $request, AuthenticationException $exception): ?Response
        {
            throw new AccessDeniedException();
        }
    }
    ```
4. В файле `config/packages/security.yaml` меняем содержимое секции `firewalls.main`
    ```yaml
    main:
        lazy: true
        stateless: true
        custom_authenticator: App\Application\Security\ApiTokenAuthenticator
    ```
5. Выполняем запрос Get user из Postman-коллекции v4 без авторизации, получаем ошибку 401
6. Выполняем запрос Get token из Postman-коллекции v4, полученный токен заносим в Bearer-авторизацию запроса Get user и
   выполняем его, видим ошибку 403
7. Добавляем пользователю в БД роль `ROLE_ADMIN` и проверяем, что запрос Get user сразу же возвращает успешный ответ

## Добавляем JWT-аутентификатор

1. Устанавливаем пакет `lexik/jwt-authentication-bundle`
2. Генерируем ключи, используя passphrase из файла `.env` командами
    ```shell
    mkdir config/jwt
    openssl genrsa -out config/jwt/private.pem -aes256 4096
    openssl rsa -pubout -in config/jwt/private.pem -out config/jwt/public.pem
    chmod 777 config/jwt -R
    ```
3. В файл `.env` добавляем параметр
    ```shell
    JWT_TTL_SEC=3600
    ```
4. В файл `config/packages/lexik_jwt_authentication.yaml` добавляем строку
    ```yaml
    token_ttl: '%env(JWT_TTL_SEC)%'
    ```
5. В классе `App\Application\Security\AuthService`
    1. Добавляем зависимость от `Lexik\Bundle\JWTAuthenticationBundle\Encoder\JWTEncoderInterface` и целочисленный параметр `tokenTTL`
        ```php
        public function __construct(
            private readonly UserService $userService,
            private readonly UserPasswordHasherInterface $passwordHasher,
            private readonly JWTEncoderInterface $jwtEncoder,
            private readonly int $tokenTTL,
        ) {
        }
        ```         
    2. Исправляем метод `getToken`
        ```php
        /**
         * @throws JWTEncodeFailureException
         */
        public function getToken(string $login): string
        {
            $tokenData = ['username' => $login, 'exp' => time() + $this->tokenTTL];
   
            return $this->jwtEncoder->encode($tokenData);
        }
        ```
6. В файле `config/services.yaml` добавляем новый сервис
    ```yaml
    App\Application\Security\AuthService:
        arguments:
            $tokenTTL: '%env(JWT_TTL_SEC)%'
    ```
7. Добавляем класс `App\Application\Security\AuthUser`
    ```php
    <?php
    
    namespace App\Application\Security;
    
    use Symfony\Component\Security\Core\User\UserInterface;
    
    class AuthUser implements UserInterface
    {
        private string $username;
        
        /** @var string[] */
        private array $roles;
    
        public function __construct(array $credentials)
        {
            $this->username = $credentials['username'];
            $this->roles = array_unique(array_merge($credentials['roles'] ?? [], ['ROLE_USER']));
        }
    
        /**
         * @return string[]
         */
        public function getRoles(): array
        {
            return $this->roles;
        }
    
        public function getPassword(): string
        {
            return '';
        }
    
        public function getUserIdentifier(): string
        {
            return $this->username;
        }
    
        public function eraseCredentials(): void
        {
        }
    }
    ```
8. Добавляем класс `App\Application\Security\JwtAuthenticator`
    ```php
    <?php
    
    namespace App\Application\Security;
    
    use App\Controller\Exception\AccessDeniedException;
    use App\Controller\Exception\UnauthorizedException;
    use Lexik\Bundle\JWTAuthenticationBundle\Encoder\JWTEncoderInterface;
    use Lexik\Bundle\JWTAuthenticationBundle\TokenExtractor\AuthorizationHeaderTokenExtractor;
    use Symfony\Component\HttpFoundation\Request;
    use Symfony\Component\HttpFoundation\Response;
    use Symfony\Component\Security\Core\Authentication\Token\TokenInterface;
    use Symfony\Component\Security\Core\Exception\AuthenticationException;
    use Symfony\Component\Security\Http\Authenticator\AbstractAuthenticator;
    use Symfony\Component\Security\Http\Authenticator\Passport\Badge\UserBadge;
    use Symfony\Component\Security\Http\Authenticator\Passport\Passport;
    use Symfony\Component\Security\Http\Authenticator\Passport\SelfValidatingPassport;
    
    class JwtAuthenticator extends AbstractAuthenticator
    {
        public function __construct(private readonly JWTEncoderInterface $jwtEncoder)
        {
        }
    
        public function supports(Request $request): ?bool
        {
            return true;
        }
    
        public function authenticate(Request $request): Passport
        {
            $extractor = new AuthorizationHeaderTokenExtractor('Bearer', 'Authorization');
            $token = $extractor->extract($request);
            if ($token === null) {
                throw new UnauthorizedException();
            }
            $tokenData = $this->jwtEncoder->decode($token);
            if (!isset($tokenData['username'])) {
                throw new UnauthorizedException();
            }
    
            return new SelfValidatingPassport(
                new UserBadge($tokenData['username'], fn() => new AuthUser($tokenData))
            );
        }
    
        public function onAuthenticationSuccess(Request $request, TokenInterface $token, string $firewallName): ?Response
        {
            return null;
        }
    
        public function onAuthenticationFailure(Request $request, AuthenticationException $exception): ?Response
        {
            throw new AccessDeniedException();
        }
    }
    ```
9. В файле `config/packages/security.yaml` заменяем в секции `firewalls.main` значение поля `custom_authenticator` на
   `App\Application\Security\JwtAuthenticator`
10. Выполняем запрос Get user из Postman-коллекции v4 со старым токеном, получаем ошибку 500 с сообщением Invalid JWT
    Token
11. Выполняем запрос Get token из Postman-коллекции v4, полученный токен заносим в Bearer-авторизацию запроса Get user и
    выполняем его, получаем ошибку 403

## Исправляем получение JWT

1. В классе `App\Application\Security\AuthService` исправляем метод `getToken`
     ```php
     public function getToken(string $login): string
     {
         $user = $this->userService->findUserByLogin($login);
         $tokenData = [
             'username' => $login,
             'roles' => $user?->getRoles() ?? [],
             'exp' => time() + $this->tokenTTL,
         ];

         return $this->jwtEncoder->encode($tokenData);
     }
     ```
2. Перевыпускаем токен запросом Get token из Postman-коллекции v4, полученный токен заносим в Bearer-авторизацию
   запроса Get user. Выполняем запрос Get user, получаем результат
3. Удалям у пользователя в БД роль `ROLE_ADMIN`
4. Выполняем запрос Get user и видим результат, хоть роль и была удалена в БД
5. Ещё раз перевыпускаем токен запросом Get token из Postman-коллекции v4, полученный токен заносим в
   Bearer-авторизацию запроса Get user и выполняем его, получаем ошибку 403

## Добавляем реэмиссию JWT

1. В классе `App\Application\Security\AuthService` исправляем метод `getToken`
    ```php
    /**
     * @throws JWTEncodeFailureException
     */
    public function getToken(string $login): string
    {
        $user = $this->userService->findUserByLogin($login);
        $refreshToken = $this->userService->updateUserToken($login);
        $tokenData = [
            'username' => $login,
            'roles' => $user?->getRoles() ?? [],
            'exp' => time(),
            'refresh_token' => $refreshToken,
        ];

        return $this->jwtEncoder->encode($tokenData);
    }
    ```
2. В классе `App\Infrastructure\Repository\UserRepository` добавляем метод `clearUserToken`
    ```php
    public function clearUserToken(User $user): void
    {
        $user->setToken(null);
        $this->flush();
    }
    ```
3. В классе `App\Domain\Service\UserService` добавляем метод `clearUserToken`
    ```php
    public function clearUserToken(string $login): void
    {
        $user = $this->findUserByLogin($login);
        if ($user !== null) {
            $this->userRepository->clearUserToken($user);
        }
    }
    ```
4. Добавляем класс `App\Controller\Web\RefreshToken\v1\Manager`
    ```php
    <?php
    
    namespace App\Controller\Web\RefreshToken\v1;
    
    use App\Application\Security\AuthService;
    use App\Domain\Service\UserService;
    use Symfony\Component\Security\Core\User\UserInterface;
    
    class Manager
    {
        public function __construct(
            private readonly AuthService $authService,
            private readonly UserService $userService,
        ) {
        }
    
        public function refreshToken(UserInterface $user): string
        {
            $this->userService->clearUserToken($user->getUserIdentifier());
        
            return $this->authService->getToken($user->getUserIdentifier());
        }
    }
    ```
5. Добавляем класс `App\Controller\Web\RefreshToken\v1\Controller`
     ```php
     <?php
     
     namespace App\Controller\Web\RefreshToken\v1;
     
     use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
     use Symfony\Component\HttpFoundation\JsonResponse;
     use Symfony\Component\HttpFoundation\Request;
     use Symfony\Component\HttpFoundation\Response;
     use Symfony\Component\Routing\Attribute\Route;
     
     class Controller extends AbstractController
     {
         public function __construct(private readonly Manager $manager) {
         }
     
         #[Route(path: 'api/v1/refresh-token', methods: ['POST'])]
         public function __invoke(Request $request): Response
         {
             return new JsonResponse(['token' => $this->manager->refreshToken($this->getUser())]);
         }
     }
     ```
6. В файле `config/packages/security.yaml` добавляем в секцию `firewalls` после секции `dev`
    ```yaml
    refreshToken:
        pattern: ^/api/v1/refresh-token
        stateless: true
        custom_authenticator: App\Application\Security\ApiTokenAuthenticator
    ``` 
7. В классе `App\Controller\Exception\UnauthorizedException` исправляем метод `getHttpResponseBody`
    ```php
    public function getHttpResponseBody(): string
    {
        return empty($this->getMessage()) ? 'Unauthorized' : $this->getMessage();
    }
    ```
8. В классе `App\Application\Security\JwtAuthenticator` исправляем метод `authenticate`
    ```php
    public function authenticate(Request $request): Passport
    {
        $extractor = new AuthorizationHeaderTokenExtractor('Bearer', 'Authorization');
        $token = $extractor->extract($request);
        if ($token === null) {
            throw new UnauthorizedException();
        }
        try {
            $tokenData = $this->jwtEncoder->decode($token);
        } catch (JWTDecodeFailureException $exception) {
            $message = $exception->getReason() === JWTDecodeFailureException::EXPIRED_TOKEN ? 'Expired token' : ''; 
            throw new UnauthorizedException($message);
        }
        if (!isset($tokenData['username'])) {
            throw new UnauthorizedException();
        }

        return new SelfValidatingPassport(
            new UserBadge($tokenData['username'], fn() => new AuthUser($tokenData))
        );
    }
    ```
9. Перевыпускаем токен запросом Get token из Postman-коллекции v4, полученный токен заносим в Bearer-авторизацию
   запроса Get user. Выполняем запрос Get user, получаем ошибку с кодом 401 и текстом `Expired token`
10. Выполняем запрос Refresh token из Postman-коллекции v4 с токеном из поля `refresh_token` в JWT, получаем новый JWT
11. Ещё раз выполняем запрос Refresh token с тем же токеном, получаем ошибку 403




# REST-приложения и FOSRestBundle

Запускаем контейнеры командой `docker-compose up -d`

## Устанавливам API Platform

1. Заходим в контейнер `php` командой `docker exec -it php sh`. Дальнейшие команды выполняются из контейнера
2. Устанавливаем пакет `api-platform/core`
3. В файле `config/packages/api_platform.yaml`
    1. удаляем параметр `keep_legacy_inflector`
    2. добавляем секцию `api_plaftorm.mapping`
        ```yaml
        mapping:
            paths:
                - '%kernel.project_dir%/src/Domain/Entity'
        ```
4. В файле `config/routes/api_platform.yaml` меняем префикс API
    ```yaml
    api_platform:
        resource: .
        type: api_platform
        prefix: /api-platform
    ```
5. В файле `config/packages/security.yaml` в секцию `security.firewalls.main` добавляем `security: false`
6. Удаляем директорию `src/ApiResource`
7. К классу `App\Domain\Entity\PhoneUser` добавляем атрибут `#[ApiResource]`
8. Переходим в браузере по адресу `http://localhost:7777/api-platform`

## Проверяем работоспособность API

1. Выполняем запрос Add user v2 из Postman-коллекции v5, чтобы в БД появился пользователь
2. Выполняем запрос на получение коллекции PhoneUser в интерфейсе API-документации API Platform, видим полные данные
   нашего пользователя
3. Выполняем запрос на создание PhoneUser в интерфейсе API-документации API Platform с payload из запроса Add user v2
   из Postman-коллекции v5 с заменой логина на уникальный, видим успешный ответ и созданного в БД пользователя, но
   пароль не зашифрован

## Добавляем процессор для сохранения сущности

1. Добавляем класс `App\Domain\ApiPlatform\State\UserProcessor`
    ```php
    <?php
    
    namespace App\Domain\ApiPlatform\State;
    
    use ApiPlatform\Metadata\Operation;
    use ApiPlatform\State\ProcessorInterface;
    use App\Domain\Entity\User;
    use App\Infrastructure\Repository\UserRepository;
    use Symfony\Component\PasswordHasher\Hasher\UserPasswordHasherInterface;
    
    /**
     * @implements ProcessorInterface<User, User|void>
     */
    class UserProcessor implements ProcessorInterface
    {
        public function __construct(
            private readonly UserPasswordHasherInterface $userPasswordHasher,
            private readonly UserRepository $userRepository,
        ) {
        }
    
        /**
         * @param User $data
         * @return User|void
         */
        public function process(mixed $data, Operation $operation, array $uriVariables = [], array $context = [])
        {
            $data->setPassword($this->userPasswordHasher->hashPassword($data, $data->getPassword()));
            $this->userRepository->create($data);
    
            return $data;
        }
    }
    ``` 
2. К классу `App\Domain\Entity\PhoneUser` добавляем атрибут `#[Post(processor: UserProcessor::class)]`
3. Ещё раз выполняем запрос на создание PhoneUser в интерфейсе API-документации API Platform с payload из запроса Add
   user v2 из Postman-коллекции v5 с изменённым на уникальный логином, видим успешный ответ и зашифрованный пароль в БД.
4. Выполняем запрос Get token из Postman-коллекции v5 с реквизитами добавленного пользователя, видим успешный ответ.

## Переходим на DTO при сохранении

1. В классе `App\Domain\Entity\PhoneUser` исправляем атрибут
    ```php
    #[Post(input: CreateUserDTO::class, output: CreatedUserDTO::class, processor: UserProcessor::class)]
    ```
2. Исправляем класс `App\Domain\ApiPlatform\State\UserProcessor`
    ```php
    <?php
    
    namespace App\Domain\ApiPlatform\State;
    
    use ApiPlatform\Metadata\Operation;
    use ApiPlatform\State\ProcessorInterface;
    use App\Controller\Web\CreateUser\v2\Input\CreateUserDTO;
    use App\Controller\Web\CreateUser\v2\Manager;
    use App\Controller\Web\CreateUser\v2\Output\CreatedUserDTO;
    
    /**
     * @implements ProcessorInterface<CreateUserDTO, CreatedUserDTO|void>
     */
    class UserProcessor implements ProcessorInterface
    {
        public function __construct(private readonly Manager $manager)
        {
        }
    
        /**
         * @param CreateUserDTO $data
         * @return CreatedUserDTO|void
         */
        public function process(mixed $data, Operation $operation, array $uriVariables = [], array $context = [])
        {
            return $this->manager->create($data);
        }
    }
    ```
3. Ещё раз выполняем запрос на создание PhoneUser в интерфейсе API-документации API Platform с payload из запроса Add
   user v2 из Postman-коллекции v5 с изменённым на уникальный логином, видим успешный ответ и зашифрованный пароль в БД.

## Делаем общий ресурс для обоих типов пользователей

1. Переносим атрибуты `ApiResource` и `Post` из класса `App\Domain\Entity\PhoneUser` в `App\Domain\Entity\User`
2. Обновляем страницу в браузере.
3. Выполняем запросы на создание User в интерфейсе API-документации API Platform с email и телефоном, видим успешно
   созданные записи.

## Добавляем декоратор для провайдера для использования исходящего DTO

1. Добавляем класс `App\Domain\ApiPlatform\State\UserProviderDecorator`
    ```php
    <?php
    
    namespace App\Domain\ApiPlatform\State;
    
    use ApiPlatform\Metadata\Operation;
    use ApiPlatform\State\ProviderInterface;
    use App\Controller\Web\CreateUser\v2\Output\CreatedUserDTO;
    use App\Domain\Entity\EmailUser;
    use App\Domain\Entity\PhoneUser;
    use App\Domain\Entity\User;
    use Symfony\Component\DependencyInjection\Attribute\Autowire;
    
    /**
     * @implements ProviderInterface<CreatedUserDTO>
     */
    class UserProviderDecorator implements ProviderInterface
    {
        public function __construct(
            #[Autowire(service: 'api_platform.doctrine.orm.state.item_provider')]
            private readonly ProviderInterface $itemProvider,
        ) {
        }
    
        public function provide(Operation $operation, array $uriVariables = [], array $context = []): object|array|null
        {
            /** @var User $user */
            $user = $this->itemProvider->provide($operation, $uriVariables, $context);
            
            return new CreatedUserDTO(
                $user->getId(),
                $user->getLogin(),
                $user->getAvatarLink(),
                $user->getRoles(),
                $user->getCreatedAt()->format('Y-m-d H:i:s'),
                $user->getUpdatedAt()->format('Y-m-d H:i:s'),
                $user instanceof PhoneUser ? $user->getPhone() : null,
                $user instanceof EmailUser ? $user->getEmail() : null,
            );
        }
    }
    ```
2. К классу `App\Domain\Entity\User` добавляем атрибут
    ```php
    #[Get(output: CreatedUserDTO::class, provider: UserProviderDecorator::class)]
    ```
3. Обновляем страницу в браузере.
4. Выполняем запросы на получение User в интерфейсе API-документации API Platform с id пользователей с телефоном и
   email, видим успешные ответы

## Получаем JSON Schema

1. Добавляем класс `App\Controller\Web\GetJsonSchema\v1\Manager`
    ```php
    <?php
    
    namespace App\Controller\Web\GetJsonSchema\v1;
    
    use ApiPlatform\JsonSchema\SchemaFactoryInterface;
    
    class Manager
    {
        public function __construct(private readonly SchemaFactoryInterface $jsonSchemaFactory)
        {
        }
    
        public function getJsonSchemaAction(string $resource): array
        {
            $className = 'App\\Domain\\Entity\\'.ucfirst($resource);
            $schema = $this->jsonSchemaFactory->buildSchema($className);
    
            return json_decode(json_encode($schema), true);
        }
    }
    ```
2. Добавляем класс `App\Controller\Web\GetJsonSchema\v1\Controller`
    ```php
    <?php
    
    namespace App\Controller\Web\GetJsonSchema\v1;
    
    use Symfony\Component\HttpFoundation\JsonResponse;
    use Symfony\Component\HttpFoundation\Response;
    use Symfony\Component\HttpKernel\Attribute\AsController;
    use Symfony\Component\Routing\Attribute\Route;
    
    #[AsController]
    class Controller
    {
        public function __construct(private readonly Manager $manager) {
        }
    
        #[Route(path: 'api/v1/get-json-schema/{resource}', methods: ['GET'])]
        public function __invoke(string $resource): Response
        {
            return new JsonResponse($this->manager->getJsonSchemaAction($resource));
        }
    }
    ```
3. Выполняем запрос Get JSON Schema из Postman-коллекции v5

## Добавляем аутентификацию с помощью JWT

1. В файле `config/packages/security.yaml`
    1. удаляем `security: false` из `security.firewalls.main`
    2. добавляем секцию `security.firewalss.api-platform` перед `main`
        ```yaml
        apiPlatform:
            pattern: ^/api-platform$
            security: false      
        ```
2. В классе `App\Application\Security\AuthService` исправляем метод `getToken`
    ```php
    /**
     * @throws JWTEncodeFailureException
     */
    public function getToken(string $login): string
    {
        $user = $this->userService->findUserByLogin($login);
        $refreshToken = $this->userService->updateUserToken($login);
        $tokenData = [
            'username' => $login,
            'roles' => $user?->getRoles() ?? [],
            'exp' => time() + $this->tokenTTL,
            'refresh_token' => $refreshToken,
        ];

        return $this->jwtEncoder->encode($tokenData);
    }
    ```
3. В файле `config/api-platform.yaml` добавляем секцию `swagger`
    ```yaml
    swagger:
        api_keys:
            JWT:
                name: Authorization
                type: header
    ```
4. Выполняем запрос Get token из Postman-коллекции v5.
5. Перезагружаем страницу в браузере, нажимаем на кнопку Authorize и вводим `Bearer TOKEN`, где `TOKEN` – токен из
   предыдущего шага
6. Выполняем запрос Get JSON Schema из Postman-коллекции v11, видим ошибку 401
7. Выполняем запрос на получение User в интерфейсе API-документации API Platform с id существующего пользователя, видим
   успешный ответ





# Внедряем GraphQL

Запускаем контейнеры командой `docker-compose up -d`

## Устанавливаем GraphQL

1. Устанавливаем пакет `webonyx/graphql-php`
2. В файле `confgi/packages/security.yaml` в секции `security.firewalls.main` добавляем `security: false`
3. Добавляем несколько пользователей с телефонами с помощью запроса Add user v2 из Postman-коллекции v6
4. Заходим по адресу http://localhost:7777/api-platform/graphql, видим GraphQL-песочницу
5. Делаем в ней запрос
    ```
    {
      users {
        edges {
          node {
            id
            _id
            login
          }
        }
      }
    } 
    ```

## Получим связанные сущности

1. В классе `App\Domain\Entity\Subscription` добавляем атрибут `#[ApiResource]`
2. Добавляем класс `App\Controller\Web\CreateSubscription\v1\Manager`
    ```php
    <?php
    
    namespace App\Controller\Web\CreateSubscription\v1;
    
    use App\Domain\Entity\User;
    use App\Domain\Service\SubscriptionService;
    
    class Manager
    {
        public function __construct(private readonly SubscriptionService $subscriptionService)
        {
        }
        
        public function create(User $author, User $follower): void
        {
            $this->subscriptionService->addSubscription($author, $follower);
        }
    }
    ```
3. Добавляем класс `App\Controller\Web\CreateSubscription\v1\Controller`
    ```php
    <?php
    
    namespace App\Controller\Web\CreateSubscription\v1;
    
    use App\Domain\Entity\User;
    use Symfony\Bridge\Doctrine\Attribute\MapEntity;
    use Symfony\Component\HttpFoundation\JsonResponse;
    use Symfony\Component\HttpFoundation\Response;
    use Symfony\Component\HttpKernel\Attribute\AsController;
    use Symfony\Component\Routing\Attribute\Route;
    
    #[AsController]
    class Controller
    {
        public function __construct(
            private readonly Manager $manager,
        ) {
        }
    
        #[Route(
            path: 'api/v1/create-subscription/{author}/{follower}',
            requirements: ['author' => '\d+', 'follower' => '\d+'],
            methods: ['POST'])
        ]
        public function __invoke(#[MapEntity(id: 'author')] User $author, #[MapEntity(id: 'follower')] User $follower): Response
        {
            $this->manager->create($author, $follower);
            
            return new JsonResponse(null, Response::HTTP_CREATED);
        }
    }
    ```
4. Выполняем запрос Add subscription из Postman-коллекции v6 с идентификаторами добавленных ранее пользователей, чтобы
   сделать одному из них несколько подписок.
5. В классе `App\Domain\Entity\User`
   1. добавляем метод `getSubscriptionFollowers`
       ```php
       /**
        * @return Subscription[]
        */
       public function getSubscriptionFollowers(): array
       {
           return $this->subscriptionFollowers->toArray();
       }
       ```
   2. удаляем атрибут класса `#[Get(output: CreatedUserDTO::class, provider: UserProviderDecorator::class)]`
6. В классе `App\Domain\Entity\PhoneUser` добавляем атрибут `#[ApiResourse]`
7. В браузере перезапускаем страницу с GraphQL-песочницей и делаем запрос для получения пользователей и id их подписчиков
    ```
    {
      phoneUsers {
        edges {
          node {
            id
            _id
            login
            phone
            subscriptionFollowers {
              edges {
                node {
                  id
                }
              }
            }
          }
        }
      }
    }
    ```
8. Добавляем ограничение и метаинформацию
    ```
    {
      phoneUsers {
        edges {
          node {
            id
            _id
            login
            phone
            subscriptionFollowers(first: 3) {
              totalCount
              edges {
                node {
                  id
                }
                cursor
              }
              pageInfo {
                endCursor
                hasNextPage
              }
            }
          }
        }
      }
    }
    ```
9. Для получения следующей страницы добавляем параметр `after`
    ```
    {
      phoneUsers {
        edges {
          node {
            id
            _id
            login
            phone
            subscriptionFollowers(first: 3 after: "Mg==") {
              totalCount
              edges {
                node {
                  id
                }
                cursor
              }
              pageInfo {
                endCursor
                hasNextPage
              }
            }
          }
        }
      }
    }
    ```
10. Раскрываем дополнительно информацию о подписке ещё на один уровень
     ```
     {
       phoneUsers {
         edges {
           node {
             id
             _id
             login
             phone
             subscriptionFollowers(first: 3 after: "Mg==") {
               edges {
                 node {
                   follower {
                     id
                     _id
                     login
                   }
                 }
               }
               pageInfo {
                 endCursor
                 hasNextPage
               }
             }
           }
         }
       }
     }
     ```
11. Получаем данные о пользователе через фильтр по id
     ```
     {
       phoneUser(id: "/api-platform/phone_users/1") {
         id
         _id
         login
         phone
         subscriptionFollowers(first: 3, after: "Mg==") {
           edges {
             node {
               follower {
                 id
                 _id
                 login
               }
             }
           }
           pageInfo {
             endCursor
             hasNextPage
           }
         }
       }
     }
     ```

## Добавляем фильтрацию

1. К классу `App\Domain\Entity\PhoneUser` добавляем атрибут для фильтрации
    ```php
    #[ApiFilter(SearchFilter::class, properties: ['login' => 'partial'])]
    ```
2. В браузере перезапускаем страницу с GraphQL-песочницей и получаем данные о пользователях с фильтром
    ```
    {
      phoneUsers(login: "user3") {
        edges {
          node {
            id
            _id
            login
            phone
          }
        }
      }
    }
    ```

## Добавляем сортировку

1. К классу `App\Domain\Entity\PhoneUser` добавляем атрибут для сортировки
    ```php
    #[ApiFilter(OrderFilter::class, properties: ['login'])]
    ```
2. В браузере перезапускаем страницу с GraphQL-песочницей и получаем данные о пользователях с сортировкой
    ```
    {
      phoneUsers(first: 3 order: { login: "DESC" }) {
        edges {
          node {
            _id
            login
            phone
          }
        }
      }
    }
    ```

## Добавляем поиск по вложенному полю

1. К классу `App\Domain\Entity\Subscription` добавляем атрибут для поиска по вложенному полю
    ```php
    #[ApiFilter(SearchFilter::class, properties: ['follower.login' => 'partial'])]
    ```
2. В браузере перезапускаем страницу с GraphQL-песочницей и получаем данные с фильтрацией подзапроса
    ```
    {
      phoneUser(id: "/api-platform/phone_users/1") {
        _id
        login
        subscriptionFollowers(follower_login: "user3") {
          edges {
            node {
              follower {
                _id
                login
              }
            }
          }
        }
      }
    }
    ```

## Работаем с мутаторами

1. Добавим пользователя запросом
    ```
    mutation CreatePhoneUser($login:String!, $password:String!, $age:Int!, $phone:String!) {
      createPhoneUser(input:{login:$login, password:$password, age:$age, isActive:true, roles:[], phone:$phone}) {
        phoneUser {
          _id
        }
      }
    }
    ```
   с переменными
    ```json
    {
      "login":"graphql_user",
      "password": "graphql_password",
      "age": 35,
      "phone": "+1234567890"
    }
    ```
2. Изменим пользователя запросом
    ```
    mutation UpdatePhoneUser($id:ID!, $login:String!, $password:String!, $age:Int!, $phone:String!) {
      updatePhoneUser(input:{id:$id, login:$login, password:$password, age:$age, phone:$phone}) {
        phoneUser {
          _id
          login
          password
          age
          phone
        }
      }
    }
    ```
   с переменными
    ```json
    {
      "id":"/api-platform/phone_users/3",
      "login":"new_graphql_user",
      "password": "new_graphql_password",
      "age": 135,
      "phone": "+0987654321"
    }
    ```
3. Удалим пользователя запросом
    ```
    mutation DeletePhoneUser($id:ID!) {
      deletePhoneUser(input:{id:$id}) {
        phoneUser {
          id
        }
      }
    }
    ```
   с переменными
    ```json
    {
      "id":"/api-platform/phone_users/ID"
    }
    ```
   где ID – идентификатор добавленного через GraphQL пользователя.

## Делаем кастомный резолвер коллекций

1. В класс `App\Domain\Entity\User` добавим новое поле, геттер и сеттер
    ```php
    #[ORM\Column(type: 'boolean', nullable: true)]
    private ?bool $isProtected;

    public function isProtected(): bool
    {
        return $this->isProtected ?? false;
    }

    public function setIsProtected(bool $isProtected): void
    {
        $this->isProtected = $isProtected;
    }
    ```
2. Генерируем миграцию командой `php bin/console doctrine:migrations:diff`
3. Проверяем сгенерированную миграцию и применяем её с помощью команды `php bin/console doctrine:migrations:migrate`
4. Добавляем класс `App\Domain\ApiPlatform\GraphQL\Resolver\UserCollectionResolver`
    ```php
    <?php
    
    namespace App\Domain\ApiPlatform\GraphQL\Resolver;
    
    use ApiPlatform\GraphQl\Resolver\QueryCollectionResolverInterface;
    use App\Domain\Entity\User;
    
    class UserCollectionResolver implements QueryCollectionResolverInterface
    {
        private const MASK = '****';
    
        /**
         * @param iterable<User> $collection
         * @param array $context
         *
         * @return iterable<User>
         */
        public function __invoke(iterable $collection, array $context): iterable
        {
            /** @var User $user */
            foreach ($collection as $user) {
                if ($user->isProtected()) {
                    $user->setLogin(self::MASK);
                    $user->setPassword(self::MASK);
                }
            }
    
            return $collection;
        }
    } 
    ```
5. В классе `App\Domain\Entity\User` исправляем атрибут `#[ApiResource]`
    ```php
    #[ApiResource(
        graphQlOperations: [new QueryCollection(resolver: UserCollectionResolver::class, name: 'protected')]
    )]
    ```
6. Изменяем в БД у пользователя с подписчиками значение поля `is_protected` на `true`
7. В браузере перезапускаем страницу с GraphQL-песочницей и получаем данные новым запросом
    ```
    {
      protectedUsers {
        edges {
          node {
            _id
            login
            password
          }
        }
      }
    }
    ```

## Делаем кастомный резолвер одной сущности

1. Добавляем класс `App\Domain\ApiPlatform\GraphQL\Resolver\UserResolver`
    ```php
    <?php
    
    namespace App\Domain\ApiPlatform\GraphQL\Resolver;
    
    use ApiPlatform\GraphQl\Resolver\QueryItemResolverInterface;
    use App\Domain\Entity\User;
    
    class UserResolver implements QueryItemResolverInterface
    {
        private const MASK = '****';
    
        /**
         * @param User|null $item
         */
        public function __invoke($item, array $context): User
        {
            if ($item->isProtected()) {
                $item->setLogin(self::MASK);
                $item->setPassword(self::MASK);
            }
    
            return $item;
        }
    }
    ```
2. В классе `App\Domain\Entity\User` исправляем атрибут `#[ApiResource]`
    ```php
    #[ApiResource(
        graphQlOperations: [
            new Query(),
            new QueryCollection(),
            new QueryCollection(resolver: UserCollectionResolver::class, name: 'protected'),
            new Query(resolver: UserResolver::class, name: 'protected')
        ]
    )]
    ```
3. В браузере перезапускаем страницу с GraphQL-песочницей и получаем данные новым запросом
    ```
    {
      protectedUser(id: "/api-platform/users/1") {
        _id
        login
        password
      }
    }
    ```

## Добавляем фильтрацию в кастомный резолвер

1. Исправляяем класс `App\Domain\ApiPlatform\GraphQL\Resolver\UserResolver`
    ```php
    <?php
    
    namespace App\Domain\ApiPlatform\GraphQL\Resolver;
    
    use ApiPlatform\GraphQl\Resolver\QueryItemResolverInterface;
    use App\Domain\Entity\User;
    use App\Domain\Service\UserService;
    
    class UserResolver implements QueryItemResolverInterface
    {
        private const MASK = '****';
        
        public function __construct(private readonly UserService $userService) {
        }
    
        /**
         * @param User|null $item
         */
        public function __invoke($item, array $context): User
        {
            if (isset($context['args']['_id'])) {
                $item = $this->userService->findUserById($context['args']['_id']);
            } elseif (isset($context['args']['login'])) {
                $item = $this->userService->findUserByLogin($context['args']['login']);
            }
    
            if ($item->isProtected()) {
                $item->setLogin(self::MASK);
                $item->setPassword(self::MASK);
            }
    
            return $item;
        }
    }
    ```
2. В классе `App\Domain\Entity\User` исправляем атрибут `#[ApiResource]`
    ```php
    #[ApiResource(
        graphQlOperations: [
            new Query(),
            new QueryCollection(),
            new QueryCollection(resolver: UserCollectionResolver::class, name: 'protected'),
            new Query(
                resolver: UserResolver::class,
                args: ['_id' => ['type' => 'Int'], 'login' => ['type' => 'String']],
                name: 'protected'
            ),
        ]
    )]
    ```
3. В браузере перезапускаем страницу с GraphQL-песочницей и получаем пользователя по логину
    ```
    {
      protectedUser(login: "my_user") {
        _id
        login
        password
      }
    }
    ```
4. В браузере перезапускаем страницу с GraphQL-песочницей и получаем пользователя по id
    ```
    {
      protectedUser(_id: 1) {
        _id
        login
        password
      }
    }
    ```




# Логирование и пониторинг

Запускаем контейнеры командой `docker-compose up -d`

## Логирование с помощью Monolog

### Добавляем monolog-bundle и логируем сообщения

1. Входим в контейнер командой `docker exec -it php sh`. Дальнейшие команды выполняются из контейнера
2. Устанавливаем пакет `symfony/monolog-bundle`
3. Исправляем класс `App\Controller\Web\CreateUser\v2\Manager`
    ```php
    <?php
    
    namespace App\Controller\Web\CreateUser\v2;
    
    use App\Controller\Web\CreateUser\v2\Input\CreateUserDTO;
    use App\Controller\Web\CreateUser\v2\Output\CreatedUserDTO;
    use App\Domain\Entity\EmailUser;
    use App\Domain\Entity\PhoneUser;
    use App\Domain\Model\CreateUserModel;
    use App\Domain\Service\ModelFactory;
    use App\Domain\Service\UserService;
    use App\Domain\ValueObject\CommunicationChannelEnum;
    use Psr\Log\LoggerInterface;
    
    class Manager
    {
        public function __construct(
            /** @var ModelFactory<CreateUserModel> */
            private readonly ModelFactory $modelFactory,
            private readonly UserService $userService,
            private readonly LoggerInterface $logger,
        ) {
        }
    
        public function create(CreateUserDTO $createUserDTO): CreatedUserDTO
        {
            $this->addLogs();
            $communicationMethod = $createUserDTO->phone ?? $createUserDTO->email;
            $communicationChannel = $createUserDTO->phone === null ? CommunicationChannelEnum::Email : CommunicationChannelEnum::Phone;
            $createUserModel = $this->modelFactory->makeModel(
                CreateUserModel::class,
                $createUserDTO->login,
                $communicationMethod,
                $communicationChannel,
                $createUserDTO->password,
                $createUserDTO->age,
                $createUserDTO->isActive,
                $createUserDTO->roles
            );
            $user = $this->userService->create($createUserModel);
    
            return new CreatedUserDTO(
                $user->getId(),
                $user->getLogin(),
                $user->getAvatarLink(),
                $user->getRoles(),
                $user->getCreatedAt()->format('Y-m-d H:i:s'),
                $user->getUpdatedAt()->format('Y-m-d H:i:s'),
                $user instanceof PhoneUser ? $user->getPhone() : null,
                $user instanceof EmailUser ? $user->getEmail() : null,
            );
        }
        
        private function addLogs(): void
        {
            $this->logger->debug('This is debug message');
            $this->logger->info('This is info message');
            $this->logger->notice('This is notice message');
            $this->logger->warning('This is warning message');
            $this->logger->error('This is error message');
            $this->logger->critical('This is critical message');
            $this->logger->alert('This is alert message');
            $this->logger->emergency('This is emergency message');
        }
    }
    ```
4. Выполняем запрос Add user v2 из Postman-коллекции v6 и проверяем, что логи попадают в файл `var/log/dev.log`

### Настраиваем минимальный уровень логирования и убираем ненужные каналы

1. В файле `config/packages/monolog.yaml` исправляем содержимое секции `when@dev.monolog.handlers.main`
    ```yaml
    type: stream
    path: "%kernel.logs_dir%/%kernel.environment%.log"
    level: critical
    channels: ["!event", "!doctrine", "!cache"]
    ```
2. Выполняем запрос Add user v2 из Postman-коллекции v6 и проверяем, что в файл `var/log/dev.log` попадают только
   сообщения с уровнями `critical`, `alert` и `emergency`

### Настраиваем режим fingers crossed

1. В файле `config/packages/monolog.yaml`
   1. Заменяем содержимое секции `when@dev.monolog.handlers.main`
       ```yaml
       type: fingers_crossed
       action_level: error
       handler: nested
       buffer_size: 3
       channels: ["!event", "!doctrine", "!cache"]
       ```
   2. Добавляем в секцию `when@dev.monolog.handlers`
       ```yaml
       nested:
           type: stream
           path: "%kernel.logs_dir%/%kernel.environment%.log"
           level: debug
       ```
2. Выполняем запрос Add user v2 из Postman-коллекции v6 и проверяем, что в файл `var/log/dev.log` дополнительно попадают
   сообщение с уровнем `error` и два предыдущих сообщения с уровнем ниже

### Добавляем форматирование

1. В файле `config/packages/monolog.yaml`
   1. Добавляем секцию `monolog.services`
       ```yaml
       services:
          monolog.formatter.app_formatter:
               class: Monolog\Formatter\LineFormatter
               arguments:
                   - "[%%level_name%%]: [%%datetime%%] %%message%%\n"
       ```
   2. В секцию `when@dev.monolog.handlers.main` добавляем форматтер
       ```yaml
       formatter: monolog.formatter.app_formatter
       ```
2. Выполняем запрос Add user v2 из Postman-коллекции v6 и проверяем, что в файл `var/log/dev.log` новые сообщения
   попадают с новом формате

## Best practices реализации логирования

### Декорируем сервис

1. Исправялем класс `App\Controller\Web\CreateUser\v2\Manager`
    ```php
    <?php
    
    namespace App\Controller\Web\CreateUser\v2;
    
    use App\Controller\Web\CreateUser\v2\Input\CreateUserDTO;
    use App\Controller\Web\CreateUser\v2\Output\CreatedUserDTO;
    use App\Domain\Entity\EmailUser;
    use App\Domain\Entity\PhoneUser;
    use App\Domain\Model\CreateUserModel;
    use App\Domain\Service\ModelFactory;
    use App\Domain\Service\UserService;
    use App\Domain\ValueObject\CommunicationChannelEnum;
    
    class Manager
    {
        public function __construct(
            /** @var ModelFactory<CreateUserModel> */
            private readonly ModelFactory $modelFactory,
            private readonly UserService $userService,
        ) {
        }
    
        public function create(CreateUserDTO $createUserDTO): CreatedUserDTO
        {
            $communicationMethod = $createUserDTO->phone ?? $createUserDTO->email;
            $communicationChannel = $createUserDTO->phone === null ? CommunicationChannelEnum::Email : CommunicationChannelEnum::Phone;
            $createUserModel = $this->modelFactory->makeModel(
                CreateUserModel::class,
                $createUserDTO->login,
                $communicationMethod,
                $communicationChannel,
                $createUserDTO->password,
                $createUserDTO->age,
                $createUserDTO->isActive,
                $createUserDTO->roles
            );
            $user = $this->userService->create($createUserModel);
    
            return new CreatedUserDTO(
                $user->getId(),
                $user->getLogin(),
                $user->getAvatarLink(),
                $user->getRoles(),
                $user->getCreatedAt()->format('Y-m-d H:i:s'),
                $user->getUpdatedAt()->format('Y-m-d H:i:s'),
                $user instanceof PhoneUser ? $user->getPhone() : null,
                $user instanceof EmailUser ? $user->getEmail() : null,
            );
        }
    }
    ```
2. Добавляем класс `App\Controller\Web\CreateUser\v2\ManagerLoggerDecorator`
    ```php
    <?php
    
    namespace App\Controller\Web\CreateUser\v2;
    
    use App\Controller\Web\CreateUser\v2\Input\CreateUserDTO;
    use App\Controller\Web\CreateUser\v2\Output\CreatedUserDTO;
    use App\Domain\Model\CreateUserModel;
    use App\Domain\Service\ModelFactory;
    use App\Domain\Service\UserService;
    use Psr\Log\LoggerInterface;
    
    class ManagerLoggerDecorator extends Manager
    {
        public function __construct(
            /** @var ModelFactory<CreateUserModel> */
            private readonly ModelFactory $modelFactory,
            private readonly UserService $userService,
            private readonly LoggerInterface $logger,
        ) {
            parent::__construct($this->modelFactory, $this->userService);
        }
    
        public function create(CreateUserDTO $createUserDTO): CreatedUserDTO
        {
            $this->addLogs();
            
            return parent::create($createUserDTO);
        }
    
        private function addLogs(): void
        {
            $this->logger->debug('This is debug message');
            $this->logger->info('This is info message');
            $this->logger->notice('This is notice message');
            $this->logger->warning('This is warning message');
            $this->logger->error('This is error message');
            $this->logger->critical('This is critical message');
            $this->logger->alert('This is alert message');
            $this->logger->emergency('This is emergency message');
        }
    }
    ```
3. В файле `config/services.yaml` добавляем описание нового сервиса
    ```php
    App\Controller\Web\CreateUser\v2\ManagerLoggerDecorator:
        decorates: App\Controller\Web\CreateUser\v2\Manager
    ```
4. Выполняем запрос Add user v2 из Postman-коллекции v6 и проверяем, что в файл `var/log/dev.log` попадают сообщения

### Используем классический паттерн "Декоратор"

1. Создаем интерфейс `App\Controller\Web\CreateUser\v1\ManagerInterface`
    ```php
    <?php

    namespace App\Controller\Web\CreateUser\v2;
    
    use App\Controller\Web\CreateUser\v2\Input\CreateUserDTO;
    use App\Controller\Web\CreateUser\v2\Output\CreatedUserDTO;
    
    interface ManagerInterface
    {
        public function create(CreateUserDTO $createUserDTO): CreatedUserDTO;
    }
    ```
2. Имплементируем его в `App\Controller\Web\CreateUser\v2\Manager`
3. Исправляем класс `App\Controller\Web\CreateUser\v2\ManagerLoggerDecorator`
    ```php
    <?php
    
    namespace App\Controller\Web\CreateUser\v2;
    
    use App\Controller\Web\CreateUser\v2\Input\CreateUserDTO;
    use App\Controller\Web\CreateUser\v2\Output\CreatedUserDTO;
    use Psr\Log\LoggerInterface;
    
    class ManagerLoggerDecorator implements ManagerInterface
    {
        public function __construct(
            private readonly ManagerInterface $manager,
            private readonly LoggerInterface $logger,
        ) {
        }
    
        public function create(CreateUserDTO $createUserDTO): CreatedUserDTO
        {
            $this->addLogs();
    
            return $this->manager->create($createUserDTO);
        }
    
        private function addLogs(): void
        {
            $this->logger->debug('This is debug message');
            $this->logger->info('This is info message');
            $this->logger->notice('This is notice message');
            $this->logger->warning('This is warning message');
            $this->logger->error('This is error message');
            $this->logger->critical('This is critical message');
            $this->logger->alert('This is alert message');
            $this->logger->emergency('This is emergency message');
        }
    }
    ```
4. В классе `App\Controller\Web\CreateUser\v2\Controller` делаем инъекцию зависимости через интерфейс
5. В `config/services.yaml`
   1. Исправляем описание сервиса `App\Controller\Web\CreateUser\v2\ManagerLoggerDecorator`
       ```yaml
       App\Controller\Web\CreateUser\v2\ManagerLoggerDecorator:
           arguments:
               $manager: '@App\Controller\Web\CreateUser\v2\Manager'
       ```
   2. Добавляем биндинг интерфейса
       ```yaml
       App\Controller\Web\CreateUser\v2\ManagerInterface:
           alias: App\Controller\Web\CreateUser\v2\ManagerLoggerDecorator
       ```
6. Выполняем запрос Add user v2 из Postman-коллекции v6 и проверяем, что в файл `var/log/dev.log` попадают сообщения

### Логируем с помощью событий

1. Добавляем класс `App\Domain\Event\UserIsCreatedEvent`
    ```php
    <?php
    
    namespace App\Domain\Event;
    
    class UserIsCreatedEvent
    {
        public function __construct(
            public readonly int $id,
            public readonly string $login,
        ) {
        }
    }
    ```
2. Исправляем класс `App\Domain\EventSubscriber\UserEventSubscriber`
    ```php
    <?php
    
    namespace App\Domain\EventSubscriber;
    
    use App\Domain\Event\CreateUserEvent;
    use App\Domain\Event\UserIsCreatedEvent;
    use App\Domain\Service\UserService;
    use Psr\Log\LoggerInterface;
    use Symfony\Component\EventDispatcher\EventSubscriberInterface;
    
    class UserEventSubscriber implements EventSubscriberInterface
    {
        public function __construct(
            private readonly UserService $userService,
            private readonly LoggerInterface $logger,
        ) {
        }
    
        public static function getSubscribedEvents(): array
        {
            return [
                CreateUserEvent::class => 'onCreateUser',
                UserIsCreatedEvent::class => 'onUserIsCreated',
            ];
        }
    
        public function onCreateUser(CreateUserEvent $event): void
        {
            $user = null;
    
            if ($event->phone !== null) {
                $user = $this->userService->createWithPhone($event->login, $event->phone);
            } elseif ($event->email !== null) {
                $user = $this->userService->createWithEmail($event->login, $event->email);
            }
    
            $event->id = $user?->getId();
    
        }
    
        public function onUserIsCreated(UserIsCreatedEvent $event): void
        {
            $this->logger->info("User is created: id {$event->id}, login {$event->login}");
        }
    }
    ```
3. В классе `App\Domain\Service\UserService`
   1. добавляем зависимость от `EventDispatcherInterface`
   2. исправляем метод `create`
       ```php
       public function create(CreateUserModel $createUserModel): User
       {
           $user = match($createUserModel->communicationChannel) {
                CommunicationChannelEnum::Email => (new EmailUser())->setEmail($createUserModel->communicationMethod),
                CommunicationChannelEnum::Phone => (new PhoneUser())->setPhone($createUserModel->communicationMethod),
           };
           $user->setLogin($createUserModel->login);
           $user->setPassword($this->userPasswordHasher->hashPassword($user, $createUserModel->password));
           $user->setAge($createUserModel->age);
           $user->setIsActive($createUserModel->isActive);
           $user->setRoles($createUserModel->roles);
           $this->userRepository->create($user);
           $this->eventDispatcher->dispatch(new UserIsCreatedEvent($user->getId(), $user->getLogin()));
    
           return $user;
       }
       ```
4. В файле `config/packages/monolog.yaml` в секции `when@dev.monolog.handlers` удаляем подсекцию `nested` и исправляем
   секцию `main`
    ```yaml
    main:
        type: stream
        path: "%kernel.logs_dir%/%kernel.environment%.log"
        level: debug
        channels: ["!event", "!doctrine", "!cache"]
        formatter: monolog.formatter.app_formatter 
    ```
5. Выполняем запрос Add user v2 из Postman-коллекции v6 и проверяем, что в файл `var/log/dev.log` попадают новые
   сообщения

## Elasticsearch и Kibana для логов

1. Добавляем сервисы `elasticsearch` и `kibana` в `docker-compose.yml`
    ```yaml
    elasticsearch:
        image: docker.elastic.co/elasticsearch/elasticsearch:7.9.2
        container_name: 'elasticsearch'
        environment:
          - cluster.name=docker-cluster
          - bootstrap.memory_lock=true
          - discovery.type=single-node
          - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
        ulimits:
          memlock:
            soft: -1
            hard: -1
        ports:
          - 9200:9200
          - 9300:9300

    kibana:
        image: docker.elastic.co/kibana/kibana:7.9.2
        container_name: 'kibana'
        depends_on:
          - elasticsearch
        ports:
          - 5601:5601
    ```
2. Выходим из контейнера и запускаем новые контейнеры командой `docker-compose up -d`
3. Заходим в контейнер командой `docker exec -it php sh`. Дальнейшие команды выполняются из контейнера
4. Устанавливаем пакет `symfony/http-client`
5. В файле `config/packages/monolog.yaml`
   1. добавляем в `monolog.channels` новый канал `elasticsearch`
   2. Добавляем новый обработчик в секцию `monolog.handlers`
       ```yaml
       elasticsearch:
           type: service
           id: Symfony\Bridge\Monolog\Handler\ElasticsearchLogstashHandler
           channels: elasticsearch
       ```
   3. Добавляем новые сервисы в секцию `services`:
       ```yaml
       Psr\Log\NullLogger:
           class: Psr\Log\NullLogger
     
       http_client_without_logs:
           class: Symfony\Component\HttpClient\CurlHttpClient
           calls:
               - [setLogger, ['@Psr\Log\NullLogger']]
       
       Symfony\Bridge\Monolog\Handler\ElasticsearchLogstashHandler:
           arguments:
               - 'http://elasticsearch:9200'
               - 'monolog'
               - '@http_client_without_logs'
       ```
6. В классе `App\Domain\EventSubscriber\UserEventSubscriber`
   1. Изменяем название параметра `$logger` на `$elasticsearchLogger`
   2. исправляем метод `onUserIsCreated`
       ```php
       public function onUserIsCreated(UserIsCreatedEvent $event): void
       {
           $this->elasticsearchLogger->info("User is created: id {$event->id}, login {$event->login}");
       }
       ```
7. Выполняем запрос Add user v2 из Postman-коллекции v6
8. Заходим в Kibana `http://localhost:5601`.
9. Заходим в Stack Management -> Index Patterns
10. Создаём index pattern на базе индекса `monolog`, переходим в `Discover`, видим наше сообщение

## Grafana для сбора метрик, интеграция с Graphite

1. Устанавливаем пакет `slickdeals/statsd`
2. Добавляем сервисы Graphite и Grafana в `docker-compose.yml`
    ```yaml
    graphite:
        image: graphiteapp/graphite-statsd
        container_name: 'graphite'
        restart: always
        ports:
          - 8000:80
          - 2003:2003
          - 2004:2004
          - 2023:2023
          - 2024:2024
          - 8125:8125/udp
          - 8126:8126

    grafana:
        image: grafana/grafana
        container_name: 'grafana'
        restart: always
        ports:
          - 3000:3000
    ```
3. Выходим из контейнера `php` и запускаем новые контейнеры командой `docker-compose up -d`
4. Проверяем, что можем зайти в интерфейс Graphite по адресу `localhost:8000`
5. Проверяем, что можем зайти в интерфейс Grafana по адресу `localhost:3000`, логин / пароль - `admin` / `admin`
6. Добавляем класс `App\Storage\MetricsStorage`
    ```php
    <?php
    
    namespace App\Infrastructure\Storage;
    
    use Domnikl\Statsd\Client;
    use Domnikl\Statsd\Connection\UdpSocket;
    
    class MetricsStorage
    {
        public const USER_CREATED = 'user_created';

        private const DEFAULT_SAMPLE_RATE = 1.0;
    
        private Client $client;
    
        public function __construct(string $host, int $port, string $namespace)
        {
            $connection = new UdpSocket($host, $port);
            $this->client = new Client($connection, $namespace);
        }
    
        public function increment(string $key, ?float $sampleRate = null, ?array $tags = null): void
        {
            $this->client->increment($key, $sampleRate ?? self::DEFAULT_SAMPLE_RATE, $tags ?? []);
        }
    }
    ```
7. В файле `config/services.yaml` добавляем новое описание сервиса
    ```yaml
    App\Infrastructure\Storage\MetricsStorage:
        arguments: 
            - graphite
            - 8125
            - my_app
    ```
8. В классе `App\Domain\EventSubscriber\UserEventSubscriber`
   1. Добавляем зависимость от `MetricsStorage`
   2. Исправляем метод `onUserIsCreated`
       ```php
       public function onUserIsCreated(UserIsCreatedEvent $event): void
       {
           $this->elasticsearchLogger->info("User is created: id {$event->id}, login {$event->login}");
           $this->metricsStorage->increment(MetricsStorage::USER_CREATED);
       }
       ```
9. Выполняем несколько раз запрос Add user v2 из Postman-коллекции v6 и проверяем, что в Graphite появляются события
10. Настраиваем график в Grafana
   1. добавляем в Data source с типом Graphite и адресом graphite:80
   2. добавляем новый Dashboard
   3. на дашборде добавляем панель с запросом в Graphite счётчика `stats_counts.my_app.user_created`
   4. видим график с запросами
11. Выполняем ещё несколько раз запрос Add user v2 из Postman-коллекции v6 и проверяем, что в Grafana обновились данные





# Кэширование

## Memcached в качестве кэша Doctrine

### Устанавливаем Memcached

1. Добавляем в файл `docker/Dockerfile`
   1. Установку пакета `libmemcached-dev` через `apk`
   2. Установку расширения `memcached` через `pecl`
   3. Включение расширения командой `echo "extension=memcached.so" > /usr/local/etc/php/conf.d/memcached.ini`
2. Добавляем сервис Memcached в `docker-compose.yml`
    ```yaml
    memcached:
        image: memcached:latest
        container_name: 'memcached'
        restart: always
        ports:
           - 11211:11211
    ```
3. В файл `.env` добавляем
    ```shell
    MEMCACHED_DSN=memcached://memcached:11211
    ```
4. Пересобираем и запускаем контейнеры командой `docker-compose up -d --build`
5. Подключаемся к Memcached командой `telnet 127.0.0.1 11211` и проверяем, что он пустой (команда `stats items`)

### Добавляем контроллер для получения твитов

1. Выполняем запрос Add user v2 из Postman-коллекции v7
2. Добавим в БД 10 тысяч случайных твитов запросом
    ```sql
    INSERT INTO tweet (created_at, updated_at, author_id, text)
    SELECT NOW(), NOW(), 1, md5(random()::TEXT) FROM generate_series(1,10000);
    ```
3. В классе `App\Infrastructure\Repository\TweetRepository` добавляем метод `getTweetsPaginated`
    ```php
    /**
     * @return Tweet[]
     */
    public function getTweetsPaginated(int $page, int $perPage): array
    {
        $qb = $this->entityManager->createQueryBuilder();
        $qb->select('t')
            ->from(Tweet::class, 't')
            ->orderBy('t.id', 'DESC')
            ->setFirstResult($perPage * $page)
            ->setMaxResults($perPage);

        return $qb->getQuery()->getResult();
    }
    ```
4. В класс `App\Domain\Service\TweetService` добавляем метод `getTweetsPaginated`
    ```php
    /**
     * @return Tweet[]
     */
    public function getTweetsPaginated(int $page, int $perPage): array
    {
        return $this->tweetRepository->getTweetsPaginated($page, $perPage);
    }
    ```
5. Добавляем класс `App\Controller\Web\GetTweet\v1\Output\TweetDTO`
    ```php
    <?php
    
    namespace App\Controller\Web\GetTweet\v1\Output;
    
    class TweetDTO
    {
        public function __construct(
            public readonly int $id,
            public readonly string $text,
            public readonly string $author,
            public readonly string $createdAt 
        ) {
        }
    }
    ```
6. Добавляем класс `App\Controller\Web\GetTweet\v1\Manager`
    ```php
    <?php
    
    namespace App\Controller\Web\GetTweet\v1;
    
    use App\Controller\Web\GetTweet\v1\Output\TweetDTO;
    use App\Domain\Entity\Tweet;
    use App\Domain\Service\TweetService;
    
    class Manager
    {
        public function __construct(private readonly TweetService $tweetService)
        {
        }
    
        /**
         * @return Tweet[]
         */
        public function getTweetsPaginated(int $page, int $perPage): array
        {
            return array_map(
                static fn (Tweet $tweet) => new TweetDTO(
                    $tweet->getId(),
                    $tweet->getText(),
                    $tweet->getAuthor()->getLogin(),
                    $tweet->getCreatedAt()->format('Y-m-d H:i:s'),
                ),
                $this->tweetService->getTweetsPaginated($page, $perPage)
            );
        }
    }
    ```
7. Добавляем класс `App\Controller\Web\GetTweet\v1\Controller`
    ```php
    <?php
    
    namespace App\Controller\Web\GetTweet\v1;
    
    use Symfony\Component\HttpFoundation\JsonResponse;
    use Symfony\Component\HttpFoundation\Response;
    use Symfony\Component\HttpKernel\Attribute\AsController;
    use Symfony\Component\HttpKernel\Attribute\MapQueryParameter;
    use Symfony\Component\Routing\Attribute\Route;
    
    #[AsController]
    class Controller
    {
        public function __construct(private readonly Manager $manager) {
        }
    
        #[Route(path: 'api/v1/get-tweet', methods: ['GET'])]
        public function __invoke(#[MapQueryParameter]int $page, #[MapQueryParameter]int $perPage): Response
        {
            return new JsonResponse(['tweets' => $this->manager->getTweetsPaginated($page, $perPage)]);
        }
    }
    ```
8. Выполняем запрос Get tweet из Postman-коллекции v7, видим, что результат возвращается

### Включаем кэширование в Doctrine

1. В файле `config/packages/doctrine.yaml`:
   1. Добавляем в секцию `orm`
       ```yaml
       metadata_cache_driver:
           type: pool
           pool: doctrine.system_cache_pool
       query_cache_driver:
           type: pool
           pool: doctrine.system_cache_pool
       result_cache_driver:
           type: pool
           pool: doctrine.result_cache_pool
       ```
   2. Добавляем секцию `services`
       ```yaml
       services:
           doctrine_memcached_provider:
               class: Memcached
               factory: Symfony\Component\Cache\Adapter\MemcachedAdapter::createConnection
               arguments:
                   - '%env(MEMCACHED_DSN)%'
                   - PREFIX_KEY: 'my_app_doctrine'
       ```
   3. Добавляем секцию `framework`
       ```yaml
       framework:
           cache:
               pools:
                   doctrine.result_cache_pool:
                       adapter: cache.adapter.memcached
                       provider: doctrine_memcached_provider
                   doctrine.system_cache_pool:
                       adapter: cache.adapter.memcached
                       provider: doctrine_memcached_provider
       ```
2. Выполняем запрос Get Tweet из Postman-коллекции v7 для прогрева кэша
3. Проверяем, что кэш прогрелся
   1. В Memcached выполняем `stats items`, видим там запись (или две записи)
   2. Выводим каждую запись командой `stats cachedump K 1000`, где K - идентификатор записи
   3. Получаем содержимое ключей командой `get KEY`, где `KEY` - ключ из записи
   4. Удостоверяемся, что это query и metadata кэши

### Добавляем кэширование результата запроса

1. Включаем result cache в класс `App\Infrastructure\Repository\TweetRepository` в методе `getTweetsPaginated` в
   последней строке
    ```php
    return $qb->getQuery()->enableResultCache(null, "tweets_{$page}_$perPage")->getResult();
    ```
2. Выполняем запрос Get tweet из Postman-коллекции v7 для прогрева кэша
3. В Memcached находим ключ с суффиксом tweets_PAGE_PER_PAGE, где `PAGE` и `PER_PAGE` - значения одноимённых параметров
   запроса, и выполняем для него команду `get`, видим содержимое result cache

## Redis в качестве кэша на уровне приложения

### Подключаем redis

1. Добавляем сервис Memcached в `docker-compose.yml`
    ```yaml
    redis:
        container_name: 'redis'
        image: redis:alpine
        ports:
          - 6379:6379
    ```
2. Для включения кэша на уровне приложения в файле `config/packages/cache.yaml` добавляем в секцию `cache`
    ```yaml
    app: cache.adapter.redis
    default_redis_provider: '%env(REDIS_DSN)%'
    ```
3. В файл `.env` добавляем
    ```shell
    REDIS_DSN=redis://redis:6379
    ```
4. Запускаем новые контейнеры командой `docker-compose up -d`
5. Подключаемся к Redis командой `telnet 127.0.0.1 6379`
6. Выполняем `keys *`, видим, что кэш пустой

### Подключаем кэш на уровне приложения

1. Заходим в контейнер командой `docker exec -it php sh`
2. Обновляем пакет `symfony/cache` (минимальная подходящая нам версия 7.1.4)
3. Добавляем класс `App\Domain\Model\TweetModel`
    ```php
    <?php
    
    namespace App\Domain\Model;
    
    use DateTime;
    
    class TweetModel
    {
        public function __construct(
            public readonly int $id,
            public readonly string $author,
            public readonly string $text,
            public readonly DateTime $createdAt,
        ) {
        }
    }
    ``` 
4. Добавляем интерфейс `App\Domain\Repository\TweetRepositoryInterface`
    ```php
    <?php
    
    namespace App\Domain\Repository;
    
    use App\Domain\Entity\Tweet;
    use App\Domain\Model\TweetModel;
    
    interface TweetRepositoryInterface
    {
        public function create(Tweet $tweet): int;
    
        /**
         * @return TweetModel[]
         */
        public function getTweetsPaginated(int $page, int $perPage): array;
    }
    ```
5. Добавляем класс `App\Infrastructure\Repository\TweetRepositoryCacheDecorator`
    ```php
    <?php
    
    namespace App\Infrastructure\Repository;
    
    use App\Domain\Entity\Tweet;
    use App\Domain\Model\TweetModel;
    use App\Domain\Repository\TweetRepositoryInterface;
    use Psr\Cache\CacheItemPoolInterface;
    use Psr\Cache\InvalidArgumentException;
    
    class TweetRepositoryCacheDecorator implements TweetRepositoryInterface
    {
        public function __construct(
            private readonly TweetRepository $tweetRepository,
            private readonly CacheItemPoolInterface $cacheItemPool,
        ) {
        }
    
        public function create(Tweet $tweet): int
        {
            return $this->tweetRepository->create($tweet);
        }
    
        /**
         * @return TweetModel[]
         * @throws InvalidArgumentException
         */
        public function getTweetsPaginated(int $page, int $perPage): array
        {
            $tweetsItem = $this->cacheItemPool->getItem($this->getCacheKey($page, $perPage));
            if (!$tweetsItem->isHit()) {
                $tweets = $this->tweetRepository->getTweetsPaginated($page, $perPage);
                $tweetsItem->set(
                    array_map(
                        static fn (Tweet $tweet): TweetModel => new TweetModel(
                            $tweet->getId(),
                            $tweet->getAuthor()->getLogin(),
                            $tweet->getText(),
                            $tweet->getCreatedAt(),
                        ),
                        $tweets
                    )
                );
                $this->cacheItemPool->save($tweetsItem);
            }
    
            return $tweetsItem->get();
        }
    
        private function getCacheKey(int $page, int $perPage): string
        {
            return "tweets_{$page}_$perPage";
        }
    }
    ```
6. В файле `config/services.yaml` добавляем новый биндинг
    ```yaml
    App\Domain\Repository\TweetRepositoryInterface:
        alias: App\Infrastructure\Repository\TweetRepositoryCacheDecorator
    ```
7. Исправляем класс `App\Domain\Service\TweetService`
    ```php
    <?php
    
    namespace App\Domain\Service;
    
    use App\Domain\Entity\Tweet;
    use App\Domain\Entity\User;
    use App\Domain\Model\TweetModel;
    use App\Domain\Repository\TweetRepositoryInterface;
    
    class TweetService
    {
        public function __construct(private readonly TweetRepositoryInterface $tweetRepository)
        {
        }
    
        public function postTweet(User $author, string $text): void
        {
            $tweet = new Tweet();
            $tweet->setAuthor($author);
            $tweet->setText($text);
            $tweet->setCreatedAt();
            $tweet->setUpdatedAt();
            $author->addTweet($tweet);
            $this->tweetRepository->create($tweet);
        }
    
        /**
         * @return TweetModel[]
         */
        public function getTweetsPaginated(int $page, int $perPage): array
        {
            return $this->tweetRepository->getTweetsPaginated($page, $perPage);
        }
    }
    ```
8. Исправляем класс `App\Controller\Web\GetTweet\v1\Manager`
    ```php
    <?php
    
    namespace App\Controller\Web\GetTweet\v1;
    
    use App\Controller\Web\GetTweet\v1\Output\TweetDTO;
    use App\Domain\Model\TweetModel;
    use App\Domain\Service\TweetService;
    
    class Manager
    {
        public function __construct(private readonly TweetService $tweetService)
        {
        }
    
        /**
         * @return TweetModel[]
         */
        public function getTweetsPaginated(int $page, int $perPage): array
        {
            return array_map(
                static fn (TweetModel $tweet) => new TweetDTO(
                    $tweet->id,
                    $tweet->text,
                    $tweet->author,
                    $tweet->createdAt->format('Y-m-d H:i:s'),
                ),
                $this->tweetService->getTweetsPaginated($page, $perPage)
            );
        }
    }
    ```
9. Выполняем запрос Get tweet из Postman-коллекции v7 для прогрева кэша
10. В Redis ищем ключи от приложения командой `keys *tweets*`
11. Выводим найденный ключ командой `get KEY`, где `KEY` - найденный ключ

### Подсчитываем количество cache hit/miss

1. В классе `App\Infrastructure\Storage\MetricsStorage` добавляем новые константы
    ```php
    public const CACHE_HIT_PREFIX = 'cache.hit.';
    public const CACHE_MISS_PREFIX = 'cache.miss.';
    ```
2. Добавляем класс `App\Application\Symfony\AdapterCountingDecorator`
    ```php
    <?php
    
    namespace App\Application\Symfony;
    
    use App\Infrastructure\Storage\MetricsStorage;
    use Psr\Cache\CacheItemInterface;
    use Psr\Cache\InvalidArgumentException;
    use Psr\Log\LoggerAwareInterface;
    use Psr\Log\LoggerInterface;
    use Symfony\Component\Cache\Adapter\AbstractAdapter;
    use Symfony\Component\Cache\Adapter\AdapterInterface;
    use Symfony\Component\Cache\CacheItem;
    use Symfony\Component\Cache\ResettableInterface;
    use Symfony\Contracts\Cache\CacheInterface;
    
    class AdapterCountingDecorator implements AdapterInterface, CacheInterface, LoggerAwareInterface, ResettableInterface
    {
        public function __construct(
            private readonly AbstractAdapter $adapter,
            private readonly MetricsStorage $metricsStorage,
        )
        {
            $this->adapter->setCallbackWrapper(null);
        }
    
        public function getItem($key): CacheItem
        {
            $result = $this->adapter->getItem($key);
            $this->incCounter($result);
    
            return $result;
        }
    
        /**
         * @param string[] $keys
         *
         * @return iterable
         *
         * @throws InvalidArgumentException
         */
        public function getItems(array $keys = []): array
        {
            $result = $this->adapter->getItems($keys);
            foreach ($result as $item) {
                $this->incCounter($item);
            }
    
            return $result;
        }
    
        public function clear(string $prefix = ''): bool
        {
            return $this->adapter->clear($prefix);
        }
    
        public function get(string $key, callable $callback, float $beta = null, array &$metadata = null): mixed
        {
            return $this->adapter->get($key, $callback, $beta, $metadata);
        }
    
        public function delete(string $key): bool
        {
            return $this->adapter->delete($key);
        }
    
        public function hasItem($key): bool
        {
            return $this->adapter->hasItem($key);
        }
    
        public function deleteItem($key): bool
        {
            return $this->adapter->deleteItem($key);
        }
    
        public function deleteItems(array $keys): bool
        {
            return $this->adapter->deleteItems($keys);
        }
    
        public function save(CacheItemInterface $item): bool
        {
            return $this->adapter->save($item);
        }
    
        public function saveDeferred(CacheItemInterface $item): bool
        {
            return $this->adapter->saveDeferred($item);
        }
    
        public function commit(): bool
        {
            return $this->adapter->commit();
        }
    
        public function setLogger(LoggerInterface $logger): void
        {
            $this->adapter->setLogger($logger);
        }
    
        public function reset(): void
        {
            $this->adapter->reset();
        }
    
        private function incCounter(CacheItemInterface $cacheItem): void
        {
            if ($cacheItem->isHit()) {
                $this->metricsStorage->increment(MetricsStorage::CACHE_HIT_PREFIX.$cacheItem->getKey());
            } else {
                $this->metricsStorage->increment(MetricsStorage::CACHE_MISS_PREFIX.$cacheItem->getKey());
            }
        }
    }
    ```
3. В файле `config/services.yaml` добавляем новые описания сервисов
    ```yaml
    redis_client:
        class: Redis
        factory: Symfony\Component\Cache\Adapter\RedisAdapter::createConnection
        arguments:
            - '%env(REDIS_DSN)%'

    redis_adapter:
        class: Symfony\Component\Cache\Adapter\RedisAdapter
        arguments:
            - '@redis_client'
            - 'my_app'

    App\Application\Symfony\AdapterCountingDecorator:
        arguments:
            $adapter: '@redis_adapter'

    App\Infrastructure\Repository\TweetRepositoryCacheDecorator:
        arguments:
            $cacheItemPool: '@App\Application\Symfony\AdapterCountingDecorator'
    ```
4. Выполняем два одинаковых запроса Get Tweet list из Postman-коллекции v6 для прогрева кэша и появления метрик
5. Заходим в Grafana, добавляем новую панель
6. Добавляем на панель метрики `sumSeries(stats_counts.my_app.cache.hit.*)` и
   `sumSeries(stats_counts.my_app.cache.miss.*)`

### Инвалидация кэша с помощью тэгов

1. В классе `App\Entity\Tweet` добавляем атрибут `ORM\HasLifecycleCallbacks` для класса и атрибуты для методов
   `setCreatedAt()` и `setUpdatedAt()`
    ```php
    #[ORM\PrePersist]
    public function setCreatedAt(): void {
        $this->createdAt = new DateTime();
    }

    #[ORM\PrePersist]
    #[ORM\PreUpdate]
    public function setUpdatedAt(): void {
        $this->updatedAt = new DateTime();
    }
    ```
2. В классе `App\Domain\Service\TweetService` исправляем метод `postTweet`
    ```php
    public function postTweet(User $author, string $text): void
    {
        $tweet = new Tweet();
        $tweet->setAuthor($author);
        $tweet->setText($text);
        $author->addTweet($tweet);
        $this->tweetRepository->create($tweet);
    }
    ```
3. В файле `config/services.yaml`
   1. добавляем новое описание сервиса
       ```yaml
       redis_tag_aware_adapter:
           class: Symfony\Component\Cache\Adapter\RedisTagAwareAdapter
           arguments:
               - '@redis_client'
               - 'my_app'
       ```
   2. исправляем описание для сервиса `App\Infrastructure\Repository\TweetRepositoryCacheDecorator`
       ```yaml
       App\Infrastructure\Repository\TweetRepositoryCacheDecorator:
           arguments:
               $cache: '@redis_tag_aware_adapter'
       ```
4. Исправляем класс `App\Infrastructure\Repository\TweetRepositoryCacheDecorator`
    ```php
    <?php
    
    namespace App\Infrastructure\Repository;
    
    use App\Domain\Entity\Tweet;
    use App\Domain\Model\TweetModel;
    use App\Domain\Repository\TweetRepositoryInterface;
    use Psr\Cache\CacheException;
    use Psr\Cache\InvalidArgumentException;
    use Symfony\Contracts\Cache\ItemInterface;
    use Symfony\Contracts\Cache\TagAwareCacheInterface;
    
    class TweetRepositoryCacheDecorator implements TweetRepositoryInterface
    {
        public function __construct(
            private readonly TweetRepository $tweetRepository,
            private readonly TagAwareCacheInterface $cache,
        ) {
        }
    
        /**
         * @throws InvalidArgumentException
         */
        public function create(Tweet $tweet): int
        {
            $result = $this->tweetRepository->create($tweet);
            $this->cache->invalidateTags([$this->getCacheTag()]);
            
            return $result;
        }
    
        /**
         * @return TweetModel[]
         * @throws InvalidArgumentException
         * @throws CacheException
         */
        public function getTweetsPaginated(int $page, int $perPage): array
        {
            return $this->cache->get(
                $this->getCacheKey($page, $perPage),
                function (ItemInterface $item) use ($page, $perPage) {
                    $tweets = $this->tweetRepository->getTweetsPaginated($page, $perPage);
                    $tweetModels = array_map(
                        static fn (Tweet $tweet): TweetModel => new TweetModel(
                            $tweet->getId(),
                            $tweet->getAuthor()->getLogin(),
                            $tweet->getText(),
                            $tweet->getCreatedAt(),
                        ),
                        $tweets
                    );
                    $item->set($tweetModels);
                    $item->tag($this->getCacheTag());
                    
                    return $tweetModels;
                }
            );
        }
    
        private function getCacheKey(int $page, int $perPage): string
        {
            return "tweets_{$page}_$perPage";
        }
        
        private function getCacheTag(): string
        {
            return 'tweets';
        }
    }
    ``` 
5. Добавляем класс `App\Controller\Web\PostTweet\v1\Input\PostTweetDTO`
    ```php
    <?php
    
    namespace App\Controller\Web\PostTweet\v1\Input;
    
    class PostTweetDTO
    {
        public function __construct(
            public readonly int $userId,
            public readonly string $text,
        ) {
        }
    }
    ```
6. Добавляем класс `App\Controller\Web\PostTweet\v1\Manager`
    ```php
    <?php
    
    namespace App\Controller\Web\PostTweet\v1;
    
    use App\Controller\Exception\AccessDeniedException;
    use App\Controller\Web\PostTweet\v1\Input\PostTweetDTO;
    use App\Domain\Service\TweetService;
    use App\Domain\Service\UserService;
    
    class Manager
    {
        public function __construct(
            private readonly UserService $userService,
            private readonly TweetService $tweetService,
        ) {
        }
    
        public function postTweet(PostTweetDTO $tweetDTO): bool
        {
            $user = $this->userService->findUserById($tweetDTO->userId);
            
            if ($user === null) {
                return false;
            }
            
            $this->tweetService->postTweet($user, $tweetDTO->text);
            
            return true;
        }
    }
    ```
6. Добавим класс `App\Controller\Web\PostTweet\v1\Controller`
    ```php
    <?php
    
    namespace App\Controller\Web\PostTweet\v1;
    
    use App\Controller\Web\PostTweet\v1\Input\PostTweetDTO;
    use Symfony\Component\HttpFoundation\JsonResponse;
    use Symfony\Component\HttpFoundation\Response;
    use Symfony\Component\HttpKernel\Attribute\AsController;
    use Symfony\Component\HttpKernel\Attribute\MapRequestPayload;
    use Symfony\Component\Routing\Attribute\Route;
    
    #[AsController]
    class Controller
    {
        public function __construct(private readonly Manager $manager) {
        }
    
        #[Route(path: 'api/v1/post-tweet', methods: ['POST'])]
        public function __invoke(#[MapRequestPayload]PostTweetDTO $postTweetDTO): Response
        {
            return new JsonResponse(['success' => $this->manager->postTweet($postTweetDTO)]);
        }
    } 
    ```
7. Выполняем запрос Post tweet из Postman-коллекции v7, видим ошибку
8. Заходим в контейнер с приложением командой `docker exec -it php sh`
9. В контейнере выполняем команду `php bin/console doctrine:cache:clear-metadata`
10. Ещё раз выполняем запрос Post tweet из Postman-коллекции v6, видим успешное сохранение
11. В Redis выполняем `flushall`
12. Выполняем несколько запросов Get tweet из Postman-коллекции v7 с разными значениями параметров для прогрева кэша
13. Проверяем, что в Redis есть ключи для твитов, командой `keys *tweets*`
14. Выполняем запрос Post tweet из Postman-коллекции v7
15. Проверяем, что в Redis удалились все ключи, командой `keys *tweets*`




# Очереди: начало

Запускаем контейнеры командой `docker-compose up -d`

## Установка RabbitMQ и rabbitmq-bundle

1. Добавляем в файл `docker\Dockerfile` установку расширения `pcntl` (в команду `docker-php-ext-install`)
2. В файл `docker-compose.yml` добавляем новый сервис:
    ```yaml
    rabbitmq:
      image: rabbitmq:3-management
      working_dir: /app
      hostname: rabbit-mq
      container_name: 'rabbit-mq'
      ports:
        - 15672:15672
        - 5672:5672
      environment:
        RABBITMQ_DEFAULT_USER: user
        RABBITMQ_DEFAULT_PASS: password
    ```
3. Запускаем контейнеры с пересборкой командой `docker-compose up -d --build`
4. Заходим в контейнер командой `docker exec -it php sh` и устанавливаем пакет `php-amqplib/rabbitmq-bundle`, дальнейшие
   команды выполняются из контейнера
5. Добавляем параметры для подключения к RabbitMQ в файл `.env`
    ```shell
    RABBITMQ_URL=amqp://user:password@rabbit-mq:5672
    RABBITMQ_VHOST=/
    ```
6. Заходим по адресу `localhost:15672` и авторизуемся с указанными реквизитами

## Добавляем функционал для асинхронной обработки

1. В классе `App\Entity\Subscription` добавляем атрибут `ORM\HasLifecycleCallbacks` для класса и атрибуты для методов
   `setCreatedAt()` и `setUpdatedAt()`
    ```php
    #[ORM\PrePersist]
    public function setCreatedAt(): void {
        $this->createdAt = new DateTime();
    }

    #[ORM\PrePersist]
    #[ORM\PreUpdate]
    public function setUpdatedAt(): void {
        $this->updatedAt = new DateTime();
    }
    ```
2. В классе `App\Domain\Service\SubscriptionService` исправляем метод `addSubscription`
    ```php
    public function addSubscription(User $author, User $follower): void
    {
        $subscription = new Subscription();
        $subscription->setAuthor($author);
        $subscription->setFollower($follower);
        $author->addSubscriptionFollower($subscription);
        $follower->addSubscriptionAuthor($subscription);
        $this->subscriptionRepository->create($subscription);
    }
    ```
3. Добавляем класс `App\Service\FollowerService`
    ```php
    <?php
    
    namespace App\Domain\Service;
    
    use App\Domain\Entity\User;
    use App\Domain\Model\CreateUserModel;
    use App\Domain\ValueObject\CommunicationChannelEnum;
    
    class FollowerService
    {
        public function __construct(
            private readonly UserService $userService,
            private readonly SubscriptionService $subscriptionService,
        ) {
            
        }
        
        public function addFollowers(User $user, string $followerLoginPrefix, int $count): int
        {
            $createdFollowers = 0;
            for ($i = 0; $i < $count; $i++) {
                $login = "{$followerLoginPrefix}_$i";
                $channel = random_int(0, 2) === 1 ? CommunicationChannelEnum::Email : CommunicationChannelEnum::Phone;
                $model = new CreateUserModel(
                    $login,
                    match ($channel) {
                        CommunicationChannelEnum::Email => "{$login}@mail.ru",
                        CommunicationChannelEnum::Phone => '+'.str_pad((string)abs(crc32($login)), 10, '0'),
                    },
                    $channel,
                    "{$login}_password",
                );
                $follower = $this->userService->create($model);
                $this->subscriptionService->addSubscription($user, $follower);
                $createdFollowers++;
            }
    
            return $createdFollowers;
        }
    }
    ```
4. Добавляем класс `App\Controller\Web\AddFollowers\v1\Input\AddFollowersDTO`
    ```php
    <?php
    
    namespace App\Controller\Web\AddFollowers\v1\Input;
    
    class AddFollowersDTO
    {
        public function __construct(
           public readonly int $authorId,
           public readonly string $followerLoginPrefix,
           public readonly int $count, 
        ) {
        }
    }
    ```
5. Добавляем класс `App\Controller\Web\AddFollowers\v1\Manager`
    ```php
    <?php
    
    namespace App\Controller\Web\AddFollowers\v1;
    
    use App\Controller\Web\AddFollowers\v1\Input\AddFollowersDTO;
    use App\Domain\Entity\User;
    use App\Domain\Service\FollowerService;
    
    class Manager
    {
        public function __construct(private readonly FollowerService $followerService)
        {
        }
        
        public function addFollowers(User $author, AddFollowersDTO $addFollowersDTO): int
        {
            return $this->followerService->addFollowers($author, $addFollowersDTO->followerLoginPrefix, $addFollowersDTO->count);
        }
    }
    ```
6. Добавляем класс `App\Controller\Web\AddFollowers\v1\Controller`
    ```php
    <?php
    
    namespace App\Controller\Web\AddFollowers\v1;
    
    use App\Controller\Web\AddFollowers\v1\Input\AddFollowersDTO;
    use App\Domain\Entity\User;
    use Symfony\Bridge\Doctrine\Attribute\MapEntity;
    use Symfony\Component\HttpFoundation\JsonResponse;
    use Symfony\Component\HttpFoundation\Response;
    use Symfony\Component\HttpKernel\Attribute\AsController;
    use Symfony\Component\HttpKernel\Attribute\MapRequestPayload;
    use Symfony\Component\Routing\Attribute\Route;
    
    #[AsController]
    class Controller
    {
        public function __construct(
            private readonly Manager $manager,
        ) {
        }
    
        #[Route(path: 'api/v1/add-followers/{id}', requirements: ['id' => '\d+'], methods: ['POST'])]
        public function __invoke(#[MapEntity(id: 'id')] User $author, #[MapRequestPayload] AddFollowersDTO $addFollowersDTO): Response
        {
            return new JsonResponse(['count' => $this->manager->addFollowers($author, $addFollowersDTO)]);
        }
    }
    ```
7. Сбрасываем кэш метаданных Doctrine командой `php bin/console doctrine:cache:clear-metadata`
8. Выполняем запрос Add user v2 из Postman-коллекции v8 для добавления автора.
9. Выполняем запрос Add followers из Postman-коллекции v8, чтобы добавить этому автору 30 подписчиков.

## Переходим на асинхронное взаимодействие

1. Удаляем директорию `src/Consumer`
2. В файл `config/packages/old_sound_rabbit_mq.yaml` добавляем описание продюсера и консьюмера
    ```yaml
    producers:
     add_followers:
       connection: default
       exchange_options: {name: 'old_sound_rabbit_mq.add_followers', type: direct}
    
    consumers:
     add_followers:
       connection: default
       exchange_options: {name: 'old_sound_rabbit_mq.add_followers', type: direct}
       queue_options: {name: 'old_sound_rabbit_mq.consumer.add_followers'}
       callback: App\Controller\Amqp\AddFollowers\Consumer
       idle_timeout: 300
       idle_timeout_exit_code: 0
       graceful_max_execution:
         timeout: 1800
         exit_code: 0
       qos_options: {prefetch_size: 0, prefetch_count: 1, global: false}
    ```
3. Добавляем классс `App\Application\RabbitMq\AbstractConsumer`
    ```php
    <?php
    
    namespace App\Application\RabbitMq;
    
    use Doctrine\ORM\EntityManagerInterface;
    use OldSound\RabbitMqBundle\RabbitMq\ConsumerInterface;
    use PhpAmqpLib\Message\AMQPMessage;
    use Symfony\Component\Serializer\Exception\UnsupportedFormatException;
    use Symfony\Component\Serializer\SerializerInterface;
    use Symfony\Component\Validator\Validator\ValidatorInterface;
    use Symfony\Contracts\Service\Attribute\Required;
    
    abstract class AbstractConsumer implements ConsumerInterface
    {
        private readonly EntityManagerInterface $entityManager;
        private readonly ValidatorInterface $validator;
        private readonly SerializerInterface $serializer;
    
        abstract protected function getMessageClass(): string;
    
        abstract protected function handle($message): int;
        
        #[Required]
        public function setEntityManager(EntityManagerInterface $entityManager): void
        {
            $this->entityManager = $entityManager;
        }
    
        #[Required]
        public function setValidator(ValidatorInterface $validator): void
        {
            $this->validator = $validator;
        }
    
        #[Required]
        public function setSerializer(SerializerInterface $serializer): void
        {
            $this->serializer = $serializer;
        }
    
        public function execute(AMQPMessage $msg): int
        {
            try {
                $message = $this->serializer->deserialize($msg->getBody(), $this->getMessageClass(), 'json');
                $errors = $this->validator->validate($message);
                if ($errors->count() > 0) {
                    return $this->reject((string)$errors);
                }
    
                return $this->handle($message);
            } catch (UnsupportedFormatException $e) {
                return $this->reject($e->getMessage());
            } finally {
                $this->entityManager->clear();
                $this->entityManager->getConnection()->close();
            }
        }
    
        protected function reject(string $error): int
        {
            echo "Incorrect message: $error";
    
            return self::MSG_REJECT;
        }
    }
    ```
4. Добавляем класс `App\Controller\Amqp\AddFollowers\Input\Message`
   ```php
   <?php
    
   namespace App\Controller\Amqp\AddFollowers\Input;
    
   use Symfony\Component\Validator\Constraints as Assert;
    
   class Message
   {
       public function __construct(
           #[Assert\Type('numeric')]
           public readonly int $userId,
           #[Assert\Type('string')]
           #[Assert\Length(max: 32)]
           public readonly string $followerLogin,
           #[Assert\Type('numeric')]
           public readonly int $count,
       ) {
       }
   }
   ```
5. Добавляем класс `App\Controller\Amqp\AddFollowers\Consumer`
    ```php
    <?php
    
    namespace App\Controller\Amqp\AddFollowers;
    
    use App\Application\RabbitMq\AbstractConsumer;
    use App\Controller\Amqp\AddFollowers\Input\Message;
    use App\Domain\Entity\User;
    use App\Domain\Service\FollowerService;
    use App\Domain\Service\UserService;
    
    class Consumer extends AbstractConsumer
    {
        public function __construct(
            private readonly UserService $userService,
            private readonly FollowerService $followerService,
        ) {
        }
    
        protected function getMessageClass(): string
        {
            return Message::class;
        }
    
        /**
         * @param Message $message
         */
        protected function handle($message): int
        {
            $user = $this->userService->findUserById($message->userId);
            if (!($user instanceof User)) {
                return $this->reject(sprintf('User ID %s was not found', $message->userId));
            }
    
            $this->followerService->addFollowersSync($user, $message->followerLogin, $message->count);
            
            return self::MSG_ACK;
        }
    }
    ```
6. Добавляем перечисление `App\Infrastructure\Bus\AmqpQueueEnum`
    ```php
    <?php
    
    namespace App\Infrastructure\Bus;
    
    enum AmqpQueueEnum: string
    {
        case AddFollowers = 'add_followers';
    }
    ```
7. Добавляем класс `App\Infrastructure\Bus\RabbitMqBus`
    ```php
    <?php
    
    namespace App\Infrastructure\Bus;
    
    use OldSound\RabbitMqBundle\RabbitMq\ProducerInterface;
    use Symfony\Component\Serializer\Normalizer\AbstractObjectNormalizer;
    use Symfony\Component\Serializer\SerializerInterface;
    
    class RabbitMqBus
    {
        /** @var array<string,ProducerInterface> */
        private array $producers;
    
        public function __construct(private readonly SerializerInterface $serializer)
        {
            $this->producers = [];
        }
    
        public function registerProducer(AmqpExchangeEnum $exchange, ProducerInterface $producer): void
        {
            $this->producers[$exchange->value] = $producer;
        }
    
        public function publishToExchange(AmqpExchangeEnum $exchange, $message, ?string $routingKey = null, ?array $additionalProperties = null): bool
        {
            $serializedMessage = $this->serializer->serialize($message, 'json', [AbstractObjectNormalizer::SKIP_NULL_VALUES => true]);
            if (isset($this->producers[$exchange->value])) {
                $this->producers[$exchange->value]->publish($serializedMessage, $routingKey ?? '', $additionalProperties ?? []);
    
                return true;
            }
    
            return false;
        }
    }
    ```
8. Добавляем класс `App\Domain\DTO\AddFollowersDTO`
    ```php
    <?php
    
    namespace App\Domain\DTO;
    
    class AddFollowersMessage
    {
        public function __construct(
            public readonly int $userId,
            public readonly string $followerLogin,
            public readonly int $count
        ) {
        }
    }
     ```
9. Исправляем класс `App\Controller\Web\AddFollowers\v1\Input\AddFollowersDTO`
    ```php
    <?php
    
    namespace App\Controller\Web\AddFollowers\v1\Input;
    
    class AddFollowersDTO
    {
        public function __construct(
            public readonly string $followerLoginPrefix,
            public readonly int $count,
            public readonly bool $async = false,
        ) {
        }
    }    
    ```
10. Добавляем интерфейс `App\Domain\Bus\AddFollowersBusInterface`
    ```php
    <?php
    
    namespace App\Domain\Bus;
    
    use App\Domain\DTO\AddFollowersDTO;
    
    interface AddFollowersBusInterface
    {
        public function sendAddFollowersMessage(AddFollowersDTO $addFollowersDTO);
    }
    ```
11. Добавляем класс `App\Infrastructure\Bus\Adapter\AddFollowersRabbitMqBus`
12. Исправляем класс `App\Domain\Service\FollowerService`
    ```php
    <?php
    
    namespace App\Infrastructure\Bus\Adapter;
    
    use App\Domain\Bus\AddFollowersBusInterface;
    use App\Domain\DTO\AddFollowersDTO;
    use App\Infrastructure\Bus\AmqpExchangeEnum;
    use App\Infrastructure\Bus\RabbitMqBus;
    
    class AddFollowersRabbitMqBus implements AddFollowersBusInterface
    {
        public function __construct(private readonly RabbitMqBus $rabbitMqBus)
        {
        }
    
        public function sendAddFollowersMessage(AddFollowersDTO $addFollowersDTO): bool
        {
            return $this->rabbitMqBus->publishToExchange(AmqpExchangeEnum::AddFollowers, $addFollowersDTO);
        }
    }
    ```
13. Исправляем класс `App\Domain\Service\FollowerService`
    ```php
    <?php
    
    namespace App\Domain\Service;
    
    use App\Domain\Bus\AddFollowersBusInterface;
    use App\Domain\DTO\AddFollowersDTO;
    use App\Domain\Entity\User;
    use App\Domain\Model\CreateUserModel;
    use App\Domain\ValueObject\CommunicationChannelEnum;
    
    class FollowerService
    {
        public function __construct(
            private readonly UserService $userService,
            private readonly SubscriptionService $subscriptionService,
            private readonly AddFollowersBusInterface $addFollowersBus,
        ) {
    
        }
    
        public function addFollowersSync(User $user, string $followerLoginPrefix, int $count): int
        {
            $createdFollowers = 0;
            for ($i = 0; $i < $count; $i++) {
                $login = "{$followerLoginPrefix}_$i";
                $channel = random_int(0, 2) === 1 ? CommunicationChannelEnum::Email : CommunicationChannelEnum::Phone;
                $model = new CreateUserModel(
                    $login,
                    match ($channel) {
                        CommunicationChannelEnum::Email => "{$login}@mail.ru",
                        CommunicationChannelEnum::Phone => '+'.str_pad((string)abs(crc32($login)), 10, '0'),
                    },
                    $channel,
                    "{$login}_password",
                );
                $follower = $this->userService->create($model);
                $this->subscriptionService->addSubscription($user, $follower);
                $createdFollowers++;
            }
    
            return $createdFollowers;
        }
    
        public function addFollowersAsync(AddFollowersDTO $addFollowersDTO): int
        {
            return $this->addFollowersBus->sendAddFollowersMessage($addFollowersDTO) ? $addFollowersDTO->count : 0;
        }
    }
    ```
14. Исправляем класс `App\Controller\Web\AddFollowers\v1\Manager`
    ```php
    <?php
    
    namespace App\Controller\Web\AddFollowers\v1;
    
    use App\Controller\Web\AddFollowers\v1\Input\AddFollowersDTO;
    use App\Domain\DTO\AddFollowersDTO as InternalAddFollowersDTO;
    use App\Domain\Entity\User;
    use App\Domain\Service\FollowerService;
    
    class Manager
    {
        public function __construct(private readonly FollowerService $followerService)
        {
        }
    
        public function addFollowers(User $author, AddFollowersDTO $addFollowersDTO): int
        {
            return $addFollowersDTO->async ?
                $this->followerService->addFollowersAsync(
                    new InternalAddFollowersDTO(
                        $author->getId(),
                        $addFollowersDTO->followerLoginPrefix,
                        $addFollowersDTO->count
                    )
                ) :
                $this->followerService->addFollowersSync(
                    $author,
                    $addFollowersDTO->followerLoginPrefix,
                    $addFollowersDTO->count
                ); 
        }
    }
    ```
15. В файл `config/services.yaml` добавляем новый сервис
     ```yaml
     App\Infrastructure\Bus\RabbitMqBus:
         calls:
             - [ 'registerProducer', [ !php/enum App\Infrastructure\Bus\AmqpExchangeEnum::AddFollowers, '@old_sound_rabbit_mq.add_followers_producer' ] ]
     ```
16. Запускаем консьюмер командой `php bin/console rabbitmq:consumer add_followers -m 100`
17. Выполняем запрос Add followers из Postman-коллекции v8 с параметром `async` = 1, видим в интерфейсе RabbitMQ
    пришедшее сообщение и то, что в БД добавились подписчики

## Эмулируем многократную доставку

1. В классе `App\Controller\Amqp\AddFollowers\Consumer` в методе `handle` безусловно выбрасываем исключение перед
   последней строкой
    ```php
    throw new \RuntimeException('Something happens');
    ```
2. Перезапускаем консьюмер командой `php bin/console rabbitmq:consumer add_followers -m 100`
3. Выполняем запрос Add followers из Postman-коллекции v8 с параметром `async` = 1, видим в интерфейсе RabbitMQ
   пришедшее сообщение и то, что оно не обработалось, хотя в БД добавились подписчики
4. В консоли видим сообщение об ошибке от консьюмера, перезапускаем его командой
   `php bin/console rabbitmq:consumer add_followers -m 100` и видим ошибку уже добавления в БД из-за нарушения
   уникальности логина

## Исправляем проблему многократной доставки

1. В классе `App\Application\RabbitMq\AbstractConsumer` исправляем метод `execute`
    ```php
    <?php
    
    namespace App\Application\RabbitMq;
    
    use Doctrine\ORM\EntityManagerInterface;
    use OldSound\RabbitMqBundle\RabbitMq\ConsumerInterface;
    use PhpAmqpLib\Message\AMQPMessage;
    use Symfony\Component\Serializer\SerializerInterface;
    use Symfony\Component\Validator\Validator\ValidatorInterface;
    use Symfony\Contracts\Service\Attribute\Required;
    use Throwable;
    
    abstract class AbstractConsumer implements ConsumerInterface
    {
        private readonly EntityManagerInterface $entityManager;
        private readonly ValidatorInterface $validator;
        private readonly SerializerInterface $serializer;
    
        abstract protected function getMessageClass(): string;
    
        abstract protected function handle($message): int;
    
        #[Required]
        public function setEntityManager(EntityManagerInterface $entityManager): void
        {
            $this->entityManager = $entityManager;
        }
    
        #[Required]
        public function setValidator(ValidatorInterface $validator): void
        {
            $this->validator = $validator;
        }
    
        #[Required]
        public function setSerializer(SerializerInterface $serializer): void
        {
            $this->serializer = $serializer;
        }
    
        public function execute(AMQPMessage $msg): int
        {
            try {
                $message = $this->serializer->deserialize($msg->getBody(), $this->getMessageClass(), 'json');
                $errors = $this->validator->validate($message);
                if ($errors->count() > 0) {
                    return $this->reject((string)$errors);
                }
    
                return $this->handle($message);
            } catch (Throwable $e) {
                return $this->reject($e->getMessage());
            } finally {
                $this->entityManager->clear();
                $this->entityManager->getConnection()->close();
            }
        }
    
        protected function reject(string $error): int
        {
            echo "Incorrect message: $error";
    
            return self::MSG_REJECT;
        }
    }
    ```
2. Перезапускаем консьюмер из контейнера командой `php bin/console rabbitmq:consumer add_followers -m 100`
3. Видим сообщение об ошибке, но в интерфейсе RabbitMQ сообщение из очереди уходит
4. Останавливаем консьюмер в контейнере

## Эмулируем "убийственную" задачу

1. В классе `App\Controller\Amqp\AddFollowers\Consumer` исправляем метод `handle`
    ```php
    protected function handle($message): int
    {
        $user = $this->userService->findUserById($message->userId);
        if (!($user instanceof User)) {
            return $this->reject(sprintf('User ID %s was not found', $message->userId));
        }
        
        if ($message->count === 5) {
            sleep(1000);
        }

        $this->followerService->addFollowersSync($user, $message->followerLogin, $message->count);

        return self::MSG_ACK;
    }
    ```
2. Выполняем несколько запрос Add followers из Postman-коллекции v8 с параметром `async` = 1 и разными значениями
   `count`: сначала не равными 5, потом 5, потом ещё каким-нибудь не равные 5.
3. Перезапускаем консьюмер из контейнера командой `php bin/console rabbitmq:consumer add_followers -m 100`
4. Видим, что до "убийственной" задачи сообщения разобрались, но затем всё остановилось.
5. Останавливаем консьюмер и запускаем два параллельных консьюмера командой
    ```shell
    php bin/console rabbitmq:consumer add_followers -m 100 &
    php bin/console rabbitmq:consumer add_followers -m 100 &
    ```
6. Видим, что разобрались все сообщения, кроме "убийственной" задачи
7. Останавливаем консьюмеры командой `kill` c PID процессов консьюмеров и делаем Purge messages из очереди

## Работа с большим количеством сообщений за раз

1. В класс `App\Infrastructure\Bus\RabbitMqBus` добавляем метод `publishMultipleToExchange`
    ```php
    public function publishMultipleToExchange(AmqpExchangeEnum $exchange, array $messages, ?string $routingKey = null, ?array $additionalProperties = null): bool
    {
        $sentCount = 0;
        foreach ($messages as $message) {
            if ($this->publishToExchange($exchange, $message, $routingKey, $additionalProperties)) {
                $sentCount++;
            }
        }

        return $sentCount;
    }
    ```
2. В классе `App\Infrastructure\Bus\Adapter\AddFollowersRabbitMqBus` исправляем метод `sendAddFollowersMessage`
    ```php
    public function sendAddFollowersMessage(AddFollowersDTO $addFollowersDTO): bool
    {
        $messages = [];
        for ($i = 0; $i < $addFollowersDTO->count; $i++) {
            $messages[] = new AddFollowersDTO($addFollowersDTO->userId, $addFollowersDTO->followerLogin."_$i", 1);
        }

        return $this->rabbitMqBus->publishMultipleToExchange(AmqpExchangeEnum::AddFollowers, $messages);
    }
    ```
3. В классе `App\Controller\Amqp\AddFollowers\Consumer` исправляем метод `handle`
    ```php
    protected function handle($message): int
    {
        $user = $this->userService->findUserById($message->userId);
        if (!($user instanceof User)) {
            return $this->reject(sprintf('User ID %s was not found', $message->userId));
        }

        $this->followerService->addFollowersSync($user, $message->followerLogin, $message->count);
        sleep(1);

        return self::MSG_ACK;
    }
    ```
4. В файле `config/packages/old_sound_rabbit_mq.yaml` исправляем значение параметра `consumers.add_followers.qos_options`
    ```yaml
    qos_options: {prefetch_size: 0, prefetch_count: 30, global: false}
    ```
5. Выполняем запрос Add followers из Postman-коллекции v8 с параметром `async` = 1 и значением `count` = 100, видим
   полученные сообщения в интерфейсе RabbitMQ
6. Запускаем два параллельных консьюмера командой
    ```shell
    php bin/console rabbitmq:consumer add_followers -m 100 &
    php bin/console rabbitmq:consumer add_followers -m 100 &
    ```
7. Видим в БД, что консьюмеры забирают по 30 сообщений и обрабатывают их параллельно, т.е. порядок обработки нарушен
8. Останавливаем консьюмеры командой `kill` c PID процессов консьюмеров

## Эмулируем ошибку при работе с prefetch

1. В классе `App\Controller\Amqp\AddFollowers\Consumer` исправляем метод `handle`
    ```php
    protected function handle($message): int
    {
        $user = $this->userService->findUserById($message->userId);
        if (!($user instanceof User)) {
            return $this->reject(sprintf('User ID %s was not found', $message->userId));
        }

        if ($message->followerLogin === 'multi_follower_error_11') {
            die();
        }
        
        $this->followerService->addFollowersSync($user, $message->followerLogin, $message->count);
        sleep(1);

        return self::MSG_ACK;
    }
    ```
2. Выполняем запрос Add followers из Postman-коллекции v8 с параметрами `async` = 1, `followersLogin` =
   `multi_follower_error` и `count` = 100, видим полученные сообщения в интерфейсе RabbitMQ
3. Запускаем два параллельных консьюмера командой
    ```shell
    php bin/console rabbitmq:consumer add_followers -m 100 &
    php bin/console rabbitmq:consumer add_followers -m 100 &
    ```
4. Видим в БД, что после падения одного из консьюмеров порядок обработки вторым становится совсем нелогичным, и затем
   он тоже падает на том же сообщении, которое вернулось в очередь
5. Делаем Purge messages из очереди
6. В классе `App\Consumer\AddFollowers\Consumer` в методе `execute` изменяем проверяемый логин на
   `multi_follower_error2_11`
7. Выполняем запрос Add followers из Postman-коллекции v8 с параметрами `async` = 1, `followersLogin` =
   `multi_follower_error2` и `count` = 100, видим полученные сообщения в интерфейсе RabbitMQ
8. Запускаем консьюмер командой `php bin/console rabbitmq:consumer add_followers -m 100`
9. После падения перезапускаем консьюмер и видим, что он сразу же падает





# Очереди: расширенные возможности

## Добавляем supervisor

1. Добавляем файл `docker\supervisor\Dockerfile`
    ```dockerfile
    FROM php:8.3-cli-alpine
    
    # Install dev dependencies
    RUN apk update \
        && apk upgrade --available \
        && apk add --virtual build-deps \
            autoconf \
            build-base \
            icu-dev \
            libevent-dev \
            openssl-dev \
            zlib-dev \
            libzip \
            libzip-dev \
            zlib \
            zlib-dev \
            bzip2 \
            git \
            libpng \
            libpng-dev \
            libjpeg \
            libjpeg-turbo-dev \
            libwebp-dev \
            freetype \
            freetype-dev \
            postgresql-dev \
            linux-headers \
            libmemcached-dev \
            curl \
            wget \
            bash
        
    # Install Composer
    RUN curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/bin --filename=composer
    
    # Install PHP extensions
    RUN docker-php-ext-configure gd --with-freetype=/usr/include/ --with-jpeg=/usr/include/
    RUN docker-php-ext-install -j$(getconf _NPROCESSORS_ONLN) \
        intl \
        gd \
        bcmath \
        pcntl \
        pdo_pgsql \
        sockets \
        zip
    RUN pecl channel-update pecl.php.net \
        && pecl install -o -f \
            memcached \
            redis \
            event \
        && rm -rf /tmp/pear \
        && echo "extension=redis.so" > /usr/local/etc/php/conf.d/redis.ini \
        && echo "extension=event.so" > /usr/local/etc/php/conf.d/event.ini \
        && echo "extension=memcached.so" > /usr/local/etc/php/conf.d/memcached.ini
    
    RUN apk add supervisor && mkdir /var/log/supervisor
    ```
2. Добавляем файл `docker\supervisor\supervisord.conf`
    ```ini
    [supervisord]
    logfile=/var/log/supervisor/supervisord.log
    pidfile=/var/run/supervisord.pid
    nodaemon=true
    
    [include]
    files=/app/supervisor/*.conf
    ```
3. Добавляем в `docker-compose.yml` сервис
    ```yaml
    supervisor:
       build: docker/supervisor
       container_name: 'supervisor'
       volumes:
           - ./:/app
           - ./docker/supervisor/supervisord.conf:/etc/supervisor/supervisord.conf
       working_dir: /app
       command: ["supervisord", "-c", "/etc/supervisor/supervisord.conf"]
    ```
4. Добавляем конфигурацию для запуска консьюмеров в файле `supervisor/consumer.conf`
    ```ini
    [program:add_followers]
    command=php -dmemory_limit=1G /app/bin/console rabbitmq:consumer -m 100 add_followers --env=dev -vv
    process_name=add_follower_%(process_num)02d
    numprocs=1
    directory=/tmp
    autostart=true
    autorestart=true
    startsecs=3
    startretries=10
    user=www-data
    redirect_stderr=false
    stdout_logfile=/app/var/log/supervisor.add_followers.out.log
    stdout_capture_maxbytes=1MB
    stderr_logfile=/app/var/log/supervisor.add_followers.error.log
    stderr_capture_maxbytes=1MB
    ```
5. Запускаем контейнеры командой `docker-compose up -d`
6. Проверяем в RabbitMQ, что консьюмер запущен
7. Выполняем запрос Add user v2 из Postman-коллекции v9
8. Выполняем запрос Add followers из Postman-коллекции v9 с параметрами `async` = 1 и `count` = 1000, проверяем, что
   подписчики добавились

## Добавляем функционал ленты и нотификаций

1. Добавляем класс `App\Domain\Entity\Feed`
    ```php
    <?php
    
    namespace App\Domain\Entity;
    
    use DateTime;
    use Doctrine\ORM\Mapping as ORM;
    
    #[ORM\Table(name: 'feed')]
    #[ORM\UniqueConstraint(columns: ['reader_id'])]
    #[ORM\Entity]
    #[ORM\HasLifecycleCallbacks]
    class Feed implements EntityInterface
    {
        #[ORM\Column(name: 'id', type: 'bigint', unique:true)]
        #[ORM\Id]
        #[ORM\GeneratedValue(strategy: 'IDENTITY')]
        private int $id;
    
        #[ORM\ManyToOne(targetEntity: User::class)]
        #[ORM\JoinColumn(name: 'reader_id', referencedColumnName: 'id')]
        private User $reader;
    
        #[ORM\Column(type: 'json', nullable: true)]
        private ?array $tweets;
    
        #[ORM\Column(name: 'created_at', type: 'datetime', nullable: false)]
        private DateTime $createdAt;
    
        #[ORM\Column(name: 'updated_at', type: 'datetime', nullable: false)]
        private DateTime $updatedAt;
    
        public function getId(): int
        {
            return $this->id;
        }
    
        public function setId(int $id): void
        {
            $this->id = $id;
        }
    
        public function getReader(): User
        {
            return $this->reader;
        }
    
        public function setReader(User $reader): void
        {
            $this->reader = $reader;
        }
    
        public function getTweets(): ?array
        {
            return $this->tweets;
        }
    
        public function setTweets(?array $tweets): void
        {
            $this->tweets = $tweets;
        }
    
        public function getCreatedAt(): DateTime {
            return $this->createdAt;
        }
    
        #[ORM\PrePersist]
        public function setCreatedAt(): void {
            $this->createdAt = new DateTime();
        }
    
        public function getUpdatedAt(): DateTime {
            return $this->updatedAt;
        }
    
        #[ORM\PrePersist]
        #[ORM\PreUpdate]
        public function setUpdatedAt(): void {
            $this->updatedAt = new DateTime();
        }
    }
    ```
2. Добавляем класс `App\Domain\Entity\EmailNotification`
    ```php
    <?php
    
    namespace App\Domain\Entity;
    
    use DateTime;
    use Doctrine\ORM\Mapping as ORM;
    
    #[ORM\Table(name: 'email_notification')]
    #[ORM\Entity]
    #[ORM\HasLifecycleCallbacks]
    class EmailNotification implements EntityInterface
    {
        #[ORM\Column(name: 'id', type: 'bigint', unique:true)]
        #[ORM\Id]
        #[ORM\GeneratedValue(strategy: 'IDENTITY')]
        private int $id;
    
        #[ORM\Column(type: 'string', length: 128, nullable: false)]
        private string $email;
    
        #[ORM\Column(type: 'string', length: 512, nullable: false)]
        private string $text;
    
        #[ORM\Column(name: 'created_at', type: 'datetime', nullable: false)]
        private DateTime $createdAt;
    
        #[ORM\Column(name: 'updated_at', type: 'datetime', nullable: false)]
        private DateTime $updatedAt;
    
        public function getId(): int
        {
            return $this->id;
        }
    
        public function setId(int $id): void
        {
            $this->id = $id;
        }
    
        public function getEmail(): string
        {
            return $this->email;
        }
    
        public function setEmail(string $email): void
        {
            $this->email = $email;
        }
    
        public function getText(): string
        {
            return $this->text;
        }
    
        public function setText(string $text): void
        {
            $this->text = $text;
        }
    
        public function getCreatedAt(): DateTime {
            return $this->createdAt;
        }
    
        #[ORM\PrePersist]
        public function setCreatedAt(): void {
            $this->createdAt = new DateTime();
        }
    
        public function getUpdatedAt(): DateTime {
            return $this->updatedAt;
        }
    
        #[ORM\PrePersist]
        #[ORM\PreUpdate]
        public function setUpdatedAt(): void {
            $this->updatedAt = new DateTime();
        }
    }
    ```
3. Добавляем класс `App\Domain\Entity\SmsNotification`
    ```php
    <?php
    
    namespace App\Domain\Entity;
    
    use DateTime;
    use Doctrine\ORM\Mapping as ORM;
    
    #[ORM\Table(name: 'sms_notification')]
    #[ORM\Entity]
    #[ORM\HasLifecycleCallbacks]
    class SmsNotification implements EntityInterface
    {
        #[ORM\Column(name: 'id', type: 'bigint', unique:true)]
        #[ORM\Id]
        #[ORM\GeneratedValue(strategy: 'IDENTITY')]
        private int $id;
    
        #[ORM\Column(type: 'string', length: 11, nullable: false)]
        private string $phone;
    
        #[ORM\Column(type: 'string', length: 60, nullable: false)]
        private string $text;
    
        #[ORM\Column(name: 'created_at', type: 'datetime', nullable: false)]
        private DateTime $createdAt;
    
        #[ORM\Column(name: 'updated_at', type: 'datetime', nullable: false)]
        private DateTime $updatedAt;
    
        public function getId(): int
        {
            return $this->id;
        }
    
        public function setId(int $id): void
        {
            $this->id = $id;
        }
    
        public function getPhone(): string
        {
            return $this->phone;
        }
    
        public function setPhone(string $phone): void
        {
            $this->phone = $phone;
        }
    
        public function getText(): string
        {
            return $this->text;
        }
    
        public function setText(string $text): void
        {
            $this->text = $text;
        }
    
        public function getCreatedAt(): DateTime {
            return $this->createdAt;
        }
    
        #[ORM\PrePersist]
        public function setCreatedAt(): void {
            $this->createdAt = new DateTime();
        }
    
        public function getUpdatedAt(): DateTime {
            return $this->updatedAt;
        }
    
        #[ORM\PrePersist]
        #[ORM\PreUpdate]
        public function setUpdatedAt(): void {
            $this->updatedAt = new DateTime();
        }
    }
    ```
4. Заходим в контейнер командой `docker exec -it php sh`. Дальнейшие команды выполняем из контейнера
5. Создаём миграцию и применяем её командами
    ```shell
    php bin/console doctrine:migrations:diff
    php bin/console doctrine:migrations:migrate
    ```
6. Исправляем класс `App\Domain\Model\TweetModel`
     ```php
     <?php
     
     namespace App\Domain\Model;
     
     use DateTime;
     
     class TweetModel
     {
         public function __construct(
             public readonly int $id,
             public readonly string $author,
             public readonly int $authorId,
             public readonly string $text,
             public readonly DateTime $createdAt,
         ) {
         }
     
         public function toFeed(): array
         {
             return [
                 'id' => $this->id,
                 'author' => $this->author,
                 'text' => $this->text,
                 'createdAt' => $this->createdAt->format('Y-m-d H:i:s'),
             ];
         }
     }
     ```
7. В классе `App\Infrastructure\Repository\TweetRepositoryCacheDecorator` добавляем заполнение нового поля в
   `TweetModel`
8. Исправляем перечисление `App\Infrastructure\Bus\AmqpExchangeEnum`
    ```php
    <?php
    
    namespace App\Infrastructure\Bus;
    
    enum AmqpExchangeEnum: string
    {
        case AddFollowers = 'add_followers';
        case PublishTweet = 'publish_tweet';
        case SendNotification = 'send_notification';
    }
    ```
9. Добавляем интерфейс `App\Domain\Bus\PublishTweetBusInterface`
    ```php
    <?php
    
    namespace App\Domain\Bus;
    
    use App\Domain\Model\TweetModel;
    
    interface PublishTweetBusInterface
    {
        public function sendPublishTweetMessage(TweetModel $tweetModel): bool;
    }
    ```
10. Добавляем класс `App\Infrastructure\Bus\Adapter\PublishTweetRabbitMqBus`
     ```php
     <?php
    
     namespace App\Infrastructure\Bus\Adapter;
    
     use App\Domain\Bus\PublishTweetBusInterface;
     use App\Domain\Model\TweetModel;
     use App\Infrastructure\Bus\AmqpExchangeEnum;
     use App\Infrastructure\Bus\RabbitMqBus;
    
     class PublishTweetRabbitMqBus implements PublishTweetBusInterface
     {
         public function __construct(private readonly RabbitMqBus $rabbitMqBus)
         {
         }
    
         public function sendPublishTweetMessage(TweetModel $tweetModel): bool
         {
             return $this->rabbitMqBus->publishToExchange(AmqpExchangeEnum::PublishTweet, $tweetModel);
         }
     }
     ```
11. В классе `App\Infrastructure\Repository\SubscriptionRepository` добавляем метод `findAllByAuthor`
    ```php
    /**
     * @return Subscription[]
     */
    public function findAllByAuthor(User $author): array
    {
        $subscriptionRepository = $this->entityManager->getRepository(Subscription::class);
        return $subscriptionRepository->findBy(['author' => $author]) ?? [];
    }
    ```
12. Добавляем класс `App\Infrastructure\Repository\FeedRepository`
     ```php
     <?php
    
     namespace App\Infrastructure\Repository;
    
     use App\Domain\Entity\Feed;
     use App\Domain\Entity\User;
     use App\Domain\Model\TweetModel;
    
     class FeedRepository extends AbstractRepository
     {
         public function putTweetToReaderFeed(TweetModel $tweet, User $reader): bool
         {
             $feed = $this->ensureFeedForReader($reader);
             if ($feed === null) {
                 return false;
             }
             $tweets = $feed->getTweets();
             $tweets[] = $tweet->toFeed();
             $feed->setTweets($tweets);
             $this->flush();
    
             return true;
         }
    
         public function ensureFeedForReader(User $reader): ?Feed
         {
             $feedRepository = $this->entityManager->getRepository(Feed::class);
             $feed = $feedRepository->findOneBy(['reader' => $reader]);
             if (!($feed instanceof Feed)) {
                 $feed = new Feed();
                 $feed->setReader($reader);
                 $feed->setTweets([]);
                 $this->store($feed);
             }
    
             return $feed;
         }
     }
     ```
13. Исправляем класс `App\Domain\Service\SubscriptionService`
     ```php
     <?php
    
     namespace App\Domain\Service;
    
     use App\Domain\Entity\Subscription;
     use App\Domain\Entity\User;
     use App\Infrastructure\Repository\SubscriptionRepository;
     use App\Infrastructure\Repository\UserRepository;
    
     class SubscriptionService
     {
         public function __construct(
             private readonly UserRepository $userRepository,
             private readonly SubscriptionRepository $subscriptionRepository,
         ) {
         }
    
         public function addSubscription(User $author, User $follower): void
         {
             $subscription = new Subscription();
             $subscription->setAuthor($author);
             $subscription->setFollower($follower);
             $author->addSubscriptionFollower($subscription);
             $follower->addSubscriptionAuthor($subscription);
             $this->subscriptionRepository->create($subscription);
         }
    
         /**
          * @return User[]
          */
         public function getFollowers(int $authorId): array
         {
             $subscriptions = $this->getSubscriptionsByAuthorId($authorId);
             $mapper = static function(Subscription $subscription) {
                 return $subscription->getFollower();
             };
    
             return array_map($mapper, $subscriptions);
         }
    
         /**
          * @return Subscription[]
          */
         private function getSubscriptionsByAuthorId(int $authorId): array
         {
             $author = $this->userRepository->find($authorId);
             if (!($author instanceof User)) {
                 return [];
             }
            
             return $this->subscriptionRepository->findAllByAuthor($author);
         }
     }
     ```
14. Добавляем класс `App\Domain\Service\FeedService`
    ```php
    <?php
    
    namespace App\Domain\Service;
    
    use App\Domain\Bus\PublishTweetBusInterface;
    use App\Domain\Entity\User;
    use App\Domain\Model\TweetModel;
    use App\Infrastructure\Repository\FeedRepository;
    
    class FeedService
    {
        public function __construct(
            private readonly FeedRepository $feedRepository,
            private readonly SubscriptionService $subscriptionService,
            private readonly PublishTweetBusInterface $publishTweetBus,
        ) {
        }
    
        public function ensureFeed(User $user, int $count): array
        {
            $feed = $this->feedRepository->ensureFeedForReader($user);
    
            return $feed === null ? [] : array_slice($feed->getTweets(), -$count);
        }
    
        public function spreadTweetAsync(TweetModel $tweet): void
        {
            $this->publishTweetBus->sendPublishTweetMessage($tweet);
        }
    
        public function spreadTweetSync(TweetModel $tweet): void
        {
            $followers = $this->subscriptionService->getFollowers($tweet->authorId);
    
            foreach ($followers as $follower) {
                $this->feedRepository->putTweetToReaderFeed($tweet, $follower);
            }
        }
    }
    ```
15. Исправляем класс `App\Domain\Service\TweetService`
    ```php
    <?php
    
    namespace App\Domain\Service;
    
    use App\Domain\Entity\Tweet;
    use App\Domain\Entity\User;
    use App\Domain\Model\TweetModel;
    use App\Domain\Repository\TweetRepositoryInterface;
    
    class TweetService
    {
        public function __construct(
            private readonly TweetRepositoryInterface $tweetRepository,
            private readonly FeedService $feedService,
        ) {
        }
    
        public function postTweet(User $author, string $text, bool $async): void
        {
            $tweet = new Tweet();
            $tweet->setAuthor($author);
            $tweet->setText($text);
            $author->addTweet($tweet);
            $this->tweetRepository->create($tweet);
            $tweetModel = new TweetModel(
                $tweet->getId(),
                $tweet->getAuthor()->getLogin(),
                $tweet->getAuthor()->getId(),
                $tweet->getText(),
                $tweet->getCreatedAt()
            );
            if ($async) {
                $this->feedService->spreadTweetAsync($tweetModel);
            } else {
                $this->feedService->spreadTweetSync($tweetModel);
            }
        }
    
        /**
         * @return TweetModel[]
         */
        public function getTweetsPaginated(int $page, int $perPage): array
        {
            return $this->tweetRepository->getTweetsPaginated($page, $perPage);
        }
    }
    ```
16. Исправляем класс `App\Controller\Web\PostTweet\v1\PostTweetDTO`
    ```php
    <?php
    
    namespace App\Controller\Web\PostTweet\v1\Input;
    
    class PostTweetDTO
    {
        public function __construct(
            public readonly int $userId,
            public readonly string $text,
            public readonly bool $async = false,
        ) {
        }
    }
    ```
17. В классе `App\Controller\Web\PostTweet\v1\Manager` исправляем метод `postTweet`
    ```php
    public function postTweet(PostTweetDTO $tweetDTO): bool
    {
        $user = $this->userService->findUserById($tweetDTO->userId);

        if ($user === null) {
            return false;
        }

        $this->tweetService->postTweet($user, $tweetDTO->text, $tweetDTO->async);

        return true;
    }
    ```
18. Добавляем класс `App\Controller\Web\GetFeed\v1\Manager`
    ```php
    <?php
    
    namespace App\Controller\Web\GetFeed\v1;
    
    use App\Domain\Entity\User;
    use App\Domain\Service\FeedService;
    
    class Manager
    {
        private const DEFAULT_FEED_SIZE = 20;
        
        public function __construct(private readonly FeedService $feedService)
        {
        }
        
        public function getFeed(User $user, ?int $count = null): array
        {
            return $this->feedService->ensureFeed($user, $count ?? self::DEFAULT_FEED_SIZE);
        }
    }
    ```
19. Добавляем класс `App\Controller\Web\GetFeed\v1\Controller`
    ```php
    <?php
    
    namespace App\Controller\Web\GetFeed\v1;
    
    use App\Domain\Entity\User;
    use Symfony\Bridge\Doctrine\Attribute\MapEntity;
    use Symfony\Component\HttpFoundation\JsonResponse;
    use Symfony\Component\HttpFoundation\Response;
    use Symfony\Component\HttpKernel\Attribute\AsController;
    use Symfony\Component\HttpKernel\Attribute\MapQueryParameter;
    use Symfony\Component\Routing\Attribute\Route;
    
    #[AsController]
    class Controller
    {
        public function __construct(private readonly Manager $manager) {
        }
    
        #[Route(path: 'api/v1/get-feed/{id}', methods: ['GET'])]
        public function __invoke(#[MapEntity(id: 'id')]User $user, #[MapQueryParameter]?int $count = null): Response
        {
            return new JsonResponse(['tweets' => $this->manager->getFeed($user, $count)]);
        }
    }
    ```
20. В классе `App\Controller\Amqp\AddFollowers\Consumer` в методе `execute` убираем `sleep` и запланированную ошибку
21. Выполняем запрос Post tweet из Postman-коллекции v9 с параметром `async` = 0, проверяем, что ленты материализовались

## Добавляем консьюмеры

1. Добавляем класс `App\Infrastructure\Repository\EmailNotificationRepository`
    ```php
    <?php
    
    namespace App\Infrastructure\Repository;
    
    use App\Domain\Entity\EmailNotification;
    
    class EmailNotificationRepository extends AbstractRepository
    {
        public function create(EmailNotification $notification): int
        {
            return $this->store($notification);
        }
    }
    ```
2. Добавляем класс `App\Domain\Service\EmailNotificationService`
    ```php
    <?php
    
    namespace App\Domain\Service;
    
    use App\Domain\Entity\EmailNotification;
    use App\Infrastructure\Repository\EmailNotificationRepository;
    
    class EmailNotificationService
    {
        public function __construct(private readonly EmailNotificationRepository $emailNotificationRepository)
        {
        }
    
        public function saveEmailNotification(string $email, string $text): void {
            $emailNotification = new EmailNotification();
            $emailNotification->setEmail($email);
            $emailNotification->setText($text);
            $this->emailNotificationRepository->create($emailNotification);
        }
    }
    ```
3. Добавляем класс `App\Infrastructure\Repository\SmsNotificationRepository`
    ```php
    <?php
    
    namespace App\Infrastructure\Repository;
    
    use App\Domain\Entity\SmsNotification;
    
    class SmsNotificationRepository extends AbstractRepository
    {
        public function create(SmsNotification $notification): int
        {
            return $this->store($notification);
        }
    }
    ```
4. Добавляем класс `App\Domain\Service\PhoneNotificationService`
    ```php
    <?php
    
    namespace App\Domain\Service;
    
    use App\Domain\Entity\SmsNotification;
    use App\Infrastructure\Repository\SmsNotificationRepository;
    
    class SmsNotificationService
    {
        public function __construct(private readonly SmsNotificationRepository $emailNotificationRepository)
        {
        }
    
        public function saveSmsNotification(string $phone, string $text): void {
            $emailNotification = new SmsNotification();
            $emailNotification->setPhone($phone);
            $emailNotification->setText($text);
            $this->emailNotificationRepository->create($emailNotification);
        }
    }
    ```
5. Добавляем класс `App\Controller\Amqp\SendEmailNotification\Input\Message`
    ```php
    <?php
    
    namespace App\Controller\Amqp\SendEmailNotification\Input;
    
    use Symfony\Component\Validator\Constraints as Assert;
    
    class Message
    {
        public function __construct(
            #[Assert\Type('numeric')]
            public readonly int $userId,
            #[Assert\Type('string')]
            #[Assert\Length(max: 512)]
            public readonly string $text,
        ) {
        }
    }
    ```
6. Добавляем класс `App\Controller\Amqp\SendEmailNotification\Consumer`
    ```php
    <?php
    
    namespace App\Controller\Amqp\SendEmailNotification;
    
    use App\Application\RabbitMq\AbstractConsumer;
    use App\Controller\Amqp\SendEmailNotification\Input\Message;
    use App\Domain\Entity\EmailUser;
    use App\Domain\Service\EmailNotificationService;
    use App\Domain\Service\UserService;
    
    class Consumer extends AbstractConsumer
    {
        public function __construct(
            private readonly UserService $userService,
            private readonly EmailNotificationService $emailNotificationService,
        ) {
        }
    
        protected function getMessageClass(): string
        {
            return Message::class;
        }
    
        /**
         * @param Message $message
         */
        protected function handle($message): int
        {
            $user = $this->userService->findUserById($message->userId);
            if (!($user instanceof EmailUser)) {
                return $this->reject(sprintf('User ID %s was not found or does not use email', $message->userId));
            }
            
            $this->emailNotificationService->saveEmailNotification($user->getEmail(), $message->text);
    
            return self::MSG_ACK;
        }
    }
    ```
7. Добавляем класс `App\Controller\Amqp\SendSmsNotification\Input\Message`
    ```php
    <?php
    
    namespace App\Controller\Amqp\SendSmsNotification\Input;
    
    use Symfony\Component\Validator\Constraints as Assert;
    
    class Message
    {
        public function __construct(
            #[Assert\Type('numeric')]
            public readonly int $userId,
            #[Assert\Type('string')]
            #[Assert\Length(max: 60)]
            public readonly string $text,
        ) {
        }
    }
    ```
8. Добавляем класс `App\Controller\Amqp\SendSmsNotification\Consumer`
    ```php
    <?php
    
    namespace App\Controller\Amqp\SendSmsNotification;
    
    use App\Application\RabbitMq\AbstractConsumer;
    use App\Controller\Amqp\SendSmsNotification\Input\Message;
    use App\Domain\Entity\PhoneUser;
    use App\Domain\Service\SmsNotificationService;
    use App\Domain\Service\UserService;
    
    class Consumer extends AbstractConsumer
    {
        public function __construct(
            private readonly UserService $userService,
            private readonly SmsNotificationService $emailNotificationService,
        ) {
        }
    
        protected function getMessageClass(): string
        {
            return Message::class;
        }
    
        /**
         * @param Message $message
         */
        protected function handle($message): int
        {
            $user = $this->userService->findUserById($message->userId);
            if (!($user instanceof PhoneUser)) {
                return $this->reject(sprintf('User ID %s was not found or does not use phone', $message->userId));
            }
    
            $this->emailNotificationService->saveSmsNotification($user->getPhone(), $message->text);
    
            return self::MSG_ACK;
        }
    }
    ```
9. Добавляем класс `App\Domain\DTO\SendNotificationDTO`
    ```php
    <?php
    
    namespace App\Domain\DTO;
    
    use App\Domain\ValueObject\CommunicationChannelEnum;

    class SendNotificationDTO
    {
        public function __construct(
            public readonly int $userId,
            public readonly string $text,
            public readonly CommunicationChannelEnum $channel,
        ) {
        }
    }
    ```
10. Добавляем интерфейс `App\Domain\Bus\SendNotificationBusInterface`
    ```php
    <?php
    
    namespace App\Domain\Bus;
    
    use App\Domain\DTO\SendNotificationDTO;
    
    interface SendNotificationBusInterface
    {
        public function sendNotification(SendNotificationDTO $sendNotificationDTO): bool;
    }
    ```
11. Добавляем класс `App\Instrastructure\Bus\Adapter\SendNotificationRabbitMqBus`
    ```php
    <?php
    
    namespace App\Infrastructure\Bus\Adapter;
    
    use App\Domain\Bus\SendNotificationBusInterface;
    use App\Domain\DTO\SendNotificationDTO;
    use App\Infrastructure\Bus\AmqpExchangeEnum;
    use App\Infrastructure\Bus\RabbitMqBus;
    
    class SendNotificationRabbitMqBus implements SendNotificationBusInterface
    {
        public function __construct(private readonly RabbitMqBus $rabbitMqBus)
        {
        }
    
        public function sendNotification(SendNotificationDTO $sendNotificationDTO): bool
        {
            return $this->rabbitMqBus->publishToExchange(
                AmqpExchangeEnum::SendNotification,
                $sendNotificationDTO,
                $sendNotificationDTO->channel->value
            );
        }
    }
    ```
12. Добавляем класс `App\Controller\Amqp\PublishTweet\Input\Message`
    ```php
    <?php
    
    namespace App\Controller\Amqp\PublishTweet\Input;
    
    use DateTime;
    use Symfony\Component\Validator\Constraints as Assert;
    
    class Message
    {
        public function __construct(
            #[Assert\Type('numeric')]
            public readonly int $id,
            public readonly string $author,
            #[Assert\Type('numeric')]
            public readonly int $authorId,
            public readonly string $text,
            public readonly DateTime $createdAt,
        ) {            
        }
    } 
    ```
13. Добавляем класс `App\Controller\Amqp\PublishTweet\Consumer`
    ```php
    <?php
    
    namespace App\Controller\Amqp\PublishTweet;
    
    use App\Application\RabbitMq\AbstractConsumer;
    use App\Controller\Amqp\PublishTweet\Input\Message;
    use App\Domain\Model\TweetModel;
    use App\Domain\Service\FeedService;
    
    class Consumer extends AbstractConsumer
    {
        public function __construct(
            private readonly FeedService $feedService,
        ) {
        }
    
        protected function getMessageClass(): string
        {
            return Message::class;
        }
    
        /**
         * @param Message $message
         */
        protected function handle($message): int
        {
            $tweet = new TweetModel(
                $message->id,
                $message->author,
                $message->authorId,
                $message->text,
                $message->createdAt,
            );
            $this->feedService->spreadTweetSync($tweet);
    
            return self::MSG_ACK;
        }
    }
    ```
14. Исправляем класс `App\Domain\Service\FeedService`
    ```php
    <?php
    
    namespace App\Domain\Service;
    
    use App\Domain\Bus\PublishTweetBusInterface;
    use App\Domain\Bus\SendNotificationBusInterface;
    use App\Domain\DTO\SendNotificationDTO;
    use App\Domain\Entity\User;
    use App\Domain\Model\TweetModel;
    use App\Infrastructure\Repository\FeedRepository;
    
    class FeedService
    {
        public function __construct(
            private readonly FeedRepository $feedRepository,
            private readonly SubscriptionService $subscriptionService,
            private readonly PublishTweetBusInterface $publishTweetBus,
            private readonly SendNotificationBusInterface $sendNotificationBus,
        ) {
        }
    
        public function ensureFeed(User $user, int $count): array
        {
            $feed = $this->feedRepository->ensureFeedForReader($user);
    
            return $feed === null ? [] : array_slice($feed->getTweets(), -$count);
        }
    
        public function spreadTweetAsync(TweetModel $tweet): void
        {
            $this->publishTweetBus->sendPublishTweetMessage($tweet);
        }
    
        public function spreadTweetSync(TweetModel $tweet): void
        {
            $followers = $this->subscriptionService->getFollowers($tweet->authorId);
    
            foreach ($followers as $follower) {
                $this->feedRepository->putTweetToReaderFeed($tweet, $follower);
                $sendNotificationDTO = new SendNotificationDTO(
                    $follower->getId(),
                    $tweet->text,
                    $follower instanceof EmailUser ? CommunicationChannelEnum::Email : CommunicationChannelEnum::Phone
                );
                $this->sendNotificationBus->sendNotification($sendNotificationDTO);
            }
        }
    }
    ```
15. В файл `config/services.yaml` добавляем к сервису `App\Infrastructure\Bus\RabbitMqBus` регистрацию новых продюсеров:
     ```yaml
     - [ 'registerProducer', [ !php/enum App\Infrastructure\Bus\AmqpExchangeEnum::SendNotification, '@old_sound_rabbit_mq.send_notification_producer' ] ]
     - [ 'registerProducer', [ !php/enum App\Infrastructure\Bus\AmqpExchangeEnum::PublishTweet, '@old_sound_rabbit_mq.publish_tweet_producer' ] ]
     ```
16. Добавляем описание новых продюсеров и консьюмеров в файл `config/packages/old_sound_rabbit_mq.yaml`
   1. в секцию `producers`
       ```yaml
       publish_tweet:
           connection: default
           exchange_options: {name: 'old_sound_rabbit_mq.publish_tweet', type: direct}
       send_notification:
           connection: default
           exchange_options: {name: 'old_sound_rabbit_mq.send_notification', type: topic}
       ```
   2. в секцию `consumers`
       ```yaml
       publish_tweet:
           connection: default
           exchange_options: {name: 'old_sound_rabbit_mq.publish_tweet', type: direct}
           queue_options: {name: 'old_sound_rabbit_mq.consumer.publish_tweet'}
           callback: App\Controller\Amqp\PublishTweet\Consumer
           idle_timeout: 300
           idle_timeout_exit_code: 0
           graceful_max_execution:
               timeout: 1800
               exit_code: 0
           qos_options: {prefetch_size: 0, prefetch_count: 1, global: false}
       send_notification.email:
           connection: default
           exchange_options: {name: 'old_sound_rabbit_mq.send_notification', type: topic}
           queue_options: 
               name: 'old_sound_rabbit_mq.consumer.send_notification.email'
               routing_keys: ['email']
           callback: App\Controller\Amqp\SendEmailNotification\Consumer
           idle_timeout: 300
           idle_timeout_exit_code: 0
           graceful_max_execution:
               timeout: 1800
               exit_code: 0
           qos_options: {prefetch_size: 0, prefetch_count: 1, global: false}
       send_notification.sms:
           connection: default
           exchange_options: {name: 'old_sound_rabbit_mq.send_notification', type: topic}
           queue_options: 
               name: 'old_sound_rabbit_mq.consumer.send_notification.sms'
               routing_keys: ['phone']
           callback: App\Controller\Amqp\SendSmsNotification\Consumer
           idle_timeout: 300
           idle_timeout_exit_code: 0
           graceful_max_execution:
               timeout: 1800
               exit_code: 0
           qos_options: {prefetch_size: 0, prefetch_count: 1, global: false}
       ```
17. Исправляем файл `supervisor/consumer.conf`
    ```ini
    [program:add_followers]
    command=php -dmemory_limit=1G /app/bin/console rabbitmq:consumer -m 100 add_followers --env=dev -vv
    process_name=add_follower_%(process_num)02d
    numprocs=1
    directory=/tmp
    autostart=true
    autorestart=true
    startsecs=3
    startretries=10
    user=www-data
    redirect_stderr=false
    stdout_logfile=/app/var/log/supervisor.add_followers.out.log
    stdout_capture_maxbytes=1MB
    stderr_logfile=/app/var/log/supervisor.add_followers.error.log
    stderr_capture_maxbytes=1MB
   
    [program:publish_tweet]
    command=php -dmemory_limit=1G /app/bin/console rabbitmq:consumer -m 100 publish_tweet --env=dev -vv
    process_name=publish_tweet_%(process_num)02d
    numprocs=1
    directory=/tmp
    autostart=true
    autorestart=true
    startsecs=3
    startretries=10
    user=www-data
    redirect_stderr=false
    stdout_logfile=/app/var/log/supervisor.publish_tweet.out.log
    stdout_capture_maxbytes=1MB
    stderr_logfile=/app/var/log/supervisor.publish_tweet.error.log
    stderr_capture_maxbytes=1MB

    [program:send_notification_email]
    command=php -dmemory_limit=1G /app/bin/console rabbitmq:consumer -m 100 send_notification.email --env=dev -vv
    process_name=send_notification_email_%(process_num)02d
    numprocs=1
    directory=/tmp
    autostart=true
    autorestart=true
    startsecs=3
    startretries=10
    user=www-data
    redirect_stderr=false
    stdout_logfile=/app/var/log/supervisor.send_notification_email.out.log
    stdout_capture_maxbytes=1MB
    stderr_logfile=/app/var/log/supervisor.send_notification_email.error.log
    stderr_capture_maxbytes=1MB
   
    [program:send_notification_sms]
    command=php -dmemory_limit=1G /app/bin/console rabbitmq:consumer -m 100 send_notification.sms --env=dev -vv
    process_name=send_notification_sms_%(process_num)02d
    numprocs=1
    directory=/tmp
    autostart=true
    autorestart=true
    startsecs=3
    startretries=10
    user=www-data
    redirect_stderr=false
    stdout_logfile=/app/var/log/supervisor.send_notification_sms.out.log
    stdout_capture_maxbytes=1MB
    stderr_logfile=/app/var/log/supervisor.send_notification_sms.error.log
    stderr_capture_maxbytes=1MB
    ```
18. Перезапускаем контейнер супервизора командой `docker-compose restart supervisor`
19. Выполняем запрос Post tweet из Postman-коллекции v8 с параметром `async` = 1
20. Видим, что сообщения из точки обмена `old_sound_rabbit_mq.send_notification` распределились по двум очередям
    `old_sound_rabbit_mq.consumer.send_notification.email` и `old_sound_rabbit_mq.consumer.send_notification.sms`

## Добавляем согласованное хэширование

1. Входим в контейнер `rabbit-mq` командой `docker exec -it rabbit-mq sh` и выполняем в нём команду
    ```shell
    rabbitmq-plugins enable rabbitmq_consistent_hash_exchange
    ```
2. Исправляем класс `App\Domain\Service\FeedService`
    ```php
    <?php
    
    namespace App\Domain\Service;
    
    use App\Domain\Bus\PublishTweetBusInterface;
    use App\Domain\Bus\SendNotificationBusInterface;
    use App\Domain\DTO\SendNotificationDTO;
    use App\Domain\Entity\EmailUser;
    use App\Domain\Entity\User;
    use App\Domain\Model\TweetModel;
    use App\Domain\ValueObject\CommunicationChannelEnum;
    use App\Infrastructure\Repository\FeedRepository;
    
    class FeedService
    {
        public function __construct(
            private readonly FeedRepository $feedRepository,
            private readonly SubscriptionService $subscriptionService,
            private readonly PublishTweetBusInterface $publishTweetBus,
            private readonly SendNotificationBusInterface $sendNotificationBus,
        ) {
        }
    
        public function ensureFeed(User $user, int $count): array
        {
            $feed = $this->feedRepository->ensureFeedForReader($user);
    
            return $feed === null ? [] : array_slice($feed->getTweets(), -$count);
        }
    
        public function spreadTweetAsync(TweetModel $tweet): void
        {
            $this->publishTweetBus->sendPublishTweetMessage($tweet);
        }
    
        public function spreadTweetSync(TweetModel $tweet): void
        {
            $followers = $this->subscriptionService->getFollowers($tweet->authorId);
    
            foreach ($followers as $follower) {
                $this->materializeTweet($tweet, $follower);
            }
        }
        
        public function materializeTweet(TweetModel $tweet, User $follower): void
        {
            $this->feedRepository->putTweetToReaderFeed($tweet, $follower);
            $sendNotificationDTO = new SendNotificationDTO(
                $follower->getId(),
                $tweet->text,
                $follower instanceof EmailUser ? CommunicationChannelEnum::Email : CommunicationChannelEnum::Phone
            );
            $this->sendNotificationBus->sendNotification($sendNotificationDTO);
        }
    }
    ```
3. Создаём класс `App\Controller\Amqp\UpdateFeed\Input\Message`
    ```php
    <?php
    
    namespace App\Controller\Amqp\UpdateFeed\Input;
    
    use DateTime;
    use Symfony\Component\Validator\Constraints as Assert;
    
    class Message
    {
        public function __construct(
            #[Assert\Type('numeric')]
            public readonly int $id,
            public readonly string $author,
            #[Assert\Type('numeric')]
            public readonly int $authorId,
            public readonly string $text,
            public readonly DateTime $createdAt,
            #[Assert\Type('numeric')]
            public readonly int $followerId,
        ) {
        }
    }
    ``` 
4. Создаём класс `App\Controller\Amqp\UpdateFeed\Consumer`
    ```php
    <?php
    
    namespace App\Controller\Amqp\UpdateFeed;
    
    use App\Application\RabbitMq\AbstractConsumer;
    use App\Controller\Amqp\UpdateFeed\Input\Message;
    use App\Domain\Entity\User;
    use App\Domain\Model\TweetModel;
    use App\Domain\Service\FeedService;
    use App\Domain\Service\UserService;
    
    class Consumer extends AbstractConsumer
    {
        public function __construct(
            private readonly FeedService $feedService,
            private readonly UserService $userService,
        ) {
        }
    
        protected function getMessageClass(): string
        {
            return Message::class;
        }
    
        /**
         * @param Message $message
         */
        protected function handle($message): int
        {
            $tweet = new TweetModel(
                $message->id,
                $message->author,
                $message->authorId,
                $message->text,
                $message->createdAt,
            );
            $user = $this->userService->findUserById($message->followerId);
            if (!($user instanceof User)) {
                $this->reject('User {$message->followerId} was not found');
            }
            $this->feedService->materializeTweet($tweet, $user);
    
            return self::MSG_ACK;
        }
    }
    ```
5. Добавляем класс `App\Domain\DTO\UpdateFeedDTO`
    ```php
    <?php
    
    namespace App\Domain\DTO;
    
    use DateTime;
    
    class UpdateFeedDTO
    {
        public function __construct(
            public readonly int $id,
            public readonly string $author,
            public readonly int $authorId,
            public readonly string $text,
            public readonly DateTime $createdAt,
            public readonly int $followerId,
        ) {
        }
    }
    ```
6. Добавляем интерфейс `App\Domain\Bus\UpdateFeedBusInterface`
    ```php
    <?php
    
    namespace App\Domain\Bus;
    
    use App\Domain\DTO\UpdateFeedDTO;
    
    interface UpdateFeedBusInterface
    {
        public function sendUpdateFeedMessage(UpdateFeedDTO $updateFeedDTO): bool;
    }
    ```
7. Добавляем класс `App\Infrastructure\Bus\Adapter\UpdateFeedRabbitMqBus`
    ```php
    <?php
    
    namespace App\Infrastructure\Bus\Adapter;
    
    use App\Domain\Bus\UpdateFeedBusInterface;
    use App\Domain\DTO\UpdateFeedDTO;
    use App\Infrastructure\Bus\AmqpExchangeEnum;
    use App\Infrastructure\Bus\RabbitMqBus;
    
    class UpdateFeedRabbitMqBus implements UpdateFeedBusInterface
    {
        public function __construct(private readonly RabbitMqBus $rabbitMqBus)
        {
        }
    
        public function sendUpdateFeedMessage(UpdateFeedDTO $updateFeedDTO): bool
        {
            return $this->rabbitMqBus->publishToExchange(
                AmqpExchangeEnum::UpdateFeed,
                $updateFeedDTO,
                (string)$updateFeedDTO->followerId
            );
        }
    }
    ```
8. Исправляем перечисление `App\Infrastructure\Bus\AmqpExchangeEnum`
    ```php
    <?php
    
    namespace App\Infrastructure\Bus;
    
    enum AmqpExchangeEnum: string
    {
        case AddFollowers = 'add_followers';
        case PublishTweet = 'publish_tweet';
        case SendNotification = 'send_notification';
        case UpdateFeed = 'update_feed';
    }
    ```
9. В файл `config/services.yaml` добавляем к сервису `App\Infrastructure\Bus\RabbitMqBus` регистрацию нового продюсера:
    ```yaml
    - [ 'registerProducer', [ !php/enum App\Infrastructure\Bus\AmqpExchangeEnum::UpdateFeed, '@old_sound_rabbit_mq.update_feed_producer' ] ]
    ```
10. Исправляем класс `App\Controller\Amqp\PublishTweet\Consumer`
    ```php
    <?php
   
    namespace App\Controller\Amqp\PublishTweet;
   
    use App\Application\RabbitMq\AbstractConsumer;
    use App\Controller\Amqp\PublishTweet\Input\Message;
    use App\Domain\Bus\UpdateFeedBusInterface;
    use App\Domain\DTO\UpdateFeedDTO;
    use App\Domain\Service\SubscriptionService;
   
    class Consumer extends AbstractConsumer
    {
        public function __construct(
            private readonly SubscriptionService $subscriptionService,
            private readonly UpdateFeedBusInterface $updateFeedBus,
        ) {
        }
   
        protected function getMessageClass(): string
        {
            return Message::class;
        }
   
        /**
         * @param Message $message
         */
        protected function handle($message): int
        {
            $followers = $this->subscriptionService->getFollowers($message->authorId);
            foreach ($followers as $follower) {
                $updateFeedDTO = new UpdateFeedDTO(
                    $message->id,
                    $message->author,
                    $message->authorId,
                    $message->text,
                    $message->createdAt,
                    $follower->getId(),
                );
                $this->updateFeedBus->sendUpdateFeedMessage($updateFeedDTO);
            }
   
            return self::MSG_ACK;
        }
    }
    ```
11. Добавляем описание нового продюсера и консьюмеров в файл `config/packages/old_sound_rabbit_mq.yaml`
   1. в секцию `producers`
       ```yaml
       update_feed:
           connection: default
           exchange_options: {name: 'old_sound_rabbit_mq.update_feed', type: x-consistent-hash}
       ```
   2. в секцию `consumers`
        ```yaml
        update_feed_0:
            connection: default
            exchange_options: {name: 'old_sound_rabbit_mq.update_feed', type: x-consistent-hash}
            queue_options: {name: 'old_sound_rabbit_mq.consumer.update_feed_0', routing_key: '1'}
            callback: App\Controller\Amqp\UpdateFeed\Consumer
            idle_timeout: 300
            idle_timeout_exit_code: 0
            graceful_max_execution:
                timeout: 1800
                exit_code: 0
            qos_options: {prefetch_size: 0, prefetch_count: 1, global: false}
        update_feed_1:
            connection: default
            exchange_options: {name: 'old_sound_rabbit_mq.update_feed', type: x-consistent-hash}
            queue_options: {name: 'old_sound_rabbit_mq.consumer.update_feed_1', routing_key: '1'}
            callback: App\Controller\Amqp\UpdateFeed\Consumer
            idle_timeout: 300
            idle_timeout_exit_code: 0
            graceful_max_execution:
                timeout: 1800
                exit_code: 0
            qos_options: {prefetch_size: 0, prefetch_count: 1, global: false}
        update_feed_2:
            connection: default
            exchange_options: {name: 'old_sound_rabbit_mq.update_feed', type: x-consistent-hash}
            queue_options: {name: 'old_sound_rabbit_mq.consumer.update_feed_2', routing_key: '1'}
            callback: App\Controller\Amqp\UpdateFeed\Consumer
            idle_timeout: 300
            idle_timeout_exit_code: 0
            graceful_max_execution:
                timeout: 1800
                exit_code: 0
            qos_options: {prefetch_size: 0, prefetch_count: 1, global: false}
        update_feed_3:
            connection: default
            exchange_options: {name: 'old_sound_rabbit_mq.update_feed', type: x-consistent-hash}
            queue_options: {name: 'old_sound_rabbit_mq.consumer.update_feed_3', routing_key: '1'}
            callback: App\Controller\Amqp\UpdateFeed\Consumer
            idle_timeout: 300
            idle_timeout_exit_code: 0
            graceful_max_execution:
                timeout: 1800
                exit_code: 0
            qos_options: {prefetch_size: 0, prefetch_count: 1, global: false}
        update_feed_4:
            connection: default
            exchange_options: {name: 'old_sound_rabbit_mq.update_feed', type: x-consistent-hash}
            queue_options: {name: 'old_sound_rabbit_mq.consumer.update_feed_4', routing_key: '1'}
            callback: App\Controller\Amqp\UpdateFeed\Consumer
            idle_timeout: 300
            idle_timeout_exit_code: 0
            graceful_max_execution:
                timeout: 1800
                exit_code: 0
            qos_options: {prefetch_size: 0, prefetch_count: 1, global: false}
        update_feed_5:
            connection: default
            exchange_options: {name: 'old_sound_rabbit_mq.update_feed', type: x-consistent-hash}
            queue_options: {name: 'old_sound_rabbit_mq.consumer.update_feed_5', routing_key: '1'}
            callback: App\Controller\Amqp\UpdateFeed\Consumer
            idle_timeout: 300
            idle_timeout_exit_code: 0
            graceful_max_execution:
                timeout: 1800
                exit_code: 0
            qos_options: {prefetch_size: 0, prefetch_count: 1, global: false}
        update_feed_6:
            connection: default
            exchange_options: {name: 'old_sound_rabbit_mq.update_feed', type: x-consistent-hash}
            queue_options: {name: 'old_sound_rabbit_mq.consumer.update_feed_6', routing_key: '1'}
            callback: App\Controller\Amqp\UpdateFeed\Consumer
            idle_timeout: 300
            idle_timeout_exit_code: 0
            graceful_max_execution:
                timeout: 1800
                exit_code: 0
            qos_options: {prefetch_size: 0, prefetch_count: 1, global: false}
        update_feed_7:
            connection: default
            exchange_options: {name: 'old_sound_rabbit_mq.update_feed', type: x-consistent-hash}
            queue_options: {name: 'old_sound_rabbit_mq.consumer.update_feed_7', routing_key: '1'}
            callback: App\Controller\Amqp\UpdateFeed\Consumer
            idle_timeout: 300
            idle_timeout_exit_code: 0
            graceful_max_execution:
                timeout: 1800
                exit_code: 0
            qos_options: {prefetch_size: 0, prefetch_count: 1, global: false}
        update_feed_8:
            connection: default
            exchange_options: {name: 'old_sound_rabbit_mq.update_feed', type: x-consistent-hash}
            queue_options: {name: 'old_sound_rabbit_mq.consumer.update_feed_8', routing_key: '1'}
            callback: App\Controller\Amqp\UpdateFeed\Consumer
            idle_timeout: 300
            idle_timeout_exit_code: 0
            graceful_max_execution:
                timeout: 1800
                exit_code: 0
            qos_options: {prefetch_size: 0, prefetch_count: 1, global: false}
        update_feed_9:
            connection: default
            exchange_options: {name: 'old_sound_rabbit_mq.update_feed', type: x-consistent-hash}
            queue_options: {name: 'old_sound_rabbit_mq.consumer.update_feed_9', routing_key: '1'}
            callback: App\Controller\Amqp\UpdateFeed\Consumer
            idle_timeout: 300
            idle_timeout_exit_code: 0
            graceful_max_execution:
                timeout: 1800
                exit_code: 0
            qos_options: {prefetch_size: 0, prefetch_count: 1, global: false}
        ```
12. Добавляем новые консьюмеры в конфигурацию `supervisor` в файле `supervisor/consumer.conf`
     ```ini
     [program:update_feed_0]
     command=php -dmemory_limit=1G /app/bin/console rabbitmq:consumer -m 100 update_feed_0 --env=dev -vv
     process_name=update_feed_0_%(process_num)02d
     numprocs=1
     directory=/tmp
     autostart=true
     autorestart=true
     startsecs=3
     startretries=10
     user=www-data
     redirect_stderr=false
     stdout_logfile=/app/var/log/supervisor.update_feed.out.log
     stdout_capture_maxbytes=1MB
     stderr_logfile=/app/var/log/supervisor.update_feed.error.log
     stderr_capture_maxbytes=1MB
    
     [program:update_feed_1]
     command=php -dmemory_limit=1G /app/bin/console rabbitmq:consumer -m 100 update_feed_1 --env=dev -vv
     process_name=update_feed_1_%(process_num)02d
     numprocs=1
     directory=/tmp
     autostart=true
     autorestart=true
     startsecs=3
     startretries=10
     user=www-data
     redirect_stderr=false
     stdout_logfile=/app/var/log/supervisor.update_feed.out.log
     stdout_capture_maxbytes=1MB
     stderr_logfile=/app/var/log/supervisor.update_feed.error.log
     stderr_capture_maxbytes=1MB
    
     [program:update_feed_2]
     command=php -dmemory_limit=1G /app/bin/console rabbitmq:consumer -m 100 update_feed_2 --env=dev -vv
     process_name=update_feed_2_%(process_num)02d
     numprocs=1
     directory=/tmp
     autostart=true
     autorestart=true
     startsecs=3
     startretries=10
     user=www-data
     redirect_stderr=false
     stdout_logfile=/app/var/log/supervisor.update_feed.out.log
     stdout_capture_maxbytes=1MB
     stderr_logfile=/app/var/log/supervisor.update_feed.error.log
     stderr_capture_maxbytes=1MB
    
     [program:update_feed_3]
     command=php -dmemory_limit=1G /app/bin/console rabbitmq:consumer -m 100 update_feed_3 --env=dev -vv
     process_name=update_feed_3_%(process_num)02d
     numprocs=1
     directory=/tmp
     autostart=true
     autorestart=true
     startsecs=3
     startretries=10
     user=www-data
     redirect_stderr=false
     stdout_logfile=/app/var/log/supervisor.update_feed.out.log
     stdout_capture_maxbytes=1MB
     stderr_logfile=/app/var/log/supervisor.update_feed.error.log
     stderr_capture_maxbytes=1MB
    
     [program:update_feed_4]
     command=php -dmemory_limit=1G /app/bin/console rabbitmq:consumer -m 100 update_feed_4 --env=dev -vv
     process_name=update_feed_4_%(process_num)02d
     numprocs=1
     directory=/tmp
     autostart=true
     autorestart=true
     startsecs=3
     startretries=10
     user=www-data
     redirect_stderr=false
     stdout_logfile=/app/var/log/supervisor.update_feed.out.log
     stdout_capture_maxbytes=1MB
     stderr_logfile=/app/var/log/supervisor.update_feed.error.log
     stderr_capture_maxbytes=1MB
    
     [program:update_feed_5]
     command=php -dmemory_limit=1G /app/bin/console rabbitmq:consumer -m 100 update_feed_5 --env=dev -vv
     process_name=update_feed_5_%(process_num)02d
     numprocs=1
     directory=/tmp
     autostart=true
     autorestart=true
     startsecs=3
     startretries=10
     user=www-data
     redirect_stderr=false
     stdout_logfile=/app/var/log/supervisor.update_feed.out.log
     stdout_capture_maxbytes=1MB
     stderr_logfile=/app/var/log/supervisor.update_feed.error.log
     stderr_capture_maxbytes=1MB
    
     [program:update_feed_6]
     command=php -dmemory_limit=1G /app/bin/console rabbitmq:consumer -m 100 update_feed_6 --env=dev -vv
     process_name=update_feed_6_%(process_num)02d
     numprocs=1
     directory=/tmp
     autostart=true
     autorestart=true
     startsecs=3
     startretries=10
     user=www-data
     redirect_stderr=false
     stdout_logfile=/app/var/log/supervisor.update_feed.out.log
     stdout_capture_maxbytes=1MB
     stderr_logfile=/app/var/log/supervisor.update_feed.error.log
     stderr_capture_maxbytes=1MB
    
     [program:update_feed_7]
     command=php -dmemory_limit=1G /app/bin/console rabbitmq:consumer -m 100 update_feed_7 --env=dev -vv
     process_name=update_feed_7_%(process_num)02d
     numprocs=1
     directory=/tmp
     autostart=true
     autorestart=true
     startsecs=3
     startretries=10
     user=www-data
     redirect_stderr=false
     stdout_logfile=/app/var/log/supervisor.update_feed.out.log
     stdout_capture_maxbytes=1MB
     stderr_logfile=/app/var/log/supervisor.update_feed.error.log
     stderr_capture_maxbytes=1MB
    
     [program:update_feed_8]
     command=php -dmemory_limit=1G /app/bin/console rabbitmq:consumer -m 100 update_feed_8 --env=dev -vv
     process_name=update_feed_8_%(process_num)02d
     numprocs=1
     directory=/tmp
     autostart=true
     autorestart=true
     startsecs=3
     startretries=10
     user=www-data
     redirect_stderr=false
     stdout_logfile=/app/var/log/supervisor.update_feed.out.log
     stdout_capture_maxbytes=1MB
     stderr_logfile=/app/var/log/supervisor.update_feed.error.log
     stderr_capture_maxbytes=1MB
    
     [program:update_feed_9]
     command=php -dmemory_limit=1G /app/bin/console rabbitmq:consumer -m 100 update_feed_9 --env=dev -vv
     process_name=update_feed_9_%(process_num)02d
     numprocs=1
     directory=/tmp
     autostart=true
     autorestart=true
     startsecs=3
     startretries=10
     user=www-data
     redirect_stderr=false
     stdout_logfile=/app/var/log/supervisor.update_feed.out.log
     stdout_capture_maxbytes=1MB
     stderr_logfile=/app/var/log/supervisor.update_feed.error.log
     stderr_capture_maxbytes=1MB
     ```
13. Перезапускаем контейнер `supervisor` командой `docker-compose restart supervisor`
14. Видим, что в RabbitMQ появились очереди с консьюмерами и точка обмена типа `x-consistent-hash`
15. Выполняем запрос Post tweet из Postman-коллекции v8 с параметром `async` = 1
16. В интерфейсе RabbitMQ можно увидеть, что в некоторые очереди насыпались сообщения, но сложно оценить равномерность
    распределения

## Добавляем мониторинг

1. Исправляем класс `App\Controller\Amqp\UpdateFeed\Consumer`
    ```php
    <?php
    
    namespace App\Controller\Amqp\UpdateFeed;
    
    use App\Application\RabbitMq\AbstractConsumer;
    use App\Controller\Amqp\UpdateFeed\Input\Message;
    use App\Domain\Entity\User;
    use App\Domain\Model\TweetModel;
    use App\Domain\Service\FeedService;
    use App\Domain\Service\UserService;
    use App\Infrastructure\Storage\MetricsStorage;
    
    class Consumer extends AbstractConsumer
    {
        public function __construct(
            private readonly FeedService $feedService,
            private readonly UserService $userService,
            private readonly MetricsStorage $metricsStorage,
            private readonly string $key,
        ) {
        }
    
        protected function getMessageClass(): string
        {
            return Message::class;
        }
    
        /**
         * @param Message $message
         */
        protected function handle($message): int
        {
            $tweet = new TweetModel(
                $message->id,
                $message->author,
                $message->authorId,
                $message->text,
                $message->createdAt,
            );
            $user = $this->userService->findUserById($message->followerId);
            if (!($user instanceof User)) {
                $this->reject('User {$message->followerId} was not found');
            }
            $this->feedService->materializeTweet($tweet, $user);
            $this->metricsStorage->increment($this->key);
    
            return self::MSG_ACK;
        }
    }
    ```
2. Добавляем в `config/services.yaml` инъекцию названий метрик в консьюмеры
    ```yaml
    App\Controller\Amqp\UpdateFeed\Consumer0:
        class: App\Controller\Amqp\UpdateFeed\Consumer
        arguments:
            $key: 'update_feed_0'

    App\Controller\Amqp\UpdateFeed\Consumer1:
        class: App\Controller\Amqp\UpdateFeed\Consumer
        arguments:
            $key: 'update_feed_1'

    App\Controller\Amqp\UpdateFeed\Consumer2:
        class: App\Controller\Amqp\UpdateFeed\Consumer
        arguments:
            $key: 'update_feed_2'

    App\Controller\Amqp\UpdateFeed\Consumer3:
        class: App\Controller\Amqp\UpdateFeed\Consumer
        arguments:
            $key: 'update_feed_3'

    App\Controller\Amqp\UpdateFeed\Consumer4:
        class: App\Controller\Amqp\UpdateFeed\Consumer
        arguments:
            $key: 'update_feed_4'

    App\Controller\Amqp\UpdateFeed\Consumer5:
        class: App\Controller\Amqp\UpdateFeed\Consumer
        arguments:
            $key: 'update_feed_5'

    App\Controller\Amqp\UpdateFeed\Consumer6:
        class: App\Controller\Amqp\UpdateFeed\Consumer
        arguments:
            $key: 'update_feed_6'

    App\Controller\Amqp\UpdateFeed\Consumer7:
        class: App\Controller\Amqp\UpdateFeed\Consumer
        arguments:
            $key: 'update_feed_7'

    App\Controller\Amqp\UpdateFeed\Consumer8:
        class: App\Controller\Amqp\UpdateFeed\Consumer
        arguments:
            $key: 'update_feed_8'

    App\Controller\Amqp\UpdateFeed\Consumer9:
        class: App\Controller\Amqp\UpdateFeed\Consumer
        arguments:
            $key: 'update_feed_9'            
    ```
3. В файл `config/packages/old_sound_rabbit_mq.yaml` в секции `consumers` исправляем коллбэки для каждого консьюмера на
   `App\Controller\Amqp\UpdateFeed\ConsumerK`
    ```yaml
    update_feed_0:
      connection: default
      exchange_options: {name: 'old_sound_rabbit_mq.update_feed', type: x-consistent-hash}
      queue_options: {name: 'old_sound_rabbit_mq.consumer.update_feed_0', routing_key: '1'}
      callback: App\Controller\Amqp\UpdateFeed\Consumer0
      idle_timeout: 300
      idle_timeout_exit_code: 0
      graceful_max_execution:
        timeout: 1800
        exit_code: 0
      qos_options: {prefetch_size: 0, prefetch_count: 1, global: false}
    update_feed_1:
      connection: default
      exchange_options: {name: 'old_sound_rabbit_mq.update_feed', type: x-consistent-hash}
      queue_options: {name: 'old_sound_rabbit_mq.consumer.update_feed_1', routing_key: '1'}
      callback: App\Controller\Amqp\UpdateFeed\Consumer1
      idle_timeout: 300
      idle_timeout_exit_code: 0
      graceful_max_execution:
        timeout: 1800
        exit_code: 0
      qos_options: {prefetch_size: 0, prefetch_count: 1, global: false}
    update_feed_2:
      connection: default
      exchange_options: {name: 'old_sound_rabbit_mq.update_feed', type: x-consistent-hash}
      queue_options: {name: 'old_sound_rabbit_mq.consumer.update_feed_2', routing_key: '1'}
      callback: App\Controller\Amqp\UpdateFeed\Consumer2
      idle_timeout: 300
      idle_timeout_exit_code: 0
      graceful_max_execution:
        timeout: 1800
        exit_code: 0
      qos_options: {prefetch_size: 0, prefetch_count: 1, global: false}
    update_feed_3:
      connection: default
      exchange_options: {name: 'old_sound_rabbit_mq.update_feed', type: x-consistent-hash}
      queue_options: {name: 'old_sound_rabbit_mq.consumer.update_feed_3', routing_key: '1'}
      callback: App\Controller\Amqp\UpdateFeed\Consumer3
      idle_timeout: 300
      idle_timeout_exit_code: 0
      graceful_max_execution:
        timeout: 1800
        exit_code: 0
      qos_options: {prefetch_size: 0, prefetch_count: 1, global: false}
    update_feed_4:
      connection: default
      exchange_options: {name: 'old_sound_rabbit_mq.update_feed', type: x-consistent-hash}
      queue_options: {name: 'old_sound_rabbit_mq.consumer.update_feed_4', routing_key: '1'}
      callback: App\Controller\Amqp\UpdateFeed\Consumer4
      idle_timeout: 300
      idle_timeout_exit_code: 0
      graceful_max_execution:
        timeout: 1800
        exit_code: 0
      qos_options: {prefetch_size: 0, prefetch_count: 1, global: false}
    update_feed_5:
      connection: default
      exchange_options: {name: 'old_sound_rabbit_mq.update_feed', type: x-consistent-hash}
      queue_options: {name: 'old_sound_rabbit_mq.consumer.update_feed_5', routing_key: '1'}
      callback: App\Controller\Amqp\UpdateFeed\Consumer5
      idle_timeout: 300
      idle_timeout_exit_code: 0
      graceful_max_execution:
        timeout: 1800
        exit_code: 0
      qos_options: {prefetch_size: 0, prefetch_count: 1, global: false}
    update_feed_6:
      connection: default
      exchange_options: {name: 'old_sound_rabbit_mq.update_feed', type: x-consistent-hash}
      queue_options: {name: 'old_sound_rabbit_mq.consumer.update_feed_6', routing_key: '1'}
      callback: App\Controller\Amqp\UpdateFeed\Consumer6
      idle_timeout: 300
      idle_timeout_exit_code: 0
      graceful_max_execution:
        timeout: 1800
        exit_code: 0
      qos_options: {prefetch_size: 0, prefetch_count: 1, global: false}
    update_feed_7:
      connection: default
      exchange_options: {name: 'old_sound_rabbit_mq.update_feed', type: x-consistent-hash}
      queue_options: {name: 'old_sound_rabbit_mq.consumer.update_feed_7', routing_key: '1'}
      callback: App\Controller\Amqp\UpdateFeed\Consumer7
      idle_timeout: 300
      idle_timeout_exit_code: 0
      graceful_max_execution:
        timeout: 1800
        exit_code: 0
      qos_options: {prefetch_size: 0, prefetch_count: 1, global: false}
    update_feed_8:
      connection: default
      exchange_options: {name: 'old_sound_rabbit_mq.update_feed', type: x-consistent-hash}
      queue_options: {name: 'old_sound_rabbit_mq.consumer.update_feed_8', routing_key: '1'}
      callback: App\Controller\Amqp\UpdateFeed\Consumer8
      idle_timeout: 300
      idle_timeout_exit_code: 0
      graceful_max_execution:
        timeout: 1800
        exit_code: 0
      qos_options: {prefetch_size: 0, prefetch_count: 1, global: false}
    update_feed_9:
      connection: default
      exchange_options: {name: 'old_sound_rabbit_mq.update_feed', type: x-consistent-hash}
      queue_options: {name: 'old_sound_rabbit_mq.consumer.update_feed_9', routing_key: '1'}
      callback: App\Controller\Amqp\UpdateFeed\Consumer9
      idle_timeout: 300
      idle_timeout_exit_code: 0
      graceful_max_execution:
        timeout: 1800
        exit_code: 0
      qos_options: {prefetch_size: 0, prefetch_count: 1, global: false}
    ```
4. Перезапускаем контейнер `supervisor` командой `docker-compose restart supervisor`
5. Выполняем несколько запросов Post tweet из Postman-коллекции v8 с параметром `async` = 1
6. Заходим в Grafana по адресу `localhost:3000` с логином / паролем `admin` / `admin`
7. Добавляем Data source с типом Graphite и url `http://graphite:80`
8. Добавляем Dashboard и Panel
9. В панели отображаем графики для метрик stats_counts.my_app.update_feedX, где X – номер консьюмера
10. Видим, что распределение не очень равномерное

## Балансируем консьюмеры

1. В файл `config/packages/old_sound_rabbit_mq.yaml` в секции `consumers` исправляем для каждого консьюмера значение на
   `routing_key` на 20
2. Перезапускаем контейнер `supervisor` командой `docker-compose restart supervisor`
3. Выполняем несколько запросов Post tweet из Postman-коллекции v9 с параметром `async` = 1
4. Видим, что распределение стало гораздо равномернее   




# Полнотекстовый поиск, Elastica

## Установка elastica-bundle

1. Устанавливаем пакет `friendsofsymfony/elastica-bundle`
2. В файле `.env` исправляем DSN для ElasticSearch
    ```shell script
    ELASTICSEARCH_URL=http://elasticsearch:9200/
    ```
3. Выполняем запрос Add user v2 из Postman-коллекции  v10
4. Выполняем запрос Add followers из Postman-коллекции  v10, чтобы получить побольше записей в БД
5. В файле `config/packages/fos_elastica.yaml` в секции `fos_elastica.indexes` удаляем `app` и добавляем секцию `user`:
    ```yaml
    user:
        persistence:
            driver: orm
            model: App\Domain\Entity\User
        properties:
            login: ~
            age: ~
            phone: ~
            email: ~
    ```
6. Заполняем индекс командой `php bin/console fos:elastica:populate`
7. Заходим в Kibana по адресу `http://localhost:5601`
8. В Kibana заходим в Stack Management -> Index patterns и создаём index pattern на базе индекса `user`
9. Переходим в `Discover`, видим наши данные в новом шаблоне, причём телефон и email корректно заполняются

## Добавляем кастомное свойство

1. Добавляем класс `App\Application\Elastica\UserPropertyListener`
    ```php
    <?php
    
    namespace App\Application\Elastica;
    
    use App\Domain\Entity\EmailUser;
    use App\Domain\Entity\User;
    use App\Domain\ValueObject\CommunicationChannelEnum;
    use FOS\ElasticaBundle\Event\PostTransformEvent;
    use Symfony\Component\EventDispatcher\EventSubscriberInterface;
    
    class UserPropertyListener implements EventSubscriberInterface
    {
        public static function getSubscribedEvents()
        {
            return [
                PostTransformEvent::class => 'addCommunicationMethodProperties'
            ];
        }
    
        public function addCommunicationMethodProperties(PostTransformEvent $event): void
        {
            $user = $event->getObject();
            if ($user instanceof User) {
                $document = $event->getDocument();
                $document->set(
                    'communicationMethod',
                    $user instanceof EmailUser ? CommunicationChannelEnum::Email : CommunicationChannelEnum::Phone
                );
            }
        }
    }
    ```
2. Ещё раз заполняем индекс командой `php bin/console fos:elastica:populate`
3. В Kibana обновляем страницу, видим новое поле.

## Используем вложенные документы

1. Выполняем запрос Post tweet из Postman-коллекции  v10, чтобы получить запись в таблице `tweet`
2. Добавим индекс с составными полями в `config/packages/fos_elastica.yaml` в секцию `fos_elastica.indexes`
    ```yaml
    tweet:
        persistence:
            driver: orm
            model: App\Domain\Entity\Tweet
        properties:
            author:
                type: nested
                properties:
                    name:
                        property_path: login
                    age: ~
                    phone: ~
                    email: ~
            text: ~
    ```
3. В контейнере ещё раз заполняем индекс командой `php bin/console fos:elastica:populate`
4. В Kibana заходим в Stack Management -> Index patterns и создаём index pattern на базе индекса `tweet`
5. Переходим в `Discover`, видим наши данные в новом шаблоне

## Используем сериализацию вместо описания схемы

1. В файле `config/packages/fos_elastica.yaml`
   1. Включаем сериализацию
       ```yaml
       serializer: ~
       ```
   2. Для каждого индекса (`user`, `tweet`) удаляем секцию `properties` и добавляем секцию `serializer`
       ```yaml
       serializer:
           groups: [elastica]
       ```
2. В классе `App\Domain\Entity\User` добавляем атрибуты для полей `id`, `login`, `age`
    ```php
    #[ORM\Column(name: 'id', type: 'bigint', unique: true)]
    #[ORM\Id]
    #[ORM\GeneratedValue(strategy: 'IDENTITY')]
    #[Groups(['elastica'])]
    private ?int $id = null;
   
    #[ORM\Column(type: 'string', length: 32, unique: true, nullable: false)]
    #[Groups(['elastica'])]
    private string $login;

    #[ORM\Column(type: 'integer', nullable: false)]
    #[Groups(['elastica'])]
    private int $age;
    ```
3. В классе `App\Domain\Entity\PhoneUser` добавляем атрибут для поля `phone`
    ```php
    #[ORM\Column(type: 'string', length: 20, nullable: false)]
    #[Groups(['elastica'])]
    private string $phone;
    ```
4. В классе `App\Domain\Entity\EmailUser` добавляем атрибут для поля `email`
    ```php
    #[ORM\Column(type: 'string', nullable: false)]
    #[Groups(['elastica'])]
    private string $email;
    ```
5. В классе `App\Domain\Entity\Tweet` добавляем атрибуты для полей `id`, `author` и `text`
    ```php
    #[ORM\Column(name: 'id', type: 'bigint', unique: true)]
    #[ORM\Id]
    #[ORM\GeneratedValue(strategy: 'IDENTITY')]
    #[Groups(['elastica'])]
    private ?int $id = null;
   
    #[ORM\ManyToOne(targetEntity: 'User', inversedBy: 'tweets')]
    #[ORM\JoinColumn(name: 'author_id', referencedColumnName: 'id')]
    #[Groups(['elastica'])]
    private User $author;

    #[ORM\Column(type: 'string', length: 140, nullable: false)]
    #[Groups(['elastica'])]
    private string $text;
    ```
6. Ещё раз заполняем индекс командой `php bin/console fos:elastica:populate`
7. Проверяем в Kibana, что в индексах данные присутствуют

##  Отключаем автообновление индекса

1. Выполняем запрос Add user v2 из Postman-коллекции  v10
2. Проверяем в Kibana, что новая запись появилась в индексе
3. Отключаем listener для insert в файле `config/fos_elastica.yaml` путём добавления секции
   `fos_elastica.indexes.user.persistence.listener`
    ```yaml
    listener:
        insert: false
        update: true
        delete: true
    ```
4. Выполняем ещё один запрос Add user v2 из Postman-коллекции  v10
5. Проверяем в Kibana, что новая запись не появилась в индексе, хотя в БД она есть

### Ищем по индексу

1. В классе `App\Infrastructure\Repository\UserRepository`
   1. добавляем конструктор
       ```php
       public function __construct(
           EntityManagerInterface $entityManager,
           private readonly PaginatedFinderInterface $finder,
       ) {
           parent::__construct($entityManager);
       }
       ```
   2. Добавляем метод `findUsersByQuery`
       ```php
       /**
        * @return User[]
        */
       public function findUsersByQuery(string $query, int $perPage, int $page): array
       {
           $paginatedResult = $this->finder->findPaginated($query);
           $paginatedResult->setMaxPerPage($perPage);
           $paginatedResult->setCurrentPage($page);
   
           return [...$paginatedResult->getCurrentPageResults()];
       }
       ```
2. В файле `config/services.yaml` добавляем новый сервис:
    ```yaml
    App\Infrastructure\Repository\UserRepository:
        arguments:
            $finder: '@fos_elastica.finder.user'
    ```
3. В классе `App\Domain\Service\UserService` добавляем метод `findUsersByQuery`
    ```php
    /**
     * @return User[]
     */
    public function findUsersByQuery(string $query, int $perPage, int $page): array
    {
        return $this->userRepository->findUsersByQuery($query, $perPage, $page);
    }
    ```
4. Добавляем класс `App\Controller\Web\GetUsersByQuery\v1\Manager`
    ```php
    <?php
    
    namespace App\Controller\Web\GetUsersByQuery\v1;
    
    use App\Domain\Entity\User;
    use App\Domain\Service\UserService;
    
    class Manager
    {
        public function __construct(private readonly UserService $userService)
        {
        }
    
        /**
         * @return User[]
         */
        public function findUsersByQuery(string $query, int $perPage, int $page): array
        {
            return $this->userService->findUsersByQuery($query, $perPage, $page);
        }
    }
    ```
5. Добавляем класс `App\Controller\Web\GetUsersByQuery\v1\Controller`
    ```php
    <?php
    
    namespace App\Controller\Web\GetUsersByQuery\v1;
    
    use App\Domain\Entity\User;
    use Symfony\Component\HttpFoundation\JsonResponse;
    use Symfony\Component\HttpFoundation\Response;
    use Symfony\Component\HttpKernel\Attribute\AsController;
    use Symfony\Component\HttpKernel\Attribute\MapQueryParameter;
    use Symfony\Component\Routing\Attribute\Route;
    
    #[AsController]
    class Controller
    {
        public function __construct(private readonly Manager $manager) {
        }
    
        #[Route(path: 'api/v1/get-users-by-query', methods: ['GET'])]
        public function __invoke(#[MapQueryParameter]string $query, #[MapQueryParameter]int $perPage, #[MapQueryParameter]int $page): Response
        {
            return new JsonResponse(
                [
                    'users' => array_map(
                        static fn (User $user): array => $user->toArray(),
                        $this->manager->findUsersByQuery($query, $perPage, $page)
                    )
                ]
            );
        }
    }
    ```
6. Выполняем несколько запросов Get users by query из Postman-коллекции v10 с данными из разных полей разных
   пользователей

## Делаем нечёткий поиск

1. В классе `App\Infrastructure\Repository\UserRepository` исправляем метод `findUsersByQuery`
    ```php
    /**
     * @return User[]
     */
    public function findUsersByQuery(string $query, int $perPage, int $page): array
    {
        $paginatedResult = $this->finder->findPaginated($query.'~2');
        $paginatedResult->setMaxPerPage($perPage);
        $paginatedResult->setCurrentPage($page);

        return [...$paginatedResult->getCurrentPageResults()];
    }
    ```
2. Выполняем несколько запросов Get users by query из Postman-коллекции v10 с двумя опечатками

## Совмещаем фильтрацию по БД и Elasticsearch

1. Добавляем класс `App\Application\Doctrine\Repository\UserRepository`
    ```php
    <?php
    
    namespace App\Application\Doctrine;
    
    use Doctrine\ORM\EntityRepository;
    use Doctrine\ORM\QueryBuilder;
    
    class UserRepository extends EntityRepository
    {
        public function createIsActiveQueryBuilder(string $alias): QueryBuilder
        {
            return $this->createQueryBuilder($alias)
                ->andWhere("$alias.isActive = :isActive")
                ->setParameter('isActive', true);
        }
    }
    ```
2. В классе `App\Domain\Entity\User` исправляем атрибут
    ```php
    #[ORM\Entity(repositoryClass: UserRepository::class)]
    ```
3. Выполняем команду `php bin/console doctrine:cache:clear-metadata`
4. В файле `config/packages/fos_elastica.yaml` добавляем в секцию `fos_elastica.indexes.user.persistence` новую
   подсекцию
    ```yaml
    elastica_to_model_transformer:
        query_builder_method: createIsActiveQueryBuilder
        ignore_missing: true
    ```
5. Выполняем запрос Get users by query из Postman-коллекции v10 с данными любого пользователя, видим результат
6. Проставляем для этого пользователя в БД `is_active = false`
7. Ещё раз выполняем запрос Get users by query из Postman-коллекции v10 с данными любого пользователя, видим, что
   пользователь не возвращается в ответе
