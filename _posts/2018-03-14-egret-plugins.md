---
layout: post
title: 一个非典型创业公司的DevOps架构
---

白鹭引擎在升级到5.1的同时, 迎来了一个重大功能的更新. 引擎和编译彻底分开, 同时, 对于编译功能引入的插件机制. 极大的方便了定制工作流.

## 1 插件机制

首先来介绍一下5.1之后的机制.

当使用新创建一个基于5.1的项目之后. 会发现与之前不同, 会多出一个`scripts`文件夹. 这个文件夹里面就是编译相关的类型, 其中`config.ts`便是编译的入口文件了, 类似于`webpack.config.js`.

由于自动生成的里面有很多 '无效' 的代码, 所以我们先来精简一下配置(关键代码).

```
const config: ResourceManagerConfig = {
    configPath: '',
    resourceRoot: () => 'resource',
    buildConfig: (params) => {
        const { target, command, projectName, version } = params;
        if (command === 'build') {
            const outputDir = '.';
            return {
                outputDir,
                commands: [
                    new ExmlPlugin('debug'), // 非 EUI 项目关闭此设置
                    new IncrementCompilePlugin(),
                ],
            };
        } else if (command === 'publish') {
            const outputDir = `bin-release/`;
            rimraf(path.join(params.projectRoot, outputDir));
            return {
                outputDir,
                commands: [
                    new CompilePlugin({ libraryType: 'release' }),
                    new ExmlPlugin('commonjs'), // 非 EUI 项目关闭此设置
                    new UglifyPlugin([
                        {
                            sources: ['main.js', 'config.js'],
                            target: 'main.min.js',
                        },
                    ]),
                ],
            };
        } else {
            throw `unknown command : ${params.command}`;
        }
    },
    mergeSelector: (path) => null,
    typeSelector: (path) => null,
};
```


其中保留最核心的几个插件

对于开发时的build, 只需要保留`IncrementCompilePlugin`(增量更新), 和ExmlPlugin(如果你用到了eui的话).

发布的时候删除了release文件夹(并没有采取默认的按增量的版本号来发布, 因为是用服务器上做CI的时候发布的, 你也可以不修改).


## 2 插件开发

首先所有的插件都是要实现`plugins.Command`这个接口. 按照提供的顺序执行.



`plugins.Command`其包含两个函数需要实现

`onFile(file: plugins.File): Promise<plugins.File>` 

和


`onFinish(context: plugins.CommandContext): Promise<void>`

其中`onFile`会对每个文件执行一遍(在前一个插件执行完后剩下的文件), 如果返回`null`的话, 表示不生成此文件(删除此文件), 否则返回file. 可以通过修改`file.path`改变其路径和`file.content`改变其内容


下一节我们来看一个简单的插件开发

## 3 文件hash版本号


```
export default class VersionedManifest implements plugins.Command {
    private manifestGameFiles: string[] = [];
    private manifestInitialFiles: string[] = [];

    async onFile(file: plugins.File): Promise<plugins.File> {
        if (file.extname === '.js') {
            const hash = crypto.createHash('md5').update(file.contents).digest('hex');
            file.path = file.path.substr(0, file.path.length - 3) + '.' + hash + '.js';
            if (file.basename.indexOf('main.min') >= 0) {
                this.manifestGameFiles.push(file.relative);
            } else {
                this.manifestInitialFiles.push(file.relative);
            }
        }
        return file;
    }

    async onFinish(context: plugins.CommandContext): Promise<void> {
        const manifest = JSON.stringify({
            initial: this.manifestInitialFiles,
            game: this.manifestGameFiles,
        });
        const hash = crypto.createHash('md5').update(manifest).digest('hex');
        await context.createFile(`manifest.${hash}.json`, new Buffer(manifest));
    }
}
```

首先, 我们在`onFile`的时候判断. 如果是js文件, 则hash其内容产出一个文件签名, 并添加到文件名中, 并缓存文件名.

最后, 在所有文件处理完成后, 将带有签名的js文件生成一个带版本号的`manifest.json`.

这样, 生成的文件就可以在CDN上做`200 缓存`了.

## 4. 资源版本控制

