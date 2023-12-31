# This Makefile is used by CI/CD builds.
#
# Prerequisites:
# 	- Docker 1.9.1 or newer
# 	- You must be a part of 'docker' group to use Docker without sudo
# 	- Git
#

OPS_URL ?=
LOCALDIR := $(dir $(CURDIR)/$(word $(words $(MAKEFILE_LIST)),$(MAKEFILE_LIST)))
LOCAL_BUILDDIR ?= /gopath/src/github.com/gravitational/gravity/build
# Path to the local gravity build assets directory used inside the container
LOCAL_GRAVITY_BUILDDIR ?= /gopath/src/github.com/gravitational/gravity/build/$(GRAVITY_VERSION)

DOCKER_ARGS ?= --pull
BBOX = gravity-buildbox:latest

GRAVITY_WEB_APP_DIR ?= $(abspath $(LOCALDIR)/../web)/

# Git versions (refspecs) for build targets
TELEPORT_TAG ?= master
PLANET_TAG ?= master
PLANET_BRANCH ?= $(PLANET_TAG)
BANDWAGON_TAG ?= 1.0.0
BANDWAGON_BRANCH ?= $(BANDWAGON_TAG)
LOGGING_APP_TAG ?= 0.0.1
LOGGING_APP_BRANCH ?= $(LOGGING_APP_TAG)
MONITORING_APP_TAG ?= 0.0.1
MONITORING_APP_BRANCH ?= $(MONITORING_APP_TAG)
K8S_APP_TAG ?= 0.0.1
TILLER_APP_TAG ?= 0.0.1
GOLFLAGS ?= -w -s

# Git repositories
TELEPORT_REPO = git@github.com:gravitational/teleport.git
PLANET_REPO = git@github.com:gravitational/planet.git
LOGGING_APP_REPO = git@github.com:gravitational/logging-app.git
MONITORING_APP_REPO = git@github.com:gravitational/monitoring-app.git
BANDWAGON_REPO = git@github.com:gravitational/bandwagon.git

# Amazon S3
BUILD_BUCKET_URL = s3://clientbuilds.gravitational.io
S3_OPTS = --region us-east-1
GRAVITY_BUCKET := $(BUILD_BUCKET_URL)/gravity/$(GRAVITY_VERSION)
LATEST_GRAVITY_BUCKET := $(BUILD_BUCKET_URL)/gravity/latest

# Directory with sources inside the container
SRCDIR := /gopath/src/github.com/gravitational/gravity

# Where teleport sources are inside build container
TELEPORT_SRCDIR_CONTAINER := /gopath/src/github.com/gravitational/teleport
TELEPORT_PKG_PATH := github.com/gravitational/teleport

DNS_APP_IMAGE := dns-app

OS := $(shell uname | tr '[:upper:]' '[:lower:]')
ARCH := $(shell uname -m)

# Address of OpsCenter to publish telekube binaries to
DISTRIBUTION_OPSCENTER ?= https://get.gravitational.io
S3_DISTRIBUTION_BUCKET ?= s3://get.gravitational.io/releases/$(GRAVITY_VERSION)

# Telekube package names
TELEKUBE_GRAVITY_PKG := gravitational.io/gravity_$(OS)_$(ARCH):$(GRAVITY_TAG)
TELEKUBE_TELE_PKG := gravitational.io/tele_$(OS)_$(ARCH):$(GRAVITY_TAG)
TELEKUBE_TSH_PKG := gravitational.io/tsh_$(OS)_$(ARCH):$(GRAVITY_TAG)

# Version of tsh binary that gets published into distribution OpsCenter, may differ from
# the one Gravity currently depends on
TELEKUBE_TSH_TAG := v3.2.13

# Extra flags that may be provided when publishing telekube artifacts (e.g. --insecure)
TELEKUBE_PUBLISH_FLAGS ?=

BINARIES ?= tele gravity terraform-provider-gravity

