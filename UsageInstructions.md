# First steps #
## Generating code out of the WSDL file ##
Once you obtain WSDL2ObjC, code generation is pretty simple.

  1. Launch the app
  1. Browse to a WSDL file or enter in a URL
  1. Browse to an output directory
  1. Click "Parse WSDL"

Source code files will be added to the output directory you've specified.<br>
There will be one pair of .h/.m files for each namespace in your WSDL.<br>
<br>
<h2>Including the generated source files</h2>
You can add the output files to your project or create a web service framework from them. <br>
Each project that uses the generated web service code will need to link against <b>libxml2</b> by performing the following for each target in your XCode project:<br>
<br>
<ol><li>Get info on the target and go to the build tab<br>
</li><li>Add "-lxml2" to the Other Linker Flags property<br>
</li><li>Add "-I/usr/include/libxml2" to the Other C Flags property</li></ol>

If you are building an iPhone project also perform the following:<br>
<br>
<ol><li>Right click on the Frameworks folder in your project and select Add -> Existing Frameworks...<br>
</li><li>Select the <b>CFNetwork.framework</b> appropriate for your iPhone build</li></ol>

<h2>Using the generated code</h2>
You can use a given web service as follows:<br>
<pre><code>#import "MyWebService.h"<br>
<br>
MyWebServiceBinding *binding = [MyWebService MyWebServiceBinding];<br>
binding.logXMLInOut = YES;<br>
<br>
ns1_MyOperationRequest *request = [[ns1_MyOperationRequest new] autorelease];<br>
request.attribute = @"attributeValue";<br>
request.element = [[ns1_MyElement new] autorelease];<br>
request.element.value = @"elementValue"];<br>
<br>
MyWebServiceBindingResponse *response = [binding myOperationUsingParameters:request];<br>
<br>
NSArray *responseHeaders = response.headers;<br>
NSArray *responseBodyParts = response.bodyParts;<br>
<br>
for(id header in responseHeaders) {<br>
  if([header isKindOfClass:[ns2_MyHeaderResponse class]]) {<br>
    ns2_MyHeaderResponse *headerResponse = (ns2_MyHeaderResponse*)header;<br>
    <br>
    // ... Handle ns2_MyHeaderResponse ...<br>
  }<br>
}<br>
<br>
for(id bodyPart in responseBodyParts) {<br>
  if([bodyPart isKindOfClass:[ns2_MyBodyResponse class]]) {<br>
    ns2_MyBodyResponse *body = (ns2_MyBodyResponse*)bodyPart;<br>
    <br>
    // ... Handle ns2_MyBodyResponse ...<br>
  }<br>
}<br>
</code></pre>

<h3>Example</h3>
Assume the following:<br>
<ul><li>A SOAP service called "Friends"<br>
</li><li>A SOAP method called GetFavoriteColor that has a request attribute called Friend, and a response attribute called Color (i.e. you're asking it to return you the favorite color for a given a friend)<br>
</li><li>All the methods in this service ask for basic HTTP authentication, using a username and password that you acquired from the user via text fields</li></ul>

<pre><code>- (IBAction)pressedRequestButton:(id)sender {<br>
	FriendsBinding *bFriends = [[FriendsService FriendsBinding] retain];<br>
	bFriends.logXMLInOut = YES;<br>
        bFriends.authUsername = u.text;	<br>
        bFriends.authPassword = p.text;       <br>
<br>
        types_getFavoriteColorRequestType *cRequest = [[types_getFavoriteColorRequestType new] autorelease];<br>
	cRequest.friend = @"Johnny";<br>
	[bFriends getFavoriteColorAsyncUsingRequest:cRequest delegate:self];<br>
}<br>
<br>
- (void) operation:(FriendsBindingOperation *)operation completedWithResponse:(FriendsBindingResponse *)response<br>
{<br>
	NSArray *responseHeaders = response.headers;<br>
	NSArray *responseBodyParts = response.bodyParts;<br>
	<br>
	for(id header in responseHeaders) {<br>
		// here do what you want with the headers, if there's anything of value in them<br>
	}<br>
	<br>
	for(id bodyPart in responseBodyParts) {<br>
		/****<br>
		 * SOAP Fault Error<br>
		 ****/<br>
		if ([bodyPart isKindOfClass:[SOAPFault class]]) {<br>
			// You can get the error like this:<br>
			tV.text = ((SOAPFault *)bodyPart).simpleFaultString;<br>
			continue;<br>
		}<br>
		<br>
		/****<br>
		 * Get Favorite Color<br>
		 ****/		<br>
		if([bodyPart isKindOfClass:[types_getFavoriteColorResponseType class]]) {<br>
			types_getFavoriteColorResponseType *body = (types_getFavoriteColorResponseType*)bodyPart;<br>
			// Now you can extract the color from the response<br>
			q.text = body.color;<br>
			continue;<br>
		}<br>
// ...<br>
}<br>
</code></pre>

<h1>Advanced Options</h1>
The given example above covers basic authentication, as implemented in versions 0.6 and 0.7-pre1.<br>
The code in trunk has changed to support some more advanced security options, including advanced authentication and SOAP Security.<br>
To get a brief introduction to this features please follow this <a href='http://code.google.com/p/wsdl2objc/wiki/AdvancedOptions'>link</a>.<br>
<hr />
<h3>NOTE</h3>
If a WSDL has defined a string type that has attributes, then wsdl2objc will map it to a generic NSObject with a property called "<i>content" which will hold the actual string.<br>
So if you want to use the string, you have to call object.</i>content. If you want its attributes, they're also properties of the object.<br>
The short reason for this is that Cocoa makes it very hard to subclass NSStrings.