---
author: Carlos Mendible
categories:
- azure
date: "2023-03-26T10:00:00Z"
description: Azure Function HTTP Trigger with Golang
images: ["/assets/img/posts/azurefunctions.jpg"]
published: true
tags: ["golang", "azure functions", "serverless"]
title: Azure Function HTTP Trigger with Golang
---

Back in 2017 I wrote a post about how to [run a precompiled .NET Core Azure Function in a container]({{< ref "/posts/2017/2017-12-28-run-a-precomplied-net-core-azure-function-in-a-container.md" >}}). Fast forward to 2023 and, as some of you know, I've been playing with [Golang](https://golang.org/) for a while now so I thought it was about time to translate the .NET code and make it work with Golang.

**Prerequisites**:

* [Azure Functions Core Tools](https://learn.microsoft.com/en-us/azure/azure-functions/functions-run-local?tabs=v4%2Cwindows%2Ccsharp%2Cportal%2Cbash)
* [Golang](https://golang.org/)

## Create an Azure Function with a custom worker runtime

Create an Azure Function with a custom worker runtime

``` bash
mkdir dni
cd dni
func init --worker-runtime custom
```

## Edit the host.json file

Modify the `defaultExecutablePath` to the name of the executable file you want to run. In this case: **handler**. Also, set the `enableForwardingHttpRequest` to `true` so the function can access the HTTP request:

``` json
{
  "version": "2.0",
  "logging": {
    "applicationInsights": {
      "samplingSettings": {
        "isEnabled": true,
        "excludedTypes": "Request"
      }
    }
  },
  "extensionBundle": {
    "id": "Microsoft.Azure.Functions.ExtensionBundle",
    "version": "[2.*, 3.0.0)"
  },
  "customHandler": {
    "description": {
      "defaultExecutablePath": "handlers",
      "workingDirectory": "",
      "arguments": []
    },
    "enableForwardingHttpRequest": true
  }
}
``` 

## Create a new HTTP Trigger Azure Function

Create a new HTTP Trigger Azure Function

``` bash
func new -n dni -t httptrigger
```

## Create a golang handler for the function.

Create a `handlers.go` file with the following contents:

``` go
package main

import (
	"encoding/json"
	"fmt"
	"log"
	"net/http"
	"os"
	"strconv"
	"strings"
)

type InvokeRequest struct {
	Data     map[string]json.RawMessage
	Metadata map[string]interface{}
}

type InvokeResponse struct {
	Outputs     map[string]interface{}
	Logs        []string
	ReturnValue interface{}
}

func dniHandler(w http.ResponseWriter, r *http.Request) {
	ua := r.Header.Get("User-Agent")
	fmt.Printf("user agent is: %s \n", ua)
	invocationid := r.Header.Get("X-Azure-Functions-InvocationId")
	fmt.Printf("invocationid is: %s \n", invocationid)

	queryParams := r.URL.Query()

	if dni := queryParams["dni"]; dni != nil {
		valid := validateDNI(dni[0])
		js, err := json.Marshal(valid)
		if err != nil {
			http.Error(w, err.Error(), http.StatusInternalServerError)
			return
		}

		w.Header().Set("Content-Type", "application/json")
		w.Write(js)
	} else {
		http.Error(w, "dni query parameter not present", http.StatusInternalServerError)
	}
}

func validateDNI(dni string) bool {
	table := "TRWAGMYFPDXBNJZSQVHLCKE"
	foreignerDigits := map[string]string{
		"X": "0",
		"Y": "1",
		"Z": "2",
	}
	parsedDNI := strings.ToUpper(dni)
	if len(parsedDNI) == 9 {
		checkDigit := parsedDNI[8]
		parsedDNI = parsedDNI[:8]
		if foreignerDigits[strings.ToUpper(string(parsedDNI[0]))] != "" {
			parsedDNI = strings.Replace(parsedDNI, string(parsedDNI[0]), foreignerDigits[string(parsedDNI[0])], 1)
		}
		dniNumbers, err := strconv.Atoi(parsedDNI)
		if err != nil {
			fmt.Println("Error during conversion")
		}

		return table[dniNumbers%23] == checkDigit
	}
	return false
}

func main() {
	customHandlerPort, exists := os.LookupEnv("FUNCTIONS_CUSTOMHANDLER_PORT")
	if !exists {
		customHandlerPort = "8080"
	}
	mux := http.NewServeMux()
	mux.HandleFunc("/api/dni", dniHandler)
	fmt.Println("Go server Listening on: ", customHandlerPort)
	log.Fatal(http.ListenAndServe(":"+customHandlerPort, mux))
}
```

> **Note**: THe code intends to validate a Spanish DNI (Documento Nacional de Identidad) number. It gets the DNI number from the query string and returns a boolean value indicating if the DNI number is valid or not.

Build the executable:

``` bash
go build ./handlers.go
```

## Test the Azure Function locally

To test the Azure Function locally run the following command:

``` bash
func start --verbose
```

and from another terminal window run the following command:

``` bash
curl -i http://localhost:7071/api/dni?dni=59658914L
```

You should get a response similar to the following:

``` bash
HTTP/1.1 200 OK
Content-Length: 4
Content-Type: application/json
Date: Sun, 26 Mar 2023 11:01:24 GMT
Server: Kestrel

true
```	

Download all code and files [here](https://github.com/cmendible/azure.samples/tree/main/function_golang/src/dni).

Hope it helps!