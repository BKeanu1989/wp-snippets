# Mokka Wordpress Snipptes #

# ***Inhaltsverzeichnis***
## ***Woocommerce***
1. [Wc Emails](#wc-emails)
2. [Custom Template Loader](#custom-template-loader)
3. [Products](#products)
4. [Variations](#variations)
5. [List of Hooks](#wc-hooks)
6. [Orders](#orders) | [Documentation - WC_Order](https://woocommerce.github.io/code-reference/classes/WC-Order.html)
7. [Shipping](#shipping)

## ***Wordpress***
1. [List of Hooks](#wp-hooks)
2. [Cron](#cron)
3. [Custom Logger](#custom-logger)
4. [API](#api)

## ***Wp Hooks***
1. [Ajax Hooks](#ajax-hooks) 
2. [Localize Script](#localize-script)
3. [Nonce Beispiel](#ajax-hooks) & Resource: [Documenation](https://codex.wordpress.org/WordPress_Nonces) | [Handbook](https://developer.wordpress.org/themes/theme-security/using-nonces/)

## ***Woocommerce***

### ***Wc Emails***
> email.php
>
> Email Datei muss existieren

```php
if (! class_exists('Mokka_Wp_Bandcamp_Tracking_Email')) {
    class Mokka_Wp_Bandcamp_Tracking_Email extends WC_Email {
    
        public $helper;

        /**
         * Constructor
        */
        public function __construct() {
            $this->id             = 'mokka_bandcamp_tracking_email';
            $this->customer_email = true;
            $this->title          = __( 'Bandcamp Tracking erhalten', 'mokka-bandcamp' );
            $this->description    = __( 'Diese Email wird versandt sobald ein Tracking eingegangen ist.', 'mokka-bandcamp' );

            $this->template_base = MOKKA_WP_BANDCAMP_PLUGIN_PATH . 'templates/';
            $this->template_html  = 'emails/mokka-bandcamp.php';
            $this->template_plain = 'emails/plain/mokka-bandcamp.php';
            // $this->helper         = wc_gzd_get_email_helper( $this );

            // Triggers for this email
            // add_action( 'woocommerce_order_status_pending_to_processing_notification', array( $this, 'trigger' ), 30 );
            // order status for delivery
            // add_action( 'woocommerce_order_status_processing_to_delivery_notification', array( $this, 'trigger' ), 30 );
            $this->placeholders = array(
                '{site_title}'   => $this->get_blogname(),
                '{order_number}' => '',
            );

            // Call parent constuctor
            parent::__construct();

        }
        
        /**
         * Get email subject.
         *
         * @return string
         * @since  3.1.0
        */
        public function get_default_subject() {
            return __( 'Tracking erhalten für deine Bestellung {order_number}', 'mokka-bandcamp' );
        }
        
        /**
         * Get email heading.
         *
         * @return string
         * @since  3.1.0
        */
        public function get_default_heading() {
            return __( 'Tracking erhalten', 'mokka-bandcamp' );
        }
        
        /**
         * trigger function.
         *
         * @access public
         * @return void
        */
        public function trigger( $order_id ) {
            // TODO: might need to setup locale

            
            if ( $order_id ) {
                $this->object    = wc_get_order( $order_id );

                if (!is_a($this->object, 'WC_Order')) return;

                $this->recipient = $this->object->get_billing_email();
                
                $this->placeholders['{order_number}'] = $this->object->get_order_number();
                
                // set order id
                $this->order_id = $this->object->get_id();
            }

            // TODO: we might need to setup locale and everything else for sending emails - like woocommerce germanized
            // $this->helper->setup_email_locale();

            if ( $this->is_enabled() && $this->get_recipient() ) {

                // Make sure gateways do not insert data here
                // remove_all_actions( 'woocommerce_email_before_order_table' );

                $this->send( $this->get_recipient(), $this->get_subject(), $this->get_content(), $this->get_headers(), $this->get_attachments() );
            }

            // $this->helper->restore_email_locale();
            // $this->helper->restore_locale();
        }

        /**
         * Return content from the additional_content field.
         *
         * Displayed above the footer.
         *
         * @since 3.0.4
         * @return string
        */
        public function get_additional_content() {
            if ( is_callable( 'parent::get_additional_content' ) ) {
                return parent::get_additional_content();
            }

            return '';
        }

        /**
         * Get content html.
         *
         * @access public
         * @return string
        */
        public function get_content_html() {
            return wc_get_template_html( $this->template_html, array(
                'order'              => $this->object,
                'email_heading'      => $this->get_heading(),
                'additional_content' => $this->get_additional_content(),
                'sent_to_admin'      => false,
                'plain_text'         => false,
                'email'              => $this,
                'tracking_code' 	 => get_post_meta($this->order_id, '_tracking', true ),
                'carrier' 			 => get_post_meta($this->order_id, '_carrier', true)
            ) );
        }

        /**
         * Get content plain.
         *
         * @access public
         * @return string
        */
        public function get_content_plain() {
            return wc_get_template_html( $this->template_plain, array(
                'order'              => $this->object,
                'email_heading'      => $this->get_heading(),
                'additional_content' => $this->get_additional_content(),
                'sent_to_admin'      => false,
                'plain_text'         => true,
                'email'              => $this,
                'tracking_code' 	 => get_post_meta($this->order_id, '_tracking', true ),
                'carrier' 			 => get_post_meta($this->order_id, '_carrier', true)

            ) );
        }

    }
    return new Mokka_Wp_Bandcamp_Tracking_Email();
}

```

> email-filter.php
```php
class Mokka_Wp_Bandcamp_Email_Filter {
    public function __construct() {
        add_filter('woocommerce_email_classes', array($this, 'add_emails'), 11, 1);
        // add_filter('woocommerce_locate_core_template', array($this, 'email_templates'), 1, 3);
        add_filter('wc_get_template', array($this, 'set_path'), 11, 5);
    }

    public function add_emails($mails) {
        $mails['Mokka_Wp_Bandcamp_Tracking_Email'] = require_once MOKKA_WP_BANDCAMP_PLUGIN_PATH . '/includes/Classes/Mokka_Wp_Bandcamp_Tracking_Email.php';
        
        return $mails;
    }

    /**
	 * Filter Email template to include WooCommerce Germanized template files
	 *
	 * @param string $core_file
	 * @param string $template
	 * @param string $template_base
	 *
	 * @return string
	*/
	public function email_templates($core_file, $template, $template_base) {
		if (!file_exists($template_base . $template) && file_exists(MOKKA_WP_BANDCAMP_PLUGIN_PATH . '/woocommerce/' . $template)) {
			$core_file = MOKKA_WP_BANDCAMP_PLUGIN_PATH . '/woocommerce/' . $template;
		}

		/**
		 * Filters email templates.
		 *
		 * @param string $core_file The core template file.
		 * @param string $template The template name.
		 * @param string $template_base The template base folder.
		 *
		 * @since 1.0.0
		 *
		*/
		// return apply_filters('mokka_tracking_email_template_hook', $core_file, $template, $template_base);
    }
    
    public function set_path($template, $template_name, $args, $template_path, $default_path) {
		if ($template_name === 'emails/bandcamp-tracking.php') {
			$template = MOKKA_WP_BANDCAMP_PLUGIN_PATH . '/woocommerce/emails/bandcamp-tracking.php';
		}
		return $template;
	}
}

new Mokka_Wp_Bandcamp_Email_Filter();
```

### ***Custom Template Loader***
> template-loader.php
>
> needs /template folder
```php
function mwpbc_load_template($file_name, $args) {
    global $wp_version;

    // if template supports args argument
    if (version_compare($wp_version, '5.5.0', '>=')) {
        if ( $overridden_template = locate_template( $file_name ) ) {
            /*
             * locate_template() returns path to file.
             * if either the child theme or the parent theme have overridden the template.
             */
            load_template( $overridden_template, false, $args );
        } else {
            /*
             * If neither the child nor parent theme have overridden the template,
             * we load the template from the 'templates' sub-directory of the directory this file is in.
             */
            load_template( MOKKA_WP_BANDCAMP_PLUGIN_PATH . "templates/$file_name", false, $args );
        }
    } else {
        if ( $overridden_template = locate_template( $file_name ) ) {
            /*
             * locate_template() returns path to file.
             * if either the child theme or the parent theme have overridden the template.
             */
            set_query_var('args', $args);
            load_template( $overridden_template, false);
        } else {
            /*
             * If neither the child nor parent theme have overridden the template,
             * we load the template from the 'templates' sub-directory of the directory this file is in.
             */
            set_query_var('args', $args);
            load_template( MOKKA_WP_BANDCAMP_PLUGIN_PATH . "templates/$file_name");
        }
    }
}
```
### ***Products***
> Products

### ***Variations***
> Variations

### ***Wc Hooks***
> Empty For now

### ***Orders*** 
> empty still
<dl>
  <dt>Definition list</dt>
  <dd>Is something people use sometimes.</dd>

  <dt>Markdown in HTML</dt>
  <dd>Does *not* work **very** well. Use HTML <em>tags</em>.</dd>
</dl>

### ***Shipping***
> Empty For now

<!-- ---- -->
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
> empty now


### ***Api***
> empty for now
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