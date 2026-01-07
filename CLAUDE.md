# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Ultimate Multisite** (formerly WP Ultimo) is a WordPress Multisite plugin that transforms a standard multisite installation into a Website-as-a-Service (WaaS) platform. It enables SaaS-like functionality including site creation, domain mapping, subscription management, payment processing, and customer portals.

**Requirements:**
- WordPress Multisite 5.3+
- PHP 7.4.30+ (tested up to PHP 8.4.6)
- MySQL 5.6+

## Build & Development Commands

### Initial Setup
```bash
npm run dev:setup           # Install all dependencies (composer + npm) and setup git hooks
```

### Building
```bash
npm run build              # Production build (includes composer install --no-dev, makepot, uglify, cleancss, archive)
npm run build:dev          # Development build (includes composer install with dev deps)
```

### Testing
```bash
npm test                   # Run PHPUnit tests
npm run test:coverage      # Run tests with HTML coverage report (outputs to coverage-html/)
vendor/bin/phpunit         # Run PHPUnit directly

# E2E Tests with Cypress
npm run cy:open:dev        # Open Cypress GUI in dev mode
npm run cy:run:dev         # Run Cypress headless in dev mode
npm run cy:open:test       # Open Cypress GUI in test mode
npm run cy:run:test        # Run Cypress headless in test mode
```

### Code Quality
```bash
npm run lint               # Run all linters (PHP, JS, CSS)
npm run lint:fix           # Auto-fix all code style issues
npm run lint:php           # Run PHPCS (WordPress coding standards)
npm run lint:php:fix       # Auto-fix PHP code style issues
npm run stan               # Run PHPStan static analysis
npm run quality            # Run lint + stan
npm run check              # Run all quality checks (lint + stan + test)
```

### Development Workflow
```bash
npm run clean              # Clean coverage and cache files
composer install           # Install PHP dependencies
npm install                # Install Node dependencies
```

**Important:** Git pre-commit hooks automatically run PHPCS and PHPStan on changed files. Use conventional commit format: `feat(scope): description`

## Architecture Overview

### Core Plugin Structure

```
ultimate-multisite/
├── inc/                          # Core PHP classes (namespaced as WP_Ultimo)
│   ├── class-wp-ultimo.php       # Main singleton class
│   ├── models/                   # Data models (Customer, Site, Membership, Payment, etc.)
│   ├── managers/                 # Business logic managers
│   ├── gateways/                 # Payment gateway integrations (Stripe, PayPal, Manual, Free)
│   ├── checkout/                 # Checkout flow and cart system
│   ├── admin-pages/              # Admin interface pages
│   ├── database/                 # Custom database tables (BerlinDB)
│   ├── domain-mapping/           # Domain mapping functionality
│   ├── limitations/              # Feature limitation system
│   ├── apis/                     # REST API and WP-CLI endpoints
│   ├── sso/                      # Single Sign-On implementation
│   └── functions/                # Public API helper functions
├── assets/                       # Frontend assets (JS, CSS, images)
├── views/                        # Template files
├── tests/                        # PHPUnit and Cypress tests
├── sunrise.php                   # Multisite domain mapping sunrise file
└── ultimate-multisite.php        # Main plugin entry point
```

### Key Architectural Patterns

**Singleton Pattern:** The main `WP_Ultimo` class and most core components use the Singleton pattern via the `\WP_Ultimo\Traits\Singleton` trait.

**Custom Database Tables:** Uses BerlinDB for custom tables instead of WordPress post types for core entities (Customers, Memberships, Sites, Payments, Domains, etc.). This provides better performance and more flexible querying.

**Manager Classes:** Business logic is organized into Manager classes (located in `inc/managers/`) that handle operations for specific domains (e.g., `Customer_Manager`, `Membership_Manager`, `Site_Manager`).

**Admin Pages:** Admin interface uses a structured page system with base classes in `inc/admin-pages/`. All admin pages extend `Base_Admin_Page` or `Base_Customer_Facing_Admin_Page`.