```
function hash(content: string | Buffer): string {
    return crypto.createHash('md5').update(content).digest('hex');
}

export default class VersionedResource implements plugins.Command {
    private defaultResJson: Buffer;
    private cachedResource: {[key: string]: Buffer} = {};

    async onFile(file: plugins.File): Promise<plugins.File> {
       if (file.basename.indexOf('default.res.json') >= 0) {
            this.defaultResJson = file.contents;
            return null;
        } else if (file.relative.startsWith('resource') && !file.relative.endsWith('.js')) {
            this.cachedResource[file.relative.substr(9)] = file.contents;
            return null;
        }

        return file;
    }

    async onFinish(context: plugins.CommandContext): Promise<void> {
        const json = JSON.parse(this.defaultResJson.toString());
        const resources = [];
        for (const resource of json.resources) {
            switch (resource.type) {
            case 'sheet': {
                let jsonPath = resource.url;
                const json = JSON.parse(this.cachedResource[jsonPath].toString());
                let pngPath = path.relative(
                    context.buildConfig.projectRoot,
                    path.resolve(jsonPath, '../', json.file),
                );
                const pngBuffer = this.cachedResource[pngPath];

                const pngHash = hash(pngBuffer);
                pngPath = pngPath.substr(0, pngPath.length - 4) + '.' + pngHash + '.png';

                json.file = pngPath.split('/').slice(1).join('/');
                const jsonBuffer = new Buffer(JSON.stringify(json));
                const jsonHash = hash(jsonBuffer);
                jsonPath = jsonPath.substr(0, jsonPath.length - 5) + '.' + jsonHash + '.json';

                context.createFile(path.join('resource', pngPath), pngBuffer);
                context.createFile(path.join('resource', jsonPath), new Buffer(jsonBuffer));
                resource.url = jsonPath;
                resources.push(resource);
                break;
            }
            case 'sound':
            case 'json':
            case 'image': {
                const file = this.cachedResource[resource.url];
                const h = hash(file);
                const ext = path.extname(resource.url);
                resource.url =
                    resource.url.substr(0, resource.url.length - ext.length) + '.' + h + ext;
                context.createFile(path.join('resource', resource.url), file);
                resources.push(resource);
                break;
            }
            default:
                throw new Error(`resource type: ${resource.type} is not supported`);
            }
        }

        json.resources = resources;
        const buffer = new Buffer(JSON.stringify(json));
        const h = hash(buffer);
        context.createFile(path.join('resource', 'default.res.' + h + '.json'), buffer);
    }
}
```

这个插件最主要的功能就是把所用到的资源也给加上一个版本号.

但是和`.js`文件稍微有些区别. 我们得首先获取到`default.res.json`里面的内容, 根据其用到的文件来进行哈希.

在处理资源的时候, 对于普通资源. 只需要简单的读取其文件内容, 哈希, 写回即可.

但是针对于`纹理集`(SpriteSheet)的时候, 情况稍微有一点特殊. `sheet`类型的资源所指向的是一个`json`文件, `json`文件再指向雪碧图的`png`. 所以需要先读取`json`, 修改其指向的`png`的文件后, 再整体哈希, 写回.


## 5 合并纹理图集

在h5游戏中, 为了更好更快的加载资源. 我们通常会将一起用到的图片资源资源打包成一个雪碧图.

白鹭提供了`Texture Merger`来生成雪碧图, 并用自带的`game`库来解析. 但是在实际的开发体验中, 这样的体验并不流畅. 每次美工修改一点点细节都需要重新打包资源, 替代.

工作中, 我们开发了这样一套插件. 所有的资源使用原生态(不打包). 但是在开发中, 在`default.res.json`中的引用时, 命名会有规则. 
> 例如, 原本要打包到`ui`这个包下面的资源`btn_close`, 直接命名为`ui.btn_close`. 事后打包时, 会将所有`ui.`开头的资源打包在一个`SpiteSheet`中

这样我们写一个helper funciton直接`getAsset('ui.btn_close')`, 开发环境中会直接取这个资源, 而生产环境中去`ui`这个`SpiteSheet`中取.


### 5.1 合并纹理图集插件

