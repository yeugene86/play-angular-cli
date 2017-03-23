# play-angular-cli


## Prerequisites

* Install activator and sbt
* Install node, npm and angular-cli

## Setup

### PlayFramework backend

```
$ activator new backend play-scala
```

## Configuration

`replace` the source code in controllers.HomeController.scala by  

```scala
package controllers

import javax.inject.{Inject,Singleton}

import play.api.{Environment,Mode}
import play.api.mvc.{Controller,Action}

// Import required for injection
import play.api.libs.ws.WSClient
import scala.concurrent.ExecutionContext

/**
 * This controller creates an `Action` to handle HTTP requests to the
 * application's home page.
 */
@Singleton
class HomeController @Inject()(ws: WSClient, environment: Environment)(implicit ec: ExecutionContext) extends Controller {

  def index = Assets.versioned(path="/public/dist", "index.html")

  def dist(file: String) = environment.mode match {
    case Mode.Dev => Action.async {
      ws.url("http://localhost:4200/dist/" + file).get().map { response =>
        Ok(response.body)
      }
    }
    // If Production, use build files.
    case Mode.Prod => Assets.versioned(path="/public/dist", file)
  }

}
```

`add` the below source code to `conf/routes`   

```yaml

<...>

# Bundle files generated by Webpack
GET     /dist/*file                 controllers.HomeController.dist(file)
```

`add` the below source code in `built.sbt`

```sbt

<...>

/* ================================= WEBPACK ================================== */

val frontEndProjectName = "frontend"
val backEndProjectName = "backend"
val distDirectory = ".." + backEndProjectName + "public/dist"


// Starts: angularCLI build task
val frontendDirectory = baseDirectory {_ /".."/frontEndProjectName}

// Unix
// val webpack = "node_modules/.bin/webpack --progress --colors --display-error-details"

// Windows
val webpack = "cmd /c node_modules\\.bin\\webpack --config webpack.config.js --progress --colors --display-error-details"


val ngBuild = taskKey[Unit]("webpack build task.")
ngBuild := { Process(webpack , frontendDirectory.value) ! }
(packageBin in Universal) <<= (packageBin in Universal) dependsOn ngBuild
// Ends.


// Starts: ngServe process when running locally and build actions for production bundle
PlayKeys.playRunHooks <+= frontendDirectory.map(base => ng(base))
// Ends.
```

`create` a file with the below source code in `project/ng.scala`
```scala
import java.io.File
import java.net.InetSocketAddress

import play.sbt.PlayRunHook
import sbt.Process

object ng {
    def apply(base: File): PlayRunHook = {

        object ngServe extends PlayRunHook {

            var process: Option[Process] = None // This is really ugly, how can I do this functionally?

            override def afterStarted(addr: InetSocketAddress): Unit = {
                process = Some (Process( "ng serve --watch " , base).run)
            }

            override def afterStopped(): Unit = {
                process.foreach(_.destroy)
                process = None
            }
        }

        ngServe
    }
}
```

# Frontend

### angular-cli frontend
```bash
$ ng new frontend
```


## Adding webpack
```bash
$ cd frontend
```

* Add webpack to the project (package.json)  
  `note:` that using the parameters --save-dev and -save will respectively 
  add the packages to package.json sections devDependencies and dependencies

  `warning:` do not install webpack it's already installed by angular-cli
  > npm install webpack@beta --save-dev 


## webpack [loaders](https://webpack.js.org/concepts/loaders/)

