+++
title = 'Manylinux 32 and 64 bit'
date = 2025-07-16
draft = false
+++


If you want to build python packages that include both 32 and 64-bit libraries
you'll probably start by reaching for a [manylinux](https://github.com/pypa/manylinux)
docker container. However, you'll find that while there are manylinux
containers with 32-bit toolchains (i686) and for 64-bit ones (x64\_64), there
aren't containers that include both. Setting one up for manylinux2014 isn't too
hard, but it's not trivial or obvious either, so I've made this post to walk
through it.

Note: If you don't care about the details and just want the container now, you
can stop reading here and use the published image: [whenning42/manylinux2014\_x86\_64\_i386](https://hub.docker.com/r/whenning42/manylinux2014_x86_64_i386).

Also before we get into the container's details, it's fair to ask why would
someone want to build a package that includes both 32 and 64-bit libraries.
In my case, my package [libtimecontrol](https://pypi.org/project/libtimecontrol/)
targets 64-bit linux, but it LD\_PRELOADs libraries into other processes, some
of which will be 32-bit, and so it needs to include 32-bit libraries in its
64-bit build. If you want to hear more about the `libtimecontrol` package,
my next post will be on it, so stay tuned.

With all of that out of the way, let's get into how we set up a manylinux
container for 32 and 64-bit builds. The part of the manylinux container that
builds native code is the devtoolset. It's really cool in that it's backported
from a recent version on to the really old manylinux containers. This allows
you to do cool things like use C++17 or C++20 inside of a container based
on a distribution from 2014.

So, to get our container building 32 and 64-bit libraries, we need to get 32
and 64-bit versions of the devtoolset in to our container. We can get one for
free by basing our container on a working manylinux container, in our case
we'll go with manylinux2014\_x86\_64. Then we just need to get the 32-bit
toolchain. By looking through the official manylinux implementation, we see
that ([src](https://github.com/pypa/manylinux/blob/0807daf4e5b141de39061e87c4a4edb81f7ab33e/docker/build_scripts/install-runtime-packages.sh#L116))
the i686 (32-bit) containers pull the devtoolset from mayeut's i386 devtoolset
[fedora repo](https://copr.fedorainfracloud.org/coprs/mayeut/devtoolset-10).

So, we just need to pull that repo into our container and we'll be set.

We add the repo by:
- curling it
- patching its baseurl to explicitly point to i386 instead doing so through
`$basearch`.

which we do with these commands:

```
curl -fsSLo /etc/yum.repos.d/mayeut-devtoolset-10.repo \
    https://copr.fedorainfracloud.org/coprs/mayeut/devtoolset-10/repo/custom-1/mayeut-devtoolset-10-custom-1.repo
sed -i 's/\$basearch/i386/g' /etc/yum.repos.d/mayeut-devtoolset-10.repo
```

Finally, since we're depending on what's basically a manylinux implementation
detail by pointing to mayeut's repo directly, we add a mitigation against
that repo becoming unavailable by installing what I guess are the most useful
devtoolset packages, directly into our published base images.

We do that with this command:

```
yum install -y \
  devtoolset-10-libstdc++-devel.i686 \
  devtoolset-10-libatomic-devel.i686 \
  devtoolset-10-make-devel.i686 \
  devtoolset-10-libasan-devel.i686 \
  devtoolset-10-libubsan-devel.i686 \
  devtoolset-10-libquadmath-devel.i686 \
  devtoolset-10-valgrind-devel
```
Note: We don't don't install the i686 gcc package, since the x86\_64 gcc
package already provides the i686 compiler and the two packages conflict.

And there you have it. The published container is
[whenning42/manylinux2014\_x86\_64\_i386](https://hub.docker.com/r/whenning42/manylinux2014_x86_64_i386).
If you want a different base manylinux container, I imagine the process will be
similar, but the details will probably differ, since the fedora package we
depend on to get the 32-bit devtoolset only appears to be used in the
manylinux2014 build.

