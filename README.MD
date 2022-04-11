# import-export-plugin-settings

This library provides an easy way to integrate export and import of settings in your WordPress plugin. 

## Installation

Library can be installed using composer.

First you need to add our repository to the list of recognized repositories in `composer.json`:
```
"repositories": [
    {
        "type": "vcs",
        "url": "https://github.com/wpwhitesecurity/import-export-plugin-settings"
    }
]
```

Next add the library to your dependencies by running:
```
composer require wpwhitesecurity/import-export-plugin-settings
```

## Usage

First you need to make sure the library is loaded. Including the composer autoloader is sufficient:
```
// Require Composer autoloader if it exists.
if ( file_exists( YOUR_PLUGIN_PATH . '/vendor/autoload.php' ) ) {
    require_once YOUR_PLUGIN_PATH . '/vendor/autoload.php';
}
```

Next step is initialising the library with couple of configurations items. These should be passed to the constructor of the `SettingsImportExport` class in the same order as listed below.

- Path to the library base folder relative to the WP plugins folder. It's necessary to load the translation file for the library.
- Absolute URL to the library base folder. It will be used to load required JavaScript files and CSS stylesheets on the front-end.
- Instance key. Used to prefix nonce actions, field names, AJAX actions etc. For example `wsal` or `wp2fa`.
- Callback to retrieve a list of options to export. The result should be a list of arrays with keys `option_name` and `option_value`.
- Callback to get type of option used during the pre-import check. If one of values `user`, `role` or `post_type` is returned, the existence of option value is verified automatically. 
- Callback to run an extra pre-import check of option that doesn't have a specific type. It should return WP_Error if the option cannot be imported for some reason.
- Callback to run to see if scripts and styles can be enqueued. It should return `true` only in case the assets are allowed to load.

The UI itself can be rendered using function `SettingsImportExport::render()`.

### Example

```
if ( class_exists( 'SettingsImportExport' ) ) {
    $settings_importer = new SettingsImportExport(
        'vendor/wpwhitesecurity/import-export-plugin-settings',
        plugin_dir_url( __FILE__ ) . 'vendor/wpwhitesecurity/import-export-plugin-settings',
        'my_plugin',
        'my_plugin_get_option_list',
        'my_plugin_get_option_type',
        'my_plugin_run_pre_import_check',
        'my_plugin_can_enqueue_scripts'
    );
}

/**
 * Retrieves a list of options to export.
 *
 * @return array
 */
function my_plugin_get_option_list() {
    global $wpdb;
    $prepared_query = $wpdb->prepare(
        "SELECT `option_name`, `option_value` FROM `{$wpdb->options}` WHERE `option_name` LIKE %s ORDER BY `option_name` ASC",
        'myplugin_%'
    );

    return $wpdb->get_results( $prepared_query ); // phpcs:ignore
}

/**
 * Determines the option type for known options.
 *
 * @param string $setting_name Setting name.
 *
 * @return string|null
 */
function my_plugin_get_option_type( $setting_name ) {
    if ( 'myplugin_custom-post-types' === $setting_name ) {
        return 'post_type';

    }

    if ( 'myplugin_excluded-roles' === $setting_name ) {
        return 'role';
    }

    if ( 'myplugin_excluded-users' === $setting_name ) {
        return 'user';
    }

    if ( 0 === strpos( $setting_name, 'myplugin_usersessions_policy_' ) ) {
        return 'role';
    }

    return null;
}

/**
 * Runs a pre-import check for settings that require extra handling.
 *
 * @param string $setting_name  Setting name.
 * @param mixed  $setting_value Setting value.
 *
 * @return void|WP_Error Error object if the setting needs special handling.
 */
function my_plugin_run_pre_import_check( $setting_name, $setting_value ) {
    if ( strpos( $setting_name, 'myplugin_notification-' ) === 0 && strpos( $setting_name, 'built-in' ) === false ) {
        return new WP_Error( 'not_supported', esc_html__( 'Custom notifications are not supported', 'myplugin' ) );
    }

    if ( 'myplugin_restrict-plugin-settings' === $setting_name && 'only_me' === $setting_value ) {
        return new WP_Error( 'check_restrict_access', esc_html__( 'Settings access', 'myplugin' ) );
    }
}

/**
 * Checks if the scripts can be enqueued on current screen.
 *
 * @return bool True if scripts and styles should be loaded.
 */
function my_plugin_can_enqueue_scripts() {
    if ( ! array_key_exists( 'page', $_GET ) ) {
        return false;
    }

    $current_page = filter_input( INPUT_GET, 'page', FILTER_SANITIZE_STRING );
    if ( 'myplugin-settings' !== $current_page ) {
        return false;
    }

    if ( ! array_key_exists( 'tab', $_GET ) ) {
        return false;
    }

    $current_tab = filter_input( INPUT_GET, 'tab', FILTER_SANITIZE_STRING );

    return ( 'settings-export-import' === $current_tab );
}
```