* Install [TypeScript's loaders](https://webpack.js.org/guides/webpack-and-typescript/)
```bash
$ npm install ng-router-loader awesome-typescript-loader angular2-template-loader --save-dev 
```

Style loaders
```bash
$ npm install to-string-loader style-loader css-loader --save-dev
```
Extra Libraries
```
$  npm install lodash moment include-media bootstrap --save
```

* Create webpack configuration file   
   `rename`: `'/.index.js'` to `'./src/main.ts'` in `entry:`  
   `rename`: `'/'` to `'../backend/public/dist'` in `output.path:`  
   `add`: `/dist` to `output.publicPath:`

* `copy`: `webpack-template/vendor.ts` to `src`
```
$ cp ../webpack-template/vendor.ts src/
```

* create file `webpack.config.js`

or copy from webpack-template
```
$ cp ../webpack-template/webpack.config.js .
```

webpack.config.js
```javascript
const HtmlWebpackPlugin        = require('html-webpack-plugin'); //installed via npm

const path                     = require('path');
const CommonsChunkPlugin       = require('webpack/lib/optimize/CommonsChunkPlugin');
const ContextReplacementPlugin = require('webpack/lib/ContextReplacementPlugin');
const DefinePlugin             = require('webpack/lib/DefinePlugin');

const ENV  = process.env.NODE_ENV = 'development';

const metadata = {
  env : ENV
};

module.exports = {
  devtool: 'source-map',
  entry: {
    'main'  : './src/main.ts',
    'vendor': './src/vendor.ts'
  },
  module: {
    loaders: [
      {test: /\.css$/,  loader: 'raw-loader', exclude: /node_modules/},
      {test: /\.css$/,  loader: 'style!css?-minimize', exclude: /src/},
      {test: /\.html$/, loader: 'raw-loader'},
      {test: /\.ts$/,   loaders: [
          {loader: 'ts-loader', query: {compilerOptions: {noEmit: false}}},
          {loader: 'angular2-template-loader'}
        ],
        exclude: [/\.(spec|e2e)\.ts$/]
      }
    ]
  },
  output: {
    filename: '[name].js',
    path: '../backend/public/dist',
    publicPath: "/dist"
  },
  plugins: [
    new CommonsChunkPlugin({name: 'vendor', filename: 'vendor.bundle.js', minChunks: Infinity}),
    new DefinePlugin({'webpack': {'ENV': JSON.stringify(metadata.env)}}),
    new ContextReplacementPlugin(
      // needed as a workaround for the Angular's internal use System.import()
      // The (\\|\/) piece accounts for path separators in *nix and Windows
      /angular(\\|\/)core(\\|\/)(esm(\\|\/)src|src)(\\|\/)linker/,
      path.join(__dirname, 'src') // location of your src
    ),
    new HtmlWebpackPlugin({template: './src/index.html'})
  ],
  resolve: {
    extensions: ['.ts', '.js']
  }
};
```
# Route Module Management

```
$ npm install ng-route-loader --save-dev 
```
Add the module entries
```javascript
  entry: {
    'admin'  : './src/app/admin/admin.module.ts',
    'crisis-center'  : './src/app/crisis-center/crisis-center.module.ts',
    <...>
  },
```
and add the loader to the `.ts` test entry
```javascript
      {test: /\.ts$/,   loaders: [
          <....>
          {loader: 'angular2-router-loader'}
          <....>
      }
```


## webpack plugins

* Add the [html-webpack-plugin](https://webpack.js.org/concepts/plugins/#configuration) to change the index.html file  
  this change will allow `<script src=/asset/...>`

```
$ npm install html-webpack-plugin --save-dev
```
* Add HtmlWebpackPlugin object

webpack.config.js
```javascript
const HtmlWebpackPlugin = require('html-webpack-plugin'); //installed via npm
```
* point the template to the index.html file  

```javascript
  plugins: [
    new HtmlWebpackPlugin({template: './src/index.html'}),
    ...
  ]
```



## Running (Prod)

```
$ sbt dist
```

```
$ sbt playGenerateSecret
$ export PLAY_APP_SECRET="put the result here"
```

```
$ unzip target/universal/backend-1.0-SNAPSHOT.zip
$ backend-1.0-SNAPSHOT/bin/backend -Dplay.crypto.secret=$PLAY_APP_SECRET
```




# Reference

heavily inspired by https://github.com/wigahluk/play-webpack  

## angular.io and webpack guide

https://angular.io/docs/ts/latest/guide/webpack.html
