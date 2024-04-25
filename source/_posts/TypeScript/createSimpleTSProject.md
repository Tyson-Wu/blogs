---
title: 创建最简单的TS项目
date: 2024-4-19 17:01:34
categories:
- [TS]
tags:
- TS
---

- 创建文件夹`TSProjectDemo`
``` bash
[TSProjectDemo]/
```
- 进入文件夹，打开cmd控制台
- 在控制台输入命令`npm init -y`,会创建`package.json`文件,
``` bash
[TSProjectDemo]/
    package.json
```
``` json
{
  "name": "TSProjectDemo",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "keywords": [],
  "author": "",
  "license": "ISC"
}
```

- 在控制台可以执行`npm run test`来执行上面的`test`脚本

- 在目录下再创建一个`Src`子目录，ts脚本后面都放这里面
``` bash
[TSProjectDemo]/
    package.json
    [Src]/
```
- 在控制台输入`tsc --init`，会创建`tsconfig.json`
``` bash
[TSProjectDemo]/
    package.json
    tsconfig.json
    [Src]/
```
- 修改为
``` json
{
    "compilerOptions":{
        "target": "ES6",
        "module": "NodeNext",
        "esModuleInterop": true,
        "strict": false,
        "outDir": "output",
        "moduleResolution": "NodeNext"
    },
    "include": [
        // 表示编译Src目录下的所有ts文件
        "./Src/**/*.ts"
    ],
    "exclude": [
        "node_modules"
    ]
}
```

- 进Src目录，创建一个测试脚本`main.ts`
``` bash
[TSProjectDemo]/
    package.json
    tsconfig.json
    [Src]/
        main.ts
```
``` ts
// main.ts
const message : string = "test log .... "
console.log(message)
```

- 回到根目录，然后运行命令`tsc`,会编译出js脚本
``` bash
[TSProjectDemo]/
    package.json
    tsconfig.json
    [Src]/
        main.ts
    [output]/
        main.js
```

- 运行命令`node ./output/mian.js`可以执行代码
- 也可以修改`package.json`中的test脚本，这样就可以使用`npm run test`运行
``` json
{
  "name": "TSProjectDemo",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "test": "node ./output/mian.js"
  },
  "keywords": [],
  "author": "",
  "license": "ISC"
}
```
- 上面是最简单的ts工程了，但是我们开发的时候可能需要外部库，比如nodejs相关的。
- 比如`require`指令，或者`fs`文件库。这时候我们要安装`npm install --save-dev @types/node`
- 安装完成后，脚本里面可以引入`fs`包，import和require两种方式都可以
``` ts
// main.ts
import * as fs from "fs"
const fsLib = require("fs")
const messa : string = "test log .... "
console.log(messa)
```

- 假如我想读写xml文件，可以安装`npm install xml2js`,`xml2js`可以用来读写xml,实现js对象和xml数据进行转换
``` ts
// main.ts
import { rejects } from "assert";
import * as fs from "fs";
import { resolve } from "path";
import * as xml2js from "xml2js";

function readXmlFile(filePath: string):Promise<any>{
    return new Promise((resolve, rejects)=>{
        fs.readFile(filePath, 'utf-8', (err, data)=>{
            if(err){
                rejects(err);
            }
            else{
                xml2js.parseString(data, (err, result)=>{
                    if(err){
                        rejects(err);
                    }
                    else{
                        resolve(result);
                    }
                });
            }
        });
    });
}

function writeXmlFile(filePath:string, data:any):Promise<void>{
    return new Promise((resolve, reject)=>{
        
        const buildOption : xml2js.BuilderOptions = {
            xmldec:{version:"", encoding:"utf-8",standalone:undefined}
        }

        const builder = new xml2js.Builder(buildOption);
        const xml = builder.buildObject(data);
        fs.writeFile(filePath, xml, (err)=>{
            if(err){
                reject(err);
            }
            else{
                resolve();
            }
        });
    });
}

async function main() {
    try{
        // 读xml文件
        const xmlData = await readXmlFile("jj.xml");
        console.log(xmlData);

        // xmlData 是一个js对象，根节点是component, $表示xml属性
        xmlData.component.$.attribute = "new value --- 100"

        // js对象转换为xml
        await writeXmlFile("jj2.xml", xmlData)
        console.log("xml file write")
    }
    catch(err){
        console.error("err: ", err);
    }
}
main()
```

- 上面的`xml2js`库是没有代码提示的，我们可以额外安装`npm install --save-dev @types/xml2js`, 这样在vscode中就可以看到代码提示了

- 上面安装后，`package.json`文件会变成
``` json
{
  "name": "TSProjectDemo",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "test": "node ./output/main.js"
  },
  "keywords": [],
  "author": "",
  "license": "ISC",
  "devDependencies": {
    "@types/node": "^20.12.7",
    "@types/xml2js": "^0.4.14"
  },
  "dependencies": {
    "xml2js": "^0.6.2"
  }
}
```

基于上面的流程，我们就可以开发一些工具了
