Webpack本身并不处理样式, 需要使用`loader`和`plugin`加载样式文件. 在本章节中将设置CSS文件, 并实现与`webpack-dev-server`配合, 实现自动刷新.

# 加载CSS

需要使用`css-loader`和`style-loader`. 前者用于引入样式文件; 后者用于在html中插入style元素, 病实现了HMR接口.

匹配的文件可以用`file-loader`和`url-loader`来处理, 具体细节在[Loading Asserts]()章节.

由于内联样式并不适用于生产模式, 可以使用`ExtractTextPlugin`来生成单独的CSS文件, 将在下一节具体介绍.

首先执行:

```shell
npm install css-loader style-loader --save-dev
```

然后, 在`part`配置的最后添加:

**webpack.parts.js**

```javascript
exports.loadCSS = ({ include, exclude } = {}) => ({
  module: {
    rules: [
      {
        test: /\.css$/,
        include,
        exclude,

        use: ['style-loader', 'css-loader'],
      },
    ],
  },
});
```
并在`config`配置文件中:

**webpack.config.js**

```javascript
const commonConfig = merge([
    ...
    
    parts.loadCSS(),
]);
```

增加的配置表示对`.css`结尾的文件调用`loader`.

对于`loader`而言, 它会加载源文件, 并返回转换后的新文件. 其调用的顺序是从右到左, 即`styleLoader(cssLoader(inputFile))`.

# 设置初始CSS文件

首先需要在入口中引入CSS文件: 

**app/index.js**

```javascript
import './main.css';

...
```
