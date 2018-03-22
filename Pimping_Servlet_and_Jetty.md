# Pimping Servlet and Jetty

Sometimes you just want a servlet, not a whole web application framework.   But to a Scala programmer the servlet API is pretty ugly.

## The Starting Point

Consider the GET service method from the servlet spec:

```scala
protected void doGet(HttpServletRequest req, HttpServletResponse resp)
```

Implementations create a response as a side-effect by mutating the HttpServletResponse object. The resulting code is not pretty. We really want to return our response explicitly. Here is an example from one of the lightweight scala web frameworks called Step:

```scala
get("/") {
  <h1>Hello {params("yourname")}!</h1>
}
```

The part in braces is effectively a service function with type:

() `=> Any`

We simply return an object as a response. Nice! But where did `params(...)` come from? The request information is passed via a thread local variable instead of as a parameter of the service function.  Not so nice.

## The Desired Result

The aim of the pimped servlet API is to let you write:

```scala
get("/") { 
    request => <h1>Hello {request("yourname") getOrElse "stranger"}!</h1>
}
```

This is similar to the previous example, but it has a more functional style.  Here is a larger example returning some KML for Google Earth to display:

```scala
implicit val contentType = ContentType("application/vnd.google-earth.kml+xml") // note 1
get("/cafe") {
  case Params(BBox(w, s, e, n)) =>   // note 2
    <kml xmlns="http://www.opengis.net/kml/2.2">
      <Document>
      {  
        for((name, lon, lat) <- findCafesIn(w, s, e, n)) yield
        <Placemark>
          <name>{name}</name>
          <Point>
            <coordinates>{lon},{lat},0</coordinates>
          </Point>
        </Placemark>
      }
      </Document>
    </kml>
  case _ => 404 // note 3
} 
```

The example extracts the coordinates of a bounding box `(w, s, e, n)` from a Google Earth request and uses them to find a collection of locations via `findCafes(..)`, which is supposedly a spatial query on some data source.  The locations are wrapped in KML markup and returned.

This shows how to set an implicit content type (note 1); the use of extractors to obtain request parameters (note 2); and implicit conversion of responses such as XML or integers. An integer gets converted to an error response (note 3).

## The Pimping

### Responses

The servlet API pimping starts by lifting the service method out of the servlet class as a standalone scala function:

```scala
(HttpServletRequest, HttpServletResponse) => Unit
```

Curry that to get;

```scala
HttpServletRequest => HttpServletResponse => Unit
```

Now lets call the second part of this the Responder.

```scala
type Responder = HttpServletResponse => Unit
```

So our service function is:

```scala
HttpServletRequest => Responder
```

The `Responder` contains all the ugly side effects. But, conveniently, we can write these separately and a few canned Responders cover most needs.  Implicit conversions will wrap result types such as XML in appropriate Responders.  Here is the conversion from `Int` to `Responder`:

```scala
implicit def  toErrorResponse(code: Int): Responder = { _.sendError(code) }
```

XML is handled by an only slightly more complicated Responder:

```scala
implicit def toXMLResponse(content: Node)(implicit contentType: ContentType): Responder = {
  response =>
    response.setContentType(contentType.mime)
    response.getWriter.write(content.toString)
}
```

You can always include a specific `Responder` definition right in the service function and deal with the `HttpServletResponse` object directly, if you need to.

### Requests

Similarly, the naked `HttpServletRequest` object is passed to the service function and you can deal with it directly.  However, extractors allow you to write the service function as a series of case clauses, dealing mainly with the contents of the request.

Here is an example where we extract the parameter map and query it:

```scala
get("/stock") {
   case Params(params) if params contains "SKU" => ...
  ...
}
```

Here we set up an extractor for a string parameter and a, possibly repeated, integer parameter:

```scala
val Action = StringParam("ACTION")
val AssetNumber = LongParam("INUM")
post("/assets") {
  case Params(Action("delete")&AssetNumber(inum)) => ... // delete an asset with number 'inum'
  case Params(Action("allocate")&AssetNumber(from, until)) => ... // allocate assets in the range 'from' .. 'until'
 }
```

When the pattern matching style is not suitable, an implicit wrapper, RichRequest, lets you query the HttpServletRequest using scala conventions:

```scala
get("/stock") { 
  request => request("SKU") map lookupSKU getOrElse generateSKUList 
}
```

### Jetty

The get and post methods shown in the examples have this signature:

```scala
def get( servletContext: String)( serviceFunction:  HttpServletRequest => HttpServletResponse => Unit)
```

This binds the `serviceFunction` to an `HTTP GET` with the given `servletContext` (ie the leading part of the path in the request).  One way to implement this binding would be to write a central routing servlet.   But designing routing schemes quickly goes beyond pimping the servlet API.

This is where Jetty comes in.

The implementation given here relies on the routing system provided by Jetty based on the standard concept of a servlet context. Jetty provides an API for defining servlet contexts that we can exploit instead of writing a web.xml configuration file.  Jetty also provides virtual host routing.  This is exposed by a more general binding method that underlies get() and post() :

```scala
def on(method: String, vhost: Option[String], path: String)(serviceFunction: HttpServletRequest => HttpServletResponse => Unit)
```

## Putting it all Together

The pimping layer is very thin and does not prevent you from getting at the HttpServletRequest and HttpServletResponse if you need to.  It is divided into small modules, which can be seen in this complete example:

```scala
package example
import au.com.langdale.webserver._
import Driver._       // start and stop Jetty
import Connectors._   // Jetty-specific connection methods such as listen()
import Handlers._     // Jetty-specific binding methods get("...") and post("...")
import Responders._   // generic servlet Responders and implicit conversions for responses
import Requests._     // generic servlet request extractors and wrapper 
object HelloWorld {
  val host = "127.0.0.1"
  val port = 9986
  // tell Jetty where to receive connections
  listen(host, port)
  // may be omitted since this is the default for XML responses
  implicit val contentType = ContentType("application/xhtml+xml")
  // hello world server
  get("/test") { request =>
    <html xmlns="http://www.w3.org/1999/xhtml">
      <head><title>The Test Page</title></head>
      <body>
        <h1>Test</h1>
        <p>Hello world.</p></body>
    </html>
  }
  def main(args: Array[String]) = { start; join }
}
```

## Get the Code

The code for all this is on github: [http://github.com/arnolddevos/JettyS](http://github.com/arnolddevos/JettyS) There is not much to it, feel free to embrace and extend.