```
export default class MergeTexture implements plugins.Command {
    private imageBufferMap: Map<string, Buffer>;
    private resFile: Buffer;

    constructor(private output: string) {
        this.imageBufferMap = new Map<string, Buffer>();
    }

    async onFile(file: plugins.File): Promise<plugins.File> {
        if (file.extname === '.png') {
            this.imageBufferMap.set(file.relative, file.contents);
            return null;
        } else if (file.basename.indexOf('default.res.json') >= 0) {
            this.resFile = file.contents;
            return null;
        }
        return file;
    }

    async onFinish(pluginContext: plugins.CommandContext): Promise<void> {
        const components: {[key: string]: any} = {};
        const json = JSON.parse(this.resFile.toString());
        const resources: any[] = [];
        for (const res of json.resources) {
            if (res.name.indexOf('.') >= 0) {
                const url = path.join('resource', res.url);
                const image = this.imageBufferMap.get(url);
                const group = res.name.split('.')[0];
                const name = res.name.split('.').slice(1).join('.');
                components[group] = components[group] || [];
                components[group].push({
                    name,
                    image,
                });
            } else {
                resources.push(res);
            }
        }

        for (const group in components) {
            const resultSet = await merge(components[group]);
            await pluginContext.createFile(
                path.join('resource/', this.output, group + '.png'),
                resultSet.image,
            );
            await pluginContext.createFile(
                path.join('resource/',this.output, group + '.json'),
                new Buffer(JSON.stringify({
                    file: group + '.png',
                    frames: resultSet.frames,
                })),
            );
            const subkeys = [];
            for (const key in resultSet.frames) {
                subkeys.push(key);
            }
            resources.push({
                subkeys,
                url: path.join(this.output,  group + '.json'),
                type: 'sheet',
                name: group,
            });

            json.groups = json.groups.map((g) => {
                let keys = g.keys.split(',');
                const length = keys.length;
                keys = keys.filter(key => !key.startsWith(group));
                if (length !== keys.length) {
                    keys.push(group);
                }
                g.keys = keys.join(',');
                return g;
            });

        }

        json.resources = resources;
        const res = JSON.stringify(json, null, 2);
        await pluginContext.createFile('resource/default.res.json', new Buffer(res));

    }
}
```

这个插件涉及到的步骤比较多, 我们分开来讲.

首先在`onFile`里

```
if (file.extname === '.png') {
    this.imageBufferMap.set(file.relative, file.contents);
    return null;
} 
```
我们先把所有的`.png`文件内容拦截下来(因为后面要生成`纹理集`, *散装*的图片用不到了), 并将其内容存下来. 

```
} else if (file.basename.indexOf('default.res.json') >= 0) {
        this.resFile = file.contents;
        return null;
    }
    return file;
}
```

然后我们拦截下来`default.res.json`里面的内容


在所有文件遍历过后, 在`onFinish`里面.

```
const components: {[key: string]: any} = {};
const json = JSON.parse(this.resFile.toString());
const resources: any[] = [];
for (const res of json.resources) {
    if (res.name.indexOf('.') >= 0) {
        const url = path.join('resource', res.url);
        const image = this.imageBufferMap.get(url);
        const group = res.name.split('.')[0];
        const name = res.name.split('.').slice(1).join('.');
        components[group] = components[group] || [];
        components[group].push({
            name,
            image,
        });
    } else {
        resources.push(res);
    }
}
```
我们先根据`default.res.json`里面引用的资源按前缀名(如果有)来分组, 并取得其内容(在`onFile`里缓存的)

然后再在对每一个组的循环里面,

```
const resultSet = await merge(components[group]);
await pluginContext.createFile(
    path.join('resource/', this.output, group + '.png'),
    resultSet.image,
);
await pluginContext.createFile(
    path.join('resource/',this.output, group + '.json'),
    new Buffer(JSON.stringify({
        file: group + '.png',
        frames: resultSet.frames,
    })),
);
```

上述代码将每一个组里面的图片进行合并(合并的逻辑将在下面讲到), 并将这个`纹理集`(对应一个`json`和一个`png`)写入到硬盘上.

然后下面的一步, 将`default.res.json`里面对于'散件'的引用移除, 替换为对`纹理集`的引用.

```
const subkeys = [];
for (const key in resultSet.frames) {
    subkeys.push(key);
}
resources.push({
    subkeys,
    url: path.join(this.output,  group + '.json'),
    type: 'sheet',
    name: group,
});

json.groups = json.groups.map((g) => {
    let keys = g.keys.split(',');
    const length = keys.length;
    keys = keys.filter(key => !key.startsWith(group));
    if (length !== keys.length) {
        keys.push(group);
    }
    g.keys = keys.join(',');
    return g;
});
```

最后将`default.res.json`的内容写入磁盘


