# Copyright (c) 2015 Cisco and/or its affiliates.
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at:
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

# Scripts require non-POSIX parts of bash
SHELL := /bin/bash

DPDK_BUILD_DIR        ?= $(CURDIR)/_build
DPDK_INSTALL_DIR      ?= $(CURDIR)/_install
DPDK_PKTMBUF_HEADROOM ?= 256
DPDK_DOWNLOAD_DIR     ?= $(HOME)/Downloads
DPDK_MARCH            ?= native
DPDK_DEBUG            ?= n

B := $(DPDK_BUILD_DIR)
I := $(DPDK_INSTALL_DIR)
DPDK_VERSION ?= 2.2.0
DPDK_TARBALL := dpdk-$(DPDK_VERSION).tar.gz
DPDK_TAR_URL := http://dpdk.org/browse/dpdk/snapshot/$(DPDK_TARBALL)
DPDK_2.1.0_TARBALL_MD5_CKSUM := 205a0d12bfd6eb717d57506272f43519
DPDK_2.2.0_TARBALL_MD5_CKSUM := 22e2fd68cd5504f43fe9a5a6fd6dd938
DPDK_SOURCE := $(B)/dpdk-$(DPDK_VERSION)
DPDK_TARGET := x86_64-native-linuxapp-gcc
JOBS := $(shell grep processor /proc/cpuinfo | wc -l)

# compiler/linker custom arguments
DPDK_CPU_CFLAGS := -pie -fPIC
DPDK_CPU_LDFLAGS := -pie -fPIC
DPDK_EXTRA_LDFLAGS := -g

ifeq ($(DPDK_DEBUG),n)
DPDK_EXTRA_CFLAGS := -g 
else
DPDK_EXTRA_CFLAGS := -g -O0 
endif

# translate gcc march values to DPDK arch
ifeq ($(DPDK_MARCH),native)
DPDK_MACHINE:=native                     # autodetect host CPU
else ifeq ($(DPDK_MARCH),corei7)
DPDK_MACHINE:=nhm                        # Nehalem / Westmere
else ifeq ($(DPDK_MARCH),corei7-avx)
DPDK_MACHINE:=snb                        # Sandy Bridge
else ifeq ($(DPDK_MARCH),core-avx-i)
DPDK_MACHINE:=ivb                        # Ivy Bridge
else ifeq ($(DPDK_MARCH),core-avx2)
DPDK_MACHINE:=hsw                        # Haswell
else
$(error Unknown DPDK_MARCH)
endif

# assemble DPDK make arguments
DPDK_MAKE_ARGS := -C $(DPDK_SOURCE) -j $(JOBS) \
	T=$(DPDK_TARGET) \
	RTE_CONFIG_TEMPLATE=../custom-config \
	RTE_OUTPUT=$(I) \
	EXTRA_CFLAGS="$(DPDK_EXTRA_CFLAGS)" \
	EXTRA_LDFLAGS="$(DPDK_EXTRA_LDFLAGS)" \
	CPU_CFLAGS="$(DPDK_CPU_CFLAGS)" \
	CPU_LDFLAGS="$(DPDK_CPU_LDFLAGS)"

DPDK_SOURCE_FILES := $(shell  [ -e $(DPDK_SOURCE) ] && find $(DPDK_SOURCE) -name "*.[chS]")  

define set
@if grep -q CONFIG_$1 $@ ; \
	then sed -i -e 's/.*\(CONFIG_$1=\).*/\1$2/' $@ ; \
	else echo CONFIG_$1=$2 >> $@ ; \
fi
endef

all: build

