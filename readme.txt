Using NGINX, PHP-FPM+APC and Varnish to make WordPress Websites fly
by Tobias Baldauf
WordPress is one of the most popular content platforms on the Internet. It powers the majority of all freshly released websites and has a huge user-community. Running a WordPress site enables editors to easily and regularly publish content which might very well end up on Hacker News – so let us make sure the web server is up to its job!

This setup will easily let you serve hundreds of thousands of users per day, including brief peaks of up to 100 concurrent users on a ~15$/month machine with only 2 cores and 2GB of RAM.


Outrageous Assumptions

This guide assumes that you have a working knowledge of content management systems, web servers and their configurations. Additionally, you should be familiar with installing, starting and stopping services on a remote server via ssh. In short: if you know how to work a CLI, you’ll be fine.

Step 0/9: Use the Source, or: Enjoy the Repository

If you don’t need to adhere to a strict company security policy, use the Dotdeb repository for Debian to gain access to newer versions of NGINX and PHP. For Varnish, get the Varnish Debian repository. They are the easiest to use.

Step 1/9: The Little Engine that Could

Let’s start by firing up your favorite package management tool, potentially with Super Cow Powers, and install NGINX. The following configuration optimizes NGINX for WordPress. Put it into /etc/nginx/nginx.conf to make NGINX use both CPU cores, do proper Gzipping and more. Note: single-line configurations shown here may extend over multiple lines for readability – take care when copying.

user www-data;
worker_processes 2;
pid /var/run/nginx.pid;
 
events {
    worker_connections 768;
    multi_accept on;
    use epoll;
}
 
