#
# Copyright (C) 2006-2010 OpenWrt.org
#
# This is free software, licensed under the GNU General Public License v2.
# See /LICENSE for more information.
#

curdir:=package

include $(INCLUDE_DIR)/feeds.mk
include $(INCLUDE_DIR)/rootfs.mk

-include $(TMP_DIR)/.packagedeps
package-y += kernel/linux
$(curdir)/autoremove:=1
$(curdir)/builddirs:=$(sort $(package-) $(package-y) $(package-m))
$(curdir)/builddirs-default:=. $(sort $(package-y) $(package-m))
$(curdir)/builddirs-prereq:=. $(sort $(prereq-y) $(prereq-m))
ifdef CHECK_ALL
$(curdir)/builddirs-check:=$($(curdir)/builddirs)
$(curdir)/builddirs-download:=$($(curdir)/builddirs)
endif
ifneq ($(IGNORE_ERRORS),)
  package-y-filter := $(package-y)
  package-m-filter := $(filter-out $(package-y),$(package-m))
  package-n-filter := $(filter-out $(package-y) $(package-m),$(package-))
  package-ignore-errors := $(filter n m y,$(IGNORE_ERRORS))
  package-ignore-errors := $(if $(package-ignore-errors),$(package-ignore-errors),n m)
  package-ignore-subdirs := $(sort $(foreach m,$(package-ignore-errors),$(package-$(m)-filter)))
  $(curdir)/builddirs-ignore-download := $(package-ignore-subdirs)
  $(curdir)/builddirs-ignore-compile := $(package-ignore-subdirs)
  $(curdir)/builddirs-ignore-host-download := $(package-ignore-subdirs)
  $(curdir)/builddirs-ignore-host-compile := $(package-ignore-subdirs)
endif

PACKAGE_INSTALL_FILES:= \
	$(foreach pkg,$(sort $(package-y)), \
		$(foreach variant, \
			$(if $(strip $(package/$(pkg)/variants)), \
				$(package/$(pkg)/variants), \
				$(if $(package/$(pkg)/default-variant), \
					$(package/$(pkg)/default-variant), \
					default \
				) \
			), \
			$(PKG_INFO_DIR)/$(lastword $(subst /,$(space),$(pkg))).$(variant).install \
		) \
	)

$(curdir)/cleanup: $(TMP_DIR)/.build
	rm -rf $(STAGING_DIR_ROOT)

$(curdir)/merge:
	rm -rf $(PACKAGE_DIR_ALL)
	mkdir -p $(PACKAGE_DIR_ALL)
