# Nagad API Proxy Bridge Setup

## Overview
This solution enables a PHP application hosted on a foreign server (e.g., Singapore VPS) to communicate with the Nagad Payment Gateway by routing traffic through a proxy script hosted on a Bangladeshi server. This is necessary because Nagad requires requests to originate from a whitelisted Bangladeshi IP address.

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

// SECURITY: Replace this with a strong, unique random string
define('PROXY_SECRET', 'YOUR_STRONG_SECRET_KEY_HERE');

// 1. Receive Input
$input = file_get_contents('php://input');
$data = json_decode($input, true);

// 2. Authenticate Request
if (!isset($data['secret']) || $data['secret'] !== PROXY_SECRET) {
    http_response_code(403);
    header('Content-Type: application/json');
    echo json_encode(['status' => 'error', 'message' => 'Unauthorized Proxy Access']);
    exit;
}

// 3. Initialize cURL
$ch = curl_init();
$target_url = $data['target_url'];
$method = $data['method']; // 'POST' or 'GET'
$headers = isset($data['headers']) ? $data['headers'] : [];
$payload = isset($data['body']) ? $data['body'] : null;

// 4. Configure cURL
curl_setopt($ch, CURLOPT_URL, $target_url);
curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
curl_setopt($ch, CURLOPT_FOLLOWLOCATION, true);

// SSL Verification (Disable if necessary for shared hosting, otherwise keep true)
curl_setopt($ch, CURLOPT_SSL_VERIFYHOST, 0); 
curl_setopt($ch, CURLOPT_SSL_VERIFYPEER, 0);

// Set Forwarded Headers
curl_setopt($ch, CURLOPT_HTTPHEADER, $headers);

// Handle POST/GET Methods
if ($method === 'POST') {
    curl_setopt($ch, CURLOPT_POST, true);
    // If payload is array, encode it. If string (already encoded), pass as is.
    $post_fields = is_array($payload) ? json_encode($payload) : $payload;
    curl_setopt($ch, CURLOPT_POSTFIELDS, $post_fields);
}

// 5. Execute & Return
$response = curl_exec($ch);
$error = curl_error($ch);
curl_close($ch);

if ($error) {
    header('Content-Type: application/json');
    echo json_encode(['proxy_error' => $error]);
} else {
    // Pass raw Nagad response back to VPS
    echo $response;
}
?>
```

---

## Step 2: Client-Side Integration (Main VPS)

You need to modify the `Helper.php` file in the Nagad library to route requests through the bridge instead of sending them directly.

**Target File Path:** `vendor-nagad-merchant/xenon/nagad-api/src/Helper.php`

### Configuration
At the top of your modified functions, define your bridge configuration:
```php
$proxyUrl = '[https://your-bd-domain.com/nagad-bridge.php](https://your-bd-domain.com/nagad-bridge.php)'; 
$proxySecret = 'YOUR_STRONG_SECRET_KEY_HERE'; // Must match the key in nagad-bridge.php
```

### Modification 1: `httpPostMethod`

Replace the existing `httpPostMethod` function with:

```php
/**
 * Routes POST requests through the BD Proxy Bridge
 */
public function httpPostMethod(string $postUrl, array $postData)
{
    // BRIDGE CONFIG
    $proxyUrl = '[https://your-bd-domain.com/nagad-bridge.php](https://your-bd-domain.com/nagad-bridge.php)'; 
    $proxySecret = 'YOUR_STRONG_SECRET_KEY_HERE';

    // Headers for Nagad
    $header = array(
        'Content-Type:application/json',
        'X-KM-Api-Version:v-0.2.0',
        'X-KM-IP-V4:' . $this->getClientIP(), // Customer's IP
        'X-KM-Client-Type:PC_WEB'
    );

    // Payload wrapper for Bridge
    $bridgePayload = [
        'secret' => $proxySecret,
        'target_url' => $postUrl,
        'method' => 'POST',
        'headers' => $header,
        'body' => $postData
    ];

    // Send to Bridge
    $ch = curl_init($proxyUrl);
    curl_setopt($ch, CURLOPT_CUSTOMREQUEST, "POST");
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($bridgePayload));
    curl_setopt($ch, CURLOPT_HTTPHEADER, ['Content-Type:application/json']);
    curl_setopt($ch, CURLOPT_FOLLOWLOCATION, 1);
    
    // Adjust SSL settings based on your VPS config
    curl_setopt($ch, CURLOPT_SSL_VERIFYHOST, 0);
    curl_setopt($ch, CURLOPT_SSL_VERIFYPEER, 0);
    
    $resultData = curl_exec($ch);
    $curl_error = curl_error($ch);

    if (!empty($curl_error)) {
        return ['error' => 'Bridge Connection Error: ' . $curl_error];
    }

    $response = json_decode($resultData, true);
    curl_close($ch);
    return $response;
}
```

### Modification 2: `httpGet`

Replace the existing `httpGet` function with:

```php
/**
 * Routes GET requests (Verification) through the BD Proxy Bridge
 */
public static function httpGet($url)
{
    // BRIDGE CONFIG
    $proxyUrl = '[https://your-bd-domain.com/nagad-bridge.php](https://your-bd-domain.com/nagad-bridge.php)'; 
    $proxySecret = 'YOUR_STRONG_SECRET_KEY_HERE';
    
    // Payload wrapper for Bridge
    $bridgePayload = [
        'secret' => $proxySecret,
        'target_url' => $url,
        'method' => 'GET',
        'headers' => [
            'Content-Type:application/json',
            'X-KM-Api-Version:v-0.2.0'
        ]
    ];

    // Send to Bridge (Always POST to bridge to carry payload)
    $ch = curl_init($proxyUrl);
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, 1);
    curl_setopt($ch, CURLOPT_POST, true);
    curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($bridgePayload));
    curl_setopt($ch, CURLOPT_HTTPHEADER, ['Content-Type:application/json']);
    curl_setopt($ch, CURLOPT_SSL_VERIFYPEER, false);

    $response = curl_exec($ch);
    
    if (curl_errno($ch)) {
        $error_msg = curl_error($ch);
        throw new NagadPaymentException('Bridge Verification Error: ' . $error_msg);
    }

    curl_close($ch);
    return $response;
}
```

---

## Security Checklist

1.  **Secret Key:** Ensure `YOUR_STRONG_SECRET_KEY_HERE` is a long, random string. Do not use "123456". If this key leaks, anyone can use your bridge to proxy traffic.
2.  **HTTPS:** Ensure your Bangladeshi domain supports HTTPS (`https://your-bd-domain.com`) to encrypt the data (including the secret key) in transit between your VPS and the BD host.
3.  **Logs:** If you enable logging on the BD script for debugging, ensure you disable it in production to prevent leaking sensitive Nagad payment data.
