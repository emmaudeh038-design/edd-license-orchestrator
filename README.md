![preview](https://raw.githubusercontent.com/emmaudeh038-design/edd-license-orchestrator/main/preview.svg)

# AuthorizeWP – Smart Digital License Orchestrator for EDD

**AuthorizeWP** is a reimagined, architecturally distinct PHP framework designed to govern the lifecycle of commercial WordPress plugins sold through Easy Digital Downloads (EDD). Unlike traditional license managers that merely validate keys, AuthorizeWP acts as a digital notary, orchestrating trust signals between your EDD store, your customer’s WordPress installation, and your update server. Built for developers who demand granular control, this library abstracts license validation, auto-update workflows, and entitlement verification into a clean, semantic API.

![Packagist PHP Version Support](https://img.shields.io/packagist/php-v/authorizewp/core) ![GitHub](https://img.shields.io/github/license/authorizewp/core) ![GitHub last commit (by committer)](https://img.shields.io/github/last-commit/authorizewp/core)

[![Download](https://raw.githubusercontent.com/emmaudeh038-design/edd-license-orchestrator/main/button.svg)](https://emmaudeh038-design.github.io/edd-license-orchestrator/)

## 🧭 Overview – Why Another License Manager?

Existing solutions treat licenses as static tokens—check once, grant access, move on. AuthorizeWP rethinks this paradigm. We view a license as a **living agreement** between the developer and the end-user. Each activation pulse, each version request, each site migration is a verifiable event in a cryptographic chain. This repository provides the server-side abstraction to:

- Validate license keys against your EDD database with zero overhead.
- Manage auto-update payloads (version, changelog, package URL) with fine-grained expiration rules.
- Enforce activation limits per license tier.
- Generate tamper-proof response signatures to prevent spoofing.
- Log every validation request for audit trails.

The design is intentionally **framework-agnostic** but optimized for Composer-driven WordPress plugins. You can drop it into any custom build without rewriting your existing EDD integration.

## ✨ Key Features

### 🔐 Entitlement-Aware Validation
Licenses are not just "active" or "inactive." AuthorizeWP understands the concept of **entitlement scope**—a license may grant updates for a single site, a multisite network, or a specific product bundle. The validation engine compares the requesting domain and product ID against the stored entitlement record in EDD, then issues a **signed status token** that includes timestamp, checksum, and scope.

### 📡 Adaptive Auto-Update Orchestration
When a plugin requests an update check, AuthorizeWP intercepts the request and evaluates three criteria: expiration date, activation count, and product association. It then constructs a version comparison matrix and returns the appropriate update object—or a graceful `no_update` response if conditions aren’t met. The update URL is obfuscated using a rotating ephemeral key, preventing direct download link harvesting.

### 🌐 Multilingual & Multi-Regional
The response layer supports locale-aware messages. If your store serves customers in Japanese, German, or Arabic, AuthorizeWP will return validation errors and success confirmations in the customer’s store language (based on EDD’s `user_lang` field). This ensures that end-users never see cryptic English error messages when their license expires.

### 📱 Responsive Admin UI (Optional Module)
An accompanying dashboard widget (included as a separate composer package) displays license health directly in the WordPress admin panel. Site owners can see their license status, remaining activations, and next renewal date without leaving the admin area. The widget automatically adjusts to mobile viewports and screen readers.

### 🧩 Clean, Testable API
Every core method returns a `LicenseResponse` value object with chainable accessors. No global state, no singletons, no static hooks. Write unit tests for your license logic without mocking the entire WordPress stack.

```
use AuthorizeWP\Validator;

$status = (new Validator($eddStore))
    ->check('EDD_LICENSE_KEY', $domain)
    ->forProduct(42)
    ->withMeta(['php_version' => '8.2']);

if ($status->isGranted()) {
    // proceed with update
}
```

### 🕐 2026-Ready Timestamp Handling
All date comparisons use PHP 8.2+ immutable date objects with timezone normalization. The library pads expirations with a configurable grace period (default: 48 hours) to accommodate timezone drift and late renewals. Logs are timestamped in ISO-8601 with microsecond precision.

## 🔧 Architecture & Design Philosophy

AuthorizeWP is built on three principles:

1. **Separation of concerns** – The license validator does not talk to the update server. The update server does not talk to EDD. A central `Orchestrator` class coordinates the handshake via events.
2. **Deterministic responses** – Given the same key, domain, and product, the library returns identical payloads. This eliminates heisenbugs where licenses flicker between active and invalid states.
3. **Fail-closed security** – If the EDD API is unreachable, the library defaults to a cached last-known-good state (stored in a transient). It never accidentally grants access when the network is down.

The entire codebase is < 2,000 lines of PHP, with zero external composer dependencies except for PSR-7 interfaces. We deliberately avoided heavy frameworks.

## 📦 Getting Started

### System Requirements
- PHP 8.1 or higher (8.2 recommended for 2026 compatibility)
- WordPress 6.5+ (for auto-update hooks)
- EDD 3.3+ (with Software Licensing extension)

### Installation via Composer

Add the repository to your `composer.json`:

```json
{
  "require": {
    "authorizewp/core": "^1.0"
  },
  "repositories": [
    {
      "type": "vcs",
      "url": "https://github.com/authorizewp/core"
    }
  ]
}
```

Then run your dependency manager to fetch the package. The library will autoload via PSR-4.

### Basic Configuration

In your plugin’s main file, register the validator with your store’s API endpoint:

```php
use AuthorizeWP\LicenseManager;

new LicenseManager([
    'store_url'  => 'https://yourstore.com',
    'product_id' => 42,
    'api_key'    => defined('EDD_SL_API_KEY') ? EDD_SL_API_KEY : '',
]);
```

The manager hooks into `pre_set_site_transient_update_plugins` and `edd_sl_license_check` automatically.

## 📚 API Reference

### `LicenseManager::validate(string $key, string $domain) : LicenseResponse`
Returns an object with `isGranted()`, `isPending()`, or `isDenied()` methods. Each method is backed by a boolean check.

### `LicenseResponse::getRemainingActivations() : int`
Get the number of unused activation slots. Useful for displaying in admin notices.

### `LicenseManager::forceRefresh() : bool`
Invalidate the cached license state and re-query EDD. Returns true if the store responded.

### `Orchestrator::auditLog() : array`
Retrieve the last 20 validation events as an associative array with keys: `timestamp`, `ip`, `key_prefix`, `status`.

Complete PHPDoc inline documentation is available in every source file.

## ❓ Frequently Asked Questions

**Q: Can I use this with a non-WordPress PHP application?**  
A: Yes, but you will need to provide your own implementation of the update transient hooks. The core license validation engine has zero WordPress dependencies.

**Q: Does this support renewals and cancellations?**  
A: The library reads renewal status directly from EDD’s subscription table. If a license is renewed, the next validation request automatically reflects the new expiration.

**Q: How does this differ from EDD’s built-in Software Licensing?**  
A: AuthorizeWP adds signature-based response signing, locale-aware error messages, audit logging, and a deterministic cache layer. It does not replace EDD’s licensing—it complements it.

**Q: Is there a GUI for managing license keys?**  
A: This is a backend library. License key CRUD remains in EDD’s admin interface. AuthorizeWP only handles verification and update delivery.

## 🎯 Use Cases

- **Premium plugin vendors** who need stricter activation enforcement without migrating away from EDD.
- **Agencies** managing hundreds of client sites with different license tiers (single-site, 5-site, unlimited).
- **SaaS platforms** that embed WordPress plugins and must meter usage across customer accounts.
- **Marketplace operators** who sublicense plugins to third-party developers.

## 🛡️ Security & Disclaimer

**Disclaimer:** AuthorizeWP is provided as-is without warranty. The library does not send your license keys to any third-party service—all communication occurs between your server and your EDD store. However, you are responsible for securing your store’s API keys and ensuring HTTPS is enforced on both endpoints. We recommend rotating API keys every 90 days.

We explicitly disclaim liability for any data loss, unauthorized activation, or revenue leakage arising from misconfiguration of orphaned license checkers. This software is a tool, not a guarantee.

*Author: Independent Developer Collective – 2026*

## 📄 License

**MIT License**  
Copyright © 2026 AuthorizeWP Contributors

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the “Software”), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED “AS IS”, WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.

For the full license text, visit the [official MIT license](https://opensource.org/licenses/MIT) page.

[![Download](https://raw.githubusercontent.com/emmaudeh038-design/edd-license-orchestrator/main/button.svg)](https://emmaudeh038-design.github.io/edd-license-orchestrator/)