http {
 
    # Let NGINX get the real client IP for its access logs
    set_real_ip_from 127.0.0.1;
    real_ip_header X-Forwarded-For;
 
    # Basic Settings
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 20;
    client_max_body_size 15m;
    client_body_timeout 60;
    client_header_timeout 60;
    client_body_buffer_size  1K;
    client_header_buffer_size 1k;
    large_client_header_buffers 4 8k;
    send_timeout 60;
    reset_timedout_connection on;
    types_hash_max_size 2048;
    server_tokens off;
 
    # server_names_hash_bucket_size 64;
    # server_name_in_redirect off;
 
    include /etc/nginx/mime.types;
    default_type application/octet-stream;
 
    # Logging Settings
    # access_log /var/log/nginx/access.log;
    error_log /var/log/nginx/error.log;
 
    # Log Format
    log_format main '$remote_addr - $remote_user [$time_local] '
    '"$request" $status $body_bytes_sent "$http_referer" '
    '"$http_user_agent" "$http_x_forwarded_for"';
 
    # Gzip Settings
    gzip on;
    gzip_static on;
    gzip_disable "msie6";
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_min_length 512;
    gzip_buffers 16 8k;
    gzip_http_version 1.1;
    gzip_types text/css text/javascript text/xml text/plain text/x-component 
    application/javascript application/x-javascript application/json 
    application/xml  application/rss+xml font/truetype application/x-font-ttf 
    font/opentype application/vnd.ms-fontobject image/svg+xml;
 
    # Virtual Host Configs
    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/sites-enabled/*;
}
In case your site will make use of custom webfonts, NGINX may need some help to deliver them with the correct MIME type – otherwise it will default to application/octet-stream. Edit /etc/nginx/mime.types and check that the webfont types are set to the following:

    application/vnd.ms-fontobject           eot;
    application/x-font-ttf                  ttf;
    font/opentype                           ott;
    application/font-woff                   woff;
Step 2/9: Who Enters my Domain?

Now that you have done the basic setup for NGNIX, let’s make it play nicely with your domain: remove the symlink to default from /etc/nginx/sites-enabled/, create a new configuration file at /etc/nginx/sites-available/ named yourdomain.tld and symlink to it at /etc/nginx/sites-enabled/yourdomain.tld. NGINX will use the new configuration file which we will fill now. Put the following into /etc/nginx/sites-available/yourdomain.tld:

server {
    # Default server block blacklisting all unconfigured access
    listen [::]:8080 default_server;
    server_name _;
    return 444;
}
 
server {
    # Configure the domain that will run WordPress
    server_name yourdomain.tld;
    listen [::]:8080 deferred;
    port_in_redirect off;
    server_tokens off;
    autoindex off;
 
    client_max_body_size 15m;
    client_body_buffer_size 128k;
 
    # WordPress needs to be in the webroot of /var/www/ in this case
    root /var/www;
    index index.html index.htm index.php;
    try_files $uri $uri/ /index.php?q=$uri&amp;$args;
 
    # Define default caching of 24h
    expires 86400s;
    add_header Pragma public;
    add_header Cache-Control "max-age=86400, public, must-revalidate, proxy-revalidate";
 
    # deliver a static 404
    error_page 404 /404.html;
    location  /404.html {
        internal;
    }
 
    # Deliver 404 instead of 403 "Forbidden"
    error_page 403 = 404;
 
    # Do not allow access to files giving away your WordPress version
    location ~ /(\.|wp-config.php|readme.html|licence.txt) {
        return 404;
    }
 
    # Add trailing slash to */wp-admin requests.
    rewrite /wp-admin$ $scheme://$host$uri/ permanent;
 
    # Don't log robots.txt requests
    location = /robots.txt {
        allow all;
        log_not_found off;
        access_log off;
    }
 
    # Rewrite for versioned CSS+JS via filemtime
    location ~* ^.+\.(css|js)$ {
        rewrite ^(.+)\.(\d+)\.(css|js)$ $1.$3 last;
        expires 31536000s;
        access_log off;
        log_not_found off;
        add_header Pragma public;
        add_header Cache-Control "max-age=31536000, public";
    }
 
    # Aggressive caching for static files
    # If you alter static files often, please use 
    # add_header Cache-Control "max-age=31536000, public, must-revalidate, proxy-revalidate";
    location ~* \.(asf|asx|wax|wmv|wmx|avi|bmp|class|divx|doc|docx|eot|exe|
    gif|gz|gzip|ico|jpg|jpeg|jpe|mdb|mid|midi|mov|qt|mp3|m4a|mp4|m4v|mpeg|
    mpg|mpe|mpp|odb|odc|odf|odg|odp|ods|odt|ogg|ogv|otf|pdf|png|pot|pps|
    ppt|pptx|ra|ram|svg|svgz|swf|tar|t?gz|tif|tiff|ttf|wav|webm|wma|woff|
    wri|xla|xls|xlsx|xlt|xlw|zip)$ {
        expires 31536000s;
        access_log off;
        log_not_found off;
        add_header Pragma public;
        add_header Cache-Control "max-age=31536000, public";
    }
 
    # pass PHP scripts to Fastcgi listening on Unix socket
    # Do not process them if inside WP uploads directory
    # If using Multisite or a custom uploads directory,
    # please set the */uploads/* directory in the regex below
    location ~* (^(?!(?:(?!(php|inc)).)*/uploads/).*?(php)) {
        try_files $uri = 404;
        fastcgi_split_path_info ^(.+.php)(.*)$;
        fastcgi_pass unix:/var/run/php-fpm.socket;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
        fastcgi_intercept_errors on;
        fastcgi_ignore_client_abort off;
        fastcgi_connect_timeout 60;
        fastcgi_send_timeout 180;
        fastcgi_read_timeout 180;
        fastcgi_buffer_size 128k;
        fastcgi_buffers 4 256k;
        fastcgi_busy_buffers_size 256k;
        fastcgi_temp_file_write_size 256k;
    }
 
    # Deny access to hidden files
    location ~ /\. {
        deny all;
        access_log off;
        log_not_found off;
    }
 
}
 
