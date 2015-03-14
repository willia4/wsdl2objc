The actual running of WSDL2ObjC is broken into 2 stages:

# Stage 1: Parsing #
The parsing stage deserializes the specified WSDL file (and each of its xsd imports) into an in-memory representation of the web service description.

# Stage 2: Code Generation #
The code generation stage takes advantage of a modified STS Template Engine to create a class for each complex type, an enum for each enumerated simple type, a service class, a binding class, and operations in the binding class for each specified operation.  WSDL2ObjC is packaged as an app instead of a command line tool so it can be bundled with the templates used to generate this code.