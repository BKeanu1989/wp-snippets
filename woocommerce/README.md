# ***Inhaltsverzeichnis***
## ***Woocommerce***
1. [Wc Emails](#wc-emails)
2. [Custom Template Loader](#custom-template-loader)
3. [Products](#products)
4. [Variations](#variations)
5. [List of Hooks](#wc-hooks)
6. [Orders](#orders) | [Documentation - WC_Order](https://woocommerce.github.io/code-reference/classes/WC-Order.html)
7. [Shipping](#shipping)
8. [WC Order Items](#wc-order-items)


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
```php
add_filter( 'woocommerce_locate_template', 'woo_adon_plugin_template', 1, 3 );
   function woo_adon_plugin_template( $template, $template_name, $template_path ) {
     global $woocommerce;
     $_template = $template;
     if ( ! $template_path ) 
        $template_path = $woocommerce->template_url;
 
     $plugin_path  = untrailingslashit( plugin_dir_path( __FILE__ ) )  . '/template/woocommerce/';
 
    // Look within passed path within the theme - this is priority
    $template = locate_template(
    array(
      $template_path . $template_name,
      $template_name
    )
   );
 
   if( ! $template && file_exists( $plugin_path . $template_name ) )
    $template = $plugin_path . $template_name;
 
   if ( ! $template )
    $template = $_template;

   return $template;
}
```

```php
    add_action( 'woocommerce_single_product_summary', array($this, 'custom_product_add_to_cart'), 60 );

    public function custom_product_add_to_cart () {
        global $product;
        
        if ( 'ticket' == $product->get_type() ) {
            // do_action( 'woocommerce_before_add_to_cart_button' ); 
            require_once SNOWDANCE_PLUGIN_PATH . 'includes/tickets/templates/single-product/add-to-cart/simple.php';
        }
    }
```
### ***Products***
> Products
```php
$id = 123;
$_product = wc_get_product($id);
```

### ***Variations***
> Variations

### ***Wc Hooks***
- wc_get_template  
- Visual Hooks
    - [WooCommerce Single Product Page](https://www.businessbloomer.com/woocommerce-visual-hook-guide-single-product-page/)
    - [WooCommerce Visual Hook Guide: Archive / Shop / Cat Pages](https://www.businessbloomer.com/woocommerce-visual-hook-guide-archiveshopcat-page/)
    - [WooCommerce Visual Hook Guide: Checkout Page](https://www.businessbloomer.com/woocommerce-visual-hook-guide-checkout-page/)
- [Hooks Helpdesk](https://hooks.wpdesk.org/)
### ***Orders*** 
```php
$_order = wc_get_order($id); 
```
#### ***Get Items***
[See](#get-order-items)

### ***Shipping***
> Empty For now

### ***Cart***  


### ***WC Order Items***
> Resource (https://woocommerce.github.io/code-reference/classes/WC-Order-Item.html) 

#### get order items
```php
$_order = wc_get_order($id); 
if ($_order) {
    $items = $_order->get_items(); // returns WC_Order_Item_Product
}
```