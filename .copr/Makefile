# Copr "make_srpm" SCM source method entry point. Invoked by Copr's
# builder as `make -f .copr/Makefile srpm outdir=<dir> spec=<path>`,
# already inside a clone of this repo at the configured commit/branch.
#
# This repo is Fedora's dist-git layout for xorg-x11-xinit (spec, plain
# flat-file sources, patches) plus our own fix commit(s) on top, not a
# source tree. Fedora's own spec still uses a literal Release (no
# rpmautospec); rather than track Fedora's own current Release number to
# stay ahead of it (fragile - a future Fedora bump could outrank us),
# this follows the same convention as the other forked packages here
# (gdm, mutter, gnome-shell, ...): Release = <HEAD commit timestamp>.g
# <short hash>%{?dist}. The timestamp numerically outranks any realistic
# small-integer Fedora Release on its own, with no base number to track,
# and it changes automatically with every new commit.

PKG := xorg-x11-xinit
VERSION := 1.4.3

MOCK_CHROOT ?= fedora-45-x86_64
LOCAL_REPO ?= $(HOME)/gnome-x11-build/copr/RPMS-f45

.PHONY: srpm
srpm:
	set -e; \
	{ command -v git >/dev/null 2>&1 && command -v spectool >/dev/null 2>&1; } \
		|| dnf install -y git-core rpmdevtools; \
	workdir=$$(mktemp -d); \
	mkdir -p "$$workdir/SOURCES"; \
	cp *.patch Xsession xinitrc xinitrc-common Xclients Xmodmap Xresources \
		localuser.sh xinit-compat xinit-compat.desktop "$$workdir/SOURCES/"; \
	hash=$$(git rev-parse --short=8 HEAD); \
	ts=$$(git log -1 --format=%ct HEAD); \
	sed -e "s/^Release:.*/Release:    $${ts}.g$${hash}%{?dist}/" \
		"$(spec)" > "$$workdir/$(PKG).spec"; \
	spectool -g -C "$$workdir/SOURCES" "$$workdir/$(PKG).spec"; \
	( cd "$$workdir/SOURCES" && sha512sum -c "$(CURDIR)/sources" ); \
	rpmbuild -bs \
		--define "_topdir $$workdir" \
		--define "_sourcedir $$workdir/SOURCES" \
		--define "_specdir $$workdir" \
		--define "_builddir $$workdir/BUILD" \
		--define "_srcrpmdir $(outdir)" \
		"$$workdir/$(PKG).spec"; \
	rm -rf "$$workdir"

# Local-only: build the SRPM (above) into a binary RPM via mock, then
# publish it into the shared local repo like the other packages do. Copr
# never calls this target.
.PHONY: local-build
local-build: outdir := $(CURDIR)/srpm-out-f45
local-build:
	set -e; \
	mkdir -p "$(outdir)"; \
	$(MAKE) -f .copr/Makefile srpm outdir="$(outdir)" spec=xorg-x11-xinit.spec; \
	mkdir -p "$(LOCAL_REPO)/x86_64" "$(LOCAL_REPO)/noarch"; \
	[ -f "$(LOCAL_REPO)/repodata/repomd.xml" ] || createrepo_c "$(LOCAL_REPO)"; \
	srpm=$$(ls -t "$(outdir)"/$(PKG)-$(VERSION)-*.src.rpm | head -n1); \
	resultdir=$$(mktemp -d); \
	physical_cores=$$(lscpu | awk -F: '/^Core\(s\) per socket/{c=$$2} /^Socket\(s\)/{s=$$2} END{gsub(/ /,"",c); gsub(/ /,"",s); print c*s}'); \
	mock -r $(MOCK_CHROOT) --resultdir="$$resultdir" \
		--define "_smp_mflags -j$${physical_cores}" \
		--rebuild "$$srpm"; \
	cp "$$resultdir"/*.x86_64.rpm "$(LOCAL_REPO)/x86_64/" 2>/dev/null || true; \
	cp "$$resultdir"/*.noarch.rpm "$(LOCAL_REPO)/noarch/" 2>/dev/null || true; \
	createrepo_c "$(LOCAL_REPO)"; \
	rm -rf "$$resultdir"
