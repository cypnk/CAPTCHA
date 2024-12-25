# CAPTCHA
Simple, client-side, CAPTCHA generator

This is a small snippet of JavaScript which generates a [CAPTCHA](https://en.wikipedia.org/wiki/CAPTCHA) image in a \<canvas\> HTML element using the Crypto API, which is now widely available in most modern browsers. 

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

Order of operations:
* SHA-256 hash nonce + cnonce 
* Base64 encode result
* Remove potentially confusing/invalid characters: 0, o, O, 1, l, i, N, z, Z, 2, m, 3, =, /
* Shorten to 8 characters (default length, can be increased or decreased)

To verify the CAPTCHA, take the sent nonce and check it against the value stored in the visitor's session first. Then follow the same order of operation using your server-side language of choice. Abbreviated PHP and Perl examples are provided here. In both examples, $snonce is the server-generated, session-stored value. Get $nonce, $cnonce, and $captcha from user form data.

Example usage in PHP. Remember to sanitize in production:
```
# Constants in use
define( 'CAPTCHA_SIZE',	8 );	// Must match client-side JavaScript
define( 'NONCE_SIZE',	64 );	// Default SHA-256 output size

// Get your $snonce from session data
function validateCaptcha( $snonce, $nonce, $cnonce, $captcha ) {
	
	// Anything missing? Fail early
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
		return false;
	}
	
	// Create a hash with nonce and cnonce and widen character set
	$chk		= base64_encode( hash( 'sha256', $nonce . $cnonce ) );
	
	// Remove confusing characters (must match client-side code)
	$chk		= preg_replace( '/[0oO1liNzZ2m3=\/]/', '', $chk );
	
	// Limit to CAPTCHA length (must match client-side code)
	if ( 0 == strcasecmp( substr( $chk, 0, CAPTCHA_SIZE ), $captcha ) ) {
		return true;
	}
	
	return false; // Default to fail
}
```

Example usage in Perl:
```
# Required modules
use MIME::Base64;
use Digest::SHA qw( sha256_hex );

# Constants in use
use constant {
	CAPTCHA_SIZE		=> 8,	# Must match client-side JavaScript
	NONCE_SIZE		=> 64	# Default SHA-256 output size
};

sub validateCaptcha {
	my ( $snonce, $nonce, $cnonce, $captcha )	= @_;
	
	# Anything missing? Fail early
	if ( 
		( !defined $nonce || $nonce eq '' )	|| 
		( !defined $cnonce || $cnonce eq '' )	|| 
		( !defined $captcha || $captcha eq '' )	||
	) {
		return 0;
	}
	
	# Match session stored nonce with form nonce
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
