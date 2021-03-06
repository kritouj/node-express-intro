# Unit 1-C Lecture


This lecture will cover the following topics:
 * The Node.js event loop
 * Callbacks
 * Promises
 * Timers
 * Network requests
 * Async and Await

It is recommended to follow along by making a project directory for this lecture and running the code samples:

```
mkdir lecture1C
cd lecture1C
npm init
```

## The Node.js Event Loop
The Node.js event loop is what allows Node.js to perform non-blocking I/O operations despite the fact that JavaScript is single-threaded.

Although JavaScript is single threaded, Node.js can offload operations to the system kernel when it needs to process things in the background.

Node.js usually does this when you try to read or write from a file, make a network request, or schedule a function to run later with a timer. Once one of those requests completes, the kernel tells Node.js, then Node.js places a callback function on a queue within the event loop to be processed. If multiple requests finish around the same time, they will pile up on the queue and will be processed in the order they came in. 



The Node.js event loop has multiple phases and processes different callbacks at different phases of the loop:
1. timers - executes callbacks set by setTimeout() or setInterval()
2. pending callbacks - executes I/O callbacks (e.g. file reading or network request)
3. idle/prepare - preparing for next phase
4. poll - wait for "some time" and poll for new I/O callbacks
    1. If there are any I/O callbacks queued up, execute them all 
    2. If there aren't any I/O callbacks queued up, keep polling for a bit before moving to the Check phase. If new callbacks from timers (the first phase) have been scheduled, move onto the Check phase instead of polling
5. check - executes callbacks set by setImmediate()
6. close callbacks - executes callbacks set by closing events such as process.exit() or socket.on('close',fn)


To read more about the Node.js Event Loop read here:
https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/


## Callbacks

As we mentioned earlier, callback functions are the sequences of code that are put on queues within the event loop to be executed later after a particular event has finished.

Callback functions are usually passed in as arguments to other functions that trigger an asynchronous event.

This is true for the fs.readFile() function that we covered in the last lecture:

```js

fs.readFile('file.txt', 'utf-8', (err, data) => {
  if (err) {
    //handle error
    return;
  }
  //handle data
  console.log(data);
  //prints Hello World
});


```

The last argument to fs.readFile() is a callback function that will be run once the system finished reading the file. The system will also load the `data` argument that is passed to the callback with the appropriate data from the file that was just read.

**Best Practice:** It is important to handle that data within the callback. The data won't exist outside the scope of the callback, and you can't move it out by passing it to a variable that has a wider scope.

For example, this is a common mistake that happens:

```js
let fileData = 'old value';

fs.readFile('file.txt', 'utf-8', (err, data) => {
  if (err) {
    return;
  }
  fileData = data; //you may think that fileData now has the data from the file and can be accessed outside of the callback...
  console.log(fileData); //it seems like the the variable has the correct data...
  //prints the correct file data
});

console.log(fileData); //however, this still prints the original value of old value
//prints old value

```

In the above example, the fileData variable had global scope and inside the callback function we tried to store the file data within it. Later, when we console.log the fileData variable, it still has its original value of nothing, even though if we check the console, we see that the correct file data was printed out.

You may think that the event loop should trigger the callback and that the callback will load the data in the fileData variable, but this is not the case.

The reason has to do with the timing of synchronous and asynchronous portions of the code.  Before the first loop of the event loop even occurs, all function declarations and expressions in the Node.js file are processed in order from top to bottom. If an asynchronous function is encountered, Node.js will kick off an I/O process but won't queue up the callback until the asynchronous process is over. Thus in the above example, let fileData = "old value" and console.log(fileData) happens almost instantaneously while the readFile callback is processed after the file finishes reading, which could take a couple miliseconds.

If you look at the order of the console.logs, it starts to make sense as the correct file data was printed after the old value was logged.

```
old value
correct file data
```

## setTimeout and setInterval

### setTimeout()
setTimeout() is used to schedule a function to run after a specific amount of time. The callback function is processed in the Timer phase of the event loop.


setTimeout returns an id for the timer, that can be used later to cancel the scheduled callback early if needed with clearTimeout().


```js
function myFunction(){
    console.log("hi")
}

let id = setTimeout(myFunction,1000)//schedules myFunction to run 1000 ms later

clearTimeout(id) //cancels the scheduled timer

```