# Redirect all www. queries to non-www
# Change in case your site is to be available at "www.yourdomain.tld"
server {
    listen [::]:8080;
    server_name www.yourdomain.tld;
    rewrite ^ $scheme://yourdomain.tld$request_uri? permanent;
}
This configuration is for a single-site WordPress install at /var/www/ webroot with a media upload limit of 15MB. Read the comments within the configuration if you want to change things: nginx -t is your friend when altering configuration files.

Step 3/9: Let’s get the Elephant into the Room

Next, we’ll get PHP-FPM onboard. Install it any way you like, preferably PHP version 5.4. Just make sure you get APC as well. After successful installation, we’ll need to edit several configuration files. Let’s start by editing the well-known /etc/php5/fpm/php.ini. It’s a huge file, so instead of showing the entire file here, please go through the following configuration line by line and set them to the respective values in your php.ini:

short_open_tag = Off
ignore_user_abort = Off
post_max_size = 15M
upload_max_filesize = 15M
default_charset = "UTF-8"
allow_url_fopen = Off
default_socket_timeout = 30
mysql.allow_persistent = Off
At the very end of your php.ini, add the following block which will configure APC, the opcode cache:

[apc]
apc.stat = "0"
apc.max_file_size = "1M"
apc.localcache = "1"
apc.localcache.size = "256"
apc.shm_segments = "1"
apc.ttl = "3600"
apc.user_ttl = "7200"
apc.gc_ttl = "3600"
apc.cache_by_default = "1"
apc.filters = ""
apc.write_lock = "1"
apc.num_files_hint= "512"
apc.user_entries_hint="4096"
apc.shm_size = "256M"
apc.mmap_file_mask=/tmp/apc.XXXXXX
apc.include_once_override = "0"
apc.file_update_protection="2"
apc.canonicalize = "1"
apc.report_autofilter="0"
apc.stat_ctime="0"
;This should be used when you are finished with PHP file changes.
;As you must clear the APC cache to recompile already cached files.
;If you are still developing, set this to 1.
apc.stat="0"
In your /etc/php5/fpm/php-fpm.conf, please set the following lines to their respective values:

pid = /var/run/php5-fpm.pid
error_log = /var/log/php5-fpm.log
emergency_restart_threshold = 5
emergency_restart_interval = 2
events.mechanism = epoll
Finally, in /etc/php5/fpm/pool.d/www.conf, do the same procedure again:

user = www-data
group = www-data
listen = /var/run/php-fpm.socket
listen.owner = www-data
listen.group = www-data
listen.mode = 0666
listen.allowed_clients = 127.0.0.1
pm = dynamic
pm.max_children = 50
pm.start_servers = 15
pm.min_spare_servers = 5
pm.max_spare_servers = 25
pm.process_idle_timeout = 60s
request_terminate_timeout = 30
security.limit_extensions = .php
and add the following to the end of it:

php_flag[display_errors] = off
php_admin_value[error_reporting] = 0
php_admin_value[error_log] = /var/log/php5-fpm.log
php_admin_flag[log_errors] = on
php_admin_value[memory_limit] = 128M
Kudos! You have just configured PHP to run on a high performance UNIX socket, spawn child processes to deal with requests and cache everything so that there’s less load on the system.

Step 4/9: Give Varnish a Polish

At this point, you already have a lean, mean webserving machine – albeit on port 8080. Now imagine ~95% of all your incoming traffic hitting an additional layer above which keeps static content in RAM. PHP won’t even have to process anything most of the time and your database will receive more load from editors adding new content than from queries caused by the frontend. That’s worth the extra mile, right? So let’s go install Varnish!

After installing, edit /etc/default/varnish to make it use the two cores, static content in RAM and proper timeouts:

DAEMON_OPTS="-a :80 \
    -T localhost:6082 \
    -f /etc/varnish/default.vcl \
    -u www-data -g www-data \
    -S /etc/varnish/secret \
    -p thread_pools=2 \
    -p thread_pool_min=25 \
    -p thread_pool_max=250 \
    -p thread_pool_add_delay=2 \
    -p session_linger=50 \
    -p sess_workspace=262144 \
    -p cli_timeout=40 \
    -s malloc,768m"
