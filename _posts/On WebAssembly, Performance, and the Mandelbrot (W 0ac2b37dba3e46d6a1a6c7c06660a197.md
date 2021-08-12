# On WebAssembly, Performance, and the Mandelbrot (WORKING TITLE)

![On%20WebAssembly,%20Performance,%20and%20the%20Mandelbrot%20(W%200ac2b37dba3e46d6a1a6c7c06660a197/benchmark_how.png](On%20WebAssembly,%20Performance,%20and%20the%20Mandelbrot%20(W%200ac2b37dba3e46d6a1a6c7c06660a197/benchmark_how.png)

## [How to Post](https://www.notion.so/How-to-Post-Things-edcb0c0c263f45a9888596d41af2ff5c)

## Introduction

Hey, welcome to the Feather Systems blog. This is the first post of many, documenting our journey towards creating simpler systems for better performance on the web. 

This post will document our initial experiments rendering the Mandelbrot set with JavaScript, Web-workers & emerging web technologies like WebAssembly.  You can access the project here (insert link) and view the code here (insert link).

These experiments were tested and measured by the team to properly document each technology's advantages & disadvantages. 

To elucidate, let's start with fractals & the madnelbrot set:

## THE MANDELBROT SET - Abhi Allu

The Mandelbrot Set is defined as the set of complex numbers that the function ${ z_{n+1}=z_{n}^{2}+c }$ converges when $z_{0} = 0$. A sample image of the Mandelbrot set looks as follows. In black denotes the area on which the location converges. The small smudge at the front is a smaller version of the main island. The function is a fractal, meaning there is a lot of self-similarity, a finite area, but an infinite perimeter. It is also a very chaotic function, meaning that a small change in $c$ would lead to drastically different results.

![On%20WebAssembly,%20Performance,%20and%20the%20Mandelbrot%20(W%200ac2b37dba3e46d6a1a6c7c06660a197/Untitled.png](On%20WebAssembly,%20Performance,%20and%20the%20Mandelbrot%20(W%200ac2b37dba3e46d6a1a6c7c06660a197/Untitled.png)

The Mandelbrot Set also has a lot of beautiful images when zoomed in on the edges. I highly recommend watching any of the millions of videos on YouTube of people zooming in onto various points of the Mandelbrot set. If you would prefer to zoom in yourself, we have a website where you can do so yourself. Click to zoom and Shift + Click to zoom out. There is a soft limit as to how far you can zoom in though, do to the precision of floating points.

[https://mandelbrot-ascript.vercel.app/](https://mandelbrot-ascript.vercel.app/)

[https://mandelbrot-ascript.vercel.app/](https://mandelbrot-ascript.vercel.app/)

### Benchmark Structure

[Github](https://github.com/AO-Design-Inc/js-wasm-mandelbrot-benchmark)

The benchmarking site is designed to display each iteration we developed with a rough estimate for performance measured in milliseconds. For each version, a function was written to perform the Mandelbrot calculations and return a JavaScript ImageData object that could be easily put on the Canvas. Once the user selects a version from the drop down menu, the corresponding function is called. Since the functions can take some time to complete, they are called from a Web-worker to prevent the main thread from freezing while the function runs. The performance is calculated using `performance.now()` calls before and after the function call that returns the ImageData object. 

Across all iterations, we used the following values:

const START_X_TOTAL:f32 = 0.300283

const START_Y_TOTAL:f32 =  -0.48857

const WINDOW:f32 = 0.01

const step_X:f32= WINDOW/f32(canvas_width);

const step_Y:f32 = WINDOW/f32(canvas_height);

const ITER_CONST:i32 = 1000

**FOR MULTITHREADED**

const N_THREADS = 4

We assume that the canvas width and height are integers. To determine if a point was in the Mandelbrot set, we iterated the Mandelbrot function on the point up to `ITER_CONST`, returning the number of times the function had iterated. We use the returned Mandelbrot value to set the color. 

You may notice that RGB values get clamped at 255, making it seem like the exercise of iterating all the way to 1000 is just for the sake of testing performance, which it is. (Here is a whole list of [smarter and better coloring algorithms](https://en.wikipedia.org/wiki/Plotting_algorithms_for_the_Mandelbrot_set) that we will not be using in this benchmarking exercise.)

As of when this article was published, the following versions are available on the Benchmarking Site:

- Naive JS single threaded
- Rust WASM using bindgen (single threaded)
- ~~Rust WASM using bindgen + rayon (multi threaded)~~
- Assemblyscript old version (single threaded, not thread safe)
- Assemblyscript new version (single threaded, thread safe)
- Assemblyscript new version (multithreaded)
- Assemblyscript SIMD WASM (singlethreaded, multithreaded)
- New JS(less memory allocation) singlethreaded
- New JS multithreaded
- New JS SharedMemoryBuffer multithreaded

## IMPLEMENTING THE MANDELBROT SET IN JAVASCRIPT

We started with a naive approach to implement the Mandelbrot set in JavaScript and then try and optimize later. We started by creating a class for Complex numbers. This class would be able to add, multiply and return the magnitude. For the magnitude, we used library function Math.hypot. Down below is our first initial implementation of the Complex class.

```jsx
class Complex {
    constructor(real, imag){
      this.real = real;
      this.imag = imag;
    }
//Function that add this complex number with another complex number, cplx
    add(cplx) {
      this.real += cplx.real;
      this.imag += cplx.imag;
      return this;
    }
//Fuction that returns the magnitude of this complex number
     mag(cplx) {
      return Math.hypot(this.real,this.imag);
    }
//Function that multiplies this complex number with another complex number, cplx
    mul(cplx) {
      const real_part = this.real*cplx.real - this.imag*cplx.imag;
      const imag_part = this.imag*cplx.real + this.real*cplx.imag;
      this.real = real_part;
      this.imag = imag_part;
      return this;
    }
  }
```

We iterated through each element of the canvas to compute whether or not they were in the Mandelbrot set. 

```jsx
function mandelbrot(cplx) {
  let z = new Complex(0,0);
  
  let count = 0;
  while (z.mag() <= 2 && count < ITER_CONST) {
    z = (z.mul(z)).add(cplx);
    count++;
  }
  
  return count;
}
```

These datapoints were added to an array which then painted onto the Canvas. We had some ideas about optimizations with help from the profiler, however the most fascinating one was that the Math.hypot function for two numbers is an order of magnitude slower than squaring, adding, and then computing the square root. During the profiling of the initial code we found out that Math.hypot was taking up most of the time. This is because the monitor never sent Math.hypot to the compiler, as the Math.hypot function can take up to n variables. It is drastically slower to call Math.hypot when you know how many variables it is supposed to take.  

![On%20WebAssembly,%20Performance,%20and%20the%20Mandelbrot%20(W%200ac2b37dba3e46d6a1a6c7c06660a197/math_hypot_function.png](On%20WebAssembly,%20Performance,%20and%20the%20Mandelbrot%20(W%200ac2b37dba3e46d6a1a6c7c06660a197/math_hypot_function.png)

### The Perf Log

[node-perf2.svg](On%20WebAssembly,%20Performance,%20and%20the%20Mandelbrot%20(W%200ac2b37dba3e46d6a1a6c7c06660a197/node-perf2.svg)

## A DEEPER LOOK INTO THE JAVASCRIPT ENGINE

The Javascript Engine uses a JIT (Just-In Time) Compiler which splits the work into two parts. It first compiles the code iteratively while a monitor watches for hot functions.  Once a function is recognized as hot, the JIT compiles the whole section to make it faster for future calls. 

This is a JSConf video that explains the Javascript Engine and Webassembly

- [https://www.youtube.com/watch?v=HktWin_LPf4&ab_channel=JSConf](https://www.youtube.com/watch?v=HktWin_LPf4&ab_channel=JSConf)

### JAVASCRIPT

To improve the initial naive approach, we decided to add multithreading. We started this with a naive way of having the worker compute each point and then send it back to receive the next point, but the message overhead was really drastic and was essentially a single threaded function as we only had one worker. The next step we did was experiment with divisibility classes to add multiple workers. But this had the same problem where the overhead of sending a message to and from the workers was just too much. Eventually we figured out that we would have each worker compute the set number of rows at a time. 

```jsx
//This is in the main function which calls the workers to do the work
for(let i=0; i<workerCount; i++){
  const worker = new Worker("worker.js");
  const INDEXES_PER_WORKER = Math.floor((X_LEN*Y_LEN)/workerCount);
  const N_ROWS_PER_THREAD = Math.floor(X_LEN/workerCount);
  const START_XC = N_ROWS_PER_THREAD * i;
  var START_INDEX = i*INDEXES_PER_WORKER;
  var END_INDEX= START_INDEX + INDEXES_PER_WORKER;
  
  worker.onmessage = ({data}) => {
    const COMPUTED_ROWS = data;
    points_array[i]= COMPUTED_ROWS;
     doneCount++;
    if(doneCount == workerCount) {
      points_array = points_array.flat();

      
      draw(points_array);
    }
  }
```

```jsx
//This is how it looks now in the worker function
onmessage = function({data}) {
  const{START_XC, STEP_X, STEP_Y, N_ROWS_PER_THREAD, Y_LEN} = data;
  const points_in_thread = new Array(Y_LEN * N_ROWS_PER_THREAD);

    for(let x = -2.0 + START_XC * STEP_X, count_x = 0; count_x < N_ROWS_PER_THREAD; x+=STEP_X, count_x++){
      for( let y = -2.0, count_y =0; count_y < Y_LEN; y+=STEP_Y, count_y++){
        points_in_thread[count_x * Y_LEN + count_y] = mandelbrot(new Complex(y,x));
      }
    }
    postMessage(points_in_thread);
  }
```

### Shared Buffer

Since all the points would be added to the same array it would make sense that having a shared buffer to reduce the overhead needed to send array points would decrease the compute time. So we added a shared buffer that was formatted like an ImageData object so it would be easier to paint it onto the canvas. This required some rewriting of the some steps here and there, but it ended up being faster.

The other issue with using a Shared Buffer is that we had to add promises to the code, so we refactored it to include promises. 

 

```jsx
//This is in the main function which calls the workers to do the work
return new Promise((resolve) => {
    var worker = new Worker("benchmarks/multithreaded-js/sharedworker.js");

    worker.postMessage(parameters);

    worker.onmessage = function(){
      resolve("worker finished");
    }
  });
}

async function returnSharedBufferjs(START_X_TOTAL, START_Y_TOTAL, CANVAS_WIDTH, CANVAS_HEIGHT, WINDOW){
  
  const X_LEN = CANVAS_WIDTH;
  const Y_LEN = CANVAS_HEIGHT;
  const window = WINDOW;
  const STEP_X = window/X_LEN;
  const STEP_Y = window/Y_LEN;
  const workerCount = 4;
  const sharedBuffer = new SharedArrayBuffer(X_LEN*Y_LEN*4);
  const sharedArray = new Uint8ClampedArray(sharedBuffer);
  sharedArray.fill(0);

  const N_ROWS_PER_THREAD = Math.floor(X_LEN/workerCount);
  var START_YC = N_ROWS_PER_THREAD;

  var parameters = {START_X_TOTAL,START_Y_TOTAL, START_YC, STEP_X, STEP_Y, N_ROWS_PER_THREAD, X_LEN, sharedArray};
  var promises = [];

  for(let i=0; i<workerCount; i++){

    parameters.START_YC = N_ROWS_PER_THREAD * i;

    promises.push(createWorker(parameters));
  }
```

```jsx
//This is how it looks now in the worker function
onmessage = function({data}) {
    const{START_X_TOTAL,START_Y_TOTAL,START_YC, STEP_X, STEP_Y, N_ROWS_PER_THREAD, X_LEN, sharedArray} = data;
    for( let y = START_Y_TOTAL+ START_YC * STEP_Y, count_y =START_YC; count_y < N_ROWS_PER_THREAD+START_YC; y+=STEP_Y, count_y++)
    {
      for(let x = START_X_TOTAL , count_x = 0; count_x < X_LEN; x+=STEP_X, count_x++)
      {
        
        let index = 4*(count_x  + count_y* X_LEN);
        val = mandelbrot(new Complex(x,y));
         sharedArray[index+ 0] = val; 
         sharedArray[index+ 1] = val; 
         sharedArray[index+ 2] = val; 
         sharedArray[index+ 3] = 255;      
      }
    }
      postMessage("done");
  }
```

## RUST IMPLEMENTATION - *Jo Gao*

### Singlethreaded

I wrote a similar implementation of the Mandelbrot function in Rust that would return a ImageData object. It uses [wasm-bindgen](https://rustwasm.github.io/wasm-bindgen/), a Rust library that will compile Rust code to webAssembly, and then generate the bindings and glue between JavaScript and webAssembly so that it can be run on the web. It handles all the type conversions between JavaScript and webAssembly and facilitates the compilation of Rust to WebAssembly. 

In the iterative Mandelbrot function, `count` is let to iterate all the way to 1000 if possible for benchmarking purposes. When returning, `count` is set to 255 if larger than 255 to prevent having to cast numbers larger than 255 to u8.

```rust
fn in_mandelbrot(cplx: &Complex) -> u8 {
    const ITER_CONST: i32 = 1000;
    let mut z = build_complex(0.0, 0.0);
    let mut count: i32 = 0;

    while z.mag() <= 2.0 && count < ITER_CONST {
        z = (z.square()).add(&cplx); // z = z^2 + cplx
        count += 1;
    }

    if count > 255 {
        count = 255;
    }
    count as u8
}
```

The magic of wasm-bindgen comes in here, allowing us to create the ImageData object right in the Rust code by providing the type and the constructor signature:

```rust
#[wasm_bindgen]
extern "C" {
    pub type ImageData;

    #[wasm_bindgen(constructor, catch)]
    fn new(data: &Uint8ClampedArray, width: f64, height: f64) -> Result<ImageData, JsValue>;
}

#[wasm_bindgen]
pub fn run(start_x: f64, start_y: f64, width: u32, height: u32, window: f64) -> ImageData{

    let y_len: i32 = height as i32;
    let x_len: i32 = width as i32;

    let pixels: usize = (x_len * y_len) as usize;

    let mut points_array = vec![0; pixels*4];
    fill_mandelbrot(&mut points_array, start_x, start_y, x_len, y_len, window);

    let pointer = points_array.as_ptr() as usize;

    let mem = wasm_bindgen::memory().unchecked_into::<WebAssembly::Memory>();
    let new_array = Uint8ClampedArray::new(&mem.buffer()).slice(pointer as u32, (pointer + pixels*4) as u32);
    ImageData::new(&new_array, x_len.into(), y_len.into()).unwrap()
}
```

After compiling this code, wasm-bindgen outputs a number of files that are meant to be bundled using Webpack and imported to your JavaScript file as an ES6 module, allowing access to the functions that were written in Rust. The idea is that after compiling and importing the module, calling `run(...)`would return a whole JavaScript ImageData Object. 

Unfortunately, our Benchmarking Site is structured to use a web worker to call this function. As far as modules and web workers are concerned, modules cannot be imported to classic web workers. However, they can be imported to module workers, web workers that support module import, but as of the end of 2019, only Chrome 80+ fully supports module workers. 

This is an infeasible solution because we were unwilling to limit the availability of our benchmark site for this implementation. After many hours of tinkering with the existing setup, Abhishek and I found a workaround that eliminated the need for Webpack altogether: After compiling with wasm-bindgen, we copy and pasted the entire outputted .js file with the import and exports removed, appended the wrapper function I had written to it, and called init on the outputted .wasm. 

After confirming that this worked, we wrote a shell script to automatically do this after compilation. Albeit a little unpleasant on the eyes, it does the job:

```bash
sed '/import\W/d;s/^export//g;/default/d' pkg/Mandelbrot.js > tmp_mandel_import.js
cat tmp_mandel_import.js  mandel_src.js > Mandelbrot.js
rm tmp_mandel_import.js
```

Getting JS objects in Rust with wasm-bindgen: [https://rustwasm.github.io/wasm-bindgen/contributing/design/js-objects-in-rust.html](https://rustwasm.github.io/wasm-bindgen/contributing/design/js-objects-in-rust.html)

module worker support: [https://developer.mozilla.org/en-US/docs/Web/API/Worker/Worker](https://developer.mozilla.org/en-US/docs/Web/API/Worker/Worker)

### RUST AND WEBASSEMBLY MULTITHREADED - *Jo Gao*

To further improve performance in Rust, we pursued performing the same calculations, but in parallel this time. An advantage of the Mandelbrot Set is that determining whether a pixel falls within the set or not can be done completely independently of other pixels. The outcome of each pixels has no effect on the others, making the transition to multithreading straightforward.

We used the [Parallel Raytracing example](https://github.com/rustwasm/wasm-bindgen/tree/master/examples/raytrace-parallel). given in the Wasm-Bindgen repo, as a starting point.  Using a combination of WebWorkers and the Rayon Crate for data parallelism in Rust yielded a pretty fast multithreaded Mandelbrot Set. 

*Unfortunately, we were unable to include this version of the Mandelbrot in the final Benchmarking website* because we ran into a similar problem with the wasm-bingen and module output. This time, it was further complicated by the Webworkers being created from within the Rust code. The funky workaround that had allowed for the singlethreaded RustWasm example to run on the Benchmarking Site failed to work the same magic for this implementation. 

This version of the Mandelbrot also makes use of the SharedArrayBuffer, and had it been on the site, would not work in Safari and browsers that do not support SharedArrayBuffer Objects.

## WEBASSEMBLY - *Abhishek Cherath*

Insofar as an overview is concerned, nothing beats Lin Clark's cartoon intro [here](https://hacks.mozilla.org/2017/02/a-cartoon-intro-to-webassembly/), and the MDN docs [here](https://developer.mozilla.org/en-US/docs/WebAssembly). Put simply, WebAssembly is a low level typed language that targets the browser's VM. For some reasons[1] , it's very quick and easy to compile into decently fast machine code.

Fundamentally, the language was designed to offer the best possible latency (time to execution) and speed of execution. Low latency is achieved by having a small binary representation (vs javascript's textual representation) and having type definitions and a function table at the start of a .wasm module, which allows compilers to compile in a streaming fashion, instead of needing to have access to the entire file before starting. 

Speed of execution requires slow compilation with multiple passes for the compiler to accurately assess the best register allocation, optimize out unnecessary instructions and do whatever other things compilers do. (*note: I have frankly no idea what goes into compiler optimization, I just know it takes time*)

To satisfy both these goals, browsers (atleast Firefox and Chrome) have two compilers for WebAssembly, the first compiles the module as it's called from the network, and is basically instant ([here](https://v8.dev/docs/wasm-compilation-pipeline), chrome devs claim that theirs can do on the order of 10+ mb of code per second, so the bottleneck will almost always be the speed of the network rather than the compiler.) The second does optimization and dynamically replaces the unoptimized bytecode from the first when it's done.

It's interesting to note here that, *a priori* there's no reason to expect that WebAssembly code for high performance stuff [2] (like fractal calculation) will be any faster than its javascript counterpart. Since the hot loop (calculating escape time) is running many thousands of times, one would expect the javascript JIT compiler to infer the output types and get basically similar bytecode, while amortizing its greater overhead considering workload size.

In a later article, I will be examining this conjecture, and taking a deeper look at the flamegraphs and bytecode generated by our benchmarks.

The other interesting thing about WebAssembly is its memory model, programs do not have access to their own instructions and stack machine memory (so no self modifying code, and no messing with values on the stack). What they do have access to is a block of linear memory, which is backed by a javascript ArrayBuffer, and byte addressed (ie. memory[0] is 1 byte.)

**References:**

David Beazley. (2019). A Talk Near the Future of Python (a.k.a., Dave live-codes a WebAssembly Interpreter). [https://www.youtube.com/watch?v=r-A78RgMhZU](https://www.youtube.com/watch?v=r-A78RgMhZU)

Firefox’s low-latency webassembly compiler – wingolog. (n.d.). Retrieved June 19, 2021, from [https://wingolog.org/archives/2020/03/25/firefoxs-low-latency-webassembly-compiler](https://wingolog.org/archives/2020/03/25/firefoxs-low-latency-webassembly-compiler)

firefox’s low-latency webassembly compiler—Wingolog. (n.d.). Retrieved June 19, 2021, from [https://wingolog.org/archives/2020/03/25/firefoxs-low-latency-webassembly-compiler](https://wingolog.org/archives/2020/03/25/firefoxs-low-latency-webassembly-compiler)

Pereira, F. M. Q., & Palsberg, J. (2006). Register Allocation After Classical SSA Elimination is NP-Complete. In L. Aceto & A. Ingólfsdóttir (Eds.), Foundations of Software Science and Computation Structures (Vol. 3921, pp. 79–93). Springer Berlin Heidelberg. [https://doi.org/10.1007/11690634_6](https://doi.org/10.1007/11690634_6)

WebAssembly. (n.d.). Retrieved June 19, 2021, from [https://webassembly.org/](https://webassembly.org/)

WebAssembly compilation pipeline · V8. (n.d.). Retrieved June 19, 2021, from [https://v8.dev/docs/wasm-compilation-pipeline](https://v8.dev/docs/wasm-compilation-pipeline)

*[1]I'm not entirely sure why, but it's some combination of coding for a simple stack machine, having type information, linear memory, and being easy to parse.*

*[2]This is **mostly** true, except for vectorization. SIMD (Single Instruction Multiple Data) instructions are featured in the WebAssembly spec and implemented by chrome and firefox, which do not expose them in js, and have no plans to do so. (although I suppose there's nothing stopping a JIT from autovectorizing, is there?)*

## ASSEMBLYSCRIPT

Assemblyscript compiles a subset of typescript to webassembly. To get an idea of how simple it is to write it, here's essentially the same javascript code from earlier, with type annotations and minor changes for WebAssembly:

```jsx
declare const canvas_width: i32;
declare const canvas_height: i32;
declare const ITER_CONST: i16;
declare const START_X_TOTAL:f32
declare const START_Y_TOTAL:f32
declare const WINDOW:f32

@unmanaged
class Complex {
	real: f64 = 0;
	imag: f64 = 0;

	constructor(real:f64, imag:f64){
		this.real = real;
		this.imag = imag;
	}

	@inline
	add(cplx: Complex): Complex {
		this.real = this.real + cplx.real;
		this.imag = this.imag + cplx.imag;
		return this;
	}

	@inline
	mag(): f64 {
		return Math.sqrt(this.real * this.real + this.imag * this.imag)
	}

	@inline
	mul(cplx: Complex): Complex {
		const __tempr  = this.real*cplx.real - this.imag*cplx.imag;
		const __tempi = this.imag*cplx.real + this.real*cplx.imag;
		this.real = __tempr;
		this.imag = __tempi;
		return this
	}

	set(real:f64,imag:f64): void {
		this.real = real;
		this.imag = imag;
	}

}

const z:Complex = new Complex(0,0);
const cplx:Complex = new Complex(0,0)

@inline
function mandelbrot(real:f64,imag:f64):i16{
	z.set(0,0)
	cplx.set(real,imag)

	let count:i16 = 0;
	for (; z.mag() <= 2; count++) {
		(z.mul(z)).add(cplx); // z = z^2 + cplx
		if (count > ITER_CONST) {
			break;
		}
	}
	return count;
}
/*
stores array of i16s at start of memory 
corresponding to escape count at pixels in
array
*/
export function compute(): void {
	let memcounter = 0
	const step_X:f32 = WINDOW/f32(canvas_width);
	const step_Y:f32 = WINDOW/f32(canvas_height);
	for (let y = START_Y_TOTAL, count_y = 0; count_y < canvas_height; y += step_Y, count_y++){
		for (let x = START_X_TOTAL, count_x = 0; count_x < canvas_width; x += step_X, count_x++){
			store<i16>(memcounter, mandelbrot(x,y));
			memcounter += 2
		}
	}
}
```

The only really big change was to not have `new` in the hot loop. The Assemblyscript garbage collector had some trouble with it and total allocation would exceed WebAssembly's 100 memory page limit[1]. I suspect that sort of code will be usable once the WebAssembly GC proposals, which allow it to hook into the browser's GC, are implemented.

### A NOTE ON THREADING:

The fastest way to thread code in browsers is to use [SharedArrayBuffer](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/SharedArrayBuffer), this also requires the following CORS headers.

```jsx
Cross-Origin-Opener-Policy: same-origin
Cross-Origin-Embedder-Policy: require-corp
```

This is to ensure cross-origin isolation, which protects against memory being exfiltrated somehow (I might explore this in the future, but that's about all I know for the moment.) WebAssembly memories can also be created with a backing SharedArrayBuffer (Link to explanation), which allows fast multithreading, as separate web-workers can run the same Webassembly module and write their results to the same memory, meaning that large objects don't need to be passed around using postmessage, and the many O(N) overheads to do with copying and creating new arrays can be avoided.

However, while Assemblyscript [allows](https://www.assemblyscript.org/stdlib/builtins.html#atomics-%F0%9F%A6%84) access to atomic instructions, it does not implement any sort of locking. So any program that reads and writes to memory will need to be adjusted to thread with shared memory. This means that the program seen above, with its Complex class and set calls reading and writing to the linear memory quite often, is very prone to... interesting behaviour when threaded.

![On%20WebAssembly,%20Performance,%20and%20the%20Mandelbrot%20(W%200ac2b37dba3e46d6a1a6c7c06660a197/Untitled%201.png](On%20WebAssembly,%20Performance,%20and%20the%20Mandelbrot%20(W%200ac2b37dba3e46d6a1a6c7c06660a197/Untitled%201.png)

Exhibit A of nondeterministic behavior

That being said, for small examples such as fractal calculation, its quite easy to limit mutability to local variables (which are on the stack, not in the linear memory, and so are not shared) and avoid the problem entirely. Like in the following code:

```jsx
@inline
function mandelbrot(c_r:f64, c_i:f64):i16{
	let count:i16 = 0;
	let z_r:f64 = 0., z_i:f64 = 0., t_r:f64 = 0., t_i:f64 = 0.;
	for(; z_r*z_r + z_i*z_i < 4; count++) {
		t_r = z_r*z_r - z_i*z_i + c_r;
		t_i = 2*z_i*z_r + c_i;
		z_r = t_r;
		z_i = t_i;
		if(count > ITER_CONST) {
			break;
		}
	}
	return count;
}
/*
stores array of i16s at start of memory 
corresponding to escape count at pixels in
array
*/
export function compute():void {
	let memcounter = 0
	const step_X:f32 = WINDOW/f32(canvas_width);
	const step_Y:f32 = WINDOW/f32(canvas_height);
	for (let y = START_Y_TOTAL, count_y = 0; count_y < canvas_height; y += step_Y, count_y++){
		for (let x = START_X_TOTAL, count_x = 0; count_x < canvas_width; x += step_X, count_x++){
			store<i16>(memcounter, mandelbrot(x,y));
			memcounter += 2
		}
	}
}
```

*note that the store instruction there might look light it would cause issues, but as long as threads work on different parts of the image, that should not be a problem.*

### SIMD:

SIMD (Single Instruction Multiple Data) instructions are special instructions on the CPU that allow multiple additions, multiplications etc to be carried out in parallel on the same core)

![On%20WebAssembly,%20Performance,%20and%20the%20Mandelbrot%20(W%200ac2b37dba3e46d6a1a6c7c06660a197/SIMD2.svg](On%20WebAssembly,%20Performance,%20and%20the%20Mandelbrot%20(W%200ac2b37dba3e46d6a1a6c7c06660a197/SIMD2.svg)

By Vadikus - Own work, CC BY-SA 4.0, [https://commons.wikimedia.org/w/index.php?curid=39715273](https://commons.wikimedia.org/w/index.php?curid=39715273)

They're not in the webassembly spec as yet, but should be [soon](https://webassembly.github.io/simd/core/exec/instructions.html#simd-instructions), and are currently available on chrome and firefox (although not on ARM in firefox 89, ie. apple M1.) Assemblyscript does not autovectorize code, but [allows](https://www.assemblyscript.org/stdlib/builtins.html#simd-%F0%9F%A6%84) access to the instructions, and provides some sugar for different types and functions to initialise constant vectors.

The vector size in the proposal is 128 bit, which is a decent compromise and is backed by hardware instructions on most modern computers. In the future, [flexible vectors](https://github.com/WebAssembly/flexible-vectors), should be able to allow access to 256, 512 bit vectors that modern intel and AMD CPUs support (consumer ARM chips currently only have 128 bit vectors, as far as I'm aware, although [supercomputer ARM chips do have 512 bit vectors](https://en.wikipedia.org/wiki/AArch64#Scalable_Vector_Extension_(SVE)).)

Vectorizing the mandelbrot code above is not too bad, althought it does look rather messy:

```jsx
@inline
function mandelbrot_simd(c_rl:v128,c_il:v128):v128{
    let in_set : v128 = i32x4(0,0,0,0);
    let z_r : v128 = f32x4(0.,0.,0.,0.);
    let z_i : v128 = f32x4(0.,0.,0.,0.);
    let t_r : v128 = f32x4(0.,0.,0.,0.);
    let t_i : v128 = f32x4(0.,0.,0.,0.);
    const ones = i32x4(1,1,1,1)
    const fours = f32x4(4.,4.,4.,4.)
    const ITER_CONSTS:v128 = v128.splat<i32>(ITER_CONST);
    let count:v128=i32x4(0,0,0,0)
    for(
        let total_count:i32 = 0, any_in_convergence_region:boolean = true;
        any_in_convergence_region &&
        total_count < ITER_CONST;
        total_count++;
    ) {
        z_i = v128.add<f32>(c_il, v128.mul<f32>(v128.add<f32>(z_r,z_r),z_i));
        z_r = v128.add<f32>(c_rl, v128.sub<f32>(t_r,t_i));
        t_r = v128.mul<f32>(z_r,z_r);
        t_i = v128.mul<f32>(z_i,z_i);
        const mask = v128.le<f32>(v128.add<f32>(t_r,t_i),fours);
        any_in_convergence_region = v128.any_true(mask);
        count = v128.add<i32>(count, v128.and(ones,mask));
    }
    return count;
}
```

This should basically be a 4x speedup over the previous code, when run single threaded. Multithreaded, that number will vary depending on how threads are allocated, as work only gets completed as fast as the slowest thread.

### COMPILER FLAGS

For the examples above, some memory is reserved for the purposes of storing the image, in assemblyscript this is achieved with the `--memoryBase` flag to the `asc` compiler (as seen in the makefile below)

The other flags are tuned for maximum performance (thanks to [Max Graey](https://github.com/MaxGraey) for -O3 instead of -O3s/z) and to enable necessary features (threads, SIMD.)

```makefile
MEMORY_FLAGS = --maximumMemory 80 --importMemory --noExportMemory --initialMemory 80 --memoryBase 4000000
OPTIMIZATION_FLAGS = -O3 --converge
DEBUG_FLAGS = --sourceMap
FEATURE_FLAGS = --enable simd --enable threads
RUNTIME_FLAGS = --runtime stub
start: install
	npx asc assembly/mandel_final.ts -b build/mandel_final.wasm -t build/mandel_final.wat $(OPTIMIZATION_FLAGS) $(MEMORY_FLAGS) $(FEATURE_FLAGS) $(RUNTIME_FLAGS) $(DEBUG_FLAGS)

install:
	npm i 
```

### MULTITHREADED SIMD SPEED

Benchmark results will be discussed near the end of this article, but the speed of the multithreaded SIMD code is quite astonishing, roughly 3x the speed of multithreaded JS using sharedarraybuffers, and I'm guessing that's held back by module sending overheads. [HERE](https://mandelbrot-ascript.vercel.app/) is a basic mandelbrot zoom implementation using it (only works in firefox(≥90 on apple m1) and chrome, [here's](https://github.com/pretentious7/mandelbrot-ascript) the github repo.) I'll be working in boundary estimation and period checking along with xaos zoom algorithm into it in a couple of months, so stay tuned for that!

I'm also fairly certain that this is the first(?) SIMD mandelbrot implementation for webassembly, so if anyone wants to use it for anything (under the terms of the GPLv3 License) feel free!

*[1] A page being 64KiB (65,536 bytes), so about 6.5 mb max memory*

## Results

First, we know that our benchmarks here are not rigorous in the slightest. Their purpose was to give us an idea of the rough performance capabilities of the tools we're working with, to help inform our choices as we port certain programs (stay tuned!) to WebAssembly. We will be refining this tool in the future, and hope to have (somehow) bytecode outputs and flamegraphs within the browser.

On a 

[Bechmarks](https://www.notion.so/8b37c240ac0d40b2b913df85f9ffeb0b)

*All numbers approximate, gathered from fresh browser session after reboot on an HP Envy 13 with an i5-8265U CPU @ 1.60GHz and 8gb of RAM running Ubuntu 20.04.2 with kernel 5.8.0. Average of 5 runs*

The results of our benchmark (at least approximately) line up with our original expectations, but we found a few items of interest.

First, as far as the javascript implementations are concerned, fire

[Here](https://js-wasm-mandelbrot-benchmark-3.vercel.app/) 

## FINAL THOUGHTS,

- WebAssembly has a place in modern web stuff
- Browsers have a built in tool that allows us to utilize the speed of WebAssembly.

In an upcoming article, we will be taking a deeper look into the performance of each version and comparing their performances

## ACKNOWLEDGEMENTS