### setInterval()
setInterval() is used to schedule a function to rerun repeatedly with a specified interval time. The callback function is processed in the Timer phase of the event loop.

setInterval returns an id for the timer, that can be used later to cancel the interval later.


```js
function myFunction() {
  console.log('hi');
}

let id = setInterval(myFunction, 1000); //schedules myFunction to run every 1000ms

clearInterval(id); //cancels the scheduled interval


```

### setImmediate

setImmediate() is used to schedule a function to run once the current Poll phase completes. The callback function is processed during the Check phase of the event loop.


setImmediate returns an id for the timer, that can be used later to cancel the interval later.


```js
function myFunction() {
  console.log('hi');
}

let id = setImmediate(myFunction, 1000); //schedules myFunction to run after the current Poll phase completes

clearImmediate(id); //cancels the scheduled timer

```

## Knowledge Check 1

```js
function myPrint() {
  console.log('hi');
}
setTimeout(myPrint, 0);
console.log('bye');


```

If the code shown above is run, which of the following prints first?
```
A. hi
B. bye
```


## Promises

Promises were created to avoid what programmers refer to as "callback hell", which is when nested callbacks become increasingly harder to read as they shift more towards the right of the page. 

Example of callback hell taken from callbackhell.com:
```js
fs.readdir(source, function (err, files) {
  if (err) {
    console.log('Error finding files: ' + err);
  } else {
    files.forEach(function (filename, fileIndex) {
      console.log(filename);
      gm(source + filename).size(function (err, values) {
        if (err) {
          console.log('Error identifying file size: ' + err);
        } else {
          console.log(filename + ' : ' + values);
          aspect = values.width / values.height;
          widths.forEach(
            function (width, widthIndex) {
              height = Math.round(width / aspect);
              console.log(
                'resizing ' + filename + 'to ' + height + 'x' + height
              );
              this.resize(width, height).write(
                dest + 'w' + width + '_' + filename,
                function (err) {
                  if (err) console.log('Error writing file: ' + err);
                }
              );
            }.bind(this)
          );
        }
      });
    });
  }
});


```
In the above example, you also have to be careful how you name your callback arguments so that you don't accidentally take over an argument name that you needed from an earlier callback.

Promises solve the problem of callback hell by introducing a new syntax for chaining callbacks after eachother and for handling errors.

**Additional Context:** Promises are incredibly important to fully understand because all requests to external APIs will be returned as Promises.

### Promise States

Promises are containers for values that are not yet available yet but may eventually become available once an asynchronous request successfully completes or fails. Promises also allow you to define what to do in either case if your asynchronous request was successful or if it failed.


A Promise is always in one of three states:
* pending: initial state, neither fulfilled nor rejected.
* fulfilled: meaning that the asynchronous operation was completed successfully.
* rejected: meaning that the asynchronous operation failed.

![](./images/promises.png)

### Handling Promises with .then() and .catch()

To handle a promise after running it, you need to attach a .then() and .catch() method.

The .then() method of a Promise takes in a callback function that handles Promises that become fulfilled. On the other hand, the .catch() method of a promise takes in a callback function that handles Promises that become rejected.

```js
//for demonstration purposes, this Promise asynchronous returns an error if the input is less than 0, and asynchronously returns the original input if the input is above 0
function asyncRequest(val) {
  if (val < 0) {
    return Promise.reject('error'); //Promise.reject creates a Promise that instantly rejects and puts a callback on the event loop queue
  } else {
    return Promise.resolve(val); //Promise.reject creates a Promise that instantly becomes fulfilled and puts a callback on the event loop queue
  }
}

asyncRequest(5)
  .then((result) => {
    //handle result
    console.log(result);
  })
  .catch((err) => {
    //handle error
    console.log(err);
  });

```

If you run the above code, you will see 5 printed in the console, meaning that the Promise was successful and that the .then() handler was able to retrieve the fulfilled result of the Promise.

If you rerun the above code with asyncRequest(-5) instead, you'll see "error" printed to the console, meaning that the Promise was rejected and that the .catch() handler retrieved the rejected result of the Promise.


### Handling Promises with .then(onSuccess, onRejected)