And for the very last step in this web server stack setup, put the following configuration into /etc/varnish/default – it’s well commented so you can see what each part does:

# We only have one backend to define: NGINX
backend default {
    .host = "127.0.0.1";
    .port = "8080";
}
 
# Only allow purging from specific IPs      
acl purge {
    "localhost";
    "127.0.0.1";
}
 
sub vcl_recv {
    # Handle compression correctly. Different browsers send different
    # "Accept-Encoding" headers, even though they mostly support the same
    # compression mechanisms. By consolidating compression headers into
    # a consistent format, we reduce the cache size and get more hits.
    # @see: http:// varnish.projects.linpro.no/wiki/FAQ/Compression
    if (req.http.Accept-Encoding) {
        if (req.http.Accept-Encoding ~ "gzip") {
            # If the browser supports it, we'll use gzip.
            set req.http.Accept-Encoding = "gzip";
        }
        else if (req.http.Accept-Encoding ~ "deflate") {
            # Next, try deflate if it is supported.
            set req.http.Accept-Encoding = "deflate";
        }
        else {
            # Unknown algorithm. Remove it and send unencoded.
            unset req.http.Accept-Encoding;
        }
    }
 
    # Set client IP
    if (req.http.x-forwarded-for) {
        set req.http.X-Forwarded-For =
        req.http.X-Forwarded-For + ", " + client.ip;
    } else {
        set req.http.X-Forwarded-For = client.ip;
    }
 
    # Check if we may purge (only localhost)
    if (req.request == "PURGE") {
        if (!client.ip ~ purge) {
            error 405 "Not allowed.";
        }
        return(lookup);
    }
 
    if (req.request != "GET" &amp;&amp;
        req.request != "HEAD" &amp;&amp;
        req.request != "PUT" &amp;&amp;
        req.request != "POST" &amp;&amp;
        req.request != "TRACE" &amp;&amp;
        req.request != "OPTIONS" &amp;&amp;
        req.request != "DELETE") {
            # /* Non-RFC2616 or CONNECT which is weird. */
            return (pipe);
    }
 
    if (req.request != "GET" &amp;&amp; req.request != "HEAD") {
        # /* We only deal with GET and HEAD by default */
        return (pass);
    }
 
    # admin users always miss the cache
    if( req.url ~ "^/wp-(login|admin)" || 
        req.http.Cookie ~ "wordpress_logged_in_" ){
            return (pass);
    }
 
    # Remove cookies set by Google Analytics (pattern: '__utmABC')
    if (req.http.Cookie) {
        set req.http.Cookie = regsuball(req.http.Cookie,
            "(^|; ) *__utm.=[^;]+;? *", "\1");
        if (req.http.Cookie == "") {
            remove req.http.Cookie;
        }
    }
 
    # always pass through POST requests and those with basic auth
    if (req.http.Authorization || req.request == "POST") {
        return (pass);
    }
 
    # Do not cache these paths
    if (req.url ~ "^/wp-cron\.php$" ||
        req.url ~ "^/xmlrpc\.php$" ||
        req.url ~ "^/wp-admin/.*$" ||
        req.url ~ "^/wp-includes/.*$" ||
        req.url ~ "\?s=") {
            return (pass);
    }
 
    # Define the default grace period to serve cached content
    set req.grace = 30s;
 
    # By ignoring any other cookies, it is now ok to get a page
    unset req.http.Cookie;
    return (lookup);
}
 
