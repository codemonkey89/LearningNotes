<h1 id="beginners-guide-to-linkers">Beginner’s Guide to Linkers</h1>

<p>Original Link:  <em><a href="http://www.lurklurk.org/linkers/linkers.html">http://www.lurklurk.org/linkers/linkers.html</a></em></p>

<p><div class="toc">
<ul>
<li><a href="#beginners-guide-to-linkers">Beginner’s Guide to Linkers</a><ul>
<li><ul>
<li><a href="#declaration-vs-definition">Declaration vs. Definition</a><ul>
<li><a href="#declaration">Declaration</a></li>
<li><a href="#definition">Definition</a></li>
</ul>
</li>
<li><a href="#what-the-c-compiler-does">What The C Compiler Does</a><ul>
<li><a href="#dissecting-an-object-file">Dissecting An Object File</a></li>
</ul>
</li>
<li><a href="#what-the-linker-does-part-1">What The Linker Does: Part 1</a><ul>
<li><a href="#duplicate-symbols">Duplicate Symbols</a></li>
</ul>
</li>
<li><a href="#what-the-operating-system-does">What The Operating System Does</a></li>
<li><a href="#what-the-linker-does-part-2">What The Linker Does: Part 2</a><ul>
<li><a href="#static-libraries">Static Libraries</a></li>
<li><a href="#shared-libraries">Shared Libraries</a></li>
</ul>
</li>
<li><a href="#adding-c-to-the-picture">Adding C++ To The Picture</a><ul>
<li><a href="#function-overloading-name-mangling">Function Overloading &amp; Name Mangling</a></li>
<li><a href="#initialization-of-statics-ie-global-static-variables">Initialization of Statics (i.e. global static variables)</a></li>
<li><a href="#templates">Templates</a></li>
</ul>
</li>
<li><a href="#dynamically-linked-libraries">Dynamically Linked Libraries</a></li>
</ul>
</li>
</ul>
</li>
</ul>
</div>
</p>



<h3 id="declaration-vs-definition">Declaration vs. Definition</h3>



<h4 id="declaration">Declaration</h4>

<p>A <em>declaration</em> tells the C compiler that a definition of something (with a particular name) exists elsewhere in the program, probably in a different C file. </p>

<ul>
<li><strong><em>global variables</em></strong>,  which exist for the whole lifetime of the program (“<strong>static</strong> extent”), and which are usually accessible in lots of different functions</li>
<li><strong><em>local variable</em></strong>, which only exist while a particular function is being executed (“<strong>local</strong> extent”) and are only accessible within that function</li>
</ul>



<h4 id="definition">Definition</h4>

<p>A <em>definition</em> associates a name with an implementation of that name, which could be either data or code. A definition of a variable induces the compiler to</p>

<ul>
<li>reserve some space for that variable, and possibly fill that space with a particular value.</li>
<li>generate code for that function.</li>
</ul>

<table>
<thead>
<tr>
  <th align="left">Phase</th>
  <th align="left">Code</th>
  <th align="left">Global Initialized</th>
  <th align="left">Global uninitialized</th>
  <th align="left">Local Initialized</th>
  <th align="left">Local Uninitialized</th>
  <th align="left">Dynamic</th>
</tr>
</thead>
<tbody><tr>
  <td align="left">Declaration</td>
  <td align="left"><code>int fn(int x);</code></td>
  <td align="left"><code>extern int x;</code></td>
  <td align="left"><code>extern int x;</code></td>
  <td align="left">N/A</td>
  <td align="left">N/A</td>
  <td align="left">N/A</td>
</tr>
<tr>
  <td align="left">Definition</td>
  <td align="left"><code>int fn(int x) { ... }</code></td>
  <td align="left"><code>int x = 1;</code></td>
  <td align="left"><code>int x;</code></td>
  <td align="left"><code>int x = 1;</code></td>
  <td align="left"><code>int x;</code></td>
  <td align="left"><code>(int* p = malloc(sizeof(int));)</code></td>
</tr>
</tbody></table>


<p>c_parts.c</p>



<pre class="prettyprint"><code class="language-c hljs "> <span class="hljs-comment">/* This is the definition of a uninitialized global variable */</span>
<span class="hljs-keyword">int</span> x_global_uninit;

