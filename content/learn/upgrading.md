+++
title = "Upgrading from Goa v1 or Goa v2 to v3"
weight = 3

[menu.main]
name = "Upgrading"
parent = "learn"
+++

# Upgrading from v2 to v3

v2 and v3 are functionally equivalent making the upgrade pretty straightforward. v3 requires Go module support
and therefore Go 1.11 or higher. Upgrading from v2 to v3 is as simple as:

* Enabling Go modules on your project (env GO111MODULE=on go mod init)
* Updating the import path of the goa package to goa.design/goa/v3/pkg
* Updating the import path of Goa package X from goa.design/goa/X to goa.design/goa/v3/X

That's it! Note also that the `goa` tool in v3 is backwards compatible and is
able to generate code for v2 design. This makes it possible to work concurrently
on both v2 and v3 projects by keeping v2 in the GOPATH and using v3 as a Go
module.

# Upgrading from v1 to v2 or v3

Goa v2 and v3 bring a host of new features and improvements over v1, most notably:

* a modular architecture with clear separation of concerns between transport and business logic
* support for gRPC
* fewer external dependencies
* more powerful plugin system including [Go kit](https://gokit.io) support

Some of the changes are fairly fundamental and affect how to design services
however the basic principles and value proposition remain identical:

* a single source of truth provided by a Go based design DSL
* a code generator tool that given the DSL generates documentation, server and
  client code

This document describes the changes and provides some guidelines on how to
upgrade.

>Note: Goa v2 and v3 are functionally equivalent, the only difference being that v3
>leverages and supports Go modules while v2 does not. The rest of this document refers
>to v3 but is applicable to both.

## Changes to the DSL

Goa v3 promote a clean separation of layers by making it possible to design the service APIs
independently of the underlying transport. Transport specific DSL then makes it possible to provide
mappings for each transport (HTTP and gRPC). So instead of `Resources` and `Actions` the DSL focuses
on `Services` and `Methods`. Each method describes its input and output types. Transport specific
DSL then describes how the input types are built from HTTP request or inbound gRPC messages and how
output types should be written to HTTP responses or outbound gRPC messages.

> NOTE: The v3 DSL is documented extensively in [godoc](https://godoc.org/goa.design/goa/dsl)

### Types

For the most part the DSL used to describes types remains the same with a few differences:

* `MediaType` is now [ResultType](https://godoc.org/goa.design/goa/dsl#ResultType) to make it clear
  that types described using this DSL are intended to be used as method results. Note that standard
  types defined using the [Type](https://godoc.org/goa.design/goa/dsl#Type) DSL can also be used as
  result types.
* Result types may omit defining views. If a result type does not define a view then a default
  view is automatically defined that lists all the result type attributes.
* The new [Field](https://godoc.org/goa.design/goa/dsl#Field) DSL is identical to
  [Attribute](https://godoc.org/goa.design/goa/dsl#Attribute) but makes it possible to specify an
  index for the field corresponding to the gRPC field number.
* `HashOf` is now [MapOf](https://godoc.org/goa.design/goa/dsl#MapOf) which is more intuitive to Go
  developers.
* There are new primitive types to describe more precisely the binary layout of the data:
  [Int](https://godoc.org/goa.design/goa/dsl#Int),
  [Int32](https://godoc.org/goa.design/goa/dsl#Int32),
  [Int64](https://godoc.org/goa.design/goa/dsl#Int64),
  [UInt](https://godoc.org/goa.design/goa/dsl#UInt),
  [UInt32](https://godoc.org/goa.design/goa/dsl#UInt32),
  [UInt64](https://godoc.org/goa.design/goa/dsl#UInt64),
  [Float32](https://godoc.org/goa.design/goa/dsl#Float32),
  [Float64](https://godoc.org/goa.design/goa/dsl#Float64)
  and [Bytes](https://godoc.org/goa.design/goa/dsl#Bytes).
* The types `DateTime` and `UUID` are deprecated in favor of
  [String](https://godoc.org/goa.design/goa/dsl#String) and corresponding
  [Format](https://godoc.org/goa.design/goa/dsl#Format)
  validations.

#### Example

v1 media type:

```go
var Person = MediaType("application/vnd.goa.person", func() {
	Description("A person")
	Attributes(func() {
		Attribute("name", String, "Name of person")
		Attribute("age", Integer, "Age of person")
		Required("name")
	})
	View("default", func() {  // View defines a rendering of the media type.
		Attribute("name") // Media types may have multiple views and must
		Attribute("age")  // have a "default" view.
	})
})
```

Corresponding v3 result type:

```go
var Person = ResultType("application/vnd.goa.person", func() {
	Description("A person")
	Attributes(func() {
		Attribute("name", String, "Name of person")
		Attribute("age", Int, "Age of person")
		Required("name")
	})
})
```

Corresponding v3 result type with explicit field indeces:

```go
var Person = ResultType("application/vnd.goa.person", func() {
	Description("A person")
	Attributes(func() {
		Field(1, "name", String, "Name of person")
		Field(2, "age", Int, "Age of person")
		Required("name")
	})
})
```

### API

The following changes have been made to the [API](https://godoc.org/goa.design/goa/dsl#API) DSL:

* The `Host`, `Scheme` and `BasePath` DSLs are replaced with [Server](https://godoc.org/goa.design/goa/dsl#Server).
* The [Server](https://godoc.org/goa.design/goa/dsl#Server) DSL makes it possible to define server properties for different environments. Each server may list the services it hosts making it possible to define multiple servers in one design.
* `Origin` is now implemented as part of the [CORS plugin](https://github.com/goadesign/plugins/tree/master/cors).
* `ResponseTemplate` and `Trait` have been deprecated.

#### Example

v1 API section:

```go
var _ = API("cellar", func() {
	Title("Cellar Service")
	Description("HTTP service for managing your wine cellar")
	Scheme("http")
	Host("localhost:8080")
	BasePath("/cellar")
})
```

Corresponding v3 section:

```go
var _ = API("cellar", func() {
	Title("Cellar Service")
	Description("HTTP service for managing your wine cellar")
	Server("app", func() {
		Host("localhost", func() {
			URI("http://localhost:8080/cellar")
		})
	})
})
```

Corresponding v3 section using multiple servers:

```go
var _ = API("cellar", func() {
	Title("Cellar Service")
	Description("HTTP service for managing your wine cellar")
	Server("app", func() {
		Description("App server hosts the storage and sommelier services.")
		Services("sommelier", "storage")
		Host("localhost", func() {
			Description("default host")
			URI("http://localhost:8080/cellar")
		})
	})
	Server("swagger", func() {
		Description("Swagger server hosts the service OpenAPI specification.")
		Services("swagger")
		Host("localhost", func() {
			Description("default host")
			URI("http://localhost:8088/swagger")
		})
	})
})
```

### Services

The `Resource` function is now called [Service](https://godoc.org/goa.design/goa/dsl#Service). The
DSL is now organized into a transport agnostic section and transport specific DSLs. The transport
agnostic section lists the potential errors returned by all the service methods. The transport
specific sections then map these errors to HTTP status code or gRPC response codes.

* `BasePath` is now called [Path](https://godoc.org/goa.design/goa/dsl#Path) and appears in the
  [HTTP](https://godoc.org/goa.design/goa/dsl#HTTP) DSL.
* `CanonicalActionName` is now called [CanonicalMethod](https://godoc.org/goa.design/goa/dsl#CanonicalMethod)
  and appears in the [HTTP](https://godoc.org/goa.design/goa/dsl#HTTP) DSL.
* `Response` is replaced with [Error](https://godoc.org/goa.design/goa/dsl#Error).
* `Origin` is now implemented as part of the [CORS plugin](https://github.com/goadesign/plugins/tree/master/cors).
* `DefaultMedia` is deprecated.

#### Example

v1 design:

```go
	Resource("bottle", func() {
		Description("A wine bottle")
		BasePath("/bottles")
		Parent("account")
		CanonicalActionName("get")

		Response(Unauthorized, ErrorMedia)
		Response(BadRequest, ErrorMedia)
		// ... Actions
	})
```

Equivalent v3 design:

```go
	Service("bottle", func() {
		Description("A wine bottle")
		Error("Unauthorized")
		Error("BadRequest")

		HTTP(func() {
			Path("/bottles")
			Parent("account")
			CanonicalMethod("get")
		})
		// ... Methods
	})
```

### Methods

The `Action` function is replaced by [Method](https://godoc.org/goa.design/goa/dsl#Method). As with
services the DSL is organized into a transport agnostic section and transport specific DSLs. The
transport agnostic section defines the payload and result types as well as all the possible method
specific errors not already defined at the service level. The transport specific DSLs then map the
payload and result type attributes to transport specific constructs such as HTTP headers, body etc.

* Most of the DSL present in v1 is HTTP specific and thus moved to the [HTTP](https://godoc.org/goa.design/goa/dsl#HTTP) DSL.
* The [Param](https://godoc.org/goa.design/goa/dsl#Param) and [Header](https://godoc.org/goa.design/goa/dsl#Header) functions
  now need only list the names of attributes of the corresponding method payload or result types.
* Error responses now use the [Error](https://godoc.org/goa.design/goa/dsl#Error) DSL.
* HTTP path parameters are now defined using curly braces instead of colons: `/foo/{id}` instead of `/foo/:id`.

The mapping of input and output types

v1 action design example:

```go
	Action("update", func() {
		Description("Change account name")
		Routing(
			PUT("/:accountID"),
		)
		Params(func() {
			Param("accountID", Integer, "Account ID")
		})
		Payload(func() {
			Attribute("name", String, "Account name")
			Required("name")
		})
		Response(NoContent)
		Response(NotFound)
		Response(BadRequest, ErrorMedia)
	})
```

Equivalent v3 design:

```go
	Method("update", func() {
		Description("Change account name")
		Payload(func() {
			Attribute("accountID", Int, "Account ID")
			Attribute("name", String, "Account name")
			Required("name")
		})
		Result(Empty)
		Error("NotFound")
		Error("BadRequest")

		HTTP(func() {
			PUT("/{accountID}")
		})
	})
```
