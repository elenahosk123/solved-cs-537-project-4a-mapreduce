Download Link: https://assignmentchef.com/product/solved-cs-537-project-4a-mapreduce
<br>
<h2 id="new">NEW</h2>

<ul>

 <li>Deadline extended to April 3</li>

 <li>Minor edit in spec to rename CombinerGetter to CombineGetter.</li>

 <li>Minor change in Reducer definition to include ReduceStateGetter</li>

 <li>Functions added in spec to make Eager mode easier. Please read the section on Eager mode carefully!</li>

</ul>

<h2 id="teaming-up"><em>Teaming up!</em></h2>

For this project, you have the option to work with a partner. Read more details in <a href="http://pages.cs.wisc.edu/~shivaram/cs537-sp20/p4a.html#submitting-your-implementation">Submitting Your Implementation</a> and <a href="http://pages.cs.wisc.edu/~shivaram/cs537-sp20/p4a.html#collaboration">Collaboration</a>.

<h2 id="administrivia">Administrivia</h2>

<ul>

 <li><strong>Due Date</strong> by <span style="text-decoration: line-through;">Apr 2, 2020</span> <strong>April 3, 2020</strong> at 10:00 PM</li>

 <li>Questions: We will be using Piazza for all questions.</li>

 <li>This project is best to be done on the <a href="https://csl.cs.wisc.edu/services/instructional-facilities">lab machines</a>. Alternatively, you can create Virtual Machine and work locally (<a href="http://pages.cs.wisc.edu/~shivaram/cs537-sp20/p4a.html">installation video</a>). You learn more about programming in C on a typical UNIX-based platform (Linux).</li>

 <li>Please take the quiz on Canvas that will be posted on Piazza shortly. It will check if you have read the spec closely. NOTE: The quiz does not work in groups and will have to be taken individually on Canvas.</li>

 <li>Your group (i.e., you and your partner) will have <em>2</em> slip days for the rest of the semester. If you are changing partners please talk to the instructor about it.</li>

</ul>

In 2004, engineers at Google introduced a new paradigm for large-scale parallel data processing known as MapReduce (see the original paper <a href="https://static.googleusercontent.com/media/research.google.com/en//archive/mapreduce-osdi04.pdf">here</a>, and make sure to look in the citations at the end!). One key aspect of MapReduce is that it makes programming tasks on large-scale clusters easy for developers; instead of worrying about how to manage parallelism, handle machine crashes, and many other complexities common within clusters of machines, the developer can instead just focus on writing little bits of code (described below) and the infrastructure handles the rest.

In this project, you’ll be building a simplified version of MapReduce for just a single machine. While it is somewhat easier to build MapReduce for a single machine, there are still numerous challenges, mostly in building the correct concurrency support. Thus, you’ll have to think a bit about how to build the MapReduce implementation, and then build it to work efficiently and correctly.

There are three specific objectives to this assignment:

<ul>

 <li>To learn about the general nature of the MapReduce paradigm.</li>

 <li>To implement a correct and efficient MapReduce framework using threads and related functions.</li>

 <li>To gain more experience writing concurrent code.</li>

</ul>

<h2 id="background">Background</h2>

To understand how to make progress on any project that involves concurrency, you should understand the basics of thread creation, mutual exclusion (with locks), and signaling/waiting (with condition variables). These are described in the following book chapters:

<ul>

 <li><a href="http://pages.cs.wisc.edu/~remzi/OSTEP/threads-intro.pdf">Intro to Threads</a></li>

 <li><a href="http://pages.cs.wisc.edu/~remzi/OSTEP/threads-api.pdf">Threads API</a></li>

 <li><a href="http://pages.cs.wisc.edu/~remzi/OSTEP/threads-locks.pdf">Locks</a></li>

 <li><a href="http://pages.cs.wisc.edu/~remzi/OSTEP/threads-locks-usage.pdf">Using Locks</a></li>

 <li><a href="http://pages.cs.wisc.edu/~remzi/OSTEP/threads-cv.pdf">Condition Variables</a></li>

</ul>

Read these chapters carefully in order to prepare yourself for this project.

<h2 id="general-idea">General Idea</h2>

Let’s now get into the exact code you’ll have to build. The MapReduce infrastructure you will build supports the execution of user-defined <code>Map()</code> and <code>Reduce()</code> functions.

As from the original paper: “<code>Map()</code>, written by the user, takes an input pair and produces a set of intermediate key/value pairs. The MapReduce library groups together all intermediate values associated with the same intermediate key K and passes them to the <code>Reduce()</code> function.”