<span class="hljs-comment">/* This is the definition of a initialized global variable */</span>
<span class="hljs-keyword">int</span> x_global_init = <span class="hljs-number">1</span>;

<span class="hljs-comment">/* This is the definition of a uninitialized global variable, albeit
 * one that can only be accessed by name in this C file */</span>
<span class="hljs-keyword">static</span> <span class="hljs-keyword">int</span> y_global_uninit;

<span class="hljs-comment">/* This is the definition of a initialized global variable, albeit
 * one that can only be accessed by name in this C file */</span>
<span class="hljs-keyword">static</span> <span class="hljs-keyword">int</span> y_global_init = <span class="hljs-number">2</span>;

<span class="hljs-comment">/* This is a declaration of a global variable that exists somewhere
 * else in the program */</span>
<span class="hljs-keyword">extern</span> <span class="hljs-keyword">int</span> z_global;

<span class="hljs-comment">/* This is a declaration of a function that exists somewhere else in
 * the program (you can add "extern" beforehand if you like, but it's
 * not needed) */</span>
<span class="hljs-keyword">int</span> fn_a(<span class="hljs-keyword">int</span> x, <span class="hljs-keyword">int</span> y);

<span class="hljs-comment">/* This is a definition of a function, but because it is marked as
 * static, it can only be referred to by name in this C file alone */</span>
<span class="hljs-keyword">static</span> <span class="hljs-keyword">int</span> fn_b(<span class="hljs-keyword">int</span> x)
{
  <span class="hljs-keyword">return</span> x+<span class="hljs-number">1</span>;
}

<span class="hljs-comment">/* This is a definition of a function. */</span>
<span class="hljs-comment">/* The function parameter counts as a local variable */</span>
<span class="hljs-keyword">int</span> fn_c(<span class="hljs-keyword">int</span> x_local)
{
  <span class="hljs-comment">/* This is the definition of an uninitialized local variable */</span>
  <span class="hljs-keyword">int</span> y_local_uninit;
  <span class="hljs-comment">/* This is the definition of an initialized local variable */</span>
  <span class="hljs-keyword">int</span> y_local_init = <span class="hljs-number">3</span>;

  <span class="hljs-comment">/* Code that refers to local and global variables and other
   * functions by name */</span>
  x_global_uninit = fn_a(x_local, x_global_init);
  y_local_uninit = fn_a(x_local, y_local_init);
  y_local_uninit += fn_b(z_global);
  <span class="hljs-keyword">return</span> (y_global_uninit + y_local_uninit);
}</code></pre>

<blockquote>
  <p><strong>Note:</strong> There is no ‘extern’ keyword before the fn_a() function. This is because all function declarations in the header are ‘extern’ by default. <a href="https://stackoverflow.com/questions/856636/effects-of-the-extern-keyword-on-c-functions">https://stackoverflow.com/questions/856636/effects-of-the-extern-keyword-on-c-functions</a></p>
</blockquote>



<h3 id="what-the-c-compiler-does">What The C Compiler Does</h3>

<p>Convert from C file to machine code in <em>.o</em> or <em>.obj</em> suffix that contains</p>

<ul>
<li><em>code</em>, corresponding to definitions of functions in the C file</li>
<li><em>data</em>, corresponding to definitions of <strong>global</strong> variables in the C file </li>
</ul>

<blockquote>
  <p><strong>Note:</strong> Wherever the code refers to a variable or function, the compiler only allows this if it has previously seen a declaration for that variable or function.</p>
</blockquote>

<p>The job of the linker is to make good on these promises that a definition exists somewhere else in the whole program. When generating an object file, the compiler leaves a <em>blank</em>, a “reference” that has a name associated to it, but the value corresponding to that name is not yet known. </p>

<p><img src="http://www.lurklurk.org/linkers/c_parts.png" alt="Graph" title=""></p>

<blockquote>
  <p><strong>Note:</strong> x_global_uninit, y_global_uninit, fn_a and z_global are the blanks that linker will fill in later.</p>
</blockquote>



<h4 id="dissecting-an-object-file">Dissecting An Object File</h4>

