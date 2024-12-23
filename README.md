# CAPTCHA
Simple, client-side, CAPTCHA

This is a small snippet of JavaScript which generates a CAPTCHA image in a \<canvas\> HTML element, and uses the Crypto API, which is now widely available in most modern browsers. 

This is intended to be a minor speed-bump against some bots.

![captcha-form](https://raw.githubusercontent.com/cypnk/CAPTCHA/refs/heads/main/captcha-form.png)

The script requires a server-side nonce to be sent to the client via a hidden input field. Another random nonce is generated and stored in a second hidden input field.
```
<input type="hidden" id="nonce" name="nonce" value="server-side-nonce"> <!-- Sent by the server -->
<input type="hidden" id="cnonce" name="cnonce" value=""> <!-- Created by the script -->
```

Both the server-side and client-side nonces are hashed together to create a third random string, which shortened to 8 characters for the CAPTCHA.

To verify the CAPTCHA, take the sent nonce and check it against the value stored in the visitor's session. And then combine the nonce and cnonce and use SHA-256 hash. Then take the first 8 characters to match the CAPTCHA input value after base64 encoding the output and removing some characters which are easily confused with each other from the result. 
```
// Example in PHP. Remember to sanitize in production.
function verifyCaptcha() {
	if ( session_status() === PHP_SESSION_NONE ) {
		session_start();
	}
	
	$nonce		= $_POST['nonce'] ?? '';
	$snonce		= $_SESSION['nonce'] ?? '';
	
	$cnonce		= $_POST['cnonce'] ?? '';
	$captcha	= $_POST['captcha'] ?? '';
	
	if ( empty( $nonce ) || empty( $snonce )  || empty( $cnonce ) || empty( $captcha ) ) {
		return false;
	}

	// Match session stored nonce with form nonce
	if ( 0 != strcmp( $nonce, $snonce ) ) {
		return false;
	}

	// Check sent string sizes
	if ( 8 != strlen( $captcha ) || 64 != strlen( $cnonce ) ) {
		return false;
	}
	
	$chk		= hash( 'sha256', $nonce . $cnonce );
	$chk		= preg_replace( '/[0oO1i=\/]/', '', base64_encode( $chk ) );
	
	if ( 0 == strcasecmp( substr( $chk, 0, 8 ), $captcha ) ) {
		return true;
	}
	
	return false; // Default to fail
}
```
