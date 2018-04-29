# Setting up Apache with Laravel Echo Server with ProxyPass

tl;dr: add `RewriteCond %{QUERY_STRING} transport=polling` inside VH config
before your `ProxyPass` directives to catch the `socket.io` polling


## Purpose
Example config files to hide your Laravel Echo Server behind a proxy, 
so that you don't have to open a port to your server just for this.

Note: _This is a SSL-less setup, I'll add a SSL config later_

## You'll need

- ability to
  - modify virtual host config
  - enable apache modules
  - restart apache
  - run command line on server
- `module_proxy_wstunnel` compatible Apache server
  - available officially since v2.4.5
  - can be compiled for older version if needed
- apache modules
  - [mod_proxy](https://httpd.apache.org/docs/2.4/mod/mod_proxy.html)
  - [mod_proxy_http](https://httpd.apache.org/docs/2.4/mod/mod_proxy_http.html)
  - [mod_proxy_wstunnel](https://httpd.apache.org/docs/2.4/mod/mod_proxy_wstunnel.html)


## Basically ..

Since Laravel Echo Server is an `socket.io` implementation a bit more config is
required in the virtual host config, than just your regular
`ProxyPass` / `ProxyPassReverse` directives

`socket.io` first polls the server whether it's valid and only then tries to make
the websocket connection. Except is does this on the same url, the only difference
is in the `transport` query parameter which changes from `polling` to `websocket`.
Because of this, you'll need to add the following to virtual host configuration

```
    RewriteEngine On
    RewriteCond %{QUERY_STRING} transport=polling
    RewriteRule /(.*) http://127.0.0.1:6001/$1 [P]
```
This is assuming your LES is setup to run on port `6001` and `/` path.
See the example configurations for details.


## Example configurations

For convenience/reference purposes I'm including a working example configuration
of everything required.

Note: _These are configs for Windows, but the extrapolation should be easily
made for other OS :)_

These configs assume a Laravel application running on `les.example.com` virtual
host with [broadcasting](https://laravel.com/docs/5.6/broadcasting) enabled
- `composer require predis/predis`
- `npm install --save laravel-echo` / `yarn add laravel-echo`
- `npm install --save socket.io-client` / `yarn add socket.io-client`
- uncomment the `BroadcastServiceProvider` inside your `config/app.php`
- for tests I'd recommend also
  - running `php artisan make:auth`
  - installing the `laravel/passport` package
  - [running associated commands](https://laravel.com/docs/5.6/passport)
  - create a dummy user


### `httpd.conf` Virtual Host config
```
<VirtualHost *:80>
    ServerAdmin webmaster@les.example.com
    DocumentRoot "C:/server/www/les/public/"
    ServerName les.example.com
    ErrorLog "D:/server/logs/les.example.com_error.log"
    CustomLog "D:/server/logs/les.example.com_access.log" combined

    RewriteEngine On
    RewriteCond %{QUERY_STRING} transport=polling
    RewriteRule /(.*) http://127.0.0.1:6001/$1 [P]

    ProxyPass /ws ws://127.0.0.1:6001/
    ProxyPassReverse /ws ws://127.0.0.1:6001/

    <Directory "C:/server/www/les/public/">
        Options Indexes FollowSymLinks MultiViews
        AllowOverride all
        Require all granted
    </Directory>
</VirtualHost>
```

### laravel-echo-server.json

[Follow the official instalation proccess](https://github.com/tlaverdure/Laravel-Echo-Server#getting-started),
next to nothing deviates from it. You can skip the API clients, these will come
from your laravel app.

Ususaly these two are enough to get you started
- `laravel-echo-server init`
  - fill in the config it asks you for
- `laravel-echo-server start`

An example json config file generated. I changed only one thing here, I added
the `"path" : "/"` to the `socketio: {}` portion of the config. THe rest is
left untouched.

```js
{
	"authHost": "http://les.example.com",
	"authEndpoint": "/broadcasting/auth",
	"clients": [
		{
			"appId": "[generated id]",
			"key": "[generated key]"
		}
	],
	"database": "redis",
	"databaseConfig": {
		"redis": {},
		"sqlite": {
			"databasePath": "/database/laravel-echo-server.sqlite"
		}
	},
	"devMode": true,
	"host": null,
	"port": "6001",
	"protocol": "http",
	"socketio": {
		"path" : "/"
	},
	"sslCertPath": "",
	"sslKeyPath": "",
	"sslCertChainPath": "",
	"sslPassphrase": "",
	"apiOriginAllow": {
		"allowCors": true,
		"allowOrigin": "http://les.example.com",
		"allowMethods": "GET, POST",
		"allowHeaders": "Origin, Content-Type, X-Auth-Token, X-Requested-With, Accept, Authorization, X-CSRF-TOKEN, X-Socket-Id"
	}
}

```

### resources/assets/js/bootstrap.js

Uncomment (or copy this) laravel Echo portion of the file and change the
broadcaster to socket.io and add the host part.

```js
import Echo from 'laravel-echo';

window.io = require('socket.io-client');

window.Echo = new Echo({
    broadcaster: 'socket.io',
    host       : {
        path: '/ws'
    }
});
```

### example Vue.js component

```js 
const app = new Vue({
    el: '#app',
    mounted() {
        window.Echo.private(`App.User.1`) // you'd supply an ID from the /api/user call for example :)
            .listen('.test', e => { // name of the event, see below
                console.log(e);// {now: 1563843453843.13543, user: "username", socket: null}
            });
        // testing laravel/passport :) 
        axios.get('/api/user') // Amm\User should use the HasApiTokens trait from passport for this to work!
            .then(e => {
                console.log(e); // return the user json with id, username, email, ...
            })
        // firing events to test the application example below
        setTimeout(() => {
            axios.get('/home/test')
                .then(e => {
                    console.log(e); // nothing
                })
        }, 5E3);
    }
});
```

## Example event

Just an example Event fireable by the laravel app. Example code to fire the 
event below ..

```php
<?php
      
      namespace App\Events;
      
      use Illuminate\Broadcasting\Channel;
      use Illuminate\Queue\SerializesModels;
      use Illuminate\Broadcasting\PrivateChannel;
      use Illuminate\Broadcasting\PresenceChannel;
      use Illuminate\Foundation\Events\Dispatchable;
      use Illuminate\Broadcasting\InteractsWithSockets;
      use Illuminate\Contracts\Broadcasting\ShouldBroadcast;
      use App\User;
      
      class Event implements ShouldBroadcast
      {
          use Dispatchable, InteractsWithSockets, SerializesModels;
      
          public $user;
      
          /**
           * Create a new event instance.
           *
           * @return void
           */
          public function __construct(User $user)
          {
              $this->user = $user;
          }
      
          /**
           * Get the channels the event should broadcast on.
           *
           * @return \Illuminate\Broadcasting\Channel|array
           */
          public function broadcastOn()
          {
              return new PrivateChannel('App.User.' . $this->user->id);
          }
      
          /**
           * The things that will be broadcasted to the redis -> les -> client
           *
           * @return array
           */
          public function broadcastWith()
          {
              return ['now' => microtime(true), 'user' => $this->user->username];
          }
      
          /**
           * Event name
           *
           * @return string
           */
          public static function broadcastAs()
          {
              return 'test';
          }
      }
```

I called it via adding this to the HomeController
```php

    public function test()
    {
        event(new Event(Auth::user()));
        sleep(5);
        event(new Event(Auth::user()));
    }
```