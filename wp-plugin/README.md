# Blue Billywig WordPress Plugin v1.0.15 — Code Evaluation

---

## Critical Security Vulnerabilities

### 1. Unauthenticated AJAX Endpoint Leaks Configuration
**File:** `admin/includes/class-ajax.php` — Lines 36–56

The `wp_ajax_nopriv_get_custom_playout_screen` hook exposes publication/account data to any anonymous visitor with no nonce or capability check. Any unauthenticated user can call `wp-admin/admin-ajax.php?action=get_custom_playout_screen` and retrieve account configuration.

```php
add_action('wp_ajax_nopriv_get_custom_playout_screen', array( $this, 'get_custom_playout_screen' ));

public function get_custom_playout_screen() {
    $playout_screen = get_option('blue-billywig-playout');
    $publication = get_option('blue-billywig-publication');
    echo wp_send_json(array('customPlayoutScreen' => $playout_screen, 'publication' => $publication));
}
```

**Fix:** Remove the `nopriv` hook or add a nonce and capability check.

---

### 2. All REST API Endpoints Are Publicly Accessible
**File:** `admin/includes/class-admin.php` — Lines 97–177

Four REST routes use `'permission_callback' => '__return_true'`, meaning anyone on the internet can search your media library, enumerate playlists, and retrieve configuration data without authentication.

Affected endpoints:
- `GET /wp-json/bluebillywig/v1/get_custom_playout_screen`
- `GET /wp-json/bluebillywig/v1/search_media_clips?search=...`
- `GET /wp-json/bluebillywig/v1/get_playout_list`
- `GET /wp-json/bluebillywig/v1/get_publication_data`

**Fix:** Replace `'permission_callback' => '__return_true'` with `current_user_can('manage_options')` or at minimum `is_user_logged_in`.

---

### 3. API Injection via Unsanitized Input
**File:** `includes/class-filters.php` — Lines 186–192, 208–218

User-supplied POST data (`$_POST['quary']['title']`, `$_POST['quary']['createddate']`) is interpolated directly into raw JSON filter strings and Solr-style URL query params sent to the BB API. `sanitize_text_field()` does not prevent JSON or query injection.

```php
$media_clip_title = isset($media_clip_title_data) ? '{"filters":[{"type":"MediaClipList","field":"title","operator":"contains","value":["' . $media_clip_title_data . '"]}]}' : '';
$media_clip_date  = isset($media_clip_date_data) ? 'createddate:' . $media_clip_date_data : '';
```

**Fix:** Encode/validate filter values before interpolating into JSON strings and URL paths. Use `json_encode()` for JSON construction.

---

### 4. Arbitrary File Deletion
**File:** `admin/includes/class-ajax.php` — Lines 73–87

After nonce validation, any authenticated admin can pass any server path (e.g. `/wp-config.php`) to `wp_delete_file()`. There is no restriction to the uploads directory.

```php
public function remove_uploaded_file() {
    if (!check_admin_referer('blue-billywig-upload-video', 'blue-billywig-nonce')) { ... }
    if (isset($_POST['filePath'])) {
        $file_path = sanitize_text_field(wp_unslash($_POST['filePath']));
        wp_delete_file($file_path);
    }
}
```

**Fix:** Validate `$file_path` is within `wp_upload_dir()['basedir']` before calling `wp_delete_file()`.

---

## High Severity

### 5. Stored XSS from Unescaped API Data
**File:** `admin/includes/class-ajax.php` — Lines 317–491

Video/playlist titles, descriptions, tags, and image URLs from the BB API are injected into HTML with no `esc_html()` or `esc_attr()` calls. If the BB account is compromised or a malicious title is crafted, it executes in the admin browser.

```php
$video_data .= '<div class="bb-video-title">' . $bb_media_clip_title . '</div>
    <div class="bb-video-description">' . $bb_media_clip_desc . '</div>
    <div class="bb-video-tags">' . $bb_media_clip_cat . '</div>';
```

**Fix:** Wrap all API-sourced values in `esc_html()` for content and `esc_attr()` for HTML attribute values.

---

### 6. Missing Capability Checks on All AJAX Handlers
**File:** `admin/includes/class-ajax.php` — All handlers

Every AJAX action only requires being logged in. No `current_user_can()` check guards destructive operations. Any subscriber or contributor who obtains a valid nonce can:

- Enumerate all videos and playlists
- Delete videos from the BB account
- Modify video metadata
- Upload files to the WordPress server
- Delete files from the WordPress server

