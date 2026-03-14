# COPR Makefile for automated php-curl-cffi builds
#
# This extension requires libcurl-impersonate.a (static library), which is not
# available as a system package. The Makefile builds it from source during the
# SRPM preparation phase (which has internet access), then bundles the .a into
# the SRPM so the RPM build phase can link against it offline.
#
# To auto-rebuild when PHP is updated, enable "Auto-rebuild" in the COPR
# project settings. This monitors BuildRequires (php-devel) for updates.

# COPR provides outdir, set default for local testing
outdir ?= .

# Fetch latest release versions from GitHub
EXT_VERSION := $(shell curl -s https://api.github.com/repos/DigitalCyberSoft/php-curl-cffi/releases/latest | sed -n 's/.*"tag_name": *"v\?\([^"]*\)".*/\1/p')
CURL_IMP_VERSION := $(shell curl -s https://api.github.com/repos/lwthiker/curl-impersonate/releases/latest | sed -n 's/.*"tag_name": *"v\?\([^"]*\)".*/\1/p')

EXT_URL = https://github.com/DigitalCyberSoft/php-curl-cffi/archive/refs/tags/v$(EXT_VERSION).tar.gz

# Default target for COPR
srpm:
	@echo "=== Starting php-curl-cffi SRPM build ==="
	@echo "Extension version: $(EXT_VERSION)"
	@echo "curl-impersonate version: $(CURL_IMP_VERSION)"
	@echo "Output directory: $(outdir)"
	@echo "Architecture: $$(uname -m)"

	# Validate versions were detected
	@if [ -z "$(EXT_VERSION)" ]; then \
		echo "ERROR: Could not determine php-curl-cffi version from GitHub"; \
		echo "Ensure the DigitalCyberSoft/php-curl-cffi repo has a release"; \
		exit 1; \
	fi
	@if [ -z "$(CURL_IMP_VERSION)" ]; then \
		echo "ERROR: Could not determine curl-impersonate version from GitHub"; \
		exit 1; \
	fi

	# Install build dependencies for curl-impersonate
	@echo "=== Installing curl-impersonate build dependencies ==="
	@NEED_INSTALL=0; \
	for cmd in git go cmake ninja gcc; do \
		if ! command -v $$cmd >/dev/null 2>&1; then NEED_INSTALL=1; break; fi; \
	done; \
	if [ $$NEED_INSTALL -eq 1 ]; then \
		echo "Installing missing build tools..."; \
		dnf install -y git golang cmake ninja-build gcc gcc-c++ \
			autoconf automake libtool python3 make \
			zlib-devel || exit 1; \
	else \
		echo "Build tools already available"; \
	fi

	# Build curl-impersonate from source (Chrome variant with BoringSSL)
	@echo "=== Building curl-impersonate $(CURL_IMP_VERSION) ==="
	@rm -rf _curl_imp_build; \
	git clone --depth=1 --branch v$(CURL_IMP_VERSION) \
		https://github.com/lwthiker/curl-impersonate.git \
		_curl_imp_build || exit 1; \
	\
	cd _curl_imp_build && \
	mkdir -p build && cd build && \
	../configure --prefix=$$(pwd)/dist && \
	$(MAKE) chrome-build && \
	$(MAKE) chrome-install; \
	BUILD_RC=$$?; \
	cd ../..; \
	\
	if [ $$BUILD_RC -ne 0 ]; then \
		echo "ERROR: curl-impersonate build failed"; \
		exit 1; \
	fi; \
	\
	STATIC_LIB=$$(find _curl_imp_build -name "libcurl-impersonate*.a" -type f | head -1); \
	if [ -z "$$STATIC_LIB" ]; then \
		echo "ERROR: Static library not found after build. Available .a files:"; \
		find _curl_imp_build -name "*.a" -type f; \
		exit 1; \
	fi; \
	echo "Built static library: $$STATIC_LIB"; \
	cp "$$STATIC_LIB" libcurl-impersonate.a || exit 1; \
	ls -lh libcurl-impersonate.a

	# Download PHP extension source
	@echo "=== Downloading php-curl-cffi $(EXT_VERSION) ==="
	curl -L -o php-curl-cffi-$(EXT_VERSION).tar.gz $(EXT_URL) || exit 1

	# Generate spec file
	@echo "=== Generating spec file ==="
	@( \
	echo '# we dont want -z defs linker flag'; \
	echo '%undefine _strict_symbol_defs_build'; \
	echo ''; \
	echo '%global pecl_name   curl_impersonate'; \
	echo '%global with_zts    0%{!?_without_zts:%{?__ztsphp:1}}'; \
	echo '%global ini_name    40-%{pecl_name}.ini'; \
	echo ''; \
	echo 'Name:           php-%{pecl_name}'; \
	echo 'Version:        $(EXT_VERSION)'; \
	echo 'Release:        1%{?dist}'; \
	echo 'Summary:        Browser TLS/HTTP2 fingerprint impersonation for PHP'; \
	echo ''; \
	echo 'License:        MIT'; \
	echo 'URL:            https://github.com/DigitalCyberSoft/php-curl-cffi'; \
	echo 'Source0:        php-curl-cffi-%{version}.tar.gz'; \
	echo 'Source1:        libcurl-impersonate.a'; \
	echo ''; \
	echo 'BuildRequires:  gcc'; \
	echo 'BuildRequires:  gcc-c++'; \
	echo 'BuildRequires:  make'; \
	echo 'BuildRequires:  php-devel >= 8.0'; \
	echo 'BuildRequires:  libcurl-devel'; \
	echo 'BuildRequires:  zlib-devel'; \
	echo ''; \
	echo 'Requires:       php(zend-abi) = %{php_zend_api}'; \
	echo 'Requires:       php(api) = %{php_core_api}'; \
	echo 'Requires:       libstdc++'; \
	echo 'Requires:       zlib'; \
	echo ''; \
	echo 'Provides:       bundled(libcurl-impersonate) = $(CURL_IMP_VERSION)'; \
	echo ''; \
	echo '%description'; \
	echo 'PHP extension wrapping libcurl-impersonate to enable browser TLS and HTTP/2'; \
	echo 'fingerprint impersonation from PHP. Supports Chrome, Firefox, and Safari'; \
	echo 'fingerprints to bypass anti-bot systems that inspect TLS Client Hello and'; \
	echo 'HTTP/2 settings.'; \
	echo ''; \
	echo 'Provides both a high-level Session API (similar to Python requests) and a'; \
	echo 'low-level procedural API (compatible with PHP curl_* functions).'; \
	echo ''; \
	echo 'The libcurl-impersonate static library ($(CURL_IMP_VERSION)) is bundled.'; \
	echo ''; \
	echo '%package devel'; \
	echo 'Summary:        Development files for %{name}'; \
	echo 'Requires:       %{name}%{?_isa} = %{version}-%{release}'; \
	echo 'Requires:       php-devel%{?_isa}'; \
	echo ''; \
	echo '%description devel'; \
	echo 'These are the files needed to compile programs using %{name}.'; \
	echo ''; \
	echo '%prep'; \
	echo '%setup -q -n php-curl-cffi-%{version}'; \
	echo ''; \
	echo '# Copy the pre-built static library into the source tree'; \
	echo 'cp %{SOURCE1} .'; \
	echo ''; \
	echo '# Sanity check'; \
	echo 'extver=$$(sed -n '"'"'/#define PHP_CURL_IMPERSONATE_VERSION/{s/.* "//;s/".*$$//;p}'"'"' php_curl_impersonate.h)'; \
	echo 'if test "x$${extver}" != "x%{version}"; then'; \
	echo '   : Error: Upstream extension version is $${extver}, expecting %{version}.'; \
	echo '   exit 1'; \
	echo 'fi'; \
	echo ''; \
	echo '%if %{with_zts}'; \
	echo '# Duplicate for ZTS build'; \
	echo 'cp -pr . ../ZTS'; \
	echo '%endif'; \
	echo ''; \
	echo '%build'; \
	echo '# NTS build'; \
	echo '%{_bindir}/phpize'; \
	echo '%configure \'; \
	echo '    --with-php-config=%{_bindir}/php-config \'; \
	echo '    --with-libdir=%{_lib} \'; \
	echo '    --with-curl-impersonate=$$PWD/libcurl-impersonate.a'; \
	echo ''; \
	echo 'make %{?_smp_mflags}'; \
	echo ''; \
	echo '%if %{with_zts}'; \
	echo '# ZTS build'; \
	echo 'cd ../ZTS'; \
	echo '%{_bindir}/zts-phpize'; \
	echo '%configure \'; \
	echo '    --with-php-config=%{_bindir}/zts-php-config \'; \
	echo '    --with-libdir=%{_lib} \'; \
	echo '    --with-curl-impersonate=$$PWD/libcurl-impersonate.a'; \
	echo ''; \
	echo 'make %{?_smp_mflags}'; \
	echo '%endif'; \
	echo ''; \
	echo '%install'; \
	echo '# Install NTS extension'; \
	echo 'make install INSTALL_ROOT=%{buildroot}'; \
	echo ''; \
	echo '# Install config file'; \
	echo 'install -d %{buildroot}%{php_inidir}'; \
	echo 'cat > %{buildroot}%{php_inidir}/%{ini_name} << EOF'; \
	echo '; Enable curl_impersonate extension module'; \
	echo 'extension = %{pecl_name}.so'; \
	echo 'EOF'; \
	echo ''; \
	echo '%if %{with_zts}'; \
	echo '# Install ZTS extension'; \
	echo 'cd ../ZTS'; \
	echo 'make install INSTALL_ROOT=%{buildroot}'; \
	echo ''; \
	echo '# Install ZTS config file'; \
	echo 'install -d %{buildroot}%{php_ztsinidir}'; \
	echo 'cp %{buildroot}%{php_inidir}/%{ini_name} \'; \
	echo '   %{buildroot}%{php_ztsinidir}/%{ini_name}'; \
	echo '%else'; \
	echo '# Remove any ZTS files that NTS make install may have created'; \
	echo 'find %{buildroot} -name "%{pecl_name}.so" -not -path "%{buildroot}%{php_extdir}/*" -delete 2>/dev/null || :'; \
	echo '%endif'; \
	echo ''; \
	echo '# Install headers for devel package'; \
	echo 'install -d %{buildroot}%{php_incldir}/ext/%{pecl_name}'; \
	echo 'cp -p php_curl_impersonate.h %{buildroot}%{php_incldir}/ext/%{pecl_name}/'; \
	echo ''; \
	echo '%check'; \
	echo '# Minimal load test for NTS extension'; \
	echo '%{__php} --no-php-ini \'; \
	echo '    --define extension=%{buildroot}%{php_extdir}/%{pecl_name}.so \'; \
	echo '    --modules | grep "^%{pecl_name}$$"'; \
	echo ''; \
	echo '%if %{with_zts}'; \
	echo '# Minimal load test for ZTS extension'; \
	echo '%{__ztsphp} --no-php-ini \'; \
	echo '    --define extension=%{buildroot}%{php_ztsextdir}/%{pecl_name}.so \'; \
	echo '    --modules | grep "^%{pecl_name}$$"'; \
	echo '%endif'; \
	echo ''; \
	echo '%files'; \
	echo '%doc README.md'; \
	echo '%config(noreplace) %{php_inidir}/%{ini_name}'; \
	echo '%{php_extdir}/%{pecl_name}.so'; \
	echo ''; \
	echo '%if %{with_zts}'; \
	echo '%config(noreplace) %{php_ztsinidir}/%{ini_name}'; \
	echo '%{php_ztsextdir}/%{pecl_name}.so'; \
	echo '%endif'; \
	echo ''; \
	echo '%files devel'; \
	echo '%{php_incldir}/ext/%{pecl_name}'; \
	echo ''; \
	echo '%changelog'; \
	echo "* $$(date '+%a %b %d %Y') COPR Builder - $(EXT_VERSION)-1"; \
	echo '- Automated build from GitHub release $(EXT_VERSION)'; \
	echo '- Bundled libcurl-impersonate $(CURL_IMP_VERSION)'; \
	echo '- Browser TLS/HTTP2 fingerprint impersonation for PHP'; \
	echo '- Supports both NTS and ZTS builds'; \
	) > php-curl-cffi.spec || exit 1

	@echo "Spec file generated"

	# Build SRPM
	@echo "=== Building SRPM ==="
	rpmbuild -bs \
		--define "_sourcedir $$(pwd)" \
		--define "_specdir $$(pwd)" \
		--define "_builddir $$(pwd)" \
		--define "_srcrpmdir $(outdir)" \
		--define "_rpmdir $$(pwd)" \
		php-curl-cffi.spec || exit 1

	@echo "=== SRPM build complete ==="
	ls -la $(outdir)/*.src.rpm

clean:
	@rm -f *.tar.gz *.spec *.src.rpm *.a
	@rm -rf _curl_imp_build php-curl-cffi-*/

.PHONY: srpm clean