<p>Command line tool <em>nm</em> gives information about symbols in an object file.</p>

<p>Symbols from c_parts.o:</p>

<table>
<thead>
<tr>
  <th align="left">Name</th>
  <th>Value</th>
  <th>Class</th>
  <th>Type</th>
  <th>Size</th>
  <th>Line</th>
  <th>Section</th>
</tr>
</thead>
<tbody><tr>
  <td align="left">fn_a</td>
  <td></td>
  <td>U</td>
  <td>NOTYPE</td>
  <td></td>
  <td></td>
  <td><em>UND</em></td>
</tr>
<tr>
  <td align="left">z_global</td>
  <td></td>
  <td>U</td>
  <td>NOTYPE</td>
  <td></td>
  <td></td>
  <td><em>UND</em></td>
</tr>
<tr>
  <td align="left">fn_b</td>
  <td>00000000</td>
  <td>t</td>
  <td>FUNC</td>
  <td>00000009</td>
  <td></td>
  <td>.text</td>
</tr>
<tr>
  <td align="left">x_global_init</td>
  <td>00000000</td>
  <td>D</td>
  <td>OBJECT</td>
  <td>00000004</td>
  <td></td>
  <td>.data</td>
</tr>
<tr>
  <td align="left">y_global_uninit</td>
  <td>00000000</td>
  <td>b</td>
  <td>OBJECT</td>
  <td>00000004</td>
  <td></td>
  <td>.bss</td>
</tr>
<tr>
  <td align="left">x_global_uninit</td>
  <td>00000004</td>
  <td>C</td>
  <td>OBJECT</td>
  <td>00000004</td>
  <td></td>
  <td><em>COM</em></td>
</tr>
<tr>
  <td align="left">y_global_init</td>
  <td>00000004</td>
  <td>d</td>
  <td>OBJECT</td>
  <td>00000004</td>
  <td></td>
  <td>.data</td>
</tr>
<tr>
  <td align="left">fn_c</td>
  <td>00000009</td>
  <td>T</td>
  <td>FUNC</td>
  <td>00000055</td>
  <td></td>
  <td>.text</td>
</tr>
</tbody></table>


<p>Explanation of <strong>Class</strong> column:</p>

<ul>
<li><strong>U</strong> indicates undefined references (a.k.a. blanks); equal to <strong>UND</strong> in Section column.</li>
<li><strong>t</strong> or <strong>T</strong> indicates where code is defined; equal to <strong>text</strong> in Section column. <br>
<ul><li><strong>t</strong> means function is local to this file, or function is declared with <code>static</code> <br>
<ul><li><strong><em>Important!</em></strong> <code>static</code> keyword has different meaning in C and C++ (<a href="https://stackoverflow.com/questions/558122/what-is-a-static-function">stackoverflow</a>).</li></ul></li>
<li><strong>T</strong> means otherwise</li></ul></li>
<li><strong>d</strong> or <strong>D</strong> indicates an initialized global variable; this particular class indicates whether the variable is local(<strong>d</strong>) or not (<strong>D</strong>). equal to <strong>data</strong> in Section column.</li>
<li>For an uninitialized global variable, <strong>b</strong> indicates if it’s static/local, and <strong>B</strong> or <strong>C</strong> when it’s not. equal to <strong>bss</strong> or <strong>COM</strong> in Section column.</li>
</ul>



<h3 id="what-the-linker-does-part-1">What The Linker Does: Part 1</h3>



<pre class="prettyprint"><code class="language-c hljs "><span class="hljs-comment">/* Initialized global variable */</span>
<span class="hljs-keyword">int</span> z_global = <span class="hljs-number">11</span>;
<span class="hljs-comment">/* Second global named y_global_init, but they are both static */</span>
<span class="hljs-keyword">static</span> <span class="hljs-keyword">int</span> y_global_init = <span class="hljs-number">2</span>;
<span class="hljs-comment">/* Declaration of another global variable */</span>
<span class="hljs-keyword">extern</span> <span class="hljs-keyword">int</span> x_global_init;