# release tarball path
RELEASE_TARBALL_NAME := gravity-$(GRAVITY_VERSION)-$(OS)-$(ARCH)-bin.tar.gz
RELEASE_OUT := $(GRAVITY_BUILDDIR)/$(RELEASE_TARBALL_NAME)

BASH ?= /bin/bash

# packages to test
TEST_PACKAGES ?= $(GRAVITY_PKG_PATH)/lib/... $(GRAVITY_PKG_PATH)/tool/...

# Export variables for recursive make invocations
export GRAVITY_PKG_PATH

include etcd.mk

TEST_ETCD ?= true
TEST_K8S ?= false

ifdef SKIP_S3
	AWS := :
else
	AWS := aws --quiet
endif

#
# Runs tests inside a build container
#
.PHONY: test
test: buildbox test-etcd
	docker run --net=host --rm=true $(NOROOT) \
		-v $(TOP):$(SRCDIR) \
		-e "GRAVITY_PKG_PATH=$(GRAVITY_PKG_PATH)" \
		-e "TEST_PACKAGES=$(TEST_PACKAGES)" \
		-t $(BBOX) \
		dumb-init make -C $(SRCDIR)/build.assets FLAGS='-cover -race' TEST_ETCD=true TEST_K8S=$(TEST_K8S) test-inside-container


.PHONY: test-inside-container
test-inside-container:
	TEST_ETCD=$(TEST_ETCD) TEST_ETCD_CONFIG=$(TEST_ETCD_CONFIG) TEST_K8S=$(TEST_K8S) go test $(TEST_PACKAGES) $(FLAGS)
# TODO: It turns out "go vet" never really worked because we used incorrect package path
# and now that I've fixed it, it produces a gazillion of warnings. So I'm commenting this
# out for the time being, should uncomment once we've fixed the warnings.
#	go vet github.com/gravitational/gravity/lib/...

#
# builds everything and puts the output into $(LOCAL_BUILDDIR). very slow the first time, becomes reasonable on subsequent builds.
# WARNING: do not mess with it. This is what CI/CD is using.
#
.PHONY: production
production: buildbox
	$(MAKE) build
	$(MAKE) telekube-app


# $(call build_app,name,repository,branch,package,version)
# $1 - application name
# $2 - git repo
# $3 - git branch
# $4 - package
# $5 - package version
define build_app
	echo "Building $1 in $2($3) as package $4" && echo ""
	if ! $(AWS) s3 cp $(S3_OPTS) $(BUILD_BUCKET_URL)/$1/$3 $@; then \
		echo "" && echo "----> Building $1 package..." && echo "" && \
		mkdir -p $(BUILDDIR) && \
		cd $(BUILDDIR) && \
		( test -d $(BUILDDIR)/$1 || git clone $2 $1 ) && \
		cd $(BUILDDIR)/$1 && \
		git fetch --all && \
		git checkout $3 && \
		git pull origin $3 && \
		make -C $(BUILDDIR)/$1 import \
			OPS_URL=$(OPS_URL) \
			GRAVITY="$(GRAVITY)" \
			VERSION="$5" && \
		$(GRAVITY) package export $4 $@; \
	fi
endef

#
# build gravity binaries
#
.PHONY: build
build:
	@if [ "$(OS)" = "darwin" ]; then \
		$(MAKE) build-on-host; \
		ln -sf $(GRAVITY_BUILDDIR) $(GRAVITY_CURRENT_BUILDDIR); \
	else \
		$(MAKE) build-in-container; \
		ln --symbolic --force --no-target-directory $(GRAVITY_BUILDDIR) $(GRAVITY_CURRENT_BUILDDIR); \
	fi;

#
# generate gRPC code
#
.PHONY: grpc
grpc: buildbox grpc-agents grpc-validation grpc-installer

.PHONY: grpc-installer
grpc-installer:
	docker run --rm=true -u $$(id -u) \
		   -v $(TOP):$(SRCDIR) \
		   $(BBOX) \
		   dumb-init make -C $(SRCDIR)/lib/install/proto

