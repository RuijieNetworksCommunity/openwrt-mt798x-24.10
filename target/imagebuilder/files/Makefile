# Makefile for OpenWrt
#
# Copyright (C) 2007-2015 OpenWrt.org
#
# This is free software, licensed under the GNU General Public License v2.
# See /LICENSE for more information.
#

TOPDIR:=${CURDIR}
LC_ALL:=C
LANG:=C
export TOPDIR LC_ALL LANG
export OPENWRT_VERBOSE=s
all: help

export PATH:=$(TOPDIR)/staging_dir/host/bin:$(PATH)

ifneq ($(OPENWRT_BUILD),1)
  override OPENWRT_BUILD=1
  export OPENWRT_BUILD
endif

include rules.mk
include $(INCLUDE_DIR)/debug.mk
include $(INCLUDE_DIR)/depends.mk
include $(INCLUDE_DIR)/rootfs.mk

include $(INCLUDE_DIR)/version.mk
export REVISION

define Helptext
Available Commands:
	help:	This help text
	info:	Show a list of available target profiles
	clean:	Remove images and temporary build files
	image:	Build an image (see below for more information).

Building images:
	By default 'make image' will create an image with the default
	target profile and package set. You can use the following parameters
	to change that:

	make image PROFILE="<profilename>" # override the default target profile
	make image PACKAGES="<pkg1> [<pkg2> [<pkg3> ...]]" # include extra packages
	make image FILES="<path>" # include extra files from <path>
	make image BIN_DIR="<path>" # alternative output directory for the images
	make image EXTRA_IMAGE_NAME="<string>" # Add this to the output image filename (sanitized)
	make image DISABLED_SERVICES="<svc1> [<svc2> [<svc3> ..]]" # Which services in /etc/init.d/ should be disabled
	make image ADD_LOCAL_KEY=1 # store locally generated signing key in built images
	make image ROOTFS_PARTSIZE="<size>" # override the default rootfs partition size in MegaBytes

Print manifest:
	List "all" packages which get installed into the image.
	You can use the following parameters:

	make manifest PROFILE="<profilename>" # override the default target profile
	make manifest PACKAGES="<pkg1> [<pkg2> [<pkg3> ...]]" # include extra packages
	make manifest STRIP_ABI=1 # remove ABI version from printed package names

endef
$(eval $(call shexport,Helptext))

help: FORCE
	echo "$$$(call shvar,Helptext)"


# override variables from rules.mk
BUILD_KEY_APK_SEC=$(TOPDIR)/keys/local-private-key.pem
BUILD_KEY_APK_PUB=$(TOPDIR)/keys/local-public-key.pem
export PACKAGE_DIR:=$(TOPDIR)/packages
LISTS_DIR:=$(subst $(space),/,$(patsubst %,..,$(subst /,$(space),$(TARGET_DIR))))$(DL_DIR)
export PACKAGE_DIR_ALL:=$(TOPDIR)/packages

export OPKG_KEYS:=$(TOPDIR)/keys
OPKG:=$(call opkg,$(TARGET_DIR)) \
	-f $(TOPDIR)/repositories.conf \
	--verify-program $(SCRIPT_DIR)/opkg-key \
	--cache $(DL_DIR) \
	--lists-dir $(LISTS_DIR)

export APK_KEYS:=$(TOPDIR)/keys
APK:=$(call apk,$(TARGET_DIR)) \
	--repositories-file $(TOPDIR)/repositories \
	$(if $(CONFIG_SIGNATURE_CHECK),,--allow-untrusted) \
	--cache-dir $(DL_DIR)


include $(INCLUDE_DIR)/target.mk
-include .profiles.mk