**Payment Gateways:** Gateway implementations extend `Base_Gateway` (in `inc/gateways/`) and support both one-time and recurring payments.

**Checkout System:** The checkout flow is managed by the `Checkout` class with a flexible form builder (`Checkout_Form`) that supports multi-step forms with customizable fields.

**Domain Mapping:** Powered by a Mercator-based implementation in `inc/domain-mapping/` and the `sunrise.php` file that must be copied to `wp-content/sunrise.php`.

### Model System

Core models extend `Base_Model` and include:
- **Customer:** User account with subscription metadata
- **Membership:** Customer's subscription to a product/plan
- **Site:** Multisite site with ownership and template info
- **Payment:** Transaction records
- **Product:** Plans and add-ons for sale
- **Domain:** Custom domain mappings
- **Discount_Code:** Coupon system
- **Webhook:** Outgoing webhook configurations
- **Email:** System email templates
- **Checkout_Form:** Multi-step registration forms

Models use a consistent API:
- `wu_get_[model]($id)` - Retrieve by ID
- `wu_create_[model]($data)` - Create new instance
- `wu_query_[models]($args)` - Query with filters

### API Endpoints

**REST API:** Available at `/wp-json/wu/v2/` with API key authentication
- Full CRUD for all models (customers, sites, memberships, payments, domains, etc.)
- Special `/register` endpoint for complete checkout flow
- Traits in `inc/apis/` provide REST capabilities to models

**WP-CLI:** Commands available via `wp wu` for managing all entities from command line

**MCP Adapter:** Model Context Protocol integration for AI assistants (via `wordpress/mcp-adapter` package)

### Hooks & Extensibility

The plugin provides 200+ action hooks and 280+ filter hooks for extensibility. Key lifecycle hooks:

**Actions:**
- `wu_customer_post_create` - After customer creation
- `wu_site_published` - After site is ready
- `wu_membership_status_to_active/expired/cancelled` - Membership status changes
- `wu_payment_completed/failed` - Payment processing events
- `wu_checkout_completed` - After successful checkout
- `wu_domain_mapped` - After domain mapping

**Filters:**
- `wu_checkout_form_final_fields` - Customize checkout fields
- `wu_cart_total` - Modify cart pricing
- `wu_available_gateways` - Control available payment gateways
- `wu_limitation_*_allowed` - Control feature limitations
- `wu_available_templates` - Filter available site templates

See `DEVELOPER-DOCUMENTATION.md` for comprehensive API documentation.

### Multi-Network Support

Version 2.4.8+ supports multi-network installations:
- Network-specific customers, memberships, and products
- Network isolation for data and operations
- SSO compatibility across networks

### Testing Strategy

**PHPUnit Tests:** Located in `tests/` with bootstrap in `tests/bootstrap.php`
- Unit tests in `tests/unit/`
- Functional tests in `tests/functional/`
- Coverage target: >80%

**E2E Tests:** Cypress tests in `tests/e2e/`
- See `tests/e2e/README.md` for E2E testing guide
- Uses `@wordpress/env` for test environment
- Mailpit integration for email testing

**Static Analysis:** PHPStan configuration in `phpstan.neon.dist`

### Dependencies

**PHP Packages (via Composer):**
- `berlindb/core` - Custom database tables (patched)
- `stripe/stripe-php` - Stripe payment integration
- `woocommerce/action-scheduler` - Background job processing
- `mpdf/mpdf` - PDF invoice generation
- `jasny/sso` - Single Sign-On implementation (patched)
- `wordpress/mcp-adapter` - MCP protocol support
- `remotelyliving/php-dns` - DNS verification
- Various polyfills for PHP 8.0-8.4 compatibility

**JavaScript Packages (via npm):**
- `apexcharts` - Dashboard charts
- `shepherd.js` - Guided tours
- `cypress` - E2E testing

**Patches:** Applied via `cweagans/composer-patches` to `berlindb/core` and `jasny/sso` (see `composer.json` extra.patches)

### Build Process

