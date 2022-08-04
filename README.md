This repo demonstrates a bug in Pants's docker support.

See [here](https://github.com/pantsbuild/pants/issues/16101) for original bug report.

To reproduce:

```
$ ./pants package infra/docker/build_runner_child:build_runner_child
17:44:44.68 [INFO] Completed: Building docker image build_runner:build_runner_pants
17:44:45.27 [INFO] Completed: Building docker image build_runner_child:build_runner_child
17:44:45.27 [INFO] Built docker image: build_runner_child:build_runner_child
Docker image ID: sha256:b5cf6f61770c707108a4f1d00e3b090ba0b9e25bcf6864e555b7b01e8f6eb05d

$ docker run -it build_runner_child:build_runner_child ls
test_file_parent1.txt  test_file.txt
```

This is as expected.

Now edit infra/docker/build_runner/Dockerfile:

```
$ sed -i "" s/test_file_parent1/test_file_parent2/g infra/docker/build_runner/Dockerfile
```

And run again:

```
$ ./pants package infra/docker/build_runner_child:build_runner_child
17:49:30.35 [INFO] Canceled: Building docker image build_runner:build_runner_pants
17:49:31.43 [INFO] Completed: Building docker image build_runner_child:build_runner_child
17:49:31.43 [INFO] Canceled: Building docker image build_runner:build_runner_pants
17:49:33.04 [INFO] Completed: Building docker image build_runner:build_runner_pants
17:49:33.04 [INFO] Built docker image: build_runner_child:build_runner_child
Docker image ID: sha256:b5cf6f61770c707108a4f1d00e3b090ba0b9e25bcf6864e555b7b01e8f6eb05d

$ docker run -it build_runner_child:build_runner_child ls
test_file_parent1.txt  test_file.txt
```

This is *not* as expected.

Running a second time post-edit gives us the expected output:

```
$ ./pants package infra/docker/build_runner_child:build_runner_child
17:50:37.36 [INFO] Completed: Building docker image build_runner_child:build_runner_child
17:50:37.36 [INFO] Canceled: Building docker image build_runner:build_runner_pants
17:50:38.16 [INFO] Completed: Building docker image build_runner:build_runner_pants
17:50:38.17 [INFO] Built docker image: build_runner_child:build_runner_child
Docker image ID: sha256:3c74db4f239b2b63c7cb317855cbbce03e843ca330868f1742fa553d412c6590

$ docker run -it build_runner_child:build_runner_child ls
test_file_parent2.txt  test_file.txt
```

Note that the first (badly-behaved) run has an extra log line for 
`[INFO] Canceled: Building docker image build_runner:build_runner_pants`
that the second one does not.

Note that a daemon restart after the initial run causes this not to happen.

Looking at the trace (`-ltrace`), it appears to show that the child and parent Docker
build processes are spawned concurrently, the parent is canceled (because the Dockerfile change
has been picked up so the speculation canceled, I assume), then spawned again, then canceled
again, then spawned a third time and allowed to complete.

NOTE: To reproduce while running Pants from source use:

```
ENABLE_PANTSD=true ./pants_from_sources package infra/docker/build_runner_child:build_runner_child
```