sub vcl_fetch {
    # remove some headers we never want to see
    unset beresp.http.Server;
    unset beresp.http.X-Powered-By;
 
    # only allow cookies to be set if we're in admin area
    if( beresp.http.Set-Cookie &amp;&amp; req.url !~ "^/wp-(login|admin)" ){
        unset beresp.http.Set-Cookie;
    }
 
    # don't cache response to posted requests or those with basic auth
    if ( req.request == "POST" || req.http.Authorization ) {
        return (hit_for_pass);
    }
 
    # don't cache search results
    if( req.url ~ "\?s=" ){
        return (hit_for_pass);
    }
 
    # only cache status ok
    if ( beresp.status != 200 ) {
        return (hit_for_pass);
    }
 
    # If our backend returns 5xx status this will reset the grace time
    # set in vcl_recv so that cached content will be served and 
    # the unhealthy backend will not be hammered by requests
    if (beresp.status == 500) {
        set beresp.grace = 60s;
        return (restart);
    }
 
    # GZip the cached content if possible
    if (beresp.http.content-type ~ "text") {
        set beresp.do_gzip = true;
    }
 
    # if nothing abovce matched it is now ok to cache the response
    set beresp.ttl = 24h;
    return (deliver);
}
 
sub vcl_deliver {
    # remove some headers added by varnish
    unset resp.http.Via;
    unset resp.http.X-Varnish;
}
 
sub vcl_hit {
    # Set up invalidation of the cache so purging gets done properly
    if (req.request == "PURGE") {
        purge;
        error 200 "Purged.";
    }
    return (deliver);
}
 
sub vcl_miss {
    # Set up invalidation of the cache so purging gets done properly
    if (req.request == "PURGE") {
        purge;
        error 200 "Purged.";
    }
    return (fetch);
}
 
sub vcl_error {
    if (obj.status == 503) {
                # set obj.http.location = req.http.Location;
                set obj.status = 404;
        set obj.response = "Not Found";
                return (deliver);
    }
}
This configuration will deliver high speed, gzipped content, caching aggressively while letting authenticated backend users view the live site uncached. It also protects the stack from unwelcome traffic while allowing cache purges from certain sources.

Step 5/9: Code Is Poetry

If you didn’t already have WordPress installed before, do it now. There’s plenty of good guidance on this already. Should you come across odd reloads of the same step during install, restart the three services NGINX, PHP-FPM and Varnish on your machine.

After successful installation of WordPress, we’ll need to make it talk to the high performance webserver stack we’ve just built for it. Let’s start by reading, understanding and installing the NGINX compatibility plugin for WordPress. Don’t forget to follow the instructions on the plugin site. Now, WordPress can deal with NGINX.

Next, let’s make WordPress work with PHP’s opcode cache APC: Read, understand and then install the Object-Cache plugin for WordPress.

The final step is to empower WordPress so it can purge the Varnish cache. Again, read, understand and only then install Pål-Kristian Hamre’s Varnish plugin for WordPress. Afterwards, go to its configuration page within the WordPress admin-interface and set the following parameters:

Varnish Administration IP Address: 127.0.0.1
Varnish Administration Port: 80
Varnish Secret: (get it at /etc/varnish/secret)
Check: "Also purge all page navigation" and "Also purge all comment navigation"
Varnish Version: 3
And that’s all concerning the high performance webserver stack, folks! Your WordPress will now handle a great deal of traffic while keeping TTFB to a minimum. The times of high CPU load and disk I/O troubles are now past.

The Performance Golden Rule

Steve Souders’ Performance Golden Rule clearly states that most performance gains can be achieved by optimizing the frontend side of things instead of scaling the backend to be able to deal with a huge amount of HTTP request. Or, as Patrick Meenan puts it, one should always question “the need to make the requests in the first place”. So let’s not stop at our high performance webserver, but continue with a high performance frontend!

Step 6/9: Bust that Cache Wide Open!

If you have read the configuration files for NGINX, you will have noticed some lines about a cache-buster for CSS and JS files. What it does is altering the link to your theme’s unified stylesheet and JS to something like style.1352707822.css, in which the number is the filemtime. So whenever you alter your files, their apparent filename within the site will change and clients will download the newest version of those files while you don’t need to alter any paths manually. Simply place the following into your theme’s functions.php:

/**
 * Automated cache-buster function via filemtime
 **/
