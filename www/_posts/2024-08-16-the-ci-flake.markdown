---
layout: post
title: "The CI Flake"
tags: Work Programming
permalink: /blog/the-ci-flake/
---
I analyzed a [flaky test failure](https://buildkite.com/materialize/test/builds/88495#01915a27-cdd8-422b-b459-32de02308820) in our [Materialize](https://materialize.com/) [CI](https://buildkite.com/materialize) today:

{% highlight text %}
$ docker compose up -d --scale default=0 default
no such service: default
mzcompose: error: running docker compose failed (exit status 1)
{% endhighlight %}

I had seen this error already once or twice in the last year, but it was incredibly rare in our Continuous Integration (CI) runs, and never happened locally. As usual, there were more pressing product issues to debug, so I never looked into it. But last week I switched most of our CI tests to run on [Hetzner Cloud](https://www.hetzner.com/cloud/) instead of [AWS](https://aws.amazon.com/) to save some money. Suddenly this issue started occurring more often in CI, so my thinking was that it must somehow be timing-dependent.

<!--more-->

Before investigating my first instinct was that we are not writing the `docker-compose.yaml` file properly, which had already led to [flaky calls to Docker Compose](https://github.com/MaterializeInc/materialize/issues/23481) before ([source code](https://github.com/MaterializeInc/materialize/blob/6809a54012c4d67a1088417662adea63c840c5a6/misc/python/materialize/mzcompose/composition.py#L326)):
{% highlight python %}
file = self.files.get(thread_id)
if not file:
    file = TemporaryFile(mode="w")
    os.set_inheritable(file.fileno(), True)
    yaml.dump(self.compose, file)
    self.files[thread_id] = file
{% endhighlight %}

So [yesterday](https://github.com/MaterializeInc/materialize/pull/29041) I added a `file.flush()` after the `yaml.dump` and hoped to be done with it. This morning I woke up and the issue was still occurring!

Based on the logged `docker` call this should be the code causing the problem ([source code](https://github.com/MaterializeInc/materialize/blob/main/misc/python/materialize/cli/mzcompose.py#L591-L608)):
{% highlight python %}
def handle_composition(
        self, args: argparse.Namespace, composition: Composition
    ) -> None:
        if args.workflow not in composition.workflows:
            # Restart any dependencies whose definitions have changed.
            # This is Docker Compose's default behavior for `up`, but
            # not for `run`, which is a constant irritation that we
            # paper over here. The trick, taken from Buildkite's
            # Docker Compose plugin, is to run an `up` command that
            # requests zero instances of the requested service.
            if args.workflow:
                composition.invoke(
                    "up",
                    "-d",
                    "--scale",
                    f"{args.workflow}=0",
                    args.workflow,
                )
            super().handle_composition(args, composition)
        else:
            [...]
{% endhighlight %}
Running the test in an endless loop locally had no success of reproducing the issue:
{% highlight bash %}
while true; do
    bin/mzcompose --find skip-version-upgrade down
    bin/mzcompose --find skip-version-upgrade run default || break
done
{% endhighlight %}

I modified the conditional in the above Python code to be `if True:` and surely that reproduces the issue:

{% highlight text %}
$ bin/mzcompose --find skip-version-upgrade run default
$ docker compose up -d --scale default=0 default
no such service: default
mzcompose: error: running docker compose failed (exit status 1)
{% endhighlight %}

So how is it possible to be landing in this code path sometimes? I checked how the `composition.workflows` are generated and it looks quite complex indeed ([source code](https://github.com/MaterializeInc/materialize/blob/6809a54012c4d67a1088417662adea63c840c5a6/misc/python/materialize/mzcompose/composition.py#L136-L152)):

{% highlight python %}
mzcompose_py = self.path / "mzcompose.py"
if mzcompose_py.exists():
    spec = importlib.util.spec_from_file_location("mzcompose", mzcompose_py)
    assert spec
    module = importlib.util.module_from_spec(spec)
    assert isinstance(spec.loader, importlib.abc.Loader)
    loader.composition_path = self.path
    spec.loader.exec_module(module)
    loader.composition_path = None
    self.description = inspect.getdoc(module)
    for name, fn in getmembers(module, isfunction):
        if name.startswith("workflow_"):
            # The name of the workflow is the name of the function
            # with the "workflow_" prefix stripped and any underscores
            # replaced with dashes.
            name = name[len("workflow_") :].replace("_", "-")
            self.workflows[name] = fn
{% endhighlight %}

Since we are reading and analyzing the `mzcompose.py` file, something might be running in parallel and corrupting/overwriting it. If I empty the file I can indeed reproduce the issue with the original code:

{% highlight text %}
$ echo > test/skip-version-upgrade/mzcompose.py
$ bin/mzcompose --find skip-version-upgrade run default
$ docker compose up -d --scale default=0 default
no such service: default
mzcompose: error: running docker compose failed (exit status 1)
{% endhighlight %}

It feels like I'm getting closer, but nothing should be running in parallel, even in CI each agent is running in isolation, the `git pull` is long finished at this point based on the CI logs, so it should not interfere. As an additional frustration, I've been wondering why this flake is mostly (but not only) occurring when running this test? The `mzcompose.py` file in question is one of the simplest ones around.

Until here it felt like I was getting closer, but now I had no more leads. I went as far as searching for Python bugs around inheritance that could explain this issue, but if Python was buggy in such basic behavior, this code definitely wouldn't be the first one to run into it. Could it be a cosmic ray?!

Since this issue seemed unexplainable without the universe conspiring against me, I went back to the basics and checked what else is running in CI and I found something extremely suspicious just before the actual test execution in our CI script ([source code](https://github.com/MaterializeInc/materialize/blob/6809a54012c4d67a1088417662adea63c840c5a6/ci/plugins/mzcompose/hooks/command#L63-L69)):

{% highlight bash %}
# Start dependencies under a different heading so that the main
# heading is less noisy. But not if the service is actually a
# workflow, in which case it will do its own dependency management.
if ! mzcompose --mz-quiet list-workflows | grep -q "$service"; then
    ci_collapsed_heading ":docker: Starting dependencies"
    mzcompose up -d --scale "$service=0" "$service"
fi
{% endhighlight %}

Oh no! There is another `$service=0` call here, my entire investigation so far might have been useless! From looking at the CI output it's not clear which one is actually running, but the fact that I've never seen the flake locally points to the new suspect. After all this code is only running in CI, and not locally, all in the name of cleaner CI logs! Checking the CI output there is a strange line occurring:

{% highlight text %}
write /dev/stdout: broken pipe
{% endhighlight %}

Well, there is a pipe between the `mzcompose` and `grep` processes, but why would it break? Let's run the `mzcompose` command in isolation:

{% highlight text %}
$ bin/mzcompose --find skip-version-upgrade list-workflows
default
test-version-skips
{% endhighlight %}

It lists all workflows, of which we have two. The `grep` command is only looking for the `default` line. Apparently, `grep` is smart enough to exit immediately when it finds a match when used with `-q` ([manpage](https://man7.org/linux/man-pages/man1/grep.1.html)):
{% highlight text %}
-q, --quiet, --silent
       Quiet; do not write anything to standard output. Exit
       immediately with zero status if any match is found, even if
       an error was detected. Also see the -s or --no-messages option.
{% endhighlight %}

Easy enough to verify, I ran the `true` command, which exits instantly:
{% highlight text %}
$ bin/mzcompose --find skip-version-upgrade list-workflows | true
Exception ignored in: <_io.TextIOWrapper name='<stdout>' mode='w' encoding='utf-8'>
BrokenPipeError: [Errno 32] Broken pipe
{% endhighlight %}


So this `broken pipe` output is indeed explained by `grep` exitting early. But why does the `mzcompose` process breaking lead to the code in the `if`'s body being executed? Normally the first command in a pipe failing would be ignored, since the second command succeeded, and I'm seeing the return code being `0` here. Checking the beginning of the script reveals that we use the `pipefail` setting:

{% highlight bash %}
set -euo pipefail
{% endhighlight %}

Bash's [manpage](https://www.man7.org/linux/man-pages/man1/bash.1.html) explains:

> The return status of a pipeline is the exit status of the last command, unless the pipefail option is enabled.  If pipefail is enabled, the pipeline's return status is the value of the last (rightmost) command to exit with a non-zero status, or zero if all commands exit successfully.

Typically, Python buffers its output when it doesn't write directly to a terminal, so I still can't reproduce the issue with `grep` locally. Then I remembered that we had an issue with our Python tests' stdout and stderr output being garbled, which [made us enable](https://github.com/MaterializeInc/materialize/pull/17846) `PYTHONUNBUFFERED` in our ci-builder [Dockerfile](https://github.com/MaterializeInc/materialize/blob/078ebd39d3dec03383be74eca862d4820116de59/ci/builder/Dockerfile):

{% highlight docker %}
# Ensure that all python output is unbuffered, otherwise it is not
# logged properly in Buildkite
ENV PYTHONUNBUFFERED=1
{% endhighlight %}

With this revelation the flake was easy enough to reproduce. Just running this would fail occasionally, still less than 1% on my system:
{% highlight text %}
$ set pipefail
$ PYTHONUNBUFFERED=1 bin/mzcompose --find skip-version-upgrade list-workflows | grep -q "default"
BrokenPipeError: [Errno 32] Broken pipe
{% endhighlight %}

Adding a small `sleep` between each `print` leads to a reliable failure:
{% highlight diff %}
diff --git a/misc/python/materialize/cli/mzcompose.py b/misc/python/materialize/cli/mzcompose.py
index 3a2f5dbb81..725150ed00 100644
--- a/misc/python/materialize/cli/mzcompose.py
+++ b/misc/python/materialize/cli/mzcompose.py
@@ -319,6 +319,7 @@ class ListWorkflowsCommand(Command):
         composition = load_composition(args)
         for name in sorted(composition.workflows):
             print(name)
+            time.sleep(0.1)


 class DescribeCommand(Command):
{% endhighlight %}

Without `PYTHONUNBUFFERED` it would have taken us 8 KiB of output before we'd run into the broken pipe, none of our tests has that many workflows.

That concludes this long investigation into what turned out to be an interesting flake that we've been seeing in CI! The [trivial fix](https://github.com/MaterializeInc/materialize/pull/29070) is removing the `-q` from the `grep` call, but we could also have disabled `pipefail` for that one pipe or we could have unset `PYTHONUNBUFFERED`.
One lesson from this debugging session is that even simple behaviors can combine in unexpected ways to create strange flaky failures. In this case all of these conspired to make this CI flake possible:

1. Misdirection: Run different code in CI compared to locally
2. Switch to another kind of server with slightly different performance characteristics
3. Disable Python's output buffering
4. Enable Bash's `pipefail`
5. Make `grep` run quietly

In hindsight I realized why this specific `skip-version-upgrade/mzcompose.py` was having the flake most often. It is one of the smallest test files:
{% highlight text %}
$ wc -l test/*/mzcompose.py | sort -n
      39 test/pg-rtr/mzcompose.py
      43 test/mysql-rtr/mzcompose.py
      67 test/debezium/mzcompose.py
[...]
      85 test/skip-version-upgrade/mzcompose.py
[...]
     966 test/0dt/mzcompose.py
    1333 test/bounded-memory/mzcompose.py
    1916 test/limits/mzcompose.py
    4767 test/cluster/mzcompose.py
{% endhighlight %}
But other than the test definitions that are even smaller and never had this `no such service: default` error, the `skip-version-upgrade/mzcompose.py` file contains multiple workflows, so the smaller files could not run into the issue at all:
{% highlight text %}
$ grep "def workflow_" test/*/mzcompose.py | cut -d":" -f1 | \
  uniq -c | sort -n | grep -v "   1 "
   2 test/skip-version-upgrade/mzcompose.py
   3 test/canary-environment/mzcompose.py
[...]
  57 test/cluster/mzcompose.py
{% endhighlight %}