<span class="hljs-keyword">int</span> fn_a(<span class="hljs-keyword">int</span> x, <span class="hljs-keyword">int</span> y)
{
  <span class="hljs-keyword">return</span>(x+y);
}

<span class="hljs-keyword">int</span> main(<span class="hljs-keyword">int</span> argc, <span class="hljs-keyword">char</span> *argv[])
{
  <span class="hljs-keyword">const</span> <span class="hljs-keyword">char</span> *message = <span class="hljs-string">"Hello, world"</span>;

  <span class="hljs-keyword">return</span> fn_a(<span class="hljs-number">11</span>,<span class="hljs-number">12</span>);
}</code></pre>

<p><img src="http://www.lurklurk.org/linkers/c_rest.png" alt="" title=""></p>

<p>Combining the two graphs together:</p>

<p><img src="http://www.lurklurk.org/linkers/sample1.png" alt="" title=""></p>



<h4 id="duplicate-symbols">Duplicate Symbols</h4>

<p>Linker will throw an error if the definition for a symbol cannot be found. But what if there are <em>two</em> definitions for a symbol when it comes to link time?</p>

<p>In C++, the language has a constraint known as <strong>one definition rule</strong>. <br>
In C, there has to be exactly <em>one</em> definition of any functions or initialized global variables, but C allows <strong>multiple definitions of an uninitialized global variable</strong> and will pick one of the definitions.</p>



<h3 id="what-the-operating-system-does">What The Operating System Does</h3>

<p>In short, OS need to store both <strong>code</strong> (e.g. functions) and <strong>data</strong> (e.g. variable values) in memory.</p>

<p>Running the program obviously involves executing the machine code, so the operating system clearly has to transfer the machine code from the executable file on the hard disk into the computer’s <em>memory</em>, known as the <strong>code segment</strong> or <strong>text segment</strong>.</p>

<p>Code is nothing without data, so all of the global variables need to have some space in the computer’s memory too. However, there’s a difference between <em>initialized</em> and <em>uninitialized</em> global variables. </p>

<ul>
<li>Initialized variables have particular values that need to be used to begin with, and these values are stored in the object files and in the executable file. When the program is started, the OS copies these values into  the <strong>data segment</strong>.</li>
<li>Uninitialized variables are initialized to 0 by OS in memory directly, stored in <strong>bss segment</strong> and do not need to be saved in the object files (i.e. it saves space on disk).</li>
</ul>

<p><img src="http://www.lurklurk.org/linkers/os_map1.png" alt="" title=""></p>

<blockquote>
  <p><strong>Note:</strong> Local variables (allocated on stack) and dynamically allocated memory (allocated on heap) don’t need linker involvement because their lifetime only occur when the program is running.</p>
</blockquote>



<h3 id="what-the-linker-does-part-2">What The Linker Does: Part 2</h3>

<p>A <strong>Library</strong> is a collection of related compiled object files (i.e. <code>.o</code> files), that can be reused by different programs.</p>



<h4 id="static-libraries">Static Libraries</h4>

<p>On UNIX, <code>ar</code> is commonly used to produce static libraries (usually with <code>.a</code> extension). These library files are normally prefixed with “<code>lib</code>” and passed to the linker with a “<code>-l</code>” option followed by the name of the library, without prefix or extension.</p>

<p>For example, “<code>-lfred</code>” will pick up the static library called “<code>libfred.a</code>“.  </p>

<ul>
<li>The library is only consulted <strong>after</strong> the normal linking is done and they are processed <em>in order</em>. If an object pulled in from a library late in the link line needs a symbol from a library earlier in the link time, the linker won’t automatically find it.</li>
<li>When a symbol is needed, the <strong>whole</strong> object that contains the symbol definition is included and the linker needs to resolve new undefined symbols in that object. (i.e. the granularity of what the linker pulls in from the library is at the <em>object</em> level). </li>
</ul>

<p>In the example, the linker pulls in <code>a.o</code>, <code>b.o</code>, <code>-lx</code> and <code>-ly</code>.</p>

