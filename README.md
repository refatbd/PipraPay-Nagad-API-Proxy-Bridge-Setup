# Nagad API Proxy Bridge Setup

## Overview
This solution enables a PHP application hosted on a foreign server (e.g., Singapore VPS) to communicate with the Nagad Payment Gateway by routing traffic through a proxy script hosted on a Bangladeshi server. 

This guide covers:
1. Setting up the **Bridge Script** on your BD hosting.
2. Adding **Proxy Settings** to the WordPress Admin Panel.
3. Integrating the proxy logic into the Nagad library.

**Architecture:**
`App (Singapore VPS)`  **&rarr;** `Bridge Script (BD Shared Hosting)` **&rarr;** `Nagad API`

---

## Prerequisites
1.  **Foreign Server:** Your main application/VPS (e.g., Singapore).
2.  **BD Server:** A shared hosting account or server physically located in Bangladesh (no SSH required).
3.  **Nagad Whitelist:** The IP address of your **BD Server** must be whitelisted by Nagad.

---

## Step 1: Server-Side Setup (Bangladesh Host)

1.  Create a file named `nagad-bridge.php` on your Bangladeshi hosting public directory (e.g., `public_html`).
2.  Add the following code to the file.

**File:** `nagad-bridge.php`

```php
<?php
/**
 * Nagad API Bridge/Proxy Script
 * Hosted on: Bangladesh Server
 */

// 1. CONFIGURATION
// Ensure there are no spaces around your key
define('PROXY_SECRET', 'YOUR_STRONG_SECRET_KEY_HERE');

// 2. RECEIVE INPUT
$input = file_get_contents('php://input');
$data = json_decode($input, true);

// 3. AUTHENTICATION & SECURITY CHECK
if (!isset($data['secret']) || trim($data['secret']) !== PROXY_SECRET) {
    // Return 403 Forbidden header
    http_response_code(403);
    
    // List of funny messages
    $msgs = [
        "These aren't the droids you're looking for.",
        "Nothing to see here, just a lonely server pixel.",
        "Houston, we have a problem. You are lost!",
        "Oops! You took a wrong turn at the internet.",
        "Error 404: Pizza not found.",
        "Why are you here? Go build something amazing!"
    ];
    $msg = $msgs[array_rand($msgs)];

    // Output HTML using PHP echo to prevent formatting/syntax errors
    echo '<!DOCTYPE html>
    <html lang="en">
    <head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <title>Restricted Area</title>
        <style>
            body { background: linear-gradient(135deg, #667eea 0%, #764ba2 100%); height: 100vh; margin: 0; display: flex; align-items: center; justify-content: center; font-family: sans-serif; color: white; text-align: center; }
            .container { background: rgba(255, 255, 255, 0.15); padding: 3rem; border-radius: 24px; width: 450px; max-width: 90%; box-shadow: 0 8px 32px 0 rgba(31, 38, 135, 0.37); }
            h1 { margin-top: 0; }
            .emoji { font-size: 4rem; display: block; margin-bottom: 1rem; }
        </style>
    </head>
    <body>
        <div class="container">
            <span class="emoji">👻</span>
            <h1>Access Denied</h1>
            <p>' . $msg . '</p>
        </div>
    </body>
    </html>';
    
    // Stop execution strictly
    exit;
}

// ---------------------------------------------------------
// PROXY LOGIC (Only runs if Key is Correct)
// ---------------------------------------------------------

$ch = curl_init();
$target_url = isset($data['target_url']) ? $data['target_url'] : '';
$method = isset($data['method']) ? $data['method'] : 'POST';
$headers = isset($data['headers']) ? $data['headers'] : [];
$payload = isset($data['body']) ? $data['body'] : null;

if (empty($target_url)) {
    echo json_encode(['error' => 'No target URL provided']);
    exit;
}

// Configure cURL
curl_setopt($ch, CURLOPT_URL, $target_url);
curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
curl_setopt($ch, CURLOPT_FOLLOWLOCATION, true);
curl_setopt($ch, CURLOPT_SSL_VERIFYHOST, 0); 
curl_setopt($ch, CURLOPT_SSL_VERIFYPEER, 0);
curl_setopt($ch, CURLOPT_HTTPHEADER, $headers);

if ($method === 'POST') {
    curl_setopt($ch, CURLOPT_POST, true);
    $post_fields = is_array($payload) ? json_encode($payload) : $payload;
    curl_setopt($ch, CURLOPT_POSTFIELDS, $post_fields);
}

// Execute
$response = curl_exec($ch);
$error = curl_error($ch);
curl_close($ch);

// Return Response
if ($error) {
    header('Content-Type: application/json');
    echo json_encode(['proxy_error' => $error]);
} else {
    // IMPORTANT: Do NOT set JSON header here, just pass the raw response from Nagad
    echo $response;
}
?>
```

---

## Step 2: Add Proxy Settings to Admin UI

To avoid hardcoding credentials, add configuration fields to the plugin's settings page.

**Target File:** `views/admin-ui.php`

Open the file and find the closing `</div>` tag of the **Configuration** card (usually just before `<div id="ajaxResponse">`). Insert the following code block:

