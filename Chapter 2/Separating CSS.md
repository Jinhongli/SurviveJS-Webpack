虽然现在的配置看起来还不错，但是CSS文件哪去了？在之前的配置中，CSS文件放在了Javascript中！虽然在开发环境下这样做非常方便，但并不好。

这种解决方法：

1. 无法缓存CSS。
2. 导致无样式刷新（FOUC，Flash of Unstyled Content）。因为样式需要等JS执行完毕之后才能插入HTML。

为了避免这种问题，可以将CSS抽离成单独的文件。

在webpack中可以使用`[ExtractTextPlugin](https://www.npmjs.com/package/extract-text-webpack-plugin)`来分离CSS文件。它可以将多个CSS文件合为一个，为此它还自带`loader`来完成提取过程，然后将`loader`合并的结果生成一个单独的文件。

但在编译过程中，上述过程会导致`ExtractTextPlugin`产生开销。另外它还无法与HMR同时工作。最好的解决方法就是只在生产环境中使用这一插件。

# 设置`ExtractTextPlugin`

首先安装插件：

```shell
npm install extract-text-webpack-plugin --save-dev
```

`ExtractTextPlugin.extract`是插件自带的`loader`，标记将被提取的资源，然后根据它的设置通过插件来执行。

`ExtractTextPlugin.extract`接受两个参数：`use`和`fallback`。默认情况下，`use`对原始chunk（initial chunk）使用，之后用`fallback`。只有当`allChunks: true`时才会接触分离的bundle。

如果想要抽离更多格式的CSS文件（例如Sass），应当在`use`中传入更多的`loader`。`use`和`fallback`接受：字符串(`loader`)、自定义`loader`、数组形式的自定义`loader`。

具体配置如下：

**webpack.parts.js**

```javascript
const ExtractTextPlugin = require('extract-text-webpack-plugin');


...


exports.extractCSS = ({ include, exclude, use }) => {
  // Output extracted CSS to a file
  const plugin = new ExtractTextPlugin({
    filename: '[name].css',
  });

  return {
    module: {
      rules: [
        {
          test: /\.css$/,
          include,
          exclude,

          use: plugin.extract({
            use,
            fallback: 'style-loader',
          }),
        },
      ],
    },
    plugins: [ plugin ],
  };
};
```

`[name]`占位符表示引用css的文件名（`entry`中的name）。占位符和`hash`会在[Adding Hashes to Filenames]章节详细介绍。

可能会有多个类型的文件均调用`plugin.extract`，插件会将它们都合为一个CSS文件。Another option would be to extract multiple CSS files through separate plugin definitions and then concatenate them using `merge-files-webpack-plugin`.

> 如果你想要输出CSS文件到指定的文件夹，可以添加路径实现：`filename: 'style/[name].css'`

# 写入配置

将插件写入配置文件中：

**webpack.config.js**

```javascript
const commonConfig = merge([
  ...

  <del>parts.loadCSS(),</del>

]);

const productionConfig = merge([

  parts.extractCSS({ use: 'css-loader' }),

]);

const developmentConfig = merge([
  ...

  parts.loadCSS(),

]);
```

使用这种配置，在开发模式下依然可以使用HMR，但是对于生产模式，则会产生单独的CSS文件。`HtmlWebpackPlugin`插件会自动将生成的CSS文件插入`index.html`

> If you are using CSS Modules, remember to tweak `use` accordingly as discussed in the [Loading Styles] chapter. You can maintain separate setups for normal CSS and CSS Modules so that they get loaded through separate logic.

执行`npm run build`，会打印出如下的信息：

```shell
Hash: 00f10dc611d5a9322db4
Version: webpack 2.4.1
Time: 1487ms
     Asset       Size  Chunks             Chunk Names
    app.js    3.96 kB       0  [emitted]  app
   app.css   68 bytes       0  [emitted]  app
index.html  218 bytes          [emitted]
   [0] ./app/main.css 41 bytes {0} [built]
   [1] ./app/component.js 224 bytes {0} [built]
   [2] ./app/index.js 155 bytes {0} [built]
   [3] ./~/base64-js/index.js 3.48 kB [built]
...
```

可以看到样式被抽离成了单独的CSS文件，JS文件的体积也减小了。这样做避免了FOUC问题。（浏览器不需要等待JS执行完毕才去加载样式）

> If you are getting `Module build failed: CssSyntaxError:` or `Module build failed: Unknown word error`, make sure your `common` configuration doesn't have a CSS-related section set up.

> `extract-loader` is a light alternative to ExtractTextPlugin. It does less, but can be enough for basic extraction needs.

# 在JS之外管理样式

通过JS引用样式然后打包是推荐的做法，但是也可以通过`entry`和`globbing`实现：

```javascript
...
const glob = require('glob');

// Glob CSS files as an array of CSS files. This can be
// problematic due to CSS rule ordering so be careful!
const PATHS = {
  ...

  style: glob.sync('./app/**/*.css'),

};

...

const commonConfig = merge([
  {
    entry: {
      ...

      style: PATHS.style,

    },
    ...
  },
  ...
]);
```

这样做就不需要在JS中引入CSS文件了，这也意味着CSS模块化无法工作。你将会得到`style.css`和`style.js`两个文件。The latter file contains content like webpackJsonp([1,3],[function(n,c){}]); and it doesn't do anything as discussed in the webpack issue 1967.

如果想要严格控制CSS的顺序，可以设置入口CSS文件，然后使用`@import`来引入其余文件。 Another option would be to set up a JavaScript entry and go through import to get the same effect.

> `css-entry-webpack-plugin` has been designed to help with this usage pattern. The plugin is able to extract a CSS bundle from an entry without ExtractTextPlugin.

# 总结

现在的配置可以抽离CSS文件，但其实还可以抽离HTML模版，和其他任何类型的文件，只要正确配置`ExtractTextPlugin`插件。

- 使用`ExtractTextPlugin`将CSS文件从JS中抽离出来，解决了FOUC问题，提高了缓存行为。
- `ExtractTextPlugin`并不是唯一的解决方案。在有限的上下文环境中，使用`extract-loader`可以得到同样的结果
- 如果不想在JS中引入CSS文件，可以在`entry`中设置来实现，但是需要注意CSS的引入顺序。