function autoVer($url){
  $name = explode('.', $url);
  $lastext = array_pop($name);
  array_push(
    $name, 
    filemtime($_SERVER['DOCUMENT_ROOT'] . parse_url($url, PHP_URL_PATH)), 
    $lastext);
  echo implode('.', $name) ;
}
Now use the autoVer function when including CSS and JS, e.g. in your functions.php (see wp_register_style) or header.php. Examples:

# Add in your theme's functions.php:
wp_register_style(
  'style', 
  get_bloginfo('stylesheet_directory') . autoVer('style.css'), 
  false, 
  NULL, 
  'all');
 
# Add in your theme's header.php:
autoVer(get_bloginfo('stylesheet_url'))
Step 7/9: Avoid the wicked @import for WordPress child themes

WordPress child themes rely on @import in their CSS file to get the styles from their parent theme. Sadly, @import is a wicked load-blocker. Here’s a simple trick to have two stylesheets included via link in the website’s header if it’s a child theme because stylesheets called via link can download in parallel. Remove the @import from your child-theme’s CSS file and place the following PHP snippet into your header.php when linking your stylesheets:

<?php
    /*
     * Circumvent @import CSS for WordPress child themes
     * If we're in a child theme, build links for both parent and child CSS
     * This way, we can remove the @import from the child theme's style.css
     * CSS loaded via link can load simultaneously, while @import blocks loading
     * See: http://www.stevesouders.com/blog/2009/04/09/dont-use-import/
     */
    if(is_child_theme()) {
        echo '<link rel="stylesheet" href="';
        autoVer(get_bloginfo('template_url').'/style.css');
        echo '" />'."\n\t\t";
        echo '<link rel="stylesheet" href="';
        autoVer(get_bloginfo('stylesheet_url'));
        echo '" />'."\n";
    } else {
        echo '<link rel="stylesheet" href="';
        autoVer(get_bloginfo('stylesheet_url'));
        echo '" />'."\n";
    } 
?>
Additionally, you should use @media print within your unified stylesheet because, as Stoyan Stefanov has pointed out, browsers will always download all stylesheets no matter the medium.

Step 8/9: Unification Day

After combining your CSS into a unified stylesheet and making it load in a non-blocking manner, you should think about minifying it. There are plenty of decent minification tools around.

Also, you can enable loading JS libraries from Google and other CDNs there. No matter which libraries you use, unify and compress you local JS with e.g. Google Closure Compiler. If you’re using Google Analytics, check out this excellent GA snippet by Mathias Bynens which you can easily integrate into your unified JS. If you’re not using jQuery on the frontend, deregister its default inclusion via your theme’s “functions.php”.

The last thing to be minified is the website source code outputted by WordPress. There are several minification plugins either as standalone or integrated into WP Super Cache or WP Total Cache plugins. I recommend “WP HTML Compression”, especially when used together with the Roots Theme

Step 9/9: Famous Last Words

In case the imperfections of the code outputted by WordPress keep you up at night, your performance optimizations don’t have to end here. Check out the Roots Theme, which finally brings decent relative URL structures, clean navigation menus and much more to WordPress. It’s not as easy as installing a common theme or plugin, but worth it.

If you ever dreamed of using Nicole Sullivan’s excellent OOCSS high performance CSS selectors with WordPress, check out the PHP DOMDocument parser to filter the output of WordPress and set classes for all relevant DOM objects in your source so you can adhere to the OOCSS code standards. While parsing the entire website output during creation by WordPress may seem excessive, consider that it finally gives you total control over syntax and defined classes while editors can use the familiar WordPress interface to manage content. And with such a mighty & multi-layered caching system in place, the frontend won’t slow down at all.

So Long, and Thanks for All the Time

If you’ve reached the point that you’re worrying about the performance implications of CSS selectors and the execution time of PHP’s DOMDocument, it is time to thank you. You have very likely managed to reduce your loading time of your WordPress site by more than 3 seconds. This means that with ~10.000 visitors per day, you’ve saved them about 8 hours of time collectively. That’s amazing and more than enough time to sit back and enjoy some of the season’s spirit – you’ve earned it!