This project is a replacement for Apple's WSMakeStubs utility and WS-Core.
Compile from source for the latest improvements.

9/15/2009 - Version 0.6 release notes:
  * Fixes for iPhone compatibility
  * Greatly improved WSDL compatibility

11/11/2008 - Version 0.5 release notes:
  * iPhone compiling added to the Usage Instructions (see http://code.google.com/p/wsdl2objc/wiki/UsageInstructions )
  * Generated code will now import Foundation instead of Cocoa
  * Added an option to add a tag "Svc" to the end of the service name to avoid naming conflicts.  Use this if you're seeing "redefinition of struct" errors on compiling
  * Improved WSDL compatibility (added support for xsd:float type)

10/24/2008 - Version 0.4 release notes:
  * Improved WSDL compatibility
  * long, double, int, etc types are now processed with NSNumbers instead of needing to use malloc()/free()