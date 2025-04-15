## Instructions for Undergrads

The following are instructions for building an OpenFaaS function for use on a RISC-V faasd deployment.

**Background**

OpenFaaS is a platform for deploying serverless functions packaged as Docker containers. It can be utilized with varied backends acting as the orchestrator, such as Kubernetes or Docker Swarm. In our case, we are using faasd, the lightest weight implementation that simply uses containerd to manage all of its images. 

Neither OpenFaaS nor faasd support RISC-V officially, but we have built a working installation of both that works both in QEMU and on OpenPiton running on FPGA. This image is included in the materials I’ve sent over; it’s a disk image file called faasd.img. It can be booted with the following command:

qemu-system-riscv64 -nographic -machine virt -m 8G -bios ./qemu-boot/fw_jump.elf -kernel ./qemu-boot/vmlinux-6.8.9-riscv64 -initrd ./qemu-boot/initrd.img-6.8.9-riscv64 -object rng-random,filename=/dev/urandom,id=rng0 -device virtio-rng-device,rng=rng0 -append "console=ttyS0 rw root=/dev/vda1" -device virtio-blk-device,drive=hd0 -drive file=./faasd.img,format=raw,id=hd0 -device virtio-net-device,netdev=usernet -netdev user,id=usernet,hostfwd=tcp::22222-:22

There is only a root account on this machine. Password is root.

One of the next necessary steps is to build a set of functions to act as benchmark for our system. These will be a set of Docker images built for RISC-V that we can invoke using OpenFaaS, at which point we can measure performance characteristics.

Some previous work done at Princeton produced some code that we can use to evaluate FaaS performance. This work can be found at the following repo, and the functions specifically are under the functions folder: https://github.com/PrincetonUniversity/faas-profiler. There are a couple of hurdles that need to be overcome for us to use them.

The first is that these functions (and their accompanying instructions/READMEs) are packaged up to be used with OpenWhisk, not OpenFaaS. We are going to need to take the actual logic from each of these and migrate over to the Docker container-based system that OpenFaaS uses.

The second is of course the fact that these functions have never been built for RISC-V before, and OpenFaaS obviously doesn’t have a robust ecosystem for building RISC-V functions since there is not official RISC-V release! Luckily, I have done much of the legwork on this end, at least for Python.

There are two sets of benchmarks in the faas-profiler repo, applications and microbenchmarks. We are going to start with the applications and then do the microbenchmarks. I would start with the Python ones as they will be easier than the NodeJS ones.

**Getting Started**

To build the images, you need a RISC-V Docker installation. Installing Docker on the same system as faasd will result in conflicts between the two, so there is another bootable disk image included in the materials called function_builder.img that has Docker set up and ready to go without faasd. I can be booted with the following:

qemu-system-riscv64 -nographic -machine virt -m 8G -bios ./qemu-boot/fw_jump.elf -kernel ./qemu-boot/vmlinux-6.8.9-riscv64 -initrd ./qemu-boot/initrd.img-6.8.9-riscv64 -object rng-random,filename=/dev/urandom,id=rng0 -device virtio-rng-device,rng=rng0 -append "console=ttyS0 rw root=/dev/vda1" -device virtio-blk-device,drive=hd0 -drive file=./function_builder.img,format=raw,id=hd0 -device virtio-net-device,netdev=usernet -netdev user,id=usernet,hostfwd=tcp::22222-:22

There is a user named builder on this machine, password builder. Root password is still root.

The flow of your work will then basically be: boot the function builder, add in your code and set up your Dockerfile, build the image, push it to Dockerhub, shut down the function builder, boot up the faasd system, pull the built image from Dockerhub, and test. You can use a personal Dockerhub account for now (authenticate with docker login) and we will worry about getting all of these under a single roof once you get your functions working.

OpenFaaS supplies templates for functions based on various languages, and that will be your starting point for porting over the faas-profiler functions. 

For a Docker image to be an OpenFaaS function, the Dockerfile needs to be based on (using the FROM directive) a particular base image known as the OpenFaaS “watchdog.” We’ve got the watchdog recompiled for RISC-V and available on Dockerhub at:  huangderful/openfaas-watchdog:latest. 

Below is a sort of “hello world” for bringing up a RISC-V Python function with OpenFaaS.

**Building with Python Language Template**

In the function builder:

faas-cli new --lang python3 your-function-name

This will create a new YAML file for the function and creates a “function” folder with a “handler” python file in it. The handler python file is the entry point for your business logic. By default it will receive the body of an incoming HTTP request as a string. The default handler just returns this value, so if you build this function as is, it will just print the string you passed in on the console.

This command also pulls the templates repo from GitHub. The Dockerfiles in this template folder are what the build call will use to create the actual function image, and thus this is where we will insert the RISC-V watchdog. We do so by replacing or editing the Dockerfile in the subfolder whose name corresponds to the language template we used, in this case, python3. The following is a link to a Dockerfile for a basic python image that is known to work:  [Dockerfile.python.template](https://github.com/huangderful/DockerfilesForRiscv64/raw/main/Dockerfile.python.template)

Once your code and Dockerfile are in place, call:

faas-cli build -f ./your-function-name.yml

This will build a docker image of your function. It requires Docker to be installed.

To push this function to Dockerhub, you need to tag the image and then push it. Tag it with:

docker tag your-function-name:latest dockerhub-username/your-function-or-repo-name:latest

Once it’s tagged, push it with:

docker push dockerhub-username/your-function-or-repo-name:latest

To run the function, call the following on the faasd machine:

faas-cli deploy --name function-name --image docker.io/dockerhub-username/your-function-or-repo-name

Once it’s done, you can invoke your function using curl or faas-cli, for example:

curl 127.0.0.1:8080/function/function-name -d "test request body string"

or

echo "test request body string" | faas-cli invoke function-name
