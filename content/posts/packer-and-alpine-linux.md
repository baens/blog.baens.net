Well I had recently had to get a packer image build for a work project. What I needed was to run a VM with as small as footprint as possible, that I could put on an offline laptop (...just, don't ask, long story). So I figured I would create an image that we could use that would use the alpine linux image as the base.

Now to make this more complicated, I wanted to be able to debug the build process on a Mac laptop, because that is my primary work machine. But the build pipelines were running Linux. So I had to make sure things work in both environments.

To make this possible, I setup QEMU as the hypervisor to allow me to run this machine. This means that I will be using the QEMU build in packer and make that work across the two environments.

What exactly does the QEMU setup look like, well, using the newer packer file format, here is a base file and what it looks like:

```
TODO: basic code of a alpine linux builder
```

Now note, this file will give you a basic alpine linux image, that can run under QEMU. It is a disk formatted in the qcow2 format, and what ever hypervisor can use that, you can use it.

Now that we have something basic, we will now need to expand this to work on Linux and Mac. Right now, this would just work on Mac because the terminal display is using the Mac options. So let's do that, here is what we would change:

```
TODO: files that are needed to allow multiple hosts
```