“The <code>Reduce()</code> function, also written by the user, accepts an intermediate key K and a set of values for that key. It merges together these values to form a possibly smaller set of values; typically just zero or one output value is produced per <code>Reduce()</code> invocation. The intermediate values are supplied to the user’s reduce function via an iterator.”

A classic example, written here in pseudocode, shows how to count the number of occurrences of each word in a set of documents:

<pre><code class="language-java">map(String key, String value):    // key: document name    // value: document contents    for each word w in value:        EmitIntermediate(w, "1");reduce(String key, Iterator values):    // key: a word    // values: a list of counts    int result = 0;    for each v in values:        result += ParseInt(v);    print key, value;</code></pre>

Apart from the <code>Map()</code> and <code>Reduce()</code> functions, there is an option to provide a third user-defined <code>Combine()</code> function, if the <code>Reduce()</code> function is commutative and associative.

The <code>Combine()</code> function does partial merging of the data emitted by a single <code>Mapper()</code>, before it is sent to the <code>Reduce()</code> function. More specifically, a <code>Combine()</code> function is executed as many times as the number of unique keys that its respective <code>Map()</code> function will produce.

Typically the functionality of <code>Combine()</code> and <code>Reduce()</code> functions can be very similar. The main difference between a <code>Combine()</code> and a <code>Reduce()</code> function, is that the former merges data from a single <code>Map()</code> function before it is forwarded to a reducer, while the latter from multiple mappers.

We can extend the previous example, by adding a <code>Combine()</code> function, as follows:

<pre><code class="language-java">map(String key, String value):    // key: document name    // value: document contents    for each word w in value:        EmitPrepare(w, "1");combine(String key, Iterator values):    // key: a word    // values: list of counts    int result = 0;    for each v in values:        result += ParseInt(v);    EmitIntermediate(w, result);reduce(String key, Iterator values):    // key: a word    // values: a list of counts    int result = 0;    for each v in values:        result += ParseInt(v);    print key, value;</code></pre>

What’s fascinating about MapReduce is that so many different kinds of relevant computations can be mapped onto this framework. The original paper lists many examples, including word counting (as above), a distributed grep, a URL frequency access counters, a reverse web-link graph application, a term-vector per host analysis, and others.

What’s also quite interesting is how easy it is to parallelize: many mappers can be running at the same time, and, many reducers can be running at the same time. Users don’t have to worry about how to parallelize their application; rather, they just write <code>Map()</code>, <code>Combine()</code> and <code>Reduce()</code> functions and the infrastructure does the rest.

<h2 id="code-overview">Code Overview</h2>

We give you here the <code>mapreduce.h</code> header file that specifies exactly what you must build in your MapReduce library:

<pre><code class="language-c">#ifndef __mapreduce_h__#define __mapreduce_h__// Different function pointer types used by MRtypedef char *(*CombineGetter)(char *key);typedef char *(*ReduceGetter)(char *key, int partition_number);typedef char *(*ReduceStateGetter)(char* key, int partition_number);typedef void (*Mapper)(char *file_name);typedef void (*Combiner)(char *key, CombineGetter get_next);// Simple mode: `get_state` is NULL and `get_next` can be called until you get NULL// Eager mode: `get_state` and `get_next` will only be called once inside the reducer// More details on the modes are provided latertypedef void (*Reducer)(char *key, ReduceStateGetter get_state,                        ReduceGetter get_next, int partition_number);typedef unsigned long (*Partitioner)(char *key, int num_partitions);// External functions: these are what *you must implement*void MR_EmitToCombiner(char *key, char *value);void MR_EmitToReducer(char *key, char *value);// NOTE: Needs to be implemented ONLY for the Eager modevoid MR_EmitReducerState(char* key, char* state, int partition_number);unsigned long MR_DefaultHashPartition(char *key, int num_partitions);void MR_Run(int argc, char *argv[],        Mapper map, int num_mappers,        Reducer reduce, int num_reducers,        Combiner combine,        Partitioner partition);#endif // __mapreduce_h__</code></pre>

The most important function is <code>MR_Run</code>, which takes the command line parameters of a given program, a pointer to a Map function (type <code>Mapper</code>, called <code>map</code>), the number of mapper threads your library should create (<code>num_mappers</code>), a pointer to a Reduce function (type <code>Reducer</code>, called <code>reduce</code>), the number of reducers (<code>num_reducers</code>), a pointer to a Combine function (type <code>Combiner</code>, called <code>combine</code>), and finally, a pointer to a Partition function (<code>partition</code>, described below).

