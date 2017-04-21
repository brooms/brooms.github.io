---
layout: post
title: "Adventures in Go"
modified:
author: andy_go
categories: blog
excerpt:
share: true
comments: true
tags: [software engineering]
image:
  feature: java_to_go.jpg
---

So after being insanely busy developing Java micro-services these last few months and juggling a whole bunch of other projects I've actually found the time to write about something. Whilst I was sandwiched between the Java and NodeJS runtimes, I found the time to actually use Golang in anger and I (mostly) have good things to say. 

Go (Golang) as a programming language is easy to learn, extremely powerful, a pleasure to work with and compiles extremely fast. The only major issue I found was the lack of formal structure and process around dependency management. This is something that should improve and mature with time.

For software engineers familiar with the Java stack, it is not that scary to make the leap from Java to Go. Here is a few things I learnt along the way that were integral to becoming productive in the language.

At a high level, it's important to note that even though the two languages (Go and Java) are mostly syntactically similar, they are in some cases fundamentally different.

Similar to Java, Go was inspired by C style languages, it is a statically typed, imperative language that is compiled. Like Java, it has inbuilt garbage collection which for most people (like me) who fear memory management and all the demons associated with it, this is a good thing.
It has an inbuilt concurrency model [(CSP model)](https://en.wikipedia.org/wiki/Communicating_sequential_processes), but it is important to note that it is not an object-oriented language as it lacks a type hierarchy.

A comparison at a glance:

<table class="table">
  <thead>
    <tr>
      <th style="background-color: #0d0d0d; border:none;"></th>
      <th style="background-color: #527e25">Java</th>
      <th style="background-color: #2e818f">Go</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Programming Model</td>
      <td>Procedural, Imperative, Object Oriented</td>
      <td>Procedural, Imperative, Object Oriented (kind of)</td>
    </tr>
    <tr>
      <td>Additional Paradigms</td>
      <td>Functional (Java 8), Concurrent</td>
      <td>Functional (language extensions), Concurrent</td>
    </tr>
    <tr>
      <td>Concurrency Model</td>
      <td>Threads (VM controlled)</td>
      <td>Go-routines, Channels for message passing</td>
    </tr>
    <tr>
      <td>Language Philosophy</td>
      <td>Write once, run anywhere</td>
      <td>Write once, compile anywhere</td>
    </tr>
    <tr>
      <td>Garbage Collection</td>
      <td>Yes</td>
      <td>Yes</td>
    </tr>
    <tr>
      <td>GC Model</td>
      <td>Serial or Parallel Mark-Sweep Variant</td>
      <td>Parallel Mark-Sweep</td>
    </tr>
    <tr>
      <td>Generic</td>
      <td>Yes</td>
      <td>No</td>
    </tr>
    <tr>
      <td>Reflective</td>
      <td>Yes</td>
      <td>Yes</td>
    </tr>
    <tr>
      <td>Fail safe I/O</td>
      <td>Yes</td>
      <td>Yes</td>
    </tr>
    <tr>
      <td>Polymorphism</td>
      <td>Yes</td>
      <td>No</td>
    </tr>
    <tr>
      <td>Typing Discipline</td>
      <td>Static, Strong</td>
      <td>Static, Strong</td>
    </tr>
    <tr>
      <td>Type Safety</td>
      <td>Yes</td>
      <td>Yes</td>
    </tr>
    <tr>
      <td>Type Equivalence</td>
      <td>Nominal Equivalence</td>
      <td>Structural Equivalence</td>
    </tr>
    <tr>
      <td>Type Signing</td>
      <td>Signed</td>
      <td>Signed and Unsigned Types</td>
    </tr>
  </tbody>
</table>

### No thanks Malloc, I'm fine

There is no right or wrong reason to choose Go over Java. In fact, I'm pretty certain now that for any project where Java may be the language du jour, Go can be used in it's place.

I lie a little bit in this blog post, I actually chose Go over C/C++. But as a Java programmer, there was no way in hell I was going to mess around with low level manual memory management and being so rusty in those languages I decided to take the opporunity to teach myself a new language and use it in an operational setting.
The end-result had to be cross platform compatible too, so C# was out of the question (without using mono, embedding the runtime etc). Since Go automatically embeds the Go runtime into the binary, this seemed like the perfect option. Go isn't exactly cross platform compatible, but this can be effectively managed by cross-compiling for other platforms using Go Docker containers. I was easily able to produce
working executables for MacOS, Solaris and Windows on my Macbook Pro.

In my case, I needed to create an encryption/decryption tool to hand over to a third-party that was able to read from an information source (file, in-flight message etc), encrypt or decrypt the data contained in the information source before outputting the result to another information source.
What I didn't want was to make it easy for a third-party to decompile and easily reverse engineer the code since we were dealing with sensitive information.

### Tooling and Environment Set Up

Environment setup is not much different from setting up a Java development environment. 

I chose to give Visual Studio Code a try instead of IntelliJ IDEA. It has excellent Go support through a number of third-party extensions including auto-completion, hints and a linter.

- Visual Studio Code
- Go SDK and Runtime
- Docker (for cross-compiling)

I've chosen to use the fish shell for MacOS. I can't speak more highly of it. If you want a shell built for cutting edge VGA graphics, it's definitely the shell for you.

Installing Go:

```
11:27:50, taken:78, du:83%, /Users/Andy/
> brew install go
Updating Homebrew...
==> Auto-updated Homebrew!
Updated 1 tap (homebrew/core).

...

==> Downloading https://homebrew.bintray.com/bottles/go-1.8.1.sierra.bottle.tar.gz
######################################################################## 100.0%
==> Pouring go-1.8.1.sierra.bottle.tar.gz
==> Caveats
A valid GOPATH is required to use the `go get` command.
If $GOPATH is not specified, $HOME/go will be used by default:
  https://golang.org/doc/code.html#GOPATH

You may wish to add the GOROOT-based install location to your PATH:
  export PATH=$PATH:/usr/local/opt/go/libexec/bin
==> Summary
🍺  /usr/local/Cellar/go/1.8.1: 7,030 files, 281.8MB
11:29:26, taken:96, du:84%, /Users/Andy/
> 
```

Configuring my Go [working directory](https://golang.org/doc/code.html#Workspaces) so that Go can actually find my Go code (in my shell config):

```
11:25:54, du:84%, /Users/Andy
> cd ~/.config/fish/
11:26:17, taken:23, du:84%, /Users/Andy/.config/fish
> vi config.fish

set -gx GOPATH /Users/Andy/development/github/go_code

11:26:32, taken:15, du:84%, /Users/Andy/.config/fish
> 
```

### A comparison of Primitive Types

There are a few more primitive types available in Go. Go offers both signed and unsigned numeric types, where as all Java numeric types are signed.

The Go `rune` primitive type is similar to the Java `char` type except that it explicitly represents a Unicode code point. Under the covers it is just an `int32`.

Go also treats strings as first class primitive types which are represented as a read-only slice of bytes.

Primitive types

<table class="table">
  <thead>
    <tr>
      <th style="background-color: #527e25;width:50%">Java</th>
      <th style="background-color: #2e818f;width:50%">Go</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>boolean</td>
      <td>bool</td>
    </tr>
    <tr>
      <td></td>
      <td>string</td>
    </tr>
    <tr>
      <td>short</td>
      <td>int16</td>
    </tr>
    <tr>
      <td>int</td>
      <td>int32</td>
    </tr>
    <tr>
      <td>long</td>
      <td>int64</td>
    </tr>
    <tr>
      <td>byte</td>
      <td>byte // alias for uint8</td>
    </tr>
     <tr>
       <td>char</td>
       <td>rune // alias for int32, represents a Unicode code point</td>
     </tr>      
      <tr>
        <td>float</td>
        <td>float32</td>
      </tr>  
     <tr>
       <td>double</td>
       <td>float64</td>
     </tr>  
      <tr>
        <td></td>
        <td>int  int8 
            uint uint8 uint16 uint32 uint64 uintptr // Unsigned types
            complex64 complex128 // complex types</td>
      </tr>  
  </tbody>
</table>

### Variable Declarations

Declaring a variable:

``` java
// Java
int buffer_size;

// Go
var buffer_size int
```

With initialisers:

``` java
// Java
int buffer_size = 1024;

// Go
var buffer_size int = 1024
```

Short initialisation (Go only, more useful than you realise):

``` go
// Go
buffer_size := 1024
// Unlike Java, Go supports short declarations inside functions
// Outside a function every statement must being with a keyword
```

Simple type conversions:

``` java
// Java
int buffer_size = 1024;
float bufferf = new Float(buffer_size).floatValue(); // Explicit unboxing

// Go
var buffer_size int = 1024
var bufferf float64 = float64(buffer_size)
```

Constants:

``` java
// Java
final long ISO_POLYNOMIAL = 0xD800000000000000L; 

// Go
const iso_polynomial = 0xD800000000000000 // Note lowercase, uppercase or titlecase explicitly exports the variable
```

### Flow Control

Flow control is syntactically identical to Java, except we drop the unnecessary braces used for wrapping.

```java
// Java
if ((v & 1) == 1) {
    v = (v >>> 1) ^ ISO_POLYNOMIAL;
} else {
    v = (v >>> 1);
}

// Go
if (v & 1) == 1 {
	v = (v >> 1) ^ isopoly
} else {
	v = (v >> 1)
}
```

A switch statement:

```java
// Java
switch (operation) {
    case "encrypt":
        System.out.format("Operation -> %s", operation);
        return;
    case "decrypt":
        System.out.format("Operation -> %s", operation);
        return;
    default:
        System.out.println("Invalid operation, expected one of [encrypt|decrypt]");
        usage();
}

// Go
switch operation {
    case "encrypt":
        fmt.Println("Operation -> ", operation)
        return
    case "decrypt":
        fmt.Println("Operation -> ", operation)
        return
    default:
        fmt.Println("Invalid operation, expected one of [encrypt|decrypt]")
        usage()
}
```

### For vs For-Range

Go actually only has one loop construct and that is the `for` loop. Java on the other hand has (as a base) the `for`, `while` and `do-while` loop constructs.

In addition, Go has a `for-range` style loop which loops over an array slice. This is pretty much semantically similar to the `for-each` style loop in Java.

```java
// Java
for (int i = 0; i < 256; i++) {
}

// Go
// note the use of short initialisation 
for i := 0; i < 256; i++ {
}

// Java
final long[] lookup_table;
for (final long num : lookup_table) {
    System.out.println("number -> " + num);
}

// Go
var lookuptable [256]uint64
for _, num := range lookuptable {
    fmt.Println("number -> ", num)
}
// Note the tuple return type (a Go language feature), the _ declares that we don't want to assign the value to a variable
// The first return type is the array index, the second the value at the index
```

### Methods vs Functions

Go is based on functions and function calls rather than concept of methods. However, they can essentially be seen as the same construct.

The example here shows a method and function that hashes an input byte array to a 64bit integer.

```java

// Java
public static long crc64(final byte[] data) {
    long sum = 0;
    for (final byte b : data) {
      final int lookupidx = ((int) sum ^ b) & 0xff;
      sum = (sum >>> 8) ^ LOOKUPTABLE[lookupidx];
    }
    return sum;
}
  
// Go
// The use of title case will export the function
func Crc64(data []byte) uint64 {
	var sum uint64
	for index := range data {
		lookupidx := (byte(sum) ^ data[index]) & 0xff
		sum = uint64(sum>>8) ^ lookuptable[lookupidx]
	}
	return sum
}
```

An interesting language feature in Go is the ability to declare a function that can return another function:
 
```go
func hash(data []byte) func() uint64 {
    return func() (ret uint64) {
        return Crc64(data)
    }
}
```

### Multiple Return Values

Go allows us to return multiple values. In Java, we would typically handle this by returning an Object and encapsulating the values within the object.
A typical use case I've found for returning multiple values is when returning the value at an array index, we can return the index and value or return an error type if any errors have occurred during execution of a function.

```java
// Java (much more verbose, we need to define a Tuple return type
public class Tuple {

    private final int index;
    private final String value;

    Tuple(final int index, final String value) {
      this.index = index;
      this.value = value;
    }
}

public Tuple findValueThatStartsWith(String prefix, String[] values) {
    int index = 0;
    for (final String value : values) {
      if (value.startsWith(prefix)) {
        return new Tuple(index, value);
      }
      index++;
    }
    return new Tuple(-1, null);
}

// Go
func findValueThatStartsWith(prefix string, values []string) (int, string) {
    for index, str := range values {
        if strings.HasPrefix(str, prefix) {
            return index, str
        }
    }
    return -1, ""
}
```

### Function Pointers

Pointers, something I'm familiar and comfortable with using but tend to avoid unless absolutely necessary. Go has a language feature that allows us to return a pointer to a function. There is no equivalent language feature in Java.

Function pointers can open up a can of worms, all I really needed to know is that they exist and may be used to enhance the functionality of complex types.
 
```go
// A simple type hierarchy
type Key interface {
	GetKey() []byte
}

type AesKey struct {
	keydata []byte
	Key     Key
}

// Simple function pointer to wrap a return to the key
func (key *AesKey) GetKey() []byte {
	return key.keydata
}
```

### Objects vs Structs

Unlike Java, Go does not support "Objects" as such. Instead Go uses the `struct` keyword to compose complex types and their interfaces to explicitly define similarities between structs.

``` java
// Java

public interface Key {
    byte[] getKey();
}

public class AesKey implements Key {

    private byte[] key = {
                            (byte) 0xfe, (byte) 0x63, (byte) 0x11, (byte) 0x17,
                            (byte) 0xfe, (byte) 0x63, (byte) 0x11, (byte) 0x17,
                         };

    // Default explicit constructor
    public AesKey() {
    }

    public byte[] getKey() {
        return this.key;
    }
}

// Finally, instantiation
AesKey key = new AesKey();
System.out.format("Key -> %s", new String(key.getKey()));

```

``` go
// In Go, the equivalent would be
type Key interface {
	GetKey() []byte
}

type AesKey struct {
	keydata []byte
	Key     Key
}

// Simple function pointer to wrap a return to the key
func (key *AesKey) GetKey() []byte {
	return key.keydata
}

// Inline creation
key := new(AesKey)
key.keydata = []byte{0xfe, 0x63, 0x11, 0x17, 0xfe, 0x63, 0x11, 0x17}
fmt.Println("Key -> ", key.GetKey())

```

### try-catch-finally vs defer-panic-recover

In Java, we wrap non-deterministic error prone code within a try-catch-finally block that allows us to gracefully handle errors and recover accordingly. 

Go has similar language features in the form of three in-built functions: 

* `defer` which defers the execution of a function by pushing it onto a list, the list of deferred functions execute once the surrounding function returns (similar to `try-finally` in Java).
* `panic` which stops the normal flow of control and begins panicking (analogous to throwing an `Exception` in Java)
* `recover` which allows us to regain control of a panicking goroutine (similar to `catch` in Java)

A typical use case in Java for try-catch-finally is handling I/O operations.

```java
public static void main(String... args) {
    FileInputStream inputStream = null;
    try {
      inputStream = new FileInputStream("input.txt");
      BufferedInputStream reader = new BufferedInputStream(inputStream);

      final byte[] buffer = new byte[1024];

      int readBytes;
      do {
        readBytes = reader.read(buffer);

        if (readBytes != -1) {
          System.out.format("Read %d bytes -> %s", readBytes, new String(buffer, 0, readBytes));
        }

      } while (readBytes != -1);

    } catch (IOException ex) {
      System.out.format("Unable read file -> %s", ex.getMessage());
    } finally {
      try {
        if (null != inputStream) {
          inputStream.close();
        }
      } catch (IOException ioex) {
          throw new RuntimeException("Unable to close input stream");
      }
    }
  }
```

... and the same, functionally equivalent code in Go with defer-panic-recover.

```go
func main() {

	// Deferred function to capture control and fail gracefully
	defer func() {
		if r := recover(); r != nil {
			fmt.Println("Unable read file ->", r)
		}
	}()

	inputfile, err := os.Open("input.txt")
	if err != nil {
		panic(err)
	}

	// Defer the call to close the input stream
	defer func() {
		err := inputfile.Close()
		if err != nil {
			panic(err)
		}
	}()

	fileReader := bufio.NewReader(inputfile)

	bufferSize := 1024
	buffer := make([]byte, bufferSize)
	for {
		// Read a chunk and fill the buffer
		chunk, err := io.ReadFull(fileReader, buffer)
		if err != nil {
			if err == io.ErrUnexpectedEOF {
				// ignore, buffer was partially filled but that's ok, final read
			} else if err == io.EOF {
				// End of file
				break
			} else {
				panic(err)
			}
		} else if chunk == 0 {
			// No more data to read
			break
		}

		fmt.Printf("Read %d bytes -> %s\n", chunk, string(buffer))
	}
}
```

### Build and Dependency Management

A discussion on build tools and dependency management in both languages is probably an area that could become a post within its own right. So I'll keep this section fairly brief.

Obviously the Java ecosystem is incredibly mature and we are spoilt for choices when it comes to build systems or dependency management tools. Typically, I tend to either use Maven or Gradle, both are exceptionally good at managing third party and transitive dependencies.

Go, however, lacks a consistent model for dependency management and this is an area where the language ecosystem lets us down. Dependency management in Go is just not good... at the moment. The community is working on a new dependency management tool called `dep`, which will be the official dependency management tool. As they state on the [wiki](https://github.com/golang/go/wiki/PackageManagementTools), the dep tool is currently being implemented, in pre-alpha state and should be used with caution as "Lots of functionality is knowingly missing or broken".

For now, you can either use a third-party tool or the existing [`godep`](https://github.com/tools/godep) tool. This tool works by specifying third-party dependencies within a Godeps json file, then by retrieving these dependencies from Github. It sucks. To be honest, I'm very uncomfortable about retrieving production ready third-party dependencies by using `git clone` on a revision tag and to-date have been extremely careful in how I have been retrieving dependencies.

### Cross-Compiling

Since Java is a (mostly) interpreted language and compiles to JVM byte-code, cross-compilation to other architectures is not required. Cross-compiling Go, however, may be required if you want to run your executable on another platform. 

Docker containers make it really easy to cross-compile Go code. 

After installing Docker, you can install the Docker [containers for Go](https://hub.docker.com/_/golang/):

```
> docker pull golang
Using default tag: latest
latest: Pulling from library/golang
Digest: sha256:fb555736a861ae3ee8d1d45162d0a74e187840e4fd5e852830b1c7a7f72aacf7
Status: Image is up to date for golang:latest
```

Compiling for windows/386 on MacOS:

```
> ls
test.go
14:35:03, du:87%, /Users/Andy/development/github/go_code/src/test
> docker run --rm -v $PWD:/go/src/test -v $PWD/target:/go/bin -w /go/src/test -e GOOS=windows -e GOARCH=386 golang:1.8 go build -v -o /go/bin/test.exe
runtime/internal/sys
runtime/internal/atomic
runtime
errors
internal/race
sync/atomic
unicode
unicode/utf8
math
sync
internal/syscall/windows/sysdll
unicode/utf16
io
syscall
bytes
strconv
bufio
reflect
internal/syscall/windows
internal/syscall/windows/registry
time
os
fmt
test
14:35:14, taken:11, du:87%, /Users/Andy/development/github/go_code/src/test
```

Or for linux:

```
> docker run --rm -v $PWD:/go/src/test -v $PWD/target:/go/bin -w /go/src/test -e GOOS=linux -e GOARCH=amd64 golang:1.8 go build -v -o /go/bin/test_linux
test
14:37:06, taken:243, du:87%, /Users/Andy/development/github/go_code/src/test
> 
```

You can even cross-compile for [solaris SPARC](https://hub.docker.com/r/craigbarrau/golang-solaris-sparc/) and many other platforms. Remember that Go [embeds](https://golang.org/doc/faq#Why_is_my_trivial_program_such_a_large_binary) the Go runtime into each binary so it's a really nice way of ensuring your Go code will build and run on other platforms.

### Conclusion

I've really enjoyed using Go on this project and it has been incredibly satisfying producing a production ready tool in a new language. In a way, I was proving to myself that I could be productive in a language other than something that falls under the the Java or Web stack umbrella.

Two areas I haven't explored yet are concurrency features and the world of automated unit testing for Go. Something I'll be doing in the next few days, creating an automated test suite to further industralise this Go project.

I love the language and really didn't find it that difficult to make the syntactic jump from Java -> Go. Go gets me excited, if they improve the way third-party dependencies are managed I could definitely see us using the language as a direct replacement for Java. Another language that is a piqued my interest in the last few months is Kotlin (something I will play with in the future) and I'd be interested to see how it stacks up against Go.