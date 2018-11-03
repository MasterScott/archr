# archr

Traditionally, binary analysis has been implicitly _program-centric_, meaning that the atomic unit of concern is the binary being analyzed.
This assumption is usually implicit: `angr.Project` is instantiated with the binary in question, `afl` launches the binary itself, generally hyper-modified to make it easier to fuzz, and so on.

However, outside of the CGC, programs do not exist in a vacuum.
Specific library versions, values in configuration files, environment variables, and a myriad other factors combine with the program binary itself to make a unique whilistic _target_, and in many cases, it is that target that needs to be analyzed, not just the program itself.
This is specifically true for analysis that need extreme accuracy, such as automatic exploit generation.

`archr` is an implementation of such a _target-centric_ analysis paradigm.
It consists of two main concepts: `Targets`, which describe the specification of the target itself, how it is configured, how it will be launched, and how it would be interacted with, and `Bows`, which specialize targets for specific analysis actions, such as tracing, symbolic execution, and so on.
To accomplish their tasks, Bows might inject Arrows (i.e., qemu-user, gdbserver, and so on) into the target.

We have the following Targets:

* DockerImageTarget, which takes a description of the target in the form of a docker image
* LocalTarget, which just describes running the target in the local system

The following Bows exist:

- DataScoutBow (will grabs the memory map, environment, and auxv of the process, exactly as it is at launch)
- AngrProjectBow (can create an angr project with the right libs at the right offsets)
- AngrStateBow (can create a angr states with the right env, args, and fs)
- QEMUTraceBow (does qemu tracing of the target)
- GDBServerBow (launches the target in a gdbserver)
- (TODO) RRBow (does rr tracing of the target)
- (TODO) AFLBow (launches AFL of the target)

## Using archr

To use archr, one must first create a Target.
First, build a docker image that launches your target.
Here is an example dockerfile for a `docker-cat` image:

```
from ubuntu:latest
entrypoint ["/bin/cat"]
```

Then, load it as a target:

```
import archr
t = archr.targets.DockerImageTarget('docker-cat').build()
```

And _viola!_, your target is ready to use.
archr will automatically figure out how your binary runs inside your target, and then you can launch and interact with it:

```
t.start()
assert t.run_command(stdin=subprocess.DEVNULL).wait() == 0
t.stop()
```

archr makes heavy use of `with` contexts, which will help clean up resources.
Embrace them.
For example, you can:

```
with t.start():
	with t.run_context() as p:
		print(p,"is a subprocess.Popen object!")
		p.stdin.write("hello")
		assert p.stdout.read(5) == "hello"
```

There is even a context that will allow you to temporarily replace files on the target!

```
with t.start():
	with t.replacemenet_context("/etc/passwd", "hahaha"), t.run_context(args_suffix=["/etc/passwd"]) as p:
		assert p.stdout.read() == "hahaha"
	assert t.run_command(args_suffix=["/etc/passwd"]).stdout.read() != "hahaha"
```

And even one that will _temporarily replace the target binary's code with shellcode_:

```
with t.start():
	with t.shellcode_context(asm_code="mov rax, 60; mov rdi, 42; syscall") as p:
		assert p.wait() == 42
```

You can retrieve files from the target with `retrieve_contents`, `retrieve_paths`, and `retrieve_glob`, inject files with `inject_path`, `inject_contents`, and so on, get network endpoints using `ipv4_address`, `udp_ports`, and `tcp_ports`, and some other interesting stuff!
You can also make a `LocalTarget` to just run stuff on your host, and it is almost perfectly interchange with a DockerTarget:

```
import archr
t = archr.targets.LocalTarget(["/bin/cat"]).build()
```

To figure out how to run the binary, `LocalTarget` takes at least an argv list.
You can also pass in an env.

Keep in mind that since some of the above examples need write access to files, you will need to use writable files instead of `/etc/passwd` and `/bin/cat`.

## Caveats

Some caveats at the moment:

- archr does not handle string-specified (as opposed to array-specified) entrypoint directives in the docker file.
  This isn't hard; we just haven't gotten around to it (see issue #1).
- Because angr needs a `chroot_fs` to be provided to it, archr currently requires sudo rights to mount the DockerImageTarget's filesystem in an accessible folder.
  This can be resolved with some angr work.
