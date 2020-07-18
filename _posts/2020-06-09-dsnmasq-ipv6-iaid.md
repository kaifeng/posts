---
layout: post
title: dnsmasq的ipv6问题及patch方法
author: kaifeng
date: 2020-06-09
categories: ipv6
---

问题从这里开始 https://bugzilla.redhat.com/show_bug.cgi?id=1575026

ironic在支持IPv6时也遇到一样的问题，因为在PXE启动过程中（UEFI固件、grub、ramdisk），裸机侧会发送几次DHCP请求，但由于client的变化，不同client的算法会产生不同的IAID。dnsmasq在DHCPv6时会对IAID进行检查，如果IAID变化，则拒绝发放地址，并回应no addresses available，只有在lease过期后，再次PXE才可以成功。

其实IAID是否可以变化是IPv6协议埋下的一个坑。

> As best I can see RFC 3315 does say that the IAID MUST remain
consistent across restarts of the DHCP client, but then recognizes
that "There may be no way for a client to maintain consistency of the
IAIDs if it does not have non-volatile storage and the client's
hardware configuration changes"

但dnsmasq社区没有修复的意愿，看起来redhat也不愿意在本地维护修改：http://lists.thekelleys.org.uk/pipermail/dnsmasq-discuss/2017q1/011267.html

不跟随社区的简单解决办法就是去掉host检查，实测忽略IAID检查可以规避该问题。

分析dnsmasq代码，也的确像redhat所说，着实挺乱的。
有两种改法，一种是标准改法，一种是更利于本地维护的改法。下面以dnsmasq 2.76版本为基础版本进行修改。


改法1是标准改法，增加一个布尔型参数，这会占用dnsmasq现有定义的64个bit中的一个，因此在跟随dnsmasq代码的过程中，还得基于新版本重打patch，并调整bit位。

```
commit c009241f7e9c34b44ca149ccc6aa1418a400ef3b
Author: kaifeng <kaifeng@local>
Date:   Mon Jun 8 19:56:32 2020 +0800

    Change way 1

diff --git a/src/dnsmasq.h b/src/dnsmasq.h
index 1896a64..19d686d 100644
--- a/src/dnsmasq.h
+++ b/src/dnsmasq.h
@@ -238,7 +238,8 @@ struct event_desc {
 #define OPT_SCRIPT_ARP     53
 #define OPT_MAC_B64        54
 #define OPT_MAC_HEX        55
-#define OPT_LAST           56
+#define OPT_IGNOREIAID     56
+#define OPT_LAST           57
 
 /* extra flags for my_syslog, we use a couple of facilities since they are known 
    not to occupy the same bits as priorities, no matter how syslog.h is set up. */
@@ -1057,6 +1058,7 @@ extern struct daemon {
   char *addrbuff;
   char *addrbuff2; /* only allocated when OPT_EXTRALOG */
 
+  int ignore_iaid;
 } *daemon;
 
 /* cache.c */
diff --git a/src/option.c b/src/option.c
index d8c57d6..7f64d48 100644
--- a/src/option.c
+++ b/src/option.c
@@ -159,7 +159,8 @@ struct myoption {
 #define LOPT_SCRIPT_ARP    347
 #define LOPT_DHCPTTL       348
 #define LOPT_TFTP_MTU      349
- 
+#define LOPT_IGNOREIAID    350
+
 #ifdef HAVE_GETOPT_LONG
 static const struct option opts[] =  
 #else
@@ -323,6 +324,7 @@ static const struct myoption opts[] =
     { "dns-loop-detect", 0, 0, LOPT_LOOP_DETECT },
     { "script-arp", 0, 0, LOPT_SCRIPT_ARP },
     { "dhcp-ttl", 1, 0 , LOPT_DHCPTTL },
+    { "ignore-iaid", 0, 0 , LOPT_IGNOREIAID },
     { NULL, 0, 0, 0 }
   };
 
@@ -494,6 +496,7 @@ static struct {
   { LOPT_LOOP_DETECT, OPT_LOOP_DETECT, NULL, gettext_noop("Detect and remove DNS forwarding loops."), NULL },
   { LOPT_IGNORE_ADDR, ARG_DUP, "<ipaddr>", gettext_noop("Ignore DNS responses containing ipaddr."), NULL }, 
   { LOPT_DHCPTTL, ARG_ONE, "<ttl>", gettext_noop("Set TTL in DNS responses with DHCP-derived addresses."), NULL }, 
+  { LOPT_IGNOREIAID, OPT_IGNOREIAID, NULL, gettext_noop("Ignore IAID for DHCPv6"), NULL },
   { 0, 0, NULL, NULL, NULL }
 }; 
 
diff --git a/src/rfc3315.c b/src/rfc3315.c
index 3f4d69c..a0ccdda 100644
--- a/src/rfc3315.c
+++ b/src/rfc3315.c
@@ -1723,10 +1723,19 @@ static int check_address(struct state *state, struct in6_addr *addr)
   if (!(lease = lease6_find_by_addr(addr, 128, 0)))
     return 1;
 
-  if (lease->clid_len != state->clid_len || 
-      memcmp(lease->clid, state->clid, state->clid_len) != 0 ||
-      lease->iaid != state->iaid)
-    return 0;
+  log6_packet(state, "DEBUGGING", NULL, _("Checking address"));
+  if(option_bool(OPT_IGNOREIAID)) {
+	log6_packet(state, "DEBUGGING", NULL, _("Ignore iaid is ON"));
+    if (lease->clid_len != state->clid_len || 
+        memcmp(lease->clid, state->clid, state->clid_len) != 0)
+      return 0;
+  } else {
+	log6_packet(state, "DEBUGGING", NULL, _("Ignore iaid is OFF"));
+    if (lease->clid_len != state->clid_len || 
+        memcmp(lease->clid, state->clid, state->clid_len) != 0 ||
+        lease->iaid != state->iaid)
+      return 0;
+  }
 
   return 1;
 }
```

