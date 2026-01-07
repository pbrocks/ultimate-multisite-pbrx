# Developer README

A comprehensive guide for developers working with Ultimate Multisite.

## Table of Contents

- [Quick Start](#quick-start)
- [Development Environment](#development-environment)
- [Understanding the Architecture](#understanding-the-architecture)
- [Working with Models](#working-with-models)
- [Payment Gateways](#payment-gateways)
- [Checkout System](#checkout-system)
- [Domain Mapping](#domain-mapping)
- [Extending the Plugin](#extending-the-plugin)
- [Testing](#testing)
- [Debugging](#debugging)
- [Common Tasks](#common-tasks)
- [Performance Tips](#performance-tips)
- [Security Best Practices](#security-best-practices)

## Quick Start

### Prerequisites

- **PHP 7.4.30+** (PHP 8.1+ recommended for development)
- **Composer** for PHP dependencies
- **Node.js 16+** and npm for build tools
- **WordPress Multisite** installation (5.3+)
- **MySQL 5.6+**
- **Git** for version control

### One-Command Setup

```bash
git clone https://github.com/Multisite-Ultimate/ultimate-multisite.git
cd ultimate-multisite
npm run dev:setup
```

This single command:
1. Installs Composer dependencies (with dev requirements)
2. Installs npm dependencies
3. Sets up pre-commit Git hooks (PHPCS + PHPStan)

### Manual Setup

```bash
# Install PHP dependencies
composer install

# Install Node dependencies
npm install

# Setup Git hooks
npm run setup:hooks
```

### Installing in WordPress

**Option 1: Development Installation**
```bash
# Clone into your plugins directory
cd wp-content/plugins/
git clone https://github.com/Multisite-Ultimate/ultimate-multisite.git
cd ultimate-multisite
composer install
```

**Option 2: Symlink for Development**
```bash
# Clone to your preferred location
git clone https://github.com/Multisite-Ultimate/ultimate-multisite.git ~/projects/ultimate-multisite
cd ~/projects/ultimate-multisite
composer install

# Symlink to WordPress
ln -s ~/projects/ultimate-multisite /path/to/wordpress/wp-content/plugins/ultimate-multisite
```

**Important:** After installation, network activate the plugin and copy `sunrise.php` to `wp-content/sunrise.php`:
```bash
cp wp-content/plugins/ultimate-multisite/sunrise.php wp-content/sunrise.php
```

Then add to `wp-config.php` before `/* That's all, stop editing! */`:
```php
define('SUNRISE', true);
```

## Development Environment

### Running the Local Environment

Ultimate Multisite includes a WordPress environment configuration using `@wordpress/env`:

```bash
# Start WordPress environment
npm run env:start

# Stop environment
npm run env:stop

# Clean and restart
npm run env:destroy
npm run env:start
```

The environment is configured in `.wp-env.json`.

### Essential Development Commands

```bash
# Code Quality
npm run check              # Run all checks (lint + stan + test)
npm run lint               # Check code style (PHP + JS + CSS)
npm run lint:fix           # Auto-fix code style issues
npm run stan               # Run PHPStan static analysis

# Testing
npm test                   # Run PHPUnit tests
npm run test:coverage      # Run tests with coverage report
npm run cy:open:dev        # Open Cypress for E2E testing

# Building
npm run build:dev          # Development build
npm run build              # Production build (used for releases)
npm run clean              # Clean build artifacts
```

### Pre-commit Hooks

Git hooks automatically run on commit:
- **PHPCS** checks changed PHP files
- **PHPStan** analyzes changed PHP files
- **Commit message** validation (conventional commits)

## Understanding the Architecture

### Core Plugin Flow

```php
// 1. Plugin loads (ultimate-multisite.php)
define('WP_ULTIMO_PLUGIN_FILE', __FILE__);
require_once __DIR__ . '/constants.php';
require_once __DIR__ . '/vendor/autoload_packages.php';

// 2. Main class initializes (singleton pattern)
function WP_Ultimo() {
    return WP_Ultimo::get_instance();
}
$GLOBALS['WP_Ultimo'] = WP_Ultimo();

// 3. Init hook fires
WP_Ultimo()->init();
```

### Namespace and Autoloading

All core classes use the `WP_Ultimo` namespace:

```php
namespace WP_Ultimo;

class My_Class {
    // Class code
}
```

The plugin uses Jetpack Autoloader with a custom suffix for isolation. Classes in `inc/` are autoloaded based on the classmap in `composer.json`.

### Singleton Pattern

Most core classes use the Singleton trait:

```php
namespace WP_Ultimo;

class My_Manager {
    use \WP_Ultimo\Traits\Singleton;

    public function init() {
        // Initialize the manager
        add_action('init', [$this, 'setup']);
    }
}

// Usage
$manager = My_Manager::get_instance();
```

### Custom Database Tables

Ultimate Multisite uses **BerlinDB** for custom database tables instead of WordPress post types. This provides:
- Better performance for queries
- More flexible schema
- Cleaner data structure

**Key Tables:**
- `{prefix}_wu_customers` - Customer accounts
- `{prefix}_wu_memberships` - Customer subscriptions
- `{prefix}_wu_sites` - Site metadata
- `{prefix}_wu_payments` - Transaction records
- `{prefix}_wu_domains` - Custom domain mappings
- `{prefix}_wu_products` - Plans and add-ons
- `{prefix}_wu_discount_codes` - Discount codes
- `{prefix}_wu_webhooks` - Webhook configurations
- `{prefix}_wu_events` - Event log

**Table Structure:**
```
inc/database/
├── [model-name]/
│   ├── class-[model]-table.php        # Table schema
│   ├── class-[model]-schema.php       # Schema definition
│   └── class-[model]-query.php        # Query class
```

## Working with Models

### Model Architecture

Models extend `Base_Model` and provide CRUD operations:

```php
namespace WP_Ultimo\Models;

class Customer extends Base_Model {
    protected $id;
    protected $user_id;
    protected $email_verification;
    protected $type;
    protected $vip;

    // Getters and setters
    public function get_user_id() {
        return $this->user_id;
    }

    public function set_vip($vip) {
        $this->vip = (bool) $vip;
    }
}
```

### Creating Models

```php
// Create a customer
$customer = wu_create_customer([
    'user_id' => 123,
    'email_verification' => 'verified',
    'type' => 'customer',
    'vip' => false,
]);

if (is_wp_error($customer)) {
    // Handle error
    wp_die($customer->get_error_message());
}

// Access properties
echo $customer->get_id();
echo $customer->get_user_id();
```

### Querying Models

```php
// Get single customer by ID
$customer = wu_get_customer(5);

// Get customer by user ID
$customer = wu_get_customer_by_user_id(123);

// Query multiple customers
$customers = wu_get_customers([
    'type' => 'customer',
    'vip' => true,
    'number' => 20,
    'paged' => 1,
    'orderby' => 'date_registered',
    'order' => 'DESC',
]);

foreach ($customers as $customer) {
    echo $customer->get_display_name();
}
```

### Updating Models

```php
// Get customer
$customer = wu_get_customer(5);

// Update properties
$customer->set_vip(true);
$customer->add_meta('custom_field', 'value');

// Save changes
$saved = $customer->save();

if (is_wp_error($saved)) {
    // Handle error
}
```

### Deleting Models

```php
$customer = wu_get_customer(5);
$deleted = $customer->delete();
```

### Model Relationships

Models have helper methods for relationships:

```php
$customer = wu_get_customer(5);

// Get customer's memberships
$memberships = $customer->get_memberships();

// Get customer's sites
$sites = $customer->get_sites();

// Get customer's payments
$payments = $customer->get_payments();

// Get membership's plan
$membership = wu_get_membership(10);
$plan = $membership->get_plan(); // Returns Product model

// Get site's membership
$site = wu_get_site(15);
$membership = $site->get_membership();
```

### Meta Data

All models support meta data:

```php
$customer = wu_get_customer(5);

// Add meta
$customer->add_meta('crm_id', '12345');

// Get meta
$crm_id = $customer->get_meta('crm_id');

// Update meta
$customer->update_meta('crm_id', '67890');

// Delete meta
$customer->delete_meta('crm_id');

// Get all meta
$all_meta = $customer->get_all_meta();
```

## Payment Gateways

### Gateway Structure

Payment gateways extend `Base_Gateway`:

```php
namespace WP_Ultimo\Gateways;

class My_Gateway extends Base_Gateway {

    public $id = 'my_gateway';

    public function __construct() {
        $this->title = __('My Payment Gateway', 'ultimate-multisite');
        $this->description = __('Pay via My Gateway', 'ultimate-multisite');

        // What this gateway supports
        $this->supports = ['one-time', 'recurring', 'refunds'];

        parent::__construct();
    }

    /**
     * Process one-time payment
     */
    public function process_single_payment($payment, $cart, $order) {
        try {
            // Call your payment API
            $result = $this->api_charge([
                'amount' => $payment->get_total(),
                'currency' => $payment->get_currency(),
                'description' => $cart->get_description(),
            ]);

            if ($result->success) {
                // Update payment record
                $payment->set_gateway_payment_id($result->transaction_id);
                $payment->set_status('completed');
                $payment->save();

                return true;
            }

            return new \WP_Error('payment_failed', $result->error_message);

        } catch (\Exception $e) {
            return new \WP_Error('gateway_error', $e->getMessage());
        }
    }

    /**
     * Process subscription signup
     */
    public function process_signup($membership, $customer, $cart, $order) {
        try {
            // Create customer in gateway
            $gateway_customer = $this->create_customer([
                'email' => $customer->get_email(),
                'name' => $customer->get_display_name(),
            ]);

            // Create subscription
            $subscription = $this->create_subscription([
                'customer_id' => $gateway_customer->id,
                'plan_id' => $this->get_or_create_plan($membership->get_plan()),
                'trial_days' => $membership->get_trial_days(),
            ]);

            // Store IDs
            $customer->add_meta('gateway_customer_id', $gateway_customer->id);
            $membership->set_gateway_subscription_id($subscription->id);
            $membership->save();

            return true;

        } catch (\Exception $e) {
            return new \WP_Error('subscription_failed', $e->getMessage());
        }
    }

    /**
     * Handle webhook
     */
    public function handle_webhook() {
        $payload = $this->get_webhook_payload();

        // Verify signature
        if (!$this->verify_webhook_signature($payload)) {
            wp_die('Invalid signature', 'Webhook Error', 401);
        }

        switch ($payload['event_type']) {
            case 'payment.succeeded':
                $this->handle_payment_succeeded($payload);
                break;

            case 'subscription.cancelled':
                $this->handle_subscription_cancelled($payload);
                break;
        }

        wp_die('OK', 'Webhook Received', 200);
    }
}
```

### Registering a Gateway

```php
add_filter('wu_payment_gateways', function($gateways) {
    $gateways['my_gateway'] = 'WP_Ultimo\Gateways\My_Gateway';
    return $gateways;
});
```

### Gateway Settings

Add settings to the gateway:

```php
public function get_settings() {
    return [
        'api_key' => [
            'title' => __('API Key', 'ultimate-multisite'),
            'type' => 'text',
            'required' => true,
        ],
        'api_secret' => [
            'title' => __('API Secret', 'ultimate-multisite'),
            'type' => 'password',
            'required' => true,
        ],
        'test_mode' => [
            'title' => __('Test Mode', 'ultimate-multisite'),
            'type' => 'toggle',
            'default' => true,
        ],
    ];
}
```

## Checkout System

### Checkout Flow

The checkout system consists of:
1. **Checkout Form** - Multi-step form builder
2. **Cart** - Shopping cart with line items
3. **Signup Fields** - Modular field types
4. **Checkout** - Processing logic

### Creating a Custom Signup Field

```php
namespace WP_Ultimo\Checkout\Signup_Fields;

class Signup_Field_Company extends Base_Signup_Field {

    public $type = 'company';

    public function __construct() {
        $this->id = 'company';
        $this->title = __('Company Information', 'ultimate-multisite');

        parent::__construct();
    }

    /**
     * Get field markup
     */
    public function field_markup() {
        ob_start();
        ?>
        <div class="wu-company-field">
            <label for="company_name">
                <?php _e('Company Name', 'ultimate-multisite'); ?>
            </label>
            <input
                type="text"
                name="company_name"
                id="company_name"
                value="<?php echo esc_attr($this->get_value('company_name')); ?>"
                required
            />

            <label for="company_size">
                <?php _e('Company Size', 'ultimate-multisite'); ?>
            </label>
            <select name="company_size" id="company_size">
                <option value="1-10">1-10 employees</option>
                <option value="11-50">11-50 employees</option>
                <option value="51-200">51-200 employees</option>
                <option value="201+">201+ employees</option>
            </select>
        </div>
        <?php
        return ob_get_clean();
    }

    /**
     * Validate field
     */
    public function validate() {
        if (empty($_POST['company_name'])) {
            return new \WP_Error('company_required', __('Company name is required.', 'ultimate-multisite'));
        }

        return true;
    }

    /**
     * Process field after checkout
     */
    public function process($customer, $membership, $site) {
        // Store company info
        $customer->add_meta('company_name', sanitize_text_field($_POST['company_name']));
        $customer->add_meta('company_size', sanitize_text_field($_POST['company_size']));
    }
}
```

### Register Custom Field

```php
add_filter('wu_checkout_signup_fields', function($fields) {
    $fields['company'] = 'WP_Ultimo\Checkout\Signup_Fields\Signup_Field_Company';
    return $fields;
});
```

### Modifying Cart Totals

```php
// Add a fee
add_filter('wu_cart_calculate_totals', function($cart) {
    if ($cart->should_collect_billing_address()) {
        $cart->add_fee([
            'id' => 'billing_fee',
            'label' => __('Billing Address Processing', 'ultimate-multisite'),
            'amount' => 5.00,
        ]);
    }

    return $cart;
});

// Apply a discount
add_filter('wu_cart_total', function($total, $cart) {
    $customer = $cart->get_customer();

    if ($customer && $customer->is_vip()) {
        $total = $total * 0.9; // 10% discount
    }

    return $total;
}, 10, 2);
```

### Checkout Hooks

```php
// Before checkout processing
add_action('wu_checkout_before_processing', function($cart) {
    // Validate custom requirements
    if (!custom_validation_check()) {
        wp_die(__('Custom validation failed', 'ultimate-multisite'));
    }
});

// After checkout success
add_action('wu_checkout_completed', function($payment, $customer, $membership) {
    // Send to CRM
    my_crm_api()->create_customer([
        'email' => $customer->get_email(),
        'plan' => $membership->get_plan()->get_name(),
    ]);

    // Track conversion
    if (function_exists('gtag')) {
        gtag('event', 'purchase', [
            'transaction_id' => $payment->get_id(),
            'value' => $payment->get_total(),
            'currency' => $payment->get_currency(),
        ]);
    }
}, 10, 3);
```

## Domain Mapping

Domain mapping allows customers to use custom domains for their sites.

### How Domain Mapping Works

1. **Sunrise.php** - Loaded early by WordPress (`wp-content/sunrise.php`)
2. **Domain Lookup** - Checks if incoming domain is mapped
3. **Site Switching** - Loads the correct site based on domain
4. **SSL/DNS** - Handles SSL certificates and DNS verification

### Working with Domains

```php
// Create a domain mapping
$domain = wu_create_domain([
    'domain' => 'example.com',
    'blog_id' => 5,
    'customer_id' => 10,
    'primary_domain' => 1,
    'stage' => 'domain-mapping', // or 'checking-dns', 'checking-ssl', 'done'
]);

// Get domains for a site
$domains = wu_get_domains([
    'blog_id' => 5,
]);

// Get primary domain
$primary_domain = wu_get_primary_domain(5);

// Set domain as primary
$domain->set_primary_domain(1);
$domain->save();
```

### Domain Verification

```php
// Check DNS records
$domain = wu_get_domain(10);
$dns_check = $domain->verify_dns();

if ($dns_check) {
    $domain->set_stage('checking-ssl');
    $domain->save();
}

// Verify SSL
$ssl_check = $domain->verify_ssl();

if ($ssl_check) {
    $domain->set_stage('done');
    $domain->save();
}
```

### Hosting Integration

Create a hosting integration for automated domain setup:

```php
namespace WP_Ultimo\Domain_Mapping;

class My_Host_Integration extends Base_Host_Integration {

    public $id = 'myhost';

    public function create_domain($domain_name, $site_id) {
        // Call hosting API to add domain
        $result = $this->api_call('domains/add', [
            'domain' => $domain_name,
            'document_root' => $this->get_site_path($site_id),
        ]);

        if ($result->success) {
            // Create DNS records
            $this->create_dns_records($domain_name, $site_id);

            // Request SSL certificate
            $this->request_ssl($domain_name);

            return true;
        }

        return new \WP_Error('domain_creation_failed', $result->error);
    }
}
```

## Extending the Plugin

### Creating an Addon

**Addon Structure:**
```
my-addon/
├── my-addon.php                 # Main plugin file
├── inc/
│   ├── class-addon.php
│   ├── admin-pages/
│   │   └── class-settings-page.php
│   └── models/
│       └── class-lead.php
├── assets/
│   ├── css/style.css
│   └── js/script.js
└── views/
    └── admin/settings.php
```

**Main Plugin File:**
```php
<?php
/**
 * Plugin Name: My Ultimate Multisite Addon
 * Plugin URI: https://example.com
 * Description: Extends Ultimate Multisite with custom features
 * Version: 1.0.0
 * Author: Your Name
 * Requires PHP: 7.4
 * WP Ultimo: 2.0.0
 */

namespace My_Addon;

defined('ABSPATH') || exit;

// Define constants
define('MY_ADDON_VERSION', '1.0.0');
define('MY_ADDON_FILE', __FILE__);
define('MY_ADDON_PATH', plugin_dir_path(__FILE__));

// Check if Ultimate Multisite is active
add_action('plugins_loaded', function() {
    if (!function_exists('WP_Ultimo')) {
        add_action('admin_notices', function() {
            ?>
            <div class="notice notice-error">
                <p><?php _e('My Addon requires Ultimate Multisite to be activated.', 'my-addon'); ?></p>
            </div>
            <?php
        });
        return;
    }

    // Initialize addon
    require_once MY_ADDON_PATH . 'inc/class-addon.php';
    Addon::get_instance();
});
```

### Using Hooks

```php
// Site provisioning
add_action('wu_site_published', function($site, $membership) {
    // Install essential plugins
    switch_to_blog($site->get_id());

    activate_plugin('contact-form-7/wp-contact-form-7.php');
    activate_plugin('wordfence/wordfence.php');

    // Configure site
    update_option('blogdescription', 'Powered by ' . $membership->get_plan()->get_name());

    restore_current_blog();
}, 10, 2);

// Customer management
add_action('wu_customer_post_create', function($customer) {
    // Send welcome email
    wp_mail(
        $customer->get_email(),
        'Welcome to Our Platform',
        'Thanks for signing up!'
    );

    // Create CRM record
    my_crm_sync($customer);
});

// Membership changes
add_action('wu_membership_status_to_expired', function($membership) {
    // Suspend all sites
    $sites = $membership->get_sites();

    foreach ($sites as $site) {
        $site->set_active_until(null);
        $site->save();
    }
});
```

### Adding Admin Pages

```php
namespace My_Addon\Admin_Pages;

class Settings_Page extends \WP_Ultimo\Admin_Pages\Base_Admin_Page {

    protected $id = 'my-addon-settings';
    protected $position = 99;

    public function page_menu() {
        add_submenu_page(
            'wp-ultimo',
            __('My Addon Settings', 'my-addon'),
            __('My Addon', 'my-addon'),
            'manage_network',
            $this->id,
            [$this, 'output']
        );
    }

    public function output() {
        wu_get_template('admin/settings', [
            'page' => $this,
            'settings' => $this->get_settings(),
        ]);
    }
}
```

## Testing

### Unit Tests

```php
namespace Tests\WP_Ultimo\Models;

class Customer_Test extends \WP_UnitTestCase {

    public function setUp() {
        parent::setUp();

        // Create test user
        $this->user_id = $this->factory->user->create([
            'user_login' => 'testcustomer',
            'user_email' => 'test@example.com',
        ]);
    }

    public function test_create_customer() {
        $customer = wu_create_customer([
            'user_id' => $this->user_id,
            'email_verification' => 'verified',
        ]);

        $this->assertInstanceOf('\WP_Ultimo\Models\Customer', $customer);
        $this->assertEquals($this->user_id, $customer->get_user_id());
        $this->assertEquals('verified', $customer->get_email_verification());
    }

    public function test_customer_meta() {
        $customer = wu_create_customer(['user_id' => $this->user_id]);

        $customer->add_meta('test_key', 'test_value');
        $this->assertEquals('test_value', $customer->get_meta('test_key'));

        $customer->update_meta('test_key', 'new_value');
        $this->assertEquals('new_value', $customer->get_meta('test_key'));
    }

    public function tearDown() {
        wp_delete_user($this->user_id);
        parent::tearDown();
    }
}
```

### Running Specific Tests

```bash
# Run all tests
vendor/bin/phpunit

# Run specific test file
vendor/bin/phpunit tests/WP_Ultimo/Models/Customer_Test.php

# Run specific test method
vendor/bin/phpunit --filter test_create_customer

# Run with coverage
vendor/bin/phpunit --coverage-html coverage-html
```

### E2E Testing with Cypress

```javascript
// tests/e2e/specs/checkout.spec.js
describe('Checkout Flow', () => {
    beforeEach(() => {
        cy.visit('/register');
    });

    it('completes signup successfully', () => {
        // Fill in username
        cy.get('input[name="username"]').type('newuser');

        // Fill in email
        cy.get('input[name="email"]').type('user@example.com');

        // Select plan
        cy.get('[data-product-id="1"]').click();

        // Fill in site details
        cy.get('input[name="site_url"]').type('mysite');
        cy.get('input[name="site_title"]').type('My Test Site');

        // Submit
        cy.get('button[type="submit"]').click();

        // Assert success
        cy.url().should('include', '/thank-you');
        cy.contains('Welcome to your new site');
    });
});
```

## Debugging

### Enable Debug Mode

```php
// wp-config.php
define('WP_DEBUG', true);
define('WP_DEBUG_LOG', true);
define('WP_DEBUG_DISPLAY', false);
define('SCRIPT_DEBUG', true);
```

### Using the Logger

```php
// Log debug information
wu_log_add('debug', 'Debug message', [
    'context' => 'checkout',
    'data' => $cart->get_data(),
]);

// Log errors
wu_log_add('error', 'Payment failed', [
    'gateway' => 'stripe',
    'error' => $e->getMessage(),
]);

// View logs
// Admin: WP Ultimo > System Info > Logs
```

### Query Monitoring

```php
// Enable query debugging
define('SAVEQUERIES', true);

// Check queries
global $wpdb;
print_r($wpdb->queries);
```

### Debugging Hooks

```php
// Log all hooks
add_action('all', function($hook) {
    if (strpos($hook, 'wu_') === 0) {
        error_log('Hook: ' . $hook);
    }
});
```

## Common Tasks

### Creating a Custom Limitation

```php
namespace My_Addon\Limitations;

class Custom_Feature_Limitation extends \WP_Ultimo\Limitations\Base_Limitation {

    public $id = 'custom_feature';

    public function __construct() {
        $this->title = __('Custom Feature Access', 'my-addon');
        parent::__construct();
    }

    public function is_allowed($site_id, $membership) {
        $plan = $membership->get_plan();

        // Check if plan includes this feature
        $allowed = $plan->get_limitation('custom_feature_enabled');

        if (!$allowed) {
            $this->show_upgrade_notice();
            return false;
        }

        return true;
    }
}

// Register limitation
add_filter('wu_limitations', function($limitations) {
    $limitations['custom_feature'] = 'My_Addon\Limitations\Custom_Feature_Limitation';
    return $limitations;
});
```

### Adding Custom Reports

```php
// Add report to dashboard
add_filter('wu_dashboard_widgets', function($widgets) {
    $widgets['my_report'] = [
        'title' => __('My Custom Report', 'my-addon'),
        'callback' => 'my_addon_render_report',
        'position' => 'normal',
    ];

    return $widgets;
});

function my_addon_render_report() {
    $data = get_custom_report_data();

    wu_get_template('widgets/custom-report', [
        'data' => $data,
    ]);
}
```

### Syncing with External Services

```php
// Sync customer to external CRM
add_action('wu_customer_post_create', 'sync_to_crm');
add_action('wu_customer_post_update', 'sync_to_crm');

function sync_to_crm($customer) {
    $crm = new My_CRM_API();

    $crm_data = [
        'email' => $customer->get_email(),
        'name' => $customer->get_display_name(),
        'created_at' => $customer->get_date_registered(),
    ];

    // Check if customer exists in CRM
    $crm_id = $customer->get_meta('crm_id');

    if ($crm_id) {
        // Update existing
        $crm->update_contact($crm_id, $crm_data);
    } else {
        // Create new
        $result = $crm->create_contact($crm_data);
        $customer->add_meta('crm_id', $result->id);
    }
}
```

## Performance Tips

### Caching

```php
// Cache expensive operations
$cache_key = 'my_expensive_data_' . $customer_id;
$data = wp_cache_get($cache_key);

if (false === $data) {
    $data = perform_expensive_operation($customer_id);
    wp_cache_set($cache_key, $data, '', 3600); // 1 hour
}
```

### Efficient Queries

```php
// Bad: N+1 query problem
$customers = wu_get_customers();
foreach ($customers as $customer) {
    $membership = $customer->get_membership(); // Query each time
}

// Good: Prefetch relationships
$customers = wu_get_customers([
    'with' => ['memberships'], // Prefetch memberships
]);
foreach ($customers as $customer) {
    $membership = $customer->get_membership(); // No additional query
}
```

### Background Processing

```php
// Use Action Scheduler for heavy tasks
add_action('wu_site_published', function($site) {
    // Schedule background task
    as_schedule_single_action(
        time() + 60, // Run in 1 minute
        'my_addon_setup_site',
        [$site->get_id()],
        'my-addon'
    );
});

// Process in background
add_action('my_addon_setup_site', function($site_id) {
    // Heavy processing here
    my_addon_install_plugins($site_id);
    my_addon_configure_settings($site_id);
});
```

## Security Best Practices

### Input Sanitization

```php
// Sanitize based on expected input type
$username = sanitize_user($_POST['username']);
$email = sanitize_email($_POST['email']);
$url = esc_url_raw($_POST['site_url']);
$text = sanitize_text_field($_POST['company_name']);
$textarea = sanitize_textarea_field($_POST['description']);
```

### Output Escaping

```php
// Escape based on context
echo esc_html($customer->get_name());
echo esc_attr($customer->get_email());
echo esc_url($customer->get_avatar_url());
echo wp_kses_post($customer->get_bio());
```

### Nonces

```php
// Generate nonce
wp_nonce_field('my_action', 'my_nonce');

// Verify nonce
if (!wp_verify_nonce($_POST['my_nonce'], 'my_action')) {
    wp_die(__('Security check failed', 'my-addon'));
}
```

### Capability Checks

```php
// Check capabilities
if (!current_user_can('manage_network')) {
    wp_die(__('Insufficient permissions', 'my-addon'));
}

// Customer permissions
$customer = wu_get_current_customer();
if (!$customer || !$customer->can('manage_sites')) {
    wp_die(__('You cannot perform this action', 'my-addon'));
}
```

### SQL Prepared Statements

```php
global $wpdb;

// Bad
$results = $wpdb->get_results("SELECT * FROM {$wpdb->prefix}wu_customers WHERE email = '{$email}'");

// Good
$results = $wpdb->get_results(
    $wpdb->prepare(
        "SELECT * FROM {$wpdb->prefix}wu_customers WHERE email = %s",
        $email
    )
);
```

## Additional Resources

- **API Documentation:** See `DEVELOPER-DOCUMENTATION.md`
- **Contributing Guide:** See `CONTRIBUTING.md`
- **Main README:** See `README.md`
- **GitHub Issues:** https://github.com/Multisite-Ultimate/ultimate-multisite/issues
- **GitHub Wiki:** `.wiki/` directory for integration guides

---

**Questions?** Open an issue or discussion on GitHub!