An alternative way to handle Promises without .catch() is to provide a second argument to the .then() method. The first argument will be the callback for fulfilled Promises, while the second will be the callback for rejected Promises.

```js
//for demonstration purposes, this Promise asynchronous returns an error if the input is less than 0, and asynchronously returns the original input if the input is above 0
function asyncRequest(val) {
  if (val < 0) {
    return Promise.reject('error'); //Promise.reject creates a Promise that instantly rejects and puts a callback on the event loop queue
  } else {
    return Promise.resolve(val); //Promise.reject creates a Promise that instantly becomes fulfilled and puts a callback on the event loop queue
  }
}

asyncRequest(5).then(
  (result) => {
    //handle result
    console.log(result);
  },
  (err) => {
    //handle error
    console.log(err);
  }
);

```

### Chaining Promises

An important feature about Promises is that you can chain multiple asynchronous requests together instead of building a bunch of nested callbacks. The way it works is if you return a Promise within a .then() method, the entire return value of the .then() method becomes the new Promise that you just returned inside the then() method. From there, you can chain on another .then() method to handle the result once it is fulfilled.

Imagine that there are three asynchronous functions called getRandomNumber(), getNameFromNumber() and getAgeFromName() that do the following:

* getRandomNumber() - asynchronously returns a random number
* getNameFromNumber - takes in a number and asynchronously returns a name
* getAgeFromName - takes in a name and asynchronously returns an age 


If we wanted to first call getRandomNumber() to get a number, then call getNameFromNumber() to get a name from that number, and then lastly call getAgeFromName() on the returned name to get an age then we would have to sequence them correctly.

If they were normal synchronous functions then it would be simple and would look like this:

```js
var number = getRandomNumber();
var name = getNameFromNumber(number);
var age = getAgeFromName(name);
```

However, since the functions are asynchronous, the number variable may be undefined by the time getNameFromNumber() is called, and the name variable may be undefined by the time getAgeFromName() is called.

Thus, we need to do the following to sequence them correctly, which we can do by chaining Promises together:

Here's an example:

```js
//getRandomNumber() returns a promise containing a random number
getRandomNumber()
  .then((result) => {
    console.log(result); // 42
    return getNameFromNumber(result); //returns a promise containing a string representing a name
  })
  .then((result) => {
    console.log(result); //"Bob"
    return getAgeFromName(result); //returns a promise containing a number representing an age
  })
  .then(function (result) {
    console.log(result); //21
  })
  .catch(function (error) {
    console.log(error);
  });

```

If any of the then() functions return a rejected Promise then the catch() method will handle the rejected result.

The end result is a much more readable way to chain together asynchronous requests.


### Handling multiple Promises with Promise.all()

The Promise.all() method is useful if you need to wait for multiple Promises to all complete before handling them.

The method Promise.all() method takes in an array of Promises and then waits for them to all to resolve. Once they have all finished resolving, an array of results can be obtained by using the then() method. If any of the Promises reject, then the Promise.all() method will return the first rejected Promise.

Notice how the results of all of the Promises are accessible as an array within the .then() callback:
```js
//these Promises fulfill immediately
var promise1 = Promise.resolve('hello');
var promise2 = Promise.resolve({ age: 2, height: 188 });
var promise3 = 42; //normal values work with Promise.all() too

Promise.all([promise1, promise2, promise3])
  .then((result) => {
    console.log(result); //logs the array ["hello",{age:2,height:188},42]
  })
  .catch(function (error) {
    console.log(error);
  });

```

**Additional Context**: Promise.all also is useful when you want to keep the results of Promises in the same order in the results array as the initial Promise requests.

### Handling multiple Promises with Promise.race()

The Promise.race() method takes in an array of promises and takes the result of the promise that rejects or resolves the fastest. Like normal promises, the then() and catch() methods are used to retrieve the results of the fastest promise.

The Promise.race() method can be used to choose the quickest source when there are two similar sources of the same data.

Notice how the Promise.race() method is used to take the result of the faster promise:

```js
var promise1 = new Promise(function (resolve, reject) {
  setTimeout(function () {
    resolve('finished in two seconds');
  }, 2000); //returns a resolved promise after 2 seconds
});

var promise2 = new Promise(function (resolve, reject) {
  setTimeout(function () {
    resolve('finished in five seconds');
  }, 5000); //returns a resolved promise after 5 seconds
});

Promise.race([promise1, promise2])
  .then((result) => {
    console.log(result); // logs "finished in two seconds" because promise1 resolved first
  })
  .catch((err) => {
    console.log(error);
  });


```

## Knowledge Check 2

If a Promise is successful, how do you handle the Promise result?

```
A. Promise.result()
B. Promise.then()
C. Promise.resolve()
D. Promise.success()
```


## Network Requests

Now that we know how to handle Promises, we have all the knowledge we need to understand how to make a network request.

**Best Practice**: Node.js comes with the core http module which can be used to make network requests; however, that module is somewhat difficult to use and there are third party modules such as `node-fetch` and `axios` that work better for network requests.

### Fetch

We are going to use the *node-fetch* module because it allows us to use the same Fetch API that is used on the browser when making network requests in JavaScript.

To add *node-fetch* as a dependency to your Node.js project, run the following:

```
$ npm install node-fetch
```

To import it into your code, use require():

```js
const fetch = require('node-fetch');

```

#### Basic Fetch Usage


The fetch() method takes in a URL endpoint as a string that specifies where to make a network request.

```js
fetch('https://api.github.com/users/github')
  .then((res) => res.json())
  .then((json) => console.log(json));

```

This code returns the following JSON data:

```
{
  login: 'github',
  id: 9919,
  node_id: 'MDEyOk9yZ2FuaXphdGlvbjk5MTk=',
  avatar_url: 'https://avatars1.githubusercontent.com/u/9919?v=4',
  gravatar_id: '',
  url: 'https://api.github.com/users/github',
  ...
}
```

The network request will initially return a Response stream, which is sort of like a binary of the network request data. We need to convert the response stream into either JSON data, text data, or blob data for it to be useful to us. 

We can convert the response stream using the following functions:
 * json() - for json data
 * text() - for text data
 * blob() - for blob data

**Best Practice**: In the second line of the code example, we take the response stream and return a json object to be chained on to the next .then() call. We are using arrow function syntax in order to be more concise with our code. (remember, no curly braces means you are returning whatever is to the right of the arrow). It is important to pick the right conversion function; otherwise, the stream data may not match. We know we want to use json() because we know the API returns json data.

In the third line of code, the json object from the previous .then() method has been passed down as an argument because of Promise chaining. We then log the json object and are free to do whatever we want with the object.


### Fetch Init Object

The above example is great if we want to make a simple GET request, but if we want to configure our network request, we need to pass in an init object as the second parameter to fetch()


```js
const data = { value: 123 };

fetch('https://httpbin.org/post', {
  method: 'post',
  body: JSON.stringify(data),
  headers: { 'Content-Type': 'application/json' },
})
  .then((res) => res.json())
  .then((json) => console.log(json));
```

This code returns the following JSON data:

```
{
  args: {},
  data: '{"value":123}',
  files: {},
  form: {},
  headers: {
    Accept: '*/*',
    'Accept-Encoding': 'gzip,deflate',
    'Content-Length': '13',
    'Content-Type': 'application/json',
    Host: 'httpbin.org',
    'User-Agent': 'node-fetch/1.0 (+https://github.com/bitinn/node-fetch)',
    'X-Amzn-Trace-Id': 'Root=1-5f766b9d-4439bab136ce020f57e8ee39'
  },
  json: { value: 123 },
  origin: '73.252.202.73',
  url: 'https://httpbin.org/post'
}
```

In this example, we configure the HTTP method to be POST and also specify a body to go along with our request as well as the request headers.

The method property is used to determine whether you are sending a GET, POST, PUT or DELETE request. It is set to GET by default. 

**Best Practice**: The body property is used to send a payload along with our request. It only works with POST and PUT requests. If we want to pass in JSON data, we must convert it to a JSON string with JSON.stringify before sending it; otherwise, it won't serialize correctly. Forgetting to stringify your JSON body is a common mistake that developers make.

The header property defines the request headers using key value pairs.

### Axios

If you find that always having to convert a stream to a json object with .json() to be annoying, you can use an alternative module called axios:


You can read up on the official axios documentation for more information:
https://www.npmjs.com/package/axios

