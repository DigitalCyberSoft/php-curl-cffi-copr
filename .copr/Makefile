# COPR Makefile for automated php-curl-cffi builds
#
# This extension requires libcurl-impersonate (static library), which is not
# available as a system package. The Makefile bundles the curl-impersonate
# source tarball into the SRPM, and the spec's %build compiles it natively
# on each target architecture.
#
# To auto-rebuild when PHP is updated, enable "Auto-rebuild" in the COPR
# project settings. This monitors BuildRequires (php-devel) for updates.

# COPR provides outdir, set default for local testing
outdir ?= .

# Fetch latest release versions from GitHub
EXT_VERSION := $(shell curl -s https://api.github.com/repos/DigitalCyberSoft/php-curl-cffi/releases/latest | sed -n 's/.*"tag_name": *"v\?\([^"]*\)".*/\1/p')
CURL_IMP_VERSION := $(shell curl -s https://api.github.com/repos/lwthiker/curl-impersonate/releases/latest | sed -n 's/.*"tag_name": *"v\?\([^"]*\)".*/\1/p')

EXT_URL = https://github.com/DigitalCyberSoft/php-curl-cffi/archive/refs/tags/v$(EXT_VERSION).tar.gz
CURL_IMP_URL = https://github.com/lwthiker/curl-impersonate/archive/refs/tags/v$(CURL_IMP_VERSION).tar.gz
RELEASE := $(shell cat $(CURDIR)/release 2>/dev/null || echo 1)

# Default target for COPR
srpm:
	@echo "=== Starting php-curl-cffi SRPM build ==="
	@echo "Extension version: $(EXT_VERSION)"
	@echo "curl-impersonate version: $(CURL_IMP_VERSION)"
	@echo "Output directory: $(outdir)"

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

	# Install SRPM build dependencies
	dnf -y install git rpm-build curl

	# Download PHP extension source
	@echo "=== Downloading php-curl-cffi $(EXT_VERSION) ==="
	curl -L -o php-curl-cffi-$(EXT_VERSION).tar.gz $(EXT_URL) || exit 1

	# Clone curl-impersonate source (not compiled -- built during %build)
	@echo "=== Cloning curl-impersonate $(CURL_IMP_VERSION) ==="
	rm -rf curl-impersonate
	git clone --depth=1 --branch v$(CURL_IMP_VERSION) \
		https://github.com/lwthiker/curl-impersonate.git \
		curl-impersonate || exit 1

	# Pre-fetch curl-impersonate build dependencies into its source tree
	# (RPM %build has no network access, so these must be bundled)
	@echo "=== Downloading curl-impersonate build dependencies ==="
	@cd curl-impersonate && \
	BROTLI_VER=$$(sed -n 's/^BROTLI_VERSION := //p' Makefile.in); \
	BSSL_COMMIT=$$(sed -n 's/^BORING_SSL_COMMIT := //p' Makefile.in); \
	NGHTTP2_URL=$$(sed -n 's/^NGHTTP2_URL := //p' Makefile.in); \
	NGHTTP2_VER=$$(sed -n 's/^NGHTTP2_VERSION := //p' Makefile.in); \
	CURL_VER=$$(sed -n 's/^CURL_VERSION := //p' Makefile.in); \
	echo "  brotli: $$BROTLI_VER"; \
	echo "  BoringSSL: $$BSSL_COMMIT"; \
	echo "  nghttp2: $$NGHTTP2_VER"; \
	echo "  curl: $$CURL_VER"; \
	curl -fL -o "brotli-$$BROTLI_VER.tar.gz" \
		"https://github.com/google/brotli/archive/refs/tags/v$$BROTLI_VER.tar.gz" || exit 1; \
	curl -fL -o boringssl.zip \
		"https://github.com/google/boringssl/archive/$$BSSL_COMMIT.zip" || exit 1; \
	curl -fL -o "$$NGHTTP2_VER.tar.bz2" \
		"$$NGHTTP2_URL" || exit 1; \
	curl -fL -o "$$CURL_VER.tar.xz" \
		"https://curl.se/download/$$CURL_VER.tar.xz" || exit 1; \
	echo "  All dependencies downloaded"

	# Create curl-impersonate source tarball (with pre-fetched deps, exclude .git)
	@echo "=== Packaging curl-impersonate source ==="
	tar czf curl-impersonate-$(CURL_IMP_VERSION).tar.gz \
		--exclude='.git' \
		curl-impersonate/
	@echo "  curl-impersonate tarball: $$(du -h curl-impersonate-$(CURL_IMP_VERSION).tar.gz | cut -f1)"

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
	echo 'Release:        $(RELEASE)%{?dist}'; \
	echo 'Summary:        Browser TLS/HTTP2 fingerprint impersonation for PHP'; \
	echo ''; \
	echo 'License:        MIT'; \
	echo 'URL:            https://github.com/DigitalCyberSoft/php-curl-cffi'; \
	echo 'Source0:        php-curl-cffi-%{version}.tar.gz'; \
	echo 'Source1:        curl-impersonate-$(CURL_IMP_VERSION).tar.gz'; \
	echo ''; \
	echo 'BuildRequires:  gcc'; \
	echo 'BuildRequires:  gcc-c++'; \
	echo 'BuildRequires:  make'; \
	echo 'BuildRequires:  cmake >= 3.12'; \
	echo 'BuildRequires:  ninja-build'; \
	echo 'BuildRequires:  golang'; \
	echo 'BuildRequires:  autoconf'; \
	echo 'BuildRequires:  automake'; \
	echo 'BuildRequires:  libtool'; \
	echo 'BuildRequires:  python3'; \
	echo 'BuildRequires:  unzip'; \
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
	echo '%setup -q -n php-curl-cffi-%{version} -a 1'; \
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
	echo '# Build curl-impersonate from bundled source'; \
	echo 'CURL_IMP_DIR=$$(pwd)/curl-impersonate'; \
	echo 'cd "$$CURL_IMP_DIR"'; \
	echo 'mkdir -p build && cd build'; \
	echo '# Copy pre-fetched dependency archives so make does not try to download'; \
	echo 'cp ../*.tar.gz ../*.tar.bz2 ../*.tar.xz ../*.zip . 2>/dev/null || :'; \
	echo '../configure --prefix=$$(pwd)/dist'; \
	echo 'make %{?_smp_mflags} chrome-build'; \
	echo 'make chrome-install'; \
	echo 'cd ../../'; \
	echo ''; \
	echo '# Find the built static library'; \
	echo 'STATIC_LIB=$$(find "$$CURL_IMP_DIR" -name "libcurl-impersonate*.a" -type f | head -1)'; \
	echo 'if [ -z "$$STATIC_LIB" ]; then'; \
	echo '    echo "ERROR: libcurl-impersonate static library not found after build"'; \
	echo '    find "$$CURL_IMP_DIR" -name "*.a" -type f'; \
	echo '    exit 1'; \
	echo 'fi'; \
	echo 'echo "Built static library: $$STATIC_LIB"'; \
	echo 'cp "$$STATIC_LIB" libcurl-impersonate.a'; \
	echo ''; \
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
	echo 'cp "$$STATIC_LIB" libcurl-impersonate.a'; \
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
	@rm -f *.tar.gz *.spec *.src.rpm
	@rm -rf curl-impersonate/ php-curl-cffi-*/

.PHONY: srpm clean
