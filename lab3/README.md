# Отчет по лабораторной работе №3
### Задание:

1. Написать “плохой” CI/CD файл, который работает, но в нем есть не менее трех “bad practices” по написанию CI/CD
2. Написать “хороший” CI/CD, в котором эти плохие практики исправлены
3. В Readme описать каждую из плохих практик в плохом файле, почему она плохая и как в хорошем она была исправлена, как исправление повлияло на результат
4. Прочитать историю про Васю (она быстрая, забавная и того стоит): https://habr.com/ru/articles/689234/
5. Написать обычный отчет себе на гитхаб

------

## Ход работы

Сначала мы начали искать примеры плохихи практик и на удивление быстро нашли хороший [сайт](https://appcircle.io/blog/bad-ci-cd-practices), на котором были описаны плохие практики и как их можно улучшить

Теперь нам надо научиться писать .yml файлы, для этого мы изучили пару разных источников, но самым хорошим, как нам показалось был [этот](https://habr.com/ru/articles/914614/)

Наконец приступаем к написанию плохого файла

1. Первая ошибка, которую мы специально делаем это устанавливаем все зависимости и не кэшируем их
```
- name: Setup Node.js
  uses: actions/setup-node@v3

- name: Install dependencies
  run: npm install
```
Это приводит к тому, что каждый запуск pipeline загружает все зависимости заново, что в свою очередь значительно увеличивает время выполнения и тратит ресурсы

Для предотвращения этого мы исправили и добавили кэширование
```
- name: Setup Node.js
  uses: actions/setup-node@v4
  with:
    node-version: '18'
    cache: 'npm'

- name: Install dependencies
  run: npm ci
```
Теперь зависимости кэшируются между запусками и время установки сокращается с минут до секунд

2. Вторая ошибка это постоянный запуск всех тестов
```
- name: Run ALL tests
  run: |
    npm test
    npm run integration-test
```
Все тесты запускаются при любом изменении, длинные интеграционные тесты замедляют получегие обратной связи

Исправили мы это путем разделения тестов 
```
- name: Run unit tests
  run: npm test

- name: Run integration tests (only on main)
  if: github.ref == 'refs/heads/main'
  run: npm run integration-test
```
Теперь юнит тесты запускаются всегда, а интеграционные тесты запускаются только при мерже в main, что ускорит получения обратной связи разработчикам и экономит вычислительные ресурсы

3. И последняя ошибка это захардкоденные секреты
```
- name: Deploy to production
  run: |
    scp -r ./dist deploy@server.com:/var/www/app
    ssh deploy@server.com "cd /var/www/app && ./restart.sh"
```
Учетные данные видны всем, у кого есть доступ к репозиторию, cекреты хранятся в истории Git, что сильно нарушает безопасность проекта

Вместо этого нужно хранить в специально защищенном хранилище github
```
- name: Deploy to production
  env:
    DEPLOY_HOST: ${{ secrets.DEPLOY_HOST }}
    DEPLOY_USER: ${{ secrets.DEPLOY_USER }}
    SSH_PRIVATE_KEY: ${{ secrets.SSH_PRIVATE_KEY }}
  run: |
    echo "$SSH_PRIVATE_KEY" > private_key
    chmod 600 private_key
    scp -i private_key -r ./dist $DEPLOY_USER@$DEPLOY_HOST:/var/www/app
```

Таким образом доступ к секретам контролируется через настройки репозитория и соблюдаются принципы безопасности

## Итог
Были написаны три плохие практики, которые замедляют и делают небезопасным наш CI/CD pipeline
1. Отсутствие кэширования => Медленные сборки
2. Запуск всех тестов всегда => Долгая обратная связь
3. Хардкод секретов => Серьезная уязвимость безопасности
И была прочитана история про великого DBOps-инженера Василия


Плохой файл
```
name: Bad CI/CD Pipeline

on:
  push:
    branches: [ main ]

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v3

      - name: Install dependencies
        run: npm install

      - name: Run ALL tests
        run: |
          npm test
          npm run integration-test

      - name: Build application
        run: npm run build

      - name: Deploy to production
        run: |
          echo "Deploying to production..."
          scp -r ./dist deploy@server.com:/var/www/app
          ssh deploy@server.com "cd /var/www/app && ./restart.sh"
```

Хороший файл
```
name: Good CI/CD Pipeline

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run unit tests
        run: npm test

      - name: Run integration tests (only on main)
        if: github.ref == 'refs/heads/main'
        run: npm run integration-test

  deploy:
    runs-on: ubuntu-latest
    needs: test
    if: github.ref == 'refs/heads/main'
    environment: production

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Build application
        run: npm run build

      - name: Deploy to production
        env:
          DEPLOY_HOST: ${{ secrets.DEPLOY_HOST }}
          DEPLOY_USER: ${{ secrets.DEPLOY_USER }}
          SSH_PRIVATE_KEY: ${{ secrets.SSH_PRIVATE_KEY }}
        run: |
          echo "$SSH_PRIVATE_KEY" > private_key
          chmod 600 private_key
          scp -i private_key -r ./dist $DEPLOY_USER@$DEPLOY_HOST:/var/www/app
          ssh -i private_key $DEPLOY_USER@$DEPLOY_HOST "cd /var/www/app && ./restart.sh"
```
