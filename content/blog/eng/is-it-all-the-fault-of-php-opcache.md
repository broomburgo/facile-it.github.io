---
authors: ["salvatore-cordiano"]
date: "2017-09-06"
draft: true
share: true
categories: [English, PHP, OPCache, realpath_cache]
title: "Is it all the fault of PHP OPCache?"
type: "post"
languageCode: "en-EN"
twitterImage: '/images/is-it-all-the-fault-of-php-opcache/share.jpg'
---

When I started my career as a developer I was very surprised reading the following sentence attributed to [Phil Karlton](https://martinfowler.com/bliki/TwoHardThings.html): _«There are only two hard things in Computer Science: **cache invalidation** and **naming things**»_. At the beginning, I was incredulous, because I didn't really understand the sense of these words. Not much later I started learning what it means. 

Without digging too much in the past, I'd like to talk about a recent cache issue we are experiencing on our production infrastructure. Particularly we saw a "strange behavior" after each deploy: immediately afterward a successful deployment procedure when we try to refresh pages changed with the new release we don't see the updated code but the previous code for a while. Actually, the scenario described above is very common with **PHP** web applications. We have seen this behavior in the past, but after we moved to our new production environment it makes this "phenomenon" known in a more evident way. Therefore, we started investigating on it.

Before going on it is essential to describe how our deployment procedure works.

# About deployment procedure

Our technology is mostly based on PHP and some framework like **Symfony** and **Zend Framework**. To put in production our code we use an internal project called **shark-do**. It was totally written by my team leader [Luca](https://www.linkedin.com/in/lucabo/).

Shark-do's philosophy is: _«If you can do it, you can do it in bash»_. It is a bash script which allows to define a task and to execute it from a recipe. Each project has its own recipe to manage different steps needed: for example, remove unuseful files, warmup cache, etc. We start our deployment procedure from a machine called "deploy" which is able to communicate with a "bastion" machine which in turn is able to communicate with the production environment.

Very often also more than 5 times a day, I wrote on the command line `shark-do deploy collaboratori` to start executing the task deploy for the project "collaboratori" in which I'm involved. 

The task deploy for a recipe like collaboratori will do generally the following steps:

1. pull from origin/master the last commit;
2. setup folders, remove the unnecessary file and start creating a release;
3. install parameters, run composer install, download and dump assets;
4. do cache warmup;
5. create a release archive, transfer and extract it on the bastion machine;
6. call an Ansible playbook to start release roll out using our infrastructure's REST API;
7. switch release, clean and remove old releases on the bastion machine;
8. tag the new release on New Relic and notify the end of task on Slack channel.

We need to focus on the point number 6 because it's the roll out. In that point, an Ansible procedure is responsible for copy the new release from the bastion machine on all the target machines (for example front-end and batch machines), for setting up folders and permission and for doing release switch. As described previously, each deployment procedure consists of many necessary activities, but the turning point is the change of the current project folder, it is usually done through the symlink swap from the previous release folder to the new one. The current project folder is the document root of the specific web application.

It is nothing but something like this:

```bash
ln -sf /var/www/{APP_NAME}/releases/@YYYYMMDDHHIISS /var/www/{APP_NAME}/current
```

The option `-s` is used to create a symbolic link, instead the option `-f` is used to force the symlink creation if the target already exists. `{APP_NAME}` represents the project's name.

Our deployment strategy is very common in the PHP world. We store multiple releases of the same application on the production machines and we use a symlink pointing the current version. In this manner, we try to deploy in an atomic and a safe way without impacts to production traffic.

Almost I forgot another important piece of the puzzle: we had about 15 front-end machines behind a load-balancer with round-robin workload balancing policy which is more than two times the previous number of servers.

Now the question is: what happens after release switch? I will explain in the next paragraphs.

# It is all the fault of PHP OPCache

Some caveats. The scope of this post is not to go deep in the execution flow of a PHP script, but to lay down the foundations to understand my reasoning. I am also considering only version 7 of PHP.

At this point it very useful to remember how PHP code is executed. When we run a PHP script, our source code undergoes four phases. 

![How does PHP work?](/images/is-it-all-the-fault-of-php-opcache/graph_1.png)
*How does PHP work?*

The first is managed by a lexer. The **PHP lexer** is responsible to match language keywords like for example `function`, `return`, `static` and so on and it will break up the code in pieces generally called tokens. Each token is often decorated with metadata necessary for the next step. 

The second step is managed by a parser. The **PHP parser** is responsible to analyze single or multiple tokens to match language structure patterns. For example, `$foo + 5` is recognized as a binary add operation and the variable `$foo` and the number `5` are recognized as operands. Here, the parser builds the **Abstract Syntax Tree** (AST) in a recursive way. Usually, lexer and parser are mentioned together as a single task.

The third step is the **compilation**. In this phase, the AST is visited and it is translated to an ordered sequence of OPCodes. Every OPCode could be considered as a low-level **Zend Virtual Machine** operation. The full list of OPCodes supported instruction is available [here](https://github.com/php/php-src/blob/php-7.0.0/Zend/zend_vm_opcodes.h).

The last step is the **execution**. The VM executes every single task described by OPCode and produce the result.

The first three stages of the above described "pipeline" (lexer, parser, and compiler), and particularly the third, take a lot of time and resources (memory and CPU). To minimize the weight of the compilation phase with PHP 5.5 was introduced the **Zend OPcache extension**. When enabled, the main goal of this extension is to cache the output of the compilation step (OPCodes) into shared memory (shm, mmap, etc.), so every PHP script is compiled only once and subsequently, different requests can be executed skipping the compilation task. If the source code on the non-development environment is not changing continuously the execution time of PHP should be reduced by a factor of at least two.

Not to leave out anything, it's important to remember that the OPcache extension is also responsible for the OPCodes optimization, but this topic is out of the scope of this post. 

In the light of the above, it seems that the strange behavior experienced in our production environment is all the fault of OPCache.

If our hypothesis is right, we should be able to reproduce the issue and after to solve it disabling OPCache extension. To do it I create a very simple demo environment using a **Docker** container with PHP 7.0 and Apache 2.4. The full code is available on GitHub [here](https://github.com/salvatorecordiano/php-realpath_cache-demo).

I create some shortcuts with the following bash scripts.

- `start.sh` this script is used to start the Docker container with the right configuration. 
- `release-switcher.sh` this script is used to swap the current release symlink every 10 seconds.
- `release-watcher.sh` this script is used to check the current release served by Apache every 1 second making an HTTP request.

To setup our test environment you have only to clone the GitHub repository. I'm assuming Docker is already installed on your machine.

```bash
git clone https://github.com/salvatorecordiano/php-realpath_cache-demo.git
cd php-realpath_cache-demo
```

To reproduce the cache issue you have to run in parallel the following commands using three different command line:

```bash
# start the container with production configuration
./start.sh production 
# start switching the current release
./release-switcher.sh
# start watching the current web server response 
./release-watcher.sh
```

The following video shows the output of execution.

{{< youtube uCm4OQwlv14 >}}

As expected we are experiencing the cache issue, indeed after a release switch, we haven't seen the right code as the output of an HTTP request.

We can disable OPCache extension and redo the same test.

```bash
# start the container with production configuration and opcache disabled
./start.sh production-no-opcache 
# start switching the current release
./release-switcher.sh
# start watching the current web server response 
./release-watcher.sh
```

The following video shows the output of this new execution.

{{< youtube ukp9XRTjH_s >}}

Actually, something went wrong. We are experiencing the previous behavior. Ergo, our reasoning is missing something and it isn't all the fault of OPCache.

# realpath_cache: the true culprit

When we use `include/require` functions or when we use autoload on PHP we need to think immediately to **realpath_cache**. Realpath_cache is a PHP feature that allows to cache files and folders path resolution to minimize time-consuming disk lookups and to improve performances. This is very useful when we work with many vendors or frameworks like for example Symfony, Zend or Laravel because they use a huge number of files.

The path cache mechanism was introduced in PHP 5.1.0. At the moment the official docs is not mentioning or explaining realpath_cache except for the functions `realpath_cache_get()`, `realpath_cache_size()`, `clearstatcache()` and the `php.ini` parameters `realpath_cache_size` and `realpath_cache_ttl`. 

On the web, the only one reference is an [old post](http://jpauli.github.io/2014/06/30/realpath-cache.html) written by **Julien Pauli** on 2014. In his post Pauli, a well-known PHP contributor, explains how PHP resolves a path behind the scenes.

When we access to a file in PHP, it tries to resolve the file path using a Unix system call called `stat()`. Stat returns file attributes (for example file system permission, filename extensions, and other metadata) about an **inode**. In the Unix world, an inode is a data structure used to describe a file system object such as a file or a directory. PHP puts the result of the system call in a data structure called `realpath_cache_bucket` excluding permissions, owners, etc. So if we retry to access to the same file the lookup on the bucket will avoid having a new heavy system call. In order to deepen the knowledge, I suggest reading the PHP source code [here](https://github.com/php/php-src/blob/php-7.0.0/Zend/zend_virtual_cwd.c).

The function `realpath_cache_get` was introduced with PHP 5.3.2 and it allows to get an array of all the real path cache entries. Each element of the array has as key the resolved path and as value, an array of data like `key`, `is_dir`, `realpath`, `expires`. 

The following array is the output of `print_r(realpath_cache_get());` on our Docker test environment.

```PHP
Array
(
    [/var/www/html] => Array
        (
            [key] => 1438560323331296433
            [is_dir] => 1
            [realpath] => /var/www/html
            [expires] => 1504549899
        )
    [/var/www] => Array
        (
            [key] => 1.5408950988325E+19
            [is_dir] => 1
            [realpath] => /var/www
            [expires] => 1504549899
        )
    [/var] => Array
        (
            [key] => 1.6710127960665E+19
            [is_dir] => 1
            [realpath] => /var
            [expires] => 1504549899
        )
    [/var/www/html/release1] => Array
        (
            [key] => 7631224517412515240
            [is_dir] => 1
            [realpath] => /var/www/html/release1
            [expires] => 1504549899
        )
    [/var/www/current] => Array
        (
            [key] => 1.7062595747834E+19
            [is_dir] => 1
            [realpath] => /var/www/html/release1
            [expires] => 1504549899
        )
    [/var/www/current/index.php] => Array
        (
            [key] => 6899135167081162414
            [is_dir] => 
            [realpath] => /var/www/html/release1/index.php
            [expires] => 1504549899
        )
)
```

Particularly we can say:

- `key` is a float. It's a hash associated with the path.
- `is_dir` is a boolean. It is true when the resolved path is a directory, otherwise it's false.
- `realpath` is a string. It is the resolved path.
- `expires` is an integer. It represents the timestamp after which the path in the cache will be invalidated. This value is strictly related to the parameter `realpath_cache_ttl`.

In the previous sample, we have 6 paths, but all are related with the resolution of the path `/var/www/current/index.php`. PHP has created 6 cache keys to resolve only one path. From this follows that the path resolution is made splitting a path in part and resolving it. In our case the above real path is `/var/www/html/release1/index.php` because `/var/www/current` is a symlink to the folder `/var/www/html/release1`.

Julien Pauli's post also specifies: _«The realpath cache is process bound, and not shared into shared memory»_. This means that cache must expire for every PHP process, for that reason if we are using **PHP-FPM** to clean the whole web server, we need to wait the cache expiration for every worker of the pool.

This last sentence is very useful to understand what happens during our test using the configuration `production-no-opcache`. Even if OPCache is disabled after the symlink swap PHP will start noticing every PHP process of the paths expiration slowly.

In our real production environment, we need to consider that we have 15 front-end machines and every machine have about 35 PHP-FPM worker polls + 1 master process. This explains why in the new environment the "strange behavior" is more evident. 

We can tune the real path cache impact on our web application using the above mentioned the parameters `realpath_cache_size` and `realpath_cache_ttl`.

`realpath_cache_size` determines the size of the real path bucket to be used by PHP. It is an integer and it is useful to increment this value if our web application uses a huge number of files.

The other configuration directive `realpath_cache_ttl`, as already mentioned, represents the duration of time in seconds for which to cache real path information.

Now we had the full picture of the problem and we can re-enable OPCache extension and disable real path cache setting up size and time to live (TTL) as described below:

```bash
realpath_cache_size=0k
realpath_cache_ttl=-1
```

It's time to do our last (I hope!) test.

```bash
# start the container with production configuration, opcache enabled and realpath_cache disabled
./start.sh production-no-realpath-cache 
# start switching the current release
./release-switcher.sh
# start watching the current web server response 
./release-watcher.sh
```

The following video shows the output of the last execution.

{{< youtube AzeTtOUTcc4 >}}

# Conclusions

At the end of the day, the scope of this post was to unveil the mystery about our cache issue and to share what I learned about OPCache and real path cache and their differences.

In the post, I miss underlining a very important assumption of our deployment strategy. When we deploy our code, we need to be sure (we try :-D) that two contiguous releases are compatible. Think what can happen if a request that starts on one version of the code tries to access to other files during its execution and the files are updated, moved or deleted. If you are not able to guarantee the compatibility between releases, it's necessary to implement an atomic deployment strategy, in the strict sense of the word. This could be reached for example using containers or more simply using an isolated memory pool for each release deployed.

Another way to avoid the symlinks update, in the middle of ongoing requests, is to give to the web server the responsibility to resolve the symlink and to assign it to `DOCUMENT_ROOT`. After it's needed to base all `include/require` and autoload on `DOCUMENT_ROOT`. The same result could be achieved at the PHP level in the application front controller by defining the base root via `realpath(__FILE__)`.