### 5.2 纹理集合并逻辑

上面的插件用到了一个函数`merge`, 用来将*散装*的图片打包成一个纹理集.

先上代码:
```
'use strict';

const jimp = require('jimp');
const layout = require('layout');
const co = require('co');

exports.default = co.wrap(function* (items) {
    const resultSet = {
        frames: {},
        image: null,
    };

    const layer = layout('binary-tree');
    for (const item of items) {
        const image = yield cb => jimp.read(item.image, cb);

        const sourceW = image.bitmap.width;
        const sourceH = image.bitmap.height;

        const xSpaces = Array(sourceW).fill(true);
        const ySpaces = Array(sourceH).fill(true);

        yield cb => image.scan(0, 0, sourceW, sourceH, (x, y, idx) => {
            const sum = image.bitmap.data[idx] +
                        image.bitmap.data[idx + 1] +
                        image.bitmap.data[idx + 2] +
                        image.bitmap.data[idx + 3];
            if (sum !== 0) {
                xSpaces[x] = false;
                ySpaces[y] = false;
            }
        }, cb);

        const offX = xSpaces.indexOf(false);
        const offY = ySpaces.indexOf(false);


        const frame = {
            name: item.name,
            image,
            offX,
            offY,
            sourceW,
            sourceH,
            w: xSpaces.lastIndexOf(false) + 1 - offX,
            h: ySpaces.lastIndexOf(false) + 1 - offY,
        };

        layer.addItem({
            meta: frame,
            width: frame.w,
            height: frame.h,
        });
    }

    const info = layer.export();
    resultSet.image = yield cb => new jimp(info.width, info.height, cb);

    for (const block of info.items) {
        const frame = block.meta
        const name = frame.name;
        const image = frame.image;

        delete frame.name;
        delete frame.image;

        frame.x = block.x;
        frame.y = block.y;

        resultSet.frames[name] = frame;

        yield cb => image.crop(frame.offX, frame.offY, frame.w, frame.h, cb);
        yield cb => resultSet.image.composite(image, frame.x, frame.y, cb);
    }

    resultSet.image = yield cb => resultSet.image.getBuffer(jimp.MIME_PNG, cb);

    return resultSet;
});
```

其中, `jimp`这个包是一个**纯js**的图片操作库, `layout`是一个图片打包排序的库, 以空间利用率最大的方式将所有*散装*图片组合到一个图片上.


```
const sourceW = image.bitmap.width;
const sourceH = image.bitmap.height;

const xSpaces = Array(sourceW).fill(true);
const ySpaces = Array(sourceH).fill(true);

yield cb => image.scan(0, 0, sourceW, sourceH, (x, y, idx) => {
    const sum = image.bitmap.data[idx] +
                image.bitmap.data[idx + 1] +
                image.bitmap.data[idx + 2] +
                image.bitmap.data[idx + 3];
    if (sum !== 0) {
        xSpaces[x] = false;
        ySpaces[y] = false;
    }
}, cb);

const offX = xSpaces.indexOf(false);
const offY = ySpaces.indexOf(false);
```

这一段代码是为了将图片外圈的白边(白色且透明)给识别出来. 其中`offX`和`offY`表示有效内容(非白边)开始的位置.

`frame`的格式为白鹭的`Texture Merger`生成的格式, 为了能无缝使用`game`库加载`纹理集`, 变沿用了这套格式.

将计算好的数据添加到`layer`上, export出来后的数据便是排好位置的坐标值了.

`yield cb => image.crop(frame.offX, frame.offY, frame.w, frame.h, cb);`这样将原始的*散装*图片按照之前计算的白边距离给裁剪掉(`w`和`h`是去掉白边后的宽高)

`yield cb => resultSet.image.composite(image, frame.x, frame.y, cb);`将*散装*图片内容复制到最后的`纹理集`大图上

这样就大工告成啦.


## 6 总结

当然工作中还开发了一些其他的插件, 限于篇幅和文章重心的缘由就没有列出来了. 文章中提到的几个插件算是比较核心且必备的. 

在熟悉开发插件的原理以及流程后, 你也可以开发出适合自己的工作流程的插件啦.


## 打个广告

下面是自己弄的个小公众号, 用来发发文章, 感想什么的.


![](https://user-gold-cdn.xitu.io/2018/3/28/1626854a84cd5131?w=258&h=258&f=jpeg&s=26668)
