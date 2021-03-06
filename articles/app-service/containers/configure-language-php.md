---
title: Configure PHP apps - Azure App Service | Microsoft Docs 
description: Learn how to configure PHP apps to work in Azure App Service
services: app-service
documentationcenter: ''
author: cephalin
manager: jpconnock
editor: ''

ms.service: app-service
ms.workload: na
ms.tgt_pltfrm: na
ms.devlang: dotnet
ms.topic: article
ms.date: 03/28/2019
ms.author: cephalin

---

# Configure a Linux PHP app for Azure App Service

This guide shows you how to configure the built-in PHP runtime for web apps, mobile back ends, and API apps in Azure App Service.

This guide provides key concepts and instructions for PHP developers who use a built-in Linux container in App Service. If you've never used Azure App Service, follow the [PHP quickstart](quickstart-php.md) and [PHP with MySQL tutorial](tutorial-php-mysql-app.md) first.

## Show PHP version

To show the current PHP version, run the following command in the [Cloud Shell](https://shell.azure.com):

```azurecli-interactive
az webapp config show --resource-group <resource-group-name> --name <app-name> --query linuxFxVersion
```

To show all supported PHP versions, run the following command in the [Cloud Shell](https://shell.azure.com):

```azurecli-interactive
az webapp list-runtimes --linux | grep PHP
```

## Set PHP version

Run the following command in the [Cloud Shell](https://shell.azure.com) to set the PHP version to 7.2:

```azurecli-interactive
az webapp config set --name <app-name> --resource-group <resource-group-name> --linux-fx-version "PHP|7.2"
```

## Run Composer

By default, Kudu doesn't run [Composer](https://getcomposer.org/). To enable Composer automation during Kudu deployment, you need to supply a [custom deployment script](https://github.com/projectkudu/kudu/wiki/Custom-Deployment-Script).

From a local terminal window, change directory to your repository root. Follow the [command-line installation steps](https://getcomposer.org/download/) to download *composer.phar*.

Run the following commands:

```bash
npm install kuduscript -g
kuduscript --php --scriptType bash --suppressPrompt
```

Your repository root now has two new files in addition to *composer.phar*: *.deployment* and *deploy.sh*. These files work both for Windows and Linux flavors of App Service.

Open *deploy.sh* and find the `Deployment` section. Replace the whole section with the following code:

```bash
##################################################################################################################################
# Deployment
# ----------

echo PHP deployment

# 1. KuduSync
if [[ "$IN_PLACE_DEPLOYMENT" -ne "1" ]]; then
  "$KUDU_SYNC_CMD" -v 50 -f "$DEPLOYMENT_SOURCE" -t "$DEPLOYMENT_TARGET" -n "$NEXT_MANIFEST_PATH" -p "$PREVIOUS_MANIFEST_PATH" -i ".git;.hg;.deployment;deploy.sh"
  exitWithMessageOnError "Kudu Sync failed"
fi

# 3. Initialize Composer Config
initializeDeploymentConfig

# 4. Use composer
echo "$DEPLOYMENT_TARGET"
if [ -e "$DEPLOYMENT_TARGET/composer.json" ]; then
  echo "Found composer.json"
  pushd "$DEPLOYMENT_TARGET"
  php composer.phar install $COMPOSER_ARGS
  exitWithMessageOnError "Composer install failed"
  popd
fi
##################################################################################################################################
```

Commit all your changes and deploy your code again. Composer should now be running as part of deployment automation.

## Customize start-up

By default, the built-in PHP container run the Apache server. At start-up, it runs `apache2ctl -D FOREGROUND"`. If you like, you can run a different command at start-up, by running the following command in the [Cloud Shell](https://shell.azure.com):

```azurecli-interactive
az webapp config set --resource-group <resource-group-name> --name <app-name> --startup-file "<custom-command>"
```

## Access environment variables

In App Service, you can [set app settings](../configure-common.md?toc=%2fazure%2fapp-service%2fcontainers%2ftoc.json#configure-app-settings) outside of your app code. Then you can access them using the standard [getenv()](https://secure.php.net/manual/function.getenv.php) pattern. For example, to access an app setting called `DB_HOST`, use the following code:

```php
getenv("DB_HOST")
```

## Change site root

The web framework of your choice may use a subdirectory as the site root. For example, [Laravel](https://laravel.com/), uses the `public/` subdirectory as the site root.

The default PHP image for App Service uses Apache, and it doesn't let you customize the site root for your app. To work around this limitation, add an *.htaccess* file to your repository root with the following content:

```
<IfModule mod_rewrite.c>
    RewriteEngine on

    RewriteRule ^.*$ /public/$1 [NC,L,QSA]
</IfModule>
```

If you would rather not use *.htaccess* rewrite, you can deploy your Laravel application with a [custom Docker image](quickstart-docker-go.md) instead.

## Detect HTTPS session

In App Service, [SSL termination](https://wikipedia.org/wiki/TLS_termination_proxy) happens at the network load balancers, so all HTTPS requests reach your app as unencrypted HTTP requests. If your app logic needs to check if the user requests are encrypted or not, inspect the `X-Forwarded-Proto` header.

```php
if (isset($_SERVER['X-Forwarded-Proto']) && $_SERVER['X-Forwarded-Proto'] === 'https') {
  // Do something when HTTPS is used
}
```

Popular web frameworks let you access the `X-Forwarded-*` information in your standard app pattern. In [CodeIgniter](https://codeigniter.com/), the [is_https()](https://github.com/bcit-ci/CodeIgniter/blob/master/system/core/Common.php#L338-L365) checks the value of `X_FORWARDED_PROTO` by default.

## Customize php.ini settings

If you need to make changes to your PHP installation, you can change any of the [php.ini directives](https://www.php.net/manual/ini.list.php) by following these steps.

> [!NOTE]
> The best way to see the PHP version and the current *php.ini* configuration is to call [phpinfo()](https://php.net/manual/function.phpinfo.php) in your app.
>

### <a name="Customize-non-PHP_INI_SYSTEM directives"></a>Customize-non-PHP_INI_SYSTEM directives

To customize PHP_INI_USER, PHP_INI_PERDIR, and PHP_INI_ALL directives (see [php.ini directives](https://www.php.net/manual/ini.list.php)), add an *.htaccess* file to the root directory of your app.

In the *.htaccess* file, add the directives using the `php_value <directive-name> <value>` syntax. For example:

```
php_value upload_max_filesize 1000M
php_value post_max_size 2000M
php_value memory_limit 3000M
php_value max_execution_time 180
php_value max_input_time 180
php_value display_errors On
php_value upload_max_filesize 10M
```

Redeploy your app with the changes and restart it. If you deploy it with Kudu (for example, using [Git](../deploy-local-git.md?toc=%2fazure%2fapp-service%2fcontainers%2ftoc.json)), it's automatically restarted after deployment.

As an alternative to using *.htaccess*, you can use [ini_set()](https://www.php.net/manual/function.ini-set.php) in your app to customize these non-PHP_INI_SYSTEM directives.

### <a name="customize-php_ini_system-directives"></a>Customize PHP_INI_SYSTEM directives

To customize PHP_INI_SYSTEM directives (see [php.ini directives](https://www.php.net/manual/ini.list.php)), you can't use the *.htaccess* approach. App Service provides a separate mechanism using the `PHP_INI_SCAN_DIR` app setting.

First, run the following command in the [Cloud Shell](https://shell.azure.com) to add an app setting called `PHP_INI_SCAN_DIR`:

```azurecli-interactive
az webapp config appsettings set --name <app-name> --resource-group <resource-group-name> --settings PHP_INI_SCAN_DIR="/usr/local/etc/php/conf.d:/home/site/ini"
```

`/usr/local/etc/php/conf.d` is the default directory where *php.ini* exists. `/home/site/ini` is the custom directory in which you'll add a custom *.ini* file. You separate the values with a `:`.

Navigate to the web SSH session with your Linux container (`https://cephalin-container.scm.azurewebsites.net/webssh/host`).

Create a directory in `/home/site` called `ini`, then create an *.ini* file in the `/home/site/ini` directory (for example, *settings.ini)* with the directives you want to customize. Use the same syntax you would use in a *php.ini* file. 

> [!TIP]
> In the built-in Linux containers in App Service, */home* is used as persisted shared storage. 
>

For example, to change the value of [expose_php](https://php.net/manual/ini.core.php#ini.expose-php) run the following commands:

```bash
cd /home/site
mkdir ini
echo "expose_php = Off" >> ini/setting.ini
```

For the changes to take effect, restart the app.

## Enable PHP extensions

The built-in PHP installations contain the most commonly used extensions. You can enable additional extensions in the same way that you [customize php.ini directives](#customize-php_ini_system-directives).

> [!NOTE]
> The best way to see the PHP version and the current *php.ini* configuration is to call [phpinfo()](https://php.net/manual/function.phpinfo.php) in your app.
>

To enable additional extensions, by following these steps:

Add a `bin` directory to the root directory of your app and put the `.so` extension files in it (for example, *mongodb.so*). Make sure that the extensions are compatible with the PHP version in Azure and are VC9 and non-thread-safe (nts) compatible.

Deploy your changes.

Follow the steps in [Customize PHP_INI_SYSTEM directives](#customize-php_ini_system-directives), add the extensions into the custom *.ini* file with the [extension](https://www.php.net/manual/ini.core.php#ini.extension) or [zend_extension](https://www.php.net/manual/ini.core.php#ini.zend-extension) directives.

```ini
extension=/home/site/wwwroot/bin/mongodb.so
zend_extension=/home/site/wwwroot/bin/xdebug.so
```

For the changes to take effect, restart the app.

## Access diagnostic logs

[!INCLUDE [Access diagnostic logs](../../../includes/app-service-web-logs-access-no-h.md)]

## Open SSH session in browser

[!INCLUDE [Open SSH session in browser](../../../includes/app-service-web-ssh-connect-builtin-no-h.md)]

## Troubleshooting

When a working PHP app behaves differently in App Service or has errors, try the following:

- [Access the log stream](#access-diagnostic-logs).
- Test the app locally in production mode. App Service runs your Node.js apps in production mode, so you need to make sure that your project works as expected in production mode locally. For example:
    - Depending on your *composer.json*, different packages may be installed for production mode (`require` vs. `require-dev`).
    - Certain web frameworks may deploy static files differently in production mode.
    - Certain web frameworks may use custom startup scripts when running in production mode.
- Run your app in App Service in debug mode. For example, in [Laravel](https://meanjs.org/), you can configure your app to output debug messages in production by [setting the `APP_DEBUG` app setting to `true`](../configure-common.md?toc=%2fazure%2fapp-service%2fcontainers%2ftoc.json#configure-app-settings).

### robots933456

You may see the following message in the container logs:

```
2019-04-08T14:07:56.641002476Z "-" - - [08/Apr/2019:14:07:56 +0000] "GET /robots933456.txt HTTP/1.1" 404 415 "-" "-"
```

You can safely ignore this message. `/robots933456.txt` is a dummy URL path that App Service uses to check if the container is capable of serving requests. A 404 response simply indicates that the path doesn't exist, but it lets App Service know that the container is healthy and ready to respond to requests.

## Next steps

> [!div class="nextstepaction"]
> [Tutorial: PHP app with MySQL](tutorial-php-mysql-app.md)

> [!div class="nextstepaction"]
> [App Service Linux FAQ](app-service-linux-faq.md)
