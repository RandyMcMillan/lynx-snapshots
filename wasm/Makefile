SHELL := /bin/bash
.SHELLFLAGS := -eu -o pipefail -c

ROOT := $(abspath $(dir $(lastword $(MAKEFILE_LIST)))/..)
BUILD_DIR ?= $(ROOT)/build-wasm
BUILD_DIR := $(abspath $(BUILD_DIR))
PREFIX := $(BUILD_DIR)/ncurses
SITE := $(BUILD_DIR)/site
JOBS ?= 2
NCURSES_VERSION := 6.6-20260608
NCURSES_URL := https://archive.ubuntu.com/ubuntu/pool/main/n/ncurses/ncurses_6.6+20260608.orig.tar.gz
NCURSES_SHA256 := 3b91eb714ba61b9ebfcfe09cf8e7c1c45cf2e8a6282f5887fb942db514ae93bd

.PHONY: all check
all: $(PREFIX)/lib/libncurses.a
	mkdir -p "$(BUILD_DIR)/lynx" "$(SITE)"
	cd "$(BUILD_DIR)/lynx" && PATH="$(PREFIX)/bin:$$PATH" emconfigure "$(ROOT)/configure" \
		--host=wasm32-unknown-emscripten --build="$$( "$(ROOT)/config.guess" )" \
		--with-build-cc=gcc --with-screen=ncurses --with-curses-dir="$(PREFIX)" \
		--disable-nls --disable-nsl-fork --disable-ipv6 \
		--without-ssl --without-gnutls --without-zlib --without-bzlib \
		--without-brotli --without-zstd --disable-config-info \
		--disable-exec-scripts --disable-exec-links CFLAGS=-O2
	emmake $(MAKE) -C "$(BUILD_DIR)/lynx" -j$(JOBS)
	cd "$(BUILD_DIR)/lynx/src" && emcc -O2 *.o \
		../WWW/Library/Implementation/libwww.a "$(PREFIX)/lib/libncurses.a" \
		-sMODULARIZE=1 -sEXPORT_NAME=createLynx -sENVIRONMENT=web,node \
		-sEXPORTED_FUNCTIONS=_main -sEXPORTED_RUNTIME_METHODS=FS,callMain \
		-sEXIT_RUNTIME=1 -sALLOW_MEMORY_GROWTH=1 \
		--embed-file "$(ROOT)/lynx.cfg@/etc/lynx.cfg" \
		--embed-file "$(ROOT)/test@/test" \
		--embed-file "$(ROOT)/lynx_help@/lynx_help" \
		-o "$(SITE)/lynx.js"
	cp "$(ROOT)/wasm/index.html" "$(SITE)/index.html"
	cp "$(ROOT)/COPYING" "$(ROOT)/COPYHEADER" "$(SITE)/"
	touch "$(SITE)/.nojekyll"

$(PREFIX)/lib/libncurses.a:
	mkdir -p "$(BUILD_DIR)/downloads" "$(BUILD_DIR)/ncurses-build"
	curl --fail --location --retry 3 "$(NCURSES_URL)" -o "$(BUILD_DIR)/downloads/ncurses.tar.gz"
	printf '%s  %s\n' "$(NCURSES_SHA256)" "$(BUILD_DIR)/downloads/ncurses.tar.gz" | sha256sum --check
	tar -xzf "$(BUILD_DIR)/downloads/ncurses.tar.gz" -C "$(BUILD_DIR)"
	cd "$(BUILD_DIR)/ncurses-build" && emconfigure "../ncurses-$(NCURSES_VERSION)/configure" \
		--host=wasm32-unknown-emscripten --build="$$( "$(ROOT)/config.guess" )" \
		--prefix="$(PREFIX)" --without-shared --without-debug \
		--without-cxx --without-cxx-binding --without-ada \
		--without-progs --without-tests --without-manpages \
		--disable-db-install --disable-widec
	emmake $(MAKE) -C "$(BUILD_DIR)/ncurses-build" -j$(JOBS)
	emmake $(MAKE) -C "$(BUILD_DIR)/ncurses-build" install

check:
	@printf '\n========== WebAssembly smoke test ==========\n'
	test -s "$(SITE)/lynx.wasm"
	node -e 'const fs = require("fs"); const site = process.argv[1]; let output = ""; require(site + "/lynx.js")({wasmBinary: fs.readFileSync(site + "/lynx.wasm"), noInitialRun: true, quit: (status, error) => { throw error; }, print: line => { console.log(line); output += line + "\n"; }}).then(lynx => { const status = lynx.callMain(["-cfg=/etc/lynx.cfg", "-lss=/dev/null", "-dump", "-nolist", "file:///test/quickbrown.html"]); if (status !== 0 || !/quick.*brown/i.test(output)) process.exit(1); console.log("========== WebAssembly smoke test passed =========="); }).catch(error => { console.error(error); process.exit(1); });' "$(SITE)"