.PHONY: grpc-agents
grpc-agents:
	docker run --rm=true -u $$(id -u) \
		   -v $(TOP):$(SRCDIR) \
		   $(BBOX) \
		   dumb-init make -C $(SRCDIR)/lib/rpc/proto

.PHONY: grpc-validation
grpc-validation:
	docker run --rm=true -u $$(id -u) \
		   -v $(TOP):$(SRCDIR) \
		   $(BBOX) \
		   dumb-init make -C $(SRCDIR)/lib/network/validation/proto

#
# this is a temporary target until we upgrade docker packages
# to use sirupsen/logrus
#
.PHONY: validate-deps
validate-deps: buildbox
	$(eval TMP := $(shell mktemp -d))
	$(eval REPOFILES := $(addprefix $(TOP),lib tool Makefile version.sh build.assets vendor .git .gitignore Gopkg.lock Gopkg.toml))
	cp -r $(REPOFILES) $(TMP)
	docker run --rm=true \
		   -v $(GITHUB_SSH_KEY):/root/.ssh/id_rsa \
		   -v $(TMP):$(SRCDIR) \
		   $(BBOX) \
		   dumb-init make -C $(SRCDIR) validate-deps

#
# build gravity binaries in a buildbox container
#
.PHONY: build-in-container
build-in-container: buildbox
	docker run --rm=true -u $$(id -u) \
		   -v $(TOP):$(SRCDIR) \
		   -e "LOCAL_BUILDDIR=$(LOCAL_BUILDDIR)" \
		   -e "LOCAL_GRAVITY_BUILDDIR=$(LOCAL_GRAVITY_BUILDDIR)" \
		   -e "GRAVITY_PKG_PATH=$(GRAVITY_PKG_PATH)" \
		   -e "GRAVITY_VERSION=$(GRAVITY_VERSION)" \
		   -e "GRAVITY_TAG=$(GRAVITY_TAG)" \
		   -e 'GRAVITY_LINKFLAGS=$(GRAVITY_LINKFLAGS)' \
		   -e 'BINARIES=$(BINARIES)' \
		   $(BBOX) \
		   dumb-init make -C $(SRCDIR)/build.assets -j $(BINARIES)

#
# build gravity binaries on host
#
.PHONY: build-on-host
build-on-host:
	cd $(GOPATH)/src && \
		LOCAL_GRAVITY_BUILDDIR=$(GRAVITY_BUILDDIR) \
			$(MAKE) -C $(GOPATH)/src/github.com/gravitational/gravity/build.assets -j \
			$(BINARIES)

#
# removes build artifacts.
# WARNING: do not mess with it. This is what CI/CD calls before calling 'make production'
#
.PHONY: clean
clean:
	make -C $(GRAVITY_WEB_APP_DIR) clean
	rm -rf $(LOCAL_BUILDDIR)


