This page covers some of the more advanced options, not available in versions 0.6 and 0.7-pre1.<br>

<h1>Authentication options</h1>
In order to provide proper support for advanced authentication, the SSL authentication logic has been removed from the generated code, and delegated to a user-defined delegate. The delegate will have to implement the <b><a href='http://code.google.com/p/wsdl2objc/source/browse/trunk/Templates/WSDL2ObjC%20Standard%20Additions/USAdditions_H.template'>SSLCredentialsManaging</a></b> protocol, defined in the generated sources.<br>
<br>
<h2>Basic Authentication</h2>
The generated code, provides a simple <a href='http://code.google.com/p/wsdl2objc/source/browse/trunk/Templates/WSDL2ObjC%20Standard%20Additions/USAdditions_H.template'>implementation</a> of the protocol, which can be used for basic SSL authentication.<br>
<br>
To use it, you just need to instantiate the delegate with an username and password, and set the <b>sslManager</b> property of the binding:<br>
<pre><code>...<br>
BasicSSLCredentialsManager *manager = <br>
    [BasicSSLCredentialsManager managerWithUsername:u.text andPassword:p.text];<br>
       <br>
binding.sslManager = manager;        <br>
...<br>
</code></pre>

You can check the implementation of the <b>BasicSSLCredentialsManager</b>  <a href='http://code.google.com/p/wsdl2objc/source/browse/trunk/Templates/WSDL2ObjC%20Standard%20Additions/USAdditions_M.template'>here</a>.<br>
<br>
<h2>Self-signed server certificate</h2>
If your application connects to a server which has a self-signed certificate, then simply letting the system to validate the server will not do the job, as the received certificate would come from an unknown certificate authority.<br>
In this case, you need to tell the system which certificate authorities you are expecting, and handle the validation process manually. In order to do this you need a couple of things:<br>
<ul><li>The root certificate that have signed the server side certificate. <br> One way to get this in your code is by making it part of your application binary, and then generating a NSData instance out of it:<br>
<pre><code>- (NSData *)dataCA<br>
{<br>
    uint8_t bytes[] = {CA_BYTES};<br>
    return [NSData dataWithBytes:bytes length:CA_LENGTH];<br>
}<br>
</code></pre>
</li><li>A NSArray holding the references of all the root certificates you need to support (in this case only one):<br>
<pre><code>- (NSArray *) serverAnchors<br>
{<br>
    static NSArray *anchors = nil;<br>
    if (!anchors) {<br>
        NSData *caData = [self dataCA];<br>
        SecCertificateRef caRef = SecCertificateCreateWithData(kCFAllocatorDefault, (CFDataRef) caData);<br>
      <br>
        anchors = [[NSArray arrayWithObjects:(id)caRef,  nil] retain];<br>
        <br>
        if (caRef) {<br>
            CFRelease(caRef);   <br>
        }<br>
    }<br>
    <br>
    return anchors;<br>
}<br>
</code></pre></li></ul>

<ul><li>A proper <b>SSLCredentialsManaging</b> protocol implementation. <br> In the code below we give only the most important part of the implementation, which basically tell the system that we want to handle the authentication manually, and then validates the server certificate received:<br>
<pre><code>- (BOOL)canAuthenticateForAuthenticationMethod:(NSString *)authMethod<br>
{<br>
    return [authMethod isEqualToString:NSURLAuthenticationMethodServerTrust];<br>
}<br>
</code></pre>
<pre><code>- (BOOL)validateResult:(SecTrustResultType)res<br>
{<br>
    if ((res == kSecTrustResultProceed)                 // trusted certificate<br>
        || ((res == kSecTrustResultConfirm)             // valid but user should be asked for confirmation<br>
            || (res == kSecTrustResultUnspecified)      // valid but user have not specified wheter to trust<br>
            || (res == kSecTrustResultDeny)             // valid but user does not trusts this certificate<br>
            )<br>
        )<br>
    {<br>
        return YES;<br>
    } <br>
    <br>
    return NO;<br>
}<br>
<br>
- (BOOL)authenticateForChallenge:(NSURLAuthenticationChallenge *)challenge<br>
{<br>
    if ([challenge previousFailureCount] &gt; 0) {        <br>
        return NO;<br>
    }<br>
    <br>
    NSURLCredential *newCredential = nil;<br>
    NSURLProtectionSpace *protectionSpace = [challenge protectionSpace];<br>
    SecurityCenter *securityCenter = [SecurityCenter sharedInstance];<br>
    <br>
    // server authentication - NSURLAuthenticationMethodServerTrust<br>
    if ([protectionSpace.authenticationMethod isEqualToString:NSURLAuthenticationMethodServerTrust]) {<br>
        SecTrustRef trust = [protectionSpace serverTrust];<br>
        NSArray *anchors = [securityCenter serverAnchors];<br>
        SecTrustSetAnchorCertificates(trust, (CFArrayRef)anchors);<br>
        SecTrustSetAnchorCertificatesOnly(trust, YES);<br>
        <br>
        <br>
        SecTrustResultType res = kSecTrustResultInvalid;<br>
        OSStatus sanityChesk = SecTrustEvaluate(trust, &amp;res);<br>
               <br>
        if ((sanityChesk == noErr) <br>
            &amp;&amp; [self validateResult:res]) {<br>
            <br>
            newCredential = [NSURLCredential credentialForTrust:trust];<br>
            [[challenge sender] useCredential:newCredential forAuthenticationChallenge:challenge];<br>
            <br>
            return YES;<br>
        }<br>
        <br>
        return NO;<br>
    }<br>
    <br>
    [NSException raise:@"Authentication method not supported" format:@"%@ not supported.", [protectionSpace authenticationMethod]];<br>
    return NO;<br>
}<br>
</code></pre></li></ul>