改法2则比较适合线下维护，引入一个自定义变量，自行在参数解析处增加解析，然后通过全局变量的方式控制代码行为。缺点是不能通过命令行指定参数的方式运行，需要写入配置文件，并以指定配置文件的方式运行dnsmasq。这种方式就很适合ironic inspector使用。

```
commit 34f7f596941a183097586e3d2140e3e7355079c9
Author: kaifeng <kaifeng@local>
Date:   Mon Jun 8 20:15:30 2020 +0800

    Way 2

diff --git a/src/option.c b/src/option.c
index d8c57d6..844f5a7 100644
--- a/src/option.c
+++ b/src/option.c
@@ -22,6 +22,7 @@
 static volatile int mem_recover = 0;
 static jmp_buf mem_jmp;
 static int one_file(char *file, int hard_opt);
+int global_ignore_iaid = 0;
 
 /* Solaris headers don't have facility names. */
 #ifdef HAVE_SOLARIS_NETWORK
@@ -159,7 +160,8 @@ struct myoption {
 #define LOPT_SCRIPT_ARP    347
 #define LOPT_DHCPTTL       348
 #define LOPT_TFTP_MTU      349
- 
+#define LOPT_IGNOREIAID    900
+
 #ifdef HAVE_GETOPT_LONG
 static const struct option opts[] =  
 #else
@@ -323,6 +325,7 @@ static const struct myoption opts[] =
     { "dns-loop-detect", 0, 0, LOPT_LOOP_DETECT },
     { "script-arp", 0, 0, LOPT_SCRIPT_ARP },
     { "dhcp-ttl", 1, 0 , LOPT_DHCPTTL },
+	{ "ignore-iaid", 1, 0 , LOPT_IGNOREIAID },
     { NULL, 0, 0, 0 }
   };
 
@@ -494,6 +497,7 @@ static struct {
   { LOPT_LOOP_DETECT, OPT_LOOP_DETECT, NULL, gettext_noop("Detect and remove DNS forwarding loops."), NULL },
   { LOPT_IGNORE_ADDR, ARG_DUP, "<ipaddr>", gettext_noop("Ignore DNS responses containing ipaddr."), NULL }, 
   { LOPT_DHCPTTL, ARG_ONE, "<ttl>", gettext_noop("Set TTL in DNS responses with DHCP-derived addresses."), NULL }, 
+  { LOPT_IGNOREIAID, ARG_ONE, "<1/0>", gettext_noop("Ignore IAID for DHCPv6"), NULL },
   { 0, 0, NULL, NULL, NULL }
 }; 
 
@@ -3986,6 +3990,10 @@ static int one_opt(int option, char *arg, char *errstr, char *gen_err, int comma
 	daemon->host_records_tail = new;
 	break;
       }
+	case LOPT_IGNOREIAID:
+		if (!atoi_check(arg, &global_ignore_iaid))
+	      ret_err(_("bad ignore iaid setting"));
+        break;
 
 #ifdef HAVE_DNSSEC
     case LOPT_DNSSEC_STAMP:
diff --git a/src/rfc3315.c b/src/rfc3315.c
index 3f4d69c..ef7f8fb 100644
--- a/src/rfc3315.c
+++ b/src/rfc3315.c
@@ -1715,6 +1715,7 @@ static void mark_config_used(struct dhcp_context *context, struct in6_addr *addr
       context->flags |= CONTEXT_CONF_USED;
 }
 
+extern int global_ignore_iaid;
 /* make sure address not leased to another CLID/IAID */
 static int check_address(struct state *state, struct in6_addr *addr)
 { 
@@ -1723,10 +1724,19 @@ static int check_address(struct state *state, struct in6_addr *addr)
   if (!(lease = lease6_find_by_addr(addr, 128, 0)))
     return 1;
 
-  if (lease->clid_len != state->clid_len || 
-      memcmp(lease->clid, state->clid, state->clid_len) != 0 ||
-      lease->iaid != state->iaid)
-    return 0;
+  log6_packet(state, "DEBUGGING", NULL, _("Checking address"));
+  if(global_ignore_iaid) {
+	log6_packet(state, "DEBUGGING", NULL, _("Ignore iaid is ON"));
+    if (lease->clid_len != state->clid_len || 
+        memcmp(lease->clid, state->clid, state->clid_len) != 0)
+      return 0;
+  } else {
+	log6_packet(state, "DEBUGGING", NULL, _("Ignore iaid is OFF"));
+    if (lease->clid_len != state->clid_len || 
+        memcmp(lease->clid, state->clid, state->clid_len) != 0 ||
+        lease->iaid != state->iaid)
+      return 0;
+  }
 
   return 1;
 }
```