USER_PROFILE ?= $(firstword $(PROFILE_NAMES))
PROFILE_LIST = $(foreach p,$(PROFILE_NAMES), \
	echo '$(patsubst DEVICE_%,%,$(p)):'; $(if $($(p)_NAME),echo '    $(subst ','"'"',$($(p)_NAME))'; ) \
	echo '    Packages: $($(p)_PACKAGES)'; echo '    hasImageMetadata: $($(p)_HAS_IMAGE_METADATA)'; \
	$(if $($(p)_SUPPORTED_DEVICES),echo '    SupportedDevices: $($(p)_SUPPORTED_DEVICES)';) )


.profiles.mk: .targetinfo
	@$(SCRIPT_DIR)/target-metadata.pl profile_mk $< '$(BOARD)/$(SUBTARGET)' > $@

staging_dir/host/.prereq-build: include/prereq-build.mk
	mkdir -p tmp
	@$(_SINGLE)$(NO_TRACE_MAKE) -j1 -r -s -f $(TOPDIR)/include/prereq-build.mk prereq IB=1 2>/dev/null || { \
		echo "Prerequisite check failed. Use FORCE=1 to override."; \
		false; \
	}
  ifneq ($(realpath $(TOPDIR)/include/prepare.mk),)
	@$(_SINGLE)$(NO_TRACE_MAKE) -j1 -r -s -f $(TOPDIR)/include/prepare.mk prepare 2>/dev/null || { \
		echo "Preparation failed."; \
		false; \
	}
  endif
	touch $@

_call_info: FORCE
	echo 'Current Target: "$(TARGETID)"'
	echo 'Current Revision: "$(REVISION)"'
	echo 'Default Packages: $(DEFAULT_PACKAGES)'
	echo 'Available Profiles:'
	echo; $(PROFILE_LIST)

BUILD_PACKAGES:=$(sort $(DEFAULT_PACKAGES) $($(USER_PROFILE)_PACKAGES) kernel)
# "-pkgname" in the package list means remove "pkgname" from the package list
BUILD_PACKAGES:=$(filter-out $(filter -%,$(BUILD_PACKAGES)) $(patsubst -%,%,$(filter -%,$(BUILD_PACKAGES))),$(BUILD_PACKAGES))
BUILD_PACKAGES:=$(USER_PACKAGES) $(BUILD_PACKAGES)
BUILD_PACKAGES:=$(filter-out $(filter -%,$(BUILD_PACKAGES)) $(patsubst -%,%,$(filter -%,$(BUILD_PACKAGES))),$(BUILD_PACKAGES))
PACKAGES:=

_call_image: staging_dir/host/.prereq-build
	echo 'Building images for $(BOARD)$(if $($(USER_PROFILE)_NAME), - $($(USER_PROFILE)_NAME))'
	echo 'Packages: $(BUILD_PACKAGES)'
	echo
	rm -rf $(TARGET_DIR) $(TARGET_DIR_ORIG)
	mkdir -p $(TARGET_DIR) $(BIN_DIR) $(TMP_DIR) $(DL_DIR)
	$(MAKE) package_reload
	$(MAKE) package_install
	$(MAKE) -s prepare_rootfs
	$(MAKE) -s build_image
	$(MAKE) -s json_overview_image_info
	$(MAKE) -s checksum

_call_manifest: FORCE
	rm -rf $(TARGET_DIR)
	mkdir -p $(TARGET_DIR) $(BIN_DIR) $(TMP_DIR) $(DL_DIR)
	$(MAKE) package_reload >/dev/null
	$(MAKE) package_install >/dev/null
ifeq ($(CONFIG_USE_APK),)
	$(OPKG) list-installed $(if $(STRIP_ABI),--strip-abi)
else
	$(APK) list --quiet --manifest --no-network
endif

package_index: FORCE
	@echo >&2
	@echo Building package index... >&2
	@mkdir -p $(TMP_DIR) $(TARGET_DIR)/tmp
ifeq ($(CONFIG_USE_APK),)
	(cd $(PACKAGE_DIR); $(SCRIPT_DIR)/ipkg-make-index.sh . > Packages && \
		gzip -9nc Packages > Packages.gz; \
		$(if $(CONFIG_SIGNATURE_CHECK), \
			$(STAGING_DIR_HOST)/bin/usign -S -m Packages -s $(BUILD_KEY)) \
	) >/dev/null 2>/dev/null
	$(OPKG) update >&2 || true
else
	$(APK) add --initdb
	(cd $(PACKAGE_DIR); $(APK) mkndx \
		$(if $(CONFIG_SIGNATURE_CHECK), --keys-dir $(APK_KEYS) --sign $(BUILD_KEY_APK_SEC)) \
		--allow-untrusted --output packages.adb *.apk) >/dev/null 2>/dev/null || true
	$(APK) update >&2 || true
endif

package_reload:
ifeq ($(CONFIG_USE_APK),)
	if [ -d "$(PACKAGE_DIR)" ] && ( \
			[ ! -f "$(PACKAGE_DIR)/Packages" ] || \
			[ ! -f "$(PACKAGE_DIR)/Packages.gz" ] || \
			[ "`find $(PACKAGE_DIR) -cnewer $(PACKAGE_DIR)/Packages.gz`" ] ); then \
		echo "Package list missing or not up-to-date, generating it." >&2 ;\
		$(MAKE) package_index; \
	else \
		mkdir -p $(TARGET_DIR)/tmp; \
		$(OPKG) update >&2 || true; \
	fi
else
	if [ -d "$(PACKAGE_DIR)" ] && ( \
			[ ! -f "$(PACKAGE_DIR)/packages.adb" ] || \
			[ "`find $(PACKAGE_DIR) -cnewer $(PACKAGE_DIR)/packages.adb`" ] ); then \
		echo "Package list missing or not up-to-date, generating it." >&2 ;\
		$(MAKE) package_index; \
	else \
		mkdir -p $(TARGET_DIR)/tmp; \
		$(APK) update >&2 || true; \
	fi
endif

package_list: FORCE
	@$(MAKE) -s package_reload
ifeq ($(CONFIG_USE_APK),)
	@$(OPKG) list --size 2>/dev/null
else
	@$(APK) list --size 2>/dev/null
endif

package_install: FORCE
	@echo
	@echo Installing packages...
ifeq ($(CONFIG_USE_APK),)
	$(OPKG) install $(firstword $(wildcard $(LINUX_DIR)/libc_*.ipk $(PACKAGE_DIR)/libc_*.ipk))
	$(OPKG) install $(firstword $(wildcard $(LINUX_DIR)/kernel_*.ipk $(PACKAGE_DIR)/kernel_*.ipk))
	$(OPKG) install $(BUILD_PACKAGES)
else
	$(APK) add --no-scripts $(firstword $(wildcard $(LINUX_DIR)/libc-*.apk $(PACKAGE_DIR)/libc-*.apk))
	$(APK) add --no-scripts $(firstword $(wildcard $(LINUX_DIR)/kernel-*.apk $(PACKAGE_DIR)/kernel-*.apk))
	$(APK) add --no-scripts $(BUILD_PACKAGES)
endif

prepare_rootfs: FORCE
	@echo
	@echo Finalizing root filesystem...

	$(CP) $(TARGET_DIR) $(TARGET_DIR_ORIG)
ifeq ($(CONFIG_USE_APK),)
	$(if $(CONFIG_SIGNATURE_CHECK), \
		$(if $(ADD_LOCAL_KEY), \
			OPKG_KEYS=$(TARGET_DIR)/etc/opkg/keys/ \
			$(SCRIPT_DIR)/opkg-key add $(BUILD_KEY).pub \
		) \
	)
else
	$(if $(CONFIG_SIGNATURE_CHECK), \
		$(if $(ADD_LOCAL_KEY), \
			mkdir -p $(TARGET_DIR)/etc/opkg/keys/; \
			cp $(BUILD_KEY_APK_PUB) $(TARGET_DIR)/etc/apk/keys/; \
		) \
	)
endif
	$(call prepare_rootfs,$(TARGET_DIR),$(USER_FILES),$(DISABLED_SERVICES))

build_image: FORCE
	@echo
	@echo Building images...
	rm -rf $(BUILD_DIR)/json_info_files/
	if [ -d "target/linux/feeds/$(BOARD)" ]; then \
		$(NO_TRACE_MAKE) -C target/linux/feeds/$(BOARD)/image install TARGET_BUILD=1 IB=1 EXTRA_IMAGE_NAME="$(EXTRA_IMAGE_NAME)" \
			$(if $(USER_PROFILE),PROFILE="$(USER_PROFILE)"); \
	else \
		$(NO_TRACE_MAKE) -C target/linux/$(BOARD)/image install TARGET_BUILD=1 IB=1 EXTRA_IMAGE_NAME="$(EXTRA_IMAGE_NAME)" \
			$(if $(USER_PROFILE),PROFILE="$(USER_PROFILE)"); \
	fi

$(BIN_DIR)/profiles.json: FORCE
	$(if $(CONFIG_JSON_OVERVIEW_IMAGE_INFO), \
		WORK_DIR=$(BUILD_DIR)/json_info_files \
			$(SCRIPT_DIR)/json_overview_image_info.py $@ \
	)

json_overview_image_info: $(BIN_DIR)/profiles.json

checksum: FORCE
	@echo
	@echo Calculating checksums...
	@$(call sha256sums,$(BIN_DIR))

clean:
	rm -rf $(TMP_DIR) $(DL_DIR) $(TARGET_DIR) $(BIN_DIR)


info:
	(unset PROFILE FILES PACKAGES MAKEFLAGS; $(MAKE) -s _call_info)

PROFILE_FILTER = $(filter DEVICE_$(PROFILE) $(PROFILE),$(PROFILE_NAMES))

_check_profile: FORCE
ifneq ($(PROFILE),)
  ifeq ($(PROFILE_FILTER),)
	@echo 'Profile "$(PROFILE)" does not exist!'
	@echo 'Use "make info" to get a list of available profile names.'
	@exit 1
  endif
endif

_check_keys: FORCE
ifneq ($(CONFIG_SIGNATURE_CHECK),)
ifeq ($(CONFIG_USE_APK),)
	@if [ ! -s $(BUILD_KEY) -o ! -s $(BUILD_KEY).pub ]; then \
		echo Generate local signing keys... >&2; \
		$(STAGING_DIR_HOST)/bin/usign -G \
			-s $(BUILD_KEY) -p $(BUILD_KEY).pub -c "Local build key"; \
		$(SCRIPT_DIR)/opkg-key add $(BUILD_KEY).pub; \
	fi
	if [ ! -s $(BUILD_KEY).ucert ]; then \
		echo Generate local certificate... >&2; \
		$(STAGING_DIR_HOST)/bin/ucert -I \
			-c $(BUILD_KEY).ucert \
			-p $(BUILD_KEY).pub \
			-s $(BUILD_KEY); \
	fi
else
	@if [ ! -s $(BUILD_KEY_APK_SEC) -o ! -s $(BUILD_KEY_APK_PUB) ]; then \
		echo Generate local signing keys... >&2; \
		$(STAGING_DIR_HOST)/bin/openssl ecparam -name prime256v1 -genkey -noout -out $(BUILD_KEY_APK_SEC); \
		sed -i '1s/^/untrusted comment: Local build key\n/' $(BUILD_KEY_APK_SEC); \
		$(STAGING_DIR_HOST)/bin/openssl ec -in $(BUILD_KEY_APK_SEC) -pubout > $(BUILD_KEY_APK_PUB); \
		sed -i '1s/^/untrusted comment: Local build key\n/' $(BUILD_KEY_APK_PUB); \
	fi
endif
endif

image:
	$(MAKE) -s _check_profile
	$(MAKE) -s _check_keys
	(unset PROFILE FILES PACKAGES MAKEFLAGS; \
	$(MAKE) -s _call_image \
		$(if $(PROFILE),USER_PROFILE="$(PROFILE_FILTER)") \
		$(if $(FILES),USER_FILES="$(FILES)") \
		$(if $(PACKAGES),USER_PACKAGES="$(PACKAGES)") \
		$(if $(BIN_DIR),BIN_DIR="$(BIN_DIR)") \
		$(if $(DISABLED_SERVICES),DISABLED_SERVICES="$(DISABLED_SERVICES)") \
		$(if $(ROOTFS_PARTSIZE),CONFIG_TARGET_ROOTFS_PARTSIZE="$(ROOTFS_PARTSIZE)"))

manifest: FORCE
	$(MAKE) -s _check_profile
	$(MAKE) -s _check_keys
	(unset PROFILE FILES PACKAGES MAKEFLAGS; \
	$(MAKE) -s _call_manifest \
		$(if $(PROFILE),USER_PROFILE="$(PROFILE_FILTER)") \
		$(if $(PACKAGES),USER_PACKAGES="$(PACKAGES)"))

whatdepends: FORCE
ifeq ($(PACKAGE),)
	@echo 'Variable `PACKAGE` is not set but required by `whatdepends`'
	@exit 1
endif
	@$(MAKE) -s package_reload
ifeq ($(CONFIG_USE_APK),)
	@$(OPKG) whatdepends -A $(PACKAGE)
else
	@$(APK) list --depends $(PACKAGE)
endif

.SILENT: help info image manifest whatdepends