Some of the advantages of using axios are that it will automatically convert the stream into the correct data type(json, text, or blob), and it will also automatically stringify your payload data for you.


To add axios as a dependency:

```
$ npm install axios
```

To import axios into your code:

```js
const axios = require('axios');
```

## Knowledge Check 3

If you are using node-fetch, which method do you use to convert a stream response to a json response?

```
A. .json()
B. .text()
C. .blob()
D. .response()
```


## Async and Await

Async and Await is a way to rewrite Promises in a way that looks more synchronous.

So instead of writing a Promise with .then() and .catch():

```js

function myFunction() {
  fetch('https://api.github.com/users/github')
    .then((res) => res.json())
    .then((json) => {
      //do something with json
    });
}

```

You can write it more normally like this:

```js

async function myFunction() {
  try {
    let response = await asyncRequest(5);
    let json = await response.json();
    //do something with json
  } catch (err) {
    //handle error
  }
}
//OR with arrow function syntax

const myFunction = async () => {
  try {
    let response = await asyncRequest(5);
    let json = await response.json();
    //do something with json
  } catch (err) {
    //handle error
  }
};


```

The *await* keyword will make it seem as though Node.js is waiting for the asynchronous request to finish before passing the result into the json variable. Behind the scenes, everything is happening exactly as before with Promises. 

*Await* just makes it so we can handle asynchronous results in a way that looks synchronous instead of in a callback.

Instead of using .catch(), however, you have to wrap your *await* request in a try-catch block.

Also, you can only use the *await* syntax within a function that has been labeled as *async*. You can not use the *async and await* syntax at the top level code since it has to be wrapped in an async function declaration.

Lastly, any function that has been declared *async* will return a Promise instead of a normal value.

**Best Practice**: Using async and await when making calls to databases makes code much easier to read

## Knowledge Check 4

If you use await within a function, what keyword do you need to add before the function?

```
A. await
B. callback
C. async
D. try
```

# Unit 1C Lab

## Lab Overview

This lab will be an exercise in making network requests to other APIs using Node.js. 

You will be implementing a couple of functions that make network requests to the Star Wars API. Those functions should return Promises containing data from the Star Wars API.

Here's a link to the Star Wars API documentation:
https://swapi.dev/documentation

There will be test cases already set up to verify that your functions are returning Promises that contain the right data.


## Lab Starter Code
You will be given a starter node.js project that has `jest`, `axios`, and `node-fetch` installed. There will be some function definitions in `src/starwars.js` that you will need to fill out. Your test cases are located in `tests/app.test.js`

You can get the starter code here:
https://github.com/flatiron-school/node-express-intro/tree/main/Labs/Lab-1C-Starter

To test out your lab, first, download all the lab dependencies with:

```
$ npm install
```

Then run your tests with:

```
$ npm run test
```

If all your tests pass you should see the following message:
```
 PASS  tests/app.test.js (8.94 s)
  ✓ getPeople (1728 ms)
  ✓ getFilm (1441 ms)
  ✓ getAllFilmTitles (1619 ms)
  ✓ getFilmCharacters (3432 ms)

Test Suites: 1 passed, 1 total
Tests:       4 passed, 4 total
Snapshots:   0 total
Time:        9.003 s
Ran all test suites.
```

## Lab Solution

You can view the lab solution here:
https://github.com/flatiron-school/node-express-intro/tree/main/Labs/Lab-1C-Solution

# Knowledge Check Answers

## Knowledge Check 1

```js
function myPrint(){
    console.log("hi")
}
setTimeout(myPrint,0)
console.log("bye")

```

If the code shown above is run, which of the following prints first?

```
A. hi
B. bye

```

Answer is B.

## Knowledge Check 2

If a Promise is successful, how do you handle the Promise result?

```
A. Promise.result()
B. Promise.then()
C. Promise.resolve()
D. Promise.success()
```

Answer is B.



## Knowledge Check 3

If you are using node-fetch, which method do you use to convert a stream response to a json response?

```
A. .json()
B. .text()
C. .blob()
D. .response()
```

Answer is A.


## Knowledge Check 4

If you use await within a function, what keyword do you need to add before the function?

```
A. await
B. callback
C. async
D. try
```

Answer is C.