Thus, when a user is writing a MapReduce computation with your library, they will implement a Map function, a Reduce function, and possibly a Combine or Partition function (or both), and then call <code>MR_Run()</code>. The infrastructure will then create threads as appropriate and run the computation. If we do not wish to use a <code>Combine()</code> function, then <code>NULL</code> is passed instead as the value of <code>combine</code> argument. Note that your code should work in both cases (i.e. use the <code>Combine</code> function if a valid function pointer is passed, or entirely skip the combine step if <code>NULL</code> is passed).

One basic assumption is that the library will create <code>num_mappers</code> threads (in a thread pool) that perform the map tasks. Another is that your library will create <code>num_reducers</code> threads to perform the reduction tasks. Finally, your library will create some kind of internal data structure to pass keys and values from mappers to combiners and from combiners to reducers; more on this below.

<h2 id="simple-example-wordcount">Simple Example: Wordcount</h2>

Here is a simple (but functional) wordcount program, written to use this infrastructure:

<pre><code class="language-c">#include &lt;assert.h&gt;#include &lt;stdio.h&gt;#include &lt;stdlib.h&gt;#include &lt;string.h&gt;#include "mapreduce.h"void Map(char *file_name) {    FILE *fp = fopen(file_name, "r");    assert(fp != NULL);    char *line = NULL;    size_t size = 0;    while (getline(&amp;line, &amp;size, fp) != -1) {        char *token, *dummy = line;        while ((token = strsep(&amp;dummy, " t
r")) != NULL) {            MR_EmitToCombiner(token, "1");        }    }    free(line);    fclose(fp);}void Combine(char *key, CombineGetter get_next) {    int count = 0;    char *value;    while ((value = get_next(key)) != NULL) {        count ++; // Emmited Map values are "1"s    }    // Convert integer (count) to string (value)    value = (char*)malloc(10 * sizeof(char));    sprintf(value, "%d", count);    MR_EmitToReducer(key, value);    free(value);}void Reduce(char *key, ReduceStateGetter get_state,            ReduceGetter get_next, int partition_number) {    // `get_state` is only being used for "eager mode" (explained later)    assert(get_state == NULL);    int count = 0;    char *value;    while ((value = get_next(key, partition_number)) != NULL) {        count += atoi(value);    }    // Convert integer (count) to string (value)    value = (char*)malloc(10 * sizeof(char));    sprintf(value, "%d", count);    printf("%s %s
", key, value);    free(value);}int main(int argc, char *argv[]) {    MR_Run(argc, argv, Map, 10,        Reduce, 10, Combine, MR_DefaultHashPartition);}</code></pre>

Let’s walk through this code, in order to see what it is doing. First, notice that <code>Map()</code> is called with a file name. In general, we assume that this type of computation is being run over many files; each invocation of <code>Map()</code> is thus handed one file name and is expected to process that file in its entirety.

In this example, the code above just reads through the file, one line at a time, and uses <code>strsep()</code> to chop the line into tokens. Each token is then emitted using the <code>MR_EmitToCombiner()</code> function, which takes two strings as input: a key and a value. The key here is the word itself, and the token is just a count, in this case, 1 (as a string). It then closes the file.

The <code>MR_EmitToCombiner()</code> function is used to propagate data from a mapper to its respective combiner; it takes key/values pairs from a <strong>single</strong> mapper and temporarily stores them so that the the subsequent <code>Combine()</code> calls (i.e. one for each unique emitted key from its mapper), can retrieve and merge its values for each key.

The <code>Combine()</code> function is invoked for each unique key produced by the respective invocation of <code>Map()</code> function. It first goes through all the values for a specific key using the provided <code>get_next</code> function, which returns a pointer to the value passed by <code>MR_EmitToCombiner()</code> or <code>NULL</code> if all values have been processed. It then merges these values to produce <em>exactly one</em> key/value pair. This pair is emmited to the reducers using the <code>MR_EmitToReducer()</code> function.

Finally, the <code>MR_EmitToReducer()</code> function is another key part of your library; it needs to take key/value pairs from the many different combiners and store them in a way that reducers can access them, given constraints described below. Designing and implementing this data structure is thus a central challenge of the project.

The <code>Reduce()</code> function is invoked once per intermediate key, and is passed the key along with a function that enables iteration over all of the values that produced that same key. To iterate, the code just calls <code>get_next()</code> repeatedly until a NULL value is returned; <code>get_next</code> returns a pointer to the value passed in by the <code>MR_EmitToReducer()</code> function above, or NULL when the key’s values have been processed. The output, in the example, is just a count of how many times a given word has appeared, which is just printed in the standard output.

