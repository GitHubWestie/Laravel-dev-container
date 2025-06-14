# Laravel-dev-container
A guide on creating a functional containerised Laravel environment using devcontainers in VSCode.

This guide assumes you have `VS Code`, `Docker`, and the `Dev Containers` extension installed.

## Add Dev Container configuration:
* In VS Code open the Command Palette (`Ctrl+Shift+P` or `Cmd+Shift+P`) or click the remote connections icon in the bottom left of the window.
* Search for Dev Containers: Add Dev Container Configuration Files....
* Select PHP.
* VS Code will generate a .devcontainer folder with a Dockerfile and devcontainer.json.

## Create and Configure Dockerfile
* Create a `Dockerfile` inside the .devcontainer directory
* Give it the following content
```dockerfile
# .devcontainer/Dockerfile

# Use the same base image as specified in your devcontainer.json (e.g., PHP 8.2 Bullseye)
FROM mcr.microsoft.com/devcontainers/php:1-8.2-bullseye

# Install the pcntl extension using Docker's official PHP extension installer
# Add any other required PHP extensions here (e.g., pdo_mysql, gd, etc.)
RUN docker-php-ext-install pcntl pdo_mysql # Add pdo_mysql if you need database connectivity

# You can also add other OS-level packages if needed
# RUN apt-get update && apt-get install -y <package-name> && rm -rf /var/lib/apt/lists/*
```

## Create Custom Xdebug Config File
* Create `xdebug-custom.ini` in the .devcontainer directory
* Give it the following content
```ini
; .devcontainer/xdebug-custom.ini

; Loads the Xdebug extension - VERIFY THIS PATH
zend_extension=/usr/local/lib/php/extensions/no-debug-non-zts-20220829/xdebug.so

; Xdebug 3.x configuration
xdebug.mode = debug
xdebug.start_with_request = trigger ; Recommended for development
xdebug.client_host = host.docker.internal ; Directs Xdebug to your host machine
xdebug.client_port = 9003 ; Xdebug 3.x default debugging port
```

## Configure `devcontainer.json`
Update the devcontainer.json file as follows
```js
// .devcontainer/devcontainer.json
{
    "name": "PHP Laravel Development",
    // Point to your custom Dockerfile
    "build": {
        "dockerfile": "Dockerfile",
        "context": "." // Context is the current directory (.devcontainer)
    },

    // Features to add to the dev container.
    "features": {
        // Node.js feature for npm
        "ghcr.io/devcontainers/features/node:1": {
            "version": "lts" // Or "latest", or a specific version like "18", "20"
        }
        // IMPORTANT: Do NOT include the 'php' feature here, as we're managing extensions in Dockerfile
    },

    // Forward ports from the container to your host machine
    "forwardPorts": [8000, 5173, 9003],
    "portsAttributes": {
        "5173": {
            "label": "Vite Dev Server",
            "onAutoForward": "notify"
        },
        "9003": {
            "label": "Xdebug",
            "onAutoForward": "notify"
        }
    },
    "customizations": {
        "vscode": {
            "extensions": [
                "bmewburn.vscode-intelephense-client", // PHP IntelliSense
                "felixfbecker.php-debug", // PHP Debug Adapter for Xdebug
                "onecentlin.laravel-blade", // Blade syntax highlighting
                "shufo.vscode-blade-formatter" // Blade formatter
            ]
        }
    },

    // Commands to run after the container is created.
    // This ensures Laravel installer is available and Xdebug config is applied.
    "postCreateCommand": "bash -c \"/usr/local/bin/composer global require laravel/installer && echo 'export PATH=\\\"$PATH:$HOME/.composer/vendor/bin\\\"' >> ~/.bashrc && source ~/.bashrc && sudo cp .devcontainer/xdebug-custom.ini /usr/local/etc/php/conf.d/xdebug.ini\"",
    // ^^^ Ensure '/usr/local/etc/php/conf.d/xdebug.ini' is the correct target for overwriting

    // Uncomment to connect as root instead (generally not recommended for daily dev).
    // "remoteUser": "root"
}
```
*Ensure `'/usr/local/etc/php/conf.d/xdebug.ini'` is the correct target for overwriting. If unsure run `php --ini` in the terminal and this should show the correct path*

After updating `devcontainer.json`, VSCode should prompt you to rebuild the container. If it doesn't click on the remote connection icon and select rebuild container or open the Command Palette (`Ctrl+Shift+P` or `Cmd+Shift+P`) and search Dev Container: Rebuild container...

## Configure Vite
* Once the container has rebuilt open a ***new*** terminal and run `laravel new` to initiate the interactive Laravel project creation process.
* Once built open `vite.config.js` and update it with the following `server config`
```js
// vite.config.js
import { defineConfig } from 'vite';
import laravel from 'laravel-vite-plugin';

export default defineConfig({
    plugins: [
        laravel({
            input: [
                'resources/css/app.css',
                'resources/js/app.js',
            ],
            refresh: true,
        }),
    ],
    server: {
        host: '0.0.0.0', // Listen on all interfaces inside the container
        port: 5173,    // Explicit Vite dev server port
        hmr: {
            host: 'localhost', // Public URL for HMR connections (your host machine's localhost)
            port: 5173,
            protocol: 'ws' // Use ws for websocket connection
        },
        watch: {
            usePolling: true // Essential for file watching in Docker containers
        }
    },
});
```
*If Laravel has already created a hot config file in the public directory delete it `rm public/hot`*

## Serve your App
* In the terminal:
  * `cd your-laravel-project`
  * `composer run dev`
 
By now you should have a fully functional Laravel 12 project capable of running in a container with hot reload.

### Optional Extra: Configure VS Code for Xdebug
If you want to check the VSCode configuration for Xdebug follow these steps but in testing I have found VSCode will follow the custom ini file setup earlier
* Open the "Run and Debug" view (Ctrl+Shift+D or Cmd+Shift+D).
* Click the gear icon to open launch.json.
* Ensure your PHP debug configuration has port 9003:
```js
// .vscode/launch.json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Listen for Xdebug",
            "type": "php",
            "request": "launch",
            "port": 9003 // Ensure this is 9003 for Xdebug 3.x
        },
        {
            "name": "Launch currently open script",
            "type": "php",
            "request": "launch",
            "program": "${file}",
            "cwd": "${fileDirname}",
            "port": 9003
        }
    ]
}
```

#### So far I have only tested this with a basic Laravel 12 project i.e. no Vue/React and just a sqlite database but they might work too.
