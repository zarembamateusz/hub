# Common issues

If your issue is not listed here, please search the [github issues](https://github.com/tensorflow/hub/issues) before filling a new one.


## Cannot download a module

In the process of using a module from an URL there are many errors that can show
up due to the network stack. Often this is a problem specific to the machine
running the code and not an issue with the library. Here is a list of the common
ones:

* **"EOF occurred in violation of protocol"** - This issue is likely to be
generated if the installed python version does not support the TLS requirements
of the server hosting the module. Notably, python 2.7.5 is known to fail
resolving modules from tfhub.dev domain. **FIX**: Please update to a newer
python version.

* **"cannot verify tfhub.dev's certificate"** - This issue is likely to be
generated if something on the network is trying to act as the dev gTLD.
Before .dev was used as a gTLD, developers and frameworks would sometimes use
.dev names to help testing code. **FIX:** Identify and reconfigure the software
that intercepts name resolution in the ".dev" domain.

If the above errors and fixes do not work, one can try to manually download a
module by simulating the protocol of attaching `?tf-hub-format=compressed`
to the URL to download a tar compressed file that has to be manually decompressed
into a local file. The path to the local file can then be used instead of the
URL. Here is a quick example:

```bash
# Create a folder for the TF hub module.
$ mkdir /tmp/moduleA
# Download the module, and uncompress it to the destination folder. You might want to do this manually.
$ curl -L "https://tfhub.dev/google/universal-sentence-encoder/2?tf-hub-format=compressed" | tar -zxvC /tmp/moduleA
# Test to make sure it works.
$ python
> import tensorflow_hub as hub
> hub.Module("/tmp/moduleA")
```

## Running inference on a pre-initialized module

If you are applying a module over data multiple times (e.g. to serve user
requests) you should use TensorFlow Session.run to avoid the overhead of
constructing and initializing parts of the graph multiple times.

Assuming your use-case model is **initialization** and subsequent **requests**
(for example Django, Flask, custom HTTP server, etc.), you can set-up the
serving as follows:

* In the initialization part:
    * Build the graph with a **placeholder** - entry point into the graph.
    * Initialize the session.

```python
import tensorflow as tf
import tensorflow_hub as hub

# Create graph and finalize (finalizing optional but recommended).
g = tf.Graph()
with g.as_default():
  # We will be feeding 1D tensors of text into the graph.
  text_input = tf.placeholder(dtype=tf.string, shape=[None])
  embed = hub.Module("https://tfhub.dev/google/universal-sentence-encoder/2")
  embedded_text = embed(text_input)
  init_op = tf.group([tf.global_variables_initializer(), tf.tables_initializer()])
g.finalize()

# Create session and initialize.
session = tf.Session(graph=g)
session.run(init_op)
```

* In the request part:
    * Use the session to feed data into the graph through the placeholder.

```python
result = session.run(embedded_text, feed_dict={text_input: ["Hello world"]})
```