$(B)/custom-config: $(B)/.patch.ok Makefile
	@echo --- generating custom config from $(DPDK_SOURCE)/config/common_linuxapp ---
	@cp $(DPDK_SOURCE)/config/common_linuxapp $@ 
	$(call set,RTE_MACHINE,$(DPDK_MACHINE))
	$(call set,RTE_ARCH,"x86_64")
	$(call set,RTE_ARCH_X86_64,y)
	$(call set,RTE_ARCH_64,y)
	$(call set,RTE_TOOLCHAIN_GCC,y)
	$(call set,RTE_TOOLCHAIN,"gcc")
	@# modify options
	$(call set,RTE_PKTMBUF_HEADROOM,$(DPDK_PKTMBUF_HEADROOM))
	$(call set,RTE_LIBEAL_USE_HPET,y)
	$(call set,RTE_BUILD_COMBINE_LIBS,y)
	$(call set,RTE_LIBRTE_I40E_16BYTE_RX_DESC,y)
	$(call set,RTE_LIBRTE_I40E_ITR_INTERVAL,16)
	@# enable debug init for device drivers
	$(call set,RTE_LIBRTE_I40E_DEBUG_INIT,$(DPDK_DEBUG))
	$(call set,RTE_LIBRTE_IXGBE_DEBUG_INIT,$(DPDK_DEBUG))
	$(call set,RTE_LIBRTE_E1000_DEBUG_INIT,$(DPDK_DEBUG))
	$(call set,RTE_LIBRTE_VIRTIO_DEBUG_INIT,$(DPDK_DEBUG))
	$(call set,RTE_LIBRTE_VMXNET3_DEBUG_INIT,$(DPDK_DEBUG))
	@# not needed
	$(call set,RTE_LIBRTE_PMD_BOND,n)
	$(call set,RTE_LIBRTE_TIMER,n)
	$(call set,RTE_LIBRTE_CFGFILE,n)
	$(call set,RTE_LIBRTE_LPM,n)
	$(call set,RTE_LIBRTE_ACL,n)
	$(call set,RTE_LIBRTE_POWER,n)
	$(call set,RTE_LIBRTE_IP_FRAG,n)
	$(call set,RTE_LIBRTE_DISTRIBUTOR,n)
	$(call set,RTE_LIBRTE_REORDER,n)
	$(call set,RTE_LIBRTE_PORT,n)
	$(call set,RTE_LIBRTE_TABLE,n)
	$(call set,RTE_LIBRTE_PIPELINE,n)
	$(call set,RTE_KNI_KMOD,n)
	@rm -f .config.ok

$(CURDIR)/$(DPDK_TARBALL):
	@mkdir -p $(B)
	@if [ -e $(DPDK_DOWNLOAD_DIR)/$(DPDK_TARBALL) ] ; \
		then cp $(DPDK_DOWNLOAD_DIR)/$(DPDK_TARBALL) $(CURDIR) ; \
		else curl -o $(CURDIR)/$(DPDK_TARBALL) -LO $(DPDK_TAR_URL) ; \
	fi
	@rm -f $(B)/.download.ok

$(B)/.download.ok: $(CURDIR)/$(DPDK_TARBALL)
	@openssl md5 $< | cut -f 2 -d " " - > $(B)/$(DPDK_TARBALL).md5sum
	@([ "$$(<$(B)/$(DPDK_TARBALL).md5sum)" = "$(DPDK_$(DPDK_VERSION)_TARBALL_MD5_CKSUM)" ] || \
	( echo "Bad Checksum! Please remove $< and retry" && \
		rm $(B)/$(DPDK_TARBALL).md5sum && false ))
	@touch $@

.PHONY: download
download: $(B)/.download.ok

$(B)/.extract.ok: $(B)/.download.ok
	@echo --- extracting $(DPDK_TARBALL) ---
	@tar --directory $(B) --extract --file $(CURDIR)/$(DPDK_TARBALL) --gzip
	@touch $@

.PHONY: extract
extract: $(B)/.extract.ok

$(B)/.patch.ok: $(B)/.extract.ok
	@echo --- patching ---
	for f in $(CURDIR)/dpdk-$(DPDK_VERSION)_patches/*.patch ; do \
		echo Applying patch: $$(basename $$f) ; \
		patch -p1 -d $(DPDK_SOURCE) < $$f ; \
	done
	@touch $@

.PHONY: patch
patch: $(B)/.patch.ok

$(B)/.config.ok: $(B)/.patch.ok $(B)/custom-config
	@make $(DPDK_MAKE_ARGS) config
	@touch $@

.PHONY: config
config: $(B)/.config.ok

$(B)/.build.ok: $(DPDK_SOURCE_FILES)
	@if [ ! -e $(B)/.config.ok ] ; then echo 'Please run "make config" first' && false ; fi
	@make $(DPDK_MAKE_ARGS) install
	@dkms/create_deb_manifest.sh $(DPDK_VERSION) $(subst $(realpath ..)/,,$(B))
	@touch $@

.PHONY: build
build: $(B)/.build.ok

.PHONY: clean
clean:
	@rm -rf $(B) $(I)