**Fix:** Add `current_user_can('manage_options')` (or appropriate capability) at the top of each handler.

---

### 7. Settings Form Has No Capability Check
**File:** `admin/includes/class-ajax.php` — Lines 365–389

Any logged-in user can overwrite the API secret, publication slug, playout, and embed settings. The handler verifies the nonce but does not call `current_user_can('manage_options')`.

**Fix:** Add `current_user_can('manage_options')` check before processing the settings form.

---

## Medium Severity

### 8. `ajaxurl` Global Is Mutated on Each Call — URL Accumulation Bug
**File:** `admin/assets/js/bb_main.js` — Lines 519, 586

The global `ajaxurl` variable is mutated with `+=` on every invocation, appending another `?noCache=...` query string each time. After the first call, all subsequent AJAX requests use broken URLs. The `cache: false` jQuery option is already set and makes this redundant.

```javascript
ajaxurl += "?noCache=" + (new Date().getTime()) + Math.random();
```

**Fix:** Use a local copy: `let url = ajaxurl + "?noCache=" + ...`

---

### 9. DOM XSS: Server HTML Injected via `.html()`
**File:** `admin/assets/js/bb_main.js` — Lines 554, 614

Server AJAX responses are injected into the DOM via jQuery's `.html()`, which executes any `<script>` tags or event-handler attributes present. This compounds the server-side XSS issue (#5).

```javascript
$('#bb-videos .bb-videos').html(response.videoData)
```

**Fix:** Sanitize server responses or build DOM elements programmatically instead of using `.html()`.

---

### 10. `is_uploaded_file()` Check Is Commented Out
**File:** `admin/includes/class-ajax.php` — Lines 171–175

The check verifying that a file was genuinely uploaded via HTTP POST has been commented out, weakening defense against file injection attacks.

**Fix:** Restore the `is_uploaded_file()` validation.

---

### 11. `return_bytes_fn()` Returns Wrong Values
**File:** `admin/includes/class-helper.php` — Lines 18–34

All size suffixes multiply by 1024 only once. `256M` returns 262,144 bytes instead of 268,435,456. This causes the client-side file size limit to be set ~1000x too small.

```php
case 'g':
    $new_val *= 1024;  // Should be *= 1024 * 1024 * 1024
    break;
case 'm':
    $new_val *= 1024;  // Should be *= 1024 * 1024
    break;
```

**Fix:** Use correct multipliers: G = 1,073,741,824; M = 1,048,576.

---

### 12. Dead Modal Branch — `$_POST` Strict Comparison Bug
**File:** `admin/includes/class-ajax.php` — Lines 320, 437

`$_POST` values are always strings. `true === $_POST['modal']` is never true, so the modal code path is never executed.

**Fix:** Use `filter_var($_POST['modal'], FILTER_VALIDATE_BOOLEAN)` or compare against the string `'true'`.

---

### 13. Array Variable Overwritten With String — Playout Dropdown Always Empty
**File:** `admin/includes/class-ajax.php` — Line 460

`$inner_value` (a loop array) is concatenated with `.=` to a string, casting the array to `"Array"` and silently discarding the option HTML. The playout dropdown is always empty regardless of API response.

**Fix:** Use a separate accumulator variable for the option HTML string.

---

### 14. SSL Verification Disabled in Legacy RPC Library
**File:** `vendor/bluebillywig/vmsrpc/src/RPC.php` — Line 354

```php
curl_setopt($this->curl_handle, CURLOPT_SSL_VERIFYPEER, false);
```

All outbound cURL requests through this library are vulnerable to man-in-the-middle attacks. This library is not used in current code paths but ships with the plugin.

---

## Low Severity / Code Quality

### 15. Extensive Debug `error_log()` Calls Left in Production
`admin/includes/class-ajax.php:131–162` — Verbose logging including `print_r($_FILES['file'], true)` is committed to production code, writing file paths to the server error log on every upload.

### 16. `console.log` Debug Statements in Production JavaScript
`admin/assets/js/bb_main.js:165,217,333` — Includes `console.log(globalUploadData)` which logs the full FormData including nonce on every upload, and a comment containing a developer's name.

### 17. Broken PHP Interpolation in `modal.php`
`admin/partials/modal.php:19` — The href renders as the literal string `https://' . $publication . '.bbvms.com/...` because PHP string concatenation syntax was used inside a raw HTML block.