<h2>Client-side certificate authentication</h2>
Some services (e.g. eBanking) are intended to be consumed only by a defined set of client applications. In this case you might need to use 2-way SSL, which asks the client application to authenticate with proper certificate. <br> To do this, you would need the following:<br>
<ul><li>NSData representation of a PKCS12 key store, holding the certificate and the private key for the client. <br> One way to do this is to integrate the key store as part of the application's byte code.<br>
<pre><code>- (NSData *)dataPKCS12<br>
{<br>
    uint8_t bytes[] = {PKCS12_BYTES};<br>
    return [NSData dataWithBytes:bytes length:PKCS12_LENGTH];<br>
}<br>
</code></pre>
</li><li>NSArray, in this case named <b>keystores</b>, which holds all the key/certificate entries in the key store, each represented by a single NSDictionary instance:<br>
<pre><code>- (void)importPKCS12<br>
{<br>
    if (!keystores) {<br>
        NSData *pkcsData = [self dataPKCS12];<br>
        NSString *password = [self passwordPKCS12];<br>
        <br>
        OSStatus sanityChesk = SecPKCS12Import((CFDataRef)pkcsData, <br>
                                               (CFDictionaryRef)[NSDictionary dictionaryWithObject:password <br>
                                                                                            forKey:(id)kSecImportExportPassphrase], <br>
                                               (CFArrayRef *)&amp;keystores);<br>
        CHECK_CONDITION1(sanityChesk == noErr, @"Error while importing pkcs12 [%d]", sanityChesk);<br>
        [keystores retain];<br>
    }<br>
}<br>
</code></pre>
</li><li>Logic to extract the <b>identity</b> and <b>certificates</b> for the client. <br> In this example the <b>clientKeyStore</b> variable points to one of the entries in the <b>keystores</b> variable mentioned above:<br>
<pre><code>- (SecIdentityRef)clientIdentity<br>
{<br>
    return (SecIdentityRef)[clientKeyStore objectForKey:(id)kSecImportItemIdentity];<br>
}<br>
<br>
- (NSArray *)clientCertificates<br>
{<br>
    return [clientKeyStore objectForKey:(id)kSecImportItemCertChain];<br>
}<br>
</code></pre>
</li><li>A proper <b>SSLCredentialsManaging</b> protocol implementation. <br> In the code below we give only the most important part of the implementation, which basically tell the system that we want to handle the authentication manually, and then authenticates the client with a certificate:<br>
<pre><code>- (BOOL)canAuthenticateForAuthenticationMethod:(NSString *)authMethod<br>
{<br>
    return [authMethod isEqualToString:NSURLAuthenticationMethodClientCertificate];<br>
}<br>
</code></pre>
<pre><code>- (BOOL)authenticateForChallenge:(NSURLAuthenticationChallenge *)challenge<br>
{<br>
    if ([challenge previousFailureCount] &gt; 0) {        <br>
        return NO;<br>
    }<br>
    <br>
    NSURLCredential *newCredential = nil;<br>
    NSURLProtectionSpace *protectionSpace = [challenge protectionSpace];<br>
    SecurityCenter *securityCenter = [SecurityCenter sharedInstance];<br>
    <br>
    // client authentication - NSURLAuthenticationMethodClientCertificate<br>
    if ([protectionSpace.authenticationMethod isEqualToString:NSURLAuthenticationMethodClientCertificate]) {<br>
       <br>
        NSArray *acceptedIssuers = [protectionSpace distinguishedNames];<br>
        [securityCenter setIssuerDistinguishedNames:acceptedIssuers];<br>
        <br>
        SecIdentityRef identity = (SecIdentityRef)[securityCenter clientIdentity];<br>
        NSArray *certs = [securityCenter clientCertificates];<br>
        <br>
        if (identity &amp;&amp; certs) {<br>
            newCredential = [NSURLCredential credentialWithIdentity:identity<br>
                                                       certificates:certs<br>
                                                        persistence:NSURLCredentialPersistenceNone];<br>
            [[challenge sender] useCredential:newCredential forAuthenticationChallenge:challenge];<br>
            <br>
            return YES;<br>
        }<br>
        <br>
        return NO;<br>
    }<br>
    <br>
    [NSException raise:@"Authentication method not supported" format:@"%@ not supported.", [protectionSpace authenticationMethod]];<br>
    return NO;<br>
}<br>
</code></pre>
<hr />
<h3>NOTE</h3>
Pay some extra attention on how you will make the CAs and the PKCS12 key store available to the application. E.g. A simple hex dump of the application may reveal this data to a potential attacker. Make sure you always use some sort of encryption, or at least byte-masking before integrating the data to your binary.<br>
<hr /></li></ul>

