# Copr "make_srpm" SCM source method entry point. Invoked by Copr's
# builder as `make -f .copr/Makefile srpm outdir=<dir> spec=<path>`,
# already inside a clone of this repo at the configured commit/branch.
#
# This repo is Fedora's dist-git layout for xorg-x11-xinit (spec, plain
# flat-file sources, patches) plus our own fix commit(s) on top, not a
# source tree. Fedora's own spec still uses a literal Release (no
# rpmautospec), so instead of hardcoding a Release bump that would need
# manual editing on every future commit, this Makefile computes it at
# srpm time: BASE_COMMIT_COUNT is the commit count of the unmodified
# dist-git f45 import (`git rev-list --count HEAD` at 2bad89a, before
# our first fix commit); the suffix is how many commits we've added
# since, so it goes up automatically as this branch gets more commits.

PKG := xorg-x11-xinit
VERSION := 1.4.3
BASE_RELEASE := 6
BASE_COMMIT_COUNT := 166

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
	total_commits=$$(git rev-list --count HEAD); \
	rel_suffix=$$(( total_commits - $(BASE_COMMIT_COUNT) )); \
	[ "$$rel_suffix" -ge 1 ] || rel_suffix=1; \
	sed -e "s/^Release:.*/Release:    $(BASE_RELEASE).$${rel_suffix}%{?dist}/" \
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