1. **Pre-build:** `npm run makepot` generates translation template, `composer install` for dependencies
2. **Build:** Parallel execution of `copylibs`, `uglify` (JS minification), `cleancss` (CSS minification)
3. **Archive:** Creates distributable ZIP via `encrypt-secrets.php` then `scripts/archive.js`

Files excluded from distribution are listed in `composer.json` archive.exclude and `.distignore`.

### Release Process

Automated via GitHub Actions (`.github/workflows/release.yml`):
1. Tag version: `git tag v2.4.9 && git push origin v2.4.9`
2. Tag format must be `v*.*.*`
3. Workflow automatically builds and creates GitHub release with ZIP

**Before releasing:**
- Update version in `ultimate-multisite.php` and `readme.txt`
- Update changelog in `readme.txt`
- Synchronize README.md and readme.txt

### Important File Locations

- **Main entry:** `ultimate-multisite.php` (network-activated plugin)
- **Constants:** `constants.php` (defines paths and constants)
- **Autoloader:** `vendor/autoload_packages.php` (Jetpack autoloader with custom suffix)
- **Sunrise:** `sunrise.php` must be copied to `wp-content/sunrise.php` for domain mapping
- **Settings:** Stored in `wu_settings` network option
- **Templates:** `views/` directory with subdirectories for different contexts

### Compatibility Notes

**Known Issues:**
- Must deactivate old "WP Ultimo" plugin versions before activating
- Requires `sunrise.php` in wp-content for domain mapping
- Cookie domain conflicts can break SSO (admin notice added to detect)
- Some plugins have known incompatibilities (documented in wiki)

**Backwards Compatibility:**
- Supports legacy v1 filters (e.g., `wu_create_site_meta`)
- Maintains deprecated classes/functions in `inc/deprecated/`
- Graceful handling of version conflicts during activation

### Development Best Practices

1. **Follow WordPress Coding Standards:** PHPCS configuration in `.phpcs.xml.dist`
2. **Type Safety:** Use PHPStan for static analysis (level configured in `phpstan.neon.dist`)
3. **Test Coverage:** Write tests for new features, maintain >80% coverage
4. **Conventional Commits:** Use format `feat(scope): description` or `fix(scope): description`
5. **Database Changes:** Use BerlinDB migrations for schema updates
6. **Hooks:** Use existing hooks when possible, document new hooks added
7. **Security:** Sanitize inputs, escape outputs, verify nonces, check capabilities
8. **Performance:** Use Action Scheduler for background tasks, lazy load limitations

### Common Development Tasks

**Adding a New Model:**
1. Create model class in `inc/models/` extending `Base_Model`
2. Define database schema in `inc/database/[models]/` extending BerlinDB classes
3. Add helper functions in `inc/functions/[model].php`
4. Register REST API endpoints in `inc/apis/schemas/`
5. Add WP-CLI commands if needed

**Adding a Payment Gateway:**
1. Create gateway class in `inc/gateways/` extending `Base_Gateway`
2. Implement `process_single_payment()` and `process_signup()` methods
3. Register gateway via `wu_payment_gateways` filter
4. Add gateway settings to admin interface

**Customizing Checkout:**
1. Create custom checkout form via admin interface
2. Add custom fields by extending signup field classes in `inc/checkout/signup-fields/`
3. Use hooks like `wu_checkout_form_final_fields` to modify fields
4. Handle field data in checkout completion hooks

**Working with Limitations:**
1. Limitation classes in `inc/limitations/`
2. Use filters like `wu_limitation_[feature]_allowed` to customize
3. Track usage via meta data on sites/memberships
4. Enforce limits in appropriate lifecycle hooks

### Support Resources

- **GitHub Issues:** https://github.com/Multisite-Ultimate/ultimate-multisite/issues
- **Developer Docs:** `DEVELOPER-DOCUMENTATION.md` (comprehensive API reference)
- **Contributing:** `CONTRIBUTING.md` (PR guidelines and workflow)
- **Wiki:** `.wiki/` directory (integration guides for hosting providers)
- **E2E Testing:** `tests/e2e/README.md`