FROM linuxkit/alpine:96ad1eb5ec262b4cd0eef574cdc0b225b502d9ee AS kernel-build
RUN apk add \
    argp-standalone \
    automake \
    bash \
    bc \
    binutils-dev \
    bison \
    build-base \
    curl \
    diffutils \
    flex \
    git \
    gmp-dev \
    gnupg \
    installkernel \
    kmod \
    libelf-dev \
    libressl-dev \
    linux-headers \
    mpc1-dev \
    mpfr-dev \
    ncurses-dev \
    patch \
    sed \
    squashfs-tools \
    tar \
    xz \
    xz-dev \
    zlib-dev

# libunwind-dev pkg is missed from arm64 now, below statement will be removed if the pkg is available.
RUN [ $(uname -m) == x86_64 ] && apk add libunwind-dev || true

ARG KERNEL_VERSION
ARG KERNEL_SERIES
ARG EXTRA
ARG DEBUG

ENV KERNEL_SOURCE=https://www.kernel.org/pub/linux/kernel/v4.x/linux-${KERNEL_VERSION}.tar.xz
ENV KERNEL_SHA256_SUMS=https://www.kernel.org/pub/linux/kernel/v4.x/sha256sums.asc
ENV KERNEL_PGP2_SIGN=https://www.kernel.org/pub/linux/kernel/v4.x/linux-${KERNEL_VERSION}.tar.sign

ENV WIREGUARD_VERSION=0.0.20180304
ENV WIREGUARD_SHA256=efb1652f0da67fb2731040439b6abb820a5e2f1bc177aa15c5dce68ea3327787
ENV WIREGUARD_URL=https://git.zx2c4.com/WireGuard/snapshot/WireGuard-${WIREGUARD_VERSION}.tar.xz

# We copy the entire directory. This copies some unneeded files, but
# allows us to check for the existence /patches-${KERNEL_SERIES} to
# build kernels without patches.
COPY / /

# Download and verify kernel
# PGP keys: 589DA6B1 (greg@kroah.com) & 6092693E (autosigner@kernel.org) & 00411886 (torvalds@linux-foundation.org)
RUN \
    [ -f linux-${KERNEL_VERSION}.tar.xz ] || curl -fsSLO ${KERNEL_SOURCE} && \
    xz -d linux-${KERNEL_VERSION}.tar.xz && \
    cat linux-${KERNEL_VERSION}.tar | tar --absolute-names -x && mv /linux-${KERNEL_VERSION} /linux

WORKDIR /linux
# Apply local specific patches if present
RUN set -e && \
    if [ -n "${EXTRA}" ] && [ -d /patches-${KERNEL_SERIES}${EXTRA} ]; then \
	echo "Patching ${EXTRA} kernel"; \
	for patch in /patches-${KERNEL_SERIES}${EXTRA}/*.patch; do \
            echo "Applying $patch"; \
            patch -t -F0 -N -u -p1 < "$patch"; \
	done; \
    fi

# Apply local common patches if present
RUN set -e && \
    if [ -d /patches-${KERNEL_SERIES} ]; then \
        for patch in /patches-${KERNEL_SERIES}/*.patch; do \
            echo "Applying $patch"; \
            patch -t -F0 -N -u -p1 < "$patch"; \
        done; \
    fi

# Kernel config
RUN case $(uname -m) in \
    x86_64) \
        KERNEL_DEF_CONF=/linux/arch/x86/configs/x86_64_defconfig; \
        ;; \
    aarch64) \
        KERNEL_DEF_CONF=/linux/arch/arm64/configs/defconfig; \
        ;; \
    s390x) \
        KERNEL_DEF_CONF=/linux/arch/s390/defconfig; \
        ;; \
    esac  && \
    cp /config-${KERNEL_SERIES}-$(uname -m) ${KERNEL_DEF_CONF}; \
    if [ -n "${EXTRA}" ] && [ -f "/config-${KERNEL_SERIES}-$(uname -m)${EXTRA}" ]; then \
        cat /config-${KERNEL_SERIES}-$(uname -m)${EXTRA} >> ${KERNEL_DEF_CONF}; \
    fi; \
    sed -i "s/CONFIG_LOCALVERSION=\"-linuxkit\"/CONFIG_LOCALVERSION=\"-linuxkit${EXTRA}${DEBUG}\"/" ${KERNEL_DEF_CONF}; \
    if [ -n "${DEBUG}" ]; then \
        sed -i 's/CONFIG_PANIC_ON_OOPS=y/# CONFIG_PANIC_ON_OOPS is not set/' ${KERNEL_DEF_CONF}; \
        cat /config${DEBUG} >> ${KERNEL_DEF_CONF}; \
    fi && \
    make defconfig && \
    make oldconfig && \
    if [ -z "${EXTRA}" ] && [ -z "${DEBUG}" ]; then diff .config ${KERNEL_DEF_CONF}; fi

RUN mkdir /out

# Kernel
RUN make -j "$(getconf _NPROCESSORS_ONLN)" KCFLAGS="-fno-pie" && \
    case $(uname -m) in \
    x86_64) \
        cp arch/x86_64/boot/bzImage /out/kernel; \
        ;; \
    aarch64) \
        cp arch/arm64/boot/Image.gz /out/kernel; \
        ;; \
    s390x) \
        cp arch/s390/boot/bzImage /out/kernel; \
        ;; \
    esac && \
    cp System.map /out && \
    ([ -n "${DEBUG}" ] && cp vmlinux /out || true)

RUN printf "KERNEL_SOURCE=${KERNEL_SOURCE}\n" > /out/kernel-source-info

# perf (Don't compile for 4.4.x, it's broken and tedious to fix)
RUN if [ "${KERNEL_SERIES}" != "4.4.x" ]; then \
       mkdir -p /build/perf && \
       make -C tools/perf LDFLAGS=-static O=/build/perf && \
       strip /build/perf/perf && \
       cp /build/perf/perf /out; \
     fi

FROM scratch
ENTRYPOINT []
CMD []
WORKDIR /
COPY --from=kernel-build /out/* /