#
# publish gravity, tele and tsh binaries to distribution OpsCenter
#
.PHONY: publish-telekube
publish-telekube: build
	TELEPORT_REPOTAG=$(TELEKUBE_TSH_TAG) $(MAKE) build-tsh
	@echo -e "\n----> Publishing Gravity to $(DISTRIBUTION_OPSCENTER)...\n"
	$(GRAVITY_OUT) package delete --ops-url=$(DISTRIBUTION_OPSCENTER) $(TELEKUBE_PUBLISH_FLAGS) --force $(TELEKUBE_GRAVITY_PKG) && \
	$(GRAVITY_OUT) package import --ops-url=$(DISTRIBUTION_OPSCENTER) $(TELEKUBE_PUBLISH_FLAGS) $(GRAVITY_OUT) $(TELEKUBE_GRAVITY_PKG)
	@echo -e "\n----> Publishing Tele to $(DISTRIBUTION_OPSCENTER)...\n"
	$(GRAVITY_OUT) package delete --ops-url=$(DISTRIBUTION_OPSCENTER) $(TELEKUBE_PUBLISH_FLAGS) --force $(TELEKUBE_TELE_PKG) && \
	$(GRAVITY_OUT) package import --ops-url=$(DISTRIBUTION_OPSCENTER) $(TELEKUBE_PUBLISH_FLAGS) $(TELE_OUT) $(TELEKUBE_TELE_PKG)
	@echo -e "\n----> Publishing Tsh to $(DISTRIBUTION_OPSCENTER)...\n"
	$(GRAVITY_OUT) package delete --ops-url=$(DISTRIBUTION_OPSCENTER) $(TELEKUBE_PUBLISH_FLAGS) --force $(TELEKUBE_TSH_PKG) && \
	$(GRAVITY_OUT) package import --ops-url=$(DISTRIBUTION_OPSCENTER) $(TELEKUBE_PUBLISH_FLAGS) $(TSH_OUT) $(TELEKUBE_TSH_PKG)
	@echo -e "\n----> Publishing terraform-provider-gravity to $(DISTRIBUTION_OPSCENTER)...\n"
	$(GRAVITY_OUT) package delete --ops-url=$(DISTRIBUTION_OPSCENTER) $(TELEKUBE_PUBLISH_FLAGS) --force $(TF_PROVIDER_GRAVITY_PKG) && \
	$(GRAVITY_OUT) package import --ops-url=$(DISTRIBUTION_OPSCENTER) $(TELEKUBE_PUBLISH_FLAGS) $(TF_PROVIDER_GRAVITY_OUT) $(TF_PROVIDER_GRAVITY_PKG)
	@echo -e "\n----> Publishing terraform-provider-gravityenterprise to $(DISTRIBUTION_OPSCENTER)...\n"
	$(GRAVITY_OUT) package delete --ops-url=$(DISTRIBUTION_OPSCENTER) $(TELEKUBE_PUBLISH_FLAGS) --force $(TF_PROVIDER_GRAVITYENTERPRISE_PKG) && \
	$(GRAVITY_OUT) package import --ops-url=$(DISTRIBUTION_OPSCENTER) $(TELEKUBE_PUBLISH_FLAGS) $(TF_PROVIDER_GRAVITYENTERPRISE_OUT) $(TF_PROVIDER_GRAVITYENTERPRISE_PKG)


#
# publish telekube/opscenter application packages to S3
#
.PHONY: publish-telekube-s3
publish-telekube-s3:
	@echo -e "\n----> Publishing Telekube application to $(S3_DISTRIBUTION_BUCKET)...\n"
	$(AWS) s3 cp $(S3_OPTS) "$(GRAVITY_BUILDDIR)/telekube.tar" $(S3_DISTRIBUTION_BUCKET)/
	@echo -e "\n----> Publishing Ops Center application to $(S3_DISTRIBUTION_BUCKET)...\n"
	$(AWS) s3 cp $(S3_OPTS) "$(GRAVITY_BUILDDIR)/opscenter.tar" $(S3_DISTRIBUTION_BUCKET)/
	@echo -e "\n----> Publishing Gravity to $(S3_DISTRIBUTION_BUCKET)...\n"
	$(AWS) s3 cp $(S3_OPTS) "$(GRAVITY_OUT)" $(S3_DISTRIBUTION_BUCKET)/
	@echo -e "\n----> Publishing Tele to $(S3_DISTRIBUTION_BUCKET)...\n"
	$(AWS) s3 cp $(S3_OPTS) "$(TELE_OUT)" $(S3_DISTRIBUTION_BUCKET)/


#
# publish artifacts to S3
#
.PHONY: publish
publish: push-teleport push-gravity push-bandwagon push-logging-app push-monitoring-app

#
# prepare ansible variables for publishing to the hub
#
.PHONY: hub-vars
hub-vars:
	$(BASH) ./hub-vars.sh