### 18. N+1 API Calls in Video Loop
`admin/includes/class-ajax.php:447–448` — `get_playout_list` makes a remote API call inside a `foreach` loop over videos. 15 videos = 15 identical API calls.

**Fix:** Hoist the `get_playout_list` call outside the loop.

### 19. Double API Call on Every Library Load
`admin/includes/class-ajax.php:421–422` — Two API calls are made per page load: one paginated for display, one unpaginated solely to count total items. The count is likely available in the paginated response.

### 20. Orphaned Upload Files
If the user closes the browser after the file is uploaded to WordPress but before it is pushed to Blue Billywig, the local file is never deleted. There is no server-side cleanup mechanism.

### 21. Incomplete Uninstall Cleanup
`uninstall.php` — The `blue-billywig-playout-video-status` option created per post is not removed on uninstallation.

### 22. MD5 Password Hashing in Legacy RPC Library
`vendor/bluebillywig/vmsrpc/src/RPC.php:248–252` — Double-MD5 with a nonce is used for authentication. MD5 is cryptographically broken. Not exercised by current code but present in the shipped bundle.

### 23. API Credentials Stored as Block Attributes
`admin/assets/js/block.js:34–57` — The publication slug is stored in Gutenberg block attributes saved to post content, distributing it widely through the database.

### 24. `settings_submit_form` Sends No Response to JS Caller
`admin/includes/class-ajax.php:365–389` — The function calls `wp_die()` without sending a JSON response. The JavaScript always shows "Data updated" regardless of success or failure.

---

## Summary Table

| # | Severity | Issue | File | Lines |
|---|----------|-------|------|-------|
| 1 | Critical | Unauth AJAX exposes config | class-ajax.php | 36–56 |
| 2 | Critical | Unauth REST endpoints | class-admin.php | 97–177 |
| 3 | Critical | API injection via POST data | class-filters.php | 186–218 |
| 4 | Critical | Arbitrary file deletion | class-ajax.php | 73–87 |
| 5 | High | Stored/reflected XSS | class-ajax.php | 317–491 |
| 6 | High | Missing capability checks | class-ajax.php | All handlers |
| 7 | High | Settings form privilege escalation | class-ajax.php | 365–389 |
| 8 | Medium | ajaxurl mutation bug | bb_main.js | 519, 586 |
| 9 | Medium | DOM XSS via .html() | bb_main.js | 554, 614 |
| 10 | Medium | is_uploaded_file() commented out | class-ajax.php | 171–175 |
| 11 | Medium | Broken byte conversion | class-helper.php | 18–34 |
| 12 | Medium | Dead modal branch (type bug) | class-ajax.php | 320, 437 |
| 13 | Medium | Array/string concat bug | class-ajax.php | 460 |
| 14 | Medium | SSL verification disabled | vmsrpc/RPC.php | 354 |
| 15 | Low | Debug error_log in production | class-ajax.php | 131–162 |
| 16 | Low | console.log in production JS | bb_main.js | 165, 217, 333 |
| 17 | Low | Broken PHP interpolation | modal.php | 19 |
| 18 | Low | N+1 API calls in loop | class-ajax.php | 447–448 |
| 19 | Low | Double API call per page | class-ajax.php | 421–422 |
| 20 | Low | Orphaned upload files | bb_main.js / AJAX | — |
| 21 | Low | Incomplete uninstall | uninstall.php | — |
| 22 | Low | MD5 password hashing | vmsrpc/RPC.php | 248–252 |
| 23 | Low | Credentials in block attributes | block.js | 34–57 |
| 24 | Low | AJAX handler sends no response | class-ajax.php | 365–389 |

---

## Top Recommendations

1. Add `current_user_can('manage_options')` to **every** AJAX handler and REST route
2. Restrict REST routes to at minimum `'permission_callback' => 'is_user_logged_in'`
3. Escape all API-sourced data with `esc_html()` / `esc_attr()` before HTML output
4. Validate file paths in `remove_uploaded_file()` against `wp_upload_dir()['basedir']`
5. Fix `ajaxurl` mutation in `bb_main.js` — use a local variable for cache-busting
6. Fix `return_bytes_fn()` multipliers: G = ×1,073,741,824 | M = ×1,048,576
7. Hoist `get_playout_list` API call outside the video foreach loop
8. Restore the `is_uploaded_file()` check
9. Use `json_encode()` for constructing JSON filter strings
10. Remove all `error_log()` and `console.log()` debug statements before release
