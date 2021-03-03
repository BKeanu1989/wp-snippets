# Inhaltsverzeichnis

## Resource
> Boilerplate https://wppb.me/

## Settings / Options
```php
class Mokka_White_Label_Mokka_Main {
    public function __construct() {
        // include all templates/hooks/templates

        $this->includeFiles();
        $this->registerHooks();
    }

    public function includeFiles() {
        // classes
    }

    public function registerHooks () {
        // Register hooks
        add_action('admin_menu', array($this, 'mwc_menu_page'));
    }

    public function mwc_menu_page() {
        add_menu_page(
            __('Mokka Whitelabel Client', 'mokka-bills'),
            'Mokka Whitelabel Client',
            'manage_options',
            MWC_MOKKA_MAIN_PLUGIN_PATH . 'views/main.php',
            null,
            'dashicons-chart-area',
            10
        );
    }
}

class Mokka_White_Label_Mokka_Settings {
    public function __construct() {
        // include all templates/hooks/templates

        $this->includeFiles();
        $this->registerHooks();
    }

    public function includeFiles() {
        // classes
        require_once(MWC_MOKKA_SETTINGS_PLUGIN_PATH . 'includes/classes/settings-handler.php');
    }

    public function registerHooks () {
        // Register hooks
        add_action('admin_menu', array($this, 'me_menu_page'));
        add_action('admin_post_save_settings', array($this, 'save_mwc_settings')); // save selected settings
    }

    // Adding View public Functions
    function me_menu_page() {
        add_submenu_page(
            MWC_MOKKA_MAIN_PLUGIN_PATH . 'views/main.php',
                'Settings für Mokka Whitelabel Client',
                'Options',
                'administrator',
                'mwc-option',
                fn() => require_once(MWC_MOKKA_SETTINGS_PLUGIN_PATH . 'views/settings.php') 
        );
    }

    public function save_mwc_settings() {
        if (isset($_POST) && count($_POST) > 0) {
            Settings_Handler::save_settings($_POST);
        }

        Mokka_Response_Handler::send_response(200, 'Gepeichert');
    }
}
```

> settings-view.php
```php
<?php
global $wpdb;
$mwc_bills_emails_blacklisted = get_option('mwc-bills-emails-blacklisted');
$checkBoxVal = get_option( 'mwc_exporter_cron_job_check_box', 'no' );
$checked = $checkBoxVal == 'true' ? 'checked' : '';

if (!$mwc_bills_emails_blacklisted) {
    $mwc_bills_emails_blacklisted = [];
}

$mwc_bills_emails_blacklisted = implode(',', $mwc_bills_emails_blacklisted);
$action =  'mwc-save-settings';
?>
```
```html
<form method="POST" action="" method="post" id="mokka-settings-form">
    <input type="hidden" name="action" value="save_settings">
    <?php echo wp_nonce_field($action); ?>
    <div class="container">
        <h1><?php _e('Einstellungen', 'mwc') ?></h1>
        <div id="mwc-settings-accordion">
            <div class="card settings-card-item">
                <div class="card-header" id="headingOne">
                    <h5 class="mb-0">
                        <button type="button" class="btn btn-link" data-toggle="collapse" data-target="#mwc-main-options-section" aria-expanded="true" aria-controls="mwc-main-options-section">
                            <?php _e('Whitelabel Einstellungen') ?>
                        </button>
                    </h5>
                </div>
    
                <div id="mwc-main-options-section" class="collapse show" aria-labelledby="headingOne" data-parent="#mwc-settings-accordion">
                    <div class="card-body">
                        <div class="form-group">
                            <label for="mwc_identifier">Whitelabel Identifier:</label>
                            <input type="text" class="form-control" name="mwc_identifier" value="<?php echo get_option('mwc-identifier') ?>" aria-describedby="mwc-id" placeholder=""> 
                            <small id="mwc-id" class="form-text text-muted">Referenznummer zu mokka</small>
                        </div>
                    </div>
                </div>
            </div>
            <div class="card settings-card-item">
                <div class="card-header" id="headingTwo">
                    <h5 class="mb-0">
                        <button type="button" class="btn btn-link collapsed" data-toggle="collapse" data-target="#mwc-billing-options-section" aria-expanded="false" aria-controls="mwc-billing-options-section">
                            <?php _e('Abrechnungseinstellungen') ?>
                        </button>
                    </h5>
                </div>
                <div id="mwc-billing-options-section" class="collapse" aria-labelledby="headingTwo" data-parent="#mwc-settings-accordion">
                    <div class="card-body">
                        <div class="form-group">
                            <label for="mwc_address">Url:</label>
                            <input type="text" class="form-control" name="mwc_address" value="<?php echo get_option('mwc-address') ?>" aria-describedby="mwc-address" placeholder=""> 
                            <small id="mwc-address" class="form-text text-muted">Referenznummer zu mokka</small>
                        </div>  
                        <div class="form-group">
                            <label for="mwc_bills_emails_blacklisted">Blacklisted Emails:</label>
                            <input type="text" class="form-control" name="mwc_bills_emails_blacklisted" value="<?php echo $mwc_bills_emails_blacklisted ?>" aria-describedby="mwc-blacklist-emails" placeholder=""> 
                            <small id="mwc-blacklist-emails" class="form-text text-muted">Referenznummer zu mokka</small>
                        </div>  
                    </div>
                </div>
            </div>
            <div class="card settings-card-item">
                <div class="card-header" id="headingThree">
                    <h5 class="mb-0">
                        <button type="button" class="btn btn-link collapsed" data-toggle="collapse" data-target="#mwc-exporter-options-sections" aria-expanded="false" aria-controls="mwc-exporter-options-sections">
                            <?php _e('Exporter Einstellungen') ?>
                        </button>
                    </h5>
                </div>
                <div id="mwc-exporter-options-sections" class="collapse" aria-labelledby="headingThree" data-parent="#mwc-settings-accordion">
                    <div class="card-body">
                        <div class="form-group">
                            <input type="checkbox" name="single-export-check-box" value="true" aria-describedby="mwc-exporter-cron" id="single-export-check-box" <?php echo $checked; ?>>
                            <label for="single-export-check-box">Cron Job</label>
                            <small id="mwc-exporter-cron" class="form-text text-muted">Der Cron Job triggert den Exporter alle 2h automatisch. Zum Einschalten die Checkbox aktivieren und auf Speichern klicken</small>
                        </div>  
                    </div>
                </div>
            </div>
        </div>
        <button id="mokka-export-settings-submit-btn" class="btn btn-primary float-right mt-2" type="submit">Speichern</button>
    </div>
</form>
```
```js
    jQuery(document).ready(function() {

        jQuery('#mokka-settings-form').submit(ajaxSubmit);

        function ajaxSubmit(event) {
            event.preventDefault()
            var submitform = jQuery('#mokka-settings-form').serialize();
            var site_url = "<?php echo home_url() . '/wp-admin/admin-post.php'; ?>";
            jQuery.ajax({
                type:'POST',
                url: site_url,
                data: submitform,
                success: function(data) {
                    const response = JSON.parse(data);
                    mwc_display_notice(response);
                }
            });

            return false;
        }
    });
```