#
# produce release tarball with binaries
#
.PHONY: release
release:
	$(BASH) ./release.sh

.PHONY: push-gravity
push-gravity:
	$(foreach t,$(GRAVITY_PUBLISH_TARGETS),$(AWS) s3 cp $(S3_OPTS) "$(t)" $(GRAVITY_BUCKET)/;)
	$(foreach t,$(GRAVITY_PUBLISH_TARGETS),$(AWS) s3 cp $(S3_OPTS) "$(t)" $(LATEST_GRAVITY_BUCKET)/;)
	@echo published

$(GRAVITY_BUILDDIR):
	mkdir -p $@

# Builds binaries inside a docker container
.PHONY: $(BINARIES)
$(BINARIES):
	@echo -e "\n----> Building $(LOCAL_GRAVITY_BUILDDIR)/$@ binary...\n"
	go build -o $(LOCAL_GRAVITY_BUILDDIR)/$@ -ldflags $(GRAVITY_LINKFLAGS) $(GRAVITY_PKG_PATH)/tool/$@
	@echo "\nDone --> $(LOCAL_GRAVITY_BUILDDIR)/$@\n"

.PHONY: compile
compile: | $(GRAVITY_BUILDDIR)
	$(MAKE) -j $(LOCAL_GRAVITY_BUILDDIR)/tele $(LOCAL_GRAVITY_BUILDDIR)/gravity

.PHONY: gravity-package
gravity-package:
	$(GRAVITY) package delete $(GRAVITY_PKG) $(DELETE_OPTS) && \
	$(GRAVITY) package import $(GRAVITY_OUT) $(GRAVITY_PKG)

.PHONY: web-assets
web-assets: web-app
	tar -C $(GRAVITY_WEB_APP_DIR)/dist -czf $(WEB_ASSETS_OUT) .
	$(GRAVITY) package delete $(WEB_ASSETS_PKG) $(DELETE_OPTS) && \
		$(GRAVITY) package import $(WEB_ASSETS_OUT) $(WEB_ASSETS_PKG)

.PHONY: web-app
web-app:
	$(MAKE) -C $(GRAVITY_WEB_APP_DIR)

K8S_IMPORT_OPTIONS := \
		--version=$(K8S_APP_TAG) \
		--set-dep=$(GRAVITY_PKG) \
		--set-dep=$(WEB_ASSETS_PKG) \
		--set-dep=$(TELEPORT_PKG) \
		--set-dep=$(PLANET_PKG) \
		--set-dep=$(RBAC_APP_PKG) \
		--set-dep=$(DNS_APP_PKG) \
		--set-dep=$(LOGGING_APP_PKG) \
		--set-dep=$(MONITORING_APP_PKG) \
		--set-dep=$(BANDWAGON_PKG) \
		--set-dep=$(TILLER_APP_PKG) \
		--set-dep=$(SITE_APP_PKG)

.PHONY: rbac-app
rbac-app:
	@echo -e "\n----> Building rbac-app...\n"
	VERSION=$(RBAC_APP_TAG) make -C "$(ASSETSDIR)/rbac-app"
	VERSION=$(RBAC_APP_TAG) GRAVITY="$(GRAVITY)" OPS_URL=$(OPS_URL) make -C $(ASSETSDIR)/rbac-app import
	$(GRAVITY) package export $(RBAC_APP_PKG) $(RBAC_APP_OUT)

