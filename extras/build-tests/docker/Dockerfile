# ZenIRCd test environment: builds ZenIRCd exactly like CI (ASAN+UBSAN, -Werror)
# and runs the full upstream test suite against Anope and Atheme.
#
# Don't use this file directly. Run:  extras/build-tests/docker/run
#
# Why a container: the test framework kills every process of the current user
# whose name matches ircd/services/anope/val..., and overwrites services configs
# and databases. It must never run on a developer machine or server.
#
# Base image choices:
# - Python 3.11: unrealircd-tests needs asyncore/asynchat (removed in 3.12).
# - Debian trixie (GCC 14): bookworm's GCC 12 AddressSanitizer runtime can hang in an
#   infinite "AddressSanitizer:DEADLYSIGNAL" loop on recent kernels with high mmap
#   ASLR entropy (seen on Linux 7.0: 5 of 20 runs).
FROM python:3.11-trixie

RUN apt-get update && apt-get install -y --no-install-recommends \
      build-essential cmake pkg-config libssl-dev wget xz-utils psmisc procps git ca-certificates \
    && rm -rf /var/lib/apt/lists/*

# The test scripts use '#!/usr/bin/python3', which on trixie is the system Python 3.13.
# Point it at the image's Python 3.11 (this container is throwaway).
RUN ln -sf /usr/local/bin/python3.11 /usr/bin/python3 \
    && /usr/bin/python3 -c "import asyncore, asynchat"

RUN useradd -m -s /bin/bash ircd
USER ircd
WORKDIR /home/ircd

# Services are built first, at the versions pinned by unrealircd-tests, so that
# source changes only rebuild the ZenIRCd layer. Keep TESTS_REF in sync with
# extras/build-tests/nix/run-tests.
ARG TESTS_REF=8043c407497b3ea465cef9ed2e759f14d5079ad6
RUN git clone -q --branch unreal60 https://github.com/unrealircd/unrealircd-tests.git /home/ircd/t \
    && git -C /home/ircd/t -c advice.detachedHead=false checkout -q "$TESTS_REF"
# Every long step is time-limited and file writes are capped at ~2 GB, so a hang
# fails the build instead of filling the disk.
RUN ulimit -f 2000000 && timeout 3600 /home/ircd/t/serverconfig/anope/install_anope > /home/ircd/anope-build.log 2>&1 \
    || (tail -60 /home/ircd/anope-build.log; exit 1)
RUN ulimit -f 2000000 && timeout 3600 /home/ircd/t/serverconfig/atheme/install_atheme > /home/ircd/atheme-build.log 2>&1 \
    || (tail -60 /home/ircd/atheme-build.log; exit 1)

COPY --chown=ircd:ircd . /home/ircd/zenircd
RUN ulimit -f 2000000 && cd zenircd && timeout 3600 extras/build-tests/nix/build > /home/ircd/build.log 2>&1 \
    || (tail -100 /home/ircd/build.log; exit 1)

WORKDIR /home/ircd/zenircd
ENV TESTS_REF=$TESTS_REF
CMD ["bash", "-c", "ulimit -f 2000000; extras/build-tests/nix/run-tests.bbwrapper"]
