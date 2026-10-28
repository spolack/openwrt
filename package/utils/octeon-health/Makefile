#
# This is free software, licensed under the GNU General Public License v2.
# See /LICENSE for more information.
#

include $(TOPDIR)/rules.mk

PKG_NAME:=octeon-health
PKG_RELEASE:=2

PKG_MAINTAINER:=Simon Polack <spolack+git@mailbox.org>
PKG_LICENSE:=GPL-2.0-only

include $(INCLUDE_DIR)/package.mk

define Package/octeon-health
  SECTION:=utils
  CATEGORY:=Utilities
  TITLE:=OCTEON packet I/O health report for remote debugging
  DEPENDS:=@TARGET_octeon +iperf3 +jsonfilter +tc-tiny +conntrack
  PKGARCH:=all
endef

define Package/octeon-health/description
  Collects what is needed to debug the OCTEON Ethernet (octeon_pip) remotely:
  a snapshot of the system, the links, the pinning and the registers (against
  the values the bootloader left, logged by the test kernel), benchmarks with
  the tester's PC, and one report to attach to a bug report.
  octeon-itus-debug finds out why an SGMII/QSGMII port passes no traffic
  (PHY, PHY interrupt, PCS in-band state, Marvell QSGMII side) and runs
  the experiments that tell the causes apart.
endef

define Build/Compile
endef

define Package/octeon-health/install
	$(INSTALL_DIR) $(1)/usr/sbin $(1)/etc/init.d $(1)/etc/uci-defaults
	$(INSTALL_BIN) ./files/octeon-health $(1)/usr/sbin/octeon-health
	$(INSTALL_BIN) ./files/octeon-itus-debug $(1)/usr/sbin/octeon-itus-debug
	$(INSTALL_BIN) ./files/octeon-health.init $(1)/etc/init.d/octeon-health
	$(INSTALL_BIN) ./files/octeon-health.defaults $(1)/etc/uci-defaults/99-octeon-health
endef

$(eval $(call BuildPackage,octeon-health))
