---
title: JavaScript Async/Await
date: 2016-09-17 20:10:58
tags: 
    - JavaScript
    - ES6
    - fetch
---
有时候我们写代码的时候会使用回调函数`callbacks`,然后在回调中继续回调，俗称回调地狱（callback hell）。 例如, 使用 `GitHub API`实现下面功能的例子:

1. 获取github代码库的README.md
2. 将README.md从Markdown转换为HTML
3. 将转换得到的HTML保存为文件

由于API调用和文件写入都涉及异步`I / O`操作，为实现功能，我们至少需要3次嵌套回调：
```javascript
function getReadmeInHtml (repo) {
  getReadme(repo, (err, readme) => {
    convertMdToHtml(readme, (err, html) => {
      saveHtml(html, (err, filename) => {
        // HTML file is now saved
      })
    })
  })
}
```
虽然多年来我们已经适应了这种模式, 但说真的对刚入门的我们来说这很难理解，维护起来也很麻烦。如果用过Python, Java等语言，你可能会怀念这些语言中使用`return`，`try / catch`语句是多么的简单。

幸运的是，ES2015 (ES6)来了，并带来了[Promises](https://developer.mozilla.org/en/docs/Web/JavaScript/Reference/Global_Objects/Promise)，它给我们提供了一个处理异步问题的方式。这样一来，我们可以以`Promises`为基础重构我们的异步操作。

下文将会经常提到`Promises`的概念，如果你还不太了解`Promises`，在阅读之前建议先看一下 [JavaScript Promises: Plain and Simple](https://coligo.io/javascript-promises-plain-simple/)。

## Async Functions

你可能想知道“`async`和`Promises`是怎么联系到一起的？”为了回答这个问题，让我们通过使用`async / await`语法改写上述`getReadmeInHtml (repo)`来说明：
```javascript
async function getReadmeInHtml (repo) {
  const readme = await getReadme(repo)
  const html = await convertMdToHtml(readme)
  const filename = await saveHtml(html)
}
```
像上面写的这样，`async / await`版本的代码看起来很像其他的同步代码。但是，有几点需要注意：

1. 一定要在`async`关键字后使用`await`关键字
2. `await`关键字会导致当前函数（`getReadmeInHtml`）停止执行，直到等待中函数（`getReadme`，`convertMdToHtml`，`saveHtml`）返回`Promises`才会继续执行下一个语句

看到上面第二点，自然也就明白了`async`和`Promises`有何关联了。那就是我们使用`await`等待的函数必须返回一个`Promises`。

在上面的例子中，我们等待`getReadme(repo)`函数必须返回一个`Promise`。一旦`Promise`得到`resolved`，`resolved`的值会赋给变量并执行下一条语句，就有了：`convertMdToHtml (readme)`等方法执行起来就像执行同步代码一样！

让我们来看看`getReadme (repo)`使用`Promises`的实现：
```javascript
function getReadme (repo) {
  let requestUrl = `https://api.github.com/repos/${repo}/readme`

  // fetch returns a Promise
  return fetch(requestUrl, {method: 'GET'}).then((res) => {
    // get the response in JSON format
    return res.json()
  }).then((data) => {
    // decode the base64 string returned by GitHub's API
    return atob(data.content)
  })
}
```
由于`fetch`会返回一个`Promise`，我们可以使用`await`关键字来等待异步执行的结果，此时`getReadmeInHtml`将会暂停执行，直到在`getReadme`的请求完成。

重要的事再说一遍：任何你希望使用的`await`关键字的方法都必须返回一个`Promise`。

## Async Functions Return Promises

A point that is not immediately obvious about async functions is that they always return a Promise. For instance, an async function that returns a value is actually returning a Promise which resolves to that value. The following are equivalent:
```javascript
async function greet (name) {
  return 'hello ' + name
}
```
and
```javascript
async function greet (name) {
  return Promise.resolve('hello ' + name)
}
```
This is because the async keyword will result in the return value being wrapped in a Promise object implicitly.

If you want to reject the Promise you can throw an error or explicitly reject a Promise:
```javascript
async function greet (name) {
  if (name)
    return 'hello ' + name
  throw new Error('A name must be provided')
}
```
is the same as:
```javascript
async function greet (name) {
  if (name)
    return Promise.resolve('hello ' + name)
  return Promise.reject('A name must be provided')
}
```
The key point from this section is that: async functions always return a Promise.

## Error Handling in Async Functions

One of the great things about async functions is that you can use the already familiar `try / catch` construct for handling errors thrown by an asynchronous operation. You no longer need to redundantly check for errors, like so:
```javascript
asyncOperation1((err, res) => {
  if (err) console.log(err)
  asyncOperation2((err, res) => {
    if (err) console.log(err)
  })
})
```
Instead, you can wrap await statements in `try / catch` blocks to capture and handle the errors from the await-ed promises just as you would with synchronous code:
```javascript
async function getReadmeInHtml (repo) {
  try {
    const readme = await getReadme(repo)
    const html = await convertMdToHtml(readme)
    const filename = await saveHtml(html)
  } catch (err) {
    console.log('An error has occurred: ', err)
  }
}
```
It's that simple! If any of the await-ed functions results in a rejected Promise, the error will be caught and handled accordingly.

It's worth noting that if a function returning a Promise is rejected, the error will be swallowed silently in some browsers/environments. For this reason, it's best to wrap await-ed functions in `try / catch` blocks to avoid having errors go unhandled.

## Async Functions and Loops

You might find yourself commonly using asynchronous functions in loops. For example, assume you wanted process a list of READMEs for GitHub repos in the order they are provided:

Read a file containing a list of GitHub repos
For each repo call getReadmeInHtml
With async functions, we can easily achieve this:
```javascript
async function processReadmesFromFile (filename) {
  // read the list of repos from the file into an array
  const repos = await readRepos(filename)

  // we can use the for..of syntax to loop through the array of repositories
  for (let repo of repos) {
    // wait until the README is done processing before going onto the next
    await getReadmeInHtml(repo)
  }
}
```
Now each repository's README is grabbed, converted, and saved in the order specified in the file. We are able to perform these steps sequentially thanks to the await getReadmeInHtml(repo) statement, which waits until a repository's README is processed before going onto the next iteration of the for loop and processing the next README.

This is because any async function always returns a Promise as we mentioned earlier. Therefore the await clause in the example above causes the loop to suspend until the Promise returned from the getReadmeInHtml function resolves.

## Concurrency with Promise.all

Let's revisit the example in the previous section where we read a file and processed the READMEs one at a time (sequentially). You may not care about the order in which the READMEs are processed and instead you want to process them as quickly as possible.

To avoid wasting time by waiting for each I/O operation to complete, we can use Promise.all to execute these requests concurrently and rewrite the example in the previous example to look like this:
```javascript
async function processReadmesFromFile (filename) {
  // read the list of repos from the file into an array
  const repos = await readRepos(filename)

  // use `Promise.all` to wait until all promises have been resolved
  await Promise.all(repos.map((repo) => getReadmeInHtml(repo)))

  // all the READMEs have been processed at this point, you can access
  // the files or do whatever you need here
}
```
Let's take a second to break down the following statement from the example above:
```javascript
await Promise.all(repos.map((repo) => getReadmeInHtml(repo)))
```
First we are using map method to create a new array of Promises returned by the getReadmeInHtml function. We then feed this array of promises to the Promise.all method which returns a Promise that resolves when all the Promises in the array have resolved.

Since Promise.all itself returns a Promise, we can use the await keyword to wait until all the promises have resolved before moving on to executing the next statement or returning from the function.

If it's easier to read, we can break down that statement into 2 lines, like so:
```javascript
await Promise.all(repos.map((repo) => getReadmeInHtml(repo)))

// is the same as:

// create an array of promises by calling `getReadmeInHtml` on each of the repos
let promises = repos.map((repo) => getReadmeInHtml(repo))

// use Promise.all to let us know when ALL the promises have resolved
await Promise.all(promises)
```
## Using Async Functions Today

There are many ways you can start using async functions in your applications and libraries today:

- Using Google's Traceur transpiler
- Facebook's Regenerator
- Babel's Async to generator transform
I'll show you a quick and easy way to use babel-cli and the transform-async-to-generator plugin to transpile your code which uses async functions into generators - an ES6 feature implemented in Node v6 and most modern browsers.

First, in an empty directory, initialize a package.json with the following command:
``` bash
mkdir node-async && cd node-async
npm init -y
```
Next we need to install babel-cli and babel-plugin-transform-async-to-generator:
```bash
npm install --save-dev babel-cli babel-plugin-transform-async-to-generator
```
Once the dependencies have been installed, we need to configure Babel to use the plugin we just installed. You can do that through a .babelrc file or directly in the package.json, like so:
``` json
{
  "name": "node-async",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "start": "babel-node index.js",
    "build": "babel index.js --out-file build.js"
  },
  "keywords": [],
  "author": "",
  "license": "ISC",
  "devDependencies": {
    "babel-cli": "^6.11.4",
    "babel-plugin-transform-async-to-generator": "^6.8.0"
  },
  "babel": {
    "plugins": [
      "transform-async-to-generator"
    ]
  }
}
```
You will also notice 2 scripts defined:

npm run start will automatically compile and run your Node.js code using babel-node, which is handy when developing
npm run build will output the compiled code to a file called `build.js`, ready to be run using `node build.js`
And that's it! Try it out for yourself by creating a simple async function and saving it in index.js:
```javascript
async function test () {
  return 'Testing async functions'
}

test().then(val => console.log(val))
```
then running npm run start.