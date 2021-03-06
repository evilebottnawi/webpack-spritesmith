[![npm](https://img.shields.io/npm/v/webpack-spritesmith.svg)](https://www.npmjs.com/package/webpack-spritesmith)

Webpack plugin that converts set of images into a spritesheet and SASS/LESS/Stylus mixins, using
[spritesmith](https://github.com/Ensighten/spritesmith) and [spritesheet-templates](https://github.com/twolfson/spritesheet-templates) 

All ideas are shamelessly taken from [gulp.spritesmith](https://github.com/twolfson/gulp.spritesmith).

### Example

Let's say you have following folder structure

```
/
|-src
| |-ico
| | |-new.png
| | |-open.png
| | |-save.png
| | ...
| |-style.styl
| ...
|-webpack.config.js

```

Then you need to instantiate plugin in webpack config like this:

```javascript

//webpack.config.js
var path = require('path');

var SpritesmithPlugin = require('webpack-spritesmith');

module.exports = {
    // ...
    module: {
        loaders: [
            {test: /\.styl$/, loaders: [
                'style',
                'css',
                'stylus'
            ]},
            {test: /\.png$/, loaders: [
                'file?name=i/[hash].[ext]'
            ]}
        ]
    },
    resolve: {
        modulesDirectories: ["web_modules", "node_modules", "spritesmith-generated"]
    },
    plugins: [
        new SpritesmithPlugin({
            src: {
                cwd: path.resolve(__dirname, 'src/ico'),
                glob: '*.png'
            },
            target: {
                image: path.resolve(__dirname, 'src/spritesmith-generated/sprite.png'),
                css: path.resolve(__dirname, 'src/spritesmith-generated/sprite.styl')
            },
            apiOptions: {
                cssImageRef: "~sprite.png"
            }
        })
    ]
    // ...
};


```

And then just use it


```stylus

//style.styl
@import '~sprite.styl'

.close-button
    sprite($close)
.open-button
    sprite($open)

```

There are few things to notice in config

- file-loader used for generated image
- `resolve` contains location of where generated image is
- cssImageRef is specified as '~sprite.png'

So the way generated image is accessed from generated API at the moment has to be specified manually.

### Config

- `src` - used to build list of source images
    - `cwd` should be the closest common directory for all source images;
    - `glob` well... it is a glob

    `cwd` and `glob` both will be passed directly to [glob](https://github.com/isaacs/node-glob) (and [gaze](https://github.com/shama/gaze)
    in watch mode), then resulting list of files will be used as list of source images

- `target` - generated files
    - `image` - target image filename
    - `css` - can be one of the following
        - `"full/path/to/spritesheet/api"` - for example `path.resolve(__dirname, 'src/spritesmith-generated/sprite.styl')`
        - `["full/path/to/spritesheet/api1", "full/path/to/spritesheet/api2"]`,
        - `["full/path/to/spritesheet/api1", ["full/path/to/spritesheet/api2", spritesmithTemplatesOptions]]`
            spritesmithTemplatesOptions - is the second argument [here](https://github.com/twolfson/spritesheet-templates#templaterdata-options)

            for example

            ```javascript
                ...
                css: [
                    path.resolve(__dirname, 'src/spritesmith-generated/sprite.styl'),
                    [path.resolve(__dirname, 'src/spritesmith-generated/sprite.json'), {
                        format: 'json_texture'
                    }]
                ]
            ```
- `apiOptions` - optional
    - `generateSpriteName` - function. Takes full path to source image file and expected to return
    name by which it will be referenced in API. Return value will be used as `sprite.name` for
    [spritesheet-templates](https://github.com/twolfson/spritesheet-templates). Default behaviour is to
    use filename (without dirname and extension)
    - `cssImageRef` - path by which generated image will be referenced in API 
- `spritesmithOptions` - optional. Options for [spritesmith](https://github.com/Ensighten/spritesmith)
- `retina` - optional, when specified, uses retina capabilities of [spritesheet-templates](https://github.com/twolfson/spritesheet-templates). Can be either suffix string (like '@2x') or object consisting of three fields:
    - `classifier` - `Function` that allows to say which source is for retina spritesheet and which is not. Will be called with full path to source file, and should return an object of this format -
        ```javascript
        
            {
                type: String, // determines which kind of source is this. Can contain one of two values: 'retina' and 'normal'
                normalName: String, //full path to corresponding normal source image
                retinaName: String, //full path to corresponding retina source image
            }
        ```
    - `targetImage` - full path to generated retina image
    - `cssImageRef` - path by which generated image will be referenced in API

    When used as suffix string it applies to source files, filename for retina spritesheet image and cssImageRef

    `apiOptions.generateSpriteName` will be applied to `normalName` returned by retina.classifier
- `customTemplates` - optional. Object with keys and values corresponding to format names and template descriptions respectively.
    Template description can be either a `path/to/handlebars/template/file` or template function

    You can use templates registered here as `format` in "target.css"

    For example you can write something like this

    ```javascript

    //webpack.config.js
    var templateFunction = function (data) {
        var shared = '.ico { background-image: url(I) }'
            .replace('I', data.sprites[0].image);

        var perSprite = data.sprites.map(function (sprite) {
            return '.ico-N { width: Wpx; height: Hpx; background-position: Xpx Ypx; }'
                .replace('N', sprite.name)
                .replace('W', sprite.width)
                .replace('H', sprite.height)
                .replace('X', sprite.offset_x)
                .replace('Y', sprite.offset_y);
        }).join('\n');

        return shared + '\n' + perSprite;
    });

    module.exports = {
        ...
        plugins: [
            new SpritesmithPlugin({
                target: {
                    ...
                    css: [
                        [path.resolve(__dirname, 'src/spritesmith-generated/sprite-1.css'), {
                            format: 'function_based_template'
                        }],
                        [path.resolve(__dirname, 'src/spritesmith-generated/sprite-2.css'), {
                            format: 'handlebars_based_template'
                        }]
                    ]
                },
                customTemplates: {
                    'function_based_template': templateFunction,
                    'handlebars_based_template': path.resolve(__dirname, '../my_handlebars_template.handlebars')
                },
                ...
            })
        ]
    }

    ```
