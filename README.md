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
The HTML must contain a ```<noscript>``` element which has the attributes necessary to render the CAPTCHA image.

E.G.
```
<noscript data-captcha="true" 
	data-width="150" 
	data-height="45"
	data-size="8">Code generation requires JavaScript</noscript>
```

```data-captcha``` Tells the script the \<canvas\> element will be generated here. ```data-width``` And ```data-height``` specify the width and height of the CAPTCHA image, while ```data-size``` is the number of characters to generate.

Both the server-side and client-side nonces are hashed together to create a third random string, which is base64 encoded and shortened to 8 characters, with some which are easily confused with each other removed, for the CAPTCHA.

To verify the CAPTCHA, take the sent nonce and check it against the value stored in the visitor's session. Then combine the nonce and cnonce, in that order, and apply a SHA-256 hash. Finally, after base64 encoding the output and removing the same easily confused characters as in the JavaScript from the result, take the first 8 characters to match the CAPTCHA input value. 

Example usage in PHP Remember to sanitize in production.
```
// Get your $snonce from session data
function validateCaptcha( $snonce ) {
	$nonce		= $_POST['nonce'] ?? '';
	$cnonce		= $_POST['cnonce'] ?? '';
	$captcha	= $_POST['captcha'] ?? '';
	
	if ( empty( $nonce ) || empty( $cnonce ) || empty( $captcha ) ) {
		return false;
	}
	
	// Match session stored nonce with form nonce
	if ( 0 != strcmp( $nonce, $snonce ) ) {
		return false;
	}
	
	// Check sent string sizes against constants
	if  ( 
		CAPTCHA_SIZE	!= strlen( $captcha )	|| 
		NONCE_SIZE	!= strlen( $cnonce ) 
	) {
		return 0;
	}
	
	// Create a hash with nonce and cnonce and widen character set
	$chk		= hash( 'sha256', $nonce . $cnonce );
	
	// Remove confusing characters (must match client-side code)
	$chk		= preg_replace( '/[0oO1liNzZ2m3=\/]/', '', base64_encode( $chk ) );
	
	// Limit to CAPTCHA length (must match client-side code)
	if ( 0 == strcasecmp( substr( $chk, 0, CAPTCHA_SIZE ), $captcha ) ) {
		return true;
	}
	
	return false; // Default to fail
}
```
Example usage in Perl. ```formData``` Is a placeholder helper. Use your own user input filter.

```
sub validateCaptcha {
	my ( $snonce )	= @_;
	
	my %data	= formData();
	if ( 
		!defined( $data->{nonce} )	|| 
		!defined( $data->{cnonce} )	|| 
		!defined( $data->{captcha} )
	) {
		return 0;
	}
	
	my ( $nonce, $cnonce, $captcha ) = ( 
		$data->{nonce}, $data->{cnonce}, $data->{captcha} 
	);
	
	// Match session stored nonce with form nonce
	if ( $snonce ne $nonce ) {
		return 0;
	}
	
	# Check sent string sizes against constants
	if  ( 
		CAPTCHA_SIZE	!= length( $captcha )	|| 
		NONCE_SIZE	!= length( $cnonce ) 
	) {
		return 0;
	}
	
	# Create a hash with nonce and cnonce and widen character set
	my $chk	= encode_base64( sha256_hex( $nonce . $cnonce ), '' );
	
	# Remove confusing characters (must match client-side code)
	$chk	=~ s/[0oO1liNzZ2m3=\/]//g;
	
	# Limit to CAPTCHA length (must match client-side code)
	if ( lc( substr( $chk, 0, CAPTCHA_SIZE ) ) eq lc( $captcha ) ) {
		return 1;
	}
	
	# Default to fail
	return 0;
}
```
