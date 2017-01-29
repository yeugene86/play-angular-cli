# play-angular-cli


## Prerequisites

## Setup

### PlayFramework backend

```
$ activator new backend play-scala
```

### angular-cli frontend
```
$ ng new frontend
```

## Configuration

controllers.ngController.scala

```
package controllers

import javax.inject.{Inject, Singleton}

import play.api.mvc.{Action, Controller}
import play.api.{Environment, Mode}
import play.api.libs.ws.WSClient

import scala.concurrent.ExecutionContext

/**
  * This controller creates an `Action` to handle HTTP requests to the
  * angular-CLI's page.
  */
@Singleton
class ngController @Inject()(ws: WSClient, environment: Environment)(implicit ec: ExecutionContext) extends Controller {

  /**
    * Create an Action to render an HTML page with a welcome message.
    * The configuration in the `routes` file means that this method
    * will be called when the application receives a `GET` request with
    * a path of `/`.
    */
  def index = Action {
    Ok(views.html.ng("NG2 is ready."))
  }


  def dist(file: String) = environment.mode match {
    case Mode.Dev => Action.async {
      ws.url("http://localhost:4200/" + file).get().map { response =>
        Ok(response.body)
      }
    }
    // If Production, use build files.
    case Mode.Prod => Assets.versioned(path="public/dist", file)
  }

}
```
views.ng.scala.html
```
@*
* This template is called from the `index` template. This template
* handles the rendering of the page header and body tags. It takes
* two arguments, a `String` for the title of the page and an `Html`
* object to insert into the body of the page.
*@
@(title: String)

<!DOCTYPE html>
<html lang="en">
  <head>
    @* Here's where we render the page title `String`. *@
    <title>@title</title>
    <base href="/">
    <link rel="stylesheet" media="screen" href="@routes.Assets.versioned("stylesheets/main.css")">
    <link rel="shortcut icon" type="image/png" href="@routes.Assets.versioned("images/favicon.png")">
    <meta name="viewport" content="width=device-width, initial-scale=1">
  </head>
  <body>
    <app-root>Loading...</app-root>
    <link href="dist/styles.bundle.css" rel="stylesheet">
    <script type="text/javascript" src="dist/inline.bundle.js"></script>
    <script type="text/javascript" src="dist/vendor.bundle.js"></script>
    <script type="text/javascript" src="dist/main.bundle.js"></script>
  </body>
</html>
```


conf/routes

```
# replace the / by the following controller
GET     /                        controllers.ngController.index

# Bundle files generated by Webpack
GET     /dist/*file              controllers.ngController.dist(file)
```

build.sbt
```
val frontEndProjectName = "frontend"
val backEndProjectName = "backend"
val distDirectory = ".." + backEndProjectName + "public/dist"


// Starts: angularCLI build task
lazy val frontendDirectory = baseDirectory {_ /".."/frontEndProjectName}

val buildCmd = "ng build --output-path ../" + backEndProjectName + "/public/dist"

val ngBuild = taskKey[Unit]("ng build task.")
ngBuild := { Process(buildCmd , frontendDirectory.value) ! }
(packageBin in Universal) <<= (packageBin in Universal) dependsOn ngBuild
// Ends.


// Starts: ngServe process when running locally and build actions for production bundle
PlayKeys.playRunHooks <+= frontendDirectory.map(base => ng(base))
// Ends.
```

```
$ sbt playGenerateSecret
$ export PLAY_APP_SECRET=`put the result here`
```

```
$ unzip target/universal/backend-1.0-SNAPSHOT.zip
$ backend-1.0-SNAPSHOT/bin/backend -Dplay.crypto.secret=$PLAY_APP_SECRET
```

## Adding webpack
```
$ cd frontend
```

* Add webpack to the project (package.json)  
  `note:` that using the parameters --save-dev and -save will respectively 
  add the packages to package.json sections devDependencies and dependencies

* Install webpack
```
$ npm install webpack@beta --save-dev 
```
* Install TypeScript's loader  
  `note:` TypeScript should already been installed when using Angular2
```
$ npm install webpack@beta typescript ts-loader --save-dev 
```

# webpack plugins

```
npm install html-webpack-plugin --save-dev
```
