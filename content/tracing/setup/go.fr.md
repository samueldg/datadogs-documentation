---
title: Tracer des application Go
kind: Documentation
aliases:
  - /fr/tracing/go/
  - /fr/tracing/languages/go
further_reading:
  - link: 'https://github.com/DataDog/dd-trace-go'
    tag: « Github »
    text: Code source
  - link: 'https://godoc.org/github.com/DataDog/dd-trace-go/tracer'
    tag: GoDoc
    text: Page des packages
  - link: tracing/visualization/
    tag: Documentation
    text: 'Explorez vos services, vos ressources et vos traces'
---
## Installation

Pour commencer à tracer les applications écrites en Go, commencez par [installer et configurer Datadog Agent][1] (consultez la documentation supplémentaire pour [le traçage des applications Docker](/tracing/setup/docker/)).

Installez le Go Tracer depuis le dépôt Github :

```go
go get "github.com/DataDog/dd-trace-go/tracer"
```

Enfin, importez le traceur et instrumentez votre code!

## Exemple

```go
package main

import (
  "github.com/DataDog/dd-trace-go/tracer"
)

func main {
  span := tracer.NewRootSpan("web.request", "my_service", "resource_name")
  defer span.Finish()
  span.SetMeta("my_tag", "my_value")
}
```

Pour un autre exemple, consultez le fichier [`example_test.go`][2] dans le paquet Go Tracer

## Compatibilité

Actuellement, que la version 1.7+ de Go est compatible.

### Compatibilité de Framework

La bibliothèque ddtrace fonctionne avec plusieurs frameworks web, y inclus :

___

{{% table responsive="true" %}}
| Framework   | Documentation du Framework               |  Documentation GoDoc Datadog                                                         |
|-------------|---------------------------------------|------------------------------------------------------------------------------------|
| Gin         | https://gin-gonic.github.io/gin/      | https://godoc.org/github.com/DataDog/dd-trace-go/contrib/gin-gonic/gin |
| Gorilla Mux | http://www.gorillatoolkit.org/pkg/mux | https://godoc.org/github.com/DataDog/dd-trace-go/contrib/gorilla/mux   |
| gRPC        | https://github.com/grpc/grpc-go       | https://godoc.org/github.com/DataDog/dd-trace-go/contrib/google.golang.org/grpc         |
|gRPC v1.2 | https://github.com/grpc/grpc-go | https://godoc.org/github.com/DataDog/dd-trace-go/contrib/google.golang.org/grpc.v12 |
{{% /table %}}

### Compatibilité des bilbliothèques

Il inclut également le support pour les bibliothèques et data stores suivants:

___

{{% table responsive="true" %}}
| Bibliothèque | Documentation de la Bibliothèque | Documentation GoDoc Datadog |
|-------------------|--------------------------------------------------|------------------------------------------------------------------------------------------|
|Elasticsearch | https://github.com/olivere/elastic | https://godoc.org/github.com/DataDog/dd-trace-go/contrib/olivere/elastic |
|gocql| https://github.com/gocql/gocql | https://godoc.org/github.com/DataDog/dd-trace-go/contrib/gocql/gocql |
|Go Redis| https://github.com/go-redis/redis |https://godoc.org/github.com/DataDog/dd-trace-go/contrib/go-redis/redis |
|HTTP | https://godoc.org/github.com/DataDog/dd-trace-go/contrib/net/http | https://godoc.org/github.com/DataDog/dd-trace-go/contrib/net/http |
|HTTP router|https://github.com/julienschmidt/httprouter| https://godoc.org/github.com/DataDog/dd-trace-go/contrib/julienschmidt/httprouter |
|Redigo Redis| https://github.com/garyburd/redigo | https://godoc.org/github.com/DataDog/dd-trace-go/contrib/garyburd/redigo |
|SQL| https://godoc.org/github.com/DataDog/dd-trace-go/contrib/database/sql |https://godoc.org/github.com/DataDog/dd-trace-go/contrib/database/sql |
|SQLx | https://github.com/jmoiron/sqlx | https://godoc.org/github.com/DataDog/dd-trace-go/contrib/jmoiron/sqlx |
{{% /table %}}

### API OpenTracing

Client APM Datadog qui implémente un Tracer [OpenTracing][3].

**Initialisation**

Afin d'utiliser le Datadog Tracer avec un API OpenTracing, vous devez initialiser le tracer avec un objet `Configuration` complet :

```go
import (
  // ddtrace namespace is suggested
  ddtrace "github.com/DataDog/dd-trace-go/opentracing"
  opentracing "github.com/opentracing/opentracing-go"
)

func main() {
  // create a Tracer configuration
  config := ddtrace.NewConfiguration()
  config.ServiceName = "api-intake"
  config.AgentHostname = "ddagent.consul.local"

  // initialize a Tracer and ensure a graceful shutdown
  // using the `closer.Close()`
  tracer, closer, err := ddtrace.NewTracer(config)
  if err != nil {
    // handle the configuration error
  }
  defer closer.Close()

  // set the Datadog tracer as a GlobalTracer
  opentracing.SetGlobalTracer(tracer)
  startWebServer()
}
```

La fonction `NewTracer(config)` retourne une instance de `io.Closer` qui peut être utilisée pour fermer le `tracer`. Vous devez toujours appeler `closer.Close()` - sinon, les buffers internes ne seront pas vidés et vous risquez de perdre des traces.

**Usage**

Consultez la [documentation Opentracing][4] pour des modèles d'utilisation. Documentation antérieurs est disponible en [format GoDoc][5].

## En apprendre plus

{{< partial name="whats-next/whats-next.html" >}}

[1]: /tracing/setup
[2]: https://github.com/DataDog/dd-trace-go/blob/master/tracer/example_test.go
[3]: http://opentracing.io
[4]: https://github.com/opentracing/opentracing-go
[5]: https://godoc.org/github.com/DataDog/dd-trace-go/tracer