All of this computation is started off by a call to <code>MR_Run()</code> in the <code>main()</code> routine of the user program. This function is passed the <code>argv</code> array, and assumes that <code>argv[1]</code> … <code>argv[n-1]</code> (with <code>argc</code> equal to <code>n</code>) all contain file names that will be passed to the mappers.

One interesting function that you also need to pass to <code>MR_Run()</code> is the partitioning function. In most cases, programs will use the default function (<code>MR_DefaultHashPartition</code>), which should be implemented by your code. Here is its implementation:

<pre><code class="language-c">unsigned long MR_DefaultHashPartition(char *key, int num_partitions) {    unsigned long hash = 5381;    int c;    while ((c = *key++) != ' ')        hash = hash * 33 + c;    return hash % num_partitions;}</code></pre>

The function’s role is to take a given <code>key</code> and map it to a number, from <code>0</code> to <code>num_partitions - 1</code>. Its use is internal to the MapReduce library, but critical. Specifically, your MR library should use this function to decide which partition (and hence, which reducer thread) gets a particular key/list of values to process. For some applications, which reducer thread processes a particular key is not important (and thus the default function above should be passed in to <code>MR_Run()</code>); for others, it is, and this is why the user can pass in their own partitioning function as need be.

<h2 id="considerations">Considerations</h2>

Here are a few things to consider in your implementation:

<ul>

 <li><strong>Thread Management</strong> This part is fairly straightforward. You should create <code>num_mappers</code> mapping threads, and assign a file to each <code>Map()</code> invocation in some manner you think is best (e.g., Round Robin, Shortest-File-First, etc.). Which way might lead to best performance? You should also create <code>num_reducers</code> reducer threads at some point, to work on the map’d output (more details below).</li>

</ul>

<ul>

 <li><strong>Memory Management</strong> Another concern is memory management. The <code>MR_Emit*()</code> functions are passed a key/value pair; it is the responsibility of the MR library to make copies of each of these. However, avoid making too many copies of them since the goal is to design an efficient concurrent data processing engine. Then, when the entire mapping and reduction is complete, it is the responsibility of the MR library to free everything.</li>

 <li><strong>API</strong> You are allowed to use data structures and APIs <strong>only</strong> from <code>stdio.h</code>, <code>stdlib.h</code>, <code>pthread.h</code>, <code>string.h</code>, <code>sys/stat.h</code>, and <code>semaphore.h</code>. If you want to implement some other data structures you can add that as a part of your submissions.</li>

</ul>

<h2 id="partitioning">Partitioning</h2>

Assuming that a valid combine function is passed, there are two places where it is necessary to store intermediate results in order to allow different functions to access them: (a) store mappers output to be accessed by combiners, and (b) store combiners output to be accessed by reducers. You should think which data structure is appropriate for each scenario, and what properties you need to guarantee in your implementation. For example, can you re-use the same data structure for both cases? Should both data structures support concurrent access? It is essential to have clearly answered the above questions before you start writing any code.

Depending on whether you are working alone or in a group, there are different requirements regarding the functionality of reducers:

<ul>

 <li><strong>One person group</strong>: You should launch the reducer threads once all the combiners have finished (i.e. <strong>simple mode</strong>). Waiting for all combiners (and thus mappers) to finish means that the mappers output data would be available in your intermediate data structure. Then, each reducer thread should start invoking <code>Reduce()</code> functions, where each one will process the data for a single intermediate key. In this case the <code>get_state</code> (of type <code>ReduceStateGetter</code>) should always be <code>NULL</code>. For the simple mode, you <em>do not</em> need to implement the <code>MR_EmitReducerState()</code> function.</li>

 <li><strong>Two person group</strong>: You should implement additional functionality (i.e. <strong>eager mode</strong>) that will allow reducers to process output from combiners “on-the-fly”. This means that mapper and reduce threads should be running simultaneously, and thus be launched from the beginning. Recall that each reducer thread is responsible for a set of keys that are assigned to it (i.e. partition number). Therefore, each reducer thread should wait until data becomes available (for any key that belongs into that set) in the intermediate data structure. This data is “pushed” by the combiners using the <code>MR_EmitToReducer()</code> function. When that occurs, the reducer thread must (wake up if it sleeps and) remove the newly arrived data from the data structure. Then, it should <em>immediately</em> process it by running the appropriate <code>Reduce()</code> function. Finally, it will get the merged partial result from the <code>ReduceStateGetter</code> and store the updated partial result using the <code>MR_EmitReduceState()</code>. If no data is available at a certain point, it should go to sleep again, in order to yield resources to the mappers. Note that it is entirely possible to run the <code>Reduce()</code> multiple times for the same key, as data from different mappers will keep being inserted into the intermediate data structure. When the mappers (or combiners) finish, and there are no more values to merge, the <code>get_next()</code> should return <code>NULL</code>. This can be used to “notify” the reducer that the final result is ready, and therefore can be printed.</li>