```php
<div class="row mb-4">
    <div class="col-sm-12">
        <hr>
        <h4 class="card-title h6 mb-3">Proxy Configuration (Optional)</h4>
    </div>

    <div class="col-sm-6">
        <label for="proxy_url" class="col-sm-12 col-form-label form-label">Proxy Bridge URL</label>
        <div class="input-group">
            <input type="text" class="form-control" name="proxy_url" id="proxy_url" 
                   placeholder="[https://your-bd-domain.com/nagad-bridge.php](https://your-bd-domain.com/nagad-bridge.php)"
                   value="<?= htmlspecialchars($settings['proxy_url'] ?? '') ?>">
        </div>
        <div class="text-secondary mt-2 small">Full URL to the bridge script on your BD hosting.</div>
    </div>

    <div class="col-sm-6">
        <label for="proxy_secret" class="col-sm-12 col-form-label form-label">Proxy Secret Key</label>
        <div class="input-group">
            <input type="password" class="form-control" name="proxy_secret" id="proxy_secret" 
                   placeholder="Your Secret Key"
                   value="<?= htmlspecialchars($settings['proxy_secret'] ?? '') ?>">
        </div>
        <div class="text-secondary mt-2 small">The secret key defined in your bridge script.</div>
    </div>
</div>
```

---

## Step 3: Integrate Logic in `Helper.php`

Modify the library to check for the Admin Panel settings and route traffic through the bridge if they exist.

**Target File:** `vendor-nagad-merchant/xenon/nagad-api/src/Helper.php`

Add the `getProxyConfig` method and replace `httpPostMethod` and `httpGet` with the versions below:

```php
    /**
     * Helper to get Proxy Config dynamically
     */
    private function getProxyConfig() {
        // Fetch settings for the plugin. 
        // NOTE: Ensure 'nagad-merchant-api' matches your actual plugin slug.
        $settings = function_exists('\pp_get_plugin_setting') ? \pp_get_plugin_setting('nagad-merchant-api') : [];
        
        return [
            'url' => $settings['proxy_url'] ?? '',
            'secret' => $settings['proxy_secret'] ?? ''
        ];
    }

    /**
     * Modified httpPostMethod (Uses Admin Settings)
     */
    public function httpPostMethod(string $postUrl, array $postData)
    {
        $proxy = $this->getProxyConfig();

        // If Proxy URL is NOT set in admin, fallback to normal direct request logic
        // (You can optionally uncomment the error below if you WANT to force proxy)
        if (empty($proxy['url'])) {
             // return ['error' => 'Proxy URL is missing in Admin Settings']; // Optional strict mode
             // For now, let's just error out or you can paste the original logic here for fallback
             return ['error' => 'Proxy URL is missing. Please configure it in Gateway Settings.'];
        }

        $header = array(
            'Content-Type:application/json',
            'X-KM-Api-Version:v-0.2.0',
            'X-KM-IP-V4:' . $this->getClientIP(),
            'X-KM-Client-Type:PC_WEB'
        );

        $bridgePayload = [
            'secret' => $proxy['secret'],
            'target_url' => $postUrl,
            'method' => 'POST',
            'headers' => $header,
            'body' => $postData
        ];

        $ch = curl_init($proxy['url']);
        curl_setopt($ch, CURLOPT_CUSTOMREQUEST, "POST");
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
        curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($bridgePayload));
        curl_setopt($ch, CURLOPT_HTTPHEADER, ['Content-Type:application/json']);
        curl_setopt($ch, CURLOPT_FOLLOWLOCATION, 1);
        curl_setopt($ch, CURLOPT_SSL_VERIFYHOST, 0);
        curl_setopt($ch, CURLOPT_SSL_VERIFYPEER, 0);
        
        $resultData = curl_exec($ch);
        $curl_error = curl_error($ch);

        if (!empty($curl_error)) {
            return ['error' => 'Bridge Error: ' . $curl_error];
        }

        $response = json_decode($resultData, true);
        curl_close($ch);
        return $response;
    }

    /**
     * Modified httpGet (Uses Admin Settings)
     */
    public static function httpGet($url)
    {
        // Fetch settings. Ensure slug matches your plugin.
        $settings = function_exists('\pp_get_plugin_setting') ? \pp_get_plugin_setting('nagad-merchant-api') : [];
        $proxyUrl = $settings['proxy_url'] ?? '';
        $proxySecret = $settings['proxy_secret'] ?? '';

        if (empty($proxyUrl)) {
             throw new NagadPaymentException('Proxy URL is missing in Admin Settings');
        }

        $bridgePayload = [
            'secret' => $proxySecret,
            'target_url' => $url,
            'method' => 'GET',
            'headers' => [
                'Content-Type:application/json',
                'X-KM-Api-Version:v-0.2.0'
            ]
        ];

        $ch = curl_init($proxyUrl);
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, 1);
        curl_setopt($ch, CURLOPT_POST, true);
        curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($bridgePayload));
        curl_setopt($ch, CURLOPT_HTTPHEADER, ['Content-Type:application/json']);
        curl_setopt($ch, CURLOPT_SSL_VERIFYPEER, false);

        $response = curl_exec($ch);
        if (curl_errno($ch)) {
            $error_msg = curl_error($ch);
            throw new NagadPaymentException($error_msg);
        }

        curl_close($ch);
        return $response;
    }
```

---

## Final Setup
1.  **Configure Admin Panel:** Go to your plugin settings page in WordPress.
2.  **Input Details:** Enter your **Proxy Bridge URL** (e.g., `https://your-bd-domain.com/nagad-bridge.php`) and **Proxy Secret Key**.
3.  **Save:** Click "Save Settings".

## Security Checklist
1.  **Secret Key:** Ensure `YOUR_STRONG_SECRET_KEY_HERE` is a long, random string. 
2.  **HTTPS:** Ensure your Bangladeshi domain supports HTTPS.
3.  **Logs:** Disable debug logging on the BD script in production.
