+++
title = "Accept interfaces return concrete types"
description = "Outlines why you should write your libraries and packages to accept interfaces and return concrete types"
author = "Craig Brookes"
tags = [
]
date = "2016-11-02T10:30:10Z"
draft = false
+++

**Note this post was edited after taking on board feedback from [reddit discussion](https://www.reddit.com/r/golang/comments/5cos3x/accept_interfaces_return_structs/)**

I have heard mentioned several times this idea of accepting interfaces and returning concrete types. 
Here I will try to outline why I think this is good practice. It is important to note that this is intended as a principal to keep in mind when writing 
Go code, rather than an absolute rule. 

When writing libraries and packages our goal is for them to be consumed by someone. Either by our own code, but also, hopefully, by others too.
We want to make this as simple and frictionless as possible.
Accepting interfaces and returning structs can be a powerful way to achieve this. It allows the consumers of our packages to reduce the coupling between their code and yours. It helps clearly define the contract between API and the consumer, 
it makes it easier when consumers of your code are writing tests for the code that depends on your package. 
Lets look at some examples to help illustrate.

### Accepting Interfaces

Accepting a concrete type can limit the uses or our API and also cause difficulty for consumers of our code when it comes to testing. For example, if the public API of our library or package were to accept the concrete type ```*os.File```  instead 
of the ```io.Writer``` interface, it would force consumers to use the same types in order to use our API. Now if we were instead to accept an interface, it would ensure that the requirements of our API
are met, while not forcing a concrete type on the consumer.

Below is a contrived, simple example, but it is something you can often hit in real world code.

*Using a concrete type*

```go 
package myapi

import "os"

type MyWriter struct {}

func (mw *MyWriter) UpdateSomething(f *os.File) error {
	//code using the file to write ...
	return nil
}

func New() *MyWriter {
	return &MyWriterApi{}
}

```

So we mentioned how this pattern effect the consumers of our API. Lets look at how the above API would be consumed and tested:

```go 
  package myconsumer 

  import (
      "github.com/someone/myapi"
  )

  func UseMyApi(doer *myapi.MyWriter, f *os.File)error{
      //do awesome business logic
      return doer.UpdateSomething(f)
  }

```

In our application code, at first, this seems ok as long as we only need to use files with this API. The difficulty shows itself best when we implement a test. 

```go 
  package myconsumer_test 

  import (
      "github.com/someone/myconsumer"
      "os"
  )

  type mockDoer struct{}
  func (md mockDoer)UpdateSomething(f *os.File)error{
      return nil
  }

  func TestUseMyApi(t *testing.T){

      //we now need to get a concrete implementation of  *os.File somehow to use with our test.
      f,err := os.Open("/some/path/to/fixture/file.txt")
      if err != nil{
          t.Fatalf("failed to open file for test %s",err.Error())
      }
      defer f.Close()
      if err := myconsumer.UseMyApi(mockDoer{},f); err != nil{
          t.Errorf("did not expect error calling UseMyApi but got %s ", err.Error())
      }
       
  }

```

As the API implementation can only accept a concret ```*os.File``` we are now forced to use a real file in our test. 
So how dow we solve this problem and make our API better for its consumers? 

*Accepting an interface*

Back to API code:

```go

package myapi

import "io"


type MyWriter struct {}

#Swithing from an io.File to an io.Writer makes things far easer for the consumer.
func (mw *myWriter) UpdateSomething(w io.Writer) error {
	//code using the writer to write ...
	return nil
}

func New() *MyWriter {
	return &MyWriterApi{}
}

```

Now our implementation takes anything that implements the builtin ```io.Writer``` interface. Although
we are using a builtin interface here, in code, within specific business domains, this could well be a custom interface 
expressing the concerns of your domain. So how does this change impact our test code?

```go 
  package myconsumer_test 

  import (
      "github.com/someone/myconsumer"
      "io"
      "bytes"
  )

  type mockDoer struct{}
  func (md mockDoer)UpdateSomething(w io.Writer)error{
      return nil
  }

  func TestUseMyApi(t *testing.T){

      //we no longer need an actual file, all we need is something that 
      //implements the write method from io.Writer. bytes.Buffer is one such type.
      var b bytes.Buffer
    
      if err := myconsumer.UseMyApi(mockDoer{},&b); err != nil{
          t.Errorf("did not expect error calling UseMyApi but got %s ", err.Error())
      }
       
  }

```

Another advantage here is that if our UseMyApi function, also writes using the writer, we can assert based on the contents of the bytes.Buffer.

#### Reducing our footprint and coupling

Our libraries may use other libraries for some of its functionality. In our public API we should avoid exposing third party types to the consumers of our API. If our public API exposes a 3rd party type, then our consumers will also need to import and use that third party type. This couples there code to a dependency of your code and means they need to know too much about how the innards of your API works. It is a leaky abstraction. To avoid this we should either define our own types that internally can be translated to the required type or define and accept an interface.

### Returning Concrete Types

So what are the advantages of returning concrete types? If we want to accept interfaces, why would we not also return interfaces?

Navigating to the implementation code of a returned type, is something that you will often do when using a library even if it is only to get a better understanding of how something works. Returning an interface makes this far more difficult for the consumer as they will first navigate to an interface definition and then need to spend additional time trying to find the implementation code.
Consumers of our code, my only be interested in a small subset of the functionality, while returning an interface doesn't stop them from defining a new interface, nor does returning a concrete type. So given the uneeded indirection a returned interface will cause, it makes more sense and reduces friction to return a concrete type.
