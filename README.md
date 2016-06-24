## Deprecation notice

This package is deprecated. Please use [contentful-extension-cli](https://www.npmjs.com/package/contentful-extension-cli).

## Introduction [![Build Status](https://travis-ci.com/contentful/contentful-widget-cli.svg?token=GuTGqA8WbzXbprUpRd2Y&branch=master)](https://travis-ci.com/contentful/contentful-widget-cli)
Contentful allows customers to customize and tailor the UI using custom made widgets. Widgets have to be uploaded to Contentful in order to be able to use them in the UI.

This repo hosts `contentful-widget` a Command Line Tool (CLI) developed to simplify the management tasks associated with custom widgets. With the CLI you can:

- Create widgets
- Update existing widgets
- Read widgets
- Delete widgets

## Installation

```
npm install -g contentful-widget-cli
```


## Available commands

`contentful-widget` is composed of 4 subcommands that you can use to manage the widgets.

**create a widget**

```
contentful-widget create [options]
```
Use this subcommand to create the widget for the first time. Succesive modifications made to the widget will be have to be using the `update` subcommand.

**read a widget**

```
contentful-widget read [options]
```
Use this subcommand to read the widget payload from Contentful. With this subcommand you can also list all the widgets in one space.

**update a widget**

```
contentful-widget update [options]
```
Use this subcommand to modify an existing widget.
 
**delete a widget**

```
contentful-widget delete [options]
```

Use this subcommand to pertmanently delete a widget from Contentful.

For a full list of all the options available on every subcommand use the `--help` option.

## Misc

The following sections describe a series of concepts around the widgets and how the CLI deals with them.

### Widget properties

The following table describes the properties that can be set on a widget.

Property | Required| Type | Description
---------|---------|------|------------
name | yes | String | Widget name
fieldTypes | yes | Array\<String\> * | Field types where a widget can be used
src | ** | String | URL where the root HTML document of the widget can be found
srcdoc | ** | String | Path to the local widget HTML document
sidebar | no | Boolean | Controls the location of the widget. If `true` it will be rendered on the sidebar

\* Valid field types are: `Symbol`, `Symbols`, `Text`, `Integer`, `Number`, `Date`, `Boolean`, `Object`, `Entry`, `Entries`, `Asset`, `Assets`

\** One of `src` or `srcdoc` have to be present

#### Difference between `src` and `srcdoc` properties

When using `src` property, a widget is considered 3rd party hosted. Relative links in the root HTML document are supported as expected.

When using `srcdoc` property, a widget is considered internally hosted. A file being pointed by the `srcdoc` property will be loaded and uploaded as a string to Contentful. All local dependencies have to be manually inlined into the file. The command line tool does not take care of link resolving and inlining of referenced local resources. The maximal size of a file used with the `srcdoc` property is 200kB. Use [HTML minifier with `minifyJS` option](https://www.npmjs.com/package/html-minifier) and use CDN sources for libraries that your widget is depending on.

If a relative value of `srcdoc` property is used, the path is resolved from a directory in which the descriptor file is placed or a working directory when using the `--srcdoc` command line option.

Use `src` property when you want to be as flexible as possible with your development and deployment process. Use `srcdoc` property if you don't want to host anything on your own and can accept the drawbacks (need for a non-standard build, filesize limitation). 

#### Specifying widget properties

Subcommands that create of modify a widgets (`create` and `upload`) accept the properties for the widget in two forms: command line options or a JSON file.

##### Command line options

For every property in the widget there's a corresponding long option with the same name. So for example, there's a `name` property and so a `--name` option too.

```
contentful-widget create --space-id 123 --name foo --src foo.com/widget
```
Note that camelcased property names like `fieldTypes` are hyphenated (`--field-types`).

##### Descriptor files

Descriptor files are JSON files that contain the values that will be sent to the API to create the widget. By default the CLI will look in the current working directory for a descriptor file called `widget.json`. Another file can be used witht the `--descriptor` option.

A descriptor file can contain:

- All the widget properties (`name`, `src`, ...). Please note that the `srcdoc` property has to be a path to a file containing the widget HTML document.
- An `id` property. Including the `id` in the descriptor file means that you won't have to use the `--id` option when creating or updating a widget.

All the properties included in a descriptor file can be overriden by its counterpart command line options. This means that, for example, a `--name bar` option will take precedence over the `name` property in the descriptor file. Following is an example were the usage of descriptor files is explained:

Assuming that there's a `widget.json` file in the directory where the CLI is run and that's its contents are:

```json
{
  "name": "foo",
  "src": "foo.com/widget",
  "id": "foo-widget"
}
```

The following command

```
contentful-widget create --space-id 123 --name bar
```

Will create the following widget. Note that the `name` is `bar` and that the `id` is `foo-widget`.

```json
{
  "name": "bar",
  "src": "foo.com/widget",
  "id": "foo-widget"
  "sys": {
   "id": "foo-widget"
   ...
  }
}
```


### Authentication

Widgets are managed via the Contentful Management API (CMA).
You will therefore need to provide a valid access token in the
`CONTENTFUL_MANAGEMENT_ACCESS_TOKEN` environment variable.

Our documentation describes [how to obtain a token](https://www.contentful.com/developers/docs/references/authentication/#getting-an-oauth-token).


### Version locking

Contentful API use [optimistic locking](https://www.contentful.com/developers/docs/references/content-management-api/#/introduction/updating-and-version-locking) to ensure that accidental non-idemptotent operations (`update` or `delete`) can't happen.

This means that the CLI  needs to know the current version of the widget when using the `update` and `delete` subcommands. On these case you have to specify the version of the widget using the `--version` option.

If you don't want to use the `--version` option on every update or deletion, the alternative is to use `--force`. When the `--force` option is present the CLI will automatically use the latest version of the widget. Be aware that using `--force` option might lead to accidental overwrites if multiple people are working on the same widget.

### Programmatic usage

You can also use CLI's methods with a programmatic interface (for example in your build process). A client can be created simply by requiring `contentful-widget-cli` npm package:

```js
const cli = require('contentful-widget-cli');

const client = cli.createClient({
  accessToken: process.env.CONTENTFUL_MANAGEMENT_ACCESS_TOKEN,
  spaceId: 'xxxyyyzzz',
  host: 'https://api.contentful.com' // optional, default value shown
});

// getting an array of all widgets in the space
client.getAll().then(function (widgets) {});

// getting a single widget
client.get(widgetId).then(function (widget) {});

// save method takes an object of widget properties described above
client.save({
  id: 'test-id',
  name: 'test',
  src: 'https://widget.example'
}).then(function (savedWidget) {});

// the only difference is that srcdoc is a HTML document string
// instead of a path (so it can be fed with custom build data)
client.save({
  id: 'test-id',
  name: 'test',
  srcdoc: '<!doctype html><html><body><p>test...'
}).then(function (savedWidget) {});

// if widget was saved, a result of the get method call will contain
// version that has to be supplied to the consecutive save call
client.save({
  id: 'test-id',
  name: 'test',
  src: 'https://widget.example',
  version: 123
}).then(function (savedWidget) {});

// delete method also requires a version number
client.delete(widgetId, currentWidgetVersion).then(function () {});
```
