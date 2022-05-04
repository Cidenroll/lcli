How to install the symfony webapp:

1. create a new folder with: `mkdir X` and `cd X`
2. create a new file `docker-compose.yml`, containing:
    [
        version: '3.8'
        services:
        database:
            container_name: database
            image: mysql:8.0
            command: --default-authentication-plugin=mysql_native_password
            environment:
            MYSQL_ROOT_PASSWORD: secret
            MYSQL_DATABASE: lcli
            MYSQL_USER: sadmin
            MYSQL_PASSWORD: sadmin
            ports:
                - '4306:3306'
            volumes:
                - ./mysql:/var/lib/mysql
        php:
            container_name: php
            build:
            context: ./php
            ports:
                - '9000:9000'
            volumes:
                - ./app:/var/www/symfony_docker
            depends_on:
                - database
        nginx:
            container_name: nginx
            image: nginx:stable-alpine
            ports:
            - '8080:80'
            volumes:
                - ./app:/var/www/symfony_docker
                - ./nginx/default.conf:/etc/nginx/conf.d/default.conf
            depends_on:
            - php
            - database    
    ]

3. create a php folder in the main root `mkdir php` and `touch Dockerfile` inside it, with:
    [
        FROM php:8.0-fpm
        RUN apt update \
            && apt install -y zlib1g-dev g++ git libicu-dev zip libzip-dev zip \
            && docker-php-ext-install intl opcache pdo pdo_mysql \
            && pecl install apcu \
            && docker-php-ext-enable apcu \
            && docker-php-ext-configure zip \
            && docker-php-ext-install zip
        WORKDIR /var/www/symfony_docker
        RUN curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer
        RUN curl -sS https://get.symfony.com/cli/installer | bash
        RUN mv /root/.symfony/bin/symfony /usr/local/bin/symfony
        RUN git config --global user.email "you@example.com" \ 
            && git config --global user.name "Your Name"
    ]  

4. `mdkir app` and `mkdir nginx` and create in this one `default.conf` with the following:
    [
        server {
            listen 80;
            index index.php;
            server_name localhost;
            root /var/www/symfony_docker/public;
            error_log /var/log/nginx/project_error.log;
            access_log /var/log/nginx/project_access.log;
            location / {
                try_files $uri /index.php$is_args$args;
            }
            location ~ ^/index\\.php(/|$) {
                fastcgi_pass php:9000;
                fastcgi_split_path_info ^(.+\\.php)(/.*)$;
                include fastcgi_params;
                fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
                fastcgi_param DOCUMENT_ROOT $realpath_root;
                fastcgi_buffer_size 128k;
                fastcgi_buffers 4 256k;
                fastcgi_busy_buffers_size 256k;
                internal;
            }
            location ~ \\.php$ {
                return 404;
            }
        }
    ]

5. docker-compose up -d --build
6. ssh inside the php container with `docker-compose exec php /bin/bash` and:
    6.1. `symfony check:requirements`
    6.2. `symfony new . --webapp`
    6.3. `composer req --dev maker ormfixtures fakerphp/faker`
    6.4. `composer req doctrine twig`
    6.5. `yarn install`

7. from the ./app/ do `cp .env .env.local` and change the DATABASE_URL:
    `DATABASE_URL="mysql://root:secret@database:3306/symfony_docker?serverVersion=8.0"`

8. to ssh into the database container:
    8.1. `docker-compose exec database /bin/bash`
    8.2. `mysql -u root -p symfony_docker`, the password is in the docker-compose.yml > database container settings
    8.3. Run `composer require asset` 

9. ssh to php container again with `docker-compose exec php /bin/bash`:
    9.1. install encore: `composer require symfony/webpack-encore-bundle`
    9.2. package.json - shows all used dependencies + (custom) scripts 
    9.3. webpack.config.js > will be using encore abstraction to configure webpack 
                           > will compile css/js into the public/build folder
                           > to access the public path @ /build
    9.4. in the app > assets folder  
    9.5. !~ to refresh changes in .css or .js files - FIRST compile the thru the encore dev command from package.json - scripts => run them thru npm    
        ex: `npm run dev`    
                  
10. To add assets into twig template, you can use the following:
    10.1. in base.html.twig > `{% block stylesheets %}{% endblock %}` contains the styles used by default
    10.2. to use an asset in twig: ex: 
        `{% block stylesheets %}<link rel="stylesheet" href="{{ asset('build/app.css') }}">{% endblock %}`

11. TAILWINDCSS
    11.1. Run in ./app/: `npm install -D tailwindcss postcss-loader purgecss-webpack-plugin glob-all path`
    11.2. to check if packages are imported: open package.json > devDependencies
    11.3. run `npx tailwindcss init -p`
    11.4. in the new `postcss.config.js`, fill it with:
        [
            let tailwindcss = require("tailwindcss")
            module.exports = {
            plugins: [
                tailwindcss('./tailwind.config.js'),
                require('postcss-import'),
                require('autoprefixer')
                ]
            }
        ]

    11.5. in `webpack.config.js`, modify before the module.exports:   
        [
            ...
                .enablePostCssLoader((options) => {
                    options.postcssOptions = {
                        config: './postcss.config.js'
                    }
                })
            ;
            ...
        ]

    11.6. in `assets\styles\app.css`:     
        [
            @tailwind base;
            @tailwind components;
            @tailwind utilities;
        ]

    11.7. in `tailwind.config.js`:
        [
            module.exports = {
                content: [
                    "./assets/**/*.{vue,js,ts,jsx,tsx}",
                    "./templates/**/*.{html,twig}"
                ],
                theme: {
                    extend: {},
                },
                plugins: [],
            }
        ]

    11.8. Compile tailwind in cli: `npx tailwindcss -i ./assets/styles/app.css -o ./public/builds/app.css`    
