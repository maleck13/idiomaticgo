+++
date = "2016-10-16T10:20:44+01:00"
title = "Idiomatic Go Tests"
draft = false

+++

What does it mean to write idiomatic go tests? I will outline what it means to me and try to convey through details and examples what I wish I had known when starting out with Go.

### Idiomatic go tests are written using the builtin testing package

Go comes with a builtin [testing](https://golang.org/pkg/testing/) package. There is no need for an external testing package. Although these do exist and it is not for me to say you shouldn’t use them. However I would encourage writing tests using the builtin testing package, and only when you begin to feel it is failing you somehow, look at what external packages can offer.


Tests in a Go program are normally defined along with the source code for the package. Notice below that the test file is named ```*_test.go``` Go expects your tests to be named this way.

```
myPkg
├── repo.go
├── repo_test.go
```

The testing package is built to be used with the go test command. It looks for tests that are defined in the following way:

```
func TestXxx(*testing.T)
```

Lets look at a very simple example the first file is sum.go and the second file sum_test.go:

``` sum.go ```

```
package myPkg  

Sum(a,b int)int{
    return a + b
}
```

Next we have our test for this package. Things to notice here include the package name and the test name:

``` sum_test.go ```


```
package myPkg_test //notice we name it _test this allows us to work only with the exported api of our package.

import "testing"
import "github.com/maleck13/myPkg"

func TestSum(t *testing.T){
    if 3 != myPkg.Sum(1,2){
        t.Fatalf("expected the sum to equal %d",3)
    } 
}

```


Somethings worth pointing out:

- in our test the package name is ```package_test``` this is a convention in Go. It allows the test file to be in the same package as the source code, but only have access to the exported types, and functions etc.

- Notice that in our test, we do not have an inbuilt assert. Read the code again, notice that there is very little different in how our test code reads compared with our business logic code, it is just more Go code. This is intentional. It may mean you write a few more lines of code, but has the advantage of allowing anyone who knows Go to be able to read and modify your test code without the overhead of learning a testing DSL.

### Running Tests

You can run tests using the builtin ```go test``` command. 
There are some very useful features in this command. To get package level coverage reports run ``` go test -cover ```
To test for race conditions you can add the ```-race``` flag. 
To run a single test you can use the ```-run``` flag that takes a regular expression. For example ```go test -run ^TestSum$```.
Using ```go test -h ``` will show you the wide set of options available.

### Mocks and Stubs

Mocks are used frequently in Go tests. The most common way you will see a mock used is to provide a different implementation of a required interface. Mocks can be stored anywhere in your test code, but often they would be stored under a mock or test package. Lets look at an example of mocking in Go.


```
package mything

type Doer interface{
    Do()error 
}

func BusinessDoer (d Doer)error{
    //some business logic 
    return d.Do()
}

```

And now our test file:


```
package mything_test 

import mything

//This mock could be in a separate file.

type mockDoer struct{
    Error error 
}
// implment our Doer iterface
func(md mockDoer)Do()error{
    return md.Error
}

func TestBusinessDoer(t *testing.T){
    doer := mockDoer{}
    if err := mything.BusinessDoer(doer); err != nil{
        t.Fatalf("did not expect an error but got %s",err.Error())
    }
}
```

So in our test we have swapped out the original implementation of the Doer interface for our own implementation that we can easily control the behaviour of. It is important that if you want to unit test your code, that you express the dependencies used by that piece of code as an interface as this will allow for simple mocking. It is also a good reason to aim to keep your interfaces small. The smaller the interface the less work it is to mock out.


### Table driven tests

In the above example we defined a test that tested some business logic for one case where the business logic does not return an error. Normally we would also want to at least test one other case, the case where the the dependency does return an error. Now we could define a new test entirely for this, however often in Go code you will see an approach for this called table testing. Lets look at an example that expands on the first example.

```
package mything

type Doer interface{
    Do()error 
}

func BusinessDoer (d Doer)error{
    //some business logic 
    return d.Do()
}
```

And now our test file:

```
package mything_test 

import mything

//This mock could be in a separate file.

type mockDoer struct{
    Error error 
}
// implment our Doer iterface
func(md mockDoer)Do()error{
    return md.Error
}

func TestBusinessDoer(t *testing.T){
    cases := []struct{
        Name string
        ExpectError bool 
        Error error
    }{
        {
            Name:"test does business logic",
            ExpectError : false,
            Error : nil,
        },
        {
            Name:"test fails when dependency errors",
            ExpectError : true,
            Error : errors.New("an error"),
        },
    }

    for _,td := range cases{
        t.Run(td.Name, func (t *testing.T){
            doer := mockDoer{Error: td.Error}
            err := mything.BusinessDoer(doer) 
            if td.ExpectError && err == nil{
                t.Fatalf("expected an error but got none")
            } 
            if ! td.ExpectError && err != nil{
                t.Fatalf("did not expect an error but got one %s ", err.Error())
            }
        })
    }
}
```

So now we are defining a set of test cases in a slice. We are defining some inputs and expected behaviour. Then we are ranging over each of these tests cases and running them in a separate sub test. Sub tests allow us to re run a single failing test case rather than them all. In the above case if you wanted to run the sub test you would do so by running the following command from within the package.

```
go test -run TestBusinessDoer/test_fails_when_dependency_errors
```