<h1>SOAPSecurity</h1>
Let me say this short and clear: SOAPSecurity for iPhone is <b>EXTREMELY</b> difficult to implement.<br>
That said, I will only give an example of what the code generated by wsdl2objc offers. For anything else you are on your own :)<br>
<br>
<h2>XMLSigning</h2>
One of the features covered by the SOAPSecurity extension, is the XMLSigning standard, which can be used to sign portions of the XML request, in order to ensure it is received unmodified by the server. To make things more flexible, the generated code uses a delegate, which if set is called to sign the generated XML regrets before sending it to the server.<br>
<br>
Out of the box, the generated code comes with a <b><a href='http://code.google.com/p/wsdl2objc/source/browse/trunk/Templates/WSDL2ObjC%20Standard%20Additions/USAdditions_M.template'>SOAPSigner</a></b> implementation which signs the the whole XML body, using the RSA-SHA1 algorithm.<br>
<br>
To use this feature you need to:<br>
<ul><li>Implement the SOAPSignerDelegate protocol (lets say in a class called SimpleSOAPSignerDelegate), and set an instance of this delegate to the SOAPSigner instance you would use:<br>
<pre><code>...<br>
soapSigner = [[SOAPSigner alloc] initWithDelegate:[SimpleSOAPSignerDelegate sharedInstance]];<br>
...<br>
</code></pre>
</li><li>And set the instantiated SOAPSigner as a delegate to the binding:<br>
<pre><code>...<br>
id binding = [self binding];<br>
[binding setSoapSigner:soapSigner];<br>
...<br>
</code></pre></li></ul>

The <b>SimpleSOAPSignerDelegate</b> should use the iPhone SDK's <b>Security</b> framework to sign the data:<br>
<pre><code>- (NSData *)signData:(NSData *)input key:(SecKeyRef)key<br>
{<br>
    CHECK_CONDITION([input length], @"The input length must not 0");<br>
    CHECK_CONDITION(key, @"The private key must not be NULL");<br>
    <br>
    OSStatus sanityCheck = noErr;<br>
	NSData * signedHash = nil;<br>
	<br>
	uint8_t * signature = NULL;<br>
	size_t signatureSize = SecKeyGetBlockSize(key);<br>
	<br>
	// Malloc a buffer to hold signature.<br>
	signature = malloc( signatureSize );<br>
	memset((void *)signature, 0x0, signatureSize);<br>
	<br>
	// Sign the SHA1 hash.<br>
    NSData *checksum = [self checksumSHA1:[input bytes] length:[input length]];<br>
	sanityCheck = SecKeyRawSign(key, <br>
                                kSecPaddingPKCS1SHA1, <br>
                                (const uint8_t *)[checksum bytes], <br>
                                CC_SHA1_DIGEST_LENGTH, <br>
                                signature, <br>
                                &amp;signatureSize);<br>
	<br>
	CHECK_CONDITION1(sanityCheck == noErr, @"Problem signing the SHA1 hash, OSStatus == %d.", sanityCheck );<br>
	<br>
	// Build up signed SHA1 blob.<br>
	signedHash = [NSData dataWithBytes:(const void *)signature length:(NSUInteger)signatureSize];<br>
	if (signature) free(signature);<br>
	<br>
	return signedHash;   <br>
}<br>
</code></pre>

What is happening behind the scene is:<br>
<ul><li>The proper security header would be added to the XML. <br> This header specifies all the algorithms used (e.g. for signing and xml canonicalization), and contains the actual signature at the end.<br>
</li><li>The XML is transformed into a canonical form. <br> This is needed as the XML doesn't mind if it has some extra space or a new line between the tags. However all of this changes the byte representation of the XML and of course  the final signature. So the canonicalization transforms the XML into a standard form which will ensure the same byte representation on both client and server side.<br>
</li><li>The bytes of the transformed XML are given for signing.<br>
</li><li>And finally the signature is included into the XML header.<br>
<hr />
<h3>NOTE</h3>
Third party C libraries exist that can do this with small or no effort by you.<br>
However the Apple's terms and conditions forbid any third-party encryption methods to be used (at least without some special permission).<br>
<hr />