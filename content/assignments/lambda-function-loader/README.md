# Lambda Function Loader

## Application Development

All development work should be carried out exclusively within the [virtual machines dedicated to Operating Systems](https://cs-pub-ro.github.io/operating-systems/resources#virtual-machine).

**Note:**
It's crucial to avoid running tests locally, either on personal computers or within your allocated virtual machines.
Discrepancies might arise between local environments and the dedicated OS virtual machines.
Grading will consider results obtained solely in the OS virtual machine environment.

The core of this project revolves around the construction of a system capable of dynamic library loading and subsequent function execution.
The project architecture embraces a [client-server model](https://cs-pub-ro.github.io/operating-systems/Lab/IO/client-server-model), where the server is responsible for receiving and fulfilling requests to execute functions sourced from specific dynamic libraries present within the system.

This functionality marks the initial phase toward realizing capabilities akin to [AWS Lambda](https://aws.amazon.com/lambda/).
In this context, users can load and execute functions across distinct servers upon request.
In our envisioned project, these functions are pre-implemented within libraries, and the client instigates their execution by issuing specific commands to the server.
Detailed below are the primary requisites and potential enhancements.

### Implementation Details and Notes

The implementation involves the capability to receive commands utilizing [UNIX sockets](https://cs-pub-ro.github.io/operating-systems/Lab/IO/beyond-network-sockets#unix-sockets), encompassing the designation of a library (the path to the library file) and optionally specifying a function.
A command transmitted from the client to the server conforms to the format: `<libname> [<funcname> [<filename>]]` where:

- `libname` represents the path to the intended library.
- `funcname` is an optional parameter denoting the function's name within the library.
By default, if unspecified, the function used is named `run`.
- `filename` is an optional parameter indicating the filename containing input data, provided as an argument to the `funcname` function.

The resulting output data generated from execution will be written to a file, and the file's name is returned as a response to the client.
This returned file exclusively contains messages printed to the standard output by the invoked library function.

The communication sequence between the client and server follows this structure:

```plaintext
Client                                              Server
                                                    listen()
connect()                                           accept()
send()     -—---libname [funcname [filename]]-----> recv()
recv()     <--------------outputfile--------------- send()
```

Post receiving and parsing a message from a client, the server systematically triggers a series of functions, each serving distinct purposes:

- Prepare/pre-hooks: Initial operations independent of loading the library or executing functions.
- Library loading: Loading the library and other related management tasks.
- Execution of requested functionality:
Execution of the specified function from the library.
If `funcname` is absent, the `run` function is invoked.
The `filename` parameter is permissible only when `funcname` is specified.
- Unloading the library: Procedures associated with unloading the library from memory.
- Post-hooks:
Operations conducted after execution, regardless of the library's presence in memory.

**Tip:**
The mentioned five functions serve as guidelines.
Depending on the implementation, some functions might remain as stubs if not fully implemented.

**Important:**
The server's implementation must be **parallelized** to expedite the handling of client requests.
Each team is responsible for selecting the parallelization model (via processes, threads, hybrid methods, utilizing work pools, etc.) based on the project's objectives.

Fundamental functionality necessitates implementation in C, designed for the Linux operating system, employing the POSIX API:

- Establishment of client-server connection.
  - The connection will utilize UNIX Sockets through the `libipc.so` library.
  - Necessary functions for creating, connecting, and sending/receiving data in this library (`libipc.so`) will be included.
  - The library can be modified as long as the provided test client (`checker/client.c`) compiles and connects to the server successfully.
  - The test client (`checker/client.c`) remains unmodifiable.
  Any changes required for additional functionalities should be implemented in a new test client.
- Communication between client and server: The client transmits requests following the specified format and awaits the response, which contains the file path in the system storing the results.
The server receives and processes the client's request.
- Server request handling: Parsing the received command, loading the library, and executing the requested function from the library.
  - In case of an error, an error message, `Error: <command> could not be executed.`, followed by relevant error messages (Hint: utilize `strerror()`), will be written to the output file.
  - The output file will be created with a randomly generated name employing the `mkstemp()` function, utilizing `OUTPUTFILE_TEMPLATE` as a filename template.
- Returning a response to the client:
The response entails the filename generated in the preceding step based on `OUTPUTFILE_TEMPLATE`.
- Parallelizing the client handling mode.
  - Ensuring synchronization of output data and library loading/unloading does not hinder the functionality of other threads/processes.

**Warning:**
It is crucial to ensure proper server functionality in your implementation.
Executing functions from a user-requested library can result in unexpected behavior (e.g. invalid accesses, forced closures, etc.).
The server's implementation must be robust, enabling continuous execution even in such cases.

### Necessary Theoretical Concepts

What is needed for the implementation of the proposed project:

- Understanding inter-process communication - using Unix sockets and read/write or send/receive operations:
  - [I/O Course](https://cs-pub-ro.github.io/operating-systems/Lecture/IO/)
  - [I/O Lab](https://cs-pub-ro.github.io/operating-systems/I/O/)
- Understanding the API for loading/unloading libraries and executing functions from dynamic libraries:
  - Hint: `man dlopen`
- Working with shared data between processes or threads:
  - [Compute Course](https://cs-pub-ro.github.io/operating-systems/Lecture/Compute/)
  - [Compute Lab](https://cs-pub-ro.github.io/operating-systems/Compute/)
- Working with memory
- Working with files

### Testing

#### Example Run

```bash
## Starting the server
student@os:~/skel$ ./server

## Client
# Execute the "run" function from libbasic.so
student@os:~/checker$ ./client $(realpath libbasic.so)
Output file: /home/so/checker/output/out-cfy0fl

student@os:~/checker$ cat /home/so/output/out-cfy0fl
run

# Execute the "function" function from libbasic.so
student@os:~/checker$ ./client $(realpath libbasic.so) function
Output file: /home/so/checker/output/out-vc7s03

student@os:~/checker$ cat /home/so/output/out-vc7s03
function

# Execute the "cat" function from libbasic.so with the argument "/home/so/checker/Makefile"
student@os:~/checker$ ./client $(realpath libbasic.so) cat $(realpath Makefile)
Output file: /home/so/checker/output/out-y732bN

student@os:~/checker$ cat /home/so/output/out-y732bN
CC=gcc
[...]
```

#### Checker Run

You have a checker available for partial verification of your implementation.

```text
student@os~$ cd checker

student@os~$ ./checker.sh
```

### Additional Tasks / Tie-breakers

All teams must implement the fundamental features as a base requirement.
Additionally, for differentiation purposes, teams have the option to introduce supplementary functionalities to the application.
Each added functionality should be thoroughly documented in a README file, providing details on its purpose and the testing methodologies employed.

Here are some potential additional functionalities to consider (or propose other relevant ones):

- Developing an interface for client-server connectivity:
Depending on runtime parameters, utilize either network or UNIX sockets.
- Expanding the client handling mode:
Allow the server to process multiple commands until encountering an "exit"/"quit" command.
- Enhancing server security:
Implement measures to prevent data leaks.
- Conducting performance optimizations.
- Establishing application logging.
- Managing configuration files/options:
Configure the server during startup based on specified options/configuration files such as maximum client count, socket type preferences, etc.
- Real-time server monitoring and statistical extraction (e.g. client count, loaded libraries, memory usage, etc.).
- Adapting the implementation to another programming language (the initial skeleton is in C, but the communication's nature allows implementation in any language meeting project requirements).
- Generating a suitable response in cases where the requested function exceeds a `TIMEOUT` or performs actions leading to server shutdown (e.g. invalid memory access, exit/abort calls).
The solution's complexity may vary based on how the client's connection termination is handled (whether an error message is conveyed) and how the cause of execution termination is identified.
