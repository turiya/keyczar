# Interop Testing #
The code for interop testing is located in the interop folder. To run the interop tests, execute "interop.py".
## Running Interop Testing ##
From the interop directory, run
```
./interop.py
```

### Flags and Features ###
#### Create ####
This will cause keyczar to run without creating new key files and use existing key files.
```
./interop.py --create=n
```
This will allow the same tests to be run on the same keysets multiple times for use when debugging errors that are dependent on the key data. It will also speed up the length of time of running interoperability tests.
#### Verbose ####
This will cause additional output to be printed with the basic output. It will print the output of the create operations and the exact tests that are being ignored.
```
./interop.py --verbose=y
```
#### Display ####
This will print all of the tests that will be ran, but it will not run any of them.
```
./interop.py display
```
#### Options ####
This will print out all of the applicable options for each operation. It will not account for ignored tests. To see which tests will be ignored this list should be cross referenced with the ignoredTests config file or with the output of display. This is useful for constructing a new implementation.
```
./interop.py options
```
## Writing a new Implementation ##

### Implementation JSON ###
For a language to be used with the interop testing infrastructure there needs to be a command line interface that the infrastructure can call. This command line interface must be added to the list of implementations in the "config/implementations.json" file. The format of this interface is:
```
{
  "implementationName" : ["../listOfArgs","--needed","to","--call,"implementation"],
  ...
}
```
### The command line tool ###
The command line tool for the implementation will be passed one command line argument, a JSON string. This JSON string will be a dictionary of values that will instruct the command line tool what to do. There will always be a "command" specified in this dictionary. This attribute will specify the rest of the format of the JSON string. The three possible values for "command" are "create", "generate", and "test". These will be discussed in the following sections.

#### Create ####
This command causes a keyset to be created. It assumes that the implementation will run keyczar tool with the given arguments. The "createFlags" arguments should be passed into the first call of keyczart and the "addKeyFlags" should be the arguments to a second keyczart call. The directories where the keys will be created will be emptied beforehand. The output will be printed as part of interop.py. If there is an error, have the command line tool return a non-zero return code.

Example of parameters for a create command:
```
{
  "command": "create",
  "createFlags": ["create", "--name=rsa-sign1024", "--location=keys/python/rsa-sign1024", "--purpose=sign", "--asymmetric=rsa"],
  "addKeyFlags": ["addkey", "--status=primary", "--size=1024", "--status=primary", "--location=keys/python/rsa-sign1024"]
}
```

#### Generate ####
The Generate command will generate data for other implementations to test. The Generate function will run the designated operation, using the algorithm and options listed. For example, if the operation involves signing. The generate function will compute the signature and pass that into the output. If it is an encryption operation, the output will be the encrypted data. The possible types of operations can be found in the "config/operations.json" file, along with the other options possible for generateOptions. The keys for any given algorithm can be loaded in from the path resulting from the concatenation of the "keyPath" and "algorithm" name. To understand what an individual option does, checking the code of another implementation is useful. In general, the encoding will specify whether the output is WebSafeBase64 "encoded" or it is "unencoded" and the "class" option will specify what kind of keyczar class is used: either a Crypter or an Encrypter. "testData" is the value of the data to be encrypted/signed. This will be an ASCII string. If there is an error that occurs during generation, please print a useful debugging message and return a non-zero exit code.
Example input:
```
{
  "command": "generate",
  "operation": "attached",
  "keyPath": "keys/cpp",
  "algorithm": "dsa1024",
  "generateOptions": {"encoding": "unencoded"},
  "testData": "This is some test data."
}
```
The result from the Generate function is sent to stdout. Be sure that other forms of logging are disabled or only occur when there is a failure. Otherwise the JSON will not be able to be read by interop.py.  The format of the JSON has an output parameter that contains the WebSafeBase64 encoding of the output of the Generate function. Note that this may be the WebSafeBase64 encoding of an already WebSafeBase64 encoded output. The only current exception to this formatting is SignedSessionOperations which in addition to an output parameter also have a sessionMaterial parameter which contains the sessionMaterial (all current implementations only output the sessionMaterial in WebSafeBase64 format, so the WebSafeBase64 format is not applied again).
Example output:
```
{
  "output" : "AN7ZKnEAAAAXVGhpcyBpcyBzb21lIHRlc3QgZGF0YS4wLQIUXO1lHO3-X43qIYayDyNR3LNrOjwCFQCmW7NA6qY1tQy83UkTRyySyWfcVw"
}
```
#### Test ####
The test command should ensure that the output from the generate command is valid. For encryption operations, this means decrypting the data and comparing it to the original data. For signing operations, this means verifying the signature.

The data is similar in format to the generate command. The output is a json string holding a dictionary of values, the output of the generate command. "generateOptions" show what options were used to generate the output. "testOptions" are options that are used the same output from the generate, but are different ways the data can be verified. For example, in signing operations, the "class" option can specify to verify using a "signer" or a "verifier". These options are also listed in the operations.json file. If the code fails to verify or there is an error, the test command should return a non-zero exit code along with some debugging messages to why the test failed (like a stack trace for instance). If the test command succeeds, the output does not matter and will not be output by "interop.py".
```
{
  "command": "test",
  "operation": "attached",
  "keyPath": "keys/cpp",
  "algorithm": "dsa1024",
  "generateOptions": {"encoding": "unencoded"},
  "testData": "This is some test data."
  "testOptions": {"class": "signer"},
  "output": "{\"output\":\"AN7ZKnEAAAAXVGhpcyBpcyBzb21lIHRlc3QgZGF0YS4wLQIUXO1lHO3-X43qIYayDyNR3LNrOjwCFQCmW7NA6qY1tQy83UkTRyySyWfcVw\"}\n"
}
```
### Adding Exceptions ###
In general, all possible combinations of implementations, operations, algorithms, key sizes, and options will be used. If there is a certain set of operations/algorithms/options that are not interoperable for some reason, they can be added to the ignoredTests.json file. The format for adding an exception is shown in the example below. The wildcard character ("`*`") means that the ignored test applies for all possible implementations, algorithms, operations, or options.

The implementation lists the implementation that this affects. This is for both generate and test.

The operation lists the operations that this affects. This is for both generate and test.

The algorithms lists the algorithms that this applies to. This is for both generate and test.

The options list the options that are required. If the option is not present it will not be constrained. It is a dictionary with names and lists.

The reason is the reason that these tests are ignored to be displayed each time the interop tests are run.
```
{
  "implementation" : ["*"],
  "operation" : ["signedSession"],
  "algorithm" : ["rsa-crypt1024"],
  "options" : {"*" : "*"},
  "reason" : "metadata in signed session is too big to be encrypted by 1024 bit RSA"
}
```