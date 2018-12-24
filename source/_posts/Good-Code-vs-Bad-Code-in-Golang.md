---
title: Good Code vs Bad Code in Golang
date: 2018-11-16 14:53:57
categories:
- Golang
tags:
- Golang
- Go
---
`This article was transferred from @teivah` [Original](https://medium.com/@teivah/good-code-vs-bad-code-in-golang-84cb3c5da49d)
![](./1-yh90bW8jL4f8pOTZTvbzqw.png)

Recently, I was asked to detail what makes a good code or a bad code in Golang. I found this exercice very interesting. Actually, interesting enough to write a post about that. To illustrate my answer, I have taken a concrete use cases I faced in the Air Traffic Management (ATM) domain. The project is available in [Github](https://github.com/teivah/golang-good-code-bad-code).

# Context
First, few words to explain the context of the implementation.
Eurocontrol is the organization managing the air traffic across Europe countries. The common network for exchanging data between Eurocontrol and an Air Navigation Service Provider (ANSP) is called AFTN. This network is mainly used to exchange two different message types: ADEXP and ICAO messages. Each message type has its own syntax but in terms of semantic, both types are equivalent (more or less). Given the context, performance must be a key element for the implementation.
This project has to provide two implementations for parsing ADEXP messages (ICAO is not managed in the frame of this exercise) based on Go:
* A bad implementation (package name: [bad](https://github.com/teivah/golang-good-code-bad-code/tree/master/bad))
* A refactored implementation (package name: [good](https://github.com/teivah/golang-good-code-bad-code/tree/master/good)).
An example of an ADEXP message can be found [here](https://raw.githubusercontent.com/teivah/golang-good-code-bad-code/master/resources/tests/adexp.txt).
In the frame of this exercise, the parsers handle only a subset of the fields we can find in an ADEXP message. Yet, It is still relevant to illustrate common Go mistakes.

# Parsing
In a nutshell, an ADEXP message is a set of tokens. A token type can be either:
````
-ARCID ACA878
````
Meaning the ARCID (aircraft identifier) is ACA878.
````
-EETFIR EHAA 0853
-EETFIR EBBU 0908
````
This example is a list of FIR (Flight Information Region). The first FIR is EHAA 0853 whereas the second one is EBBU 0908.
````
-GEO -GEOID GEO01 -LATTD 490000N -LONGTD 0500000W
-GEO -GEOID GEO02 -LATTD 500000N -LONGTD 0400000W
````
A repeating list of tokens. Each line contains a sublist of tokens (in this example GEOID, LATTD, LONGTD).
Given the context, it is important to implement a version leveraging parallelization. So the algorithm is the following one:
* A preprocessing step to clean and rearrange the input message (we have to clean the potential white spaces, rearrange the tokens which are multi-lined like COMMENT etc.)
* Then splitting each line in a given goroutine. Each goroutine will be in charge to process one line and to return the result.
* Last but not least, gathering the results and returning a Message structure. This structure is a common one regardless of the message type (ADEXP or ICAO).
Each package contains an adexp.go file exposing the main function ParseAdexpMessage().

# Step-by-step comparison
Let’s now see step by step what I consider as a bad code and how I refactored it.

# String vs []byte
The bad implementation handles only string inputs. As Go offers a strong support for bytes operations (basic operation like trim, regexp etc.) and that the input will most likely by a []byte (considering AFTN messages are received through TCP), there’s actually no good reason to force a string input.

# Error management
The error management is kind of terrible the bad implementation.We can find some cases where potential errors returned in the second argument are not even managed:
````
preprocessed, _ := preprocess(string)
````
The good implementation deals with each potential error:
````
preprocessed, err := preprocess(bytes)
if err != nil {
  return Message{}, err
}
````
We can also find some mistakes in the bad implementation like in the following code:
````
if len(in) == 0 {
  return "", fmt.Errorf("Input is empty")
}
````
The first mistake is a syntax one. An error string shall neither be capitalized nor end with a punctuation according to Go standards. The second mistake is due to the fact that if an error string is a simple constant (no formatting is required), a call to errors.New() is slightly more performant.
The good implementation looks like:
````
if len(in) == 0 {
	return nil, errors.New("input is empty")
}
````

# Avoid nesting
The mapLine() function is a good example of avoidable nesting calls. The bad implementation:
````
func mapLine(msg *Message, in string, ch chan string) {
    if !startWith(in, stringComment) {
        token, value := parseLine(in)
        if token != "" {
            f, contains := factory[string(token)]
            if !contains {
                ch <- "ok"
            } else {
                data := f(token, value)
                enrichMessage(msg, data)
                ch <- "ok"
            }
        } else {
            ch <- "ok"
            return
        }
    } else {
        ch <- "ok"
        return
    }
}
````
On the opposite, the good implementation is a flat representation:
````
func mapLine(in []byte, ch chan interface{}) {
    // Filter empty lines and comment lines
    if len(in) == 0 || startWith(in, bytesComment) {
        ch <- nil
        return
    }

    token, value := parseLine(in)
    if token == nil {
        ch <- nil
        log.Warnf("Token name is empty on line %v", string(in))
        return
    }

    sToken := string(token)
    if f, contains := factory[sToken]; contains {
        ch <- f(sToken, value)
        return
    }

    log.Warnf("Token %v is not managed by the parser", string(in))
    ch <- nil
}
````
This makes the code easier to read in my opinion. Furthermore, this flat representation must also be applied to errors management. As an example:
````
a, err := f1()
if err == nil {
    b, err := f2()
    if err == nil {
        return b, nil
    } else {
        return nil, err
    }
} else {
    return nil, err
}
````
Should be replaced by:
````
a, err := f1()
if err != nil {
    return nil, err
}
b, err := f2()
if err != nil {
    return nil, err
}
return b, nil
````
Once again, the second code version is easier to read.

# Passing data by reference or by value
The signature of the preprocessing function in the bad implementation is:
````
func preprocess(in container) (container, error) {
}
````
Given the context of this project (performance does matter) and considering a message can potentially be quite heavy, a better option was to pass a pointer to the container structure instead. Otherwise, in the previous example the container value will be copied during each call.The good implementation does not face this problem as it deals with slices (a simple 24-byte structure regardless of the underlying data):
````
func preprocess(in []byte) ([][]byte, error) {
}
````
More generally speaking, passing data either by reference or by value must not be an idiomatic choice.
Passing data by value could also help to make sure a function will not cause any side effect (like mutating the data passed in the function input). This has several benefits like unit testing or refactoring a code for parallelization for example (otherwise we need to check each subfunction to see if a mutation is made).
I do believe such choice must really be done carefully depending on the project context.

# Parallelization
The bad implementation is based on a good initial idea: leveraging goroutines to parallelize the data processing (one goroutine per line).
This is achieved in the bad implementation by iterating over the number of lines and spawning a mapLine() call in a goroutine.
````
for i := 0; i < len(lines); i++ {
    go mapLine(&msg, lines[i], ch)
}
````
The mapLine() function takes in arguments three parameters:
* A pointer to the final Message structure to be returned. It means each mapLine() will enrich the same variable.
* The current line
* A channel used for sending a notification once the processing of the line is done

Sending a pointer to a shared Message variable breaks one of the main Go principles:
Don’t communicate by sharing memory, share memory by communicating.
There are two main drawbacks to passing this shared variable:
* Drawback #1: Slices concurrent modifications

Because the structure contains some slices which can be modified concurrently (by two or more goroutine at the same time), in the bad implementation we had to deal with mutexes.
For example, the Message structure contains a Estdata []estdata.
Modifying the slice by appending another estdata must be done this way:
````
mutexEstdata.Lock()
for _, v := range value {
    fl := extractFlightLevel(v[subtokenFl])
    msg.Estdata = append(msg.Estdata, estdata{v[subtokenPtid], v[subtokenEto], fl})
}
mutexEstdata.Unlock()
````
Actually, except very specific use cases, having to use a mutex in a goroutine might be a code smell.
* Drawback #2: False sharing

Sharing memory across threads/goroutines is not a good idea due to potential false sharing (a cache line in a given CPU core cache can be invalidated by another CPU core cache). This means we should avoid as much as possible sharing the same variable across threads/goroutines if they intend to mutate it.
In this very example, though, I don’t think false sharing has a huge impact as the input file is quite light (running a performance test with padding fields in the Message structure gives more or less the same result). Yet, that’s always something important to bear in mind in my opinion.
Let’s see now how is the good implementation dealing with the parallelization:
````
for _, line := range in {
    go mapLine(line, ch)
}
````
Now, the mapLine() takes only two inputs:
* The current line
* A channel. This time this channel is not used to simply send a notification once a line processing is done but also to send the actual result. It means it is not up to the goroutines to modify the final Message structure.

Gathering the results is done this way by the parent goroutine (the one spawning the mapLine() calls in separate goroutines):
````
msg := Message{}

for range in {
    data := <-ch

    switch data.(type) {
        // Modify msg variable
    }
}
````
This implementation is more aligned, in my opinion, with Go principles to share memory only by communicating. The Message variable is modified by a single goroutine to prevent potential concurrent slices modifications and false sharing.
One potential criticism even with the good code is to spawn a goroutine for each line. Such implementation will work because an ADEXP message will not contain thousands of lines. Yet, the simple implementation one request triggering one goroutine does not scale very much under very high throughput. A better option would have been to create a pool of reusable goroutines for example.
`Edit`: This assumption (one line = one goroutine) was definitely not a good idea as it leads to way too much context switches. For additional information, please take a look at the link in the further reading chapter (at the end of the post).
Line processing notification
In the bad implementation, as described above, once a line processing is achieved by a mapLine() we should indicate it to the parent goroutine. This is done using a chan string channel and a call using:
````
ch <- "ok"
````
As the parent does not actually check the value sent by the channel, a better option would have been to use chan struct{} with a ch <- struct{}{} or even better (GC wise) to use a chan interface{} with a ch <- nil.
Another approach (even cleaner in my opinion) would have been to use a sync.WaitGroup as the parent goroutine just need to continue its execution once every mapLine() is done.

# If
The Go if statement allows passing a statement before the condition.
An improved version of:
````
f, contains := factory[string(token)]
if contains {
    // Do something
}
````
Can be the following implementation:
````
if f, contains := factory[sToken]; contains {
    // Do something
}
````
It slightly improves the code readability.

# Switch
Another mistake with the bad implementation is to forget the default case in the following switch:
````
switch simpleToken.token {
case tokenTitle:
    msg.Title = value
case tokenAdep:
    msg.Adep = value
case tokenAltnz:
    msg.Alternate = value 
// Other cases
}
````
The default can be optional if the developer thought about all the different cases. Yet, it is definitely better to catch this specific case like in the following example:
````
switch simpleToken.token {
case tokenTitle:
    msg.Title = value
case tokenAdep:
    msg.Adep = value
case tokenAltnz:
    msg.Alternate = value
// Other cases    
default:
    log.Errorf("unexpected token type %v", simpleToken.token)
    return Message{}, fmt.Errorf("unexpected token type %v", simpleToken.token)
}
````
Handling the default case would help in catching potential bugs made by developers as soon as possible in the development process.

# Recursion
The parseComplexLines() is a function to parse a complex token. The algorithm in the bad code is done using recursion:
````
func parseComplexLines(in string, currentMap map[string]string, 
	out []map[string]string) []map[string]string {

    match := regexpSubfield.Find([]byte(in))

    if match == nil {
        out = append(out, currentMap)
        return out
    }

    sub := string(match)

    h, l := parseLine(sub)

    _, contains := currentMap[string(h)]

    if contains {
        out = append(out, currentMap)
        currentMap = make(map[string]string)
    }

    currentMap[string(h)] = string(strings.Trim(l, stringEmpty))

    return parseComplexLines(in[len(sub):], currentMap, out)
}
````
Yet, Go does not support tail-call elimination to optimize sub-function calls. The good code produces the exact same result but using an iterative algorithm:
````
func parseComplexToken(token string, value []byte) interface{} {
    if value == nil {
        log.Warnf("Empty value")
        return complexToken{token, nil}
    }

    var v []map[string]string
    currentMap := make(map[string]string)

    matches := regexpSubfield.FindAll(value, -1)

    for _, sub := range matches {
        h, l := parseLine(sub)

        if _, contains := currentMap[string(h)]; contains {
            v = append(v, currentMap)
            currentMap = make(map[string]string)
        }

        currentMap[string(h)] = string(bytes.Trim(l, stringEmpty))
    }
    v = append(v, currentMap)

    return complexToken{token, v}
}
````
The second code will be then more performant than the first one.

# Constants management
We must manage a constant value to dissociate ADEXP and ICAO messages. The bad code is doing it this way:
````
const (
    AdexpType = 0 // TODO constant
    IcaoType  = 1
)
````
Whereas the good code is a more elegant solution based on Go (elegant) iota:
````
const (
    AdexpType = iota
    IcaoType 
)
````
It produces exactly the same result but it reduces potential developer mistakes.

# Receiver functions
Each parser provides a function to determine whether a message concerns the upper level (at least one route point above the level 350).
The bad code implements it this way:
````
func IsUpperLevel(m Message) bool {
    for _, r := range m.RoutePoints {
        if r.FlightLevel > upperLevel {
            return true
        }
    }

    return false
}
````
Meaning we have to pass a Message as an input of the function.
Whereas the good code is simply a function with a Message receiver:
````
func (m *Message) IsUpperLevel() bool {
    for _, r := range m.RoutePoints {
        if r.FlightLevel > upperLevel {
            return true
        }
    }

    return false
}
````
The second approach is preferable. We simply indicate the Message struct implements a specific behavior.
It might also be a first step to using Go interfaces. For example, if someday we need to create another structure with the same behavior (IsUpperLevel()), the initial code does not even need to be refactored (as Message already implements this behavior).

# Comments
This one is pretty obvious but the bad comment is poorly commented.
On the other side, I tried to comment the good code as I would do in a real project. Even though I’m not the kind of developer who likes to comment every single line, I still believe it is important to comment at least each function and the main steps in a complex function.
As an example:
````
// Split each line in a goroutine
for _, line := range in {
    go mapLine(line, ch)
}

msg := Message{}

// Gather the goroutine results
for range in {
    // ...
}
````
One concrete example in addition of a function comment might also be very useful:
````
// Parse a line by returning the header (token name) and the value. 
// Example: -COMMENT TEST must returns COMMENT and TEST (in byte slices)
func parseLine(in []byte) ([]byte, []byte) {
    // ...
}
````
Such concrete examples can really help another developer in better understanding an existing project.
Last but not least, according to Go best practices the package itself is also commented.
````
/*
Package good is a library for parsing the ADEXP messages.
An intermediate format Message is built by the parser.
*/

package good
````

# Logging
Another obvious example is the lack of logs produced in the bad code. As I’m not a fan of the standard log package, I used an external library called logrus in this project.

# go fmt
Go provides a set of powerful tools like go fmt. Unfortunately, we forgot to apply it to the bad code whereas it was done on the good code.

# DDD
DDD brings the concept of ubiquitous language to emphasize the importance of a shared language between all the project stakeholders (business experts, dev, testers etc.).
 This cannot be really measured here in this example, but keeping a simple structure like Message compliant with the language spoken inside of a bounded context is also a good point for the overall project maintainability.

# Performance results
On an i7–7700 4x 3.60Ghz, I ran a benchmark test to compare both parsers:
* Bad implementation: 60430 ns/op
* Good implementation: 45996 ns/op
The bad code is more than 30% slower than the good one.

# Conclusion
It is pretty difficult in my opinion to give a general definition of what is a bad and a good code. A code in one context might be considered as good whereas in another context it might be considered as bad.
The first obvious characteristic of a good code is to provide a correct solution according to given functional requirements. A code can be performant if it does not fit the requirements, it is pretty useless.
Meanwhile, it is important for a developer to care about simple, maintainable and performant code.
The performance improvement does not materialize from the air, it comes with code complexity increase.
A good developer is someone able to find the right balance between these characteristics according to a given context.
Just like in DDD, context is key :)

# Further reading
[Go code refactoring: the 23x performance hunt](https://medium.com/@val_deleplace/go-code-refactoring-the-23x-performance-hunt-156746b522f7)