.PHONY: k8s-app
k8s-app: gravity-package teleport planet web-assets site-app monitoring-app logging-app tiller-app rbac-app dns-app bandwagon
	@echo -e "\n----> Building kubernetes-app...\n"
	- $(GRAVITY) app delete $(K8S_APP_PKG) $(DELETE_OPTS) && \
	  $(GRAVITY) app import $(ASSETSDIR)/kubernetes $(VENDOR_OPTS) $(K8S_IMPORT_OPTIONS) \
		--exclude=**/*.tf && \
	  $(GRAVITY) package export $(K8S_APP_PKG) $(K8S_APP_OUT)

TELEKUBE_IMPORT_OPTIONS := \
		--version=$(TELEKUBE_APP_TAG)

.PHONY: telekube-app
telekube-app: k8s-app
	@echo -e "\n----> Building telekube-app...\n"
	- $(GRAVITY) app delete $(TELEKUBE_APP_PKG) $(DELETE_OPTS) && \
	  GRAVITY_K8S_VERSION=$(K8S_VER) $(GRAVITY) app import $(ASSETSDIR)/telekube $(VENDOR_OPTS) $(TELEKUBE_IMPORT_OPTIONS) && \
	  $(GRAVITY) package export $(TELEKUBE_APP_PKG) $(TELEKUBE_APP_OUT)

.PHONY: dns-app
dns-app:
	@echo -e "\n----> Building $@...\n"
	VERSION=$(DNS_APP_TAG) GRAVITY="$(GRAVITY)" OPS_URL=$(OPS_URL) make -C $(ASSETSDIR)/dns-app import
	$(GRAVITY) package export $(DNS_APP_PKG) $(DNS_APP_OUT)

.PHONY: tiller-app
tiller-app:
	@echo -e "\n----> Building tiller-app...\n"
	VERSION=$(TILLER_APP_TAG) GRAVITY="$(GRAVITY)" OPS_URL=$(OPS_URL) make -C $(ASSETSDIR)/tiller-app import
	$(GRAVITY) package export $(TILLER_APP_PKG) $(TILLER_APP_OUT)

.PHONY: site-app
site-app:
	$(eval TMPDIR := $(shell mktemp -d --tmpdir=$(GRAVITY_BUILDDIR)))
	cp -r $(ASSETSDIR)/site-app/* $(TMPDIR)
	cp $(GRAVITY_BUILDDIR)/gravity $(TMPDIR)/images/site
	cd $(TMPDIR) && VERSION=$(GRAVITY_TAG) GRAVITY="$(GRAVITY)" OPS_URL=$(OPS_URL) make import
	$(GRAVITY) package export $(SITE_APP_PKG) $(SITE_APP_OUT)
	rm -rf $(TMPDIR)

.PHONY: ntp-app
ntp-app:
	@echo -e "\n----> Building ntp-app...\n"
	make -C $(ASSETSDIR)/ntp-app
	$(GRAVITY) app delete $(NTP_APP_PKG) $(DELETE_OPTS) && \
		$(GRAVITY) app import $(ASSETSDIR)/ntp-app $(VENDOR_OPTS)
#
# dependency: teleport
#
.PHONY: teleport pull-teleport push-teleport
teleport: pull-teleport build-teleport
	$(GRAVITY) package delete $(TELEPORT_PKG) $(DELETE_OPTS) && \
		$(GRAVITY) package import $(TELEPORT_OUT) $(TELEPORT_PKG)

build-teleport:
	if [ ! -e "$(TELEPORT_OUT)" ]; then $(MAKE) build-teleport-sub; fi

# rule to build teleport from source
build-teleport-sub: TMPDIR := $(shell mktemp -d)
build-teleport-sub: clone-teleport
	@echo -e "\n----> Building Teleport binaries...\n"
	cd $(TELEPORT_SRCDIR)/teleport/build.assets && make build-binaries
	cp teleport.manifest.json $(TMPDIR)/orbit.manifest.json
	mkdir -p $(TMPDIR)/rootfs/usr/bin
	mkdir -p $(TMPDIR)/rootfs/usr/bin $(TMPDIR)/rootfs/usr/share/teleport
	cp -f $(TELEPORT_SRCDIR)/teleport/build/teleport $(TMPDIR)/rootfs/usr/bin
	cp -f $(TELEPORT_SRCDIR)/teleport/build/tctl $(TMPDIR)/rootfs/usr/bin
	cp -f $(TELEPORT_SRCDIR)/teleport/build/tsh $(TMPDIR)/rootfs/usr/bin
	tar -C $(TMPDIR) -czf $(TELEPORT_OUT) .
	rm -rf $(TMPDIR)

#
# build tsh binary
#
.PHONY: build-tsh
build-tsh:
	if [ "$(OS)" = "darwin" ]; then $(MAKE) build-tsh-on-host; else $(MAKE) build-tsh-in-container; fi

#
# build tsh binary in a buildbox container
#
.PHONY: build-tsh-in-container
build-tsh-in-container: clone-teleport
	docker run --rm=true -u $$(id -u) \
           -v $(TOP):$(SRCDIR) \
		   -v $(TELEPORT_SRCDIR)/teleport:$(TELEPORT_SRCDIR_CONTAINER) \
		   -e "GRAVITY_PKG_PATH=$(GRAVITY_PKG_PATH)" \
		   -e "GRAVITY_VERSION=$(GRAVITY_VERSION)" \
		   -e "GRAVITY_TAG=$(GRAVITY_TAG)" \
		   -e "LOCAL_GRAVITY_BUILDDIR=$(LOCAL_GRAVITY_BUILDDIR)" \
		   $(BBOX) \
		   dumb-init make -C $(SRCDIR)/build.assets build-tsh-inside-container

#
# build tsh binary on host
#
.PHONY: build-tsh-on-host
build-tsh-on-host:
	@echo "\n----> Building Tsh binary on host...\n"
	GOPATH=$(BUILDDIR) go get -v $(TELEPORT_PKG_PATH) && \
		cd $(BUILDDIR)/src/$(TELEPORT_PKG_PATH) && \
		git fetch --all --tags && \
		git checkout $(TELEPORT_REPOTAG) && \
		GOPATH=$(BUILDDIR) go build -ldflags "$(GOLFLAGS)" -o $(TSH_OUT) ./tool/tsh
	@echo "Done --> $(TSH_OUT)"

.PHONY: build-tsh-inside-container
build-tsh-inside-container:
	@echo "\n----> Building Tsh binary inside container...\n"
	go build -ldflags "$(GOLFLAGS)" -o $(LOCAL_GRAVITY_BUILDDIR)/tsh $(TELEPORT_PKG_PATH)/tool/tsh
	@echo "Done --> $(LOCAL_GRAVITY_BUILDDIR)/tsh"

.PHONY: clone-teleport
clone-teleport:
	@mkdir -p $(TELEPORT_SRCDIR)
	if [ ! -d "$(TELEPORT_SRCDIR)/teleport" ]; then cd $(TELEPORT_SRCDIR) && git clone $(TELEPORT_REPO); fi
	cd $(TELEPORT_SRCDIR)/teleport && git fetch --all --tags && git checkout $(TELEPORT_REPOTAG)

pull-teleport:
	-$(AWS) s3 cp $(S3_OPTS) $(BUILD_BUCKET_URL)/teleport/$(TELEPORT_TAG) $(TELEPORT_OUT)

push-teleport: $(TELEPORT_OUT)
	$(AWS) s3 cp $(S3_OPTS) $(TELEPORT_OUT) $(BUILD_BUCKET_URL)/teleport/$(TELEPORT_TAG)

#
# dependency: planet master/node packages
#
.PHONY: planet
planet: pull-planet build-planet
	$(GRAVITY) package delete $(PLANET_PKG) $(DELETE_OPTS) && \
		$(GRAVITY) package import $(PLANET_OUT) $(PLANET_PKG)

.PHONY: build-planet
build-planet:
	if [ ! -e "$(PLANET_OUT)" ]; then $(MAKE) build-planet-sub; fi

# rule to build planet from source
.PHONY: build-planet-sub
build-planet-sub:
	@mkdir -p $(PLANET_SRCDIR)
	@mkdir -p $(PLANET_BINDIR)
	if [ ! -d "$(PLANET_SRCDIR)/planet" ]; then cd $(PLANET_SRCDIR) && git clone $(PLANET_REPO); fi
	cd $(PLANET_SRCDIR)/planet && git fetch --all && git checkout $(PLANET_BRANCH)
	@echo -e "\n----> Building Planet images...\n"
	BUILDDIR=$(PLANET_BUILDDIR) make -C $(PLANET_SRCDIR)/planet production USER=jenkins
	mv -f $(PLANET_BUILDDIR)/planet.tar.gz $(PLANET_OUT)

.PHONY: pull-planet
pull-planet:
	-$(AWS) s3 sync $(S3_OPTS) $(BUILD_BUCKET_URL)/planet/$(PLANET_TAG) $(PLANET_BINDIR)


#
# dependency: installer step app
#
.PHONY: bandwagon push-bandwagon
bandwagon: $(BANDWAGON_OUT)
	if ! $(GRAVITY) package list | grep $(BANDWAGON_PKG); then \
		$(GRAVITY) app import $(BANDWAGON_OUT) $(IMPORT_OPTS) ; \
	fi

$(BANDWAGON_OUT):
	$(call build_app,bandwagon,$(BANDWAGON_REPO),$(BANDWAGON_BRANCH),$(BANDWAGON_PKG),$(BANDWAGON_TAG))

push-bandwagon: $(BANDWAGON_OUT)
	$(AWS) s3 cp $(S3_OPTS) $(BANDWAGON_OUT) $(BUILD_BUCKET_URL)/bandwagon/$(BANDWAGON_TAG)

#
# dependency: logging app
#
.PHONY: logging-app push-logging-app
logging-app: $(LOGGING_APP_OUT)
	if ! $(GRAVITY) package list | grep $(LOGGING_APP_PKG); then \
		$(GRAVITY) app import $(LOGGING_APP_OUT) $(IMPORT_OPTS) ; \
	fi

$(LOGGING_APP_OUT):
	$(call build_app,logging-app,$(LOGGING_APP_REPO),$(LOGGING_APP_BRANCH),$(LOGGING_APP_PKG),$(LOGGING_APP_TAG))

push-logging-app: $(LOGGING_APP_OUT)
	$(AWS) s3 cp $(S3_OPTS) $(LOGGING_APP_OUT) $(BUILD_BUCKET_URL)/logging-app/$(LOGGING_APP_TAG)

#
# dependency: monitoring app
#
.PHONY: monitoring-app push-monitoring-app
monitoring-app: $(MONITORING_APP_OUT)
	if ! $(GRAVITY) package list | grep $(MONITORING_APP_PKG); then \
		$(GRAVITY) app import $(MONITORING_APP_OUT) $(IMPORT_OPTS) ; \
	fi

$(MONITORING_APP_OUT):
	$(call build_app,monitoring-app,$(MONITORING_APP_REPO),$(MONITORING_APP_BRANCH),$(MONITORING_APP_PKG),$(MONITORING_APP_TAG))

push-monitoring-app: $(MONITORING_APP_OUT)
	$(AWS) s3 cp $(S3_OPTS) $(MONITORING_APP_OUT) $(BUILD_BUCKET_URL)/monitoring-app/$(MONITORING_APP_TAG)

#
# enters bbox container (for debugging the containerized build process)
#
.PHONY: enter
enter:
	docker run -ti --rm=true $(BBOX) /bin/bash

#
# buildbox container: container used for building packages
#
.PHONY: buildbox
buildbox:
	docker build \
		--build-arg PROTOC_VER=$(PROTOC_VER) \
		--build-arg PROTOC_PLATFORM=$(PROTOC_PLATFORM) \
		--build-arg GOGO_PROTO_TAG=$(GOGO_PROTO_TAG) \
		--build-arg GRPC_GATEWAY_TAG=$(GRPC_GATEWAY_TAG) \
		--build-arg VERSION_TAG=$(VERSION_TAG) \
		$(DOCKER_ARGS) --tag $(BBOX) .