<table rules="all" border="1">
  <tbody><tr>
   <th>File</th>
   <th><code>a.o</code></th>
   <th><code>b.o</code></th>
   <th colspan="3" align="center"><code>libx.a</code></th>
   <th colspan="3" align="center"><code>liby.a</code></th>
  </tr>
  <tr>
   <th>Object</th>
   <th><code>a.o</code></th>
   <th><code>b.o</code></th>
   <th><code>x1.o</code></th>
   <th><code>x2.o</code></th>
   <th><code>x3.o</code></th>
   <th><code>y1.o</code></th>
   <th><code>y2.o</code></th>
   <th><code>y3.o</code></th>
  </tr>
  <tr>
   <th>Definitions</th>
   <th><code>a1, a2, a3</code></th>
   <th><code>b1, b2</code></th>
   <th><code>x11, x12, x13</code></th>
   <th><code>x21, x22, x23</code></th>
   <th><code>x31, x32</code></th>
   <th><code>y11, y12</code></th>
   <th><code>y21, y22</code></th>
   <th><code>y31, y32</code></th>
  </tr>
  <tr>
   <th>Undefined references</th>
   <th><code>b2, x12</code></th>
   <th><code>a3, y22</code></th>
   <th><code>x23, y12</code></th>
   <th><code>y11</code></th>
   <th><code></code></th>
   <th><code>y21</code></th>
   <th><code></code></th>
   <th><code>x31</code></th>
  </tr>
 </tbody></table>

<ol>
<li>When <code>a.o</code> and <code>b.o</code> are processed, <code>b2</code> and <code>a3</code> are resolved; <code>x12</code> and <code>y22</code> are undefined.</li>
<li>When <code>libx.a</code> is processed, <code>x1.o</code> is pulled in to resolve <code>x12</code> but that adds <code>x23</code> and <code>y12</code> to the unresolved list.</li>
<li><code>x2.o</code> is then pulled in to resolve <code>x23</code>, adding <code>y11</code> to the unresolved list.</li>
<li>The linker then processes <code>liby.a</code>,  and similar logic applies.</li>
</ol>



<h4 id="shared-libraries">Shared Libraries</h4>

<p>Disadvantages of static library:</p>

<ul>
<li>take up unnecessary disk space by pulling in popular common libraries (e.g. C standard library, <code>libc</code>)</li>
<li>cannot get updated code from the statically linked libraries (e.g. bug fixes)</li>
</ul>

<p>Shared libraries normally have a <code>.so</code> extension on UNIX systems or <code>.dll</code> extension on Windows. At linking time, the linker doesn’t include the definition of the symbol in the final executable but instead records the name of the symbol and which library it is supposed to come from in the executable file.</p>

<p>Another big difference between static library and shared library is the granularity of the link. If a symbol is pulled from a particular shared library, the <em>whole</em> library (for static library, it would have been the <em>whole</em> object) is mapped into the address space of the program.</p>



<h3 id="adding-c-to-the-picture">Adding C++ To The Picture</h3>



<h4 id="function-overloading-name-mangling">Function Overloading &amp; Name Mangling</h4>



<pre class="prettyprint"><code class="language-cpp hljs "><span class="hljs-keyword">int</span> max(<span class="hljs-keyword">int</span> x, <span class="hljs-keyword">int</span> y)
{
  <span class="hljs-keyword">if</span> (x&gt;y) <span class="hljs-keyword">return</span> x;
  <span class="hljs-keyword">else</span> <span class="hljs-keyword">return</span> y;
}
<span class="hljs-keyword">float</span> max(<span class="hljs-keyword">float</span> x, <span class="hljs-keyword">float</span> y)
{
  <span class="hljs-keyword">if</span> (x&gt;y) <span class="hljs-keyword">return</span> x;
  <span class="hljs-keyword">else</span> <span class="hljs-keyword">return</span> y;
}
<span class="hljs-keyword">double</span> max(<span class="hljs-keyword">double</span> x, <span class="hljs-keyword">double</span> y)
{
  <span class="hljs-keyword">if</span> (x&gt;y) <span class="hljs-keyword">return</span> x;
  <span class="hljs-keyword">else</span> <span class="hljs-keyword">return</span> y;
}</code></pre>

<p>When some other code refers to <code>max</code>, which one does it mean?</p>