ifneq ($(CONFIG_USE_APK),)
	-$(foreach pdir,$(PACKAGE_SUBDIRS),$(if $(wildcard $(pdir)/*.apk),ln -s $(pdir)/*.apk $(PACKAGE_DIR_ALL);))
else
	-$(foreach pdir,$(PACKAGE_SUBDIRS),$(if $(wildcard $(pdir)/*.ipk),ln -s $(pdir)/*.ipk $(PACKAGE_DIR_ALL);))
endif

$(BUILD_KEY_APK_SEC):
	$(STAGING_DIR_HOST)/bin/openssl ecparam -name prime256v1 -genkey -noout -out $(BUILD_KEY_APK_SEC)

$(BUILD_KEY_APK_PUB): $(BUILD_KEY_APK_SEC)
	$(STAGING_DIR_HOST)/bin/openssl ec -in $(BUILD_KEY_APK_SEC) -pubout > $(BUILD_KEY_APK_PUB)

$(curdir)/merge-index: $(curdir)/merge
ifneq ($(CONFIG_USE_APK),)
	(cd $(PACKAGE_DIR_ALL) && $(STAGING_DIR_HOST)/bin/apk mkndx \
			--root $(TOPDIR) \
			--keys-dir $(TOPDIR) \
			--sign $(BUILD_KEY_APK_SEC) \
			--output packages.adb \
			*.apk; \
	)
else
	(cd $(PACKAGE_DIR_ALL) && $(SCRIPT_DIR)/ipkg-make-index.sh . 2>&1 > Packages; )
endif

ifndef SDK
  $(curdir)//compile = $(STAGING_DIR)/.prepared $(BIN_DIR)
ifneq ($(CONFIG_USE_APK),)
  $(curdir)//compile += $(curdir)/system/apk/host/compile $(BUILD_KEY_APK_SEC) $(BUILD_KEY_APK_PUB)
else
  $(curdir)/compile: $(curdir)/system/opkg/host/compile
endif
else
ifneq ($(CONFIG_USE_APK),)
  $(curdir)//compile += $(BUILD_KEY_APK_SEC) $(BUILD_KEY_APK_PUB)
endif
endif

$(curdir)/install: $(TMP_DIR)/.build $(curdir)/merge $(curdir)/merge-index
	- find $(STAGING_DIR_ROOT) -type d | $(XARGS) chmod 0755
	rm -rf $(TARGET_DIR) $(TARGET_DIR_ORIG)
	mkdir -p $(TARGET_DIR)/tmp
ifneq ($(CONFIG_USE_APK),)
	$(file >$(TMP_DIR)/apk_install_list,\
	    $(foreach pkg,$(shell cat $(PACKAGE_INSTALL_FILES) 2>/dev/null),$(pkg)$(call GetABISuffix,$(pkg))))
	$(call apk,$(TARGET_DIR)) add --initdb --no-scripts --arch $(ARCH_PACKAGES) \
		--repositories-file /dev/zero --repository file://$(PACKAGE_DIR_ALL)/packages.adb \
		$$(cat $(TMP_DIR)/apk_install_list)
else
	$(file >$(TMP_DIR)/opkg_install_list,\
	  $(call opkg_package_files,\
	    $(foreach pkg,$(shell cat $(PACKAGE_INSTALL_FILES) 2>/dev/null),$(pkg)$(call GetABISuffix,$(pkg)))))
	$(call opkg,$(TARGET_DIR)) install $$(cat $(TMP_DIR)/opkg_install_list)
	@for file in $(PACKAGE_INSTALL_FILES); do \
		[ -s $$file.flags ] || continue; \
		for flag in `cat $$file.flags`; do \
			$(call opkg,$(TARGET_DIR)) flag $$flag `cat $$file`; \
		done; \
	done || true
endif

	$(CP) $(TARGET_DIR) $(TARGET_DIR_ORIG)

	$(call prepare_rootfs,$(TARGET_DIR),$(TOPDIR)/files)

$(curdir)/index: FORCE
	@echo Generating package index...
ifneq ($(CONFIG_USE_APK),)
	@for d in $(PACKAGE_SUBDIRS); do \
		mkdir -p $$d; \
		cd $$d || continue; \
		ls *.apk >/dev/null 2>&1 || continue; \
		$(STAGING_DIR_HOST)/bin/apk mkndx \
			--root $(TOPDIR) \
			--keys-dir $(TOPDIR) \
			--sign $(BUILD_KEY_APK_SEC) \
			--output packages.adb \
			$$(ls *.apk | grep -v 'kernel\|libc'); \
	done
else
	@for d in $(PACKAGE_SUBDIRS); do ( \
		mkdir -p $$d; \
		cd $$d || continue; \
		$(SCRIPT_DIR)/ipkg-make-index.sh . 2>&1 > Packages.manifest; \
		grep -vE '^(Maintainer|LicenseFiles|Source|SourceName|Require|SourceDateEpoch)' Packages.manifest > Packages; \
		case "$$(((64 + $$(stat -L -c%s Packages)) % 128))" in 110|111) \
			$(call ERROR_MESSAGE,WARNING: Applying padding in $$d/Packages to workaround usign SHA-512 bug!); \
			{ echo ""; echo ""; } >> Packages;; \
		esac; \
		$(SCRIPT_DIR)/make-index-json.py -f opkg -a "$(ARCH_PACKAGES)" Packages > index.json; \
		gzip -9nc Packages > Packages.gz; \
	); done
ifdef CONFIG_SIGNED_PACKAGES
	@echo Signing package index...
	@for d in $(PACKAGE_SUBDIRS); do ( \
		[ -d $$d ] && \
			cd $$d || continue; \
		$(STAGING_DIR_HOST)/bin/usign -S -m Packages -s $(BUILD_KEY); \
	); done
endif
ifdef CONFIG_JSON_CYCLONEDX_SBOM
	@echo Creating CycloneDX package SBOMs...
	@for d in $(PACKAGE_SUBDIRS); do ( \
		[ -d $$d ] && \
			cd $$d || continue; \
		$(SCRIPT_DIR)/package-metadata.pl pkgcyclonedxsbom Packages.manifest > Packages.bom.cdx.json || true; \
	); done
endif
endif

$(curdir)/flags-install:= -j1

$(eval $(call stampfile,$(curdir),package,prereq,.config))
$(eval $(call stampfile,$(curdir),package,cleanup,$(TMP_DIR)/.build))
$(eval $(call stampfile,$(curdir),package,compile,$(TMP_DIR)/.build))
$(eval $(call stampfile,$(curdir),package,install,$(TMP_DIR)/.build))
$(eval $(call stampfile,$(curdir),package,check,$(TMP_DIR)/.build))

$(eval $(call subdir,$(curdir)))