</ul>

Below is an example of how the word count <code>Reduce()</code> function might look like specifically for the eager mode:

<pre><code class="language-c">void Reduce(char *key, ReduceStateGetter get_state,            ReduceGetter get_next, int partition_number) {    char *state = get_state(key, partition_number);    int count = (state != NULL) ? atoi(state) : 0;    char *value = get_next(key, partition_number);    if (value != NULL) {        count += atoi(value);        // Convert integer (count) to string (value)        value = (char*)malloc(10 * sizeof(char));        sprintf(value, "%d", count);        MR_EmitReducerState(key, value, partition_number);        free(value);    }    else {        printf("%s %d
", key, count);    }}</code></pre>

At first, <code>get_state</code> is called to retrieve the state (result) of potential previous <code>Reduce</code> invocations. If no state for the specified key exists, <code>get_state</code> should return <code>NULL</code>. Next, the <code>get_next</code> function retrieves the newly arrived data from a mapper. Then, the two values are merged, and finally the new state (result) is emitted (call to <code>MR_EmitReducerState</code>). Once the last value is received (i.e. get_next returns NULL), the Reducer prints the final count using printf.

<h2 id="testing">Testing</h2>

We will soon provide few test applications to test the correctness of your implementation. We will post on Piazza when they become available.

<h2 id="grading">Grading</h2>

Your code should turn in <code>mapreduce.c</code> which implements the following functions correctly and efficiently: <code>MR_EmitToCombiner()</code>, <code>MR_EmitToReducer()</code>, <code>MR_EmitReducerState()</code> (if working in groups) and <code>MR_Run()</code>. You should also have implemented two versions of <code>get_next()</code> function (one for the <code>Combiner()</code> and one for the <code>Reducer()</code>), as well as a single version of <code>get_state()</code> function.

Your program will be compiled using the following command, where test.c is a test program that contains a main function and the Mapper/Reducer (and potentially the Combiner) as shown in the wordcount example above:

<pre><code class="language-bash">gcc -o mapreduce test.c mapreduce.c mapreduce.h -Wall -Werror -pthread -O</code></pre>

It will also be valgrinded to check for memory errors.

Your code will first be measured for correctness, ensuring that it performs the maps and reductions correctly. If you pass the correctness tests, your code will be tested for performance to see if it runs within suggested time limits.

<h2 id="submitting-your-implementation">Submitting Your Implementation</h2>

Please read the following information carefully. <strong>We use automated grading scripts, so please adhere to these instructions and formats if you want your project to be graded correctly.</strong>

If you choose to work in pairs, only <strong>one</strong> of you needs to submit the source code. But, <strong>both</strong> of you need to submit an additional <strong><code>partner.login</code></strong> file, which contains <strong>one</strong> line that has the <strong>CS login of your partner</strong> (and nothing else). If there is no such file, we are assuming you are working alone. If both of you turn in your codes, we will just randomly choose one to grade.

To submit your solution, copy your files into <code>~cs537-1/handin/&lt;cs-login&gt;/p4a/</code>. One way to do this is to navigate to your solution’s working directory and execute the following command:

<pre><code class="language-bash">cp -r . ~cs537-1/handin/$USER/p4a/</code></pre>

Consider the following when you submit your project:

<ul>

 <li>If you use any slip days you have to create a <code>SLIP_DAYS</code> file in the <code>/&lt;cs-login&gt;/p4a/</code> directory otherwise we use the submission on the due date.</li>

 <li>Your files should be directly copied to <code>~cs537-1/handin/&lt;cs-login&gt;/p4a/</code> directory. Having subdirectories in <code>handin/&lt;cs-login&gt;/p4a/</code> is <strong>not acceptable</strong>.</li>

</ul>

<h2 id="collaboration">Collaboration</h2>

This project is to be done in groups of size one or two (not three or more). Now, within a group, you can share as much as you like. However, copying code across groups is considered cheating.

If you are planning to use <code>git</code> or other version control systems (<em>which are highly recommended for this project</em>), just be careful <strong>not</strong> to put your codes in a public repository.