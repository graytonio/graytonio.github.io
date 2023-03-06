---
title: "Creating Unit Tests in Go"
date: 2023-03-06
categories: [Programming, Golang]
tags: [programming, golang, devops, testing, unittests]
---

Writing unit tests for you functions is an important step in ensuring that the code you have written is creating the output you are expecting.  I'll go over a few different functions in go and how you can go about writing good unit tests for them using go recommended table based testing method.

## Simple Pure Functions

For pure functions unit testing is a breeze due to pure functions properties of never modifying or accessing data outside of its scope. Lets look at an example.

```go
func reverse(s string) (result string) {
    for _, v := range str {
        result = string(v) + result
    }
    return result
}
```

This function only acts on its inputs and has no other side effects so lets setup a test function for it. The convention for tests is to create a new file next to the source code with the suffix `_test.go` for example if this code is in `mylib.go` the test file would be named `mylib_test.go`.  Test files act just like normal go code and can exist in the same package space. We use go's `testing` package to handle the state of our tests.

```go
package mylib

import "testing"

func TestReverse(t *testing.T) {

}
```

In this `TestReverse` function we can define how we want our tests structured.  We should know what input we are putting in as well as what we expect to get out of the function given the input. We will also want to hold a name or id for our test so we can identify which inputs are not giving the correct output. We'll create a quick struct to hold this information for this test.

```go
type test struct {
    name string
    inputString string
    expectedOutput string
}
```

Not that we have a format for how we want to define our tests we can setup a couple of test cases.

```go
var tests = []test{
    {
        name: "basic string",
        inputString: "hello",
        expectedOutput: "olleh",
    },
    {
        name: "empty string",
        inputString: "",
        expectedOutput: "",
    },
    {
        name: "single character",
        inputString: "a",
        expectedOutput: "a",
    }
}
```

With these test cases we can loop over them and make sure that our cases all return correctly.  To do our comparison we are going to use the `testify` library which adds assert functions missing from the go std.

```go
for _, tt := range tests {
    t.Run(tt.name, func (t *testing.T) {
        output := reverse(tt.inputString)
        assert.Equal(t, output, tt.expectedOutput, "output does not match expectedOutput")
    }
}
```

Now when we run `go test ./...` to run all the tests in our project each of the test cases will be ran and checked against the expected output.  If any of the tests fail we will get an output about what output it got against what it should have been which can help us figure out where the bug is. All together our test file will look like.

```go
package mylib

import (
    "testing"

    "github.com/stretchr/testify/assert"
)

func TestReverse(t *testing.T) {
    type test struct {
        name string
        inputString string
        expectedOutput string
    }

    var tests = []test{
        {
            name: "basic string",
            inputString: "hello",
            expectedOutput: "olleh",
        },
        {
            name: "empty string",
            inputString: "",
            expectedOutput: "",
        },
        {
            name: "single character",
            inputString: "a",
            expectedOutput: "a",
        }
    }

    for _, tt := range tests {
        t.Run(tt.name, func (t *testing.T) {
            output := reverse(tt.inputString)
            assert.Equal(t, output, tt.expectedOutput, "output does not match expectedOutput")
        }
    }
}
```

This is a good example for simple functions with no side effects but it can become more difficult to test functions that have to do things like make http requests or reach out to databases.

## More Complex Functions and Mocking

For functions with more complex behavior we may have to mock certain interfaces. In this example we'll look at a function that makes an api request using go's builtin http client. 

```go
type MyData struct {
    Foo string `json:"foo"`
    Bar int `json:"bar"`
}

var (
    ErrUserNotFound = errors.New("user not found"),
    ErrServerError = errros.New("server error"),
)

func GetExampleData(userId string) (*MyData, error) {
    resp, err := http.Get(fmt.Sprintf("http://example.com/api/%s", userId))
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()

    body, err := io.ReadAll(resp.Body)
    if err != nil {
        return nil, err
    }

    // If request had error return relevant one
    switch resp.StatusCode {
        case 404:
            return nil, ErrUserNotFound
        case 500:
            return nil, ErrServerError
    }

    var results MyData
    err = json.Unmashall(body, &results)
    if err != nil {
        return nil, err
    }

    return &results, nil
}
```

This is a much more complicated function that not only has input from the function parameters but is pulling in other data from a server as a result. We could test this function in the same way but that would mean making real requests to the server with real data each time we wanted to test which is not ideal. 

To do the testing without actually making the requests each time we will use a package called `gock`.  Gock intercepts http requests made during testing and returns fake data which we can define to match the data we are receiving from the server.  This way we can give our function data in the same shape as the server would without having to actually make the requests.

To start our tests we want to define our fake routes with `gock`.

```go

func TestGetExampleData(t *testing.T) {
    defer gock.Off() // Ensures gock will not intercept routes after we are done testing

    gock.New("http://example.com").                                  // Base URL we want to intercept
        Get("/api/good-user").                                       // Path we want to match
        Reply(200).                                                  // Status Code to Return
        JSON(map[string]interface{}{"foo": "test-val-0", "bar": 23}) // Mock data to return
}
```

This route that we have setup will intercept a http request for `http://example.com/api/good-user` and instead of actually making the request simply return the JSON data as the "response".  We can do this for every test case we want to create

```go
gock.New("http://example.com").
    Get("/api/missing-user").
    Reply(404).
    JSON(map[string]interface{}{"error": "User missing-user not found"})

gock.New("http://example.com").
    Get("/api/server-error").
    Reply(500).
    JSON(map[string]interface{}{"error": "Server ran into an error while trying to process your request"})
```

With these routes we can setup our test struct and loop just as before. Asserting our actuall outputs match what we expect.

```go
type test struct {
    name string
    inputUserId string
    expectedOutput MyData
    expectedError error
}

var tests = []test{
    {
        name: "good response",
        inputUserId: "good-user",
        expectedOutput: MyData{ Foo: "test-val-0", Bar: 23 },
        expectedError: nil
    },
    {
        name: "user missing",
        inputUserId: "missing-user",
        expectedOutput: MyData{},
        expectedError: ErrUserNotFound
    },
    {
        name: "server error",
        inputUserId: "server-error",
        expectedOutput: MyData{},
        expectedError: ErrServerError,
    },
}

for _, tt := range tests {
    t.Run(tt.name, func(t *testing.T) {
        data, err := GetExampleData(tt.inputUserId)

        // If we are supposed to get an error check we got the right one then exit the test
        if tt.expectedError != nil {
            assert.ErrorIs(t, err, tt.expectedError)
            return
        } else {
            // If we were not expecting an error make sure we didn't get one
            assert.NoError(t, err)
        }

        // Make sure the data we got from our function is the data we should have gotten
        assert.Equal(t, data, tt.expectedOutput)
    })
}
```

Running this test will get the test data from our gock routes and parse them as if they were gotten from the real server without having to modify our functions to accept some mock client.  For even more complex functions you may want to do more checking of the output but this should give you a good framework for how to structure tests in your go project.

## Resources

**Testify** - [https://github.com/stretchr/testify](https://github.com/stretchr/testify)

**Gock** - [https://github.com/h2non/gock](https://github.com/h2non/gock)

