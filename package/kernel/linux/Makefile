#
# Copyright (C) 2006-2010 OpenWrt.org
#
# This is free software, licensed under the GNU General Public License v2.
# See /LICENSE for more information.
#

include $(TOPDIR)/rules.mk
include $(INCLUDE_DIR)/kernel.mk

PKG_NAME:=kernel
PKG_FLAGS:=hold

PKG_BUILD_DIR:=$(KERNEL_BUILD_DIR)/packages
SUBTARGETS = $(sort $(filter-out feeds,$(notdir $(wildcard $(TOPDIR)/target/linux/* $(TOPDIR)/target/linux/feeds/*))))
SUBTARGET_MODULES = $(foreach t,$(SUBTARGETS),$(firstword $(wildcard $(TOPDIR)/target/linux/feeds/$(t)/modules.mk $(TOPDIR)/target/linux/$(t)/modules.mk)))
SCAN_DEPS=modules/*.mk $(SUBTARGET_MODULES) $(TOPDIR)/include/netfilter.mk

PKG_LICENSE:=GPL-2.0-only
PKG_LICENSE_FILES:=COPYING

export SHELL:=/bin/sh
.ONESHELL:
.SHELLFLAGS = -ec

include $(INCLUDE_DIR)/package.mk

ifeq ($(DUMP),)
  STAMP_BUILT:=$(STAMP_BUILT)_$(shell $(SCRIPT_DIR)/kconfig.pl $(LINUX_DIR)/.config | $(MKHASH) md5)
  -include $(LINUX_DIR)/.config
endif

define Build/Prepare
	mkdir -p $(PKG_BUILD_DIR)
endef

define Build/Configure
endef

define Build/Quilt
endef

define Build/Compile
endef

define KernelPackage/depends
endef

CONFIG_PACKAGE_kernel=y
define Package/kernel
  SECTION:=sys
  CATEGORY:=Kernel
  DEFAULT:=y
  TITLE:=Virtual kernel package
  VERSION:=$(LINUX_VERSION)~$(LINUX_VERMAGIC)-r$(LINUX_RELEASE)
  URL:=http://www.kernel.org/
  PKG_FLAGS:=nonshared
endef

define Package/kernel/install
  # nothing to do
endef

define Package/kernel/extra_provides
	sed -e 's,.*/,,' $(LINUX_DIR)/modules.builtin;
endef

$(eval $(if $(DUMP),,$(call BuildPackage,kernel)))

include $(sort $(wildcard ./modules/*.mk))
-include $(SUBTARGET_MODULES)
