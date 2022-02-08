# Tutorial: Developing a RESTful API with Go, JSON Schema validation and OpenAPI docs 

This tutorial continues [Developing a RESTful API with Go and Gin](https://go.dev/doc/tutorial/web-service-gin) featured in Go documentation. Please check it first.

_**TL;DR** We're going to replace [`gin-gonic/gin`](https://github.com/gin-gonic/gin) with [`swaggest/rest`](https://github.com/swaggest/rest) to obtain type-safe [OpenAPI](https://swagger.io/) spec with [Swagger UI](https://swagger.io/tools/swagger-ui/) and [JSON Schema](https://json-schema.org/) request validation._

Providing reliable and accurate documentation becomes increasingly important thanks to growing integrations between the services. Whether those integrations are between your own microservices, or you are serving an API to 3rd party.

[OpenAPI v3](https://swagger.io/specification/) is currently a dominating standard to describe REST API in machine-readable format. There is a whole ecosystem of tools for variety of platforms and languages that help automating integrations and documentation using OpenAPI schema. For example, 3rd party can generate SDK from schema to use your API.

## Prerequisites

Let's start with the result of [previous tutorial](https://go.dev/doc/tutorial/web-service-gin).

```go
package main

import (
	"net/http"

	"github.com/gin-gonic/gin"
)

// album represents data about a record album.
type album struct {
	ID     string  `json:"id"`
	Title  string  `json:"title"`
	Artist string  `json:"artist"`
	Price  float64 `json:"price"`
}

// albums slice to seed record album data.
var albums = []album{
	{ID: "1", Title: "Blue Train", Artist: "John Coltrane", Price: 56.99},
	{ID: "2", Title: "Jeru", Artist: "Gerry Mulligan", Price: 17.99},
	{ID: "3", Title: "Sarah Vaughan and Clifford Brown", Artist: "Sarah Vaughan", Price: 39.99},
}

func main() {
	router := gin.Default()
	router.GET("/albums", getAlbums)
	router.GET("/albums/:id", getAlbumByID)
	router.POST("/albums", postAlbums)

	router.Run("localhost:8080")
}

// getAlbums responds with the list of all albums as JSON.
func getAlbums(c *gin.Context) {
	c.JSON(http.StatusOK, albums)
}

// postAlbums adds an album from JSON received in the request body.
func postAlbums(c *gin.Context) {
	var newAlbum album

	// Call BindJSON to bind the received JSON to
	// newAlbum.
	if err := c.BindJSON(&newAlbum); err != nil {
		return
	}

	// Add the new album to the slice.
	albums = append(albums, newAlbum)
	c.JSON(http.StatusCreated, newAlbum)
}

// getAlbumByID locates the album whose ID value matches the id
// parameter sent by the client, then returns that album as a response.
func getAlbumByID(c *gin.Context) {
	id := c.Param("id")

	// Loop through the list of albums, looking for
	// an album whose ID value matches the parameter.
	for _, a := range albums {
		if a.ID == id {
			c.JSON(http.StatusOK, a)
			return
		}
	}
	c.JSON(http.StatusNotFound, gin.H{"message": "album not found"})
}
```

## Initialize web service

Let's update `main` function to use a [web service](https://pkg.go.dev/github.com/swaggest/rest@v0.2.20/web#Service) for our use case interactors, it will be capable of collecting automated documentation and applying request validation.

Also we can provide basic information about our API using type-safe OpenAPI bindings.

```go
func main() {
	service := web.DefaultService()

	service.OpenAPI.Info.Title = "Albums API"
	service.OpenAPI.Info.WithDescription("This service provides API to manage albums.")
	service.OpenAPI.Info.Version = "v1.0.0"
```

Add `web` to imports.

```go
	"github.com/swaggest/rest/web"
```

## Upgrade a handler to return all items

In order to express more information about our http handler, we need refactor it to a [use case interactor](https://pkg.go.dev/github.com/swaggest/usecase#Interactor).

The constructor [`usecase.NewIOI`](https://pkg.go.dev/github.com/swaggest/usecase#NewIOI) takes three arguments, a sample of input, a sample of output and a function that should be called for them.

When web service receives request it will determine correct use case based on route and will prepare instances of input and output for further interaction (call of a function).

Input instance will be filled with data from http request, output instance will be created as a pointer to new output value.

In this case we don't need any request parameters, so input sample can be `nil`.

This action will provide a list of albums. So the output sample would be `[]album{}` (or you can use `new([]album)` too).

Then in the interact function we need to assert the type of output to update the value in it.

```go
func getAlbums() usecase.Interactor {
	u := usecase.NewIOI(nil, []album{}, func(ctx context.Context, _, output interface{}) error {
		out := output.(*[]album)
		*out = albums
		return nil
	})
	u.SetTags("Album")

	return u
}
```

Input and output samples are most important for automated http mapping and documentation generation. You can provide more information for the use case, for example tags to group multiple use cases together.

Now we can add upgraded use case to web service (in `main` function).

```go
	service.Get("/albums", getAlbums())
```

## Upgrade a handler to create new item

In this case we receive input as a JSON payload of `album`, so input sample would be `album{}`.

We also return received album in response, so the output would be `album{}` as well.

Mind different types in in/out type assertions. Output instance is provided as a placeholder for data, so it has to be a pointer. In contrast, input is not used after interact function is invoked, so it can be a non-pointer value.

```go
func postAlbums() usecase.Interactor {
	u := usecase.NewIOI(album{}, album{}, func(ctx context.Context, input, output interface{}) error {
		in := input.(album)
		out := output.(*album)

		// Add the new album to the slice.
		albums = append(albums, in)

		*out = in
		return nil
	})
	u.SetTags("Album")

	return u
}
```

Let's implement additional logic in this use case, to restrict `id` duplicates in the `albums`. In such case we can return conflict error.

```go
func postAlbums() usecase.Interactor {
	u := usecase.NewIOI(album{}, album{}, func(ctx context.Context, input, output interface{}) error {
		in := input.(album)
		out := output.(*album)

		// Check if id is unique.
		for _, a := range albums {
			if a.ID == in.ID {
				return status.AlreadyExists
			}
		}

		// Add the new album to the slice.
		albums = append(albums, in)

		*out = in
		return nil
	})
	u.SetTags("Album")
	u.SetExpectedErrors(status.AlreadyExists)

	return u
}
```

As you can see, we've also added `u.SetExpectedErrors(status.AlreadyExists)` to inform documentation collector that this use case may fail in a particular way.

Now we can add the upgraded use case to web service (in `main` function).

```go
	service.Post("/albums", postAlbums(), nethttp.SuccessStatus(http.StatusCreated))
```

Mind the additional option that changes successful status from default `http.StatusOK` to `http.StatusCreated`. This fine control is left outside of use case definition because it is specific to http, use case interactor can potentially be used with other transports (see [Clean Architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html) for more details on this concept).

## Add validation to `album` structure

Now let's add some validation rules to our `album` structure.

```go
// album represents data about a record album.
type album struct {
	ID     string  `json:"id" required:"true" minLength:"1" description:"ID is a unique string that determines album."`
	Title  string  `json:"title" required:"true" minLength:"1" description:"Title of the album."`
	Artist string  `json:"artist,omitempty" description:"Album author, can be empty for multi-artist compilations."`
	Price  float64 `json:"price" minimum:"0" description:"Price in USD."`
}
```

[Validation rules](https://pkg.go.dev/github.com/swaggest/jsonschema-go#Reflector.Reflect) can be added with field tags (or [special interfaces](https://pkg.go.dev/github.com/swaggest/jsonschema-go#Preparer)). Along with validation rules you can supply brief descriptions of field values.

* `ID` is a required field that can not be empty,
* `Title` as well,
* `Artist` is an optional field,
* `Price` can't be negative.

Validation is powered by JSON Schema.

## Upgrade a handler to return specific item

In this case we need to read request parameter from URL path. For that our input structure should contain a field with `path` tag to enable [data mapping](https://github.com/swaggest/rest#request-decoder).

Given this use case can end up with `Not Found` status, we add the status to expected errors for documentation. We also wrap the error in use case body to have a correct http status.

```go
func getAlbumByID() usecase.Interactor {
	type getAlbumByIDInput struct {
		ID string `path:"id"`
	}

	u := usecase.NewIOI(getAlbumByIDInput{}, album{}, func(ctx context.Context, input, output interface{}) error {
		in := input.(getAlbumByIDInput)
		out := output.(*album)

		for _, album := range albums {
			if album.ID == in.ID {
				*out = album
				return nil
			}
		}
		return status.Wrap(errors.New("album not found"), status.NotFound)
	})
	u.SetTags("Album")
	u.SetExpectedErrors(status.NotFound)

	return u
}

```

Now we can add this use case to web service (in `main` function).

```go
	service.Get("/albums/{id}", getAlbumByID())
```

Mind the path placeholder has changed from `:id` to `{id}` to comply with OpenAPI standard.

## Mount Swagger UI

You can add a web interface to the API with Swagger UI.
```go
	service.Docs("/docs", v4emb.New)
```

Add [`v4emb`](https://pkg.go.dev/github.com/swaggest/swgui/v4emb) to imports.

```go
	"github.com/swaggest/swgui/v4emb"
```

Then documentation will be served at http://localhost:8080/docs.

## Resulting program

```go
package main

import (
	"context"
	"errors"
	"log"
	"net/http"

	"github.com/swaggest/rest/nethttp"
	"github.com/swaggest/rest/web"
	"github.com/swaggest/swgui/v4emb"
	"github.com/swaggest/usecase"
	"github.com/swaggest/usecase/status"
)

// album represents data about a record album.
type album struct {
	ID     string  `json:"id" required:"true" minLength:"1" description:"ID is a unique string that determines album."`
	Title  string  `json:"title" required:"true" description:"Title of the album."`
	Artist string  `json:"artist,omitempty" description:"Album author, can be empty for multi-artist compilations."`
	Price  float64 `json:"price" minimum:"0" description:"Price in USD."`
}

// albums slice to seed record album data.
var albums = []album{
	{ID: "1", Title: "Blue Train", Artist: "John Coltrane", Price: 56.99},
	{ID: "2", Title: "Jeru", Artist: "Gerry Mulligan", Price: 17.99},
	{ID: "3", Title: "Sarah Vaughan and Clifford Brown", Artist: "Sarah Vaughan", Price: 39.99},
}

func main() {
	service := web.DefaultService()

	service.OpenAPI.Info.Title = "Albums API"
	service.OpenAPI.Info.WithDescription("This service provides API to manage albums.")
	service.OpenAPI.Info.Version = "v1.0.0"

	service.Get("/albums", getAlbums())
	service.Get("/albums/{id}", getAlbumByID())
	service.Post("/albums", postAlbums(), nethttp.SuccessStatus(http.StatusCreated))

	service.Docs("/docs", v4emb.New)

	log.Println("Starting service")
	if err := http.ListenAndServe("localhost:8080", service); err != nil {
		log.Fatal(err)
	}
}

func getAlbums() usecase.Interactor {
	u := usecase.NewIOI(nil, []album{}, func(ctx context.Context, _, output interface{}) error {
		out := output.(*[]album)
		*out = albums
		return nil
	})
	u.SetTags("Album")

	return u
}

func postAlbums() usecase.Interactor {
	u := usecase.NewIOI(album{}, album{}, func(ctx context.Context, input, output interface{}) error {
		in := input.(album)
		out := output.(*album)

		// Check if id is unique.
		for _, a := range albums {
			if a.ID == in.ID {
				return status.AlreadyExists
			}
		}

		// Add the new album to the slice.
		albums = append(albums, in)

		*out = in
		return nil
	})
	u.SetTags("Album")
	u.SetExpectedErrors(status.AlreadyExists)

	return u
}

func getAlbumByID() usecase.Interactor {
	type getAlbumByIDInput struct {
		ID string `path:"id"`
	}

	u := usecase.NewIOI(getAlbumByIDInput{}, album{}, func(ctx context.Context, input, output interface{}) error {
		in := input.(getAlbumByIDInput)
		out := output.(*album)

		for _, album := range albums {
			if album.ID == in.ID {
				*out = album
				return nil
			}
		}
		return status.Wrap(errors.New("album not found"), status.NotFound)
	})
	u.SetTags("Album")
	u.SetExpectedErrors(status.NotFound)

	return u
}
```

## Tidy modules and start the app!

In order to download necessary modules run `go mod tidy` in the directory of your module.

Then run the app with `go run main.go` and open http://localhost:8080/docs.

![Swagger UI Screenshot](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/mduhu3u8xqsnwcuts5tc.png)

OpenAPI schema will be available at http://localhost:8080/docs/openapi.json.

<details>
	<summary>openapi.json</summary>
	
	
```json
{
 "openapi": "3.0.3",
 "info": {
  "title": "Albums API",
  "description": "This service provides API to manage albums.",
  "version": "v1.0.0"
 },
 "paths": {
  "/albums": {
   "get": {
    "tags": [
     "Album"
    ],
    "summary": "Get Albums",
    "description": "",
    "operationId": "getAlbums",
    "responses": {
     "200": {
      "description": "OK",
      "content": {
       "application/json": {
        "schema": {
         "type": "array",
         "items": {
          "$ref": "#/components/schemas/Album"
         }
        }
       }
      }
     }
    }
   },
   "post": {
    "tags": [
     "Album"
    ],
    "summary": "Post Albums",
    "description": "",
    "operationId": "postAlbums",
    "requestBody": {
     "content": {
      "application/json": {
       "schema": {
        "$ref": "#/components/schemas/Album"
       }
      }
     }
    },
    "responses": {
     "201": {
      "description": "Created",
      "content": {
       "application/json": {
        "schema": {
         "$ref": "#/components/schemas/Album"
        }
       }
      }
     },
     "409": {
      "description": "Conflict",
      "content": {
       "application/json": {
        "schema": {
         "$ref": "#/components/schemas/RestErrResponse"
        }
       }
      }
     }
    }
   }
  },
  "/albums/{id}": {
   "get": {
    "tags": [
     "Album"
    ],
    "summary": "Get Album By ID",
    "description": "",
    "operationId": "getAlbumByID",
    "parameters": [
     {
      "name": "id",
      "in": "path",
      "required": true,
      "schema": {
       "type": "string"
      }
     }
    ],
    "responses": {
     "200": {
      "description": "OK",
      "content": {
       "application/json": {
        "schema": {
         "$ref": "#/components/schemas/Album"
        }
       }
      }
     },
     "404": {
      "description": "Not Found",
      "content": {
       "application/json": {
        "schema": {
         "$ref": "#/components/schemas/RestErrResponse"
        }
       }
      }
     }
    }
   }
  }
 },
 "components": {
  "schemas": {
   "Album": {
    "required": [
     "id",
     "title"
    ],
    "type": "object",
    "properties": {
     "artist": {
      "type": "string",
      "description": "Album author, can be empty for multi-artist compilations."
     },
     "id": {
      "minLength": 1,
      "type": "string",
      "description": "ID is a unique string that determines album."
     },
     "price": {
      "minimum": 0,
      "type": "number",
      "description": "Price in USD."
     },
     "title": {
      "type": "string",
      "description": "Title of the album."
     }
    }
   },
   "RestErrResponse": {
    "type": "object",
    "properties": {
     "code": {
      "type": "integer",
      "description": "Application-specific error code."
     },
     "context": {
      "type": "object",
      "additionalProperties": {},
      "description": "Application context."
     },
     "error": {
      "type": "string",
      "description": "Error message."
     },
     "status": {
      "type": "string",
      "description": "Status text."
     }
    }
   }
  }
 }
}
```
</details>