<p>Because the function signature is known at compile time, different overloaded functions get mangled to different names (i.e. <strong>name mangling</strong>), for example, the names that start with ‘_Z3max’ below.</p>

<table>
<thead>
<tr>
  <th align="left">Name</th>
  <th>Value</th>
  <th>Class</th>
  <th>Type</th>
  <th>Size</th>
  <th>Line</th>
  <th>Section</th>
</tr>
</thead>
<tbody><tr>
  <td align="left">__gxx_personality_v0</td>
  <td></td>
  <td>U</td>
  <td>NOTYPE</td>
  <td></td>
  <td></td>
  <td><em>UND</em></td>
</tr>
<tr>
  <td align="left">_Z3maxii</td>
  <td>00000000</td>
  <td>T</td>
  <td>FUNC</td>
  <td>00000021</td>
  <td></td>
  <td>.text</td>
</tr>
<tr>
  <td align="left">_Z3maxff</td>
  <td>00000022</td>
  <td>T</td>
  <td>FUNC</td>
  <td>00000029</td>
  <td></td>
  <td>.text</td>
</tr>
<tr>
  <td align="left">_Z3maxdd</td>
  <td>0000004c</td>
  <td>T</td>
  <td>FUNC</td>
  <td>00000041</td>
  <td></td>
  <td>.text</td>
</tr>
</tbody></table>


<blockquote>
  <p><strong>Note:</strong> When C and C++ code is intermingled, all symbols produced by C++ compiler are mingled; all the symbols produced by C compiler are not. To get around this, the C++ language allows you to put <code>extern "C"</code> around the declaration &amp; definition of a function, which tells the C++ compiler not to mangle such functions.</p>
  
  <p>This is necessary when some C code needs to call C++ functions or vice versa.</p>
</blockquote>



<h4 id="initialization-of-statics-ie-global-static-variables">Initialization of Statics (i.e. global static variables)</h4>

<p>In C++, the construction process is allowed to be much more complicated than just copying in a fixed value; all of the code in the various constructors for the class hierarchy has to be run, before the program itself starts running properly.</p>

<p>To deal with this, the compiler includes some extra information in the object files for each C++ file; specifically, the list of constructors that need to be called for this particular file. At link time, the linker combines all of these individual lists into one big list, and includes code that goes through the list one by one, calling all of these global object constructors.</p>

<p>Note that the order in which all of these constructors for global objects get called is not defined—it’s entirely at the mercy of what the linker chooses to do. (See Scott Meyers’ Effective C++ for more details—Item 4 in the third edition).</p>



<h4 id="templates">Templates</h4>

<p>Each of these different instantiations of the template involves different actual machine code, so by the time that the program is finally linked, the compiler and linker need to make sure that every instantiation of the template that is used has code included into the program (and no unused template instantiations are included to bloat the program size).</p>

<p>How does the compiler and linker figure that out?</p>

<ol>
<li><p><strong>Duplicate initialization</strong> <br>
Each object file contains the code for all of the templates that it uses. These definitions are listed as <strong><em>weak symbols</em></strong>, and this means that when the linker produces the final executable program, it can throw away all <em>but</em> <em>one</em> of these duplicate definitions.  <br> <br>
The most significant <strong>downside</strong> of this approach is that all of the individual object files take up much more room on the hard disk.</p></li>
<li><p><strong>Deferring instantiation until link time</strong> <br>
None of the template definitions are stored in the object files, and are all left as undefined symbols. At link time, the linker can collect together all of the undefined symbols that actually correspond to template instantiations, and generate the machine code for these instantiations then. <br></p>

<ul><li><strong>Advantage</strong>: saves space in the individual object files</li>
<li><strong>Disadvantage</strong>: the linker needs to keep track of where the header file containing the source code, and needs to be able to invoke the C++ compiler at link time (which may slow down the link).</li></ul></li>
</ol>



<h3 id="dynamically-linked-libraries">Dynamically Linked Libraries</h3>

<p>A previous section described how using shared libraries means that the final link is deferred until the moment when the program is run. On modern systems, it’s possible to defer linking to even later than that via a pair of system calls, <code>dlopen</code> and <code>dlsym</code>.</p>

<p>… (TBC)</p>