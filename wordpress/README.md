# ***Inhaltsverzeichnis***

## ***Wordpress***
1. [List of Hooks](#wp-hooks)
2. [Cron](#cron)
3. [Custom Logger](#custom-logger)
4. [API](#api)
    - 1. [WP Rest Controller](#wp-rest-controller) 
5. [Database](#database)

## ***Wp Hooks***
1. [Ajax Hooks](#ajax-hooks) 
2. [Localize Script](#localize-script)
3. [Nonce Beispiel](#ajax-hooks) & Resource: [Documenation](https://codex.wordpress.org/WordPress_Nonces) | [Handbook](https://developer.wordpress.org/themes/theme-security/using-nonces/)

# ***Wordpress***
### ***Wp Hooks***
> Empty For now


### Cron

> Pluginaktivation:

```php
        // uninstall        
		wp_clear_scheduled_hook('mwc_order_status_cron_job');

        add_action( 'mwc_order_status_cron_job', 'callback_function');

		$cron_name = 'mwc_order_status_cron_job';
		if (!wp_next_scheduled($cron_name)) {
            wp_schedule_event(time(), 'hourly', $cron_name);
            Mokka_Custom_Logger::write_log('Order Status Cron Job activated and scheduled. - next scheduled event for ' . $cron_name . ' : ' .  wp_next_scheduled($cron_name));
        }
```

```php
    add_filter('cron_schedules', array($this, 'mer_add_two_hours_cronjob'), 10, 1);

    function mer_add_two_hours_cronjob( $schedules ) {
        $schedules['every_six_hours'] = array(
            'interval' => 21600, // Every 6 hours
            'display'  => __( 'Every 6 hours' ),
        );
        return $schedules;
    }
```

### ***Custom Logger***
```php
    public static function write_log ( $message ) {
        $myfile = fopen(MOKKA_WHITELABEL_CLIENT_PLUGIN_PATH . 'custom/logging/log.log', "a") or die("Unable to open file!");

        $dateNow = date("Y-m-d H:i:s");
        $now= new DateTime("@" . strtotime($dateNow));
        $now->modify("+2 hours");
        $now = $now->format("Y-m-d H:i:s");

        if ( is_array( $message ) || is_object( $message ) ) {
            $message = print_r( $message, true );
        }

        $message ='[' . $now . ']: ' . $message . "\n";
        fwrite($myfile, $message);
        fclose($myfile);
    }
```


### ***Api***
> empty for now
- (https://kinsta.com/blog/wordpress-rest-api/)
- https://wordpress.org/plugins/jwt-authentication-for-wp-rest-api/
- [Rest API Handbook](https://developer.wordpress.org/rest-api/)
- https://developer.wordpress.org/rest-api/reference/
- https://developer.wordpress.org/rest-api/extending-the-rest-api/
- [Auth] / [Permission Callbacks](#auth)

#### ***Endpoints***
> [Resource](https://developer.wordpress.org/rest-api/extending-the-rest-api/routes-and-endpoints/)

If we wanted to create an endpoint that would return the phrase “Hello World, this is the WordPress REST API” when it receives a GET request, we would first need to register the route for that endpoint. To register routes you should use the register_rest_route() function. It needs to be called on the rest_api_init action hook. register_rest_route() handles all of the mapping for routes to endpoints. Let’s try to create a “Hello World, this is the WordPress REST API” route.
```php
/**
 * This is our callback function that embeds our phrase in a WP_REST_Response
 */
function prefix_get_endpoint_phrase() {
    // rest_ensure_response() wraps the data we want to return into a WP_REST_Response, and ensures it will be properly returned.
    return rest_ensure_response( 'Hello World, this is the WordPress REST API' );
}
 
/**
 * This function is where we register our routes for our example endpoint.
 */
function prefix_register_example_routes() {
    // register_rest_route() handles more arguments but we are going to stick to the basics for now.
    register_rest_route( 'hello-world/v1', '/phrase', array(
        // By using this constant we ensure that when the WP_REST_Server changes our readable endpoints will work as intended.
        'methods'  => WP_REST_Server::READABLE,
        // Here we register our callback. The callback is fired when this endpoint is matched by the WP_REST_Server class.
        'callback' => 'prefix_get_endpoint_phrase',
        'args' => prefix_get_color_arguments(),
    ) );
}

/**
 * We can use this function to contain our arguments for the example product endpoint.
 */
function prefix_get_color_arguments() {
    $args = array();
    // Here we are registering the schema for the filter argument.
    $args['filter'] = array(
        // description should be a human readable description of the argument.
        'description' => esc_html__( 'The filter parameter is used to filter the collection of colors', 'my-text-domain' ),
        // type specifies the type of data that the argument should be.
        'type'        => 'string',
        // enum specified what values filter can take on.
        'enum'        => array( 'red', 'green', 'blue' ),
        // Here we register the validation callback for the filter argument.
        'validate_callback' => 'prefix_filter_arg_validate_callback',
    );
    return $args;
}
 

function prefix_get_colors( $request ) {
    // In practice this function would fetch the desired data. Here we are just making stuff up.
    $colors = array(
        'blue',
        'blue',
        'red',
        'red',
        'green',
        'green',
    );
 
    if ( isset( $request['filter'] ) ) {
       $filtered_colors = array();
       foreach ( $colors as $color ) {
           if ( $request['filter'] === $color ) {
               $filtered_colors[] = $color;
           }
       }
       return rest_ensure_response( $filtered_colors );
    }
    return rest_ensure_response( $colors );
}
 
add_action( 'rest_api_init', 'prefix_register_example_routes' );
```

The first argument passed into register_rest_route() is the namespace, which provides us a way to group our routes. The second argument passed in is the resource path, or resource base. For our example, the resource we are retrieving is the “Hello World, this is the WordPress REST API” phrase. The third argument is an array of options. We specify what methods the endpoint can use and what callback should happen when the endpoint is matched (more things can be done but these are the fundamentals).

The third argument also allows us to provide a permissions callback, which can restrict access for the endpoint to only certain users. The third argument also offers a way to register arguments for the endpoint so that requests can modify the response of our endpoint. We will get into those concepts in the endpoints section of this guide.

When we go to https://ourawesomesite.com/wp-json/hello-world/v1/phrase we can now see our REST API greeting us kindly. Let’s take a look at routes a bit more in depth.

We have now specified a filter argument for this example. We can specify the argument as a query parameter when we request the endpoint. If we make a GET request to https://ourawesomesitem.com/my-colors/v1/colors?filter=blue, we will be returned only the blue colors in our collection. You could also pass these as body parameters in the request body, instead of in the query string. To understand the distinction between query parameters and body parameters you should read about the HTTP spec. Query parameters live in the query string tacked onto the URL and body parameters are directly embedded in the body of an HTTP request.

We have created an argument for our endpoint, but how do we verify that the argument is a string and tell whether it matches the value red, green, or blue. To do this we need to specify a validation callback for our argument.

##### ***Auth***
> [Resource](https://developer.wordpress.org/rest-api/using-the-rest-api/authentication/)

> nonce
```php
wp_localize_script( 'wp-api', 'wpApiSettings', array(
    'root' => esc_url_raw( rest_url() ),
    'nonce' => wp_create_nonce( 'wp_rest' )
) );

options.beforeSend = function(xhr) {
    xhr.setRequestHeader('X-WP-Nonce', wpApiSettings.nonce);
 
    if (beforeSend) {
        return beforeSend.apply(this, arguments);
    }
};

$.ajax( {
    url: wpApiSettings.root + 'wp/v2/posts/1',
    method: 'POST',
    beforeSend: function ( xhr ) {
        xhr.setRequestHeader( 'X-WP-Nonce', wpApiSettings.nonce );
    },
    data:{
        'title' : 'Hello Moon'
    }
} ).done( function ( response ) {
    console.log( response );
} );

    // get token
    jQuery.ajax({
        type: 'post',
        dataType: 'json',
        url : 'http://localhost/mokka-live/wp-json/jwt-auth/v1/token',
        contentType: 'application/json',
        data: JSON.stringify({
            username: 'USERNAME',
            password: 'PASSWORD'
        }),
        success: function (response) {
            console.log("response via jwt", response)
        },
        complete: function() {
            console.log("complete")
        },
        error: function () {
            console.log("error")
        }
    })
```

> [modifying responses](https://developer.wordpress.org/rest-api/extending-the-rest-api/modifying-responses/)

## WP Rest Controller
```php

class Mokka_Test_Api {
    public function __construct() {
        add_action( 'rest_api_init', function () {
            register_rest_route( 'testapi/v1', '/author/(?P<id>\d+)', array(
              'methods' => 'GET',
              'callback' => 'my_awesome_func',
              'args' => array(
                'id' => array(
                  'validate_callback' => 'is_numeric'
                ),
              ),
              'permission_callback' => function () {
                return current_user_can( 'edit_others_posts' );
              }
            ) );
        });

        add_action( 'rest_api_init', function () {
            register_rest_route( 'testapi/v1', '/test', array(
              'methods' => 'GET',
              'callback' => [$this, 'test_func'],
            //   'args' => array(
            //     'id' => array(
            //       'validate_callback' => 'is_numeric'
            //     ),
            //   ),
            //   'permission_callback' => function () {
            //     return current_user_can( 'edit_others_posts' );
            //   }
            ) );
        });
    }

    public function test_func() {
        return rest_ensure_response( 'Hello World, this is the WordPress REST API' );
    }
}

class My_REST_Posts_Controller {
 
    // Here initialize our namespace and resource name.
    public function __construct() {
        $this->namespace     = '/mokka-test-api/v1';
        $this->resource_name = 'test';
    }
 
    // Register our routes.
    public function register_routes() {
        register_rest_route( $this->namespace, '/' . $this->resource_name, array(
            // Here we register the readable endpoint for collections.
            array(
                'methods'   => 'GET',
                'callback'  => array( $this, 'get_items' ),
                'permission_callback' => array( $this, 'get_items_permissions_check' ),
            ),
            // Register our schema callback.
            'schema' => array( $this, 'get_item_schema' ),
        ) );
        register_rest_route( $this->namespace, '/' . $this->resource_name . '/(?P<id>[\d]+)', array(
            // Notice how we are registering multiple endpoints the 'schema' equates to an OPTIONS request.
            array(
                'methods'   => 'GET',
                'callback'  => array( $this, 'get_item' ),
                'permission_callback' => array( $this, 'get_item_permissions_check' ),
            ),
            // Register our schema callback.
            'schema' => array( $this, 'get_item_schema' ),
        ) );
    }
 
    /**
     * Check permissions for the posts.
     *
     * @param WP_REST_Request $request Current request.
     */
    public function get_items_permissions_check( $request ) {
        // if ( ! current_user_can( 'read' ) ) {
        //     return new WP_Error( 'rest_forbidden', esc_html__( 'You cannot view the post resource.' ), array( 'status' => $this->authorization_status_code() ) );
        // }
        return true;
    }
 
    /**
     * Grabs the five most recent posts and outputs them as a rest response.
     *
     * @param WP_REST_Request $request Current request.
     */
    public function get_items( $request ) {
        $args = array(
            'post_per_page' => 5,
        );
        $posts = get_posts( $args );
 
        $data = array();
 
        if ( empty( $posts ) ) {
            return rest_ensure_response( $data );
        }
 
        foreach ( $posts as $post ) {
            $response = $this->prepare_item_for_response( $post, $request );
            $data[] = $this->prepare_response_for_collection( $response );
        }
 
        // Return all of our comment response data.
        return rest_ensure_response( $data );
    }
 
    /**
     * Check permissions for the posts.
     *
     * @param WP_REST_Request $request Current request.
     */
    public function get_item_permissions_check( $request ) {
        if ( ! current_user_can( 'read' ) ) {
            return new WP_Error( 'rest_forbidden', esc_html__( 'You cannot view the post resource.' ), array( 'status' => $this->authorization_status_code() ) );
        }
        return true;
    }
 
    /**
     * Grabs the five most recent posts and outputs them as a rest response.
     *
     * @param WP_REST_Request $request Current request.
     */
    public function get_item( $request ) {
        $id = (int) $request['id'];
        $post = get_post( $id );
 
        if ( empty( $post ) ) {
            return rest_ensure_response( array() );
        }
 
        $response = $this->prepare_item_for_response( $post, $request );
 
        // Return all of our post response data.
        return $response;
    }
 
    /**
     * Matches the post data to the schema we want.
     *
     * @param WP_Post $post The comment object whose response is being prepared.
     */
    public function prepare_item_for_response( $post, $request ) {
        $post_data = array();
 
        $schema = $this->get_item_schema( $request );
 
        // We are also renaming the fields to more understandable names.
        if ( isset( $schema['properties']['id'] ) ) {
            $post_data['id'] = (int) $post->ID;
        }
 
        if ( isset( $schema['properties']['content'] ) ) {
            $post_data['content'] = apply_filters( 'the_content', $post->post_content, $post );
        }
 
        return rest_ensure_response( $post_data );
    }
 
    /**
     * Prepare a response for inserting into a collection of responses.
     *
     * This is copied from WP_REST_Controller class in the WP REST API v2 plugin.
     *
     * @param WP_REST_Response $response Response object.
     * @return array Response data, ready for insertion into collection data.
     */
    public function prepare_response_for_collection( $response ) {
        if ( ! ( $response instanceof WP_REST_Response ) ) {
            return $response;
        }
 
        $data = (array) $response->get_data();
        $server = rest_get_server();
 
        if ( method_exists( $server, 'get_compact_response_links' ) ) {
            $links = call_user_func( array( $server, 'get_compact_response_links' ), $response );
        } else {
            $links = call_user_func( array( $server, 'get_response_links' ), $response );
        }
 
        if ( ! empty( $links ) ) {
            $data['_links'] = $links;
        }
 
        return $data;
    }
 
    /**
     * Get our sample schema for a post.
     *
     * @return array The sample schema for a post
     */
    public function get_item_schema() {
        if ( $this->schema ) {
            // Since WordPress 5.3, the schema can be cached in the $schema property.
            return $this->schema;
        }
 
        $this->schema = array(
            // This tells the spec of JSON Schema we are using which is draft 4.
            '$schema'              => 'http://json-schema.org/draft-04/schema#',
            // The title property marks the identity of the resource.
            'title'                => 'post',
            'type'                 => 'object',
            // In JSON Schema you can specify object properties in the properties attribute.
            'properties'           => array(
                'id' => array(
                    'description'  => esc_html__( 'Unique identifier for the object.', 'my-textdomain' ),
                    'type'         => 'integer',
                    'context'      => array( 'view', 'edit', 'embed' ),
                    'readonly'     => true,
                ),
                'content' => array(
                    'description'  => esc_html__( 'The content for the object.', 'my-textdomain' ),
                    'type'         => 'string',
                ),
            ),
        );
 
        return $this->schema;
    }
 
    // Sets up the proper HTTP status code for authorization.
    public function authorization_status_code() {
 
        $status = 401;
 
        if ( is_user_logged_in() ) {
            $status = 403;
        }
 
        return $status;
    }
}
 
// Function to register our new routes from the controller.
function prefix_register_my_rest_routes() {
    $controller = new My_REST_Posts_Controller();
    $controller->register_routes();
}
 
add_action( 'rest_api_init', 'prefix_register_my_rest_routes' );

new Mokka_Test_Api();
```


### Database
```php
    public static function install_db() {
        global $wpdb;

        $db_artist_version = self::get_db_version();

        $charset_collate = $wpdb->get_charset_collate();
        $table_name = self::get_table_name();
        $sql = "CREATE TABLE IF NOT EXISTS $table_name (
            `id` int(11) NOT NULL AUTO_INCREMENT PRIMARY KEY,
            `artist_nummer` int(11) NOT NULL,
            `artist_name` varchar(100) NOT NULL UNIQUE,
            `registration` date NOT NULL,
            `noshipping_countries` longtext NOT NULL,
            `pre_tax_deductibility` tinyint(1) NOT NULL DEFAULT '0',
            `slug` varchar(50) NOT NULL,
            `first_name` varchar(100) NOT NULL,
            `last_name` varchar(100) NOT NULL,
            `email` varchar(100) NOT NULL,
            `street_address` varchar(100) NOT NULL,
            `zip_code` varchar(100) NOT NULL,
            `phone` varchar(100) NOT NULL,
            `bank_name` varchar(100) NOT NULL,
            `account_holder` varchar(100) NOT NULL,
            `iban` varchar(100) NOT NULL,
            `bic` varchar(100) NOT NULL,
            `parent_id` bigint(20) NOT NULL DEFAULT '0'
          ) $charset_collate";

        require_once( ABSPATH . 'wp-admin/includes/upgrade.php' );
        dbDelta( $sql );

        add_option('mwc_artist_db_version', $db_artist_version);
    }
```


---

# ***Custom Hooks***
> empty now


#### ***Ajax Hooks***
> 

> PHP
```php
// TODO: do nonce
add_action('wp_ajax_{NAME}', 'name_of_callback');
add_action('wp_ajax_nopriv_{NAME}', array($this, 'name_of_callback'));

function name_of_callback() {
    $action = 'SAMESAME';
    $post = $_POST; // has post superglobal

    wp_verify_nonce($_POST['nonce'], $action);
}

```
> JS
>
> [Localize Script](#localize-script)
```js
// nonce
jQuery.ajax({
    type : "post",
    dataType : "json",
    url : '<?php echo admin_url( 'admin-ajax.php' ) ?>', // or set via localizescript
    data : {
        action: "{NAME}",
        // nonce: dynamic_vars.save_artist_nonce, // passed via localizescript
        
    },
    success: function(response) {
        if(response.success == true) {
            console.log(response)
        }
        else {
            console.log(response)
        }
    }
})   

```

> In HTML FORM
>
> Muss die gleiche action haben
```php

<?php echo wp_nonce_field($action); ?>

```

#### ***Localize Script***

```php
$action = 'SAMESAME';
wp_enqueue_script( 'my_js_library', get_template_directory_uri() . '/js/myLibrary.js' );
 
$dataToBePassed = array(
    'home'            => get_stylesheet_directory_uri(),
    'pleaseWaitLabel' => __( 'Please wait...', 'default' )

    'save_artist_nonce' => $action

);
wp_localize_script( 'my_js_library', 'php_vars', $datatoBePassed );
```
