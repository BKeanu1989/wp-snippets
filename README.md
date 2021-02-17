# Mokka Wordpress Snipptes #
## Woocommerce ##
1. [Wc Emails](#wc-emails)
2. [Custom Template Loader](#custom-template-loader)
3. [Products](#products)
4. [Variations](#variations)
5. [List of Hooks](#wc-hooks)


### Wc Emails
email.php
Email Datei muss existieren

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

email-filter.php
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

### Custom Template Loader
template-loader.php
needs /template folder
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
### Products

### Variations
### Wc Hooks

## Wordpress
1. [List of Hooks](#wp-hooks)

### Wp Hooks
1. [Ajax Hooks](#ajax-hooks)
2. [Localize Script](#localize-script)

#### Ajax Hooks
PHP
```php
// TODO: do nonce
add_action('wp_ajax_{NAME}', 'name_of_callback');
add_action('wp_ajax_nopriv_{NAME}', array($this, 'name_of_callback'));

function name_of_callback() {
    $post = $_POST; // has post superglobal


}

```
JS
[Localize Script](#localize-script)
```js
jQuery.ajax({
    type : "post",
    dataType : "json",
    url : '<?php echo admin_url( 'admin-ajax.php' ) ?>', // or set via localizescript
    data : {
        action: "pull_new_orders",
        selectedBand: selectedBand
        
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

#### Localize Script
```php
wp_enqueue_script( 'my_js_library', get_template_directory_uri() . '/js/myLibrary.js' );
 
$dataToBePassed = array(
    'home'            => get_stylesheet_directory_uri(),
    'pleaseWaitLabel' => __( 'Please wait...', 'default' )
);
wp_localize_script( 'my_js_library', 'php_vars', $datatoBePassed );
```


**Edit a file, create a new file, and clone from Bitbucket in under 2 minutes**

When you're done, you can delete the content in this README and update the file with details for others getting started with your repository.

*We recommend that you open this README in another tab as you perform the tasks below. You can [watch our video](https://youtu.be/0ocf7u76WSo) for a full demo of all the steps in this tutorial. Open the video in a new tab to avoid leaving Bitbucket.*

---

## Edit a file

You’ll start by editing this README file to learn how to edit a file in Bitbucket.

1. Click **Source** on the left side.
2. Click the README.md link from the list of files.
3. Click the **Edit** button.
4. Delete the following text: *Delete this line to make a change to the README from Bitbucket.*
5. After making your change, click **Commit** and then **Commit** again in the dialog. The commit page will open and you’ll see the change you just made.
6. Go back to the **Source** page.

---

## Create a file

Next, you’ll add a new file to this repository.

1. Click the **New file** button at the top of the **Source** page.
2. Give the file a filename of **contributors.txt**.
3. Enter your name in the empty file space.
4. Click **Commit** and then **Commit** again in the dialog.
5. Go back to the **Source** page.

Before you move on, go ahead and explore the repository. You've already seen the **Source** page, but check out the **Commits**, **Branches**, and **Settings** pages.

---

## Clone a repository

Use these steps to clone from SourceTree, our client for using the repository command-line free. Cloning allows you to work on your files locally. If you don't yet have SourceTree, [download and install first](https://www.sourcetreeapp.com/). If you prefer to clone from the command line, see [Clone a repository](https://confluence.atlassian.com/x/4whODQ).

1. You’ll see the clone button under the **Source** heading. Click that button.
2. Now click **Check out in SourceTree**. You may need to create a SourceTree account or log in.
3. When you see the **Clone New** dialog in SourceTree, update the destination path and name if you’d like to and then click **Clone**.
4. Open the directory you just created to see your repository’s files.

Now that you're more familiar with your Bitbucket repository, go ahead and add a new file locally. You can [push your change back to Bitbucket with SourceTree](https://confluence.atlassian.com/x/iqyBMg), or you can [add, commit,](https://confluence.atlassian.com/x/8QhODQ) and [push from the command line](https://confluence.atlassian.com/x/NQ0zDQ).

### My Multi Word Header