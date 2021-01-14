# Use one of the following commands to build the manifest for Python3:
#
# - make                Building for Linux
# - make DEBUG=1        Building for Linux (with Graphene debug output)
# - make SGX=1          Building for SGX
# - make SGX=1 DEBUG=1  Building for SGX (with Graphene debug output)
#
# Use `make clean` to remove Graphene-generated files.

include ../../Scripts/Makefile.configs

# Python constants are declared in Makefile.python
include ../../Scripts/Makefile.python

SGX_SIGNER_KEY ?= ../../Pal/src/host/Linux-SGX/signer/enclave-key.pem

ifeq ($(DEBUG),1)
GRAPHENEDEBUG = inline
else
GRAPHENEDEBUG = none
endif

UBUNTU_VER = $(shell lsb_release --short --id)$(shell lsb_release --short --release)

.PHONY: all
all: python.manifest
ifeq ($(SGX),1)
all: python.manifest.sgx python.sig python.token
endif

# Define the python libraries which are dynamically loaded.
PY_LIBS = \
	$(PYTHON_STDLIB)/lib-dynload/_hashlib$(PYTHON_EXT_SUFFIX) \
	$(PYTHON_STDLIB)/lib-dynload/_ctypes$(PYTHON_EXT_SUFFIX) \
	$(PYTHON_STDLIB)/lib-dynload/_ssl$(PYTHON_EXT_SUFFIX) \
	$(PYTHON_STDLIB)/lib-dynload/_bz2$(PYTHON_EXT_SUFFIX) \
	$(PYTHON_STDLIB)/lib-dynload/_lzma$(PYTHON_EXT_SUFFIX) \
	$(PYTHON_STDLIB)/lib-dynload/_json$(PYTHON_EXT_SUFFIX)

ifeq ($(SGX),1)
PY_LIBS += $(PYTHON_DISTLIB)/apt_pkg$(PYTHON_EXT_SUFFIX)
endif

ifeq ($(PYTHONSHORTVERSION),35)
define PYTHON_TRUSTED_SCRIPTS
sgx.trusted_files.python21 = "file:$(PYTHON_STDLIB)/_sysconfigdata.py"
sgx.trusted_files.python22 = "file:$(PYTHON_STDLIB)/plat-$(PYTHON_MULTIARCH)/_sysconfigdata_m.py"

endef
else
define PYTHON_TRUSTED_SCRIPTS
sgx.trusted_files.python21 = "file:$(PYTHON_STDLIB)/_sysconfigdata_m_linux_$(PYTHON_MULTIARCH).py"

endef
endif

# Generate manifest rules for Python dependencies.
# We'll duplicate some Glibc libraries (which Graphene provides in a customized version), but
# there's no harm in this.
.INTERMEDIATE: trusted-libs
trusted-libs: ../common_tools/get_deps.sh
	../common_tools/get_deps.sh $(PY_LIBS) > $@


python.manifest: python.manifest.template trusted-libs
	(sed \
		-e 's|$$(RUNTIMEDIR)|'"$(RUNTIMEDIR)"'|g' \
		-e 's|$$(GRAPHENEDEBUG)|'"$(GRAPHENEDEBUG)"'|g' \
		-e 's|$$(PYTHON_STDLIB)|'"$(PYTHON_STDLIB)"'|g' \
		-e 's|$$(PYTHON_DISTLIB)|'"$(PYTHON_DISTLIB)"'|g' \
		-e 's|$$(PYTHON_MULTIARCH)|'"$(PYTHON_MULTIARCH)"'|g' \
		-e 's|$$(PYTHON_EXT_SUFFIX)|'"$(PYTHON_EXT_SUFFIX)"'|g' \
		-e 's|$$(PYTHONEXEC)|'"$(PYTHONEXEC)"'|g' \
		-e 's|$$(ARCH_LIBDIR)|'"$(ARCH_LIBDIR)"'|g' \
		-e 's|$$(ARCH_LONG)|'"$(ARCH_LONG)"'|g' \
		$<; \
	cat trusted-libs) > $@

# Python manifests for SGX:
#   Generating the SGX-specific manifest (python.manifest.sgx), the enclave signature,
#   and the token for enclave initialization.

python.manifest.sgx: python.manifest
	graphene-sgx-sign \
		-libpal $(RUNTIMEDIR)/libpal-Linux-SGX.so \
		-key $(SGX_SIGNER_KEY) \
		-manifest $< \
		-output $@

python.sig: python.manifest.sgx

python.token: python.sig
	graphene-sgx-get-token -output $@ -sig $<

.PHONY: check
check: all
	./run-tests.sh > OUTPUT_TEST 2> /dev/null
	@grep -q "Success 1/3" OUTPUT_TEST
	@grep -q "Success 2/3" OUTPUT_TEST
	@grep -q "Success 3/3" OUTPUT_TEST
	@rm OUTPUT_TEST

.PHONY: clean
clean:
	$(RM) *.manifest *.manifest.sgx *.token *.sig OUTPUT* *.PID
	$(RM) -r scripts/__pycache__

.PHONY: distclean
distclean: clean
